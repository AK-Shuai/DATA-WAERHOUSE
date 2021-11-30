# Spark 性能调优

##  常规性能调优
### 常规性能调优一：最优资源配置
Spark 性能调优的第一步，就是为任务分配更多的资源，在一定范围内，增加资源的分配与性能的提升是成正比的，实现了最优的资源配置后（其实就是没钱提升硬件后），在此基础上再考虑进行后面论述的性能调优策略。

资源的分配在使用脚本提交 Spark 任务时进行指定，标准的 Spark 任务提交脚本如代码清单
标准 Spark 提交脚本：
```
/usr/opt/modules/spark/bin/spark-submit \
--class com.test.spark.Analysis \
--num-executors 80 \
--driver-memory 6g \
--executor-memory 6g \
--executor-cores 3 \
/usr/opt/modules/spark/jar/spark.jar \
```

可以进行分配的资源如表 

名称 | 说明
---|:---:
--num-executors | 配置 Executor 的数量
--driver-memory  | 配置 Driver 内存（影响不大）
--executor-memory | 配置每个 Executor 的内存大小
--executor-cores  | 配置每个 Executor 的 CPU core 数量

调节原则：尽量将任务分配的资源调节到可以使用的资源的最大限度。

对于具体资源的分配，我们分别讨论 Spark 的两种 Cluster 运行模式：

第一种是 Spark Standalone 模式，你在提交任务前，一定知道或者可以从运维部门获取到你可以使用的资源情况，在编写 submit 脚本的时候，就根据可用的资源情况进行资源的分配，比如说集群有 15 台机器，每台机器为 8G 内存，2 个 CPU core，那么就指定 15 个 Executor，每个 Executor 分配 8G 内存，2 个 CPU core。

第二种是 Spark Yarn 模式，由于 Yarn 使用资源队列进行资源的分配和调度，在写 submit 脚本的时候，就根据 Spark 作业要提交到的资源队列，进行资源的分配，比如资源队列有 400G 内存，100 个 CPU core，那么指定 50 个 Executor，每个 Executor 分配 8G 内存，2 个 CPU core。

各项资源进行了调节后，得到的性能提升如表：
名称 | 解析
---|:---:
增加 Executor 个数 	| 在资源允许的情况下，增加 Executor 的个数可以提高执行 task 的并行度。比如有 4 个 Executor，每个 Executor 有 2 个 CPU core，那么可以并行执行 8 个 task，如果将 Executor 的个数增加到 8 个（资源允许的情况下），那么可以并行执行 16 个 task，此时的并行能力提升了一倍。
增加每个 Executor 的 CPU core 个数 	 | 在资源允许的情况下，增加每个 Executor 的 Cpu core 个数，可以提高执行 task 的并行度。比如有 4 个 Executor，每个 Executor 有 2 个 CPU core，那么可以并行执行 8 个 task，如果将每个 Executor 的 CPU core 个数增加到 4 个（资源允许的情况下），那么可以并行执行 16 个 task，此时的并行能力提升了一倍。
增加每个 Executor 的内存量 	 | 在资源允许的情况下，增加每个 Executor 的内存量以后，对性能的提升有三点： 1. 可以缓存更多的数据（即对 RDD 进行 cache ），写入磁盘的数据相应减少，甚至可以不写入磁盘，减少了可能的磁盘 IO； 2. 可以为 shuffle 操作提供更多内存，即有更多空间来存放 reduce 端拉取的数据，写入磁盘的数据相应减少，甚至可以不写入磁盘，减少了可能的磁盘 IO； 3. 可以为 task 的执行提供更多内存，在 task 的执行过程中可能创建很多对象，内存较小时会引发频繁的 GC，增加内存后，可以避免频繁的 GC，提升整体性能。

补充：生产环境 Spark submit 脚本配置

```
/usr/local/spark/bin/spark-submit \
--class com.test.spark.WordCount \
--num-executors 80 \
--driver-memory 6g \
--executor-memory 6g \
--executor-cores 3 \
--master yarn-cluster \
--queue root.default \
--conf spark.yarn.executor.memoryOverhead=2048 \
--conf spark.core.connection.ack.wait.timeout=300 \
/usr/local/spark/spark.jar
```

