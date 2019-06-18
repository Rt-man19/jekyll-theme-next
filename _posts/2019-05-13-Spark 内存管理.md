---

title: Spark 内存管理

categories:

- Spark

tags:

---

- StaticMemoryManager

  - getMaxExecutionMemory

  ```scala
  StaticMemoryManager.getMaxExecutionMemory(conf) {
        private def getMaxExecutionMemory(conf: SparkConf): Long = {
          val systemMaxMemory = conf.getLong("spark.testing.memory", Runtime.getRuntime.maxMemory)
  
          if (systemMaxMemory < MIN_MEMORY_BYTES) {
            throw new IllegalArgumentException(s"System memory $systemMaxMemory must " +
              s"be at least $MIN_MEMORY_BYTES. Please increase heap size using the --driver-memory " +
              s"option or spark.driver.memory in Spark configuration.")
          }
          if (conf.contains("spark.executor.memory")) {
            val executorMemory = conf.getSizeAsBytes("spark.executor.memory")
            if (executorMemory < MIN_MEMORY_BYTES) {
              throw new IllegalArgumentException(s"Executor memory $executorMemory must be at least " +
                s"$MIN_MEMORY_BYTES. Please increase executor memory using the " +
                s"--executor-memory option or spark.executor.memory in Spark configuration.")
            }
          }
          val memoryFraction = conf.getDouble("spark.shuffle.memoryFraction", 0.2)
          val safetyFraction = conf.getDouble("spark.shuffle.safetyFraction", 0.8)
          (systemMaxMemory * memoryFraction * safetyFraction).toLong
        }
  }
  ```

  > 首先去拿spark.testing.memory 这个参数，如果没有配置则使用运行时的最大内存
  > 然后判断,系统内存是否小于最小要求的内存， private val MIN_MEMORY_BYTES = 32 * 1024 * 1024   32M
  > true: 报错 "system memory $systemMaxMemory must be ast list $MIN_MEMORY_BYTES ...."
  > false: 继续往下走
  > 之后判断sparkConf里有没有设置spark.executor.memory这个参数
  > true: 把这个参数的值转成Bytes 然后和 MIN_MEMORY_BYTES比,是否小于MIN_MEMORY_BYTES
  >       true:  报错
  > false: 继续往下走
  > memoryFraction: shuffle可用的内存 20%
  > safetyFraction: shuffle时的安全系数 80%
  > 返回值: systemMaxMemory * memoryFraction * safetyFraction
  > 假设系统内存10G 则执行可用内存为 10 * 20% * 80% = 1.6G

  - getMaxStorageMemory

    ```scala
    StaticMemoryManager.getMaxStorageMemory(conf) {
        val systemMaxMemory = conf.getLong("spark.testing.memory", Runtime.getRuntime.maxMemory)
        val memoryFraction = conf.getDouble("spark.storage.memoryFraction", 0.6)
        val safetyFraction = conf.getDouble("spark.storage.safetyFraction", 0.9)
        (systemMaxMemory * memoryFraction * safetyFraction).toLong
    }
    ```

    > 首先去拿spark.testing.memory, 如果没有拿不到则使用运行时的最大内存
    > memoryFraction: 存储可用的内存系数 60%
    > safetyFraction: 存储时的安全系数: 90%
    > 返回值 systemMaxMemory * memoryFraction * safetyFraction
    > 假设系统内存10G 则存储可用内存为 10 * 60% * 90% = 5.4G

- 统一内存管理(新)

  - getMaxMemory

  ```scala
   private def getMaxMemory(conf: SparkConf): Long = {
      val systemMemory = conf.getLong("spark.testing.memory", Runtime.getRuntime.maxMemory)
      val reservedMemory = conf.getLong("spark.testing.reservedMemory",
        if (conf.contains("spark.testing")) 0 else RESERVED_SYSTEM_MEMORY_BYTES)
      val minSystemMemory = (reservedMemory * 1.5).ceil.toLong
      if (systemMemory < minSystemMemory) {
        throw new IllegalArgumentException(s"System memory $systemMemory must " +
          s"be at least $minSystemMemory. Please increase heap size using the --driver-memory " +
          s"option or spark.driver.memory in Spark configuration.")
      }
      // SPARK-12759 Check executor memory to fail fast if memory is insufficient
      if (conf.contains("spark.executor.memory")) {
        val executorMemory = conf.getSizeAsBytes("spark.executor.memory")
        if (executorMemory < minSystemMemory) {
          throw new IllegalArgumentException(s"Executor memory $executorMemory must be at least " +
            s"$minSystemMemory. Please increase executor memory using the " +
            s"--executor-memory option or spark.executor.memory in Spark configuration.")
        }
      }
      val usableMemory = systemMemory - reservedMemory
      val memoryFraction = conf.getDouble("spark.memory.fraction", 0.6)
      (usableMemory * memoryFraction).toLong
    }
  ```

  >   RESERVED_SYSTEM_MEMORY_BYTES 预留内存大小: private val RESERVED_SYSTEM_MEMORY_BYTES = 300 * 1024 * 1024 300M 
  >   首先拿spark.testing.memory,拿不到就使用运行时的最大内存
  >   然后拿预留内存大小spark.testing.reservedMemory,如果拿不到，则判断是否配置了spark.testing true: 0,false: RESERVED_SYSTEM_MEMORY_BYTES
  >   默认这两个参数是都没有配置的，所以预留大小是300M
  >   定义系统最小内存 = 预留内存 * 1.5
  >   如果系统内存比最小内存还小 报错
  >   如果配置了spark.executor.memory 则拿到执行内存，然后和最小内存比较，如果执行内存比最小内存还小 报错
  >   定义一共可用内存 = 运行时最大内存  -  预留内存 
  >   拿到内存系数
  >   返回最大可用内存 usableMemory * memoryFraction
  >   假如系统内存以后10G 则最大可用内存为 (10G - 300M) * 0.6

     ```scala
  def apply(conf: SparkConf, numCores: Int): UnifiedMemoryManager = {
    val maxMemory = getMaxMemory(conf)
    new UnifiedMemoryManager(
      conf,
      maxHeapMemory = maxMemory,
      onHeapStorageRegionSize =
        (maxMemory * conf.getDouble("spark.memory.storageFraction", 0.5)).toLong,
      numCores = numCores)
  }
     ```

>  存储可用最大内存: 最大内存的50% (10G - 300M) * 0.6 * 0.5 剩下的就是执行端的内存
>
> spark1.6.0 默认 spark.memory.fraction 是0.75
> spark2.4.0 默认 spark.memory.fraction 是0.6
>
>
> 哪些操作会使用执行内存哪些使用存储内存?
> https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/index.html
>
> 执行内存主要用来存储任务在执行 Shuffle 时占用的内存，Shuffle 是按照一定规则对 RDD 数据重新分区的过程，我们来看 Shuffle 的 Write 和 Read 两阶段对执行内存的使用：
>
> Shuffle Write
>   若在 map 端选择普通的排序方式，会采用 ExternalSorter 进行外排，在内存中存储数据时主要占用堆内执行空间。
>   若在 map 端选择 Tungsten 的排序方式，则采用 ShuffleExternalSorter 直接对以序列化形式存储的数据排序，在内存中存储数据时可以占用堆外或堆内执行空间，取决于用户是否开启了堆外内存以及堆外执行内存是否足够。
> Shuffle Read
>   在对 reduce 端的数据进行聚合时，要将数据交给 Aggregator 处理，在内存中存储数据时占用堆内执行空间。
>   如果需要进行最终结果排序，则要将再次将数据交给 ExternalSorter 处理，占用堆内执行空间。