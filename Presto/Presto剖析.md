# Presto 剖析

## 概况

即席查询

1. 充分利用内存（Presto）
2. 预查询思想（kylin）

如果想要计算的数据分散在Hdfs、Hive、ES、Hbase、MySql、Kafka中，应该怎么做？

Facebook科学家们发现目前并没有一款合适的计算引擎，最终决定开发一款MPP交互式计算引擎

2012年秋天进行研发，2013年开源出来并成功用其对300PB的数据进行运算，奠定了Presto的地位

master/slave架构

- coordinators 解析SQL语句，生成执行计划，分发执行任务给Worker节点执行
- **workers**负责执行实际查询任务，访问底层存储系统。
- Worker节点启动后向Discovery Server服务注册，Coordinator从Discovery Server获得可以正常工作的Worker节点

## 架构图

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/prosto%E6%9E%B6%E6%9E%84%E5%9B%BE.png" width="400"></div>

优点：

1. 基于内存运算，减少了硬盘IO，计算更快
2. 能够连接多个数据源，跨数据源连表查，比如从Hive查询大量网站访问记录，然后从Mysql中匹配出设备信息。

缺点：

1. Presto能处理PB级别的海量数据分析，但Presto并不是把PB即数据都放在内存中计算。而是根据场景，如Count，AVG等聚合运算，是边堵数据边计算，再清理内存，再读数据再计算，这种内存耗的并不高。但是连表查，就可能产生大量的临时数据，因此速度会变慢。

## 数据模型

PipeLine模式：

Worker节点将小任务封装成一个PrioritizedSplitRunner对象，放入pending split优先级队列中

Worker节点启动一定数目的线程进行计算，线程数task.shard.max-threads=availableProcessors() * 4

每个空闲线程从队列中取出PrioritizedSplitRunner对象执行，每隔1秒，判断任务是否执行完成，如果完成，从allSplits队列中删除，如果没有，则放回pendingSplits队列

每个任务的执行流程如下图右侧，依次遍历所有Operator，尝试从上一个Operator取一个Page对象，如果取得的Page不为空，交给下一个Operator执行

节点间流水线计算

ExchangeOperator的执行流程图，ExchangeOperator为每一个Split启动一个HttpPageBufferClient对象，主动向上一个Stage的Worker节点拉数据，拉取的最小单位也是一个Page对象，取到数据后放入Pages队列

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/prosto%E6%95%B0%E6%8D%AE%E6%A8%A1%E5%9E%8B.png" width="400"></div>

**presto采取三层表结构**：

1. catalog 对应某一类数据源，例如hive的数据，或mysql的数据
2. schema 对应mysql中的数据库
3. table 对应mysql中的表

**presto的存储单元包括**：

1. Block是一张表的一个字段对应的队列
2. Page由Block构成，一个Page包含多个Block，多个Block横切为一行。

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/presto%E5%82%A8%E5%AD%98.png" width="400"></div>


### 列式存储
- 顺序写文件。
- 一行数据多个属性，有多少个属性就有多少个文件，把一行数据放到多个文件中，对于复杂数据，肯定不行。一个ppt，每页分成一个文件。
- 内存中缓存数据，缓存到一定量后，将各个列的数据放在一起打包，各个包再按一定顺序写入文件，精8髓：按列缓存打包
- 每一列也会分成一个个数据包，这就是Page，（1000行打包一个page或8Kb打包一个page）
- 每列数据类型一致，高效压缩。数据既是索引，只访问查询涉及的列。
- ORC文件格式 ORC文件格式是一种Hadoop生态圈中的列式存储格式
- 文件footer里存储了每个条带(script)的最大值最小值，可以用来过滤
- snappy压缩
- parquet （Impala支持最好）
- orc （Presto支持最好）
- 列式存储 行式存储各有各自的优缺点，在OLAP场景下，我们需要的就是列式存储。

## 执行计划分析
1. 一个查询是由Stage、Task、Driver、Split、Operator和DataSource组成

2. 每个查询分解为多个stage

3. 每个 stage拆分多个task（每个PlanFragment都是一个Stage）

4. 每个task处理一个或多个split(Driver)

5. 每个Driver处理一个split，每个Driver都是作用于一个split的一系列Operator集合

6. 每个split对应一个或多个page对象 ：一个大的数据集中的一个小的子集

7. 每个page包含多个block对象，每个block对象是个字节数据 （presto中最小处理单位）

