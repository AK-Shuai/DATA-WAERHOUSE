# Zookeeper 架构 & Zab协议

## 重点：Zab、选举、脑裂、CAP。

## 架构与各角色职责

Zookeeper 顾名思义，动物园管理员，但是这个管理员也有不同的角色，普通管理员（Follower），代理队长（Leader），编外人员（Observer），在代理队长请假的情况下，所有的普通管理员都有资格参选队长，他们采用投票的方式来选举。小动物们（Client）有提需求（写数据）和取餐（读数据）的权利，队长和管理员都能处理取餐请求，但是提需求需要统一告诉队长让队长确认。另外出了个状况，人手不足，于是找来了编外人员，编外人员没有选举和参选权利，只能负责处理各种请求。这就是 Zookeeper 运营动物园的方式。 

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Zookeeper%E6%9E%B6%E6%9E%84%E5%9B%BE.png" width="400"></div>

- Client: Client 端的工作比较简单，就是一个请求的发起方，请求的对象可以是上图中Z ookeeper 的任意 Server 角色
- Leader: Leader 负责更新系统的状态，负责惊醒投票的发起和决议。还有处理事务请求，从图中可以看出，Client 端发起的请求，最终都是由其他角色转发给 Leader 来处理
- Follower: 接收并处理客户端读请求，并返回客户端结果；将客户端写请求转发给 Leader 处理；同步 Leader 的状态；在选举的过程中参与投票。
- Observer：接收并处理客户端读请求，并返回客户端结果；将客户端写请求转发给 Leader 处理；同步 Leader 的状态；但是在选举的过程中不参与投票
另外，Follower 和 Observer 都是 Leaner 角色，Leaner 需要做的事情上面也提到了，就是同步 Leader 的状态。

## Zab 协议

zab 协议的全称是 ZooKeeper Atomic Broadcast 即 zookeeper “原子”“广播”协议。它规定了两种模式：崩溃恢复和消息广播。Zab 协议是考察的重点，它是 Zookeeper 实现的核心，可以从选举和消息同步这两个方面取重点学习这个知识点。

### 崩溃恢复模式
在此模式下，会选举产生新的 Leader 服务器。以下三种情况 ZAB 都会进入恢复模式：

- 当 Zookeeper 在启动过程中。
- 当 Leader 服务器出现网络中断崩溃退出与重启等异常情况。
- 当有新 Server 加入到集群中且集群处于正常状态（广播模式），新 Server 会与 leader 进行数据同步，然后进入消息广播模式

进入崩溃恢复模式说明集群目前是存在问题的了，那么此时就需要开始一个选主的过程。zookeeper 使用的默认选主算法是 FastLeaderElection，它是标准的 Fast Paxos 算法实现，可解决 LeaderElection 选举算法收敛速度慢的问题。

Leader 选举完成之后，如果集群中已有的过半的 Server 与该 Leader 完成状态同步，ZAB 协议规定此时就会退出崩溃恢复模式。

#### Zab 协议规定的状态

投票的依据就是下面的两个 id，投票即是给所有服务器发送(myid,zxid)信息

- myid：用户在配置文件中自己配置，每个节点都要配置的一个唯一值，从 1 开始往后累加。
- zxid：zxid 有 64 位，分成两部分：

1. 高 32 位是 Leader 的 epoch：选举时钟，每次选出新的 Leader，epoch 累加 1
2. 低 32 位是在这轮 epoch 内的事务 id：对于用户的每一次更新操作集群都会累加 1。

注意：zk 把 epoch 和事务 id 合在一起，每次 epoch 变化，都将低 32 位的序号重置，这样做是为了方便对比出最新的数据，保证了 zxid 的全局递增性。

由于所有有效的投票都必须在同一轮次中。每开始新一轮投票自身的 epoch 自增 1。

- 接收到的 epoch 大于自己的。说明自己落后了，更新 epoch 后正常。
- 接收到的 epoch 小于自己的。忽略该票。
- 接收到的 epoch 与自己的相等，正常判断。

#### 选举的过程

<div align=center><img src="https://raw.githubusercontent.com/AK-Shuai/DATA-WAERHOUSE/main/%E5%9B%BE%E5%BA%8A/Zookeeper%E9%80%89%E4%B8%BE%E8%BF%87%E7%A8%8B.png" width="400"></div>

**发送和接收选票**：每个节点在初始状态下都会把自己的(myid,zxid)发送给其他所有节点，同时接收其他节点发来的选票并判断选票是否有效。 选票判断：

- 首先对比自身的和接收到的(myid,zxid)的 zxid 高 32 位的选举时钟 epoch，判断是否和自己在统一轮次。
- 一致则对比 zxid 低 32 的事务 id，一致则说明节点的数据是一致的，不一致则选择大的。
- 仍然一致则对比用户自己配置的 myid，选择大的。选完后广播自己的选举结果附带上(myid,zxid)。

