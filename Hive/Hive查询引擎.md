- [Hive MR内容](#Hive-MR)
- [Hive On Tez](#Hive-On-Tez)
- [Hive On Spark](#Hive-On-Spark)
- [参考](#参考)

# Hive 引擎介绍
hadoop 的基本架构中的计算框架是map-reduce，简称MR，而Hive是基于hadoop的，所以Hive的默认引擎也是MR。 MR的速度确实非常感人的慢，随着计算引擎的更新，目前主要是Tez和Spark。那么这3个计算引擎的区别是什么

因为都运行在yarn 之上，所以这3个引擎可以随意切换。
## Hive MR

MAP:
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/HiveMRMap.jpeg" width="400"></div>

Reduce:
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/HiveMRReduce.jpeg" width="400"></div>

MR将一个算法抽象成Map和Reduce两个阶段进行处理，如果一个HQL经过转化可能有多个job，那么在这中间文件就有多次落盘，速度较慢。
## Hive On Tez
该引擎核心思想是将Map和Reduce两个操作进一步拆分，即Map被拆分成Input、Processor、Sort、Merge和Output，Reduce被拆分成Input、Shuffle、Sort、Merge、Processor和Output等，这样，这些分解后的元操作可以任意灵活组合，产生新的操作，这些操作经过一些控制程序组装后，可形成一个大的DAG作业。

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/HiveOnTez.jpeg" width="400"></div>

Tez可以将多个有依赖的作业转换为一个作业（这样只需写一次HDFS，且中间节点较少），从而大大提升DAG作业的性能

## Hive On Spark

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/HiveOnSpark.jpeg" width="400"></div>

1. Spark是一个分布式的内存计算框架，其特点是能处理大规模数据，计算速度快。
2. Spark的计算过程保持在内存中，减少了硬盘读写，能够将多个操作进行合并后计算，因此提升了计算速度。同时Spark也提供了更丰富的计算API，例如filter，flatMap，count，distinct等。
3. 过程间耦合度低，单个过程的失败后可以重新计算，而不会导致整体失败；

## 参考
1. <a href="https://zhuanlan.zhihu.com/p/252288440" target="_blank">Hive三大执行引擎</a>
