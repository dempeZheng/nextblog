---
layout: post
title:  "IM开发日记-漫谈"
date:  2016-09-06
categories: [IM开发日记]
keywords: im,netty,mqtt
---

## 写在前面

最近比较感兴趣即时通讯，自己也也开始折腾折腾IM的服务端实现，顺便也记下折腾的笔记。
当然要实现稳定功能完善的IM并非易事，整个探索实现的过程可能会持续的迭代优化，甚至可能会否定推到之前的设计。故包括IM的笔记可能也会持续的更改。

项目地址：[https://github.com/dempeZheng/ocean](https://github.com/dempeZheng/ocean)

<!-- More-->
## 通讯协议

IM协议选择原则一般是:易于拓展，方便覆盖各种业务逻辑，同时又比较`节约流量`。后一点的需求在移动端IM上尤其重要。

常见的协议有:
XMPP
SIMPLE
MQTT
私有协议

### XMPP协议
XMPP(Extensible Messaging and Presence)[4,5]协议由 IETF 制定, 是符合 RFC2778 和 RFC2779 规范, 一种以 XML 为基础的开放式即时通信协议. 它源于Jabber 协议, 主要针对准实时、扩展即时通信、出席信息、维护联系人等网络实时通信. XMPP 协议继承了XML 灵活的发展特性, 该协议成熟、强大、可靠、安全, 可扩展性强; 但存在重复转发, 网络流量较大的
问题, 并且在音频、视频方面没有非常稳定的标准, 数据包也比较大, 协议较复杂、冗余(基于 XML), 费流量、费电, 部署硬件成本较高. 用户量大了会出现瓶颈，这一点可以参考陌陌的技术选型（陌陌初期xmpp，后续更为私有协议）。
尽管如此，但是对于技术实力不够，且需要快速上线业务来说，仍然是首选。毕竟目前已经有比较成熟的基于XMPP的即时通讯实现可以参考借鉴。

### SIMPLE协议
SIMPLE(SIP for Instant Messaging and PresenceLeverage Extension)[2,3]协议是 IETF 在 SIP 协议的基础上的扩展, 是移动 IM 主流协议之一. SIMPLE 增加了SUBSCRIBE 和 NOTIFY 方法来支持交换状态信息,增加 MESSAGE 方法用于消息的发送. 通过这些方法,可以满足移动 IM 的即时消息和状态服务的需求.SIMPLE 协议有着较成熟的应用基础, 较成熟的音视频标准, 并支持各种即时消息通信. 但是它的缺点也较多, 如所需流量较大、数据包较大、存在拥塞问题、效率不高、扩展新功能比较复杂等. 

### MQTT

MQTT协议简单, 小型传输, 开销很小, 协议交换最小化, 降低网络流量, 固定报头仅为 2 字节. 其使用发布/订阅消息模式, 提供一对多的消息发布, 解除了应用程序耦合, 对负载内容屏蔽的消息传输.MQTT 是基于 TCP 上的应用层协议, 它支持基于TLS/SSL 的加密通道传输. MQTT 在进行 connect 的时候, 支持认证. 
**另外 MQTT 还提供三种不同消息发布的服务质量, 分别是:**

-  “`至多一次`”: 消息发布完全依赖底层TCP/IP网络,会发生消息丢失或重复 ,
- “`至少一次`”: 确保消息到达, 但消息重复可能会发生.
-  “`只有一次`”: 确保消息到达一次, 可用于计费系统中, 消息重复或丢失导致不正确的结果.
 
MQTT是一个物联网传输协议，它被设计用于轻量级的发布/订阅式消息传输，旨在为低带宽和不稳定的网络环境中的物联网设备提供可靠的网络服务。MQTT是专门针对物联网开发的轻量级传输协议。MQTT协议针对低带宽网络，低计算能力的设备，做了特殊的优化，使得其能适应各种物联网应用场景。
但是它并不是一个专为IM设计的协议，多用于推送。


## 私有协议

而市面上几乎所有主流IM APP都是是使用私有协议，一个被良好设计的私有协议一般有如下优点:高效，节约流量(一般使用二进制协议)，安全性高，难以破解。缺点则是在开发初期没有现有样列可以参考，对于设计者的要求比较高。


一个好的协议需要满足如下条件:高效，简洁，可读性好，节约流量，易于拓展，同时又能够匹配当前团队的技术堆栈。基于如上原则，我们可以推出: 如果团队小，团队技术在IM上积累不够可以考虑使用XMPP或者MQTT+HTTP短连接的实现。反之可以考虑自己设计和实现私有协议。


对比上述各个方案, 我们可以发现, SIMPLE 协议和 XMPP 协议, 在流量、功耗、传输方面没有考虑到移动互联网的特点, 而这些特点正是移动终端应用所必需考虑的地方. 反观 MQTT 协议, 不仅能够全面的支持即时通讯的基本需求, 也解决了在流量、功耗、传输上面的问题. 首先 MQTT 流量消耗非常的小, 它的固定长度仅为 2 字节. 其次由于协议自身非常简单,再加上简小的头部, 所以解析代价低, 功耗自然就比较小. 此外 MQTT 极易扩展, 方便用户的二次开发.综上可知, MQTT 协议更适于移动终端。
当然如果团队技术实力还不错，首选当然是针对业务场景和业务偏重方向定制私有的IM协议。反之，可以考虑扩展MQTT协议满足自己的业务场景。


**[注]** 这个系列的文章都是基于扩展mqttt协议来实现im的。毕竟mqtt是开源协议，对于开发者来说比较亲和一点。

## 通讯方式
通讯方式:设备直连(P2P)和通过服务器中转。
### 设备直连

P2P多见于局域网内聊天工具，典型的应用有:飞鸽传书。这类软件在启动后一般做两件事情

进行UDP广播:发送自己信息和接受同局域网内其他端信息
开启TCP监听:等待其他端进行连接
这种方式在有种种限制和不便:一方面它只适合在线的点对点消息传输，对离线，群组等业务支持不够。另一方面由于 NAT 的存在，使得不同局域网内机器互联难度大大上升，在某些网络类型(对称NAT)下无法建立连接。

### 服务器中转
消息请求都经过服务器中转使得服务器掌握了消息的控制权，对消息的控制能力强，故它能支持很多设备直连支持不好的业务，例如离线消息，群组，聊天室，垃圾消息过滤。这也是目前大部分互联网选用服务器中转的原因。

## 架构
采用分层的设计，可以初步分为链接层，业务层，数据访问层。

![](/images/im/chat-info.png)

- ChatServer：连接层。剥离即时通讯的核心逻辑，即维护连接和消息投递(后续可能进一步拆分，将Connetor&MessageService拆分成单独服务)，保证单一职责，便于实现，扩展及维护。通过TCP长连接保持并维护连接，发送投递消息。不关心具体的业务逻辑实现，业务逻辑实现直接交由LogicService层处理。
- HttpApiServer：抛却核心业务通讯的业务，其他的所有不依赖通讯的业务都可以用过Rest接口提供服务。(Http协议作为广泛使用的协议，有非常成熟的方案，利于开发测试联调)
- LogicService：业务逻辑层，基于MotanRPC暴露处理的基础Service能力，作为上层ChatServer&HttpApiServer提供基础能力支持。无状态的服务，可无限水平扩展。
- SessionStore：提供存储的能力，主要是消息的存储

## 技术选型
- ChatServer：netty+mqtt
- HttpApiServer：SpringBoot, MotanRPC Client
- LogicService:MotanRPC Server

## 设计要点


## 后记

## 延伸阅读：

[陌陌CTO分享-高可用即时通讯架构(非常值得一听)](http://www.infoq.com/cn/presentations/high-availability-instant-communication-architecture)

----
先写到这里后续再补充。

