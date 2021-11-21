# HDFS 总结

## hdfs上跨节点如何数据迁移
hadoop distcp hdfs://nn1:8020/data hdfs://nn2:8020/
mr的执行方式只有map 默认并发20 -m 增加并发   
-bandwidth 可以控制单个map最大带宽
-overwrite 参数来覆盖已存在的文件。
-append参数将源HDFS文件的数据新增进去而不是覆盖它。