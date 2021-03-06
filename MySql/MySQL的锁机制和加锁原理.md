# MySQL的锁机制和加锁原理


首先对mysql锁进行划分：
按照锁的粒度划分：行锁、表锁、页锁
按照锁的使用方式划分：共享锁、排它锁（悲观锁的一种实现）
还有两种思想上的锁：悲观锁、乐观锁。
InnoDB中有几种行级锁类型：Record Lock、Gap Lock、Next-key Lock
Record Lock：在索引记录上加锁
Gap Lock：间隙锁
Next-key Lock：Record Lock+Gap Lock


## 行锁

行级锁是Mysql中锁定粒度最细的一种锁，表示只针对当前操作的行进行加锁。行级锁能大大减少数据库操作的冲突。其加锁粒度最小，但加锁的开销也最大。有可能会出现死锁的情况。 行级锁按照使用方式分为共享锁和排他锁。

**共享锁用法（S锁 读锁）**：

​ 若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。
```sql
select ... lock in share mode;
```

共享锁就是允许多个线程同时获取一个锁，一个锁可以同时被多个线程拥有。

**排它锁用法（X 锁 写锁）**：

​ 若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。
```
select ... for update
```
​排它锁，也称作独占锁，一个锁在某一时刻只能被一个线程占有，其它线程必须等待锁被释放之后才可能获取到锁。


## 表锁

​表级锁是mysql锁中粒度最大的一种锁，表示当前的操作对整张表加锁，资源开销比行锁少，不会出现死锁的情况，但是发生锁冲突的概率很大。被大部分的mysql引擎支持，MyISAM和InnoDB都支持表级锁，但是InnoDB默认的是行级锁。

**共享锁用法**：
```sql
LOCK TABLE table_name [ AS alias_name ] READ
```

**排它锁用法**：
```sql
LOCK TABLE table_name [AS alias_name][ LOW_PRIORITY ] WRITE
```

**解锁用法**：
```sql
unlock tables;
```

## 页锁
​页级锁是MySQL中锁定粒度介于行级锁和表级锁中间的一种锁。表级锁速度快，但冲突多，行级冲突少，但速度慢。所以取了折衷的页级，一次锁定相邻的一组记录。BDB支持页级锁

## 乐观锁和悲观锁

在数据库的锁机制中介绍过，数据库管理系统（DBMS）中的并发控制的任务是确保在多个事务同时存取数据库中同一数据时不破坏事务的隔离性和统一性以及数据库的统一性。

​乐观并发控制(乐观锁)和悲观并发控制（悲观锁）是并发控制主要采用的技术手段。

​无论是悲观锁还是乐观锁，都是人们定义出来的概念，可以认为是一种思想。其实不仅仅是关系型数据库系统中有乐观锁和悲观锁的概念，像memcache、hibernate、tair等都有类似的概念。

​针对于不同的业务场景，应该选用不同的并发控制方式。所以，不要把乐观并发控制和悲观并发控制狭义的理解为DBMS中的概念，更不要把他们和数据中提供的锁机制（行锁、表锁、排他锁、共享锁）混为一谈。其实，在DBMS中，悲观锁正是利用数据库本身提供的锁机制来实现的。

### 悲观锁

在关系数据库管理系统里，悲观并发控制（又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”）是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作对某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。悲观并发控制主要用于数据争用激烈的环境，以及发生并发冲突时使用锁保护数据的成本要低于回滚事务的成本的环境中。

​悲观锁，正如其名，它指的是对数据被外界（包括本系统当前的其他事务，以及来自外部系统的事务处理）修改持保守态度(悲观)，因此，在整个数据处理过程中，将数据处于锁定状态。 悲观锁的实现，往往依靠数据库提供的锁机制 （也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）

**悲观锁的具体流程**：
在对任意记录进行修改前，先尝试为该记录加上排他锁（exclusive locking）
如果加锁失败，说明该记录正在被修改，那么当前查询可能要等待或者抛出异常。 具体响应方式由开发者根据实际需要决定。

如果成功加锁，那么就可以对记录做修改，事务完成后就会解锁了。

其间如果有其他对该记录做修改或加排他锁的操作，都会等待我们解锁或直接抛出异常。

**在mysql/InnoDB中使用悲观锁**：
​首先我们得关闭mysql中的autocommit属性，因为mysql默认使用自动提交模式，也就是说当我们进行一个sql操作的时候，mysql会将这个操作当做一个事务并且自动提交这个操作。
```sql
1.开始事务
begin;/begin work;/start transaction; (三者选一就可以)
2.查询出商品信息
select ... for update;
4.提交事务
commit;/commit work;
```

