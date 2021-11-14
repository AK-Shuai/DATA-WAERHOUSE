- [使用语法](#使用语法：)
- [实践问题](#实践问题：)
- [参考](#参考：)

# Hive 的 explain 执行计划详解
HIVE提供了EXPLAIN命令来展示一个查询的执行计划,这个执行计划对于我们了解底层原理，hive 调优，排查数据倾斜等很有帮助

## 使用语法：
```
EXPLAIN [EXTENDED|CBO|AST|DEPENDENCY|AUTHORIZATION|LOCKS|VECTORIZATION|ANALYZE] query
```
explain 后面可以跟以下可选参数，**注意**：这几个可选参数不是 hive 每个版本都支持的：
1. EXTENDED：加上 extended 可以输出有关计划的额外信息。这通常是物理信息，例如文件名。这些额外信息对我们用处不大

2. CBO：输出由Calcite优化器生成的计划。CBO 从 hive 4.0.0 版本开始支持

3. AST：输出查询的抽象语法树。AST 在hive 2.1.0 版本删除了，存在bug，转储AST可能会导致OOM错误，将在4.0.0版本修复

4. DEPENDENCY：dependency在EXPLAIN语句中使用会产生有关计划中输入的额外信息。它显示了输入的各种属性

5. AUTHORIZATION：显示所有的实体需要被授权执行（如果存在）的查询和授权失败

6. LOCKS：这对于了解系统将获得哪些锁以运行指定的查询很有用。LOCKS 从 hive 3.2.0 开始支持

7. VECTORIZATION：将详细信息添加到EXPLAIN输出中，以显示为什么未对Map和Reduce进行矢量化。从 Hive 2.3.0 开始支持

8. ANALYZE：用实际的行数注释计划。从 Hive 2.2.0 开始支持

在 hive cli 中输入以下命令(hive 2.3.7)：
```
explain select sum(id) from test1;
```

得到结果：
``` 
STAGE DEPENDENCIES:
  Stage-1 is a root stage
  Stage-0 depends on stages: Stage-1

STAGE PLANS:
  Stage: Stage-1
    Map Reduce
      Map Operator Tree:
          TableScan
            alias: test1
            Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
            Select Operator
              expressions: id (type: int)
              outputColumnNames: id
              Statistics: Num rows: 6 Data size: 75 Basic stats: COMPLETE Column stats: NONE
              Group By Operator
                aggregations: sum(id)
                mode: hash
                outputColumnNames: _col0
                Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
                Reduce Output Operator
                  sort order:
                  Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
                  value expressions: _col0 (type: bigint)
      Reduce Operator Tree:
        Group By Operator
          aggregations: sum(VALUE._col0)
          mode: mergepartial
          outputColumnNames: _col0
          Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
          File Output Operator
            compressed: false
            Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: NONE
            table:
                input format: org.apache.hadoop.mapred.SequenceFileInputFormat
                output format: org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat
                serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe

  Stage: Stage-0
    Fetch Operator
      limit: -1
      Processor Tree:
        ListSink
```

我们将上述结果拆分看，先从最外层开始，包含两个大的部分：
1. stage dependencies： 各个stage之间的依赖性
2. stage plan： 各个stage的执行计划

先看第一部分 stage dependencies ，包含两个 stage，Stage-1 是根stage，说明这是开始的stage，Stage-0 依赖 Stage-1，Stage-1执行完成后执行Stage-0。

再看第二部分 stage plan，里面有一个 Map Reduce，一个MR的执行计划分为两个部分：
1. Map Operator Tree： MAP端的执行计划树
2. Reduce Operator Tree： Reduce端的执行计划树

这两个执行计划树里面包含这条sql语句的 operator：

1. map端第一个操作肯定是加载表，所以就是 TableScan 表扫描操作，常见的属性：
   - alias： 表名称
   - Statistics： 表统计信息，包含表中数据条数，数据大小等

2. Select Operator： 选取操作，常见的属性 ：
   - expressions：需要的字段名称及字段类型
   - outputColumnNames：输出的列名称
   - Statistics：表统计信息，包含表中数据条数，数据大小等
3. Group By Operator：分组聚合操作，常见的属性：
   - aggregations：显示聚合函数信息
   - mode：聚合模式，值有 hash：随机聚合，就是hash partition；partial：局部聚合；final：最终聚合
   - keys：分组的字段，如果没有分组，则没有此字段
   - outputColumnNames：聚合之后输出列名
   - Statistics： 表统计信息，包含分组聚合之后的数据条数，数据大小等

4. Reduce Output Operator：输出到reduce操作，常见属性：
   - sort order：值为空 不排序；值为 + 正序排序，值为 - 倒序排序；值为 +-  排序的列为两列，第一列为正序，第二列为倒序

5. Filter Operator：过滤操作，常见的属性：
   - predicate：过滤条件，如sql语句中的where id>=1，则此处显示(id >= 1)

6. Map Join Operator：join 操作，常见的属性：
   - condition map：join方式 ，如Inner Join 0 to 1 Left Outer Join0 to 2
   - keys: join 的条件字段
   - outputColumnNames： join 完成之后输出的字段
   - Statistics： join 完成之后生成的数据条数，大小等
7. File Output Operator：文件输出操作，常见的属性
   - compressed：是否压缩
   - table：表的信息，包含输入输出文件格式化方式，序列化方式等
8. Fetch Operator 客户端获取数据操作，常见的属性：
   - limit，值为 -1 表示不限制条数，其他值为限制的条数

## 实践问题：
1. join 语句会过滤 null 的值吗？
join 时会自动过滤掉关联字段为 null 值的情况，但 left join 或 full join 是不会自动过滤的。
2. group by 分组语句会进行排序吗？
sort order: + ，说明是按照 id 字段进行正序排序的
3. 哪条sql执行效率高呢？
```sql
SELECT
    a.id,
    b.user_name
FROM
    test1 a
JOIN test2 b ON a.id = b.id
WHERE
    a.id > 2;

SELECT
    a.id,
    b.user_name
FROM
    (SELECT * FROM test1 WHERE id > 2) a
JOIN test2 b ON a.id = b.id;
```
hive 底层会自动帮我们进行优化，所以这两条sql语句执行效率是一样的。


## 参考：
1. <a href="https://mp.weixin.qq.com/s/5a8bBEDgxErBfkhLsTS70g" target="_blank">explain计划详解</a>

