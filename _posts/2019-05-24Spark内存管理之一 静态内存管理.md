---
layout: post
title: "Spark内存管理之一 静态内存管理"
date: 2019-04-03
comments: true
tags: [spark,高级]
categories: Spark Other
---

### Spark内存管理简介

Spark从<font color="blue">**1.6**</font>开始引入了动态内存管理模式，即执行内存和存储内存之间可以相互抢占

Spark提供了2种内存分配模式：

- 静态内存管理
- 统一内存管理

本系列文章将分别对这两种内存管理模式的优缺点以及设计原理进行分析（主要基于Spark 1.6.1的内存管理进行分析） 

<font color="blue">在本篇文章中，将先对<font size=3 color="red">静态内存管理</font>进行介绍</font>

<!--more--> 
### 堆内内存

在Spark最初采用的静态内存管理机制下，存储内存、执行内存和其它内存的大小在Spark应用程序运行期间均为固定的，但用户可以在应用程序启动前进行配置，堆内内存的分配如下图所示：

![enter description here](/assets/blogImg/2019-04-03-内存管理1.png)

默认情况下，spark内存管理采用unified模式，如果要开启静态内存管理模式，需要将spark.memory.useLegacyMode参数调为true（默认为false），1.6.1版本的官网配置如下所示： 

![enter description here](/assets/blogImg/2019-04-03-内存管理2.png)

将参数调整为true之后，就会进入到静态内存管理中来，可以通过SparkEnv.scala中发现： 

![enter description here](/assets/blogImg/2019-04-03-内存管理3.png)

```
如果spark.memory.useLegacyMode为true，就进入到StaticMemoryManager（静态内存管理）；
如果为false，就进入到UnifiedMemoryManager（统一内存管理）；
同时我们可以发现该参数的默认值为false，即默认情况下就会调用统一内存管理类。
```

### Execution内存

####可用的Execution内存

用于shuffle操作的内存，取决于join、sort、aggregation等过程频繁的IO需要的Buffer临时数据存储 

简单来说，spark在shuffle write的过程中，每个executor会将数据写到该executor的物理磁盘上，下一个stage的task会去上一个stage拉取其所需要处理的数据，并且是边拉取边进行处理的（和MapReduce的拉取合并数据基本一样），这个时候就会用到一个aggregate的数据结构，比如hashmap这种边拉取数据边进行聚合。这部分内存就被称为execution内存

从StaticMemoryManager.scala中的getMaxExecutionMemory方法中，我们可以发现：

![enter description here](/assets/blogImg/2019-04-03-内存管理4.png)

每个executor分配给execution的内存为：

``` 
ExecutionMemory = systemMaxMemory * memoryFraction * safetyFraction 
默认情况下为：systemMaxMemory * 0.2 * 0.8 = 0.16 * systemMaxMemory 
即默认为executor最大可用内存 * 0.16
```

Execution内存再运行的时候会被分配给运行在JVM上的task；这里不同的是，分配给每个task的内存并不是固定的，而是动态的；spark不是一上来就分配固定大小的内存块给task，而是允许一个task占据JVM所有execution内存 

每个JVM上的task可以最多申请至多1/N的execution内存，其中N为active task的个数，由spark.executor.cores指定；如果task的申请没有被批准，它会释放一部分内存，并且下次申请的时候，它会申请更小的一部分内存 

**注：**

- 每个Executor单独运行在一个JVM进程中，每个Task则是运行在Executor中的线程
- spark.executor.cores设置的是每个executor的core数量
- task的数量就是partition的数量
- 一般来说，一个core设置2~4个partition

<font color="red" size=3><b>注意：</b></font>

<font color="red">
为了防止过多的spilling数据，只有当一个task分配到的内存达到execution内存1/2N的时候才会spill，如果目前空闲的内存达不到1/2N的时候，内存申请会被阻塞直到其它的task spill掉它们的内存；
 
如果不这样限制，假设当前一个任务占据了绝大部分内存，那么新来的task会一直往硬盘spill数据，这样就会导致比较严重的I/O问题；而我们做了一定程度的限制，会进行一定程度的阻塞等待，对于频繁的小数据集的I/O会有一定的减缓 

例子：某executor先启动一个task A，并在task B启动前快速占用了所有可用的内存；在B启用之后N变成了2，task B会阻塞直到task A spill，自己可以获得1/2N=1/4的execution内存的时候；而一大task B获取到了1/4的内存，A和B就都有可能spill了
</font>

