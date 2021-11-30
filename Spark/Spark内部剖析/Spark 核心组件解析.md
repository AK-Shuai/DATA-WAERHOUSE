# Spark 核心组件解析

## BlockManager 数据存储与管理机制

BlockManager 是整个 Spark 底层负责数据存储与管理的一个组件，Driver 和 Executor 的所有数据都由对应的 BlockManager 进行管理。

Driver 上有 BlockManagerMaster，负责对各个节点上的 BlockManager 内部管理的数据的元数据进行维护，比如 block 的增删改等操作，都会在这里维护好元数据的变更。

每个节点都有一个 BlockManager，每个 BlockManager 创建之后，第一件事即使去向 BlockManagerMaster 进行注册，此时 BlockManagerMaster 会为其长难句对应的 BlockManagerInfo。

BlockManager 运行原理如下图所示：
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/BlockManager%E8%BF%90%E8%A1%8C%E5%8E%9F%E7%90%86%E5%A6%82%E4%B8%8B%E5%9B%BE%E6%89%80%E7%A4%BA.jpg" width="400"></div>

BlockManagerMaster 与 BlockManager 的关系非常像 NameNode 与 DataNode 的关系，BlockManagerMaster 中保存中 BlockManager 内部管理数据的元数据，进行维护，当 BlockManager 进行 Block 增删改等操作时，都会在 BlockManagerMaster 中进行元数据的变更，这与 NameNode 维护 DataNode 的元数据信息，DataNode 中数据发生变化时 NameNode 中的元数据信息也会相应变化是一致的。

每个节点上都有一个 BlockManager，BlockManager 中有 3 个非常重要的组件：

- DiskStore：负责对磁盘数据进行读写；
- MemoryStore：负责对内存数据进行读写；
- BlockTransferService：负责建立 BlockManager 到远程其他节点的 BlockManager 的连接，负责对远程其他节点的 BlockManager 的数据进行读写；

每个 BlockManager 创建之后，做的第一件事就是想 BlockManagerMaster 进行注册，此时 BlockManagerMaster 会为其创建对应的 BlockManagerInfo。

使用 BlockManager 进行写操作时，比如说，RDD 运行过程中的一些中间数据，或者我们手动指定了 persist()，会优先将数据写入内存中，如果内存大小不够，会使用自己的算法，将内存中的部分数据写入磁盘；此外，如果 persist()指定了要 replica，那么会使用 BlockTransferService 将数据 replicate 一份到其他节点的 BlockManager 上去。

使用 BlockManager 进行读操作时，比如说，shuffleRead 操作，如果能从本地读取，就利用 DiskStore 或者 MemoryStore 从本地读取数据，但是本地没有数据的话，那么会用 BlockTransferService 与有数据的 BlockManager 建立连接，然后用 BlockTransferService 从远程 BlockManager 读取数据；例如，shuffle Read 操作中，很有可能要拉取的数据在本地没有，那么此时就会到远程有数据的节点上，找那个节点的 BlockManager 来拉取需要的数据。

只要使用 BlockManager 执行了数据增删改的操作，那么必须将 Block 的 BlockStatus 上报到 BlockManagerMaster，在 BlockManagerMaster 上会对指定 BlockManager 的 BlockManagerInfo 内部的 BlockStatus 进行增删改操作，从而达到元数据的维护功能。

## Spark 共享变量底层实现

Spark 一个非常重要的特性就是共享变量。

默认情况下，如果在一个算子的函数中使用到了某个外部的变量，那么这个变量的值会被拷贝到每个 task 中，此时每个 task 只能操作自己的那份变量副本。如果多个 task 想要共享某个变量，那么这种方式是做不到的。

