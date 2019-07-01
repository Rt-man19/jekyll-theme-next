在Spark中Shuffle往往都是作业的性能瓶颈所在，该环节中包含了大量的磁盘IO，序列化，网络传输数据等这些非常消耗性能的操作。但影响Spark作业的主要还是作业代码和给作业分配的资源，Shuffle只占一小部分，但也不可忽略。要想对Shuffle进行调优，源码的学习就是必须的，下面来逐一介绍。



# Shuffle的定义

首先来看官方给Spark中的Shuffle的定义

>In Spark, data is generally not distributed across partitions to be in the necessary place for a specific operation. During computations, a single task will operate on a single partition - thus, to organize all the data for a single reduceByKey reduce task to execute, Spark needs to perform an all-to-all operation. It must read from all partitions to find all the values for all keys, and then bring together values across partitions to compute the final result for each key - this is called the shuffle.
>
>在Spark中数据一般不会跨分区分布在特定操作所需的地方。在计算期间，一个task只会在一个分区上做操作，因此Spark需要执行all-to-all操作才可以把单个reduceByKey这类reduce task所有的数据组织起来。这也就是意味着reduce task 必须从所有分区中读取所有键的值，然后跨分区将数据合并在一起，计算每个键的最终结果，这称为shuffle

官网给的定义比较宽泛，其实这个过程可以再具体一些，从作业运行的角度来解释什么是shuffle

>Spark程序运行主要分两个部分
>
>- Driver: 其核心是SparkContext
>- Task
>
>**具体过程**(以Spark On YARN为基准) : Driver和 ClusterManager通信，向ClusterManager申请作业所需的资源(Executor)，ResourceManager会根据Spark作业设置的参数在各个工作节点中启动一定数量Executor进程。之后Driver程序就开始调度作业代码，将代码拆分为多个Stage，每个Stage执行一部分代码，并分配一批task，一个Stage的所有代码执行完成后会在各个节点本地磁盘文件写入计算中间结果。然后Driver就会调度下一个Stage，下一个Stage的输入数据就是上一个Stage的结果数据。这里，**下一个Stage向上一个Stage拿数据的过程就可以称为shuffle**



# ShuffleManage发展历史

在Spark的源码中，负责shuffle过程的执行、计算和处理的组件主要就是ShuffleManager，也即shuffle管理器。而随着Spark的版本的发展，ShuffleManager也在不断迭代，变得越来越先进。

在Spark 1.2以前，默认的shuffle计算引擎是HashShuffleManager。该ShuffleManager而HashShuffleManager有着一个非常严重的弊端，就是会产生大量的中间磁盘文件，进而由大量的磁盘IO操作影响了性能。

因此在Spark 1.2以后的版本中，默认的ShuffleManager改成了SortShuffleManager。SortShuffleManager相较于HashShuffleManager来说，有了一定的改进。主要就在于，每个Task在进行shuffle操作时，虽然也会产生较多的临时磁盘文件，但是最后会将所有的临时文件合并（merge）成一个磁盘文件，因此每个Task就只有一个磁盘文件。在下一个stage的shuffle read task拉取自己的数据时，只要根据索引读取每个磁盘文件中的部分数据即可。



# HashShuffleManager

HashShuffleManager原理部分请查看调优指南基础篇。



**SparkEnv.scala**

```scala
// Let the user specify short names for shuffle managers
    val shortShuffleMgrNames = Map(
      "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
      "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
      "tungsten-sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager")
    val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
    val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
    val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
```

在SparkEnv中定义了ShuffleManager的类型，我当前使用的是Spark1.6版本，可见这时HashShuffleManager已经不是默认的ShuffleManager了。

接着进入org.apache.spark.shuffle.hash.HashShuffleManager中



