---
layout: post
title:  "处理 InterruptedException"
date:  2018-08-31 22:39:04
type: java
categories: [java]
keywords: jvm,interrupt,InterruptedException
---
## 由一个sonar的case说起
{% fi /images/1535626831932.png, alt %}
sonar检测到InterruptedException处理的bug，建议catch Exception的逻辑里面通过

```java
   Thread.currentThread().interrupt();
```

对于 InterruptedException，一种常见的处理方式是 “生吞（swallow）” 它 —— 捕捉它，然后什么也不做（或者记录下它，不过这也好不到哪去）不幸的是，这种方法忽略了这样一个事实：这期间可能发生中断，而中断可能导致应用程序丧失及时取消活动或关闭的能力。

### 错误示范：
```java
  Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    //①吞掉异常
                }
                try {
                    TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        t.start();
        t.interrupt();//②中断异常
```
比如代码①处，我们吞掉InterruptedException ，当代码②中断线程`t`的时候，发现线程`t`就无法中断了。
### 正确的示范：

```java
  Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
//                    e.printStackTrace();
                    //调用interrupt将异常传递下去
                    Thread.currentThread().interrupt();
                }
                System.out.println("handle biz 1");
                try {
                    TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
//                    e.printStackTrace();
                    Thread.currentThread().interrupt();
                }
                System.out.println("handle biz 2");

            }
        });
        t.start();
        t.interrupt();

```
执行结果如下：

```
handle biz 1
handle biz 2
```

从结果上面来看，Thread.currentThread().interrupt();方法并没有中断当前线程，影响到正常逻辑，而是在调用Thread.sleep方法的时候才会触发中断。  
那么，Java里一个线程调用了Thread.interrupt()到底意味着什么？

## Java里一个线程调用了Thread.interrupt()到底意味着什么

>作者：Intopass
链接：https://www.zhihu.com/question/41048032/answer/89431513
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

>首先，一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。所以，Thread.stop, Thread.suspend, Thread.resume 都已经被废弃了。而 Thread.interrupt 的作用其实也不是中断线程，而是「通知线程应该中断了」，具体到底中断还是继续运行，应该由被通知的线程自己处理。

>具体来说，当对一个线程，调用 interrupt() 时，
- ① 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
- ② 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。

>interrupt() 并不能真正的中断线程，需要被调用的线程自己进行配合才行。也就是说，一个线程如果有被中断的需求，那么就可以这样做。
- ① 在正常运行任务时，经常检查本线程的中断标志位，如果被设置了中断标志就自行停止线程。
- ② 在调用阻塞方法时正确处理InterruptedException异常。（例如，catch异常后就结束线程。）Thread thread = new Thread(() -> {
 
```java
   while (!Thread.interrupted()) {
        // do more work.
    }
});
thread.start();
```

// 一段时间以后

```java
thread.interrupt();
```

>具体到你的问题，Thread.interrupted()清除标志位是为了下次继续检测标志位。如果一个线程被设置中断标志后，选择结束线程那么自然不存在下次的问题，而如果一个线程被设置中断标识后，进行了一些处理后选择继续进行任务，而且这个任务也是需要被中断的，那么当然需要清除标志位了。

## 如何处理InterruptedException
>1)Propagate the InterruptedException
Declare your method to throw the checked InterruptedException so that your caller has to deal with it.

>2)Restore the Interrupt
Sometimes you cannot throw InterruptedException. In these cases you should catch the InterruptedException and restore the interrupt status by calling the interrupt() method on the currentThread so the code higher up the call stack can see that an interrupt was issued.

简单来说处理InterruptedException有两种选择，
- 第一种：做一些清理和记录工作后，直接抛出，丢给上层业务处理
- 第二种：调用 `Thread.currentThread().interrupt();`将中断传递出去

【注意】一定不要生吞InterruptedException异常！！(生吞InterruptedException异常，可能导致当前线程无法及时的中断，进而导致服务无法正常关闭)
>有时候抛出 InterruptedException 并不合适，例如当由 Runnable 定义的任务调用一个可中断的方法时，就是如此。在这种情况下，不能重新抛出 InterruptedException，但是您也不想什么都不做。当一个阻塞方法检测到中断并抛出 InterruptedException 时，它清除中断状态。如果捕捉到 InterruptedException 但是不能重新抛出它，那么应该保留中断发生的证据，以便调用栈中更高层的代码能知道中断，并对中断作出响应。该任务可以通过调用 interrupt() 以 “重新中断” 当前线程来完成，如清单 3 所示。至少，每当捕捉到 InterruptedException 并且不重新抛出它时，就在返回之前重新中断当前线程。


