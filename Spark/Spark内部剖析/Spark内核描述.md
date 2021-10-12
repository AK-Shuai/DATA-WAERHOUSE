# Spark 内核描述
Spark 内核泛指 Spark 的核心运行机制，包括 Spark 核心组件的运行机制、Spark 任务调度机制、Spark 内存管理机制、Spark 核心功能的运行原理等
## Spark 核心组件
### Driver
Spark 驱动器节点，用于执行Spark任务中的 main 方法，负责实际代码的执行工作。Driver 在 Spark 作业执行时主要负责：  
1. 将用户程序转化为作业（job)
2. 在 Executor 质检调度任务（task)
3. 跟踪 Executor 的执行情况
4. 通过 UI 展示查询运行情况
### Executor
Spark Executor 节点是一个 JVM 进程，负责在 Spark 作业中运行具体任务，任务彼此之间相互独立。Spark 应用启动时，Executor 节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有 Executor 节点发生了故障或崩溃，Spark 应用也可以继续执行，会将出错节点上的任务调度到其他 Executor 节点上继续执行。  
Executor 有两个核心功能：
1. 负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程；
2. 它们通过自身的块管理器（Block Manager）为用户程序中要求缓存的 RDD 提供内存式存储。RDD 是直接缓存在 Executor 进程内的，因此任务可以在运行时充分利用缓存数据加速运算。
## Spark 通用运行流程概述
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark%E9%80%9A%E7%94%A8%E8%BF%90%E8%A1%8C%E6%B5%81%E7%A8%8B.jpg" width="400"></div>  
Spark 通用运行流程，不论 Spark 以何种模式进行部署，任务提交后，都会先启动 Driver 进程，随后 Driver 进程向集群管理器注册应用程序，之后集群管理器根据此任务的配置文件分配 Executor 并启动，当 Driver 所需的资源全部满足后，Driver 开始执行 main 函数，Spark 查询为懒执行，当执行到 action 算子时开始反向推算，根据宽依赖进行 stage 的划分，随后每一个 stage 对应一个 taskset，taskset 中有多个 task，根据本地化原则，task 会被分发到指定的 Executor 去执行，在任务执行的过程中，Executor 也会不断与 Driver 进行通信，报告任务运行情况。