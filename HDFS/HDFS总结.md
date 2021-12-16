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

失败情况：
1. 管道关闭
2. 正常节点更换新标识
3. 其他块节点写入
4. 备份数不足，Name Node 找到新节点继续写入

**读流程**：
HDFS客户端 调用API DistributedFileSystem 使用RPC电泳Name Node 文件系统让文件系统namespace获取元数据信息 返回 DFSInputStream 这个对象带着元数据存储位置 负责访问校验 客户端调用close关闭

读写最大区别是 访问 Name Node 文件系统是creat还是get 

失败情况：
1. 在 DFSstream 连接失败，记录下来
2. 读到数据会进行check，发现块损坏，去备份节点读取，同步工作交给 Name Node


**HDFS高可用**：
1. Zookeeper的watch监听机制主备HA 
2. JournalNode记录日志保持主备元数据统一 
3. JournalNode也是分布式也有选举机制
4. 多副本机制
5. 数据校验机制 checknums 单位512字节

**短路读机制**：  
HDFS 采用 Unix 的机制，DFS Client 发起读请求后 Data Node 返回的是文件描述等信息，DFS Client 直接本机读取。

元数据文件大小：150byte。

**机架感知**：  
首选 client 机器，次选随机选择。

不同机架，不同机房。

随机选择规则：  
1. 给定存储类型
2. 不能知识读的
3. 不能是坏的
4. 必须在线
5. 最近更新过心跳
6. 存储够的
7. I/O不忙的
8. 同机架最大副本数限制

