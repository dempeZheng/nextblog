---
layout: post
title:  "记一次netty内存溢出的问题(上)"
date:  2017-03-28
categories: [netty]
keywords: netty,ByteBuf,pool,outOfMemory
---

最近参与一个基于netty的rpc框架的开发，本来觉得只要线程模型和序列化没什么问题，性能应该不会差。然而，真正压测的时候竟然发生内存溢出。

起初，仅仅2k/s的测一下正常情况下，系统的内存占用和垃圾回收情况，跑了一天发现一切正常。本来以为剩下的只要测测系统的处理峰值，做做小的优化就没啥事了。
第二天加大压测力度，一会cpu就飙升上去了，压测停止cpu降下来，发现内存也不降的。

通过如下命令

```sh
sudo jmap -F -heap pid
```

```java
Heap Configuration:
   MinHeapFreeRatio = 40
   MaxHeapFreeRatio = 70
   MaxHeapSize      = 1073741824 (1024.0MB)
   NewSize          = 536870912 (512.0MB)
   MaxNewSize       = 536870912 (512.0MB)
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 201326592 (192.0MB)
   MaxPermSize      = 201326592 (192.0MB)
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 483196928 (460.8125MB)
   used     = 120514736 (114.93180847167969MB)
   free     = 362682192 (345.8806915283203MB)
   24.941122142233485% used
Eden Space:
   capacity = 429522944 (409.625MB)
   used     = 120514736 (114.93180847167969MB)
   free     = 309008208 (294.6931915283203MB)
   28.057811039775327% used
From Space:
   capacity = 53673984 (51.1875MB)
   used     = 0 (0.0MB)
   free     = 53673984 (51.1875MB)
   0.0% used
To Space:
   capacity = 53673984 (51.1875MB)
   used     = 0 (0.0MB)
   free     = 53673984 (51.1875MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 536870912 (512.0MB)
   used     = 532637504 (507.96270751953125MB)
   free     = 4233408 (4.03729248046875MB)
   99.21146631240845% used
Perm Generation:
   capacity = 201326592 (192.0MB)
   used     = 21220648 (20.237586975097656MB)
   free     = 180105944 (171.76241302490234MB)
   10.540409882863363% used
```

发现concurrent mark-sweep generation（老生代）已经满了，而且基本不会回收。

然后，继续压测，不出几分钟就内存溢出了。（有的时候如果不是正儿八经的做做压测或者经历高并发场景还真的很难发现问题。）


对于这种好处理内存溢出的机会，当然不能放过。

## 1.首先dump出bin文件：

```
sudo jmap -F -dump:live,format=b,file=server_dump.bin pid
```

>`-F`是强制的意思，在进程启动用户和jmap脚本执行用户不是同一用户的时候需要`-F`
>`live`参数会强制full gc

