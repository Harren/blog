---
title: Java并发-Executors框架和线程池
date: 2015-01-17 11:28:11
tags: Java
---
为了避免系统频繁的创建和销毁线程，通常会让创建的线程进行复用。一般我们进行数据库开发的时候，为了避免每次数据库查询都重新建立和销毁数据库连接，我们可以利用数据库连接池维护一些数据连接，让其长期保持在一个激活的状态，当需要使用数据库的时候，并不是创建一个连接，而是从连接池中获取一个可用的连接。

线程池也是和数据库连接池类似的概念。线程池中总有几个活跃的线程，当你需要使用线程时，可以直接从池子里随便拿出一个空闲的线程，完成任务后将线程退回到池子。
## Executors框架
Eexecutor作为灵活且强大的异步执行框架，其支持多种不同类型的任务执行策略，提供了一种标准的方法将任务的提交过程和执行过程解耦开发，基于生产者-消费者模式，其提交任务的线程相当于生产者，执行任务的线程相当于消费者，并用Runnable来表示任务，Executor的实现还提供了对生命周期的支持，以及统计信息收集，应用程序管理机制和性能监视等机制。JDK提供了一整套的Executor框架来进行线程控制，其主要成员如下：
![Executor框架结构图](http://img.blog.csdn.net/20160607192300746 "图1 Executor框架结构图")
其中：

* Executor 执行器接口，该接口定义执行Runnable任务的方式。
* ExecutorService 该接口定义提供对Executor的服务。
* ScheduledExecutorService 定时调度接口。
* AbstractExecutorService 执行框架抽象类。
* ThreadPoolExecutor JDK中线程池的具体实现。
* Executors 线程池工厂类。

ExecutorService的生命周期包括三种状态：运行、关闭、终止。创建后便进入运行状态，当调用了shutdown()方法时，便进入关闭状态，此时意味着ExecutorService不再接受新的任务，但它还在执行已经提交了的任务，当素有已经提交了的任务执行完后，便到达终止状态。如果不调用shutdown()方法，ExecutorService会一直处在运行状态，不断接收新的任务，执行新的任务，服务器端一般不需要关闭它，保持一直运行即可。

Executor框架提供了各种类型的线程池，主要有以下几个工厂方法：

```Java
/***创建固定大小的线程池***/
public static ExecutorServcie newFixedThreadPool(int nThread);
/***单线程的线程池***/
public static ExecutorServcie newSingleThreadExecutor();
/***可缓存的线程池***/
public static ExecutorServcie newCachedThreadPool();
/***定时任务调度的线程池***/
public static ExecutorServcie newSingleThreadScheduledExecutor();
/***单线程的定时任务调度线程池***/
public static ExecutorServcie newScheduledThreadPool(int corePoolSize);

```

###  newFixedThreadPool
返回一个包含指定数目线程的线程池，如果任务数量多于线程数目，那么没有没有执行的任务必须等待，直到有任务完成为止。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程，简单的展示一下该类线程池的使用
 
```

```
### newSingleThreadExecutor
### newCachedThreadPool
### newSingleThreadScheduledExecutor
### newScheduledThreadPool




## 线程池的实现



## 参考资料
1. [ForkJoin简单实现原理](http://www.infoq.com/cn/articles/fork-join-introduction)