--num-executors：50~100

--driver-memory：1G~5G

--executor-memory：6G~10G

--executor-cores：3

--master：实际生产环境一定使用 yarn-cluster

**实际场景举例**：

上面说的这些 submit 提交，相信大家都需要做，但看完这小结后你在提交任务时会进行最优配置了吗？没有实际工作过的同学肯定还是不会，因为这些说白了还是理论，对是对，但实际生产中不能生搬硬套。

比如工作中一般会用到 CDH 的公司，比如小编的，一般会在 oozie 上通过建立 workflow 进行任务的调度，那么对于跑 Spark 的任务，其参数配置就要视业务而定了。上面说到 --driver-memory 这个配置不必太大，一般都会比 --executor-memory 小，因为后者才是进行计算的地方。但小编的公司 --driver-memory 居然会设置成 --executor-memory 的两倍。为啥？因为我们有的业务需要拉到 driver 端的缓存中，小就爆了，别问我为什么拉到 driver 端，这多傻啊，业务场景不一样，需要的计算方式也不一样。当然我不否定会有更优的方式处理，这里只是说理论不能偏离实际。

**配置额外补充：动态资源分配**
```
--conf spark.shuffle.service.enabled=true //启用 External shuffle Service 服务
--conf spark.dynamicAllocation.enabled=true //开启动态资源分配
--conf spark.dynamicAllocation.minExecutors=1 //每个 Application 最小分配的 executor 数
--conf spark.dynamicAllocation.maxExecutors=40 //每个 Application 最大并发分配的 executor 数
```

开启动态分配策略后，application 会在 task 因没有足够资源被挂起的时候去动态申请资源，这种情况意味着该 application 现有的 executor 无法满足所有 task 并行运行。spark 一轮一轮的申请资源，当有 task 挂起或等待 spark.dynamicAllocation.schedulerBacklogTimeout （默认 1s）时间的时候，会开始动态资源分配；之后会每隔 spark.dynamicAllocation.sustainedSchedulerBacklogTimeout （默认 1s）时间申请一次，直到申请到足够的资源。每次申请的资源量是指数增长的，即 1,2,4,8 等。

之所以采用指数增长，出于两方面考虑：其一，开始申请的少是考虑到可能 application 会马上得到满足；其次要成倍增加，是为了防止 application 需要很多资源，而该方式可以在很少次数的申请之后得到满足。

### 常规性能调优二：RDD 优化

#### RDD 持久化

在 Spark 中，当多次对同一个 RDD 执行算子操作时，每一次都会对这个 RDD 以之前的父 RDD 重新计算一次，这种情况是必须要避免的，对同一个 RDD 的重复计算是对资源的极大浪费，因此，必须对多次使用的 RDD 进行持久化，通过持久化将公共 RDD 的数据缓存到内存/磁盘中，之后对于公共 RDD 的计算都会从内存/磁盘中直接获取 RDD 数据。

对于 RDD 的持久化，有两点需要说明：

第一，RDD 的持久化是可以进行序列化的，当内存无法将 RDD 的数据完整的进行存放的时候，可以考虑使用序列化的方式减小数据体积，将数据完整存储在内存中。

第二，如果对于数据的可靠性要求很高，并且内存充足，可以使用副本机制，对 RDD 数据进行持久化。当持久化启用了复本机制时，对于持久化的每个数据单元都存储一个副本，放在其他节点上面，由此实现数据的容错，一旦一个副本数据丢失，不需要重新计算，还可以使用另外一个副本。

#### RDD 尽可能早的 filter 操作
获取到初始 RDD 后，应该考虑尽早地过滤掉不需要的数据，进而减少对内存的占用，从而提升 Spark 作业的运行效率。

#### 常规性能调优三：并行度调节
Spark 作业中的并行度指各个 stage 的 task 的数量。

