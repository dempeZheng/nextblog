---
layout: post
title:  "Motan源码解读-codec"
date:  2016-09-23
categories: [Motan源码解读]
keywords: motan,源码,code,编解码
---


motan的codec负责rpc框架通讯协议的编解码。codec模块是每个rpc必备的模块，一个设计良好的codec关系到rpc框架编解码的性能和对不同协议的扩展。

dubbo支持http,thrift,dubbo,rmi,webservice,redis等等协议，而motan目前仅支持默认motan的协议，所以很多时候我们可能会扩展motan支持自己的协议，那么我们就得从codec入手了（有的时候甚至还会涉及到protocol）。

codec的类图如下：

![Alt text](/images/motan/codec.png)

<!--More-->

下面 codec interface的定义：

``` java
// 可基于spi扩展
@Spi(scope=Scope.PROTOTYPE)
public interface Codec {
	// channel为motan定义的channel，非netty的channel
	byte[] encode(Channel channel, Object message) throws IOException;

	/**
	 * 
	 * @param channel
	 * @param remoteIp 用来在server端decode request时能获取到client的ip。
	 * @param buffer
	 * @return
	 * @throws IOException
	 */
	// 不明白为什么要把remoteIp定义到decode接口里面，channel里面已经包含ip的信息了
	Object decode(Channel channel, String remoteIp, byte[] buffer) throws IOException;

}
```
AbstractCodec里面实现了创建输入输出流的方法，用来将协议序列化成字节数组或将字节数组反序列化成协议的。（这种方式的性能还没压测，不知道好坏）

如果我们要实现自己的协议，可以仅仅实现codec的方法。

实现AbstractCodec的有三个类

- DefaultRpcCodec

> 默认的协议编解码实现

- MockDefaultRpcCodec

> 用于测试的

- CompresssRpcCodec

>经过压缩的协议编解码实现

为了不引入复杂性，我们先看DefaultRpcCodec的实现。
要知道motan如何对协议编解码，首先我们的非常清楚motan的协议格式。

CompresssRpcCodec这个类的注释中已经有协议各个模块的实现，但是还不够清晰简明，我们先整理一个图。

## motan协议格式

- header


<table style="border-collapse:collapse;border-spacing:0;border-color:#999"><tr><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4;text-align:center" colspan="7">header</th></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">0-15</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">16-23</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" colspan="3">24-31</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">32-95</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">96-127</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">magic</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center">version</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" colspan="3">extend flag</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">request id</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">body content length</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center" rowspan="2">魔数</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center" rowspan="2">协议版本</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">24-28</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">29-30</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">31</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" rowspan="2"><br>消息id<br></td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top" rowspan="2"><br>body包长</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">保留</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">event( 可支持4种event，<br>如normal, exception等)</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA;text-align:center;vertical-align:top">0 is request , 1 is response</td></tr></table>


- request body

```  java
    /* 
     * <pre>
	 * 
	 * 	 body:
	 * 
	 * 	 byte[] data :  
	 * 
	 * 			serialize(interface_name, method_name, method_param_desc, method_param_value, attachments_size, attachments_value) 
	 * 
	 *   method_param_desc:  for_each (string.append(method_param_interface_name))
	 * 
	 *   method_param_value: for_each (method_param_name, method_param_value)
	 * 
	 * 	 attachments_value:  for_each (attachment_name, attachment_value)
	 * 
	 * </pre>
     * 
     * @param request
     * @return
     * @throws IOException
     */
    private byte[] encodeRequest(Channel channel, Request request) throws IOException ;
```

encodeRequest方法的注释很清晰的描述了request 的body部分的序列化，对jvm系列的来说非常友好，基本上把调用相关所有能用到的信息都序列化了。

我们可以把请求任何一个请求参数都反序列化java对象。所以motan默认的协议是可以支持方法内参数为java任意对象。
但是，`对于其他语言就不太友好了`。其他语言不太容易序列化java的对象。
如果希望更好的支持其他的语言，这个请求参数的类型序列化方式可能有所取舍了。

- response body

> `serialize (result) or serialize (exception)`：motan的response的message对象是通过hession2或者fastjson序列化生成的字节数组，所以这个地方就是一个字节数组（如果需要了解序列化的部分，可以看上篇博客）


## motan协议打包

协议的打包简单说一下header打包的。

