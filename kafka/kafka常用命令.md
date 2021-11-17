- [创建topic](#创建topic)
- [查看kafka集群所有的topic](#查看kafka集群所有的topic)
- [查看某一个topic的信息](#查看某一个topic的信息)
- [修改topic](#修改topic)
- [模拟生产者](#模拟生产者)
- [模拟消费者](#模拟消费者)
- [查看kafka数据](#模拟生查看kafka数据产者)

# Kafka的常用操作指令

## 创建topic
```
kafka-topic.sh --create --topic topic名称 --partitions 分区数 --replication-factor 副本数 --bootstrap-server borker节点:端口<9092>
说明：
--create 表示是创建topic
--bootstrap-server 根据输入的broker来找到kafka集群
注意: 副本数不能不要大于kafka集群的borker个数（多余的副本没什么用）
--partitions  --replication-factor 这两个参数在创建topic的时候可以不用写，默认值都是
样例：
kafka-topics.sh --create --topic first --bootstrap-server wxler1:9092,wxler2:9092 --partitions 3 --replication-factor 3
```

## 查看kafka集群所有的topic
```
kafka-topic.sh --list --bootstrap-server borker节点:端口<9092>
```

## 查看某一个topic的信息
```
kafka-topic.sh --describe --topic topic名称 --bootstrap-server borker节点:端口<9092>
说明：
可以查看的信息包含topic的分区数、副本数、leader在哪个broker上。follwer在哪些broker节点上。ISR列表
样例：
[wxler@wxler1 ~]$ kafka-topics.sh --describe --bootstrap-server wxler1:9092,wxler2:9092 --topic first
Topic: first	PartitionCount: 3	ReplicationFactor: 3	Configs: segment.bytes=1073741824
	Topic: first	Partition: 0	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
	Topic: first	Partition: 1	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
	Topic: first	Partition: 2	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
```

## 修改topic
只能够修改topic的分区，而且是只能增加分区，不能够减少分区
```
kafka-topic.sh --alter --topic topic名称 --partitions 分区数 --bootstrap-server borker节点:端口<9092>
```

## 模拟生产者
```
kafka-console-producer.sh --topic topic名称 --broker-list borker节点:端口<9092>
```

## 模拟消费者
```
kafka-console-consumer.sh --topic topic名称 --bootstrap-server borker节点:端口<9092>
```

## 查看kafka数据

```
kafka-dump-log.sh --files kafka数据文件 --print-data-log
```