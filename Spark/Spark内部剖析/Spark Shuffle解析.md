# Spark Shuffle 解析
## Shuffle 的核心要点
### ShuffleMapStage 与 ResultStage
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/ShuffleMapStage%E4%B8%8EResultStage%E7%BB%93%E6%9E%84.png" width="400"></div>

划分 stage 时，最后一个 stage 称为 finalStage，它本质上是一个 ResultStage 对象，前面的所有 stage 被称为 ShuffleMapStage。

ShuffleMapStage 的结束伴随着 shuffle 文件的写磁盘。

ResultStage 基本上对应代码中的 action 算子，即将一个函数应用在 RDD 的各个 partition 的数据集上，意味着一个 job 的运行结束。

### Shuffle 中的任务个数
我们知道，Spark Shuffle 分为 map 阶段和 reduce 阶段，或者称之为 ShuffleRead 阶段和 ShuffleWrite 阶段，那么对于一次 Shuffle，map 过程和 reduce 过程都会由若干个 task 来执行，那么 map task 和 reduce task 的数量是如何确定的呢？

假设 Spark 任务从 HDFS 中读取数据，那么初始 RDD 分区个数由该文件的 split 个数决定，也就是一个 split 对应生成的 RDD 的一个 partition，我们假设初始 partition 个数为 N。

初始 RDD 经过一系列算子计算后（假设没有执行 repartition 和 coalesce 算子进行重分区，则分区个数不变，仍为 N，如果经过重分区算子，那么分区个数变为 M），我们假设分区个数不变，当执行到 Shuffle 操作时，map 端的 task 个数和 partition 个数一致，即 map task 为 N 个。

reduce 端的 stage 默认取 spark.default.parallelism 这个配置项的值作为分区数，如果没有配置，则以 map 端的最后一个 RDD 的分区数作为其分区数（也就是 N），那么分区数就决定了 reduce 端的 task 的个数。

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/spark.default.parallelism%E5%8F%82%E6%95%B0%E5%9B%BE.jpg" width="400"></div>

### reduce 端数据的读取

根据 stage 的划分我们知道，map 端 task 和 reduce 端 task 不在相同的 stage 中，map task 位于 ShuffleMapStage，reduce task 位于 ResultStage，map task 会先执行，那么后执行的 reduce task 如何知道从哪里去拉取 map task 落盘后的数据呢？

reduce 端的数据拉取过程如下：
1. map task 执行完毕后会将计算状态以及磁盘小文件位置等信息封装到 MapStatus 对象中，然后由本进程中的 MapOutPutTrackerWorker 对象将 mapStatus 对象发送给 Driver 进程的 MapOutPutTrackerMaster 对象；

2. 在 reduce task 开始执行之前会先让本进程中的 MapOutputTrackerWorker 向 Driver 进程中的 MapoutPutTrakcerMaster 发动请求，请求磁盘小文件位置信息；

3. 当所有的 Map task 执行完毕后，Driver 进程中的 MapOutPutTrackerMaster 就掌握了所有的磁盘小文件的位置信息。此时 MapOutPutTrackerMaster 会告诉 MapOutPutTrackerWorker 磁盘小文件的位置信息；

4. 完成之前的操作之后，由 BlockTransforService 去 Executor0 所在的节点拉数据，默认会启动五个子线程。每次拉取的数据量不能超过 48M（reduce task 每次最多拉取 48M 数据，将拉来的数据存储到 Executor 内存的 20%内存中）。

## HashShuffle 解析

以下的讨论都假设每个 Executor 有 1 个 CPU core。
### 未经优化的 HashShuffleManager

shuffle write 阶段，主要就是在一个 stage 结束计算之后，为了下一个 stage 可以执行 shuffle 类的算子（比如 reduceByKey），而将每个 task 处理的数据按 key 进行“划分”。所谓“划分”，就是对相同的 key 执行 hash 算法，从而将相同 key 都写入同一个磁盘文件中，而每一个磁盘文件都只属于下游 stage 的一个 task。在将数据写入磁盘之前，会先将数据写入内存缓冲中，当内存缓冲填满之后，才会溢写到磁盘文件中去。

