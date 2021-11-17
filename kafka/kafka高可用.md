- [什么是高可用](#什么是高可用)
- [Replica备份机制](#Replica备份机制)
- [ISR机制](#ISR机制)
- [ACK机制](#ACK机制)
- [故障恢复机制](#故障恢复机制)
- [Kafka的复制机制](#Kafka的复制机制)
- [小结](#小结)
- [总结](#总结)

# kafka高可用
## 什么是高可用

「高可用性」，指系统无间断地执行其功能的能力，代表系统的可用性程度 Kafka从0.8版本开始提供了高可用机制，可保障一个或多个Broker宕机后，其他Broker能继续提供服务。

## Replica备份机制
- kafka的topic可以设置有n个副本（replica），副本数最好要小于等于broker的数量，也就是要保证一个broker上的replica最多有一个。
- 创建副本的单位是topic的分区，每个分区有1个leader和0到n-1follower，Kafka把多个replica分为Lerder replica和follower replica。

## ISR机制 
ISR副本： 就是能跟首领副本基本保持一致的跟随副本，如果同步的速度太慢的话，就会被踢出ISR副本。
副本同步：
- LEO（last end offset）：日志末端位移，记录了该副本对象底层日志文件中下一条消息的位移值，副本写入消息的时候，会自动更新 LEO 值。如果LE0 为2的时候，当前的offset为1。

- HW（high watermark）：高水印值，HW 一定不会大于 LEO 值，小于 HW 值的消息被认为是“已提交”或“已备份”的消息，并对消费者可见。

- producer向leader发送消息，之后写入到leader，leader在本地生成log，之后follow从leader拉取消息，follow写入到本地的log中，会给leader返回一个ack信号，一旦收到了ISR中的所有的ack信号，就会增加HW，然后leader返回给producer一个ack。

## ACK机制 
生产者发送消息中包含acks字段，该字段代表leader应答生产者前leader收到的应答数

- acks=0
生产者无需等待服务端的任何确认，消息被添加到生产者套接字缓冲区后就视为已发送，因此acks=0不能保证服务端已收到消息

- acks=1
只要 Partition leader 接收到消息而且写入本地磁盘了，就认为成功了，不管它其他的 Follower 有没有同步过去这条消息了

- acks=all
leader将等待ISR中的所有副本确认后再做出应答，因此只要ISR中任何一个副本还存活着，这条应答过的消息就不会丢失

acks=all是可用性最高的选择，但等待Follower应答引入了额外的响应时间。leader需要等待ISR中所有副本做出应答，此时响应时间取决于ISR中最慢的那台机器

如果说 Partition leader 刚接收到了消息，但是结果 Follower 没有收到消息，此时 leader 宕机了，那么客户端会感知到这个消息没发送成功，他会重试再次发送消息过去

Broker有个配置项min.insync.replicas(默认值为1)代表了正常写入生产者数据所需要的最少ISR个数

当ISR中的副本数量小于min.insync.replicas时，leader停止写入生产者生产的消息，并向生产者抛出NotEnoughReplicas异常，阻塞等待更多的Follower赶上并重新进入ISR

被leader应答的消息都至少有min.insync.replicas个副本，因此能够容忍min.insync.replicas-1个副本同时宕机

结论：
发送的acks=1和0消息会出现丢失情况，为不丢失消息可配置生产者acks=all & min.insync.replicas >= 2 

## 故障恢复机制
首先需要在集群所有Broker中选出一个Controller，负责各Partition的leader选举以及Replica的重新分配
- 当出现leader故障后，Controller会将leader/Follower的变动通知到需为此作出响应的Broker。

Kafka使用ZooKeeper存储Broker、Topic等状态数据，Kafka集群中的Controller和Broker会在ZooKeeper指定节点上注册Watcher(事件监听器)，以便在特定事件触发时，由ZooKeeper将事件通知到对应Broker 

### Broker
当Broker发生故障后，由Controller负责选举受影响Partition的新leader并通知到相关Broker」
- 当Broker出现故障与ZooKeeper断开连接后，该Broker在ZooKeeper对应的znode会自动被删除，ZooKeeper会触发Controller注册在该节点的Watcher；

- Controller从ZooKeeper的/brokers/ids节点上获取宕机Broker上的所有Partition；

- Controller再从ZooKeeper的/brokers/topics获取所有Partition当前的ISR；

- 对于宕机Broker是leader的Partition，Controller从ISR中选择幸存的Broker作为新leader；

- 最后Controller通过leaderAndIsrRequest请求向的Broker发送leaderAndISRRequest请求。

### Controller 

集群中的Controller也会出现故障，因此Kafka让所有Broker都在ZooKeeper的Controller节点上注册一个Watcher

Controller发生故障时对应的Controller临时节点会自动删除，此时注册在其上的Watcher会被触发，所有活着的Broker都会去竞选成为新的Controller(即创建新的Controller节点，由ZooKeeper保证只会有一个创建成功)

竞选成功者即为新的Controller

## Kafka的复制机制
kafka 每个分区都是由顺序追加的不可变的消息序列组成，每条消息都一个唯一的offset 来标记位置。
kafka中的副本机制是以分区粒度进行复制的，在kafka中创建 topic的时候，都可以设置一个复制因子(replica count)，这个复制因子决定着分区副本的个数，如果leader 挂掉了，kafka 会把分区主节点failover到其他副本节点，这样就能保证这个分区的消息是可用的。leader节点负责接收producer 发过来的消息，其他副本节点（follower）从主节点上拷贝消息。

kakfa 日志复制算法提供的保证是当一条消息在producer端认为已经committed的之后，如果leader 节点挂掉了，其他节点被选举成为了 leader 节点后，这条消息同样是可以被消费到的。

<a href="https://kafka.apache.org/documentation/#brokerconfigs_unclean.leader.election.enable" target="_blank">关键配置</a>

```
Indicates whether to enable replicas not in the ISR set to be elected as leader as a last resort, even though doing so may result in data loss

Type:	boolean
Default:	false
Valid Values:	
Importance:	high
Update Mode:	cluster-wide
```

默认为 false, 即允许不在isr中replica选为leader，这个配置可以全局配置，也可以在topic级别配置。
这样的话，leader选举的时候，只能从ISR集合中选举，集合中的每个点都必须是和leader消息同步的，也就是没有延迟，分区的leader 维护ISR 集合列表，如果某个点落后太多，就从 ISR集合中踢出去。
producer 发送一条消息到leader节点后， 只有当ISR中所有Replica都向leader发送ACK确认这条消息时，leader才commit，这时候producer才能认为这条消息commit了，正是因为如此，kafka客户端的写性能取决于ISR集合中的最慢的一个broker的接收消息的性能，如果一个点性能太差，就必须尽快的识别出来，然后从ISR集合中踢出去，以免造成性能问题。

### 如何判断副本不会被移除ISR集合？
1. follower副本最大落后leader副本的消息数:
   ```
   replica.lag.max.messages
   ```
2. 不仅指自从上次从副本获取请求以来经过的时间，而且还指自上次捕获副本以来的时间
   ```
   replica.lag.time.max.ms
   ```

## 小结
Replica的目的就是在发生意外时及时顶上，leader失效后，就需要从follower中马上选一个新的leader 。选举时优先从ISR中选定，因为这个列表中follower的数据是与leader同步的，从他们中间选取可以保证数据完整 。
但如果不幸ISR列表中的follower都不行了，就只能从其他follower中选取，这时就有数据丢失的可能了，因为不确定这个follower是否已经把leader的数据都复制完成了。
还有一种极端情况，就是所有副本都失效了，这时有两种方案：
- 等待ISR中的一个活过来，选为Leader，数据可靠，但活过来的时间不确定 。

- 选择第一个活过来的Replication，不一定是ISR中的，选为leader，以最快速度恢复可用性，但数据不一定完整。

Kafka支持通过配置选择使用哪一种方案，可以根据可用性和一致性进行权衡。

## 总结
1. kafka如何高可用？
大致分为五种机制，（Replica）备份机制、复制机制、ACK机制、ISR机制、故障恢复机制。
2. ISR 是一种什么同步机制？
不是完全同步，也不是完全异步，它基于Replica机制和ACK机制同步完成的。
3. 如果消息突然增多会不会移入移出ISR，影响应能？
考察移除ISR的两个配置参数，分别是最大落后消息数、上次捕获时间。

参考：
- <a href="https://www.bilibili.com/read/cv11024227" target="_blank">Kafka如何保证高可用</a>  
- <a href="https://juejin.cn/post/6939767567456157733" target="_blank">Kafka 高可用机制</a>

    