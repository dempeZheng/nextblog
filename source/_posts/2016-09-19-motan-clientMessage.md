---
layout: post
title:  "Motan源码解读-Client异步消息"
date:  2016-09-19
categories: [Motan源码解读]
keywords: motan,源码,client,异步消息
---

## 如何设计RPC异步消息？

异步调用的好处就不说了，我们直入主题，讨论RPC异步消息如何设计？

既然是异步的，我们有一个首要的问题需要解决：

>通常异步调用都是一问一答的模式，一个Request拿到一个Response，如果我们做了异步，怎么把这两个消息对应起来呢？

既然需要对应，那么只有一种思路就是通过一种方式把两者联系起来，所以我们在设计协议的时候必须包含一个messageID，唯一的，这个唯一性是在客户端维护（多是自增）。

客户端发送请求的req的时候，先req.setMessageId(messageID)，然后在发送；服务端拿到这个messageID的时候，把messageID原封不动的包装在Response中，这样客户端就可以把request和response对应起来。

## 纸上得来终觉浅，我们试着来撸一个

我们写一个简单的DefaultClient

``` java

public class DefaultClient {

    protected Map<Integer, Context> contextMap = new ConcurrentHashMap<Integer, Context>();
  
    public void initClientChannel(SocketChannel ch) {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.
                .addLast("ClientHandler", new ChannelHandlerAdapter() {
                    /**
                    * 收到服务端消息
                    **/
                    @Override
                    public void channelRead(ChannelHandlerContext ctx, Response resp) throws Exception {
                    
                        // 收到服务端消息，找出对应包含context的request上下文，然后移除
                        Context context = contextMap.remove(id);
                        if (context == null) {
                            LOGGER.debug("messageID:{}, take Context null", id);
                            return;
                        }
                        //异步通知结果返回
                        context.cb.onReceive(resp);
                    }
                });
    }

    /**
     * 发送消息，并等待Response
     *
     * @param request
     * @return Response
     */
    public Callback call(Request request, Callback callback) throws Exception {
        int id = getNextMessageId();
        request.setMessageID(id);
        Context context = new Context(id, request, callback);
        // 发送消息前先把request上下文保存起来，
        contextMap.put(id, context);
        writeAndFlush(request);
        // 消息发送出去就直接返回callback，不会阻塞等待服务端结果返回
        // 当客户端收到服务端返回的response首先会通过messageID找到对应的request上下文，然后调用callback的onReceive(resp)方法通知
        return callback;
    }

    public Future<Response> send(Request request) throws Exception {
        Promise<Response> future = new Promise<Response>();
        call(request, future);
        return future;
    }

    public Response sendAnWait(Request request) throws Exception {
        Future<Response> future = send(request);
        return future.await();
    }

    public Response sendAnWait(Request request, long amount, TimeUnit unit) throws Exception {
        Future<Response> future = send(request);
        return future.await(amount, unit);
    }

    public static class Context {
        final Request request;
        final Callback cb;
        private final short id;

        Context(short id, Request request, Callback cb) {
            this.id = id;
            this.cb = cb;
            this.request = request;
        }
    }

}

```

DefaultClient做的事情就是在发送Request消息时候先保存request的上下文，等到服务端消息返回，然后通过callback异步通知。
接下来我们要看看callback的实现了：

``` java

public class Promise<T> implements Callback<T>, Future<T> {
    private final CountDownLatch latch = new CountDownLatch(1);
    Throwable error;
    private T message;

    @Override
    public void onReceive(T message) {
        synchronized (this) {
            this.message = message;
            latch.countDown();
        }
    }

    // 当onReceive调用之前，await方法会阻塞，一直到timeout
    public T await(long amount, TimeUnit unit) throws Exception {
        if (latch.await(amount, unit)) {
            return get();
        } else {
            throw new TimeoutException();
        }
    }

    public T await() throws Exception {
        latch.await();
        return get();
    }

    private T get() throws Exception {
        Throwable e = error;
        if (e != null) {
            if (e instanceof RuntimeException) {
                throw (RuntimeException) e;
            } else if (e instanceof Exception) {
                throw (Exception) e;
            } else if (e instanceof Error) {
                throw (Error) e;
            } else {
                // don'M expect to hit this case.
                throw new RuntimeException(e);
            }
        }
        return message;
    }
}
```