如果并行度设置不合理而导致并行度过低，会导致资源的极大浪费，例如，20 个 Executor，每个 Executor 分配 3 个 CPU core，而 Spark 作业有 40 个 task，这样每个 Executor 分配到的 task 个数是 2 个，这就使得每个 Executor 有一个 CPU core 空闲，导致资源的浪费。

理想的并行度设置，应该是让并行度与资源相匹配，简单来说就是在资源允许的前提下，并行度要设置的尽可能大，达到可以充分利用集群资源。合理的设置并行度，可以提升整个 Spark 作业的性能和运行速度。

Spark 官方推荐，task 数量应该设置为 Spark 作业总 CPU core 数量的 2~3 倍。之所以没有推荐 task 数量与 CPU core 总数相等，是因为 task 的执行时间不同，有的 task 执行速度快而有的 task 执行速度慢，如果 task 数量与 CPU core 总数相等，那么执行快的 task 执行完成后，会出现 CPU core 空闲的情况。如果 task 数量设置为 CPU core 总数的 2~3 倍，那么一个 task 执行完毕后，CPU core 会立刻执行下一个 task，降低了资源的浪费，同时提升了 Spark 作业运行的效率。

Spark 作业并行度设置：
```
val conf = new SparkConf().set("spark.default.parallelism", "500")
```

#### 常规性能调优四：广播大变量

默认情况下，task 中的算子中如果使用了外部的变量，每个 task 都会获取一份变量的复本，这就造成了内存的极大消耗。一方面，如果后续对 RDD 进行持久化，可能就无法将 RDD 数据存入内存，只能写入磁盘，磁盘 IO 将会严重消耗性能；另一方面，task 在创建对象的时候，也许会发现堆内存无法存放新创建的对象，这就会导致频繁的 GC，GC 会导致工作线程停止，进而导致 Spark 暂停工作一段时间，严重影响 Spark 性能。

假设当前任务配置了 20 个 Executor，指定 500 个 task，有一个 20M 的变量被所有 task 共用，此时会在 500 个 task 中产生 500 个副本，耗费集群 10G 的内存，如果使用了广播变量， 那么每个 Executor 保存一个副本，一共消耗 400M 内存，内存消耗减少了 5 倍。

广播变量在每个 Executor 保存一个副本，此 Executor 的所有 task 共用此广播变量，这让变量产生的副本数量大大减少。

在初始阶段，广播变量只在 Driver 中有一份副本。task 在运行的时候，想要使用广播变量中的数据，此时首先会在自己本地的 Executor 对应的 BlockManager 中尝试获取变量，如果本地没有，BlockManager 就会从 Driver 或者其他节点的 BlockManager 上远程拉取变量的复本，并由本地的 BlockManager 进行管理；之后此 Executor 的所有 task 都会直接从本地的 BlockManager 中获取变量。

注意：这里说的大变量也是有限制的，如果变量太大，超过了 executor 内存，那么这个变量时放不进去的，就会 OOM。

#### 常规性能调优五：Kryo 序列化

默认情况下，Spark 使用 Java 的序列化机制。Java 的序列化机制使用方便，不需要额外的配置，在算子中使用的变量实现 Serializable 接口即可，但是，Java 序列化机制的效率不高，序列化速度慢并且序列化后的数据所占用的空间依然较大。

Kryo 序列化机制比 Java 序列化机制性能提高 10 倍左右，Spark 之所以没有默认使用 Kryo 作为序列化类库，是因为它不支持所有对象的序列化，同时 Kryo 需要用户在使用前注册需要序列化的类型，不够方便，但从 Spark 2.0.0 版本开始，简单类型、简单类型数组、字符串类型的 Shuffling RDDs 已经默认使用 Kryo 序列化方式了。
Kryo 序列化机制配置代码：
```
public class MyKryoRegistrator implements KryoRegistrator
{
 @Override
 public void registerClasses(Kryo kryo)
 {
  kryo.register(StartupReportLogs.class);
 }
}
```