### 预留内存

Spark之所以会有一个SafetyFraction这样的参数，是为了避免潜在的OOM。例如，进行计算时，有一个提前未预料到的比较大的数据，会导致计算时间延长甚至OOM，safetyFraction为storage和execution都提供了额外的buffer以防止此类的数据倾斜；这部分内存叫作预留内存

####Storage内存

####可用的Storage内存

该部分内存用作对RDD的缓存（如调用cache、persist等方法），节点间传输的广播变量

StaticMemoryManager.scala中的getMaxStorageMemory方法发现：

![enter description here](/assets/blogImg/2019-04-03-内存管理5.png)

最后为每个executor分配到的storage的内存： 

```
StorageMemory = systemMaxMemory * memoryFraction * safetyFraction 
默认情况下为：systemMaxMemory * 0.6 * 0.9 = 0.54 * systemMaxMemory 
即默认分配executor最大可用内存的0.54
```

#### 预留内存

同Execution内存中的预留部分

### Unroll

Unroll是storage中比较特殊的一部分，它默认占据storage总内存的20% 

BlockManager是spark自己实现的内部分布式文件系统，BlockManager接受数据（可能从本地或者其他节点）的时候是以iterator的形式，并且这些数据是有序列化和非序列化的，因此需要注意以下两点：

- Iterator在物理内存上是不连续的，如果后续spark要把数据装载进内存的话，就需要把这些数据放进一个array（物理上连续）
- 另外，序列化数据需要进行展开，如果直接展开序列化的数据，会造成OOM，所以BlockManager会逐渐的展开这个iterator，并逐渐检查内存里是否还有足够的空间用来展开数据放进array里

StaticMemoryManager.scala中的maxUnrollMemory方法：

![enter description here](/assets/blogImg/2019-04-03-内存管理6.png)

Unroll的优先级别还是比较高的，它使用的内存空间是可以从storage中借用的，如果在storage中没有现存的数据block，它甚至可以占据整个storage空间；如果storage中有数据block，它可以最大drop掉内存的数据是通过spark.storage.unrollFraction来控制的，通过源码可知这部分的默认值为0.2 

<font color="red" size=3><b>注意：</b></font>

<font color="red">
这个20%的空间并不是静态保留的，而是通过drop掉内存中的数据block来分配的（动态的分配过程）；如果unroll失败了，spark会把这部分数据evict到硬盘中去
</font>

### eviction策略

在spark技术文档中，eviction一词经常出现，eviction并不是单纯字面上驱逐的意思。说句题外话，spark通常被我们叫做内存计算框架，但是从严格意义上说，spark并不是内存计算的新技术；无论是cache还是persist这类算子，spark在内存安排上，绝大多数用的都是LRU策略（LRU可以说是一种算法，也可以算是一种原则，用来判断如何从Cache中清除对象，而LRU就是“近期最少使用”原则，当Cache溢出时，最近最少使用的对象将被从Cache中清除）。即当内存不够的时候，会evict掉最远使用过的内存数据block；当evict的时候，spark会将该数据块evict到硬盘，而不是单纯的抛弃掉 

无论是storage还是execution的内存空间，当内存区域的空间不够用的时候，spark都会evict数据到硬盘

### Other部分

这部分的内存用于程序本身运行所需要的内存，以及用户定义的数据结构和创建的对象，此内存由上面两部分：storage、execution决定的，默认为0.2

### 堆外内存

Spark1.6开始引入了Off-heap memory（详见SPARK-11389）
 
堆外的空间分配较为简单，只有存储内存和执行内存，如图所示：

![enter description here](/assets/blogImg/2019-04-03-内存管理7.png)

可用的执行内存和存储内存占用的空间大小直接由参数 spark.memory.storageFraction 决定（默认为0.5），由于堆外内存占用的空间可以被精确计算，所以无需再设定保险区域

### 局限性

在Spark的设计文档中，指出了静态内存管理的局限性：

没有适用于所有应用的默认配置，通常需要开发人员针对不同的应用进行不同的参数进行配置：比如根据任务的执行逻辑，调整shuffle和storage的内存占比来适应任务的需求

这样需要开发人员具备较高的spark原理知识

那些不cache数据的应用在运行的时候只会占用一小部分可用内存，而默认的内存配置中storage就用去了60%，造成了浪费
