# HW和LEO关系

先对 kafka 高可用机制的基本原理开个篇：

1. kafka 高可用是通过 leader-follower 多副本机制同步完成
2. 每一个分区含有多个副本 (分区是真正存储日志消息的物理存储介质)
3. 多个副本只有一个 leader，也只有它对外提供服务 (读写服务都是)
4. 其他 follower副本 只是一个备份冗余
5. follower副本 是通过不断向 leader 副本 发送 fetch 以同步消息

所以如果 leader 副本 挂了，其中一个 follower副本 就会成为该分区下新的 leader副本。问题就来了，在选为新的 leader副本 时，old leader 副本 消息没有同步完全，会导致消息丢失或者离散吗？Kafka 是如何解决 leader副本 变更时消息不会出错？


## 参考
1. <a href="https://juejin.cn/post/6979110739416088607" target="_blank">kafka HW 备份机制</a>