Kryo 序列化机制配置代码：
```
//创建 SparkConf 对象
val conf = new SparkConf().setMaster(…).setAppName(…)
//使用 Kryo 序列化库，如果要使用 Java 序列化库，需要把该行屏蔽掉
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
//在 Kryo 序列化库中注册自定义的类集合，如果要使用 Java 序列化库，需要把该行屏蔽掉
conf.set("spark.kryo.registrator", "test.com.MyKryoRegistrator")
```

#### 常规性能调优六：调节本地化等待时长

Spark 作业运行过程中，Driver 会对每一个 stage 的 task 进行分配。根据 Spark 的 task 分配算法，Spark 希望 task 能够运行在它要计算的数据所在的节点（数据本地化思想），这样就可以避免数据的网络传输。通常来说，task 可能不会被分配到它处理的数据所在的节点，因为这些节点可用的资源可能已经用尽，此时，Spark 会等待一段时间，默认 3s，如果等待指定时间后仍然无法在指定节点运行，那么会自动降级，尝试将 task 分配到比较差的本地化级别所对应的节点上，比如将 task 分配到离它要计算的数据比较近的一个节点，然后进行计算，如果当前级别仍然不行，那么继续降级。

当 task 要处理的数据不在 task 所在节点上时，会发生数据的传输。task 会通过所在节点的 BlockManager 获取数据，BlockManager 发现数据不在本地时，会通过网络传输组件从数据所在节点的 BlockManager 处获取数据。

网络传输数据的情况是我们不愿意看到的，大量的网络传输会严重影响性能，因此，我们希望通过调节本地化等待时长，如果在等待时长这段时间内，目标节点处理完成了一部分 task，那么当前的 task 将有机会得到执行，这样就能够改善 Spark 作业的整体性能。 

Spark 本地化等级：
名称 	| 解析
---|:--:
PROCESS_LOCAL 	| 进程本地化，task 和数据在同一个 Executor 中，性能最好。
NODE_LOCAL 	| 节点本地化，task 和数据在同一个节点中，但是 task 和数据不在同一个 Executor 中，数据需要在进程间进行传输。
RACK_LOCAL 	| 机架本地化，task 和数据在同一个机架的两个节点上，数据需要通过网络在节点之间进行传输。
NO_PREF 	| 对于 task 来说，从哪里获取都一样，没有好坏之分。
ANY 	| task 和数据可以在集群的任何地方，而且不在一个机架中，性能最差。

在 Spark 项目开发阶段，可以使用 client 模式对程序进行测试，此时，可以在本地看到比较全的日志信息，日志信息中有明确的 task 数据本地化的级别，如果大部分都是 PROCESSLOCAL，那么就无需进行调节，但是如果发现很多的级别都是 NODELOCAL、ANY，那么需要对本地化的等待时长进行调节，通过延长本地化等待时长，看看 task 的本地化级别有没有提升，并观察 Spark 作业的运行时间有没有缩短。

注意，过犹不及，不要将本地化等待时长延长地过长，导致因为大量的等待时长，使得 Spark 作业的运行时间反而增加了。

Spark 本地化等待时长设置示例：
```
val conf = new SparkConf().set("spark.locality.wait", "6") 
```

##  算子调优

### 算子调优一：mapPartitions
普通的 map 算子对 RDD 中的每一个元素进行操作，而 mapPartitions 算子对 RDD 中每一个分区进行操作。如果是普通的 map 算子，假设一个 partition 有 1 万条数据，那么 map 算子中的 function 要执行 1 万次，也就是对每个元素进行操作。

<div align=center><img src="" width="400"></div>
如果是 mapPartition 算子，由于一个 task 处理一个 RDD 的 partition，那么一个 task 只会执行一次 function，function 一次接收所有的 partition 数据，效率比较高。
<div align=center><img src="" width="400"></div>

比如，当要把 RDD 中的所有数据通过 JDBC 写入数据，如果使用 map 算子，那么需要对 RDD 中的每一个元素都创建一个数据库连接，这样对资源的消耗很大，如果使用 mapPartitions 算子，那么针对一个分区的数据，只需要建立一个数据库连接。

