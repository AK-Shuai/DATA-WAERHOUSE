# Spark RDD、DF、DS的区别

Spark 有三套 API RDD（2011）、DataFrame（2013）、DataSet（2015）

- [RDD](#RDD)
    - [RDD的概念](#RDD的概念)
    - [RDD的优点](#RDD的优点)
    - [RDD的短板](#RDD的短板)
- [DataFrame](#DataFrame)
    - [DataFrame概念](#DataFrame概念)
    - [DataFrame的优点](#DataFrame的优点)
    - [DataFrane的短板](#DataFrane的短板)
- [DataSet](#DataSet)
    - [DataSet概念](#DataSet概念)
    - [DataSet的优点](#DataSet的优点)
- [问题](#问题)
- [参考](#参考)

## RDD
### RDD的概念
- RDD是Spark 1.1版本开始引入的。
- RDD是Spark的基本数据结构。
- RDD是Spark的弹性分布式数据集，它是不可变的（Immutable）。
- RDD所描述的数据分布在集群的各个节点中，基于RDD提供了很多的转换的并行处理操作。
- RDD具备容错性，在任何节点上出现了故障，RDD是能够进行容错恢复的。
- RDD专注的是How！就是如何处理数据，都由我们自己来去各种算子来实现。

### RDD的优点
- 编译时类型安全，编译时就能检查出类型错误。
- 面向对象的编程风格，直接通过对象调用方法的形式来操作数据。

### RDD的短板
- 集群间通信都需要将JVM中的对象进行序列化和反序列化，RDD开销较大
- 频繁创建和销毁对象会增加GC，GC的性能开销较大


>Spark 2.0开始，RDD不再是一等公民
>从Apache Spark 2.0开始，RDD已经被降级为二等公民，RDD已经被弃用了。而且，我们一会就会发现，DataFrame/DataSet是可以和RDD相互转换的，DataFrame和DataSet也是建立在RDD上。

## DataFrame

### DataFrame概念
- DataFrame是从Spark 1.3版本开始引入的。
- 通过DataFrame可以简化Spark程序的开发，让Spark处理结构化数据变得更简单。DataFrame可以使用SQL的方式来处理数据。例如：业务分析人员可以基于编写Spark SQL来进行数据开发，而不仅仅是Spark开发人员。
- DataFrame和RDD有一些共同点，也是不可变的分布式数据集。但与RDD不一样的是，DataFrame是有schema的，有点类似于关系型数据库中的表，每一行的数据都是一样的，因为。有了schema，这也表明了DataFrame是比RDD提供更高层次的抽象。
- DataFrame支持各种数据格式的读取和写入，例如：CSV、JSON、AVRO、HDFS、Hive表。
- DataFrame使用Catalyst进行优化。
- DataFrame专注的是What！，而不是How！

### DataFrame的优点
- 因为DataFrame是有统一的schema的，所以序列化和反序列无需存储schema。这样节省了一定的空间。
- DataFrame存储在off-heap（堆外内存）中，由操作系统直接管理（RDD是JVM管理），可以将数据直接序列化为二进制存入off-heap中。操作数据也是直接操作off-heap。

### DataFrane的短板
- DataFrame不是类型安全的
- API也不是面向对象的

>针对Python或者R，不提供类型安全的DataSet，只能基于DataFrame API开发。

## DataSet

### DataSet概念
- DataSet是从Spark 1.6版本开始引入的。
- DataSet具有RDD和DataFrame的优点，既提供了更有效率的处理、以及类型安全的API。
- DataSet API都是基于Lambda函数、以及JVM对象来进行开发，所以在编译期间就可以快速检测到错误，节省开发时间和成本。
- DataSet使用起来很像，但它的执行效率、空间资源效率都要比RDD高很多。可以很方便地使用DataSet处理结构化、和非结构数据。

### DataSet的优点
- DataSet结合了RDD和DataFrame的优点。
- 当序列化数据时，Encoder生成的字节码可以直接与堆交互，实现对数据按需访问，而无需反序列化整个对象。

## 问题
1. DS 和 DF 区别和优势？
    - DS 的解析错误可以在编译时就知道
    - DS 支持泛型
    - DS 类型是安全的
    - DF 聚合最快因为数据类型确定

2. DF 或 DS 优势
    - DataFrame和DataSet API是基于Spark SQL引擎之上构建的，会使用Catalyst生成优化后的逻辑和物理执行计划。尤其是无类型的DataSet[Row]（DataFrame），它的速度更快，很适合交互式查询。

    - 由于Spark能够理解DataSet中的JVM对象类型，所以Spark会将将JVM对象映射为Tungsten的内部内存方式存储。而Tungsten编码器可以让JVM对象更有效地进行序列化和反序列化，生成更紧凑、更有效率的字节码。

> DataSet的空间存储效率是RDD的4倍

3. 对比RDD和DataSet的API

    - RDD的操作都是最底层的，Spark不会做任何的优化。是low level的API，无法执行schema的高阶声明式操作
    - DataSet支持很多类似于RDD的功能函数，而且支持DataFrame的所有操作。其实我们前面看到了DataFrame就是一种特殊的、能力稍微弱一点的DataSet。DataSet是一种High Level的API，在效率上比RDD有很大的提升。

4. 为什么使用 DataFrame
    - 因为我们使用 python 不知道类型安全，所以基于 DataFrame API 开发的。

5. DataFrame转成RDD会不会有性能的消耗
    - 由df转成rdd，df.rdd的过程会有一个反序列化的过程，会造成一定的内存消耗。

## 参考
1. <a href="https://www.cnblogs.com/mr-bigdata/p/14426049.html#rdddataframedataset%E4%BB%8B%E7%BB%8D" target="_blank">Spark RDD、DF、DS的区别</a>