利用java的CountDownLatch可以非常轻松的实现：当回调方法调用之前，获取结果的await方法一直阻塞的。

到此为止，我们基本实现异步消息调用的核心`伪代码`，但是是不是万事大吉了呢？

## But，还有一个大bug

还有什么问题？

>服务端异常状态没有机制返回给客户端？

当然这个也是一个很严重的问题，但是只能算是一个需要优化的地方，并不是下面要说的大bug。

**我们思考一下下面的问题：**
- 1.客户端发送异常的时候，contextMap中存储的Request上下文如何回收？
- 2.服务端超时或者异常情况，contextMap又如何回收？


问题1很要解决，在发送的时候捕获一下发送的异常，出现异常主动将contextMap的对象移除。伪代码如下：

``` java
    public Callback call(Request request, Callback callback)  {
        int id = getNextMessageId();
        request.setMessageID(id);
        Context context = new Context(id, request, callback);
        // 发送消息前先把request上下文保存起来，
        contextMap.put(id, context);
        try{
             writeAndFlush(request);
        }catch(Exception e){
             contextMap.remove(id);
        }
        // 消息发送出去就直接返回callback，不会阻塞等待服务端结果返回
        // 当客户端收到服务端返回的response首先会通过messageID找到对应的request上下文，然后调用callback的onReceive(resp)方法通知
        return callback;
    }
```

关键是问题2，服务端超时，或者异常，这时候客户端收不到任何消息，那么上面的代码中contextMap的request上下文就没办法清理掉。

这个时候该如何处理呢？
我们先讨论RPC一个很重的点：`TimeOut机制`。
如果没有timeout，假设某个服务响应慢或者不可用，可能造成客户端所有的connection都在等待请求响应，其他可用的服务也无法提供服务。
客户端请求分配的资源是有限的，我们不能因为某个业务不可用或者响应慢而让他占用了所有的资源。`对于每个异步调用的请求，我们最好都能给他一个默认的超时时间`。

既然RPC有timeout机制，那么处理问题2最自然的思路就是定时去扫描contextMap，将里面超时的context清理掉。`motan也是这么做的`

不过，我们还有其他的办法：

我们在客户端发送请求前，先通过`timer.newTimeout(new ClearContextMapTask(){},....)`,等到这个连接超时，就会自动执行这个ClearContextMapTask，clean对应的超时request上下文。
这个还没完，因为很多时候，消息能够正常返回，`正常返回的时候我们主动将这个任务cancel掉`。

具体的实现可以参考https://github.com/Baidu-ecom/Jprotobuf-rpc-socket，（貌似是百度出品，代码质量还是有保障的）

这个不是本文的重点，暂不详细展开。

## 言归正传，我们来说Motan怎么实现

上面说了一堆，貌似跑题了，不过接下来，我们来剖析Motan 的源代码，看Motan是怎么实现这个逻辑的。

直接看源代码，（为了简洁，略掉不相关逻辑代码）