```scala
  override def getReader[K, C](
      handle: ShuffleHandle,
      startPartition: Int,
      endPartition: Int,
      context: TaskContext): ShuffleReader[K, C] = {
    new BlockStoreShuffleReader(
      handle.asInstanceOf[BaseShuffleHandle[K, _, C]], startPartition, endPartition, context)
  }
```

getReader这个方法是对ShuffleManager这个接口中方法的实现，不需要多说继续看BlockStoreShuffleReader



BlockStoreShuffleReader.read

```scala
  override def read(): Iterator[Product2[K, C]] = {
    val blockFetcherItr = new ShuffleBlockFetcherIterator(
      context,
      blockManager.shuffleClient,
      blockManager,
      mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
      // Note: we use getSizeAsMb when no suffix is provided for backwards compatibility
      SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024)

    // Wrap the streams for compression based on configuration
    val wrappedStreams = blockFetcherItr.map { case (blockId, inputStream) =>
      blockManager.wrapForCompression(blockId, inputStream)
    }

    val ser = Serializer.getSerializer(dep.serializer)
    val serializerInstance = ser.newInstance()

    // Create a key/value iterator for each stream
    val recordIter = wrappedStreams.flatMap { wrappedStream =>
      // Note: the asKeyValueIterator below wraps a key/value iterator inside of a
      // NextIterator. The NextIterator makes sure that close() is called on the
      // underlying InputStream when all records have been read.
      serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
    }

    // Update the context task metrics for each record read.
    val readMetrics = context.taskMetrics.createShuffleReadMetricsForDependency()
    val metricIter = CompletionIterator[(Any, Any), Iterator[(Any, Any)]](
      recordIter.map(record => {
        readMetrics.incRecordsRead(1)
        record
      }),
      context.taskMetrics().updateShuffleReadMetrics())

    // An interruptible iterator must be used here in order to support task cancellation
    val interruptibleIter = new InterruptibleIterator[(Any, Any)](context, metricIter)

    val aggregatedIter: Iterator[Product2[K, C]] = if (dep.aggregator.isDefined) {
      if (dep.mapSideCombine) {
        // We are reading values that are already combined
        val combinedKeyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, C)]]
        dep.aggregator.get.combineCombinersByKey(combinedKeyValuesIterator, context)
      } else {
        // We don't know the value type, but also don't care -- the dependency *should*
        // have made sure its compatible w/ this aggregator, which will convert the value
        // type to the combined type C
        val keyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, Nothing)]]
        dep.aggregator.get.combineValuesByKey(keyValuesIterator, context)
      }
    } else {
      require(!dep.mapSideCombine, "Map-side combine without Aggregator specified!")
      interruptibleIter.asInstanceOf[Iterator[Product2[K, C]]]
    }

    // Sort the output if there is a sort ordering defined.
    dep.keyOrdering match {
      case Some(keyOrd: Ordering[K]) =>
        // Create an ExternalSorter to sort the data. Note that if spark.shuffle.spill is disabled,
        // the ExternalSorter won't spill to disk.
        val sorter =
          new ExternalSorter[K, C, C](context, ordering = Some(keyOrd), serializer = Some(ser))
        sorter.insertAll(aggregatedIter)
        context.taskMetrics().incMemoryBytesSpilled(sorter.memoryBytesSpilled)
        context.taskMetrics().incDiskBytesSpilled(sorter.diskBytesSpilled)
        context.internalMetricsToAccumulators(
          InternalAccumulator.PEAK_EXECUTION_MEMORY).add(sorter.peakMemoryUsedBytes)
        CompletionIterator[Product2[K, C], Iterator[Product2[K, C]]](sorter.iterator, sorter.stop())
      case None =>
        aggregatedIter
    }
  }
```

这一部分代码才是核心的东西。总结一下步骤:

- 首先获取一个迭代器ShuffleBlockFetcherIterator，它迭代的元素是blockId和这个block对应的读取流，很显然这个类就是实现reduce阶段数据读取的关键
- 将原始的数据流转化为反序列化的迭代器
- 





