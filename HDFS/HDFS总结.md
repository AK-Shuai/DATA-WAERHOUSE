# HDFS 总结

## hdfs上跨节点如何数据迁移
hadoop distcp hdfs://nn1:8020/data hdfs://nn2:8020/
mr的执行方式只有map 默认并发20 -m 增加并发   
-bandwidth 可以控制单个map最大带宽
-overwrite 参数来覆盖已存在的文件。
-append参数将源HDFS文件的数据新增进去而不是覆盖它。

## 自我知识链接
**写流程**
HDFS客户端 调用API DistributedFileSystem 使用RPC访问Name Node 让文件系统namespace 创建文件 Name Node返回一个对象 DFSOutPutStream 这个对象携带Data Node信息负责写入 一个一个包流入Data Node 返回ack信息 客户端给对象发close请求 调用API 通知Name Node完成了