``` java
public class NettyClient extends AbstractPoolClient implements StatisticCallback {

	// 回收过期任务
	private static ScheduledExecutorService scheduledExecutor = Executors.newScheduledThreadPool(4);

	// 异步的request，需要注册callback future
	// 触发remove的操作有： 1) service的返回结果处理。 2) timeout thread cancel
	protected ConcurrentMap<Long, NettyResponseFuture> callbackMap = new ConcurrentHashMap<Long, NettyResponseFuture>();

	private ScheduledFuture<?> timeMonitorFuture = null;
	public NettyClient(URL url) {
			timeMonitorFuture = scheduledExecutor.scheduleWithFixedDelay(
				new TimeoutMonitor("timeout_monitor_" + url.getHost() + "_" + url.getPort()),
				MotanConstants.NETTY_TIMEOUT_TIMER_PERIOD, MotanConstants.NETTY_TIMEOUT_TIMER_PERIOD,
				TimeUnit.MILLISECONDS);
	}

	/**
	 * 请求remote service
	 * 
	 * <pre>
	 * 		1)  get connection from pool
	 * 		2)  async requset
	 * 		3)  return connection to pool
	 * 		4)  check if async return response, true: return ResponseFuture;  false: return result
	 * </pre>
	 * 
	 * @param request
	 * @param async
	 * @return
	 * @throws TransportException
	 */
	private Response request(Request request, boolean async) throws TransportException {
		...
		// async request
		response = channel.request(request);
		response = asyncResponse(response, async);
		return response;
	}

	/**
	 * 初始化 netty clientBootstrap
	 */
	private void initClientBootstrap() {
		
		bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
			...
			public ChannelPipeline getPipeline() {
				ChannelPipeline pipeline = Channels.pipeline();
				pipeline.addLast("decoder", new NettyDecoder(codec, NettyClient.this, maxContentLength));
				pipeline.addLast("encoder", new NettyEncoder(codec, NettyClient.this));
				pipeline.addLast("handler", new NettyChannelHandler(NettyClient.this, new MessageHandler() {
					@Override
					public Object handle(Channel channel, Object message) {
						Response response = (Response) message;

						NettyResponseFuture responseFuture = NettyClient.this.removeCallback(response.getRequestId());

						if (responseFuture == null) {
							LoggerUtil.warn(
									"NettyClient has response from server, but resonseFuture not exist,  requestId={}",
									response.getRequestId());
							return null;
						}

						if (response.getException() != null) {
							responseFuture.onFailure(response);
						} else {
							responseFuture.onSuccess(response);
						}

						return null;
					}
				}));
				return pipeline;
			}
		});
	}

	/**
	 * 注册回调的resposne
	 * 
	 * <pre>
	 * 
	 * 		进行最大的请求并发数的控制，如果超过NETTY_CLIENT_MAX_REQUEST的话，那么throw reject exception
	 * 
	 * </pre>
	 * 
	 * @throws MotanServiceException
	 * @param requestId
	 * @param nettyResponseFuture
	 */
	public void registerCallback(long requestId, NettyResponseFuture nettyResponseFuture) {
		if (this.callbackMap.size() >= MotanConstants.NETTY_CLIENT_MAX_REQUEST) {
			// reject request, prevent from OutOfMemoryError
			throw new MotanServiceException("NettyClient over of max concurrent request, drop request, url: "
					+ url.getUri() + " requestId=" + requestId, MotanErrorMsgConstant.SERVICE_REJECT);
		}

		this.callbackMap.put(requestId, nettyResponseFuture);
	}

	/**
	 * 移除回调的response
	 * 
	 * @param requestId
	 * @return
	 */
	public NettyResponseFuture removeCallback(long requestId) {
		return callbackMap.remove(requestId);
	}

	/**
	 * 回收超时任务
	 * 
	 * @author maijunsheng
	 * 
	 */
	class TimeoutMonitor implements Runnable {
		private String name;
		public TimeoutMonitor(String name) {
			this.name = name;
		}

		public void run() {
			long currentTime = System.currentTimeMillis();
			for (Map.Entry<Long, NettyResponseFuture> entry : callbackMap.entrySet()) {
				try {
					NettyResponseFuture future = entry.getValue();

					if (future.getCreateTime() + future.getTimeout() < currentTime) {
						// timeout: remove from callback list, and then cancel
						removeCallback(entry.getKey());
						future.cancel();
					} 
				} catch (Exception e) {
					LoggerUtil.error(
							name + " clear timeout future Error: uri=" + url.getUri() + " requestId=" + entry.getKey(),
							e);
				}
			}
		}
	}
}

```
这个里面的callbackMap就是存储request上下文的map，（TimeoutMonitor就是清理超时的请求上下文的）发送request请求的时候会调用到下面的方法：

