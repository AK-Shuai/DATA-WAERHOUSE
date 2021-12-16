# Spark 自我总结

## 自我知识链接

**Spark Yarn**：  
在启动小组长之后 启动 Driver 初始化 Spark Context 同时初始化 DAG Scheduler、 Task Scheduler、 SchedulerBackend 以及 Heartbeat Receiver。
1. 一个action启动一个job
2. 一个job分配 DAG Scheduler 做图，回溯算法，根据 shuffle 算出宽窄依赖关系
3. 把图打包 task set 给 Task Scheduler
4. Task Scheduler 把 task set 封装 task set manager 加入到调度队列
5. 采用 Fair 的话，Task Scheduler 实例化 rootpool 表示树的根节点，是pool类型
6. 各个子 pool 存储所有待分配的 task set manager 

其排序算法使用：
- Crunning Tasks 比 MinShare 小的先执行
- 权重使用率低的先执行
- 程序ID靠前的先执行

task 失败重试机制：  
失败的 Task 会记录 Host 和 executor id，下次执行不会用到这个机器下这个容器执行了。

**Spark Shuffle**：  
Hash Shuffle优化和为优化、Sort Shuffle 普通机制和bypass机制

Sort Shuffle 启动 bypass 机制
1. maptask 数量小于 bypass merge 参数值
2. 不是聚合类算子

不同于普通 Sort Shuffle 机制
1. 磁盘机制不同
2. 不排序

Sort Shuffle 普通机制每个 executor 每个 task 只产生一个文件，bypass 机制就是未优化的 Hash Shuffle 差不多，但是做了磁盘合并，相对于 Hash Shuffle 性能好一点。

Hash Shuffle 减少机制原理：  
每个 executor 为每个 task 创建下一个总 task 文件，优化后为做分组每个 task 共用下一个总 task 文件，缩减的死每个 executor 执行 task 的数量。

废弃 Hash Shuffle 主要原因是产生文件数不可控，在内存不足使用磁盘时有可能导致巨多文件，寻址时间过长，下面会写内存不够情况。

**堆内内存和堆外内存占用**：  
在内存动态占用中计算内存被存储内存占用只能等待释放，而存储内存被占用是可以落盘的。



