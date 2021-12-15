- [参考](#参考)


# Kafka 概述

Kafka 最初是由 LinkedIn 即领英公司基于 Scala 和 Java 语言开发的分布式消息发布—订阅系统，现已捐献给 Apache 软件基金会。Kafka 最被广为人知的是作为一个消息队列（MQ）系统存在，而事实上 Kafka 已然成为一个流行的分布式流处理平台。其具有高吞吐、低延迟的特性，许多大数据处理系统比如 Storm、Spark、Flink 等都能很好地与之集成。按照 Wikipedia 上的说法，Kafka 的核心数据结构本质上是一个“按照分布式事务日志架构的大规模发布/订阅消息队列”。总的来讲，Kafka 通常具有 3 重角色：

- 消息系统：Kafka 和传统的消息队列比如 RabbitMQ、RocketMQ、ActiveMQ 类似，支持流量削锋、服务解耦、异步通信等核心功能。
- 流处理平台：Kafka 不仅能够与大多数流式计算框架完美整合，并且自身也提供了一个完整的流式-处理库，即 Kafka Streaming。Kafka Streaming 提供了类似 Flink 中的窗口、聚合、变换、连接等功能。
- 存储系统：通常消息队列会把消息持久化到磁盘，防止消息丢失，保证消息可靠性。Kafka 的消息持久化机制和多副本机制使其能够作为通用数据存储系统来使用。

一句话概括：Kafka 是一个分布式的基于发布/订阅模式的消息队列（Message Queue），在业界主要应用于大数据实时处理领域。


## Kafka 体系结构

如图所示，Kafka 的体系结构中通常包含多个 Producer（生产者）、多个 Consumer（消费者）、多个 Broker（Kafka 服务器）以及一个 ZooKeeper 集群。

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Kafka-ZooKeeper.png" width="400"></div>

体系结构中几个角色：

- Producer：消息发送方，即生产者，负责生产消息，并将其发送到 Kafka 服务器（Broker）中。
- Consumer：消息接收方，即消费者，负责消费消息。消费者客户端主动从 Kafka 服务器上拉取（pull）到消息，应用程序进行业务处理。
- Broker：Kafka 服务实例，即 Kafka 服务器，让生产者客户端、消费者客户端来连接，可以看做消息的中转站。多个 Broker 将组成一个 Kafka 集群。
- ZooKeeper：ZooKeeper 是在 Kafka 集群中负责管理集群元数据、控制器选举等操作的分布式协调器。

## 分区和主题

Topic（主题）和 Partition（分区）是 Kafka 中的两个核心概念。在 Kafka 中，消息以 Topic 为单位进行归类。生产者必须将消息发送到指定的 Topic，即发送到 Kafka 集群的每一条消息都必须指定一个主题；消费者消费消息也要指定主题，即消费者负责订阅主题并进行消费。

试想如果一个 Topic 在 Kafka 中只对应一个存储文件，那么海量数据场景下这个文件所在机器的 I/O 将会成为这个主题的性能瓶颈，而分区解决了这个问题。在 Kafka 中一个 Topic 可以分为多个分区（partition），每个分区通常以分布式的方式存储在不同的机器上。一个特定的 partition 只属于一个 Topic。Kafka 通常用来处理超大规模数据，因此创建主题时可以立即指定多个分区以提高处理性能。当然也可以创建完成后再修改分区数。同一个主题的不同分区包含的消息是不同的。底层存储上，每一个分区对应一个可追加写的 Log 文件，消息在被追加到分区 Log 文件时会分配一个特定的 offset（偏移量），offset 是消息在分区中的唯一标识，Kafka 通过 offset 来保证消息在分区内的有序性。offset 并不跨越分区，即 Kafka 保证的是分区有序而不是全局有序。

下图展示了消息的追加写入:

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/kafka%E8%BF%BD%E5%8A%A0%E5%86%99%E5%85%A5.png" width="400"></div>

Kafka 中的分区可以分布在不同的服务器（Broker）上，因此主题可以通过分区的方式跨越多个 Broker，相比单个 Broker、单个分区而言并行度增加了，性能提升不少。

在分区之下，Kafka 又引入了副本（Replica）的概念。如果说增加分区数实现了水平扩展，增加副本数则是实现了纵向扩展，并提升了容灾能力。同一分区的不同副本中保存的消息是相同的。需要注意的是，在同一时刻，副本之间并非完全一样，因为同步存在延迟。副本之间是一主多从的关系。leader 副本负责处理读写请求，follower 副本只负责与 leader 副本进行消息同步。副本存在不同的 Broker 中，当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供读写服务。Kafka 通过多副本机制实现了故障的自动转移，当 Kafka 集群中某个 Broker 挂掉时，副本机制保证该节点上的 partition 数据不丢失，仍然能保证 Kafka 服务可用。

下图展示了一个多副本的架构。本例中 Kafka 集群中有 4 台 Broker，主题分区数为 3，且副本因子也为 3。生产者和消费者客户端只和 leader 副本进行交互，follower 副本只负责和 leader 进行消息同步。每个分区都存在不同的 Broker 中，如果每个 Broker 单独部署一台机器的话，那么不同的 Partition 及其副本在物理上便是隔离的。
<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/kafka%E5%88%86%E5%8C%BA%E9%9A%94%E7%A6%BB.png" width="400"></div>

可以认为 Topic 是逻辑上的概念，partition 是物理上的概念，因为每个 partition 都对应于一个.log 文件存储在 Kafka 的 log 目录下，该 log 文件中存储的就是 producer 生产的数据。Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset（偏移位）。消费者组中的每个消费者，每次消费完数据都会向 Kafka 服务器提交 offset，以便出错恢复时从上次的位置继续消费。

消费组（Consumer Group）是 Kafka 的消费理念中一种特有的概念，每个消费者都属于一个消费组。生产者的消息发布到主题后，只会被投递给订阅该主题的每个消费组中的一个消费者。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；所有的消费者都有一个与之对应的消费者组，即消费者组是逻辑上的一个订阅者。消费者组之间互不影响，多个不同的消费者组可以同时订阅一个 Topic，此时消息会同时被每个消费者组中一个消费者消费。

理解上述概念有助于在实际应用中规划 Topic 分区数、消费者数、生产者数。实际生产中，一般分区数和消费者数保持相等，如果这个主题的消费者数大于主题的分区数，那么多出来的消费者将消费不到数据，只能浪费系统资源。


## 参考
1. <a href="https://gitbook.cn/books/5f68ac2e688f8b61329857c7/index.html" target="_blank">Kafka实战教程</a>

