---
layout: post
title: "Spark内存管理之三 UnifiedMemoryManager分析"
date: 2019-04-16
comments: true
tags: 
	- spark
	- 高级
categories: Spark Other
---
### acquireExecutionMemory方法
UnifiedMemoryManager中的accquireExecutionMemory方法： 

![enter description here](/assets/blogImg/2019-04-16内存管理1.png)

当前的任务尝试从executor中<font size=3>**获取numBytes这么大的内存**</font>

该方法直接向ExecutionMemoryPool索要所需内存，索要内存有以下几个关注点：

- 当ExecutionMemory 内存充足，则不会触发向Storage申请内存
- 每个Task能够被使用的内存是被限制的
- 索要内存的大小

我们通过源码来进行分析
<!--more--> 
**`UnifiedMemoryManager.scala中`**
![enter description here](/assets/blogImg/2019-04-16内存管理2.png)

我们点进去后会发现，会调用ExecutionMemoryPool.acquireMemory()方法

**`ExecutionMemoryPool.scala中`**

![enter description here](/assets/blogImg/2019-04-16内存管理3.png)

我们可以发现每Task能够被使用的内存被限制在： 

poolSize / (2 * numActiveTasks) ~ maxPoolSize / numActiveTasks 之间

val maxMemoryPerTask = maxPoolSize /numActiveTasks 

val minMemoryPerTask = poolSize / (2 * numActiveTasks)

**`UnifiedMemoryManager.scala中`**

![enter description here](/assets/blogImg/2019-04-16内存管理4.png)

其中maxPoolSize = maxMemory - math.min(storageMemoryUsed, storageRegionSize) 

maxMemory = storage + execution的最大内存 

poolSize = 当前这个pool的大小 

maxPoolSize = execution pool的最大内存

**`UnifiedMemoryManager.scala中`**

![enter description here](/assets/blogImg/2019-04-16内存管理5.png)

从上述代码中我们可以知道索要内存的大小： 

val memoryReclaimableFromStorage=math.max(storageMemoryPool.memoryFree, storageMemoryPool.poolSize -storageRegionSize) 

取决于StorageMemoryPool的剩余内存和 storageMemoryPool 从ExecutionMemory借来的内存哪个大，取最大的那个，作为可以重新归还的最大内存 

用公式表达出来就是这一个样子： 

ExecutionMemory 能借到的最大内存 = StorageMemory 借的内存 + StorageMemory 空闲内存 

**注意：**如果实际需要的小于能够借到的最大值，则以实际需要值为准

能回收的内存大小为： 

val spaceToReclaim =storageMemoryPool.freeSpaceToShrinkPool ( math.min(extraMemoryNeeded,memoryReclaimableFromStorage))

### ExecutionMemoryPool.acquireMemory()解析

```
    while (true) {
      val numActiveTasks = memoryForTask.keys.size
      val curMem = memoryForTask(taskAttemptId)
      maybeGrowPool(numBytes - memoryFree)
      val maxPoolSize = computeMaxPoolSize()
      val maxMemoryPerTask = maxPoolSize / numActiveTasks
      val minMemoryPerTask = poolSize / (2 * numActiveTasks)
      val maxToGrant = math.min(numBytes, math.max(0, maxMemoryPerTask - curMem))
      val toGrant = math.min(maxToGrant, memoryFree)
      if (toGrant < numBytes && curMem + toGrant < minMemoryPerTask) {
        logInfo(s"TID $taskAttemptId waiting for at least 1/2N of $poolName pool to be free")
        lock.wait()
      } else {
        memoryForTask(taskAttemptId) += toGrant
        return toGrant
      }
    }
```

整体流程解析： 

程序一直处理该task的请求，直到系统判定无法满足该请求或者已经为该请求分配到足够的内存为止；如果当前execution内存池剩余内存不足以满足此次请求时，会向storage部分请求释放出被借走的内存以满足此次请求

根据此刻execution内存池的总大小maxPoolSize，以及从memoryForTask中统计出的处于active状态的task的个数计算出： 

每个task能够得到的最大内存数 maxMemoryPerTask = maxPoolSize / numActiveTasks 

每个task能够得到的最少内存数 minMemoryPerTask = poolSize /(2 * numActiveTasks)

根据申请内存的task当前使用的execution内存大小决定分配给该task多少内存，总的内存不能超过maxMemoryPerTask；但是如果execution内存池能够分配的最大内存小于numBytes，并且如果把能够分配的内存分配给当前task，但是该task最终得到的execution内存还是小于minMemoryPerTask时，该task进入等待状态，等其他task申请内存时再将其唤醒，唤醒之后如果此时满足，就会返回能够分配的内存数，并且更新memoryForTask，将该task使用的内存调整为分配后的值 

<font size=3>**一个Task最少需要minMemoryPerTask才能开始执行**</font>

### acquireStorageMemory方法

流程和acquireExecutionMemory类似，当storage的内存不足时，同样会向execution借内存，但区别是当且仅当ExecutionMemory有空闲内存时，StorageMemory 才能借走该内存

**`UnifiedMemoryManager.scala中`**

![enter description here](/assets/blogImg/2019-04-16内存管理6.png)

从上述代码中我们可以知道能借到的内存数为： 

val memoryBorrowedFromExecution = Math.min(onHeapExecutionMemoryPool.memoryFree,numBytes) 

所以StorageMemory从ExecutionMemory借走的内存，完全取决于当时ExecutionMemory是不是有空闲内存；借到内存后，storageMemoryPool增加借到的这部分内存，之后同上一样，会调用StorageMemoryPool的acquireMemory()方法

### StorageMemoryPool.acquireMemory

![enter description here](/assets/blogImg/2019-04-16内存管理7.png)

整体流程解析： 

在申请内存时，如果numBytes大于此刻storage内存池的剩余内存，即if (numBytesToFree > 0)，那么需要storage内存池释放一部分内存以满足申请需求 

**注意：**这里的numBytesToFree可以理解为numBytes大小减去Storage内存池剩余大小，大于0，即所需要申请的numBytes大于Storage内存池剩余的内存 

释放内存后如果memoryFree >= numBytes，就会把这部分内存分配给申请内存的task，并且更新storage内存池的使用情况 

同时StorageMemoryPool与ExecutionMemoryPool不同的是，他不会像前者那样分不到资源就进行等待，acquireStorageMemory只会返回一个true或是false，告知内存分配是否成功