**悲观锁的优点和不足**：
悲观锁实际上是采取了“先取锁在访问”的策略，为数据的处理安全提供了保证，但是在效率方面，由于额外的加锁机制产生了额外的开销，并且增加了死锁的机会。并且降低了并发性；当一个事物所以一行数据的时候，其他事物必须等待该事务提交之后，才能操作这行数据。

### 乐观锁

乐观锁（ Optimistic Locking ） 相对悲观锁而言，乐观锁假设认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则让返回用户错误的信息，让用户决定如何去做。

​相对于悲观锁，在对数据库进行处理的时候，乐观锁并不会使用数据库提供的锁机制。一般的实现乐观锁的方式就是记录数据版本。

**乐观锁的优点和不足**：

​乐观并发控制相信事务之间的数据竞争(data race)的概率是比较小的，因此尽可能直接做下去，直到提交的时候才去锁定，所以不会产生任何锁和死锁。但如果直接简单这么做，还是有可能会遇到不可预期的结果，例如两个事务都读取了数据库的某一行，经过修改以后写回数据库，这时就遇到了问题。

## InnoDB锁的特性

1. 在不通过索引条件查询的时候，InnoDB使用的确实是表锁！
2. 由于 MySQL 的行锁是针对索引加的锁,不是针对记录加的锁,所以虽然是访问不同行 的记录,但是如果是使用相同的索引键,是会出现锁冲突的。
3. 当表有多个索引的时候,不同的事务可以使用不同的索引锁定不同的行,另外,不论 是使用主键索引、唯一索引或普通索引,InnoDB 都会使用行锁来对数据加锁。
4. 即便在条件中使用了索引字段,但是否使用索引来检索数据是由 MySQL 通过判断不同 执行计划的代价来决定的,如果 MySQL 认为全表扫 效率更高,比如对一些很小的表,它 就不会使用索引,这种情况下 InnoDB 将使用表锁,而不是行锁。因此,在分析锁冲突时, 别忘了检查 SQL 的执行计划（explain查看）,以确认是否真正使用了索引。

## Record Lock、Gap Lock、Next-key Lock锁

### Record Lock
单条索引上加锁，record lock 永远锁的是索引，而非数据本身，如果innodb表中没有索引，那么会自动创建一个隐藏的聚集索引，锁住的就是这个聚集索引。所以说当一条sql没有走任何索引时，那么将会在每一条聚集索引后面加X锁，这个类似于表锁，但原理上和表锁应该是完全不同的。
### Gap Lock
间隙锁，是在索引的间隙之间加上锁，这是为什么Repeatable Read隔离级别下能防止幻读的主要原因。

#### 什么事间隙锁
```
mysql> select * from product_copy;
+----+--------+-------+-----+
| id | name   | price | num |
+----+--------+-------+-----+
|  1 | 伊利   |    68 |   1 |
|  2 | 蒙牛   |    88 |   1 |
|  6 | tom    |  2788 |   3 |
| 10 | 优衣库 |   488 |   4 |
+----+--------+-------+-----+
其中id为主键 num为普通索引
窗口A：
mysql> select * from product_copy where num=3 for update;
+----+------+-------+-----+
| id | name | price | num |
+----+------+-------+-----+
|  6 | tom  |  2788 |   3 |
+----+------+-------+-----+
1 row in set

窗口B：
mysql> insert into product_copy values(5,'kris',1888,2);
这里会等待  直到窗口A commit才会显示下面结果
Query OK, 1 row affected

但是下面是不需要等待的
mysql> update product_copy set price=price+100 where num=1;
Query OK, 2 rows affected
Rows matched: 2  Changed: 2  Warnings: 0
mysql> insert into product_copy values(5,'kris',1888,5);
Query OK, 1 row affected
```
通过上面的例子可以看出Gap 锁的作用是在1,3的间隙之间加上了锁。而且并不是锁住了表，我更新num=1，5的数据是可以的.可以看出锁住的范围是（1,3]U[3,4)。

#### 为什么说gap锁是RR隔离级别下防止幻读的主要原因。
解决幻读的方式很简单，就是需要当事务进行当前读的时候，保证其他事务不可以在满足当前读条件的范围内进行数据操作。

​根据索引的有序性，我们可以从上面的例子推断出满足where条件的数据，只能插入在num=（1,3]U[3,4)两个区间里面，只要我们将这两个区间锁住，那么就不会发生幻读。

