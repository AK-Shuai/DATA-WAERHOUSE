### 工人介绍
ResourceManager也就是大boss 下有俩管理者 Scheduler 和 ApplicationManager

ApplicationManager 负责客户端通信和启动小leader并监控它

NodeManager(管理所有工人) 是 ResourceManager 的小弟，是一个执行leader负责分配工人，上报工人资源信息、健康状态，只负责管理工人不负责干啥

Container(工人单位) 是Yarn的最小计算单元，CPU 和内存资源

ApplicationMaster 是一个项目的小leader，每次执行需要分配一个领导的去调度，负责它们状态及监控各个任务的执行，和重启失败的任务。

### 整个流程
1. 客户端发任务给ResourceManager
2. ResourceManager 向 NodeManager 申请一个容器当小leader也就是 ApplicationMaster 
3. ApplicationMaster 被 NodeManager 启动后向 ResourceManager 进行注册，并上报自己的状态
4. ApplicationMaster 会向 ResourceManager 请求资源执
行容器运行程序
5. ResourceManager 拥有 NodeManager 实时上报的信息，所以将可用的容器分配给它
6. ApplicationMaster 请求 NodeManager 启动容器运行程序
7. 容器启动后 NodeManager 会将运行速度状态信息上报给小leader
8. 运行程序期间客户端可以通过已小leader通信 获取运行状态、进度信息
9. 运行完毕后这个小leader工作完事了，向 ResourceManager 注销关闭自己，向 NodeManager 归还容器组

### Yarn成为通用化调度的关键
ApplicationMaster 是可以开发者自己实现的，这也让 Yarn 成为了一个通用化的资源调度平台，Spark/MapReduce/Flink

### Scheduler 三种调度方式
1. Fifo first in first out 先进先出 会把所有资源占的死死的
2. Capacity 把所有资源分成队列，每个队列有独立资源 队列内部应用Fifo 如果队列很多每个获取1/n 太平均了不合理。
3. Fair 首先他有这队列的概念，但是可以设置 公平共享量和最小共享量，公平共享靠权重获取资源  最小共享保证每个程序都可以分配到资源

### 调度器为空闲节点选择 Container 的策略？
队列是树形结构的，从根队列开始遍历子队列，看看谁用的资源少，用的少反向查询一下程序id，优先分配这个程序，优先这个程序最高的 Container