**选举结束条件**：过半服务器选了同一个 Leader 则投票结束，根据投票结果更新自身状态为 Leader 或者 Follower。 
**新 Server 加入场景**：当新的节点加入时，发现集群中已经有 Leader 了，不再选举，直接将自己的状态从 LOOKING 改为 FOLLOWING。

### 消息广播模式

当集群状态稳定，即有 Leader 且过半机器状态同步完成，则退出崩溃恢复模式进入消息广播模式。在此模式下，会进行正常的消息同步，把客户端写入的数据从 Leader 同步到 Learner。 我们可以看回文章开头的那张架构图，图中也描述了消息广播的这个过程。下面是这个过程详细的描述。

1. Client 端向 Zookeeper 集群发起请求。
2. 如果接收到请求的 Server 不是 Leader 则把写请求转发给 Leader。如果是 Leader 忽略此步。
3. Leader 收到请求后首先为这个事务分配一个全局单调递增的唯一事务 ID (即 ZXID )，然后将事务以 Proposal 的形式发给所有 Follower 并等待 ACK 确认。
4. Follower 收到 Leader 的 Proposal 后将其以事务日志的形式写入到本地磁盘中去，并且在成功写入后反馈给 Leader 服务器一个 Ack 响应。
5. 当 Leader 收到半数以上 Follower 的 Ack 响应，说明该写操作可以执行，就会广播一个 Commit 消息给所有的 Follower 服务器以通知其进行事务提交，同时 Leader 自身也会完成对事务的提交。
6. Follower 收到信息后 Commit 事务。

## 相关面试题

### 什么是脑裂（Split-Brain）？

如果发生网络原因导致的心跳超时，Follower 会认为 Leader 宕机了，但其实 Leader 还存活着。这叫假死。 Leader 假死会导致新的选举发生，新 Leader 诞生后，旧 Leader 的网络又通了，导致出现了两个 Leader ，这时有的客户端连接到老的 Leader，有的客户端链接到新的 Leader。这就是脑裂。

### Zookeeper 是如何解决脑裂问题的？

上面的内容提到过，每开始一轮选举，选举时钟 epoch 都会加 1，follower 如果参与了这一轮的选举那么它会记录这个 epoch，如果出现 epoch 小于自己记录的值的请求，会一律拒绝，即事务执行失败。 如果有极个别的 follower 的 epoch 也有异常，那也没关系，因为一个事务的成功执行是需要有半数以上的 Follower 同意的，选举过程也是需要半数以上同意才会选举出新 leader，那就意味着正常的 Follower 一定是有半数以上的，只有 epoch 正常的 leader 才能够得到 Follower 的认同。

### zookeeper 是一个原子广播协议，哪里体现了原子性？

在崩溃恢复的选主过程中体现了它的原子性，zookeeper 在选主过程保证了两个问题：

- commit 过的数据不丢失，commit 过的数据半数以上参加选举的 follwer 都有，而且成为 leader 的条件是要有最高事务 id 即数据是最新的。
- 未 commit 过的数据丢弃。未 commit 过的数据只存在于 leader，但是 leader 宕机无法参加首轮选举，epoch 会小一轮，最终数据会丢弃。

### zookeeper 节点宕机如何处理？

**zookeeper **集群的机制是只要超过半数的节点正常，集群就能正常提供服务。所以半数以下的 Follower 宕机会集群不会有影响，数据不会丢失，只是损失了一点响应能力，失去了一个可以响应客户端的节点；如果是一个 Leader 宕机，Zookeeper 会选举出新的 Leader。具体流程见上面知识点的描述。

### 为什么有了 Follower 还需要 Observer ？

Observer 的设计提高了系统的扩展性。增加了 Zookeeper 可响应客户端的服务数量，提高并发能力和客户端读取数据响应效率。同时 Observer 不参与 Leader 的选举，可以使整个选举过程不会因为参与的节点过多而出现收敛速度慢的问题。

### 说说 Zookeeper 的 CAP 问题上做的取舍？

**一致性 C**：Zookeeper 是强一致性系统，为了保证较强的可用性，“一半以上成功即成功”的数据同步方式可能会导致部分节点的数据不一致。所以 Zookeeper 还提供了 sync() 操作来做所有节点的数据同步，这就关于 C 和 A 的选择问题交给了用户，因为使用 sync()势必会延长同步时间，可用性会有一些损失。 
**可用性 A**：Zookeeper 数据存储在内存中，且各个节点都可以相应读请求，具有好的响应性能。Zookeeper 保证了数据总是可用的，没有锁。并且有一大半的节点所拥有的数据是最新的。 
**分区容忍性 P**：Follower 节点过多会导致增大数据同步的延时（需要半数以上 follower 写完提交）。同时选举过程的收敛速度会变慢，可用性降低。Zookeeper 通过引入 observer 节点缓解了这个问题，增加 observer 节点后集群可接受 client 请求的节点多了，而且 observer 不参与投票，可以提高可用性和扩展性，但是节点多数据同步总归是个问题，所以一致性会有所降低。