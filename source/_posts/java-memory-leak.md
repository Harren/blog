---
title: Java内存泄漏分析
date: 2017-06-24 12:23:10
categories:
- Java
tags:
- 内存泄漏
---
> 最近在Java项目运行中遇到，程序在启动后内存不断增加，单个java应用的内存占用到了13G，由于之前一直也没考虑过内存的问题，遇到这种内存泄漏的问题在这里记录一下解决的方案。

Java中的内存泄露，广义并通俗的说，就是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成的内存空间的浪费。Java中的内存泄露与C++中的表现有所不同。在C++中，所有被分配了内存的对象，不再使用后，都必须程序员手动的释放他们。所以，每个类，都会含有一个析构函数，作用就是完成清理工作，如果我们忘记了某些对象的释放，就会造成内存泄露。

但是在Java中，我们不用（也没办法）自己释放内存，无用的对象由GC自动清理，这也极大的简化了我们的编程工作。但，实际有时候一些不再会被使用的对象，在GC看来不能被释放，就会造成内存泄露。我们知道，对象都是有生命周期的，有的长，有的短，如果长生命周期的对象持有短生命周期的引用，就很可能会出现内存泄露。

## 内存分析工具
由于确定是内存的问题，需要使用jdk工具来获取到java的内存使用情况，在这里介绍jmap和mat两个工具。

### Jmap(内存映像工具)
jmap(Memory Map for Java) 命令用于生成堆存储文件，它还可以查询finalize执行队列、Java堆和永久代的详细信息，例如空间使用率、当前使用的是哪种收集器等，其命令行格式为
```bash
jmap [ option ] <pid>
```
其中<pid>为进程号，option选项的和合法值与具体的含义如下：

|  选 项 | 作 用 | 
|-----|---------|
| -dump | 生成Java堆转储文件，格式为：-dump:[live, ]formate=b, file=<filename>, 其中live子参数说明是否只dump出存活的对象 | 
| -finalizerinfo | 显示在F-Queue中等待Finalizer线程执行finalize方法的对象。只在类Unix下有效 | 
| -heap | 显示Java堆的详细信息，如使用哪种回收期、参数配置、分代状态等。只在类Unix下有效 | 
| -histo | 显示堆中信息的统计信息，包括类、实例个数、合计容量 | 
| -permstat | 以ClassLoader为统计口径显示永久代内存状态。只在类Unix下有效 | 
| -F | 当虚拟机进城对 -dump 选项没要响应时，使用这个选项强制生成dump快照 | 

首先打印heap空间的概要，这里可以粗略的检验heap空间的使用情况
```bash
[root@bx-docker007 kanghua]# jmap -heap 128060
Attaching to process ID 128060, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.111-b14

using thread-local object allocation.
Parallel GC with 18 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 32210157568 (30718.0MB)
   NewSize                  = 703070208 (670.5MB)
   MaxNewSize               = 10736369664 (10239.0MB)
   OldSize                  = 1406664704 (1341.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 5145362432 (4907.0MB)
   used     = 228703800 (218.10894012451172MB)
   free     = 4916658632 (4688.891059875488MB)
   4.444853069584506% used
From Space:
   capacity = 18350080 (17.5MB)
   used     = 18321760 (17.472991943359375MB)
   free     = 28320 (0.027008056640625MB)
   99.84566824776786% used
To Space:
   capacity = 11534336 (11.0MB)
   used     = 0 (0.0MB)
   free     = 11534336 (11.0MB)
   0.0% used
PS Old Generation
   capacity = 5639241728 (5378.0MB)
   used     = 75488904 (71.99182891845703MB)
   free     = 5563752824 (5306.008171081543MB)
   1.3386357180821316% used
```
可以看出新生代和老年代的空间都很大，但是实际的使用的内存空间都比较小，考虑是整个程序中产生了大内存然后释放造成的整体内存很大，如果发现老年代的使用率比较高的话，可以手动进行一次FullGC观察老年代的使用率是否有变化，命令如下：
```
jmap -histo:live <pid>
```
检测gc的次数，命令如下
```
jstat -gc <pid> <period> <times>
```
输出如下所示
```
[root@bx-docker007 kanghua]# jstat -gc vmid 1000 1
S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
23040.0 20480.0  0.0   20372.5 4788224.0 2295539.6 5507072.0   81879.6   61912.0 60106.4 7168.0 6775.4  36327  493.242   7      2.401  495.643
```
其中结果中每个项目的含义可以参考官方对[jstat](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html)的文档，简单翻译如下
- S0C: Young Generation第一个survivor space的内存大小 (kB).
- S1C: Young Generation第二个survivor space的内存大小 (kB).
- S0U: Young Generation第一个Survivor space当前已使用的内存大小 (kB).
- S1U: Young Generation第二个Survivor space当前已经使用的内存大小 (kB).
- EC: Young Generation中eden space的内存大小 (kB).
- EU: Young Generation中Eden space当前已使用的内存大小 (kB).
- OC: Old Generation的内存大小 (kB).
- OU: Old Generation当前已使用的内存大小 (kB).
- PC: Permanent Generation的内存大小 (kB)
- PU: Permanent Generation当前已使用的内存大小 (kB).
- YGC: 从启动到采样时Young Generation GC的次数
- YGCT: 从启动到采样时Young Generation GC所用的时间 (s).
- FGC: 从启动到采样时Old Generation GC的次数.
- FGCT: 从启动到采样时Old Generation GC所用的时间 (s).
- GCT: 从启动到采样时GC所用的总时间 (s).

