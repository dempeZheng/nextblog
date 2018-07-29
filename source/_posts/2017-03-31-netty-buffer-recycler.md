---
layout: post
title:  "记一次netty内存溢出的问题(下)"
date:  2017-03-31
categories: [netty]
keywords: netty,ByteBuf,pool,outOfMemory,Recycler
---
上一篇：[记一次netty内存溢出的问题(上)](http://www.zhizus.com/2017-03-28-netty-outofmemory.html)


在压测基于netty的RPC框架过程中发现了老生代的内存会不断膨胀，即便触发`full gc`也不会被回收。通过工具dump下来发现跟Recycler相关。

{% note default %}
Netty的内存分配用到了Recycler机制。Recycler可以看作一个对象池，netty在分配内存的时候首先会从Recycler中`get()`,实际上是从FastLocalThread中获取，如果有便不重新分配。内存的`release()`的时候又会调用recycler()将对象回收。netty采用这种机制减少对象创建的损耗和减少垃圾回收从而提升性能。
{% endnote %}

<!--More-->
## 首先来理一理项目内存分配和释放的几个地方。
项目中仅仅在encoder&decoder中用到了内存的分配和释放，而且非常确定这两个地方用到的是netty的IO线程。分配释放都是同一个线程。

既然用到netty的IO线程，那么recycler的容量就应该是约等于 IO线程数，（netty的NioEventLoop继承的是SingleThreadEventLoop，也就是说在每个EventLoop里面encoder&decoder都是串行） 按照这么来说，肯定不会内存溢出才对。

## 查看源代码
仔细看来netty的源代码，确认decoder这块肯定是没问题，ReadEvent事件触发了，一条线程处理内存的申请和释放。
encoder这块就稍稍复杂一些。首先，项目是在自己的业务线程中 调用`ctx.writeAndFlush(message)`来写消息的。
这个方法会先判断是否在IO线程中，如果不再则会包装成一个`WriteAndFlushTask`丢到IO线程处理。

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
           //包装成WriteAndFlushTask
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        // 丢到EventLoop线程中执行，即IO线程
        //顺便说一下，如果这里不这样包装，业务开发者很容易在其他非IO线程中来write，这样很可能会造成多线程往同一条channel里面写数据，造成并发问题。早期的版本是在write的地方加锁的。这样实际上跟netty标榜的串行无所话设计相违背。
        safeExecute(executor, task, promise, m);
    }
}
```
那么可以确认 decoder也一定是在IO线程中完成的，也就是decoder的内存申请和释放理论上是在同一个线程中来完成的。

**既然这样 问题会出现在哪里呢**？

跟进WriteAndFlushTask，一步一步看他怎么处理的，既然出现了问题，那么顺着问题一定能找到原因。
跟进去我们发现 实际上writeAndFlushTask分成了两个动作，一个是write，一个是flush。

```java
  public void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
            // write 方法将会循环调用handler中的write方法，最后会调用HeadContext中write，将buffer写到ChannelOutBoundBuffer中，这一个过程中不会release buffer，因为他还没有真正的写入流中
            super.write(ctx, msg, promise);
            // flush才是 真正的将数据写入流
            ctx.invokeFlush();
        }
```

write部分并未将数据真正的写出去，而是将数据丢到ChannelOutBoundBuffer中，等到flush的时候才会调用NioSocketChannel中的doWrite来写入数据。
内存的是在`MessageToByteDecoder`中申请的，这个也是通过IO线程中，而内存的释放是在flush中，即真正将数据写入后才释放的。也就是在NioSocketChannel的doWrite方法中释放的。

```java
   @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        for (;;) {
            int size = in.size();
            if (size == 0) {
                // All written so clear OP_WRITE
                clearOpWrite();
                break;
            }
            long writtenBytes = 0;
            boolean done = false;
            boolean setOpWrite = false;

            // Ensure the pending writes are made of ByteBufs only.
            ByteBuffer[] nioBuffers = in.nioBuffers();
            int nioBufferCnt = in.nioBufferCount();
            long expectedWrittenBytes = in.nioBufferSize();
            SocketChannel ch = javaChannel();

            // Always us nioBuffers() to workaround data-corruption.
            // See https://github.com/netty/netty/issues/2761
            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    super.doWrite(in);
                    return;
                case 1:
                    // Only one ByteBuf so use non-gathering write
                    ByteBuffer nioBuffer = nioBuffers[0];
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        final int localWrittenBytes = ch.write(nioBuffer);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
                default:
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
            }

            // Release the fully written buffers, and update the indexes of the partially written buffer.
            // 这个方法包装了释放buffer的操作，调用这个才会真正的释放buffer，才会通过recycler方法回收buffer
            in.removeBytes(writtenBytes);

            if (!done) {
                // Did not write all buffers completely.
                incompleteWrite(setOpWrite);
                break;
            }
        }
    }
