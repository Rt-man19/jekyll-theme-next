---

title: SparkShuffle详解剖析

categories:

- SparkSQL

tags:

---
## HashShuffle

### 概述

所谓Shuffle就是将不同节点上相同的Key拉取到一个节点的过程。这之中涉及到各种IO，所以执行时间势必会较长，<font color=#FF4500>Spark的Shuffle在1.2之前默认的计算引擎是HashShuffleManager</font>，不过HashShuffleManager有一个十分严重的弊端，就是会产生大量的中间文件。<font color=#FF4500>在1.2之后默认Shuffle改为SortShuffleManager</font>，相对于之前，在每个Task虽然也会产生大量中间文件，但是最后会将所有的临时文件合并（merge）成一个文件。因此Shuffle read只需要读取时，根据索引拿到每个磁盘的部分数据就可以了
<!--more--> 
### 测试条件

`每个Executor只有一个CUP（core），同一时间每个Executor只能执行一个task`

### HashShuffleManager未优化版本

首先从shuffle write阶段，主要是在一个stage结束后，为了下一个stage可以执行shuffle，将每一个task的数据按照key进行分类，对key进行hash算法，从而使相同的key写入同一个文件，每个磁盘文件都由下游stage的一个task读取。在写入磁盘时，先将数据写入内存缓冲，当内存缓冲填满后，才会溢写到磁盘文件（似乎所以写文件都需要写入先写入缓冲区，然后再溢写，防止频繁IO）

我们可以先算一下当前stage的一个task会为下一个stage创建多少个磁盘文件。若下一个stage有100个task，则当前stage的每一个task都将创建100个文件，若当前stage要处理的task为50个，共有10个Executor，也就是说每个Executor共执行5个task，5x100x10=1000。也就是说这么一个小规模的操作会生产5000个文件。这是相当可观的。

而shuffle read 通常是一个stage一开始要做的事情。此时stage的每一个task去将上一个stage的计算结果的所有相同的key从不同节点拉到自己所在节点。进行聚合或join操作。在shuffle write过程，每个task给下游的每个task都创建了一个磁盘文件。在read过程task只需要去上游stage的task中拉取属于自己的磁盘文件。

shuffle read是边拉取边聚合。每一个read task都有一个buffer缓冲，然后通过内存中的Map进行聚合，每次只拉取buffer大小的数据，放到缓冲区中聚合，直到所有数据都拉取完。

![HashShuffleManager未优化版本](assets/blogImg/2019-03-06-1.png)

### 优化版本

这里说的优化，是指我们可以设置一个参数，spark.shuffle.consolidateFiles。该参数默认值为false，将其设置为true即可开启优化机制。通常来说，如果我们使用HashShuffleManager，那么都建议开启这个选项。

开启这个机制之后，在shuffle write时，task并不是为下游的每一个task创建一个磁盘文件。引入了shuffleFileGroup的概念，每个shuffleFileGroup都对应一批磁盘文件。磁盘文件数量与下游task相同。只是仅仅第一批执行的task会创建一个shuffleFIleGroup，将数据写入到对应磁盘文件。

在执行下一批的task时，会复用已经创建好的shuffleFIleGroup和磁盘文件，即数据会继续写入到已有的磁盘文件。该机制会允许不同task复用同一个磁盘文件，对于多个task进行了一定程度的合并，大幅度减少shuffle write时，文件的数量，提升性能。

相对于优化前，每个Executor之前需要创建五百个磁盘文件，因为之前需要5个task线性执行，而使用参数优化之后，就每个Executor只需要100个就可以了，这样10个Executor就是1000个文件，这比优化前整整减少了4000个文件。

![优化版本](assets/blogImg/2019-03-06-2.png)

## SortShuffle

在<font color=#FF4500>Spark1.2版本之后，出现了SortShuffle</font>，这种方式以更少的中间磁盘文件产生而远远优于HashShuffle。而它的运行机制主要分为两种。一种为普通机制，另一种为bypass机制。而bypass机制的启动条件为，当shuffle read task的数量小于等于spark.shuffle.sort.bypassMergeThreshold参数的值时（默认为200），就会启用bypass机制。即当read task不是那么多的时候，采用bypass机制是更好的选择。

### 普通运行机制

在该模式下，数据会先写入一个数据结构，聚合算子写入Map，一边通过Map局部聚合，一遍写入内存。Join算子写入ArrayList直接写入内存中。然后需要判断是否达到阈值，如果达到就会将内存数据结构的数据写入到磁盘，清空内存数据结构。

在溢写磁盘前，先根据key进行排序，排序过后的数据，会分批写入到磁盘文件中。默认批次为10000条，数据会以每批一万条写入到磁盘文件。写入磁盘文件通过缓冲区溢写的方式，每次溢写都会产生一个磁盘文件，也就是说一个task过程会产生多个临时文件。

最后在每个task中，将所有的临时文件合并，这就是merge过程，此过程将所有临时文件读取出来，一次写入到最终文件。意味着一个task的所有数据都在这一个文件中。同时单独写一份索引文件，标识下游各个task的数据在文件中的索引，start offset和end offset。

这样算来如果第一个stage 50个task，每个Executor执行一个task，那么无论下游有几个task，就需要50个磁盘文件。

![普通运行机制](assets/blogImg/2019-03-06-3.png)

### bypass机制

<font size=4><b>bypass机制运行条件：</b></font>

1. **shuffle map task数量小于spark.shuffle.sort.bypassMergeThreshold参数的值。**
2. **不是聚合类的shuffle算子（比如reduceByKey）。**

在这种机制下，当前stage的task会为每个下游的task都创建临时磁盘文件。将数据按照key值进行hash，然后根据hash值，将key写入对应的磁盘文件中（个人觉得这也相当于一次另类的排序，将相同的key放在一起了）。最终，同样会将所有临时文件依次合并成一个磁盘文件，建立索引。

### 优点

该机制与未优化的hashshuffle相比，没有那么多磁盘文件，下游task的read操作相对性能会更好。

该机制与sortshuffle的普通机制相比，在readtask不多的情况下，首先写的机制是不同，其次不会进行排序。这样就可以节约一部分性能开销。 

![优点](assets/blogImg/2019-03-06-4.png)
<font color=#FF4500 >
</font>