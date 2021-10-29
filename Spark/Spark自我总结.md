
## cache和persist的区别？
cache只有一个默认的缓存级别MEMORY_ONLY ，而persist可以根据情况设置其它的缓存级别。
查看 StorageLevel 类的源码：
```
object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(false, false, true, false)
  ......
}
```

可以看到这里列出了12种缓存级别，但这些有什么区别呢？可以看到每个缓存级别后面都跟了一个StorageLevel的构造函数，里面包含了4个或5个参数，如下
```
val MEMORY_ONLY = new StorageLevel(false, true, false, true)
```

```
class StorageLevel private(
    private var _useDisk: Boolean,
    private var _useMemory: Boolean,
    private var _useOffHeap: Boolean,
    private var _deserialized: Boolean,
    private var _replication: Int = 1)
  extends Externalizable {
  ......
  def useDisk: Boolean = _useDisk
  def useMemory: Boolean = _useMemory
  def useOffHeap: Boolean = _useOffHeap
  def deserialized: Boolean = _deserialized
  def replication: Int = _replication
  ......
}
```

可以看到StorageLevel类的主构造器包含了5个参数：
- useDisk：使用硬盘（外存）
- useMemory：使用内存
- useOffHeap：使用堆外内存，这是Java虚拟机里面的概念，堆外内存意味着把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机）。这样做的结果就是能保持一个较小的堆，以减少垃圾收集对应用的影响。
- deserialized：反序列化，其逆过程序列化（Serialization）是java提供的一种机制，将对象表示成一连串的字节；而反序列化就表示将字节恢复为对象的过程。序列化是对象永久化的一种机制，可以将对象及其属性保存起来，并能在反序列化后直接恢复这个对象
- replication：备份数（在多个节点上备份）

##  cache和persist是transformaiton算子还是action算子？ 
transformaiton 转换算子：懒加载执行
action 行动算子：触发执行
cache,persist,checkpoint 控制算子：懒执行
### cache和persist的注意事项：
1. cache和persist都是懒执行，必须有一个action类算子触发执行。
2. cache和persist算子的返回值可以赋值给一个变量，在其他job中直接使用这个变量就是使用持久化的数据了。持久化的单位是partition。
3. cache和persist算子后不能立即紧跟action算子。
4. cache和persist算子持久化的数据当applilcation执行完成之后会被清除。
错误：rdd.cache().count() 返回的不是持久化的RDD，而是一个数值了。

### checkpoint：
checkpoint将RDD持久化到磁盘，还可以切断RDD之间的依赖关系。checkpoint目录数据当application执行完之后不会被清除。

### checkpoint 的执行原理：

当RDD的job执行完毕后，会从finalRDD从后往前回溯。
当回溯到某一个RDD调用了checkpoint方法，会对当前的RDD做一个标记。
Spark框架会自动启动一个新的job，重新计算这个RDD的数据，将数据持久化到HDFS上。
优化：对RDD执行checkpoint之前，最好对这个RDD先执行cache，这样新启动的job只需要将内存中的数据拷贝到HDFS上就可以，省去了重新计算这一步。


## spark的RDD是什么东西？有什么特点？弹性体现在什么地方？ 
RDD(Resilient Distributed Datasets) ,弹性分布式数据集， 是分布式内存的一个抽象概念，指的是一个只读的，可分区的分布式数据集，这个数据集的全部或部分可以缓存在内存中，在多次计算间重用。

RDD提供了一种高度受限的共享内存模型，即RDD是只读的记录分区的集合，只能通过在其他RDD执行确定的转换操作（如map、join和group by）而创建，然而这些限制使得实现容错的开销很低。对开发者而言，RDD可以看作是Spark的一个对象，它本身运行于内存中，如读文件是一个RDD，对文件计算是一个RDD，结果集也是一个RDD ，不同的分片、 数据之间的依赖 、key-value类型的map数据都可以看做RDD。

### RDD的五大特性？
1. RDD是由一系列的partition组成的
2. RDD之间具有依赖关系
3. RDD作用在partition是上
4. partition作用在具有（k,v）格式的数据集
5. partition对外提供最佳计算位置，利于数据本地化的处理
### RDD的弹性体现在哪里？
1. 自动进行内存和磁盘切换
2. 基于lineage的高效容错
3. task如果失败会特定次数的重试
4. stage如果失败会自动进行特定次数的重试，而且只会只计算失败的分片
5. checkpoint【每次对RDD操作都会产生新的RDD，如果链条比较长，计算比较笨重，就把数据放在硬盘中】和persist 【内存或磁盘中对数据进行复用】(检查点、持久化)
6. 数据调度弹性：DAG TASK 和资源管理无关
7. 数据分片的高度弹性repartion

### RDD为何高效？
通过记录更新的方式为何很高效
1. RDD是不可变的+lazy。转化操作，行为操作。
2. RDD是粗粒度。[每次操作 都作用于所以集合] 对于RDD的写是粗粒度的 RDD的读 操作 可以是粗粒度的也可以是细粒度的： 可以读其中的一条记录。

### 常规容错机制
数据检查点[checkpoint]每次都有一个拷贝 IO开销大 网络带宽是分布式的瓶颈
记录数据的更新 每次数据变化都记录一下