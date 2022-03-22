# Paxos


## Paxos


**资料链接**

https://zhuanlan.zhihu.com/p/31780743

https://www.cnblogs.com/linbingdong/p/6253479.html

https://blog.openacid.com/algo/paxos/

chrome-extension://cdonnmffkdaoajfknoeeecmchibpmkmg/assets/pdf/web/viewer.html?file=https%3A%2F%2Fongardie.net%2Fstatic%2Fraft%2Fuserstudy%2Fpaxos.pdf

### 一、 背景问题

维基百科

- Paxos 是莱斯利·兰伯特（Leslie Lamport），1990 年提出的一种基于**消息传递**且具有**高度容错特性**的共识算法
- Paxos 常被误称为一致性（consistency）算法，但是 consistency 和 consensus 并不是一个概念。Paxos 是一个共识算法

**问题背景**

在分布式系统中，会发生各种异常错误情况，如何在一个分布式系统中，快速正确地对某个数据达成共识？Paxos 算法运行在系统中出现宕机故障，可以容忍消息的丢失、延迟、乱序以及重复，但是不考虑

Assume a collection of processes that can propose values. A consensus algorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen.

### 二、 角色

**Proposer** ： 提出提案（Proposal）。Proposal 信息包括提案编号（Proposal ID）和提议的值（Value）

**Acceptor** ：参与决策，回应 Proposers 的提案。收到提案后可以接受提案，若提案获得大多数接受，则称该提案被批准

**Learner** ： 不参与决策，从 Proposers/Acceptors 学习达成一致的提案

### 三、 算法流程

#### Phase 1.

​ (a) A **proposer** selects a `proposal` number n and sends a `prepare` request with number n to a majority of **acceptors**

​ (b) If an **acceptor** receives a `prepare` request with number n greater than that of any `prepare` request to which it has already responded, then it responds to the request with a `promise` not to accept any more `proposals` numbered less than n and with the highest-numbered proposal (if any) that it has accepted.

- `Prepare`阶段生成的 number ID 可使用时间戳 ➕Server ID

**Acceptor**进行`Promise`的承诺

（1）两个 Promise

- Acceptor 不接受 Proposal ID 小于等于当前请求的**Prepare**请求
- Acceptor 不接受 Proposal ID 小于当前请求的**Propose**请求

注意上面区别

##### （2）一个 Accept

- 回复已经接受过的提案中 Proposal ID 最大的那个提案的 Value（这个 Value 会被 Proposer 接收到后并学习）

#### Phase 2.

​ (a) If the **proposer** receives a response to its `prepare` requests (numbered n) from a majority of **acceptors**, then it sends an `accept` request to each of those **acceptors** for a `proposal` numbered n with a value v, where v is the value of the highest-numbered `proposal` among the reponses, or is any value if the responses reported no `proposals`.

​ (b) If an **acceptor** receives an `accept` request for a `proposal` numbered n, it accepts the `proposal` unless it has already responded to a `prepare` request having a number greater than n.

<img src="Paxos算法学习.assets/image-20220322110311770.png" alt="image-20220322110311770" style="zoom: 25%;" />

### 四、 Safety

- Only a value that has been proposed may be chosen
- Only a single value is chosen, and a process never learns that a value has been chosen unless it actually has been

#### Requirement

##### P1. An acceptor must accept the first proposal that it receives. （acceptor 必须接受它收到的第一个 proposal）

- But this requirement raises a problem. Several values could be proposed by different proposers at about the same time, leading to a situation in which every acceptor has accepted a value, but no single value is accepted by a majority of them. Even with just two proposed values, if each is accepted by about half the acceptors, failure of a single acceptor could make it impossibleto learn which of the values was chosen.

##### P1a. An acceptor can accept a proposal numbered n iff it has not responded to a prepare request having a number greater than n

##### P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.（如果值为 v 的 proposal 被选定，则任何被选定的具有更高编号的 proposal 值也一定是 v）

- Chosen ： A value is chosen when a single proposal with that value has been accepted by a majority of the acceptors.

- Since numbers are totally ordered, condition P2 guarantees the crucial safetyproperty that only a single value is chosen. (保证只有一个值被选中)

##### P2a. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v. （如果值为 v 的 proposal 被选定，则对所有的 acceptor，他们接受的任何具有更高编号的 proposal 的值也一定是 v）

##### P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v. （如果值为 v 的 proposal 被选定，则对所有的 proposer，他们提出的任何具有更高编号的 proposal 值也一定为 v）

