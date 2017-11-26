---
title: xmpp协议解析与使用
date: 2017-01-07 22:58:37
tags: JAVA
---
XMPP 是基于 XML 的协议，用于即时消息( IM )以及在线现场探测。最初, XMPP 作为一个框架开发, 目标是支持企业环境内的即时消息传递和联机状态应用程序 。当时的即时消息传递网络是私有的,不适合企业使用

XMPP 前身是 Jabber ( 1998 年) ,是一个开源组织定义的网络即时通信协议XMPP 是一个分散型通信网络 ,这意味着,只要网络基础设施允许,任何XMPP 用户都可以向其他任何 XMPP 用户传递消息。多个 XMPP 服务器也可以通过一个专门的“服务器 - 服务器 "协议相互通信,提供了创建分散型社交网络和协作框架的可能性。
![pinpoint](http://7xkrul.com1.z1.glb.clouddn.com/XMPP%E6%9E%B6%E6%9E%84.png "图1 xmpp通用框架")
<!-- more -->
注意，分属于不同server的client之间要通信的话，中间不能再经过其他server，这2个server必须直接通信。对于XMPP来说，server不能象email server那样，中间可以经过若干个server才能把邮件发送到目的地
## XMPP协议优缺点
- 优点 
   - 开放 
   - 标准( XMPP 的技术规格已被定义在 RFC 3920 及 RFC 3921 ) 
   - 证实可用 
   - 分散 
   - 安全 
   - 可扩展 
- 缺点 
   - 数据负载过重 
   - 没有二进制传输

## 为什么选择XMPP
>1.通信原语  Message Stanza、Presence Stanza和IQ Stanza(IQ节)
>
>2.XMPP协议引入了XML Stream(XML流)和XML Stanza(XML节)
>
>

## 伸缩架构
> 负载均衡：是通过负载策略分发客户端请求，后端的连接管理器退出服务后，请求不在分发给本台连接管理器。

> 连接管理器：是保存客户端连接，实现系统用户并发上的可伸缩，连接服务器不包含业务逻辑代码它的功能只是负载保持客户连接和转发客户请求。

>业务服务器：业务逻辑都实现在业务服务器包括用户验证模块、用户会话模块、在线状态模块、路由模块、文本消息模块等。

>组件服务器：是用于扩展非即时通讯本身核心功能的业务，是通过业务服务器包路由过来的客户端请求，并通过组件服务器应答客户端。

>代理服务器：是负责适配不同即时通讯协议实现不同即时通讯的互联。

## 为什么使用openfire
A、Openfire为Java开源项目

B、采用开放的XMPP协议

C、有多种针对不通系统的版本

D、使用Socket通讯

E、 单台服务器可支持上万并发用户,搭建分布式云服务器可轻松提供大量并发用户。

F、 Socket长连接

G、服务器稳定小

H、提供接口，可自己开发插件 

## JabberD2

### 组件

- 路由(router):路由器是jabberd的核心组件，它从其他组件接受信息，并把各个组件间传递xml数据包
- 服务器－服务器(s2s):S2S控制和其他服务器的通信，并实现服务器回呼和远程jabber服务器的验证
- 分解器(resolver):分解器是为支持S2S工作的.他为S2S回呼中验证部分提供分解主机名服务
- 会话管理(Session Manager):SM(会话管理)实现了即时消息的大部分  
      - 消息传送  
      - 状态管理(Presence)  
      - 帐户管理(Rosters)  
      - 订阅(Subscriptions)  
- 客户端－服务器端(c2s): C2S组件控制与客户端的通信  
      - 和jabbar客户端连接  
      - 传递包给SM  
      - 验证客户端  
      - 注册用户  
      - 同SM引发活动  

### 数据控制
jabberd 使用数据控制(data handing)的概念以便适应各类数据处理包。数据控制(data handling)的核心是收集器(Collection)对象概念。每个收集器(Collection)都有类型(Type)和拥有者(Owner)两个属性.类型(Type)指明什么类型的数据正在被处理,如,队列(queue),vcard,名册条目(roster-item). 拥有者(Owner)表明谁拥有这个收集器(collection).对于和用户相关的数据，拥有者(Owner)是jabber ID(JID).

### JID
jid为客户端的唯一性标识id，{$user}@{$host}组成一个唯一性jid／{$work}，根据work用于将数据发送到与他的工作相关的工具

## 参考文献
1. [XMPP权威指南](http://wiki.jabbercn.org/%E9%A6%96%E9%A1%B5)
2. [XMPP-RFC3920](http://wiki.jabbercn.org/RFC3920)

