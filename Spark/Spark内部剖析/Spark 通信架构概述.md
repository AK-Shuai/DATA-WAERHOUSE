# Spark 通讯架构

## Spark 通信架构概述

Spark2.x 版本使用 Netty 通讯框架作为内部通讯组件。spark 基于 netty 新的 rpc 框架借鉴了 Akka 中的设计，它是基于 Actor 模型，如下图所示：

<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Actor%E6%A8%A1%E5%9E%8B.jpg" width="400"></div>

Spark 通讯框架中各个组件（Client/Master/Worker）可以认为是一个个独立的实体，各个实体之间通过消息来进行通信。具体各个组件之间的关系图如下： 

<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark%E9%80%9A%E8%AE%AF%E6%9E%B6%E6%9E%84.png" width="400"></div>

Endpoint（Client/Master/Worker）有 1 个 InBox 和 N 个 OutBox（N>=1，N 取决于当前 Endpoint 与多少其他的 Endpoint 进行通信，一个与其通讯的其他 Endpoint 对应一个 OutBox），Endpoint 接收到的消息被写入 InBox，发送出去的消息写入 OutBox 并被发送到其他 Endpoint 的 InBox 中。

## Spark 通讯架构解析
Spark 通信架构如下图所示：
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark%E9%80%9A%E4%BF%A1%E6%9E%B6%E6%9E%84.png" width="400"></div>

1) RpcEndpoint：RPC 端点，Spark 针对每个节点（Client/Master/Worker）都称之为一个 Rpc 端点，且都实现 RpcEndpoint 接口，内部根据不同端点的需求，设计不同的消息和不同的业务处理，如果需要发送（询问）则调用 Dispatcher；

2) RpcEnv：RPC 上下文环境，每个 RPC 端点运行时依赖的上下文环境称为 RpcEnv；

3) Dispatcher：消息分发器，针对于 RPC 端点需要发送消息或者从远程 RPC 接收到的消息，分发至对应的指令收件箱/发件箱。如果指令接收方是自己则存入收件箱，如果指令接收方不是自己，则放入发件箱；

4) Inbox：指令消息收件箱，一个本地 RpcEndpoint 对应一个收件箱，Dispatcher 在每次向 Inbox 存入消息时，都将对应 EndpointData 加入内部 ReceiverQueue 中，另外 Dispatcher 创建时会启动一个单独线程进行轮询 ReceiverQueue，进行收件箱消息消费；

5) RpcEndpointRef：RpcEndpointRef 是对远程 RpcEndpoint 的一个引用。当我们需要向一个具体的 RpcEndpoint 发送消息时，一般我们需要获取到该 RpcEndpoint 的引用，然后通过该应用发送消息。

6) OutBox：指令消息发件箱，对于当前 RpcEndpoint 来说，一个目标 RpcEndpoint 对应一个发件箱，如果向多个目标 RpcEndpoint 发送信息，则有多个 OutBox。当消息放入 Outbox 后，紧接着通过 TransportClient 将消息发送出去。消息放入发件箱以及发送过程是在同一个线程中进行；

7) RpcAddress：表示远程的 RpcEndpointRef 的地址，Host + Port。

8) TransportClient：Netty 通信客户端，一个 OutBox 对应一个 TransportClient，TransportClient 不断轮询 OutBox，根据 OutBox 消息的 receiver 信息，请求对应的远程 TransportServer；

9) TransportServer：Netty 通信服务端，一个 RpcEndpoint 对应一个 TransportServer，接受远程消息后调用 Dispatcher 分发消息至对应收发件箱；

根据上面的分析，Spark 通信架构的高层视图如下图所示：
<div align=center><img src="https://raw.githubusercontent.com/shuainuo/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Spark%E9%80%9A%E4%BF%A1%E6%9E%B6%E6%9E%84%E7%9A%84%E9%AB%98%E5%B1%82%E8%A7%86%E5%9B%BE.jpg" width="400"></div>