mapPartitions 算子也存在一些缺点：对于普通的 map 操作，一次处理一条数据，如果在处理了 2000 条数据后内存不足，那么可以将已经处理完的 2000 条数据从内存中垃圾回收掉；但是如果使用 mapPartitions 算子，但数据量非常大时，function 一次处理一个分区的数据，如果一旦内存不足，此时无法回收内存，就可能会 OOM，即内存溢出。

因此，mapPartitions 算子适用于数据量不是特别大的时候，此时使用 mapPartitions 算子对性能的提升效果还是不错的。（当数据量很大的时候，一旦使用 mapPartitions 算子，就会直接 OOM）

在项目中，应该首先估算一下 RDD 的数据量、每个 partition 的数据量，以及分配给每个 Executor 的内存资源，如果资源允许，可以考虑使用 mapPartitions 算子代替 map。

### 算子调优二：foreachPartition 优化数据库操作

在生产环境中，通常使用 foreachPartition 算子来完成数据库的写入，通过 foreachPartition 算子的特性，可以优化写数据库的性能。

如果使用 foreach 算子完成数据库的操作，由于 foreach 算子是遍历 RDD 的每条数据，因此，每条数据都会建立一个数据库连接，这是对资源的极大浪费，因此，对于写数据库操作，我们应当使用 foreachPartition 算子。

与 mapPartitions 算子非常相似，foreachPartition 是将 RDD 的每个分区作为遍历对象，一次处理一个分区的数据，也就是说，如果涉及数据库的相关操作，一个分区的数据只需要创建一次数据库连接，如图
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/foreachPartition%E7%AE%97%E5%AD%90.jpg" width="400"></div>

使用了 foreachPartition 算子后，可以获得以下的性能提升：

1. 对于我们写的 function 函数，一次处理一整个分区的数据；

2. 对于一个分区内的数据，创建唯一的数据库连接；

3. 只需要向数据库发送一次 SQL 语句和多组参数；

在生产环境中，全部都会使用 foreachPartition 算子完成数据库操作。foreachPartition 算子存在一个问题，与 mapPartitions 算子类似，如果一个分区的数据量特别大，可能会造成 OOM，即内存溢出。

### 算子调优三：filter 与 coalesce 的配合使用

在 Spark 任务中我们经常会使用 filter 算子完成 RDD 中数据的过滤，在任务初始阶段，从各个分区中加载到的数据量是相近的，但是一旦进过 filter 过滤后，每个分区的数据量有可能会存在较大差异，如图

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/%E5%88%86%E5%8C%BA%E6%95%B0%E6%8D%AE%E8%BF%87%E6%BB%A4%E7%BB%93%E6%9E%9C%E5%9B%BE.jpg" width="400"></div>

我们可以发现两个问题：

1. 每个 partition 的数据量变小了，如果还按照之前与 partition 相等的 task 个数去处理当前数据，有点浪费 task 的计算资源；

2. 每个 partition 的数据量不一样，会导致后面的每个 task 处理每个 partition 数据的时候，每个 task 要处理的数据量不同，这很有可能导致数据倾斜问题。

第二个分区的数据过滤后只剩 100 条，而第三个分区的数据过滤后剩下 800 条，在相同的处理逻辑下，第二个分区对应的 task 处理的数据量与第三个分区对应的 task 处理的数据量差距达到了 8 倍，这也会导致运行速度可能存在数倍的差距，这也就是数据倾斜问题。

针对上述的两个问题，我们分别进行分析：

1. 针对第一个问题，既然分区的数据量变小了，我们希望可以对分区数据进行重新分配，比如将原来 4 个分区的数据转化到 2 个分区中，这样只需要用后面的两个 task 进行处理即可，避免了资源的浪费。

2. 针对第二个问题，解决方法和第一个问题的解决方法非常相似，对分区数据重新分配，让每个 partition 中的数据量差不多，这就避免了数据倾斜问题。

那么具体应该如何实现上面的解决思路？我们需要 coalesce 算子。

repartition 与 coalesce 都可以用来进行重分区，其中 repartition 只是 coalesce 接口中 shuffle 为 true 的简易实现，coalesce 默认情况下不进行 shuffle，但是可以通过参数进行设置。


