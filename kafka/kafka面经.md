# Kafka 面试题集锦

gitchat: https://gitbook.cn/books/5f68ac2e688f8b61329857c7/index.html

1. 什么场景下会出现消息漏消费、消息重复消费以及如何处理消息漏消费、重复消费？

    消费者先处理数据，后提交 offset，提交 offset 失败时会导致下一次消费还是从上一次成功提交 offset 处开始消费，产生重复消费。

    消费者先提交 offset，后处理数据，数据处理时异常会导致下一次消费忽略已提交的数据，消息漏消费。

    对于重复消费，消费者业务侧可以采用一些幂等机制，比如消息中包含唯一的业务序号，业务处理时做去重判断。

    对于消息漏消费，建议消费者先处理数据，后提交 offset，这样能保证消息不会漏消费。

2. Kafka Broker 何时返回生产者 ack 响应？  
    数据可靠性-生产者 ack 机制。ack配置和保证数据最小复制数

3. Kafka 中的 ISR、OSR、AR、HW、LEO 分别代表什么含义？

    ISR（in-sync replica set） ：和 leader 保持一定程度同步的副本集
    OSR（Out-of-Sync Replicas）：和 leader 副本同步滞后过多的副本集
    AR（assigned replicas）：所有的副本集，即 AR=OSR+ISR
    HW（high watermark）：高水位线，标识了一个特定的消息偏移量，消费者只能拉取到这个 offset 之前的数据
    LEO（log end offset）：标识当前日志文件中下一条待写入消息的 offset

4. Kafka 消费者 offset 的提交方式有哪些？消费者自动提交的逻辑在哪里完成的？

    有手动和自动，自动提交是消费者在下一次 poll 之前。

5. Kafka 如何保证消息的顺序性的？

    Kafka 只能保证区内有序，因此可以设置 Topic 有且只有一个 partition
    在生产者发送消息时，指定 key 值，会根据 key 值计算出 partition，可以保证 key 值相同时分配到同一个 partition 上
    在消费者消费时先根据 key 或者业务字段，将消息存放到一个有序内存队列中，消费者线程从对应的内存队列中取出并操作
    为防止 log 文件过大导致数据定位效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 segment。每个 segment 都有对应的.index 文件和.log 文件以及 .timeindex 文件

6. Kafka 中的分区器、序列化器、拦截器之间的处理顺序是怎样的？

    拦截器 -> 序列化器 -> 分区器

7. 消费者组中的消费者数大于 Topic 的分区数时是如何消费的？

     超过 Topic 数的消费者线程（进程）将会没有消费，浪费资源。尽量保证 consumer 数=Topic 的 partition 数

8. Kafka 生产者客户端使用几个线程来处理消息，分别起什么作用？

    Kafka 生产者客户端由 2 个线程协调运行。生产者在 Main 线程中创建消息，消息经拦截器、序列化器、分区器作用后缓存到消息累加器（RecordAccumulator）中。Sender 线程从 RecordAccumulator 中获取消息并发送到 Kafka 服务器中。不妨看成是一种 Reactor 模式。

9. 死信补偿机制

    Kafka 本身不支持死信队列。Spring-kafka 内部封装了可重试消费消息的语义，也就是可以设置为当消费数据出现异常时，重试这个消息。而且可以设置重试达到多少次后，让消息进入预定好的 Topic，也就是死信队列里。参考这篇文章。

10. 为什么批量消费建议使用异步处理消息

    防止耗时的业务影响 Kafka poll 操作的性能。

11. Topic 分区可以在主题创建后更改么？如何更改，能否减少分区？

    可以增加分区，不可以减少，因为已存在的数据无法处理。可以通过执行脚本修改：

kafka-topics.sh   --bootstrap-server 192.168.174.129:9092 --alter --topic testtopic01 --partitions 6

12. offset 存储在哪？如果指定了 offset 消费，Kafka 是如何查找到对应的消息的？

    旧版 Kafka 存储在 ZooKeeper 中，新版存储在内置 Topic 即 consumer_offsets 中，这个主题给消费者存 offset 用。
    通过二分查找 index 文件中 -> log 文件。

13. Kafka Controller 的作用？

    在 Kafka 集群中会有一个或多个 Broker，其中有一个 Broker 会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。当某个分区的 leader 副本出现故障时，由控制器负责为该分区选举新的 leader 副本。当检测到某个分区的 ISR 集合发生变化时，由控制器负责通知所有 Broker 更新其元数据信息。当使用 kafka-topics.sh 脚本为某个 topic 增加分区数量时，还是由控制器负责分区的重新分配。

14. Kafka 有哪些核心的设计保证了其高性能的？

    分布式架构、顺序写磁盘、零拷贝等。

15. Kafka 宕机了怎么办？

    多机房、多机器分布式部署、使用 Topic 的副本机制保证可靠性。

16. Kafka 数据积压的原因有哪些以及如何处理？

    Kafka 消费能力不足导致的积压，可以考虑增加 Topic 的分区数，并且增加消费者组的消费者数量。保证消费者数（消费者线程数）=分区数。consumer 实例数（线程数）大于分区数时，多余的消费实例将消费不到分区，浪费系统资源。consumer 实例数小于分区数时，一个消费者会消费多个分区。

    如果是下游消息处理不及时导致的积压，可以提高每个批次拉取消息的数量。如果每个批次拉取的数据量远小于生产的数据量，也会造成数据积压。单次调用 poll() 方法返回的记录数可以通过参数 max.poll.records 配置，默认是 500 条。
17. Kafka 事务
    Kafka 从 0.11 版本开始引入了事务支持，解决了幂等性没有解决的跨分区问题。事务在 Kafka 在 Exactly Once 语义的基础上，保证对多个分区的写入操作的原子性。即多个操作要么全部成功，要么全部失败。

    为了实现跨分区会话的事务，应用程序需要提供一个全局唯一的 TransactionID，并将 Producer 获得的 PID 和 TransactionID 绑定。当 Producer 重启后就可以通过 TransactionID 获得原来的 PID。要开启事务必须先开启幂等性，即 enable.idempotence 设置为 true。Kafka 为管理事务引入了新的组件 Transaction Coordinator（事务协调器）。Producer 从事务协调器中获取 Transaction ID 对应的事务状态。事务协调器负责将事务写入 Kafka 内部的 topic，即使服务重启，由于事务状态已经保存，进行中的事务可以恢复进行。

    上述事务机制主要是针对 Producer 而言。对于消费者，事务能保证的语义相对偏弱，无法保证已提交的事务中的所有消息都能够被消费。主要原因是 Consumer 访问任意 offset 的消息，并且不同的 Segment File 的生命周期不同，同一事务的消息可能出现重启后被删除的情形。