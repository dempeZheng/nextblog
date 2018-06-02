---
layout: post
title:  "Motan源码解读-serialize"
date:  2016-09-22
categories: [Motan源码解读]
keywords: motan,源码,serialization,序列化
---

## 简介

>**序列化：** 将数据结构或对象转换成二进制串的过程。
> **反序列化：**将在序列化过程中所生成的二进制串转换成数据结构或者对象的过程。


motan 将RPC`请求中的参数、结果等对象进行序列化与反序列化`，即进行对象与字节流的互相转换；默认使用对java更友好的hessian2进行序列化。（注意序列化的是请求的参数，和返回的结果，其他的地方不需要序列化）

<!--More-->

一个rpc框架的性能主要取决于：线程模型，IO模型，序列化三个方面。（对于有gc机制的jvm来说，零拷贝也很关键）
所以一个性能优异的序列化对于rpc是非常重要的。（这里的序列化和反序列化仅仅涉及到请求中的参数、结果等对象，协议其他的信息的序列化见后续的codec章节）

motan会对client端发送的request的请求参数对象进行序列化，也会对服务端下发给response对象进行序列化，如下图所示：

![](/code/images//motan/motan-register-server-client.jpg)

## motan目前支持两种序列化：

- hession2

```
<motan:protocol serialization="hessian2" />		
```

- fastjson

```
<motan:protocol serialization="fastjson" />		
```


## 各种序列化性能对比

![](/images/motan/serialize.png)

我们可以发现fastjson的序列化性能和thrift和protobuf差异已经不太大了，明显好过hession（当然上图比较的不知道hession的版本，可能不准确）
dubbo是有针对hession单独做优化的，但是motan默认使用hession，且没做优化。
### motan序列化的实现
``` java
// 这个可以可以扩展的，也就是你可以扩展motan用你想要的序列化方式
@Spi(scope=Scope.PROTOTYPE)
public interface Serialization {
	// 将对象序列话称字节数组
	byte[] serialize(Object obj) throws IOException;
	// 将字节数组反序列化成对象
	<T> T deserialize(byte[] bytes, Class<T> clz) throws IOException;
}
```
具体的实现要看hession2和fastjson了。

motan序列化&反序列化仅仅包含了请求参数和返回对象，其他对象的编解码，我们且看下一篇文章。


----
后记：

1.fastjson确实够快，使用也挺方便的。

2.如果我们需要扩展motan实现thrift或者protobuf还需要hession或者fastjson来辅助序列化吗？

首先hession可以排除，使用hession跨语言调用就比较难实现了。
那么需要辅助的fastjson来序列化吗？没必要。