#### 主键索引/唯一索引+当前读会加上Gap锁吗
```sql
窗口A：
mysql> select * from product_copy where id=6 for update;
+----+------+-------+-----+
| id | name | price | num |
+----+------+-------+-----+
|  6 | tom  |  2788 |   3 |
+----+------+-------+-----+

窗口B：并不会发生等待
mysql> insert into product_copy values(5,'kris',1888,3);
Query OK, 1 row affected
```
例子说明的其实就是行锁的原因，我只将id=6的行数据锁住了，用Gap锁的原理来解释的话：因为主键索引和唯一索引的值只有一个，所以满足检索条件的只有一行，故并不会出现幻读，所以并不会加上Gap锁。

#### 通过范围查询是否会加上Gap锁
​前面的例子都是通过等值查询，下面测试一下范围查询。
```sql
窗口A：
mysql> select * from product_copy where num>3 for update;
+----+--------+-------+-----+
| id | name   | price | num |
+----+--------+-------+-----+
| 10 | 优衣库 |   488 |   4 |
+----+--------+-------+-----+

窗口B：会等待
mysql> insert into product_copy values(11,'kris',1888,5);
Query OK, 1 row affected
不会等待
mysql> insert into product_copy values(3,'kris',1888,2);
Query OK, 1 row affected
```

#### 检索条件并不存在的当前读会加上Gap吗
1. 等值查询
```sql
窗口A：
mysql> select * from product_copy where num=5 for update;
Empty set

窗口B：6 和 4都会等待
mysql> insert into product_copy values(11,'kris',1888,6);
Query OK, 1 row affected

mysql> insert into product_copy values(11,'kris',1888,4);
Query OK, 1 row affected
```
原因一样会锁住（4,5]U[5,n）的区间

2. 范围查询
这里就会有点不一样
```sql
窗口A：
mysql> select * from product_copy where num>6 for update;
Empty set
窗口B：8 和 4 都会锁住
mysql> insert into product_copy values(11,'kris',1888,4);
Query OK, 1 row affected

mysql> insert into product_copy values(11,'kris',1888,8);
Query OK, 1 row affected
```
上面的2例子看出当你查询并不存在的数据的时候，mysql会将有可能出现区间全部锁住。

### Next-Key Lock
这个锁机制其实就是前面两个锁相结合的机制，既锁住记录本身还锁住索引之间的间隙。

## 死锁的原理及分析
### MVCC
​MySQL InnoDB存储引擎，实现的是基于多版本并发控制协议—MVCC(Multi-Version Concurrency Control) MVCC最大的好处，相信也是耳熟能详：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能，这也是为什么现阶段，几乎所有的RDBMS，都支持了MVCC。

### 2PL：Two-Phase Locking

​传统RDBMS（关系数据库管理系统）加锁的一个原则，就是2PL (二阶段锁)：Two-Phase Locking。相对而言，2PL比较容易理解，说的是锁操作分为两个阶段：加锁阶段与解锁阶段，并且保证加锁阶段与解锁阶段不相交。下面，仍旧以MySQL为例，来简单看看2PL在MySQL中的实现。

transaction	| mysql
---|:--:
begin	| 加锁阶段
insert | into	加insert对应的锁
update | table	加update对应的锁
delete | from	加delete对应的锁
commit	| 解锁阶段

上面的例子可以看出2PL就是将加锁、解锁分为两个阶段，并且互相不干扰。加锁阶段只加锁，解锁阶段只解锁。

### 为什么会发生死锁

MyISAM中是不会产生死锁的，因为MyISAM总是一次性获得所需的全部锁，要么全部满足，要么全部等待。而在InnoDB中，锁是逐步获得的，就造成了死锁的可能。（不过现在一般都是InnoDB引擎，关于MyISAM不做考虑）

​在InnoDB中，行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql语句操作了主键索引，MySQL就会锁定这条主键索引；如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引。

​当两个事务同时执行，一个锁住了主键索引，在等待其他相关索引。另一个锁定了非主键索引，在等待主键索引。这样就会发生死锁。

避免死锁，这里只介绍常见的三种

1. 如果不同程序会并发存取多个表，尽量约定以相同的顺序访问表，可以大大降低死锁机会。
2. 在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率；
3. 对于非常容易产生死锁的业务部分，可以尝试使用升级锁定颗粒度，通过表级锁定来减少死锁产生的概率；

**参考**：
1. <a href="https://blog.csdn.net/qq_38238296/article/details/88362999?ops_request_misc=&request_id=&biz_id=102&utm_term=mysql%20%E9%94%81&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-88362999.first_rank_v2_pc_rank_v29&spm=1018.2226.3001.4187" target="_blank">本文参考</a>
2. <a href="https://www.jianshu.com/p/b5c01bd4a306" target="_blank">mysql执行计划</a>
3. <a href="https://blog.csdn.net/qq_38238296/article/details/88363017" target="_blank">mysql幻读</a>
