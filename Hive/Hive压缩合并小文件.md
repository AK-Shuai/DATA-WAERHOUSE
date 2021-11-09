# Hive 压缩合并小文件

HDFS非常容易存储大数据文件，如果Hive中存在过多的小文件会给namecode带来巨大的性能压力。同时小文件过多会影响JOB的执行，hadoop会将一个job转换成多个task，即使对于每个小文件也需要一个task去单独处理，task作为一个独立的jvm实例，其开启和停止的开销可能会大大超过实际的任务处理时间。

同时我们知道hive输出最终是mr的输出，即reducer（或mapper）的输出，有多少个reducer（mapper）输出就会生成多少个输出文件，根据shuffle/sort的原理，每个文件按照某个值进行shuffle后的结果。

为了防止生成过多小文件，hive可以通过配置参数在mr过程中合并小文件。而且在执行sql之前将小文件都进行Merge，也会提高程序的性能。我们可以从两个方面进行优化，其一是map执行之前将小文件进行合并会提高性能，其二是输出的时候进行合并压缩，减少IO压力。

## 小文件带来的问题
HDFS的文件元信息，包括位置、大小、分块信息等，都是保存在NameNode的内存中的。每个对象大约占用150个字节，因此一千万个文件及分块就会占用约3G的内存空间，一旦接近这个量级，NameNode的性能就会开始下降了。此外，HDFS读写小文件时也会更加耗时，因为每次都需要从NameNode获取元信息，并与对应的DataNode建立连接。对于MapReduce程序来说，小文件还会增加Mapper的个数，每个脚本只处理很少的数据，浪费了大量的调度时间。当然，这个问题可以通过使用CombinedInputFile和JVM重用来解决。

1. 元数据信息增加给namenode节点造成压力
2. 读写小文件时，获取元数据信息更加耗时
3. mapper个数增加，浪费调度时间

## Hive小文件产生的原因
汇总后的数据量通常比源数据要少得多。而为了提升运算速度，我们会增加Reducer的数量，Hive本身也会做类似优化——Reducer数量等于源数据的量除以hive.exec.reducers.bytes.per.reducer所配置的量（默认1G）。Reducer数量的增加也即意味着结果文件的增加，从而产生小文件的问题。

1. 动态分区插入数据，产生大量的小文件，从而导致map数量剧增。
2. reduce数量越多，小文件也越多(reduce的个数和输出文件是对应的)。
3. 数据源本身就包含大量的小文件。

## 合并小文件
1.使用hadoop archive命令把小文件进行归档。
2.重建表，建表时减少reduce数量。
3.通过参数进行调节，设置map/reduce端的相关参数。

输入合并。即在Map前合并小文件
输出合并。即在输出结果的时候合并小文件

输入合并：
```
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;  
//执行Map前进行小文件合并
set mapred.max.split.size=256000000;  
//每个Map最大输入大小
set mapred.min.split.size.per.node=100000000; 
//一个节点上split的至少的大小 
set mapred.min.split.size.per.rack=100000000; 
//一个交换机下split的至少的大小
```

输出合并：
```
set hive.merge.mapfiles = true 
//在Map-only的任务结束时合并小文件
set hive.merge.tezfiles=true;
set hive.merge.mapredfiles = true 
//在Map-Reduce的任务结束时合并小文件
set hive.merge.size.per.task = 256*1000*1000 
//合并文件的大小
set hive.merge.smallfiles.avgsize=16000000 
//当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge
```

压缩文件：
```
sethive.exec.compress.output=true;
#默认false，是否对输出结果压缩

setmapred.output.compression.codec=org.apache.hadoop.io.compress.GzipCodec;
#压缩格式设置

setmapred.output.compression.type=BLOCK;
#一共三种压缩方式（NONE, RECORD,BLOCK），BLOCK压缩率最高，一般用BLOCK。
```