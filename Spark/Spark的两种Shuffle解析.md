# Hash Shuffle 和 Sort Shuffle 区别

## Hash Shuffle 
### 普通机制
每个 Executor 执行当前一个 task 会生成下一个总 task 数量的文件数。

例如：如果当前 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 task，那么每个 Executor 上总共就要创建 500 个磁盘文件，所有 Executor 上会创建 5000 个磁盘文件。

### 优化后的
优化后做了分组，每个 Executor 执行多少个 task 只会生成下一个总 task 数量的文件数.

例如：每个 Executor 创建的磁盘文件的数量的计算公式为：cpu core的数量 * 下一个stage的task数量，也就是说，每个 Executor 此时只会创建 100 个磁盘文件，所有 Executor 只会创建 1000 个磁盘文件。

**自我总结**：
缩减是每个 Executor 执行 task 个数的基数。

基于 Hash 的 Shuffle 机制的优缺点   
优点：  
- 可以省略不必要的排序开销。
- 避免了排序所需的内存开销。

缺点：  
- 生产的文件过多，会对文件系统造成压力。
- 大量小文件的随机读写带来一定的磁盘开销。
- 数据块写入时所需的缓存空间也会随之增加，对内存造成压力。

## Sort Shuffle

### 普通机制
每个 Executor 执行每个 task 只会有一个文件。

例如：第一个 stage 有 50 个 task，总共有 10 个 Executor，每个 Executor 执行 5 个 task，而第二个 stage 有 100 个 task。由于每个 task 最终只有一个磁盘文件，因此此时每个 Executor 上只有 5 个磁盘文件，所有 Executor 只有 50 个磁盘文件。

### bypass机制

**bypass 运行机制的触发条件如下**    
- shuffle map task 数量小于spark.shuffle.sort.bypassMergeThreshold=200参数的值。
- 不是聚合类的 shuffle 算子。

和未优化 Hash Shuffle 差不多，但是用了磁盘合并，相对于 Hash Shuffle read 性能好一些。

## 总结
1. 为什么废弃 Hash Shuffle
废弃 Hash Shuffle 主要是因为产生文件不可控。
2. Sort Shuffle 为什么需要排序
在落盘的情况下，因为文件数的减少会导致 reduce 拉数据无法知道相同 key 在哪文件下，多文件会慢但是 key 会更清晰。
3. 为什么会有 bypass 机制
Sort Shuffle 需要排序，会极大消耗计算能力，而下一个 reduce task 数量小的情况下，采用 bypass 机制，由于不用排序，文件数在可控范围内，极大减少计算时间。
