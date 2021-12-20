- [查询速度慢 如何优化](#查询速度慢-如何优化)
- [如何优化JOIN性能](#如何优化JOIN性能)
- [如何使查询简单化](#如何使查询简单化)
- [Exceeded max memory 错误](#Exceeded-max-memory-错误)
- [查询生成的大量数据优化的问题](#查询生成的大量数据优化的问题)
- [如何拼接字符串](#如何拼接字符串)
- [如何在字段包含NULL的情况下 添加default value](#如何在字段包含NULL的情况下-添加default-value)
- [如何从两个数中选出最大或最小值](#如何从两个数中选出最大或最小值)

# Presto性能优化

## 查询速度慢 如何优化
### 避免单节点处理
虽然Presto是分布式查询引擎, 但是一些操作是必须在单节点中处理的. 例如:

1. count(distinct x)
   - 考虑使用approx_distinct(x)代替
   - 但是需要注意这个函数有个大约在2.3%的标准误差, 如果需要精确统计的情况, 请绕道.
2. UNION
   - UNION有个功能是: 如果两条记录一样, 会只保留一条记录(去重).
   - 如果不考虑去重的情况, 请使用UNION ALL
3. ORDER BY
   - Presto对数据排序是作用在单节点上的
   - 如果要排序的数据量超过百万行, 要谨慎考虑. 如果非要排序,尽量将排序的字段减少些.
   - 最好和 Limit 联合使用

### 减少表扫描的范围
通过添加条件达到减少表扫描的范围.

也可以考虑将大数据量的表, 水平查分, 通过查不同的表分区达到效果.

### 避免使用 SELECT * FROM
要明确写出所有要访问的列, 能加快速度.  
例如:

```sql
SELECT * FROM my_table
```

改成:

```sql
SELECT id, name, address FROM my_table
```

### 将几个LIKE语句放到函数regexp_like
Presto的查询优化器不能改善许多LIKE语句使用的地方, 导致这样的语句查询速度慢.

例如:
```sql
SELECT
  ...
FROM
  access
WHERE
  method LIKE '%GET%' OR
  method LIKE '%POST%' OR
  method LIKE '%PUT%' OR
  method LIKE '%DELETE%'
```

上面的语句能用regexp_like函数优化成一句:

```sql
SELECT
  ...
FROM
  access
WHERE
  regexp_like(method, 'GET|POST|PUT|DELETE')
```

---
## 如何优化JOIN性能

尽量让JOIN的条件简单，最好是ON后面的比较表达式两边必涉及计算。

例如
```sql
SELECT a.date, b.name FROM
left_table a
JOIN right_table b
ON a.date = CAST((b.year * 10000 + b.month * 100 + b.day) as VARCHAR)
```
上面的SQL语句的JOIN性能不高，因为JION条件包含了表达式计算。我们可以通过子查询的形式来优化上面的语句。
```sql
SELECT a.date, b.name FROM
left_table a
JOIN (
  SELECT
    CAST((b.year * 10000 + b.month * 100 + b.day) as VARCHAR) date,  # generate join key
    name
  FROM right_table
) b
ON a.date = b.date  # Simple equi-join
```
上面的语句，就是直接比较两个VARCHAR的值，这样会比比较一个VARCHAR和一个表达式结果的性能高。

我们还能继续优化，使用Presto的WITH语句进行子查询。
```sql
WITH b AS (
  SELECT
    CAST((b.year * 10000 + b.month * 100 + b.day) as VARCHAR) date,  # generate join key
    name
  FROM right_table
)
SELECT a.date, b.name FROM
left_table a
JOIN b
ON a.date = b.date
```

---
## 如何使查询简单化
### 使用WITH语句
如果你的查询语句非常复杂或者有多层嵌套的子查询，请试着用WITH语句将子查询分离出来。

例如：
``` sql
SELECT a, b, c FROM (
   SELECT a, MAX(b) AS b, MIN(c) AS c FROM tbl GROUP BY a
) tbl_alias
```
可以被重写为线面的形式
```sql
WITH tbl_alias AS (SELECT a, MAX(b) AS b, MIN(c) AS c FROM tbl GROUP BY a)
SELECT a, b, c FROM tbl_alias
```
同样，也可以将各个步骤的子查询通过WITH语句罗列出来，子查询之间用“,”分割。
```sql
WITH tbl1 AS (SELECT a, MAX(b) AS b, MIN(c) AS c FROM tbl GROUP BY a),
     tbl2 AS (SELECT a, AVG(d) AS d FROM another_tbl GROUP BY a)
SELECT tbl1.*, tbl2.* FROM tbl1 JOIN tbl2 ON tbl1.a = tbl2.a
```

### 在CREATE TABLE语句中使用WITH语句
如果CREATE TABLE语句的查询部分很复杂或者潜逃了多层子查询，就需要考虑用WITH语句

例如：
```sql
CREATE TABLE tbl_new AS WITH tbl_alias AS (SELECT a, MAX(b) AS b, MIN(c) AS c FROM tbl1)
SELECT a, b, c FROM tbl_alias
```
```sql
CREATE TABLE tbl_new AS WITH tbl_alias1 AS (SELECT a, MAX(b) AS b, MIN(c) AS c FROM tbl1),
                             tbl_alias2 AS (SELECT a, AVG(d) AS d FROM tbl2)
SELECT tbl_alias1.*, tbl2_alias.* FROM tbl_alias1 JOIN tbl_alias2 ON tbl_alias1.a = tbl_alias2.a
```

### 用GROUP BY语句时，GROUP BY的目标可用数字代替
在Presto SQL中，GROUP BY语句需要与SELECT语句中的表达式保持一致，不然会提示语法错误。

例如：
```sql
SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd HH', 'PDT') hour, count(*) cnt
FROM my_table
GROUP BY TD_TIME_FORMAT(time, 'yyyy-MM-dd HH', 'PDT') 
```
上面的SQL语句的GROUP BY部分可以用GROUP BY 1，2，3 ...来表示
```sql
SELECT TD_TIME_FORMAT(time, 'yyyy-MM-dd HH', 'PDT') hour, count(*) cnt
FROM my_table
GROUP BY 1
```

---
## Exceeded max memory 错误
Presto会跟踪每个查询的内存使用情况.可用内存的多少是根据你的查询计划变动的,所以在大多数情况下可以从写查询语句来达到优化内存使用的目的.

下面列出来的就是内存密集型的语句块:

- district
- UNION
- ORDER BY
- GROUP BY (许多字段的情况)
- joins (各种JOIN)


### 尽量少使用distinct
distinct 会排除所有不唯一的行.下面的例子就是检查你的数据表中是否包含了相同的数据行(c1,c2,c3)
```sql
SELECT distinct c1, c2, c3 FROM my_table
```
上面的操作会存储一整字段c1,c2和c3到presto的单个工作节点的内存, 然后检查(c1,c2,c3)的唯一性. 随着字段的增多以及字段数据量的增大,所需要的内存也会直线上升.

所以, 去掉查询语句中的distinct关键字, 或者只在子查询(有有限少量字段的情况下)使用.

### 用approx_distinct(x)代替count(distinct x)
NOTE: approx_distinct(x)会返回一个正确的近似值, 如果只是需要看一个大概的趋势,可以考虑.

### 尽量用UNION ALL代替UNION
和distinct的原因类似, UNION有去重的功能, 所以会引发内存使用的问题.如果你只是拼接两个或者多个SQL查询的结果, 考虑用UNION ALL

### 尽量避免ORDER BY
```sql
SELECT c1, c2 FROM my_table ORDER BY c1
```
Presto在排序的时候启用的是单一节点进行工作, 所以整个数据需要在单节点内存限制的范围内, 超过这个内存限制就会报错.

如果你需要排序的数据在一个小的量级, 用ORDER BY没有问题; 如果需要排序的数据在GB的级别,需要考虑其他的解决方案.

例如: 大量级的数据排序可以考虑结合HIVE和presto. 首先, 用Presto将大量的数据存储到一个临时表中,然后用HIVE取对数据排序.

### 减少GROUP BY的字段
```sql
SELECT avg(c1), min_by(c2, time), max(c3), count(c4), ...
FROM my_table
GROUP BY c1, c2, c3, c4, ...
```
减少GROUP BY语句后面的排序一句字段的数量能减少内存的使用.

### 用大表取JOIN小表
下面这种用小数据表去JOIN大数据表的查询会极度消耗内存.
```sql
SELECT * FROM small_table, large_table
WHERE small_table.id = large_table.id
```
Presto 会默认执行广播式的JOIN操作,它会将左表拆分到几个工作节点上, 然后发送整个右表分别到已拆分好的处理左表的工作节点上. 如果右表非常大就会超出工作节点的内存限制,进而出错.
所以需要用小表JOIN大表
```sql
SELECT * FROM large_table, small_table
WHERE large_table.id = small_table.id
```
如果左表和右表都比较大怎么办?

1. 修改配置distributed-joins-enabled (presto version >=0.196)
2. 在每次查询开始使用distributed_join的session选项
```sql
-- set session distributed_join = 'true'
SELECT * FROM large_table, large_table1
WHERE large_table1.id = large_table.id
```
核心点就是使用distributed join. Presto的这种配置类型会将左表和右表同时以join key的hash value为分区字段进行分区. 所以即使右表也是大表,也会被拆分.缺点是会增加很多网络数据传输, 所以会比broadcast join的效率慢.

---
## 查询生成的大量数据优化的问题
Presto用JOSN text的形式保存数据。如果查询出来的数据大于100G，Presto将传输大于100G的JSON text来保存查询结果。所以，即使查询处理即将完成，输出这么大的JOSN text也会消耗很长时间。

### 不要用==SELECT *==

### 用result_output_redirect='true' 注释
在查询语句前添加注释（result_output_redirect='true'），能让查询更快些。
```sql
-- set session result_output_redirect='true'
select a, b, c, d FROM my_table
```
上面的语句能让Presto用并行的方式生成查询结果，能跳过在Presto协调器进行JSON转换的过程。

Note: 但是，如果使用了ORDER BY语句，这个魔术注释将被忽略。

---
## 如何拼接字符串

### 用 || 运算符
```sql
SELECT 'hello ' || 'presto'
```

---
## 如何在字段包含NULL的情况下 添加default value

### 用COALESCE(v1,v2,...)函数
```sql
-- This retuns 'N/A' if name value is null
SELECT COALESCE(name, 'N/A') FROM table1
```

---
## 如何从两个数中选出最大或最小值

### 用greatest / least 函数
```sql
SELECT greatest(5, 10) -- returns 10
```

### Binary函数的应用
这里主要的问题是：如何将binary/varbinary类型转换为varchar类型

### 转换SHA256/MD5
```sql
SELECT to_hex(sha256(to_utf8('support@treasure-data.com'))) as email

SELECT to_hex(md5(to_utf8('support@treasure-data.com'))) as email
```

### 转换base64
```sql
SELECT to_base64(to_utf8('support@treasure-data.com')) as email
=> "c3VwcG9ydEB0cmVhc3VyZS1kYXRhLmNvbQ=="
SELECT FROM_UTF8(from_base64('c3VwcG9ydEB0cmVhc3VyZS1kYXRhLmNvbQ=='))
=> "support@treasure-data.com"
```

## 参考
1. <a href="https://developer.aliyun.com/article/614436" target="_blank">Presto性能优化</a>
2. <a href="超链接地址" target="_blank">超链接名</a>
