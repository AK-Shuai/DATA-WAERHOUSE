## hive 架构
1. 有两种提交方式 Beeline 和 JDBC 提交任务
2. Metastore 元数据存储

## 表连接优化
1. 将大表放后头
Hive假定查询中最后的一个表是大表。它会将其它表缓存起来，然后扫描最后那个表。
因此通常需要将小表放前面，或者标记哪张表是大表：/*streamtable(table_name) */
2. 使用相同的连接键
当对3个或者更多个表进行join连接时，如果每个on子句都使用相同的连接键的话，那么只会产生一个MapReduce job。
3. 尽量尽早地过滤数据
减少每个阶段的数据量,对于分区表要加分区，同时只选择需要使用到的字段。
4. 尽量原子化操作
尽量避免一个SQL包含复杂逻辑，可以使用中间表来完成复杂的逻辑

## 用insert into替换union all
如果union all的部分个数大于2，或者每个union部分数据量大，应该拆成多个insert into 语句，实际测试过程中，执行时间能提升50%
## order by & sort by
**order by** : 对查询结果进行全局排序，消耗时间长。需要 set hive.mapred.mode=nostrict  
**sort by** : 局部排序，并非全局有序，提高效率。

## limit 语句快速出结果
一般情况下，Limit语句还是需要执行整个查询语句，然后再返回部分结果。
有一个配置属性可以开启，避免这种情况---对数据源进行抽样
hive.limit.optimize.enable=true --- 开启对数据源进行采样的功能
hive.limit.row.max.size --- 设置最小的采样容量
hive.limit.optimize.limit.file --- 设置最大的采样样本数
缺点：有可能部分数据永远不会被处理到
## 并行执行
hive会将一个查询转化为一个或多个阶段，包括：MapReduce阶段、抽样阶段、合并阶段、limit阶段等。默认情况下，一次只执行一个阶段。 不过，如果某些阶段不是互相依赖，是可以并行执行的。
set hive.exec.parallel=true,可以开启并发执行。
set hive.exec.parallel.thread.number=16; //同一个sql允许最大并行度，默认为8。

## 调整mapper和reducer的个数
1. Map阶段优化
map个数的主要的决定因素有： input的文件总个数，input的文件大小，集群设置的文件块大小（默认128M，不可自定义）
举例：
- 假设input目录下有1个文件a,大小为780M,那么hadoop会将该文件a分隔成7个块（6个128m的块和1个12m的块），从而产生7个map数
b) 假设input目录下有3个文件a,b,c,大小分别为10m，20m，130m，那么hadoop会分隔成4个块（10m,20m,128m,2m）,从而产生4个map数
即，如果文件大于块大小(128m),那么会拆分，如果小于块大小，则把该文件当成一个块。
map执行时间：map任务启动和初始化的时间+逻辑处理的时间。
- 减少map数
若有大量小文件（小于128M），会产生多个map，处理方法是：
set mapred.max.split.size=100000000; 
set mapred.min.split.size.per.node=100000000; 
set mapred.min.split.size.per.rack=100000000;
-- 前面三个参数确定合并文件块的大小，大于文件块大小128m的，按照128m来分隔，小于128m,大于100m的，按照100m来分隔，把那些小于100m的（包括小文件和分隔大文件剩下的）进行合并
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; -- 执行前进行小文件合并
- 增加map数
当input的文件都很大，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，来使得每个map处理的数据量减少，从而提高任务的执行效率。
2. Reduce阶段优化
调整方式：
参数1：hive.exec.reducers.bytes.per.reducer=1G：每个reduce任务处理的数据量
参数2：hive.exec.reducers.max=999(0.95*TaskTracker数)：每个任务最大的reduce数目
　reducer数=min(参数2,总输入数据量/参数1)
　set mapred.reduce.tasks：每个任务默认的reduce数目。典型为0.99*reduce槽数，hive将其设置为-1，自动确定reduce数目。
一般根据输入文件的总大小,用它的estimation函数来自动计算reduce的个数：reduce个数 = InputFileSize / bytes per reducer

## 数据倾斜
表现：任务进度长时间维持在99%（或100%），查看任务监控页面，发现只有少量（1个或几个）reduce子任务未完成。因为其处理的数据量和其他reduce差异过大。
单一reduce的记录数与平均记录数差异过大，通常可能达到3倍甚至更多。 最长时长远大于平均时长。
原因
- key分布不均匀
- 业务数据本身的特性
- 建表时考虑不周
- 某些SQL语句本身就有数据倾斜  


#### 关键词情形后果join其中一个表较小，但是key集中分发到某一个或几个Reduce上的数据远高于平均值join大表与大表，但是分桶的判断字段0值或空值过多这些空值都由一个reduce处理，灰常慢group bygroup by 维度过小，某值的数量过多处理某值的reduce灰常耗时count distinct某特殊值过多处理此特殊值reduce耗时
解决方案：参数调节
- hive.map.aggr=true：在map中会做部分聚集操作，效率更高但需要更多的内存。
- hive.groupby.mapaggr.checkinterval：在Map端进行聚合操作的条目数目
- hive.groupby.skewindata=true：数据倾斜时负载均衡，当选项设定为true，生成的查询计划会有两个MRJob。第一个MRJob 中，

Map的输出结果集合会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的GroupBy Key
有可能被分发到不同的Reduce中，从而达到负载均衡的目的；第二个MRJob再根据预处理的数据结果按照GroupBy Key分布到
Reduce中（这个过程可以保证相同的GroupBy Key被分布到同一个Reduce中），最后完成最终的聚合操作。

#### 合并小文件
hive.merg.mapfiles=true：合并map输出
hive.merge.mapredfiles=false：合并reduce输出
hive.merge.size.per.task=256*1000*1000：合并文件的大小
hive.mergejob.maponly=true：如果支持CombineHiveInputFormat则生成只有Map的任务执行merge
hive.merge.smallfiles.avgsize=16000000：文件的平均大小小于该值时，会启动一个MR任务执行merge。

#### 使用索引：不建议
hive.optimize.index.filter：自动使用索引
hive.optimize.index.groupby：使用聚合索引优化GROUP BY操作
- hive.exec.rowoffset：是否提供虚拟列 主要应用于字段格式长度错误导致字段插入错误，查询报错通过虚拟字段查https://blog.csdn.net/u010003835/article/details/80897518
- 推测执行：不常用实际观看时间即可
mapred.map.tasks.speculative.execution=true
mapred.reduce.tasks.speculative.execution=true
hive.mapred.reduce.tasks.speculative.execution=true;
hive.optimize.cp=true：列裁剪
hive.optimize.prunner：分区裁剪
## Map join
select /*+ mapjoin(A)*/ f.a,f.b from A t join B f  on ( f.a=t.a and f.ftime=20110802) 
## 注意事项
因为 Hive-SQL 的执行顺序为：where -> group -> having -> select -> order 所以 select 中定义的别名只能在 order by 中生效。
full join和union 区别简化达法：full join on 列合并和 union 行合并