``` java
  private byte[] encode(byte[] body, byte flag, long requestId) throws IOException {
        // 协议1的header占16个字节，先创建一个长度为16的字节数组
        byte[] header = new byte[RpcProtocolVersion.VERSION_1.getHeaderLength()];
        int offset = 0;

        // 0 - 15 bit : magic
        // 这个方法实际上是把short类型的magic转成两个字节，打包到header
        ByteUtil.short2bytes(MAGIC, header, offset);
        offset += 2;

        // 16 - 23 bit : version
        // 打包version
        header[offset++] = RpcProtocolVersion.VERSION_1.getVersion();

        // 24 - 31 bit : extend flag
        // 打包flag
        header[offset++] = flag;

        // 32 - 95 bit : requestId
        // 打包requestId
        ByteUtil.long2bytes(requestId, header, offset);
        offset += 8;

        // 96 - 127 bit : body content length
        // 打包body包长
        ByteUtil.int2bytes(body.length, header, offset);

        byte[] data = new byte[header.length + body.length];

        System.arraycopy(header, 0, data, 0, header.length);
        System.arraycopy(body, 0, data, header.length, body.length);

        return data;
    }
```


到这里motan默认的协议基本分析完，但是还没说到CompresssRpcCodec 压缩的编解码，（这个压缩的类跟协议编解码写到一起，分离出来会不会更好[如果我们能确定哪些地方需要压缩]）

## motan协议的压缩

数据压缩也是一个完善的rpc框架必备的功能之一，良好的压缩可以减少网络传输数据大小，节省流量，提升响应速度。

### 压缩算法比较

以下是Google几年前发布的一组测试数据（数据有些老了，有人近期做过测试的话希望能共享出来）：
<table style="border-collapse:collapse;border-spacing:0;border-color:#999"><tr><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4">Algorithm</th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4">% remaining</th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4">Encoding</th><th style="font-family:Arial, sans-serif;font-size:14px;font-weight:normal;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#fff;background-color:#26ADE4">Decoding</th></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">GZIP</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">13.4%</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">21 MB/s</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">118 MB/s</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">LZO</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">20.5%</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">135 MB/s</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">410 MB/s</td></tr><tr><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">Zippy/Snappy</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">22.2%</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">172 MB/s</td><td style="font-family:Arial, sans-serif;font-size:14px;padding:10px 20px;border-style:solid;border-width:1px;overflow:hidden;word-break:normal;border-color:#999;color:#444;background-color:#F7FDFA">409 MB/s</td></tr></table>



注：来自《HBase: The Definitive Guide》
其中：

1）GZIP的压缩率最高，但是其实CPU密集型的，对CPU的消耗比其他算法要多，压缩和解压速度也慢；

2）LZO的压缩率居中，比GZIP要低一些，但是压缩和解压速度明显要比GZIP快很多，其中解压速度快的更多；

3）Zippy/Snappy的压缩率最低，而压缩和解压速度要稍微比LZO要快一些。

## motan中的协议压缩

motan默认支持的压缩是gzip，这块代码比较杂，不支持扩展，如果要实现自己的压缩方式，得重新实现codec部分（继承defaultRpcCodec，重写部分方法也可以）。

``` java
// 对rpc body进行压缩。
    public byte[] compress(byte[] org, boolean useGzip, int minGzSize) throws IOException {
        if (useGzip && org.length > minGzSize) {
            ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
            GZIPOutputStream gos = new GZIPOutputStream(outputStream);
            gos.write(org);
            gos.finish();
            gos.flush();
            gos.close();
            byte[] ret = outputStream.toByteArray();
            return ret;
        } else {
            return org;
        }
    }
```

就不一一分析motan这块代码，简单总结一下motan compress：

1）motan压缩的仅仅是rpc 的body部分。header没有压缩的必要。

2）心跳包没有body，不压缩。

3） mingzSize("mingzSize", 1000), // 进行gz压缩的最小数据大小。超过此阈值才进行gz压缩

4）usegz("usegz", false), // 是否开启gzip压缩，这个可以配置

5）是否开启压缩usegz这个是通过配置来约定的，并未发现有打包到协议里面。如果要解包，首先要知道这个协议是否压缩过。（个人感觉这里不太好，对于motan自己client调用将配置文件拷贝过去即可，但是对于其他的语言就麻烦了，不太友好）


## 方法签名

motan在打包method的时候会对根据方法的interfaceName，methodName，parameterDes拼接起来，md5生成一个方法签名，如果方法签名生成成功，则将方法签名的标志和方法签名打包，否则打包完整的方法信息

如果客户端使用了方法签名，服务端怎么这个方法签名是哪一个方法呢？（首先生成签名的方法是md5的不可逆的，也就是说不可以反解。）

motan的做法是每个服务在export前都会对对应的方法生成签名，并保存签名和方法的映射。

方法签名的好处：最大的好处就是省了流量。也可以带来一点点性能提升。


---
## 后记：

motan的设计基本没考虑跨语言的调用。有些可惜。

个人并不是很喜欢motan codec的代码，故不会在这里耗太多时间。
以后遇到好的codec的代码，再来分享。