## 2.dump出bin文件后直接上工具分析：
这里使用的是eclipse旗下的工具，[Memory Analyzer (MAT)](http://www.eclipse.org/mat/downloads.php) ，将bin文件用mat打开分析，如下图：
![Alt text](/images/9e7aeccf-539e-4986-8fb4-a8d4753e1557.png)
这个图已经能很清晰的说明问题了，netty自己实现的FastThreadLocalThread占据了88%的内存。
![Alt text](/images/7679748d-5928-4666-9e57-59ceeadb9058.png)


dump bin文件是一个比较耗方案，但是往往能很清晰直观的发现问题。很多线上问题不一定等得及dump bin文件分析，如果需要轻量一点的解决方式，可以考虑直接使jmap -histo查看对象占用内存的情况。（当分配的内存过大时，dump bin文件一方面很耗时，一方面很难分析，大部分工具包括MAT都是java实现的分析工具，加载超过3g的bin文件可能直接打不开）

```
sudo jmap -F -histo pid
```
[注]-F和live不能同时使用，也就是不可以`sudo jmap -F -histo:live pid`,所以最好能用进程启动的用户来执行如下命令：`jmap -histo:live pid` （如果不加live强制full gc会看到很多unreachable对象，这些对象暂时还没被垃圾回收，很容易干扰到分析）

## 3.查找FastThreadLocalThread内存溢出的原因

netty在Recycler中会用到这个FastThreadLocalThread，用来缓存对象。
简单来说Recycler提供了一个对象池，目的是为了减少gc的频率，提升性能。
对于这个Recycler引起的内存泄漏，网上一搜一大把。但是大家产生内存泄漏的原因基本是由于内存的 allocate 和 release 不再同一个线程，造成的内存泄漏。

仔细的查看了项目的代码，发现内存的分配释放都是在decoder里面处理的，也就是netty的IO线程。既然这样压测的时候为什么会内存溢出呢？

这个就绕不开Recycler的实现了。还是得好好看看Recycler的机制。

### 3.1Recycler相关

{% note default %}
Recycler是netty轻量内存池技术的核心实现，Recycler是一个抽象类，向外部提供了两个公共方法get和recycle分别用于从对象池中获取对象和回收对象
Recycler内部主要包含三个核心组件，各个组件负责对象池实现的具体部分，Recycler向外部提供统一的对象创建和回收接口：

 - **Handle**

 - **WeakOrderQueue**

 - **Stack**

 **各组件的功能如下**

 **Handle**

 Recycler在内部类中给出了Handle的一个默认实现：DefaultHandle，Handle主要提供一个recycle接口，用于提供对象回收的具体实现，每个Handle关联一个value字段，用于存放具体的池化对象，记住，在对象池中，所有的池化对象都被这个Handle包装，Handle是对象池管理的基本单位。另外Handle指向这对应的Stack，对象存储也就是Handle存储的具体地方由Stack维护和管理。

 **Stack**

 Stack具体维护着对象池数据，向Recycler提供push和pop两个主要访问接口，pop用于从内部弹出一个可被重复使用的对象，push用于回收以后可以重复使用的对象。

 **WeakOrderQueue**

 WeakOrderQueue的功能可以由两个接口体现，add和transfer。add用于将handler（对象池管理的基本单位）放入队列，transfer用于向stack输入可以被重复使用的对象。我们可以把WeakOrderQueue看做一个对象仓库，stack内只维护一个Handle数组用于直接向Recycler提供服务，当从这个数组中拿不到对象的时候则会寻找对应WeakOrderQueue并调用其transfer方法向stack供给对象。
{% endnote %}

首先第一点，Recylcler机制和线程绑定，一个线程会有一个ThreadLocal，**对应的对象的申请和回收如果不在同一个线程一定会内存泄漏。**
第二点:如果对象申请的速度大于对象回收的速度，那么对象一定会在Recycler的queue中堆积。那么，我们来看看queue的size，如果queue上限过大，那么大量对象堆积，肯定会造成内存溢出。
通过阅读Recycler的代码我们看到，默认情况，这个上限是32k。那么我们基本可通过线程的数量来判断出实例堆积的上限基本就是线程数量×32k。而我们项目中用到这个Recycler的就是内存的分配是释放，这个都是在IO的线程里面处理的，而IO的线程数就是通过核数×2来获取的，线上压测的核数是24核，故IO线程数量为48。 48×32k已经不是一个小数目了。至于占用内存的上限要通过压测得出。

```java
private static final int DEFAULT_INITIAL_MAX_CAPACITY = 32768; // Use 32k instances as default max capacity.
 int maxCapacity = SystemPropertyUtil.getInt("io.netty.recycler.maxCapacity", DEFAULT_INITIAL_MAX_CAPACITY);
```
当然，还有一个问题需要确认，内存的申请和回收速度是否同步，如果是同步的那么这个queue里面的对象就不会膨胀。
[netty内存分配Recycler容量膨胀问题跟踪](http://code.zhizus.com/2017-03-31-netty-buffer-recycler.html)

## 4.解决方案
严格来说，这种情况不能算内存泄漏，如果内存分配再大点，IO线程数量设置少些，也是没什么问题的。但是netty也给我们提供了一些解决方案。

**第一种：直接禁用掉Recycler的功能，功过设置`-Dio.netty.recycler.maxCapacity.default=0`**
当然如果禁用 Recycler可惜的话，可以将maxCapcity设置一个合适的值。

>Netty的另一个得意设计是对象可以在线程内无锁的被回收重用。但有些场景里，某线程只创建对象，要到另一个线程里释放，一不小心，你就会发现应用缓缓变慢，heap dump时看到好多RecyleHandler对象。所以这块的设计其实在4.0.x的各个版本里变动了无数遍，貌似4.0.40版才终于在我的测试里不再泄漏了。

>但有时觉得这么辛苦的重用一个对象（而不是ByteBuffer内存本身），不如干脆禁止掉这个功能，所以4.0.0.33里，我会在启动参数里加入 `-Dio.netty.recycler.maxCapacity.default=0`。无语的是，也几乎从这个版本开始，才能通过设为0禁止它。


**第二种：异步系统限流**

netty提供了通过设置高低水位的方式进行限流，

{% note info %}
onfigure high and low write watermarks
Set sane WRITE_BUFFER_HIGH_WATER_MARK and WRITE_BUFFER_LOW_WATER_MARK
{% endnote %}


**Server**

```java
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 32 * 1024);
bootstrap.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 8 * 1024);
```

**Client**

```java
Bootstrap bootstrap = new Bootstrap();
bootstrap.option(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 32 * 1024);
bootstrap.option(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 8 * 1024);
```

## 5.优化
压测过程中发现内存老生代将要耗尽的时候会频繁的触发cms gc，而短时间内可回收的对象又相对将少，效率低下。而gc反过来又会影响系统的吞吐，造成更多的对象的堆积。
所以设置合理的堆栈大小很有必要。
不同的环境会有不同的最佳值，那么如何调整到一个合适值呢？
1.打印出gc日志，根据日志来调整

```
XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/log/java/gc.log 
```

2.通过jdk自带工具，jconsole等来查看gc的情况
暴露jmx端口，给jdk自带工具连接

```
-Dcom.sun.management.jmxremote=true -Dcom.sun.management.jmxremote.port=15214 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=yourip 
```

## 6.总结

1.netty 4 内存分配和释放一定要在同一个线程，如果自己使用netty的bufferpool更是要谨慎。
2.Recycler是个好东西，对于大量使用的重量级对象提升性能是很显著的，重要的是还减少了gc频率。当然也是有代价的，你得额外花费一些内存。
3.合理运用工具，MAT分析堆栈文件真的很靠谱，
4.对于内存泄漏，cpu 100%这种问题还是得多用实际问题练练手，不然只能停留在理论阶段。
5.netty确实是一个优秀的框架

### 延伸阅读

[Netty 踩坑记](https://caorong.github.io/2016/08/27/netty-hole/)
[Netty精粹之轻量级内存池技术实现原理与应用](https://my.oschina.net/andylucc/blog/614589)