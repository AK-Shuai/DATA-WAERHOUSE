# Zookeeper 特性与对应的场景
Zookeeper 具有一些数据库不具备的特性，支撑这它可以实现比较复杂的场景，这里我们从应用出发，看看 Zookeeper 到底有哪些特性，以及这些特性有哪些丰富的应用场景。
## 重点：特性、watch、节点类型、分布式锁。

## Zookeeper 的特性

有序性： 因为每个事务请求都会有一个事务编号，所以从同一客户端发起的事务请求，最终将会严格地按照顺序被应用到 ZooKeeper 中去。 
原子性： 所有事务请求的处理结果最终在整个集群中所有机器上的应用情况是一致的，即一个事务要么所有节点都执行了要么所有节点都没执行。 
单一系统映像 ： 客户端连任意一个 Zookeeper 集群的 Server，获取到的数据都是一致的。（数据同步延迟的特殊情况在上一篇也说过了，是可以通过 sync()方法来做到完全一致的） 
可靠性： 一旦一次事务请求被 Commit，那么该请求就会被大部分节点持久化，直到被下一次更改覆盖。且 Zab 协议保证了一般以上节点存活集群就能正常运行。