``` java
	public Response request(Request request) throws TransportException {
	 
	  ...
		boolean result = writeFuture.awaitUninterruptibly(timeout, TimeUnit.MILLISECONDS);

		if (result && writeFuture.isSuccess()) {
			response.addListener(new FutureListener() {
				@Override
				public void operationComplete(Future future) throws Exception {
					if (future.isSuccess() || (future.isDone() && ExceptionUtil.isBizException(future.getException()))) {
						// 成功的调用 
						nettyClient.resetErrorCount();
					} else {
						// 失败的调用 
						nettyClient.incrErrorCount();
					}
				}
			});
			// 如果正常情况，就直接返回response
			return response;
		}

		writeFuture.cancel();
		// 异常情况 将任务移除
		response = this.nettyClient.removeCallback(request.getRequestId());

		if (response != null) {
			response.cancel();
		}
	}
```

我们可以看到在正常情况下会直接返回response，异常情况下则是直接调用removeCallback(request.getRequestId())清除了callbackMap中的无用request。
接下来我们看正常情况下，motan是在哪里清理callbackMap中内容的？
这个时候我们又需要回到上面的NettyClient中去看NettyChannelHandler了。
收到服务器的返回的response的时候会触发NettyChannelHandler的handle方法，而handle方法中首先就是:
通过消息id，移除callbackMap中的对象。

``` java
NettyResponseFuture responseFuture = NettyClient.this.removeCallback(response.getRequestId());
```

最后，我们看NettyClient中的TimeoutMonitor,它所做的的事情就是定时清理callbackMap超时的request上下文,NettyClient在初始化的时候就初始化了这个timeoutMonitor

``` java
public NettyClient(URL url) {
			timeMonitorFuture = scheduledExecutor.scheduleWithFixedDelay(
				new TimeoutMonitor("timeout_monitor_" + url.getHost() + "_" + url.getPort()),
				MotanConstants.NETTY_TIMEOUT_TIMER_PERIOD, MotanConstants.NETTY_TIMEOUT_TIMER_PERIOD,
				TimeUnit.MILLISECONDS);
	}
```


接下来我们总结一下motan中callbackMap的逻辑：

- client发送request前先将request register到callbackMap中，如果发送前连接异常，则直接remove掉。
-  客户端收到response后，根据requestId找到对应的request上下文，并移除callbackMap中的对应的对象
-  对于服务端超时或者其他异常，客户端收不到消息的情况，则通过TimeoutMonitor来定时扫描callbackMap来清理。

到这里为止，我们还没讲到motan的异步通知是怎么实现的：
具体实现可以看callbackMap中的NettyResponseFuture对象，它并不是通过java中闭锁CountDownLatch实现，而是用原生的lock.wait(), lock.notifyAll();来实现的。并不算复杂。有兴趣可以直接看源代码。


## 后记

异步消息的重点就是在客户端维护一个唯一的消息id，发送前先建立一个消息id和请求request上下文的映射，然后发送request消息到服务端。等收到服务端的response消息后，通过之前建立的映射找到对应的request，并且通过异步地址通知对应request，收到返回消息。

这种要异步通知机制就涉及到`callback&CountDownLatch`一些相关的知识。

但是要实现一个靠谱的RPC框架，`一定要注意存放消息id和请求Request上下文映射的map特殊情况的内存释放问题`。而且这个map最好是线程安全的，一般rpc都会用到多线程。

最后，还有值得关注的地方：

实现一个完善的RPC框架，`能否有效的各种情况的异常透传到客户端`还是一个很重要的议题（用过公司的某个rpc，不管什么异常都是TimeoutException，联调接口时，如果出问题只能跪求别人查日志...），虽然本文没有讨论，但是有兴趣的可以好好研究一下motan在这一块的设计。
不过后续可能也会写对应的文章，好好探讨rpc异常设计和传递。