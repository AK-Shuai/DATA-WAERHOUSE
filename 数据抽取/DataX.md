# DAtaX 工作原理
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/DataX%E5%8E%9F%E7%90%86%E5%9B%BE.png" width="400"></div>

## DataX 概念
- Job：job是DataX 用以描述从一个源头到一个目的端的同步作业，是 DataX 数据同步的最小业务单元。比如：从一张 MySQL 的表同步到 Hive 的一个表的特定分区。 
- Task：Task 是为最大化而把 Job 拆分得到的最小执行单元。比如：读一张有 1024 个分表的 MySQL 分库分表的 Job，拆分成 1024 个读 Task，若干个任务并发执行。或者将一个大表按照 id 拆分成 1024 个分片，若干个分片任务并发执行。 
- TaskGroup：描述的是一组 Task 集合。在同一个 TaskGroupContainer 执行下的 Task 集合称之为 TaskGroup。
- JobContainer：Job 执行器，负责 Job 全局拆分、调度、前置语句和后置语句等工作的工作单元。
TaskGroupContainer：TaskGroup 执行器，负责执行一组 Task 的工作单元  
Job 和 Task 是 DataX 两种维度的抽象。
### DataX 的处理过程可描述为：
1. DataX 完成单个数据同步的作业，我们称之为 Job，DataX 接受到一个 Job 之后，将启动一个进程来完成整个作业同步过程。DataX Job 模块是单个作业的中枢管理节点，承担了数据清理、子任务切分（将单一作业计算转化为多个子 Task）、TaskGroup 管理等功能。
2. DataXJob 启动后，会根据不同的源端切分策略，将 Job 切分成多个小的 Task（子任务），以便于并发执行。Task 便是 DataX 作业的最小单元，每一个 Task 都会负责一部分数据的同步工作。
3. 切分多个 Task 之后，DataX Job 会调用 Scheduler 模块，根据配置的并发数据量，将拆分成的 Task 重新组合，组装成 TaskGroup (任务组)。每一个 TaskGroup 负责以一定的并发运行完毕分配好的所有 Task，默认单个任务组的并发数量为 5。
4. 每一个 Task 都由 TaskGroup 负责启动，Task 启动后，会固定启动 Reader—>Channel—>Writer 的线程来完成任务同步工作。
5. DataX 作业运行起来之后， Job 监控并等待多个 TaskGroup 模块任务完成，等待所有 TaskGroup 任务完成后 Job 成功退出。否则，异常退出，进程退出值非 0。

其中 Channel 是连接 Reader 和 Writer 的数据交换通道，所有的数据都会经由 Channel 进行传输，一个 Channel 代表一个并发传输通道，通过该通道实现传输速率控制。接下来我们通过源码的角度，在抽取其核心逻辑，以 MySQL 到 HDFS 的传输为例分析其工作流程。通过分析源码将会有以下几点收获：
- DataX 工作流程
- DataX 插件机制
- DataX 同步实现

## DataX 源码分析
### 部署调试环境
- 下载源代码：git clone git@github.com:alibaba/DataX.git
- 打包编译：mvn -U clean package assembly:assembly -Dmaven.test.skip=true
- 获取执行命令：反注释 datax.py 代码行 print startCommand
- 启动 DataX：python2 bin/datax.py ~/DataX/your_job.json
### 输出启动命令为：
    java -server -Xms1g -Xmx1g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=~/workplace/datax/log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=~/workplace/datax/log -Dloglevel=info -Dfile.encoding=UTF-8 -Dlogback.statusListenerClass=ch.qos.logback.core.status.NopStatusListener -Djava.security.egd=file:///dev/urandom -Ddatax.home=~/workplace/datax -Dlogback.configurationFile=~/workplace/datax/conf/logback.xml -classpath ~/workplace/datax/lib/*:.  -Dlog.file.name=x_DataX_job_job_json com.alibaba.datax.core.Engine -mode standalone -jobid -1 -job ~/workplace/datax/DataX/job/job.json