下一个 stage 的 task 有多少个，当前 stage 的每个 task 就要创建多少份磁盘文件。比如下一个 stage 总共有 100 个 task，那么当前 stage 的每个 task 都要创建 100 份磁盘文件。如果当前 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 task，那么每个 Executor 上总共就要创建 500 个磁盘文件，所有 Executor 上会创建 5000 个磁盘文件。由此可见，未经优化的 shuffle write 操作所产生的磁盘文件的数量是极其惊人的。

shuffle read 阶段，通常就是一个 stage 刚开始时要做的事情。此时该 stage 的每一个 task 就需要将上一个 stage 的计算结果中的所有相同 key，从各个节点上通过网络都拉取到自己所在的节点上，然后进行 key 的聚合或连接等操作。由于 shuffle write 的过程中，map task 给下游 stage 的每个 reduce task 都创建了一个磁盘文件，因此 shuffle read 的过程中，每个 reduce task 只要从上游 stage 的所有 map task 所在节点上，拉取属于自己的那一个磁盘文件即可。

shuffle read 的拉取过程是一边拉取一边进行聚合的。每个 shuffle read task 都会有一个自己的 buffer 缓冲，每次都只能拉取与 buffer 缓冲相同大小的数据，然后通过内存中的一个 Map 进行聚合等操作。聚合完一批数据后，再拉取下一批数据，并放到 buffer 缓冲中进行聚合操作。以此类推，直到最后将所有数据到拉取完，并得到最终的结果。

未优化的 HashShuffleManager 工作原理如图

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/%E6%9C%AA%E4%BC%98%E5%8C%96%E7%9A%84HashShuffleManager%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%9B%BE.jpg" width="400"></div>

### 优化后的 HashShuffleManager
为了优化 HashShuffleManager 我们可以设置一个参数，spark.shuffle. consolidateFiles，该参数默认值为 false，将其设置为 true 即可开启优化机制，通常来说，如果我们使用 HashShuffleManager，那么都建议开启这个选项。

开启 consolidate 机制之后，在 shuffle write 过程中，task 就不是为下游 stage 的每个 task 创建一个磁盘文件了，此时会出现shuffleFileGroup的概念，每个 **shuffleFileGroup** 会对应一批磁盘文件，磁盘文件的数量与下游 stage 的 task 数量是相同的。一个 Executor 上有多少个 CPU core，就可以并行执行多少个 task。而第一批并行执行的每个 task 都会创建一个 shuffleFileGroup，并将数据写入对应的磁盘文件内。

当 Executor 的 CPU core 执行完一批 task，接着执行下一批 task 时，下一批 task 就会复用之前已有的 shuffleFileGroup，包括其中的磁盘文件，也就是说，此时 task 会将数据写入已有的磁盘文件中，而不会写入新的磁盘文件中。因此，consolidate 机制允许不同的 task 复用同一批磁盘文件，这样就可以有效将多个 task 的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升 shuffle write 的性能。

假设第二个 stage 有 100 个 task，第一个 stage 有 50 个 task，总共还是有 10 个 Executor（Executor CPU 个数为 1），每个 Executor 执行 5 个 task。那么原本使用未经优化的 HashShuffleManager 时，每个 Executor 会产生 500 个磁盘文件，所有 Executor 会产生 5000 个磁盘文件的。但是此时经过优化之后，每个 Executor 创建的磁盘文件的数量的计算公式为：CPU core 的数量 * 下一个 stage 的 task 数量，也就是说，每个 Executor 此时只会创建 100 个磁盘文件，所有 Executor 只会创建 1000 个磁盘文件。

优化后的 HashShuffleManager 工作原理如图
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/%E4%BC%98%E5%8C%96%E5%90%8E%E7%9A%84HashShuffleManager%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%9B%BE.jpg" width="400"></div>

