# Spark API算子

Spark 从 RDD -> DataFrame -> DataSet 中算子是有更新或者删除的。

- [算子对比](#算子对比)

## 算子对比

1. RDD filter 等价于 DF/DS where

2. union 使用上并没有区别 
    - 但ds和df的union算子有所优化，效率更高。RDD直接将两个RDD相加。而ds和df则中间调用了CombineUnions函数，关键字combine，Combines all adjacent [[Union]] operators into a single [[Union]].先将邻近的分区合并，避免多余的网络传输。

3. 求交集和差集
    - RDD 交集（intersection）差集（subtract）
    - DF/DS 交集（intersect）差集（except）

4. distinct 升级版函数 dropDuplicates
    - df/ds提供了新的去重算子dropDuplicates。传统的distinct  只能对元素全量去重，dropDuplicates可以针对元素的某一个或者多个属性进行去重。

5. groupBy 区别
    - groupBy对(K,V)对的RDD经过处理后，变成 [key,{key:value1,key:value2}]。而df/ds经过groupBy操作后，变成了RelationalGroupedDataset对象。必须经过后续操作，才能继续使用。后续操作具体主是聚合函数。

6. sort 和 sortBykey
    - df/df已经摒弃了sortBykey等函数，两者没有区别。

7. join
   - 相比RDD，df/ds的join算子增加了连接条件。


**参考**：  
1. <a href="https://www.cnblogs.com/eryuan/p/7279514.html" target="_blank">spark算子之DataFrame和DataSet</a>
