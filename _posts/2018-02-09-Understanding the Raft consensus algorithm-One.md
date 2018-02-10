---
title: 我对Raft的理解 - One
---

重新阅读了 Raft [论文](https://raft.github.io/raft.pdf)，结合 John Ousterhout 在斯坦福大学的课程[视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&feature=youtu.be)，对 Raft 重新梳理了一遍，并决定用文字记录下来。

Raft 是一个共识算法，何为共识算法？通俗的说，共识算法的目的就是要实现分布式环境下各个节点上数据达成一致。那么节点的数据为什么会出现不一致？原因有很多，例如节点宕机、网络延迟、数据包乱序等等。但是要注意的是，Raft 并不考虑存在恶意的节点的情况，也就是说，不存在主动篡改数据的节点。所以可以理解为：允许节点宕机，但是只要节点没有宕机，那么它就是正常工作的。

#### Slide 1

![](/img/raft.001.jpeg)

Raft 是为 Replicated Logs 设计的共识算法。一条日志对应于一个指令。可以这么理解：如果各个节点的日志在**数量**和**顺序**都达成一致，那么节点只需顺序执行日志，就能够得到一致的结果。注意，真正执行日志的是状态机（State Machine），Raft 协调的正是日志和状态机。 

#### Slide 2

![](/img/raft.002.jpeg)

再次回顾 Replicated Log，Raft 需要实现将日志完全一致的复制到其他节点，进而创建多副本状态机（Replicated State Machine），状态机可以理解为一个**确定**的应用程序，所谓确定是指只要是相同的输入，那么任何状态机都会计算出相同的输出。至于如何实现日志完全一致的复制，则是 Raft 即一致性模块（Consensus Module）需要做的事。

重新思考，为什么需要在多个节点维护一份完全一致的日志？如果只有 1 个节点提供服务，那么它就会成为整个系统的瓶颈，如果这个节点崩溃了，服务也就不能提供了。所以很自然的，需要让多个节点能够提供服务，也就是说，如果提供服务的某个节点崩溃了，系统中其他节点依旧可以提供等价的服务，但是如何做到等价？这就需要系统中的节点维持一致的状态。注意，实际上并不需要所有的节点同时拥有一致的状态，只要大多数节点拥有即可。大多数指的是：如果一共存在 3 个节点，允许 1 个几点不能正常工作；如果一共有 5 个节点，允许 2 个节点不能正常工作。为什么是大多数？我们将通过接下来的 Slides 进一步理解。

#### Slide 3

![](/img/raft.003.jpeg)

共识算法通常分为两类：对称式共识算法和非对称式共识算法。

* 对称式共识算法指网络中不存在中心节点 Leader，所有的节点都具有相同的地位，节点与节点之间通过互相通信来达成共识，即网络拓扑结构类似 P2P 网络。可想而知，对称式类的共识协议会非常复杂，但是性能会更好，因为网络中的节点可以同时提供服务。
* 非对称式共识算法会选举出一个 Leader，剩余的节点作为 Follower，客户端只能和 Leader 通信，节点之间的共识通过 Leader 来协调。相比于对称式共识算法，非对称式共识算法能够简化算法的设计，所有的操作都通过 Leader 完成，Follower 只需被动接受来自 Leader 的消息。

Raft 是一种非对称的共识算法，也正是采用了非对称的设计，Raft 得以将整个共识过程分解：**共识算法正常运行**和 **Leader 变更**。

#### Slide 4

![](/img/raft.004.jpeg)

Raft 论文中多次强调 Raft 的设计是围绕算法的**可理解性**展开，我们将从六个部分对 Raft 进行理解。

* Leader 选举，以及如何检测异常并进行新一轮的 Leader 选举。
* 基本的日志复制操作，也就是 Raft 正常运行是时的操作。
* 在 Leader 发生变更时如何保证安全性和一致性，这是 Raft 算法最关键的部分。
* 如何避免过时的 Leader 带来的影响，因为一个 Leader 宕机后再恢复仍然会认为自己是 Leader。
* 客户端交互，所谓实现线性化语义可以理解为实现幂等性。
* 配置变更，如何维持在线增删节点时的安全性和一致性。

#### Slide 5

![](/img/raft.005.jpeg)

Raft 算法有几个关键属性，我们需要提前了解。首先是节点的状态，相比于 Paxos，Raft 简化了节点可能的状态，在任何时候，节点可能处于以下三种状态。

* Leader。Leader 负责处理客户端的请求，同时还需要协调日志的复制。在任意时刻，最多允许存在 1 个 Leader，也就是说，可能存在 0 个 Leader，什么时候会出现不存在 Leader 的情况？接下来会说明。
* Follwer。在 Raft 中，Follower 是一个完全被动的角色，Follower 只会响应消息。注意，在 Raft 中，节点之间的通信是通过 RPC 进行的。
* Candidate。Candidate 是节点从 Follower 转变为 Leader 的过渡状态。因为 Follower 是一个完全被动的状态，所以当需要重新选举时，Follower 需要将自己提升为 Candidate，然后发起选举。

Raft 正常运行时只有一个 Leader，剩下的均为 Follower。

从状态转换图可以看到，所有的节点都是从 Follower 开始，如果 Follower 经过一段时间后收不到来自 Leader 的心跳，那么 Follower 就认为需要 Leader 已经崩溃了，需要进行新一轮的选举，因此 Follower 的状态变更为 Candidate。Candidate 有可能被选举为 Leader，也有可能回退为 Follower，具体情况下文会继续分析。如果 Leader 发现自己已经过时了，它会主动变更为 Follower，Leader 如何发现自己过时了？我们下文也会分析。

#### Slide 6

![](/img/raft.006.jpeg)

Raft 的另一个关键属性是任期（Term），在分布式系统中，由于节点的物理时间戳都不统一，因此需要一个逻辑时间戳来表明事件发生的先后顺序，Term 正是起到了逻辑时间戳的作用。Raft 的运行过程被划分为一系列 Term，一次 Leader 选举会开启一个新的 Term。

因为一次选举最多允许产生一个 Leader，一次选举又会开启一个新的 Term，所以每个 Leader 都会维护自己当前的 Term（Current Term）。注意，Leader 需要持久化存储 Current Term，当 Leader 宕机后再恢复，Leader 仍然会认为自己是 Leader，除非发现自己已经过时了，如何发现自己过时？依靠的正是 Current Term  的值。

一次 Term 也可能选不出 Leader，这是因为各个 Candidate 都获得了相同数量的选票，具体细节下文会再阐述。目前我们需要知道的是 Term 在 Raft 中是一个非常关键的属性，Term 始终保持单调递增，而 Raft 认为一个节点的 Term 越大，那么它所拥有的日志就越准确。

#### Slide 7

![](/img/raft.007.jpeg)

需要注意的是，Raft 有需要持久化存储的状态，包括 Current Term、VotedFor（下文会解析）和日志。每个日志项结构非常简单，包括日志所在 Term、Index 和状态机需要执行的指令。节点之间的 RPC 消息分为两类，一类为选举时的消息，另一类为 Raft 正常运行时的消息。具体细节我们会在下文理解。

#### Slide 8 

![](/img/raft.008.jpeg)

Raft 中 Leader 和 Follower 之间需要通过心跳消息来维持关系，Follower 一旦在 Election Timeout 后没有收到来自 Leader 的心跳消息，那么 Follower 就认为 Leader 已经崩溃了，于是就发起一轮新的选举。在 Raft 中，心跳消息复用日志复制消息 AppendEntries 数据结构，只不过不携带任何日志。

#### Slide 9

![](/img/raft.009.jpeg)

现在开始正式理解 Raft 的选举过程，大部分内容已有所介绍，我们再梳理一遍。

当新的一轮选举开始时，Follower 首先要自增当前 Term，代表进入新的任期，紧接着变更状态为 Candidate，每个 Candidate 会先给自己投上一票，然后通过发送 RequestVote RPC 消息呼吁其他节点给自己投票。选举结果存在三种可能。

* Candidate 收到了大多数节点的投票，那么 Candidate 自然就成为 Leader，然后马上发送心跳消息维护自己的 Leader 地位，并对外提供服务。
* Candidate 在等待来自其他节点的选票的过程中收到了来自 Leader 的心跳消息，Candidate 可以看到当前的心跳消息中包含更新的 Term，就会意识到新的 Leader 已经被选举出来，于是就自降为 Follower。
* 各个 Candidate 都获得了相同数量的选票，那么每个节点都会继续等待选票，没有新的 Leader 产生。等待一定的时间后，重新开启选举过程，知道选出新的 Leader。

需要考虑的是 Raft 如何避免重复出现 Candidate 瓜分选票的情况：如果当前轮选举 Candidate 瓜分了选票，那么Candidate 会进入下一轮的选举，但是各个 Candidate 开始选举的时刻是随机的。

#### Slide 10

![](/img/raft.010.jpeg)

继续理解选举过程，选举过程需要保证两个特性：Safety 和 Liveness。

* Safety 要求每个 Term 最多只能选举出一个 Leader，Raft 约束每个节点除了能给自己投一票，也只能给其他节点投一票。因此，如果 Candidate A 已经获得了大多数选票，由于每个节点只能向外投一票，因此 Candidate B 不可能获得大多数选票。Safety 特性保证一段时间内只可能存在一个 Leader 提供服务并协调日志的复制，避免因为存在多个 Leader 导致日志不一致。
* Safety 要求一段时间内最多只能存在一个 Leader，而 Liveness 保证系统最终必须要有要有一个 Candidate 赢得选举成为 Leader，Leader 无法选举出来意味着系统不能对外提供服务。Raft 实现 Liveness 的方式很简单，在 Slide 9 已经提及：当某一轮选举 Candidate 瓜分了选票，那么各个节点进入下一轮选举等待的时间是随机的，Candidate 随机等待 [T, 2T]， T 为选举超时时间，这样就大大减少了再次瓜分选票的概率。

#### 小结

对 Raft Leader 选举过程的理解基本结束，Raft 为了提高算法的可理解性，将问题分解，我们接下来会继续理解 Raft 的剩余部分。