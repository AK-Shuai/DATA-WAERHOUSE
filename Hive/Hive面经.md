# Hive 面试题集锦

## 数据优化中除了用过mapjoin之外，还用过哪些join( 不是常见的五种)
分桶连接：bucket map join

    hive.optimize.bucketmapjoin= true
倾斜连接：skew join https://blog.csdn.net/stay_forever/article/details/106913507/

    //指定每个reducer处理的数据量大小
    hive.exec.reducers.bytes.per.reducer = 1000000000
    //开启倾斜优化
    set hive.optimize.skewjoin = true;
    //设置数据条数（每个reduce数据条数上线） 
    set hive.skewjoin.key = skew_key_threshold （default = 100000）
## Data modeler
数据库建模工具 没用过 也不会用
## count(*)和count(1)区别
没区别只有count字段时候为空有区别