### 其中以下这些为必需参数。
#### vm 参数
- Ddatax.home=~/workplace/datax
- classpath ~/workplace/datax/lib/*

#### application 参数
- mode standalone
- jobid -1
- job ~/workplace/datax/DataX/job/job.json

#### main class
- com.alibaba.datax.core.Engine

在 IDEA 中配置如下： 
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/DataX_java%E9%85%8D%E7%BD%AE%E5%9B%BE.png" width="400"></div>

## Datax工作流程
https://gitee.com/Liuashuai/data-warehouse/blob/master/%E6%95%B0%E6%8D%AE%E6%8A%BD%E5%8F%96/DataX%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md

## DataX 性能优化
#### 不论是 Job 还是 Task，Configuration 都贯穿其中，提供任务执行时的配置信息，Configuration 的信息来自 3 个部分：DataX 启动时提供的 JSON 配置信息、DataX 系统 conf 路径的 core.json 和 plugin 目录下的 plugin.json。以我们 MySQL 同步 Hive 的配置为例说明 DataX 的配置参数：  
    {
    "core": {
    "transport": {
      "channel": {
        "speed": {
          "record": 200000
        }
      }
    }
    },
    "job": {
      "content": [
      {
        "reader": {
          "name": "mysqlreader",
          "parameter": {
            "splitPk": "id",
            "column": [
              "`id`",
              `event`",
              "`create_time`"
              ],
            "connection": [
              {
                "jdbcUrl": [
                  "jdbc:${dbType}://${jdbcHost}:${jdbcPort}/${dbName}?socketTimeout=3600000"
                ],
                "table": [
                  "${tbName}"
                ]
              }
            ],
            "username": "${userName}",
            "password": "*******",
            "where": null
          }
        "writer": {
          "name": "hdfswriter",
          "parameter": {
            "defaultFS": "hdfs://ec2-52-11-111-222.us-west-2.compute.amazonaws.com:8020",
            "path": "/user/hadoop/idoo/stock_picking_log/2019-10-21/",
            "fileType": "orc",
            "fileName": "${tbName}",
            "writeMode": "nonConflict",
            "fieldDelimiter": "\u0001",
            "dateFormat": "yyyy-MM-dd hh:mm:ss",
            "column": [
              {
                "name": "id",
                "type": "INT"
              },
              {
                "name": "event",
                "type": "STRING"
              },
              {
                "name": "create_time",
                "type": "TIMESTAMP"
              }
            ],
          }
        }
      }
    "setting": {
      "speed": {
        "channel": 6
      }
    }
  }
}

### 通过 DataX 原理和实现的理解，自然可以知道如何提升 DataX 的同步效率。当然最直接的方式就是提高 MySQL 和 HDFS 的硬件性能如 CPU、内存、IOPS、网络带宽等。当基础资源受限的情况下，从我们的实践中可以总结如下几条经验：

1. 将不同的集群划分到同一个网络或者区域内，减少跨网络的不稳定性，如将阿里云集群迁移到 Amazon 集群，或者同一个 Amazon 集群中不同区域划分到同一个子网络内，这个可以部分减少数据延迟。
2. 对数据库按照主键划分，指定 splitPk 参数，DataX 对单个表默认一个通道，如果指定拆分主键，将会大大提升同步并发数和吞吐量。
3. 在 CPU、内存以及 MySQL 负载满足的情况下，尽可能增大 Channel 并发数，但不要超过 MySQL 的并发支撑能力。
4. 通道并发数意味着系统需要更多的内存，通过源码我们看到 DataX 对读取的每条记录，通过反射生成记录实例，这将导致大量的新生代内存开销和 GC 垃圾回收动作，因此 JVM 参数中除了加大总的可用堆内存，还应分配足够大的新生代内存占比。如果使用 Java 8 及以上，还应该通过 - XX:-UseCompressedClassPointers 禁用压缩指针，否则因为存储资源不够用会显著拖慢数据同步。
5. 当无法提升通道数量时，而且每个拆分依然很大的时候，可以考虑对每个拆分再次拆分。也就是如果对一个 Job 配置 querySql，那么 DataX 可以在条件查询的基础上再次根据 Channel 和 splitPk 拆分。要注意到配置中 job.conent 是一个 list，可以配置多个 Job。 5. 设定合适的参数，如 MySQL 超时参数等，比如在 JDBC 连接字符串中配置 socketTimeout。
