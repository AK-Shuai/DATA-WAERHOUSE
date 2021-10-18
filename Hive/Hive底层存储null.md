# Hive 底层存储null数据
null在hive的底层存储是\N 

1. 建表时直接指定
 底层数据中如何保存和标识NULL，是由 SERDEPROPERTIES('serialization.null.format' = '\N'); 参数控制的：

   ```
   CREATE TABLE `demo`(
     `province` string, 
     `city` string, 
     `num` int)
   ROW FORMAT DELIMITED 
     FIELDS TERMINATED BY ','
   TBLPROPERTIES (
     'serialization.null.format'='\"\"')
   ```
去指定 底层存储为"" 时 显示为null

2. 修改已存在的表
   ```
   alter table hive_tb set serdeproperties(‘serialization.null.format’ = ‘’);
   ```