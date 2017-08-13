---
layout: post
title:  "记一次OutOfMemoryError: unable to create new native thread"
date:  2017-07-31
categories: [实战]
keywords: OutOfMemoryError,thread
---

## 前言
今天在测试环境跑测试用例的时候发现一大片测试用例无法通过，查看日志发现`OutOfMemoryError: unable to create new native thread`。从日志信息来看，很明显，内存不足，无法创建本地线程。

遇到这个异常还是很意外的，测试用例大概就1.5k个，不至于跑个用例就来这么个异常，测试环境当时jvm的内存是4g，感觉不至于出现这个问题。（当然觉得意外的原因还是对jvm线程数量限制的理解有误）

要解决问题，首先，还是的了解jvm的线程数量到底受限哪些因素。

## jvm线程数量限制


### jvm线程受限于操作系统可用内存还是JVM的内存

1.**Java线程的实现是基于底层系统的线程机制来实现的**,程序中开的线程并不全部取决于JVM虚拟机栈，而是取决于CPU，操作系统，其他进程，Java的版本。JVM的线程与计算机本身性能相关。
这个很重要，不了解这一点，很容易像我之前一样误解为jvm线程个数首先于jvm的内存大小。

### JVM线程的栈在64位Linux操作系统上的默认大小是多少
>不显式设置-Xss或-`XX:ThreadStackSize`时，在Linux x64上`ThreadStackSize`的默认值就是1024KB，给Java线程创建栈会用这个参数指定的大小。这是前一块代码的意思。如果把-`Xss`或者`-XX:ThreadStackSize`设为0，就是使用“系统默认值”。
>而在Linux x64上HotSpot VM给Java栈定义的“系统默认”大小也是1MB。所以这个条件下普通Java线程的默认栈大小怎样都是1MB。

>至于操作系统栈大小（ulimit -s）：这个配置只影响进程的初始线程；后续用pthread_create创建的线程都可以指定栈大小。HotSpot VM为了能精确控制Java线程的栈大小，特意不使用进程的初始线程（primordial thread）作为Java线程。

作者：RednaxelaFX
链接：https://www.zhihu.com/question/27844575/answer/38370294
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

查看-Xss的大小：
```
java -XX:+PrintFlagsFinal -version | grep -i 'stack'
```

### 不断加大可用内存，线程数量也会不断增长？
这个当然不是，上面我特意加粗了不考虑系统本省限制的情况，所以说线程数量还与系统限制有关。主要跟一下几个参数有关(Linux下的)：
- `/proc/sys/kernel/pid_max` 增大，线程数量增大，`pid_max`有最高值，超过之后不再改变，而且32，64位也不一样
- `/proc/sys/kernel/thread-max` 系统可以生成最大线程数量
- `max_user_process（ulimit -u）centos`系统上才有，没有具体研究/proc/sys/vm/max_map_count 增大，数量增多

一般来说线程数量上限在4k~5k左右，实际开发中为了避免线程无限膨胀，我们要合理的使用线程池。


## 总结

**(MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize) = Number of threads**
`MaxProcessMemory` 指的是一个进程的最大内存
`JVMMemory`         JVM内存
`ReservedOsMemory`  保留的操作系统内存
`ThreadStackSize`      线程栈的大小

### 推论
- 给JVM的堆内存分配的越大，系统可创建的线程数量就越少
- 当-Xss的的值越小，可生成的线程数量就越多

## 解决办法
了解了首先因素之后，如果系统报`OutOfMemoryError: unable to create new native thread`这个错误，我们就容易对症下药，可以从两个方向着手。
1.调小线程栈为`-Xss256k`(默认为1M)，当然调用栈太深，不够用的情况可以酌情在调大些。
2.增加机器系统内存


## 参考
https://www.zhihu.com/question/27844575/answer/38370294
http//blog.csdn.net/feng27156/article/details/19333575