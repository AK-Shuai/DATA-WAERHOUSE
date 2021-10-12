# 性能问题和优化方案
## 严格模式
Hive在执行SQL命令时，可以设置严格模式，防止用户执行一些对性能影响很大的查询。

    set hive.mapred.mode=strict;

## 对于紧急作业，可以提高作业优先级，以增加处理时的响应速度：
     --5个优先级可选：VERY_HIGH，HIGH，NORMAL，LOW，VERY_LOW
    set mapred.job.priority=VERY_HIGH;
## 可能造成数据倾斜的场景：
- 关联（t1 join t2 on t1.f1=t2.f1）：关联场景的关联字段 f1 的值分布不均匀（或者 NULL/0 这种值太多），会造成数据倾斜问题产生，大量数据被分发到少数几个 Reduce 处理。
- 分组（group by f1）：同样 f1 值的分布不均匀会造成数据倾斜。
- 去重（count distinct f1）：distinct 操作是全局去重，最后会在一个 reduce 中处理结果，同样会造成数据倾斜。可以替换为 select sum(tt1.num) from (select 1 as num,f1 from t1 group by f1 ) tt。

不同字段进行join一定要转换类型防止null，自动转换容易造成精度丢失

SQL的聚合运算（Group By），是比较容易产生数据倾斜的一种操作。此时，可以提前在Map端进行数据聚合，从而减少Shuffle时传输的数据量，以减少数据倾斜程度。

    set hive.map.aggr = true;  -- Map端聚合
    set hive.groupby.mapaggr.checkinterval=100000;    -- Map端聚合的条数
    set hive.map.aggr.hash.min.reduction=0.5;  --预先取100000条数据聚合,如果聚合后的条数/100000>0.5，则不再聚合

当然，对于聚合运算产生的数据倾斜现象，可以在SQL中提前对倾斜列提前进行标记，会使用两个MR任务进行处理。第一个MR任务不会理会数据的具体内容，而是将全部数据均匀分布到Reduce节点中进行运算，此时相当于Combiner部分结果聚合，不会产生数据倾斜。而第二个MR任务，才会将Key值相同的数据发送到同一个节点中进行聚合运算，得到最终结果。因为有两个MR，所以能够极大的减轻倾斜列带来的倾斜问题。

    set hive.groupby.skewindata=true; --如果是group by过程出现数据倾斜 应该设置为true
    set hive.groupby.mapaggr.checkinterval=100000;--设置group的键对应记录数超过该值会进行优化

最后对于复杂Join，因为两张表数据分布不一致，也会导致数据倾斜现象。此时，如果某一列的数据出现倾斜，而且有一张小表，可以使用MapJoin避免Shuffle情况。

    select /*+mapjoin(x)*/* from log a left outer join user b on a.user_id = b.user_id;

如果在复杂Join中，出现数据倾斜现象，而且没有小表无法使用MapJoin，则可以使用skewjoin。SkewJoin的原理是，发现列数据中明显倾斜的数据值，则不会进行Shuffle运算，而是将要Join的表对象直接加载到内存中直接进行MapJoin，而其他没有发生倾斜的数据会正常进行Shuffle运算。

    set hive.optimize.skewjoin = true; --如果是join过程出现倾斜 应该设置为true
    set hive.skewjoin.key = 100000;
## 加大并行度
通过设置参数 hive.exec.parallel 值为 true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果 job 中并行阶段增多，那么集群利用率就会增加。
### 打开任务并行执行：
    set hive.exec.parallel=true;
### 同一个 SQL 允许最大并行度，默认为 8：
    set hive.exec.parallel.thread.number=16;
如果数据倾斜严重可以考虑直接加大 Reduce 的并行度，缓解 Reduce 端的压力，但是要根据数据量来判断 reduce 的并行度，过多的启动和初始化 reduce 也会消耗时间和资源。
### 每个 Reduce 处理的数据默认是 256MB：
    SET hive.exec.reducers.bytes.per.reducer=256000000;
### 每个任务最大的 Reduce 数，默认为 1009：
    SET hive.exec.reducers.max=1009;
### Reduce 的数量默认 -1，Hive 自动确定 Reduce 数量：
    set mapreduce.job.reduces=-1;

## 预处理
    hive.groupby.skewindata=true;  
可以对倾斜的数据做个预处理，先合并一部分的数据，Hive 中有这样一个配置：  
有数据倾斜的时候进行负载均衡，当选项设定为 true，生成的查询计划会有两个 MR Job。
- 第一个 MR Job 会把 Map 端的数据随机分发到下游 Reduce 中，因为是随机，所以相同的 Key 不会完全分发到同一个 Reduce，同时也就不存在数据倾斜的问题了，这样每个 Reduce 做的只是部分聚合操作，所以我们还需要再做一次聚合。
- 第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作。

## 合并小文件

小文件会给 HDFS 带来压力，容易在文件存储端造成瓶颈，影响处理效率。Hive 中可以通过一些配置来合并 Map 和 Reduce 的结果文件，以此来消除小文件带来的影响。

如果目标表已经存在小文件问题，可以在数据输入时，设置参数，自动进行小文件的合并。这种属于已经存在问题，所进行的事后处理。

    --每个Map最大输入大小设置为2GB（单位：字节）
    set mapred.max.split.size=2048000000
    --0.5.0版本后，已经是系统默认值
    set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

为了避免小文件的生成，在数据输出时，可以进行数据合并。这种是事前处理，避免产生小文件问题。

    set hive.merge.mapfiles = true #在Map-only的任务结束时合并小文件
    set hive.merge.mapredfiles= true #在Map-Reduce的任务结束时合并小文件
    set hive.merge.size.per.task = 1024000000 #合并后文件的大小为1GB左右
    set hive.merge.smallfiles.avgsize=1024000000 #当输出文件的平均大小小于1GB时，启动一个独立的map-reduce任务进行文件merge

## 内存不足问题
在Hive计算过程中，内存不足时，会提示以下错误。

    ERROR: java.lang.OutOfMemoryError: GC overhead limit exceeded

此时可以适当调整Container内存大小，其中Xms：启动内存，Xmx：运行时内存，Xss：线程堆栈大小。

    --设置Container内存上限
    set mapreduce.map.memory.mb=-Xmx20480m
    set mapreduce.reduce.memory.mb=-Xmx20480m
    --Container中启动的Map任务，可以使用的内存上限
    set mapreduce.map.java.opts=-Xmx16384m
    --Container中启动的Reduce任务，可以使用的内存上限
    set mapreduce.reduce.java.opts=-Xmx16384m

## 虚拟内存超标
虚拟内存占用过多时，会提示以下错误。

    container_e93_1537448956025_1531565_01_000033] is running beyond physical memory limits. Current usage: 5.1 GB of 4 GB physical memory used; 18.6 GB of 8.4 GB virtual memory used.

虚拟内存大小=yarn.nodemanager.vmem-pmem-ratio（虚拟内存比例） * mapreduce.{map or reduce}.memory.mb（任务内存大小）。所以可以适当增大内存大小，或者虚拟内存的比率。

    set mapreduce.map.memory.mb=-Xmx20480m
    set mapreduce.reduce.memory.mb=-Xmx20480m
    --虚拟内存比率默认为2.3，即使用1M内存，则可以使用2.3M虚拟内存
   set yarn.nodemanager.vmem-pmem-ratio=2.3
