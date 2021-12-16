
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

## 参考
<a href="http://www.voycn.com/article/spark-dagdeshengchenghehuafenstage" target="_blank">DAG宽窄依赖讲解</a>