```

乍一看 也发现不了什么问题，虽然write的时候先写到了`ChannelOutBoundBuffer`，但是随即我们在同一线程中调用了flush，flush之后，我们也将引用的内存释放了。
**我们回过头看看flush方法，看看是不是有什么情况，不会调用flush了**

```java
 protected void flush0() {
            if (inFlush0) {
                // Avoid re-entrance
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null || outboundBuffer.isEmpty()) {
                return;
            }

            inFlush0 = true;

            // Mark all pending write requests as failure if the channel is inactive.
            // 上面注释说标记所有的pending wirte request as failure if channel is inactive，也就说所如果channel is inactive的是所有的pending write request is failure了，这个时候也不会调用flush了。
            //但是跟进去看failFlushed方法的时候发现这里也释放了buffer。。。。。。。
            if (!isActive()) {
                try {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(FLUSH0_NOT_YET_CONNECTED_EXCEPTION, true);
                    } else {
                        // Do not trigger channelWritabilityChanged because the channel is closed already.
                        outboundBuffer.failFlushed(FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                    }
                } finally {
                    inFlush0 = false;
                }
                return;
            }

            try {
                doWrite(outboundBuffer);
            } catch (Throwable t) {
                if (t instanceof IOException && config().isAutoClose()) {
                    /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
                    close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
                } else {
                    outboundBuffer.failFlushed(t, true);
                }
            } finally {
                inFlush0 = false;
            }
        }
```

上面的代码可以看出，`if (!isActive())`这个时候直接就return了，不会调用flush了。
即便这种情况 netty也是通过failFlushed这个方法release了buffer。
细想一下，如果没有release buffer 那么肯定是内存泄漏了，一定是bug。项目用的是4.1xxRelease版本的，应该不可能有这种版本。

既然这样只能换条思路推导。
所有的buffer都会被release，而且只能在同一线程里面release，那么出现recycler膨胀的可能只有是在一些极端情况，netty将这些情况包装成Task（这些task里面会包含一些释放buffer的动作），然后丢到对应的EventLoop来处理。

好了片刻，没发现这样的task。

 打算将标题改为从查找×××到放弃了。

最后终于发现`ChannelOutBoundBuffer`中的close是包装在Task中，而且有释放buffer的操作。代码如下：

```java
 private void close(final ChannelPromise promise, final Throwable cause,
                           final ClosedChannelException closeCause, final boolean notify) {
            if (!promise.setUncancellable()) {
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                // Only needed if no VoidChannelPromise.
                if (!(promise instanceof VoidChannelPromise)) {
                    // This means close() was called before so we just register a listener and return
                    closeFuture.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            promise.setSuccess();
                        }
                    });
                }
                return;
            }

            if (closeFuture.isDone()) {
                // Closed already.
                safeSetSuccess(promise);
                return;
            }

            final boolean wasActive = isActive();
            this.outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
            Executor closeExecutor = prepareToClose();
            if (closeExecutor != null) {
                closeExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // Execute the close.
                            doClose0(promise);
                        } finally {
                            // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                            invokeLater(new Runnable() {
                                @Override
                                public void run() {
                                    // Fail all the queued messages
                                    outboundBuffer.failFlushed(cause, notify);
                                    // 这里的close里面会调用 ReferenceCountUtil.safeRelease(e.msg);这个来释放buffer，这个task可能会在eventLoop的任务队列中挤压，释放buffer的跟分配buffer就变成了异步的。
                                    outboundBuffer.close(closeCause);
                                    fireChannelInactiveAndDeregister(wasActive);
                                }
                            });
                        }
                    }
                });
            } else {
                try {
                    // Close the channel and fail the queued messages in all cases.
                    doClose0(promise);
                } finally {
                    // Fail all the queued messages.
                    outboundBuffer.failFlushed(cause, notify);
                    outboundBuffer.close(closeCause);
                }
                if (inFlush0) {
                    invokeLater(new Runnable() {
                        @Override
                        public void run() {
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                } else {
                    fireChannelInactiveAndDeregister(wasActive);
                }
            }
        }
```
当然这里只找到了一个可能造成问题的点，至于是不是我代码里面造成recycler膨胀的原因暂时还不得而知，只能下次有空再分析了。



## 总结
尽管在正常情况下recycler的容量不会增加，但是在一些极端的情况，比如压测环境，recycler是非常有可能会膨胀的。而且一旦膨胀那么这些对象将会在老生代里面存活，不会被回收。
所以在使用netty的内存分配的时候还是要谨慎，前期的压测工作不可以省。
