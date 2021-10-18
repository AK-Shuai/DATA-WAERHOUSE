# Hive 四种排序方式
select语法中的order by、sort by、distribute by、cluster by、order by语法。

## order by语法

在hiveQL中Order by语法类似于sql语言中的order by语法。

```
colOrder: ( ASC | DESC )

colNullOrder: (NULLS FIRST | NULLS LAST)           -- (Note: Available in Hive 2.1.0 and later)

orderBy: ORDER BY colName colOrder? colNullOrder? (',' colName colOrder? colNullOrder?)*

query: SELECT expression (',' expression)* FROM src orderBy
```

有一些限制在order by子句中，在严格模式下（strict），order by子句必须带有以limit子句。Limit子句不是必须的，如果你设置hive.mapred.mode是非严格模式（nostrict）。原因是为了增强全局排序的结果，这里只有一个reduce去进行最终的排序输出，如果输出结果集行集太大，这单独的reduce可能需要花费非常长的时间去处理。

注意，这列必须使用的名字，而不能指定位置号，然而在hive 0.11.0以后，在做了下面配置之后列可以指定序列号，如下：

1）从hive0.11.0到2.1.x，设置hive.groupby.orderby.position.alias 是true（默认值是false）。

2）从hive2.2.0以后，hive.orderby.position.alias默认是ture。

Order by默认排序是asc。

在hive 2.1.0以后，被选择排序的每个列是null在order by子句是支持的。默认null排序规则是asc（升序）是nulls first，当默认的排序规则是desc（时）是nulls last



## sort by语法

在hiveQL中sort by语法类似于sql语言中的order by语法。

```
colOrder: ( ASC | DESC )

sortBy: SORT BY colName colOrder? (',' colName colOrder?)*

query: SELECT expression (',' expression)* FROM src sortBy
```

Hive中被用来sort by排序的行的排序操作发生在发送这些行到reduce之前。排序的次序依赖于排序列的类型，如果列是数值类型，那么排序次序按照数值排序，如果列式字符串类型，那么排序次序将按照字典排序。

1、sort by和order by的不同点

Hive sort by的排序发生在每个reduce里，order by和sort by之间的不同点是前者保证在全局进行排序，而后者仅保证在每个reduce内排序，如果有超过1个reduce，sort by可能有部分结果有序。

注意：它也许是混乱的作为单独列排序对于sort by和cluster by。不同点在于cluster by的分区列和sort by有多重reduce，reduce内的分区数据时一致的。

通常，数据在每个reduce排序通过用户指定的规则，如下示例：

```
SELECT key, value FROM src SORT BY key ASC, value DESC
```

2、设置排序类型

转换后，参数类型一般认为是string类型，意味着数值类型将按照字典排序方式进行排序，克服这一点，第二个select语句可以在sort by之前被使用。

```
FROM (FROM (FROM src

            SELECT TRANSFORM(value)

            USING 'mapper'

            AS value, count) mapped

      SELECT cast(value as double) AS value, cast(count as int) AS count

      SORT BY value, count) sorted

SELECT TRANSFORM(value, count)

USING 'reducer'

AS whatever
```

## Cluster By 和Distribute By语法

Cluster by和distribute by主要用在进行Transform/Map-Reduce脚本。但是，他有时可以应用在select语句，如果有一个子查询需要分区和排序对于输出结果集。

**Cluster by是一个捷径对于包含distribute by和sort by的语句**。

Hive使用distribute by中的列去分发行到每个reduce中，所有同样的distribute by列的行将发送到同一个reduce。然而，分发并不保证clustering和sorting的属性在分发关键字。

例如，我们按照x进行distribute by在5行数据到2个reduce中：
x1
x2
x4
x3
x1

Reduce 1 得到的数据：
x1
x2
x1
Reduce 2 得到的数据：
x4
x3

注意：这所有制是x1的行确保被分发到同一个reduce中，但是，他们不能保证在集合中会在相邻的位置。

相比之下，如果我们使用cluster by x，这两个reduce将进一步在x上进行排序。

Reduce 1得到数据：
x1
x1
x2

Reduce 2得到数据：
x3
x4

不能替代cluster by，用户可以指定distribute by和sort by，同样，分区列和排序列可以是不同的，通常情况，分区类是排序类的前缀，但是他并不是必须的。

```
SELECT col1, col2 FROM t1 CLUSTER BY col1

SELECT col1, col2 FROM t1 DISTRIBUTE BY col1

SELECT col1, col2 FROM t1 DISTRIBUTE BY col1 SORT BY col1 ASC, col2 DESC

FROM (

FROM pv_users

MAP ( pv_users.userid, pv_users.date )

USING 'map_script'

AS c1, c2, c3

DISTRIBUTE BY c2

SORT BY c2, c1) map_output

INSERT OVERWRITE TABLE pv_users_reduced

REDUCE ( map_output.c1, map_output.c2, map_output.c3 )

USING 'reduce_script'

AS date, count;
```