假设我们希望将原本的分区个数 A 通过重新分区变为 B，那么有以下几种情况：

1. A > B（多数分区合并为少数分区）

   ① A 与 B 相差值不大（注：这里的不大指比如 100 个分区到 80 个）此时使用 coalesce 即可，无需 shuffle 过程。

   ② A 与 B 相差值很大（注：这里比如 100 个到 2 个）

   此时可以使用 coalesce 并且不启用 shuffle 过程，但是会导致合并过程性能低下，所以推荐设置 coalesce 的第二个参数为 true，即启动 shuffle 过程。

小结：若 filter 后每个分区间数据量差不多，总量减少，那么用 coalesce 即可，但注意不能减到太小，否则会数据倾斜。若 filter 后分区间数据量差异很大，那么就要重新 shuffle 了，用 reparation，目的是打散数据重新均匀分布到各个分区中。coalesce 默认状态下即便传入的分区数值变大，真实分区数是不会增大的。

2. A < B（少数分区分解为多数分区）

   此时使用 repartition 即可，如果使用 coalesce 需要将 shuffle 设置为 true，否则 coalesce 无效。

   我们可以在 filter 操作之后，使用 coalesce 算子针对每个 partition 的数据量各不相同的情况，压缩 partition 的数量，而且让每个 partition 的数据量尽量均匀紧凑，以便于后面的 task 进行计算操作，在某种程度上能够在一定程度上提升性能。

注意：local 模式是进程内模拟集群运行，已经对并行度和分区数量有了一定的内部优化，因此不用去设置并行度和分区数量。

### 算子调优四：repartition 解决 SparkSQL 低并行度问题

Spark Shuffle 过程中，reduce task 拉取属于自己的数据时，如果因为网络异常等原因导致失败会自动进行重试，在一次失败后，会等待一定的时间间隔再进行重试，可以通过加大间隔时长（比如 60s），以增加 shuffle 操作的稳定性。

reduce 端拉取数据等待间隔可以通过 spark.shuffle.io.retryWait 参数进行设置，默认值为 5s，该参数的设置方法如代码清单所示：

```
val conf = new SparkConf().set("spark.shuffle.io.retryWait", "60s") 
```

### Shuffle 调优五：调节 SortShuffle 排序操作阈值

对于 SortShuffleManager，如果 shuffle reduce task 的数量小于某一阈值则 shuffle write 过程中不会进行排序操作，而是直接按照未经优化的 HashShuffleManager 的方式去写数据，但是最后会将每个 task 产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。

当你使用 SortShuffleManager 时，如果的确不需要排序操作，那么建议将这个参数调大一些，大于 shuffle read task 的数量，那么此时 map-side 就不会进行排序了，减少了排序的性能开销，但是这种方式下，依然会产生大量的磁盘文件，因此 shuffle write 性能有待提高。

SortShuffleManager 排序操作阈值的设置可以通过 spark.shuffle.sort. bypassMergeThreshold 这一参数进行设置，默认值为 200：

reduce 端拉取数据等待间隔配置：
```
val conf = new SparkConf().set("spark.shuffle.sort.bypassMergeThreshold", "400") 
```

## JVM 调优

对于 JVM 调优，首先应该明确，(major)full gc/minor gc，都会导致 JVM 的工作线程停止工作，即 stop the world。
### JVM 调优一：降低 cache 操作的内存占比

### #静态内存管理机制

根据 Spark 静态内存管理机制，堆内存被划分为了两块，Storage 和 Execution。Storage 主要用于缓存 RDD 数据和 broadcast 数据，Execution 主要用于缓存在 shuffle 过程中产生的中间数据，Storage 占系统内存的 60%，Execution 占系统内存的 20%，并且两者完全独立。

在一般情况下，Storage 的内存都提供给了 cache 操作，但是如果在某些情况下 cache 操作内存不是很紧张，而 task 的算子中创建的对象很多，Execution 内存又相对较小，这回导致频繁的 minor gc，甚至于频繁的 full gc，进而导致 Spark 频繁的停止工作，性能影响会很大。

