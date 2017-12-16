---
title: Java并发-Executors框架和线程池
date: 2016-01-17 11:28:11
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
 
```Java
public class ThreadPoolDemo {

    public static void main(String[] args) {
        int poolSize = 5;
        ExecutorService es = Executors.newFixedThreadPool(poolSize);
        for(int i = 0;i < 10;i++) {
            es.submit(new Runnable() {
                @Override
                public void run() {
                    System.out.println("thread begin");
                }
            });
        }

    }
}

```
### newSingleThreadExecutor
该方法返回一个只有一个线程的线程池，若多于一个任务被执行到该线程池，任务会被保存在一个任务队列，等到线程空闲的时候按照先入先出的顺序执行，适用于需要顺序执行的场景。
### newCachedThreadPool
该方法返回一个可根据实际情况调整线程数量的线程池，适用于线程池的线程数量不确定，但是又需要即时执行的场景，若当前有空闲线程可复用，则会优先使用可复用的线程，否则会创建新的线程处理任务(<font color=red>PS:如果使用不当会有OOM的风险)</font>。

### newSingleThreadScheduledExecutor
该方法返回一个ScheduledExecutorService对象，线程池大小为1, ScheduledExecutorService对象拓展了在给定时间定时执行某任务的功能，例如在某个固定的延时之后执行，或者周期性执行某个任务，其任务调度的方式有三种：

* schedule() 在给定的时间，对任务执行一次调度
* scheduleAtFixedRate() 按照一定的频率进行调度，类似于linux的定时任务，以执行时间为起点，每个一定的周期执行，它不会关注上一次执行的状态，适用于能准确估计任务执行时间的场景。
* scheduleWithFixedDelay() 在上一次任务结束后，在经过delay时间后进行调度，它需要知道上一次任务执行的状态，适用于调度任务执行时间不确定的场景。

```java
public static void main(String[] args) {
        int coreSize = 10;  // the number of threads to keep in the pool
        ScheduledExecutorService ses = Executors.newSingleThreadScheduledExecutor();
        
        //ses.scheduleWithFixedDelay(
        ses.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("just test");
            }
        }, 0, 2, TimeUnit.SECONDS);
    }
```
### newScheduledThreadPool
该方法返回一个ScheduledExecutorService对象，和newSingleThreadScheduledExecutor的区别在于可以指定线程数量

## 线程池的内部实现
对于以上介绍的核心的几个线程池，尽管在功能上具有不同的特点，但是其内部实现均使用了ThreadPoolExecutor实现，下面给出几个线程池的实现方式：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                 0L, TimeUnit.MILLISECONDS,
                                 new LinkedBlockingQueue<Runnable>());
}
    
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

public static ExecutorService newSingleThreadExecutor() {
     return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}

```
以上的实现代码可以看到，核心的线程实现都是ThreadPoolExecutor的封装，其构造函数如下

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
}
```
各个参数的含义如下：

* corePoolSize: 指定线程池中的线程数量。
* maximumPoolSize: 指定线程池中的最大线程数量。
* keepAliveTime: 当线程池线程数量超过corePoolSize时，多余的空闲线程的存活时间，即超过corePoolSize的空闲线程，在多长时间内，会被销毁。
* unit: keepAliveTime的单位。
* workQueue: 任务队列，被提交但尚未执行的任务。
* threadFactory: 线程工厂，用于创建线程，一般默认即可。
* handler: 拒绝策略。当任务来不及处理时候如何拒绝任务

### 任务队列
workQueue用于存放Runnable对象，它是一个BlockingQueue接口的对象，可以使用以下几种几种BlockingQueue。

* 直接提交的队列：通过SynchronousQueue队列实现，该队列没有容量，因此每次提交的任务都不会真实的保存，如果没有空闲的线程，则会尝试创建新的线程，如果达到进程的最大值，则会执行拒绝策略，如果使用这种队列，往往需要设置较大的maximumPoolSize。
* 有界的任务队列：通过ArrayBlockingQueue实现。由于是数组，所以其构造需要指定容量参数。当有新的任务需要执行时，如果线程池的线程数小于corePoolSize，则会创建新的线程，反之，则将该任务加入队列，如果队列装满时，则会将线程数提升到maximumPoolSize后，就不会再增加，后续仍有新任务执行则直接执行拒绝策略。
* 无界的任务队列：通过LinkedBlockingQueue实现。由于是无解的任务队列，当系统的线程数小于corePoolSize数时，线程池会生成新的线程执行任务，反之，则任务直接加入队列进行等待，一但任务处理速度跟不上线程的创建速度，无界队列会无限增长，直到耗尽系统内存。
* 优先任务队列：通过PriorityBlockingQueue实现，它是一个特殊的无界队列，它可以根据自身任务的优先级瞬息先后执行而不用考虑先进先出。

结合目前的介绍，可以分析出newFixedThreadPool()由于固定了线程池，其使用了LinkedBlockingQueue作为任务队列。

newSingleThreadExecutor()返回的单进程线程池，是线程池数量为1的newFixedThreadPool()。

newCachedThreadPool()返回了corePoolSize为0，maximumPoolSize为无穷大的线程池，其将任务加入SynchronousQueue队列，而SynchronousQueue队列是直接提交的队列，它会迫使线程池增加新的线程执行任务。

### 拒绝策略
前面介绍到的，当任务数量超过系统实际承载压力时，这个时候对于后续进来的任务就需要进行拒绝，避免系统压力太大而直接崩掉，jdk内置的拒绝策略包含以下四种：

* AbortPolicy策略: 直接抛出一场，阻止系统正常工作。
* CallerRunsPolicy策略: 只要线程池未关闭，会直接在调用者线程中运行当前被丢弃的任务，虽然并不会丢弃任务，但是，任务提交线程的性能会急剧下降。
* DiscardOledestPolicy策略: 丢弃最老的一个请求，也就是被执行的一个任务，并尝试再次提交当前任务。
* DiscardPolicy策略: 丢弃无法处理的任务，不予任何处理。

如果以上的策略无法满足需求，完全可以自定义拒绝策略，下面代码简单的演示自定义线程池和拒绝策略的使用：

```java
public static void main(String[] args) {
    ExecutorService es = new ThreadPoolExecutor(0, 100, 0L, TimeUnit.MILLISECONDS,
           new SynchronousQueue<>(),
           new ThreadFactory() {
                    @Override
                    public Thread newThread(Runnable r) {
                    	 //  自定义线程生产
                        return null;
                    }

           },
           new RejectedExecutionHandler() {
                    @Override
                    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
							// 自定义拒绝策略
                    }
           });
    }
```

## 参考资料
1. Java高并发程序设计