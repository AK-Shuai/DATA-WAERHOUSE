# Hive 的架构

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Hive%E6%9E%B6%E6%9E%84%E5%9B%BE.png" width="400"></div> 

## 客户端任务提交的方式
### Hive 客户端任务提交的方式主要有两种：
- Beeline 和 JDBC 提交任务的方式是类似的，会向一个 Thrift 服务（HiverServer2）来提交 SQL 代码，然后在 HiverServer2 端进行 SQL 的解析和优化，这个过程会用到 MetaStore 中的 Hive 元数据。这个步骤是通过 Driver 驱动来做的，最后 Driver 生成的是可执行的 MapReduce 程序。如果是 DDL 语句，Driver 只会与 MetaStore 交互，操作 Hive 的相关元数据，而不会生成 MapReduce 任务。
- Hive CLI 的任务提交方式社区已经不推荐使用，这种方式 Driver 是在本地的，也就意味着编译和解析出 MapReduce 的过程是在 CLI 端完成，不需要经过 HiverServer2，可以有效减轻 HiveServer2 的压力，但是在权限控制上会有不足，这种方式 Driver 端仍然需要与 MetaStore 交互获取元数据。最终生成的 MapReduce 程序在 Yarn 上调度执行。
### Metastore 元数据存储
Metastore 是 Hive 存储元数据并提供服务的进程，元数据一般存储在 MySQL 中（测试场景可以用 Derby 存储），其中包含表名、表所属的数据库（默认是 default）、表的拥有者、列/分区字段、表的类型（是否是外部表）、表的数据所在目录等。
### Driver
Driver 是 Hive 的核心，它负责 HiveSQL 的解析，整个过程和前面提到 SparkSQL 的解析类似，分为以下几个部分。
- 解析器：使用 antlr 将 SQL 字符串转换成抽象语法树 AST；对 AST 进行语法分析，比如表是否存在、字段是否存在、SQL 语义是否有误。
- 编译器：将抽象语法树 AST 编译生成逻辑执行计划。
- 优化器：对逻辑执行计划进行优化。
- 执行器：把逻辑执行计划转换成可以运行的物理计划。对于 Hive 来说，就是 MapReduce。
### 与 Hadoop 的关系
Driver 最终会生成 MapReduce 可执行任务，MapReduce 一般会通过 Yarn 来调度资源执行，Hive 表的实际数据存储是由 HDFS 来完成，Hive 只存储元数据，不存储用户的数据。



## Hive 表的属性
### 什么是分区？
- Hive 表可以在建表时指定一或多个分区字段，每个分区以文件夹的形式单独存在表文件夹的目录下。一般分区字段为时间，即可以选择把一天内的数据放在一个分区，即把一天的数据放在一个文件夹下面。
- 分区是以字段的形式在表结构中存在，查询时，分区字段会出现在查询结果中，但是该字段实际是不存放数据内容的，即 HDFS 对应的数据文件内部是没有 dt 这个字段的数据的，它是一个虚拟的列。
- 分区的目的是提高读性能，在创建了分区表后读数据最好要带上分区字段，可以避免全表扫描，减少读的数据量，提升效率。另外，写数据也要指定写入哪个分区。

### 什么是分桶？
- 桶是更为细粒度的数据范围划分，对于每一个表或分区，Hive 还可以划分桶，桶对应的是 HDFS 分区目录下的各个文件。
- 分桶的具体流程是针对 Hive 某一列进行哈希取值，然后将哈希值除以桶的个数求余，这个余数就决定了该条数据存放在哪个桶当中。
- 分桶的目的当然也是为了提高读性能，当两个在关联列上做了分桶的表 A、B 进行 JOIN 时，表 A 的每个桶会和表 B 对应的桶直接 JOIN，而不用全表 JOIN，较少 JOIN 的数据量，明显提高查询效率。

## 什么是外部表？

外部表是将 Hive 表结构和外部的数据做一个映射，数据实际上不存储在 Hive 所属的 HDFS 目录结构下，所以当删除表的时候，只删除了表的元数据信息，表实际的数据是不会删除的。而内部表会把数据存储在 HDFS 特定的目录下，删除表会把所有数据都删除，包括元数据和实际数据。

## JOIN 的分类
inner join，left join,right join，full join,
JOIN 从原理上来划分的话，可分为 Reduce 端的 JOIN 和 Map 端的 JOIN。
### Reduce 端 JOIN
如果不指定 MapJoin 或者不符合 MapJoin（Map阶段完成join） 的条件，那么 Hive 解析器会默认把执行 Common Join（Reduce阶段完成join），即在 Reduce 阶段完成 JOIN。整个过程包含 Map、Shuffle、Reduce 阶段
- Map 阶段：Map 端首先读取源表的数据，然后构建一个 kv 结构，将 SQL 中的关联字段（可以多个字段组合）作为 key，把需要用到的字段（即 select 或者 where 中用到的字段）作为 value，同时在 value 中还会包含表信息。
- Shuffle 阶段：根据 Map 端输出的 kv 中的 key 的值进行 hash，并将 key/value 按照 hash 值推送至不同的 Reduce 中，保证 JOIN 的两个表中相同的 key 取到同一个 Reduce 中。
- Reduce 阶段：由于相同 KEY 的数据会来到同一个 Reduce，所以在 Reduce 中只需要将 key 分组，然后区分不同的表来源进行笛卡尔积即可。

### Map 端 JOIN
MapJoin 通常用于一个小表（hive.mapjoin.smalltable.filesize 配置，默认 25M）和一个大表进行 JOIN 的场景，没有 Reduce 阶段，所以能有效提高执行的效率。

如图所示，会生成两个 Task，首先会启动一个任务扫描小表 tb_small，生成 HashTable 的数据结构文件，然后加载进分布式缓存 DistributeCache 中，第二个 Task 会扫描大表 tb_big，然后根据 tb_big 每一条数据中的关联字段去和 DistributeCache 中的 tb_small 表对应的 HashTable 做关联，并直接输出结果，因为没有 Reduce 阶段，所以输出的文件个数和 Mapper 的个数一致。
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Mapjoin%E5%8E%9F%E7%90%86%E5%9B%BE.png" width="400"></div>

## Hive 的存储格式
- TextFile：默认的存储格式，每一行都是一条记录，每行都以换行符（\n）结尾。数据不做压缩，磁盘开销大，数据解析开销大。可结合 Gzip、Bzip2 使用（系统自动检查，执行查询时自动解压）。
- SequenceFile：是 Hadoop API 提供的一种二进制文件支持，其具有使用方便、可分割、可压缩的特点。支持三种压缩选择：NONE、RECORD、BLOCK。Record 压缩率低，一般建议使用 BLOCK 压缩。
- RCFile：是一种行列存储相结合的存储方式。首先，其将数据按行分块，保证同一个 record 在一个块上，避免读一个 record 需要扫描多个 block。其次，block 内部数据存储采用列式存储的方式，有利于数据压缩和快速的列存取。
- AVRO：是为 Hadoop 提供数据序列化和数据交换服务，支持二进制序列化方式，可以便捷，快速地处理大量数据；动态语言友好，可以在 Hadoop 生态系统和以任何编程语言编写的程序之间交换数据。Avro 是基于大数据 Hadoop 的应用程序中流行的文件格式之一。
- Parquet/ORC：两者都是列式存储格式，把数据按照列的方式进行组织存储，在单列查询时就只需要遍历那一列，查询数据量就回小很多，对于指定列的海量数据查询非常高效，同时对单列数据的压缩也会有一个很好的压缩率。使用 Snappy 压缩。强烈推荐使用。