## 阻塞方法

当一个方法抛出 InterruptedException 时，它不仅告诉您它可以抛出一个特定的检查异常，而且还告诉您其他一些事情。例如，它告诉您它是一个阻塞（blocking）方法，如果您响应得当的话，它将尝试消除阻塞并尽早返回。
阻塞方法不同于一般的要运行较长时间的方法。一般方法的完成只取决于它所要做的事情，以及是否有足够多可用的计算资源（CPU 周期和内存）。而阻塞方法的完成还取决于一些外部的事件，例如计时器到期，I/O 完成，或者另一个线程的动作（释放一个锁，设置一个标志，或者将一个任务放在一个工作队列中）。一般方法在它们的工作做完后即可结束，而阻塞方法较难于预测，因为它们取决于外部事件。阻塞方法可能影响响应能力，因为难于预测它们何时会结束。
阻塞方法可能因为等不到所等的事件而无法终止，因此令阻塞方法可取消 就非常有用（如果长时间运行的非阻塞方法是可取消的，那么通常也非常有用）。可取消操作是指能从外部使之在正常完成之前终止的操作。由 Thread 提供并受 Thread.sleep() 和 Object.wait() 支持的中断机制就是一种取消机制；它允许一个线程请求另一个线程停止它正在做的事情。当一个方法抛出 InterruptedException 时，它是在告诉您，如果执行该方法的线程被中断，它将尝试停止它正在做的事情而提前返回，并通过抛出 InterruptedException 表明它提前返回。 `行为良好的阻塞库方法应该能对中断作出响应并抛出 InterruptedException，以便能够用于可取消活动中，而不至于影响响应`。

>并非所有的阻塞方法都抛出 InterruptedException。输入和输出流类会阻塞等待 I/O 完成，但是它们不抛出 InterruptedException，而且在被中断的情况下也不会提前返回。然而，对于套接字 I/O，如果一个线程关闭套接字，则那个套接字上的阻塞 I/O 操作将提前结束，并抛出一个 SocketException。java.nio 中的非阻塞 I/O 类也不支持可中断 I/O，但是同样可以通过关闭通道或者请求 Selector 上的唤醒来取消阻塞操作。类似地，尝试获取一个内部锁的操作（进入一个 synchronized 块）是不能被中断的，但是 ReentrantLock 支持可中断的获取模式。


## 不可取消的任务

有些任务拒绝被中断，这使得它们是不可取消的。但是，即使是不可取消的任务也应该尝试保留中断状态，以防在不可取消的任务结束之后，调用栈上更高层的代码需要对中断进行处理。清单 6 展示了一个方法，该方法等待一个阻塞队列，直到队列中出现一个可用项目，而不管它是否被中断。为了方便他人，它在结束后在一个 finally 块中恢复中断状态，以免剥夺中断请求的调用者的权利。（它不能在更早的时候恢复中断状态，因为那将导致无限循环 —— BlockingQueue.take() 将在入口处立即轮询中断状态，并且，如果发现中断状态集，就会抛出 InterruptedException。）  
清单 6. 在返回前恢复中断状态的不可取消任务
```java
public Task getNextTask(BlockingQueue<Task> queue) {
    boolean interrupted = false;
    try {
        while (true) {
            try {
                return queue.take();
            } catch (InterruptedException e) {
                interrupted = true;
                // fall through and retry
            }
        }
    } finally {
        if (interrupted)
            Thread.currentThread().interrupt();
    }
}
```

## 结束语
您可以用 Java 平台提供的协作中断机制来构造灵活的取消策略。各活动可以自行决定它们是可取消的还是不可取消的，以及如何对中断作出响应，如果立即返回会危害应用程序完整性的话，它们还可以推迟中断。即使您想在代码中完全忽略中断，也应该确保在捕捉到 InterruptedException 但是没有重新抛出它的情况下，恢复中断状态，以免调用它的代码无法获知中断的发生。  
最后，强调一遍，**一定不要生吞InterruptedException异常！！！**

## 参考文档
https://www.zhihu.com/question/41048032  
https://www.ibm.com/developerworks/cn/java/j-jtp05236.html