在 Spark UI 中可以查看每个 stage 的运行情况，包括每个 task 的运行时间、gc 时间等等，如果发现 gc 太频繁，时间太长，就可以考虑调节 Storage 的内存占比，让 task 执行算子函数式，有更多的内存可以使用。

Storage 内存区域可以通过 spark.storage.memoryFraction 参数进行指定，默认为 0.6，即 60%，可以逐级向下递减，如代码清单 :

Storage 内存占比设置:
```
val conf = new SparkConf().set("spark.storage.memoryFraction", "0.4")
```
### 统一内存管理机制

根据 Spark 统一内存管理机制，堆内存被划分为了两块，Storage 和 Execution。Storage 主要用于缓存数据，Execution 主要用于缓存在 shuffle 过程中产生的中间数据，两者所组成的内存部分称为统一内存，Storage 和 Execution 各占统一内存的 50%，由于动态占用机制的实现，shuffle 过程需要的内存过大时，会自动占用 Storage 的内存区域，因此无需手动进行调节。

### JVM 调优二：调节 Executor 堆外内存

Executor 的堆外内存主要用于程序的共享库、Perm Space、 线程 Stack 和一些 Memory mapping 等, 或者类 C 方式 allocate object。

有时，如果你的 Spark 作业处理的数据量非常大，达到几亿的数据量，此时运行 Spark 作业会时不时地报错，例如 shuffle output file cannot find，executor lost，task lost，out of memory 等，这可能是 Executor 的堆外内存不太够用，导致 Executor 在运行的过程中内存溢出。

stage 的 task 在运行的时候，可能要从一些 Executor 中去拉取 shuffle map output 文件，但是 Executor 可能已经由于内存溢出挂掉了，其关联的 BlockManager 也没有了，这就可能会报出 shuffle output file cannot find，executor lost，task lost，out of memory 等错误，此时，就可以考虑调节一下 Executor 的堆外内存，也就可以避免报错，与此同时，堆外内存调节的比较大的时候，对于性能来讲，也会带来一定的提升。

默认情况下，Executor 堆外内存上限大概为 300 多 MB，在实际的生产环境下，对海量数据进行处理的时候，这里都会出现问题，导致 Spark 作业反复崩溃，无法运行，此时就会去调节这个参数，到至少 1G，甚至于 2G、4G。

Executor 堆外内存的配置需要在 spark-submit 脚本里配置：

Executor 堆外内存配置
```
--conf spark.yarn.executor.memoryOverhead=2048
```
以上参数配置完成后，会避免掉某些 JVM OOM 的异常问题，同时，可以提升整体 Spark 作业的性能。

### JVM 调优三：调节连接等待时长

在 Spark 作业运行过程中，Executor 优先从自己本地关联的 BlockManager 中获取某份数据，如果本地 BlockManager 没有的话，会通过 TransferService 远程连接其他节点上 Executor 的 BlockManager 来获取数据。

如果 task 在运行过程中创建大量对象或者创建的对象较大，会占用大量的内存，这回导致频繁的垃圾回收，但是垃圾回收会导致工作现场全部停止，也就是说，垃圾回收一旦执行，Spark 的 Executor 进程就会停止工作，无法提供相应，此时，由于没有响应，无法建立网络连接，会导致网络连接超时。

在生产环境下，有时会遇到 file not found、file lost 这类错误，在这种情况下，很有可能是 Executor 的 BlockManager 在拉取数据的时候，无法建立连接，然后超过默认的连接等待时长 60s 后，宣告数据拉取失败，如果反复尝试都拉取不到数据，可能会导致 Spark 作业的崩溃。这种情况也可能会导致 DAGScheduler 反复提交几次 stage，TaskScheduler 反复提交几次 task，大大延长了我们的 Spark 作业的运行时间。

此时，可以考虑调节连接的超时时长，连接等待时长需要在 spark-submit 脚本中进行设置，设置方式如代码清单：

连接等待时长配置：
```
--conf spark.core.connection.ack.wait.timeout=300
```
调节连接等待时长后，通常可以避免部分的 XX 文件拉取失败、XX 文件 lost 等报错。