---
title: Pinpoint简单介绍以及应用实例
date: 2018-02-28 14:37:03
categories:
- 技术杂谈
tags:
- 性能监控
---
Pinpoint是一个开源的APM(Application Performance Management/应用性能管理)工具，用于基于Java的大规模分布式系统，思路基于google Dapper，用于基于java的大规模分布式系统，通过跟踪分布式应用之间的调用来提供解决方案，以帮助分析系统的总体结构和内部模块之间如何相互联系，其开发的初衷主要基于以下思考元素。

![pinpoint](http://wx1.sinaimg.cn/mw1024/78d85414ly1fl64c51y2dj21ct0o5afl.jpg "图1 监控的提出")

<!-- more -->
## 系统架构
Point是基于Google Dapper，主要有三个组件：

* pinpoint-collector，日志收集器模块，主要手收集从agent端传来的数据信息并存储
* pinpoint-web，控制台视图模块，主要将collector的数据可视化的展示给用户
* pinpoint-agent，日志代理客户端模块，用于在客户段进行埋点来获取到监控信息

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1fl65eq70ilj21800oin63.jpg "图2 Pinpoint架构")


同时使用Hbase作为存储，point主要特点如下图，包括以下几点

* 分布式事务跟踪，跟踪跨越分布式应用的消息
* 自动检测应用拓扑，帮助你搞清楚应用的架构
* 水平拓展以便支持大规模服务器集群
* 提供代码级别的可见性以便轻松定位失败点和瓶颈
* 使用字节码增强技术，添加新功能无需修改代码(AOP技术)

![pinpoint](http://wx3.sinaimg.cn/mw1024/78d85414ly1fl64t0zuysj21b70n843m.jpg "图3 Pinpoint主要特点")

在比较复杂的系统中利用pinpoint可以有效的看到系统的瓶颈，其常用功能如下所示：

![pinpoint](http://wx1.sinaimg.cn/mw1024/78d85414ly1fl64t43mu4j21e70l0k0k.jpg "图4 Pinpoint主要功能")

## 主要技术
Pinpoint的主要技术设计到分布式事务跟踪，主要分为两个主要技术：事务追踪技术以及字节码增强技术来实现无侵入式的性能监控
### 数据结构
Pinpoint跟踪单个事务的分布式请求，分布式追踪系统的核心是在分布式系统中识别在Node1中处理的消息和在Node2中出的消息之间的关系。HTTP请求中的HTTP header中为消息添加一个标签信息并使用这个标签跟踪消息，即TraceID，Pinpoint中，核心数据结构由Span，Trace和TraceID组成:

* Span:跟踪的基本单元，包含一个TraceId
* Trace:多个Span集合，由关联的RPC（Spans）组成，同一个trace共享一个相同的TransactionID,Trace通过SpanId和ParentSpanId整理继承树结构。
* TraceID:由 TransactionId, SpanId(64位长度的整型), 和 ParentSpanId(64位长度的整型) 组成的key的集合. TransactionId 指明消息ID，而SpanId 和 ParentSpanId 表示RPC的父-子关系
* TransactionId：AgentIDs(建议使用hostname，服务器IP),JVM启动时间以及序列号组成

生成唯一性ID的方法，通过中央key服务器来生成key。如果实现这个模式，可能导致性能问题和网络错误，因此，大量生成key被考虑作为备选。

![](http://wx3.sinaimg.cn/mw1024/78d85414ly1fl65rkvn6qj215w0niae6.jpg '图5 核心数据结构')

### Trace行为
下图5描述在4个节点之间进行3次rpc调用：

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1fl65rqgpx9j21bs0nmtdx.jpg '图6 trace行为')

上图中，TransactionID(TxId) 体现了三次不同的rpc作为单个事务被相互关联，由于 TransactionID 本身不能精确描述rpc之间的关系，为了识别 rpc 之间的关系，需要 SpanId 和 ParentSpanId， 假设一个节点是 Tomcat， 可以将 SpanId 想象为出力http请求的线程，ParentSpanId 代表发起这个rpc调用的 SpanId。

SpanId 和 ParentSpanId 是64位长度的整型，由于这个数字是任意生成的，但是考虑到值的范围从 -2^64 ~ 2^64， 不太可能发生冲突，如果发生冲突，系统会让开发者知道发生了什么，而不是去解决冲突。

### 字节码增强

实现分布式事务跟踪的实现方法之一是开发人员自己修改代码，在发生rpc调用的地方开发人员自己添加标签信息，这就需要修改到项目代码，对代码有一定的侵入性，为了解决这个问题，pinpoint中使用了字节码增强技术，由 pinpoint-agent 干预发起rpc的代码来实现自动处理标签信息，如下图

![](http://wx3.sinaimg.cn/mw1024/78d85414ly1fl65ryql33j21990mzgom.jpg '图7 字节码增强')

在程序编译阶段通过反射方式注入代码来实现无侵入的埋点，这种代码跟踪方式于手工跟踪对比如下图8所示

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1fl65rvfmpsj21520nete5.jpg '图8 性能对比')

只需要在项目启动的过程中加入 agent 即可实现性能监控

```
javaagent:$AGENR_PATH/pinpoint-bootstrap.jar
-Dpinpoint.agentId=<Agent’s UniqueId>
-Dpinpoint.applicationName=<the name of service>
```

## 应用分析
阐述point为每个方法做了什么：

1.  请求到达tomcatA时，Pinpoint agent产生TraceID
2.  从springMVC控制器记录数据
3.  插入HttpClient.execute()方法的调用并在HttpGet中配置TraceId
4.  传输打好tag的请求到tomcatB
5.  从springmvc控制器中记录数据并完成请求
6.  从tomcatB回来的请求完成时，pp-agent发送跟踪数据到pp-collector存储在hbase
7.  在对tomcatB的http调用结束后，tomcatA的请求也完成，pp-agent发送跟踪数据到pp-collector存储在HBase中
8.  UI从Hbase中读取数据并通过排序树来创建调用栈

整体的应用分析如下图所示

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1fl65s2cda8j21eu0r0k0v.jpg '图9 应用分析')

## 参考文献
1. [Pinpoint源码](https://github.com/naver/pinpoint)
2. [Techinal Overview Of Pinpoint (包含中文手册)](https://github.com/naver/pinpoint/wiki/Technical-Overview-Of-Pinpoint)





                                                                  
                                                                         
                     
