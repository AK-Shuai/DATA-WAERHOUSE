# HDFS 迁移

## 使用场景：

1. 冷热集群数据分类存储
2. 集群数据整体搬迁
3. 数据的准实时同步，目的在于数据的双备份可用

## 数据迁移要素考量：

1. 带宽Bandwidth：需要限流

2. 性能Performance：采用单机程序还是分布式程序？

3. 增量同步Data-Increment:原始数据文件进行了追加写、原始数据文件被删除或重命名  
   在海量数据存储系统如HDFS中，一般不会在源文件内容上做修改，要么继续追加写，要么删除文件。所以 做增量数据同步，只要考虑上述两个条件即可  
   判断追加写：  
   - 先比较文件大小，如果两个阶段文件大小发生改变，说明文件在内容上已经发生变更，变更的类型有两类。截取对应原始长度部分进行checksum(校验和)比较，如果一致，则此文件发生了追加写。不一致，则说明文件在原内容上也已经改变
   - 如果文件大小一致，则计算相应的checksum，然后进行比较

4. 数据迁移的同步性
   - 数据迁移解决方案：DistCp

   - DistCp支持带宽限流，可以通过参数bandwidth来控制

   - 增量同步数据，通过update、append、diff这3个参数来控制，Update：更新目标路径，只拷贝相对于源端，目标端不存在的文件或目录，Append：追加写目标路径下已经存在的文件，如果这个文件在源端已经发生了追加写操作，Diff：通过快照的diff对比信息来同步源路径与目标路径，高效的性能：执行的分布式特性（纯map任务构成的job）、高效的MR组件

Hadoop DistCp命令：

```s
distcp OPTIONS [-source_path···] <target_path>
```

OPTIONS
-append //拷贝文件时支持对现有文件进行追加写操作
-async //异步执行distcp拷贝任务
-bandwidth //对每个map任务的带宽限速
-delete //删除相对于源端，目标端多出来的文件
-diff //通过快照diff信息进行数据的同步
-overrite //以覆盖的方式进行拷贝，如果目标端文件已经存在，则直接进行覆盖
-p //拷贝数据时，扩展属性信息的保留，包括权限信息、块大小信息等等
-skipcrccheck //拷贝数据时是否跳过校验和的校验
-update //拷贝数据时，只拷贝相对于源端，目标端不存在的文件数据

其中source_path、target_path需要带上地址前缀以区分不同的集群：
```s
hadoop distcp hdfs://nn1:8020/foo/a hdfs://nn2:8020/bar/foo
```

## DataNode节点迁移

1. 目标：
   将原DataNode所在节点的机器从A机房换到B机房，其中会涉及主机名和ip地址的改变，需要保证数据不发生丢失

2. 相关知识：
   机器迁移将使该节点停止心跳，如果超过心跳检查时间，将被认为是死节点，从而发生大量块复制现象。为了使得短时间内不成为死节点，需要人工把心跳超时检查时间设大。

   ```s
   <name>dfs.namenode.heartbeat.recheck-interval</name>
   <value>10800000</value>  #3小时
   ```
   
   执行以下操作使得配置生效：
   - 更新standby namenode的hdfs-site.xml的配置，并重启
   - 等待standby namenode退出safemode之后，再stop active namenode，更新配置并重启
   此种方案只适用于DataNode不涉及主机名和IP地址变化的情况

3. DataNode更换主机名、IP地址时的迁移方案
   - 停止集群hdfs相关的服务，最好把YARN相关的服务也停止
   - 修改HDFS集群名称相关配置

core-site.xml:
```s
<name>fs.defaultFS</name>
    <value>hdfs://clusterA</value>
    <final>true</final>
```

yarn-site.xml:
```s
<name>yarn.resourcemanager.fs.state-store.uri</name>
    <value>hdfs://clusterA/logs/yarn/rmstore</value>
```
hdfs-site.xml:
```s
<name>dfs.nameservices</name>
    <value>clusterA</value>

<name>dfs.ha.namenodes.clusterA</name>
    <value>nn1,nn2</value>

<name>dfs.namenode.rpc-address.clusterA.nn1</name>
    <value>clusternn1:9000</value>

<name>dfs.namenode.rpc-address.clusterA.nn2</name>
    <value>clusternn2:9000</value>

<name>dfs.namenode.http-address.clusterA.nn1</name>
    <value>clusternn1:50070</value>

<name>dfs.namenode.http-address.clusterA.nn2</name>
    <value>clusternn2:50070</value>

<name>dfs.client.failover.proxy.provider.clusterA</name>
     <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
```
- 重新格式化HDFS所依赖的znode

```s
hdfs zkfc -formatZK
```

-  重启HDFS相关服务，执行hadoop的fs命令是否可用，最后启动YARN 服务，并提交一个wordcount任务到YARN上做测试

## 参考
1. <a href="https://blog.csdn.net/Regan_Hoo/article/details/78610799" target="_blank">HDFS_数据迁移&节点迁移</a>