8. Operator：一个operator代表对一个split的一种操作（如过滤、转化等）

9. operator每次只会读取一个page对象 并且每次也只会产生一个page对象

10. Cli —> parser—>analysis —> 优化 —> 拆分为子plan —> 调度

### 一般分为四种Stage:
- Source Stage: 从数据源读取数据
- Fixed Stage: 进行中间的运算
- Single Stage: 也称为Root Stage，这个是必不可少的，用于最后把数据汇总给Coordinator
- Coordinator_Only：用于执行DDL语句等不需要进行计算的语句

一个简单查询，最终构造为一个QueryPlan。对于较复杂的查询，是多个QueryPlan的组合。

### Task
Task是Stage的具体分布式查询计划，由Coordinator进行划分
Task需要通过Http接口发送到具体的Node上去，然后生成对应的本地执行计划
一个Stage可以分解为多个同构的Task

### Driver
Task是发送到本地的执行计划
Task被分为多种Driver去并发的执行

### Operator
一个Driver内包含多个Operator
真正操作Page数据的是Operator
一个Operator代表一种操作，比如Scan->读取数据, Filter -> 过滤数据

**优化过程**:
谓词下推：基本思想即：将过滤表达式尽可能移动至靠近数据源的位置，以使真正执行时能直接跳过无关的数据。（将过滤表达式下推到存储层直接过滤数据，减少传输到计算层的数据量）

1. 客户端调用CN提供的查询rest接口，CN接收到请求后创建一个对应的Query对象放在队列里并转化为Statement树，同时返回一个QueryId给客户端用于后续获取状态以及获取结果
QueuedStatementResource对象
2. 在客户端初次通过QueryId进行查询时，CN会启动一个线程对这个Query对象进行调度 SqlQueryExecution对象
3. 首先将Query对象对应的Statement转化为AST对象；此阶段做语法校验、语句重写以及权限校验Analysis对象
4. 执行计划编译器将语法树层层展开，把树转成树状的执行结构，每个节点都是一个操作；此时会优化器会发挥作用，生成效率更高的执行计划树。Plan 对象
5. 对执行计划树进行分布式处理，使其可以并行的运算处理。SubPlan 对象 1~5 是发生在Coordinator进程中的预查询阶段
6. 遍历逻辑执行计划树获取需要执行的Stage集合SqlQueryScheduler#createStages方法：创建Stages集合列表 StageExecutionPlan 对象
7. 生成逻辑执行计划并开始进行任务调度，此时调度的最小单位是Task；Task的作用是在Worker拉起计算任务同时周期查询此Task的状态 SqlQueryScheduler 对象，6~7 是发生在Coordinator进程中的调度阶段
8. Worker提供接口给Coordinator进行Task的状态更新 TaskResource对象
9. Worker有缓存TaskId和SqlTask的映射，在接收到请求时会根据TaskId获取对应的SqlTask来进行请求处理,每个Task任务都有唯一对应的SqlTask对象，每个SqlTask对象都有唯一对应的TaskHolder对象,TaskHolder对象存储Task的状态以及Task处理逻辑 SqlTaskExecution
10. 先构造TaskContext 对象，再根据此对象来创建本地执行计划，最终根据本地执行计划构造Task执行器SqlTaskExecution,LocalExecutionPlanner 对象
11. 在构造SqlTaskExecution对象时，其会以依赖注入方式构造对应的TaskExecutor 对象
TaskExecutor对象通过构造函数并以线程池方式添加计算任务，由TaskRunner对象封装计算逻辑,每个TaskRunner对象从待处理split队列头获取一个split进行处理；加锁根据DriverContext和算子集合来构造并执行Driver
12. TaskExecutor对象通过DriverSplitRunner对Driver进行管理，此时执行Driver的逻辑
释放算子Operator不用的内存资源，同时根据条件进行磁盘溢写操作；更新Driver的数据源split列表信息,TaskRunner线程对象会依次处理Split/Driver队列中的任务直到队列为空，此时SqlTask对象的状态就会变为已完成状态,Driver 对象
13. 将该Task对应的TaskSource列表存到SqlTaskExecution中同时更新此Task对应的Driver的数据源。同时将当前Task状态信息返回给Coordinator
14. 在更新Task对应的数据源时，开始进行任务调度,SqlTaskExecution#schedulePartitionedSource,8~14发生在Worker节点
