# Spark 部署模式
Spark 支持 3 种集群管理器（Cluster Manager），分别为：
1. Standalone：独立模式，Spark 原生的简单集群管理器，自带完整的服务，可单独部署到一个集群中，无需依赖任何其他资源管理系统，使用 Standalone 可以很方便地搭建一个集群；
2. Apache Mesos：一个强大的分布式资源管理框架，它允许多种不同的框架部署在其上，包括 yarn；

3. Hadoop YARN：统一的资源管理机制，在上面可以运行多套计算框架，如 map reduce、storm 等，根据 driver 在集群中的位置不同，分为 yarn client 和 yarn cluster。  

实际上，除了上述这些通用的集群管理器外，Spark 内部也提供了一些方便用户测试和学习的简单集群部署模式。由于在实际工厂环境下使用的绝大多数的集群管理器是 Hadoop YARN，因此我们关注的重点是 Hadoop YARN 模式下的 Spark 集群部署。

Spark 的运行模式取决于传递给 SparkContext 的 MASTER 环境变量的值，个别模式还需要辅助的程序接口来配合使用，目前支持的 Master 字符串及 URL 包括：  
Spark 运行模式配置  
Master URL|Meaning
--|:---:
local | 在本地运行，只有一个工作进程，无并行计算能力。
local[K] | 在本地运行，有 K 个工作进程，通常设置 K 为机器的 CPU 核心数量。
local[*] | 在本地运行，工作进程数量等于机器的 CPU 核心数量。
spark://HOST:PORT | 以 Standalone 模式运行，这是 Spark 自身提供的集群运行模式，默认端口号: 7077。详细文档见:Spark standalone cluster。
mesos://HOST:PORT | 在 Mesos 集群上运行，Driver 进程和 Worker 进程运行在 Mesos 集群上，部署模式必须使用固定值:--deploy-mode cluster。详细文档见:MesosClusterDispatcher.
yarn-client | 在 Yarn 集群上运行，Driver 进程在本地，Executor 进程在 Yarn 集群上，部署模式必须使用固定值:--deploy-mode client。Yarn 集群地址必须在 HADOOPCONFDIR or YARNCONFDIR 变量里定义。
yarn-cluster | 在 Yarn 集群上运行，Driver 进程在 Yarn 集群上，Work 进程也在 Yarn 集群上，部署模式必须使用固定值:--deploy-mode cluster。Yarn 集群地址必须在 HADOOPCONFDIR or YARNCONFDIR 变量里定义。

用户在提交任务给 Spark 处理时，以下两个参数共同决定了 Spark 的运行方式。
- --master MASTER_URL ：决定了 Spark 任务提交给哪种集群处理。

- --eploy-mode DEPLOY_MODE：决定了 Driver 的运行方式，可选值为 Client 或者 Cluster。

## Standalone 模式运行机制

Standalone 集群有四个重要组成部分，分别是：

1) Driver：是一个进程，我们编写的 Spark 应用程序就运行在 Driver 上，由 Driver 进程执行；

2) Master(RM)：是一个进程，主要负责资源的调度和分配，并进行集群的监控等职责；

3) Worker(NM)：是一个进程，一个 Worker 运行在集群中的一台服务器上，主要负责两个职责，一个是用自己的内存存储 RDD 的某个或某些 partition；另一个是启动其他进程和线程（Executor），对 RDD 上的 partition 进行并行的处理和计算。

4) Executor：是一个进程，一个 Worker 上可以运行多个 Executor，Executor 通过启动多个线程（task）来执行对 RDD 的 partition 进行并行计算，也就是执行我们对 RDD 定义的例如 map、flatMap、reduce 等算子操作。

### Standalone Client 模式
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark_Standalone_Client%E6%A8%A1%E5%BC%8F.jpg" width="400"></div>

在 Standalone Client 模式下，Driver 在任务提交的本地机器上运行，Driver 启动后向 Master 注册应用程序，Master 根据 submit 脚本的资源需求找到内部资源至少可以启动一个 Executor 的所有 Worker ，然后在这些 Worker 之间分配 Executor，Worker 上的 Executor 启动后会向 Driver 反向注册，所有的 Executor 注册完成后，Driver 开始执行 main 函数，之后执行到 Action 算子时，开始划分 stage ，每个 stage 生成对应的 taskSet，之后将 task 分发到各个 Executor 上执行。

### Standalone Cluster 模式
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark_Standalone_Cluster%E6%A8%A1%E5%BC%8F.jpg" width="400"></div>

在 Standalone Cluster 模式下，任务提交后，Master 会找到一个 Worker 启动 Driver 进程， Driver 启动后向 Master 注册应用程序，Master 根据 submit 脚本的资源需求找到内部资源至少可以启动一个 Executor 的所有 Worker，然后在这些 Worker 之间分配 Executor，Worker 上的 Executor 启动后会向 Driver 反向注册，所有的 Executor 注册完成后，Driver 开始执行 main 函数，之后执行到 Action 算子时，开始划分 stage，每个 stage 生成对应的 taskSet，之后将 task 分发到各个 Executor 上执行。  

**注意** Standalone 的两种模式下（client/Cluster），Master 在接到 Driver 注册 Spark 应用程序的请求后，会获取其所管理的剩余资源能够启动一个 Executor 的所有 Worker，然后在这些 Worker 之间分发 Executor，此时的分发只考虑 Worker 上的资源是否足够使用，直到当前应用程序所需的所有 Executor 都分配完毕，Executor 反向注册完毕后，Driver 开始执行 main 程序。

## YARN 模式运行机制
### YARN Client 模式
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark_YARN_Client%E6%A8%A1%E5%BC%8F%20.jpg" width="400"></div>

在 YARN Client 模式下，Driver 在任务提交的本地机器上运行，Driver 启动后会和 ResourceManager 通讯申请启动 ApplicationMaster，随后 ResourceManager 分配 container ，在合适的 NodeManager 上启动 ApplicationMaster ，此时的 ApplicationMaster 的功能相当于一个 ExecutorLaucher ，只负责向 ResourceManager 申请 Executor 内存。

ResourceManager 接到 ApplicationMaster 的资源申请后会分配 container ，然后 ApplicationMaster 在资源分配指定的 NodeManager 上启动 Executor 进程，Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数，之后执行到 Action 算子时，触发一个 job，并根据宽依赖开始划分 stage，每个 stage 生成对应的 taskSet，之后将 task 分发到各个 Executor 上执行。

### YARN Cluster 模式
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark_YARN_Cluster%E6%A8%A1%E5%BC%8F.jpg" width="400"></div>

在 YARN Cluster 模式下，任务提交后会和 ResourceManager 通讯申请启动 ApplicationMaster，随后 ResourceManager 分配 container，在合适的 NodeManager 上启动 ApplicationMaster，此时的 ApplicationMaster 就是 Driver。

Driver 启动后向 ResourceManager 申请 Executor 内存，ResourceManager 接到 ApplicationMaster 的资源申请后会分配 container，然后在合适的 NodeManager 上启动 Executor 进程，Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数，之后执行到 Action 算子时，触发一个 job，并根据宽依赖开始划分 stage，每个 stage 生成对应的 taskSet，之后将 task 分发到各个 Executor 上执行。