## SortShuffle 解析  
SortShuffleManager 的运行机制主要分成两种，一种是普通运行机制，另一种是 bypass 运行机制。当 shuffle read task 的数量小于等于 spark.shuffle.sort. bypassMergeThreshold 参数的值时（默认为 200），就会启用 bypass 机制。
###  普通运行机制 
在该模式下，数据会先写入一个内存数据结构中，此时根据不同的 shuffle 算子，可能选用不同的数据结构。如果是 reduceByKey 这种聚合类的 shuffle 算子，那么会选用 Map 数据结构，一边通过 Map 进行聚合，一边写入内存；如果是 join 这种普通的 shuffle 算子，那么会选用 Array 数据结构，直接写入内存。接着，每写一条数据进入内存数据结构之后，就会判断一下，是否达到了某个临界阈值。如果达到临界阈值的话，那么就会尝试将内存数据结构中的数据溢写到磁盘，然后清空内存数据结构。

在溢写到磁盘文件之前，会先根据 key 对内存数据结构中已有的数据进行排序。排序过后，会分批将数据写入磁盘文件。默认的 batch 数量是 10000 条，也就是说，排序好的数据，会以每批 1 万条数据的形式分批写入磁盘文件。写入磁盘文件是通过 Java 的 BufferedOutputStream 实现的。BufferedOutputStream 是 Java 的缓冲输出流，首先会将数据缓冲在内存中，当内存缓冲满溢之后再一次写入磁盘文件中，这样可以减少磁盘 IO 次数，提升性能。

一个 task 将所有数据写入内存数据结构的过程中，会发生多次磁盘溢写操作，也就会产生多个临时文件。最后会将之前所有的临时磁盘文件都进行合并，这就是 merge 过程，此时会将之前所有临时磁盘文件中的数据读取出来，然后依次写入最终的磁盘文件之中。此外，由于一个 task 就只对应一个磁盘文件，也就意味着该 task 为下游 stage 的 task 准备的数据都在这一个文件中，因此还会单独写一份索引文件，其中标识了下游各个 task 的数据在文件中的 start offset 与 end offset。

SortShuffleManager 由于有一个磁盘文件 merge 的过程，因此大大减少了文件数量。比如第一个 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 task，而第二个 stage 有 100 个 task。由于每个 task 最终只有一个磁盘文件，因此此时每个 Executor 上只有 5 个磁盘文件，所有 Executor 只有 50 个磁盘文件。

普通运行机制的 SortShuffleManager 工作原理如图
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/%E6%99%AE%E9%80%9A%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%E7%9A%84SortShuffleManager%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.jpg" width="400"></div>

### bypass 运行机制

bypass 运行机制的触发条件如下：

- shuffle map task 数量小于 spark.shuffle.sort.bypassMergeThreshold 参数的值。
- 不是聚合类的 shuffle 算子。

此时，每个 task 会为每个下游 task 都创建一个临时磁盘文件，并将数据按 key 进行 hash 然后根据 key 的 hash 值，将 key 写入对应的磁盘文件之中。当然，写入磁盘文件时也是先写入内存缓冲，缓冲写满之后再溢写到磁盘文件的。最后，同样会将所有临时磁盘文件都合并成一个磁盘文件，并创建一个单独的索引文件。

该过程的磁盘写机制其实跟未经优化的 HashShuffleManager 是一模一样的，因为都要创建数量惊人的磁盘文件，只是在最后会做一个磁盘文件的合并而已。因此少量的最终磁盘文件，也让该机制相对未经优化的 HashShuffleManager 来说，shuffle read 的性能会更好。

而该机制与普通 SortShuffleManager 运行机制的不同在于：第一，磁盘写机制不同；第二，不会进行排序。也就是说，启用该机制的最大好处在于，shuffle write 过程中，不需要进行数据的排序操作，也就节省掉了这部分的性能开销。

bypass 运行机制的 SortShuffleManager 工作原理如图
 
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/bypass%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6%E7%9A%84SortShuffleManager%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.jpg" width="400"></div>