##### P2c. For any v and n, if a proposal with value v and number n is issued, then there is a set S consisting of a majority of acceptors such that either

- (a) no acceptor in S has accepted any proposal numbered less than n, or

- (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S

### 五、 活锁 Liveness

- 多个 proposer 同时运行时，各个 proposal 的编号交替增加，没有任何的 proposal 可以被成功接受

<img src="Paxos算法学习.assets/image-20220322110344605.png" alt="image-20220322110344605" style="zoom:25%;" />

##### 解决

- 随机延迟，停等（有点类似于 CSMA 解决冲突的思路）
- 通过选主，只有主 Proposer 才能提出提案

## 拓展资料

### B 站视频

#### 1. CAP Theorem

- 一致性 consistency
- 可用性 availability
- 分区容错性 partition tolerance

#### 2. 一致性模型

- 弱一致性
  - 最终一致性 ： 立马读取可能结果不对
    - DNS（Domain Name System）
    - Gossip（Cassandra 的通信协议）
- 强一致性
  - Paxos
  - Raft（multi-paxos）
  - ZAB（multi-paxos）

* 分布式系统对 fault tolorence 的一般解决方案是 state machine replication
  - 讨论的主题也就是 state machine replication 的共识（consensus）算法

### OpenACID Blog 学习

#### 1. 核心思想

- 在廉价的易损设备上，通过多副本的冗余策略实现可靠性

- **注意此处与 Paxos 的区别，此处在数据库感觉更多的是为了让修改记录落盘，实现一致性，而不是在多个值中选一个进行共识**

- 给出了一个数据。 多副本的数据丢失风险

  1 副本 ～ 0.63%

  2 副本 ～ 0.00395%

  3 副本 ～ 0.000001%

  n 副本 ～ x^n （x=单副本损坏率）

#### 2. 常见方案

##### 一、 主从异步复制（Mysql 的 binlog 复制）

思路：应该只有主能够写，主收到写请求后，写入本地磁盘，然后返回响应，再同步给其他从节点

问题：在返回响应与同步至从节点之间存在异步，如果此时断电或者主损坏，则数据丢失

##### 二、主从同步复制

思路：应该只有主能够写，主收到写请求后，写入本地磁盘，然后同步给其他从节点，再返回响应

问题：主返回响应之前需要确保同步到所有的从节点，这样效率很低，速度很慢，同时如果有从节点出现问题，则整个完成就完全阻塞

##### 三、 主从半同步复制

思路：应该只有主能够写，主收到写请求后，写入本地磁盘，然后同步给一些从节点（不是全部），再返回响应

问题：如果数据 a 复制给从 1 节点，数据 b 复制给从 2 节点，结果 master 宕机，则从 slave1 和 slave2 恢复出的数据是不一样的，就是说半同步复制还是会存在数据不一致问题

##### 四、多数派写（读）

思路：每条数据必须写入到**半数以上**机器上，每次读取数据都必须检查**半数以上**机器是否存在该条数据

问题 1：node1 和 node2 都写入了 a=x，下一次更新时 node2 和 node3 写入了 a=y，这时如果客户读取 node1 和 node2 会发现不一致，

解决 1：通过比较记录时间戳，来选择最新的状态，保证不产生歧义

问题 2：如果客户在写入 a=y 时，只写了 node3 的，node2 还没写进去就挂了，此时个节点中最新的状态 node1-x；node2-x；node3-y，则如果客户读取 node1 和 node2 会得到 x；读取 node2 和 node3 或者 node1 和 node3 会得到 y；因为 node3 中 y 的时间戳是最新的

#### 五、 Paxos

<img src="https://blog.openacid.com/post-res/paxos/slide-imgs/paxos-16.jpg" alt="img" style="zoom:67%;" />

- 上图存在的问题： 在 x 设置 i 值的同时，这个过程还没做完，y 就准备插一手来设置 i 值了。

  - 个人认为：核心问题发生在没有加锁

- 文章原话是说：“在 X 或 Y 写之前先做一次多数派读，以便确认是否有其他客户端进程已经在写了，如果有，则放弃”

解读

- 个人觉得这里的先做一次多数派读就是 Prepare 的过程
- Paxos 也并不是发现有人在写就放弃的，而是在 Prepare 的时候就把自己的锁给加上了，然后抢占，所以才会导致活性的问题
  - 这里如果单纯只是分布式加锁，也会存在一个问题，就是双方都只是加了锁，也都是简单的锁，因为请求可以同步过来，那么解决这个问题就是使用时间戳，只有最后一个锁，才是有效的

尚待完善...