Spark 为此提供了两种共享变量，一种是 Broadcast Variable（广播变量），另一种是 Accumulator（累加变量）。Broadcast Variable 会将用到的变量，仅仅为每个节点拷贝一份，即每个 Executor 拷贝一份，更大的用途是优化性能，减少网络传输以及内存损耗。Accumulator 则可以让多个 task 共同操作一份变量，主要可以进行累加操作。Broadcast Variable 是共享读变量，task 不能去修改它，而 Accumulator 可以让多个 task 操作一个变量。

### 广播变量

广播变量允许编程者在每个 Executor 上保留外部数据的只读变量，而不是给每个任务发送一个副本。

每个 task 都会保存一份它所使用的外部变量的副本，当一个 Executor 上的多个 task 都使用一个大型外部变量时，对于 Executor 内存的消耗是非常大的，因此，我们可以将大型外部变量封装为广播变量，此时一个 Executor 保存一个变量副本，此 Executor 上的所有 task 共用此变量，不再是一个 task 单独保存一个副本，这在一定程度上降低了 Spark 任务的内存占用。
task 使用外部变量：
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/task%E4%BD%BF%E7%94%A8%E5%A4%96%E9%83%A8%E5%8F%98%E9%87%8F.jpg" width="400"></div>
使用广播变量：
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/task%E4%BD%BF%E7%94%A8%E5%B9%BF%E6%92%AD%E5%8F%98%E9%87%8F.jpg" width="400"></div>

Spark 还尝试使用高效的广播算法分发广播变量，以降低通信成本。

Spark 提供的 Broadcast Variable 是只读的，并且在每个 Executor 上只会有一个副本，而不会为每个 task 都拷贝一份副本，因此，它的最大作用，就是减少变量到各个节点的网络传输消耗，以及在各个节点上的内存消耗。此外，Spark 内部也使用了高效的广播算法来减少网络消耗。

可以通过调用 SparkContext 的 broadcast()方法来针对每个变量创建广播变量。然后在算子的函数内，使用到广播变量时，每个 Executor 只会拷贝一份副本了，每个 task 可以使用广播变量的 value()方法获取值。

在任务运行时，Executor 并不获取广播变量，当 task 执行到 使用广播变量的代码时，会向 Executor 的内存中请求广播变量，如下图所示：
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Executor%E7%9A%84%E5%86%85%E5%AD%98%E4%B8%AD%E8%AF%B7%E6%B1%82%E5%B9%BF%E6%92%AD%E5%8F%98%E9%87%8F.jpg" width="400"></div>
之后 Executor 会通过 BlockManager 向 Driver 拉取广播变量，然后提供给 task 进行使用，如下图所示：

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Executor%E5%90%91Driver%E6%8B%89%E5%8F%96%E5%B9%BF%E6%92%AD%E5%8F%98%E9%87%8F.jpg" width="400"></div>

广播大变量是 Spark 中常用的基础优化方法，通过减少内存占用实现任务执行性能的提升。

### 累加器

累加器（accumulator）：Accumulator 是仅仅被相关操作累加的变量，因此可以在并行中被有效地支持。它们可用于实现计数器（如 MapReduce）或总和计数。

Accumulator 是存在于 Driver 端的，集群上运行的 task 进行 Accumulator 的累加，随后把值发到 Driver 端，在 Driver 端汇总（Spark UI 在 SparkContext 创建时被创建，即在 Driver 端被创建，因此它可以读取 Accumulator 的数值），由于 Accumulator 存在于 Driver 端，从节点读取不到 Accumulator 的数值。

Spark 提供的 Accumulator 主要用于多个节点对一个变量进行共享性的操作。Accumulator 只提供了累加的功能，但是却给我们提供了多个 task 对于同一个变量并行操作的功能，但是 task 只能对 Accumulator 进行累加操作，不能读取它的值，只有 Driver 程序可以读取 Accumulator 的值。

Accumulator 的底层原理如下图所示：
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Accumulator%E7%9A%84%E5%BA%95%E5%B1%82%E5%8E%9F%E7%90%86.jpg" width="400"></div>