其中，可以发现FullGC的次数为7次，从这里可以看出没有内存泄漏问题的存在，如果频繁发生FullGC，那么存在内存泄漏的问题，就需要将堆栈信息导出进行分析了,下面根据该命令对于发生内存泄露的应用 dump 快照文件
```bash
jmap -dump:format=b,file=/tmp/dump1.hprof <pid>
```

### MAT(堆转储分析工具)
Sun JDK提供了jhat(JVM Heap Analysis Tool)命令与jmap搭配使用，来分析jmap生成的堆转储快照，但是，在实际工作中，一般都不会使用jhat命令分析dump文件，主要原因有两个：一是一般不会在部署应用服务的服务器上直接分析dump文件，因为分析工作是一个耗时而且消耗硬件资源的过程；另一个原因是jhat的分析功能相对来说比较简陋，我使用专业用于分析dump文件的Eclipse Memory Analyzer进行分析查找内存泄漏问题。

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1forl2ndl5dj217h0qojvu.jpg "图1 MAT预览分析页面")

图1即是对于jmap命令生成的堆文件进行分析的界面，下面分别介绍其主要子功能模块

## 堆栈分析
图1为分析预览界面，我们需要根据其提供的信息定位到具体到溢出的类或者代码。

### Leak Suspecst
该功能模块是分析后提示你可能会出现内存泄漏的类以及内存泄漏原因，一般情况下可以在这个地方定位到具体的问题，可以发现产生内存泄漏的对象名以及详情信息，基本可以定位到程序中的具体的类名以及造成内存泄漏的原因，如果在这一步无法定位的话，继续下一步。
![](http://wx2.sinaimg.cn/mw690/78d85414ly1formexwixsj20it0emjtp.jpg "图2 泄漏报告")

### Histogram（直方图）视图
Histogram（直方图）视图，可以列出每个类产生的实例数量，以及所占用的内存大小和百分比。主界面如下图2所示

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1forlokp6loj20kc0bn427.jpg "图3 Histogram视图")

图中Shallow Heap 和 Retained Heap 分别表示对象自身不包含引用的大小和对象自身并包含引用的大小，将列表按照 Retained Heap 进行排序，在图2中可以发现有对象实例只有一个但是其占用的内存有近400M，在该类上可能存在内存泄漏问题，下面再检查下对象和引用的关系的结构图。

### Dominator Tree视图
Dominator Tree（支配树）视图，在此视图中列出了每个对象（Object Instance）与其引用关系的树状结构，同时包含了占用内存的大小和百分比。

![](http://wx1.sinaimg.cn/mw1024/78d85414ly1forlt6lvycj20my08bwhm.jpg "图4 Dominator Tree视图")

通过Dominator Tree视图可以很容易的找出占用内存最多的几个对象（根据Retained Heap或Percentage排序，图3中红色标记的类即为占用内存最多的对象。

### 定位溢出源
Histogram视图和Dominator Tree视图的角度不同，前者是基于类的角度，后者是基于对象实例的角度，并且可以更方便的看出其引用关系。

首先，在两个视图中找出疑似溢出的对象或者类（可以通过Retained Heap排序，并且可以在Class Name中输入正则表达式的关键词只显示指定的类名），然后右键选择Path To GC Roots（Histogram中没有此项）或Merge Shortest Paths to GC Roots，然后选择 exclude all phantom/weak/soft etc. reference。

GC Roots意为GC根节点，其含义见上面的 GC Roots和Reference Chain 部分，后面的 exclude all phantom/weak/soft etc. reference 意思是排除虚引用、弱引用和软引用，即只剩下强引用，因为除了强引用之外，其他的引用都可以被JVM GC掉，如果一个对象始终无法被GC，就说明有强引用存在，从而导致在GC的过程中一直得不到回收，最终就内存溢出了。

![](http://wx2.sinaimg.cn/mw690/78d85414ly1forlzkcvjwj20nf0f4dly.jpg "图5 GC溢出结果")

图5是执行的结果，上图中保留了大量的MetricRegister的引用，因此修改为单例方式就可以解决这个问题。

## 参考文献
1. [Eclipse Memory Analyzer](http://www.eclipse.org/mat/)
2. [使用MAT的Histogram和Dominator Tree定位溢出源](http://www.javatang.com/archives/2017/11/08/11582145.html)