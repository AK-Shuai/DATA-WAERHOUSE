# HDFS 总结

## hdfs上跨节点如何数据迁移
hadoop distcp hdfs://nn1:8020/data hdfs://nn2:8020/
mr的执行方式只有map 默认并发20 -m 增加并发   
-bandwidth 可以控制单个map最大带宽
-overwrite 参数来覆盖已存在的文件。
-append参数将源HDFS文件的数据新增进去而不是覆盖它。

## 自我知识链接
**写流程**：
HDFS客户端 调用API DistributedFileSystem 使用RPC访问Name Node 让文件系统namespace 创建文件 Name Node返回一个对象 DFSOutputStream 这个对象携带Data Node信息负责写入 一个一个包流入Data Node 返回ack信息 客户端给对象发close请求 调用API 通知Name Node完成了

**读流程**：
HDFS客户端 调用API DistributedFileSystem 使用RPC电泳Name Node 文件系统让文件系统namespace获取元数据信息 返回 DFSInputStream 这个对象带着元数据存储位置 负责访问校验 客户端调用close关闭

读写最大区别是 访问Name Node文件系统是creat还是get 

**HDFS高可用**：
Zookeeper的watch监听机制主备HA 
JournalNode记录日志保持主备元数据统一 
JournalNode也是分布式也有选举机制
多副本机制
数据校验机制