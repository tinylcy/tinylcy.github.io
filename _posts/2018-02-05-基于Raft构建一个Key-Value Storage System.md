---
title: '基于 Raft 构建一个分布式 Key-Value Storage System'
---

最近在做毕业设计，涉及到 Raft 等一致性协议的问题。在阅读了 Raft [论文](chrome-extension://cdonnmffkdaoajfknoeeecmchibpmkmg/assets/pdf/web/viewer.html?file=https%3A%2F%2Fraft.github.io%2Fraft.pdf)之后，很自然的需要造点东西来加深对论文的理解。我暂时不会尝试去实现 Raft 协议，而是先基于 Raft 构建一个分布式的内存型 Key-Value Store，名字叫做 [Rastore](https://github.com/tinylcy/rastore)。

Raft 的开源实现已有很多，比较流行的是 CoreOS etcd 和 HashiCorp raft。相比而言，HashiCorp raft 实现的比较完整，在使用的时候仅需关注业务逻辑的实现，并 Apply 需要同步的信息，而 CoreOS etcd 则是 Raft 协议的轻量级实现，这意味着很多协议相关的模块都需要我们去实现，这无疑是增加了复杂度，但是带来的好处是更加灵活。Rastore 选择使用 HashiCorp raft 来构建系统。

Rastore 目前具备四个主要特性。

* 容错性
* 在线增删节点
* Key-Value Store 增删改查操作
* 状态机快照

Rastore 包括 Store 和 Service 两个核心模块。Store 模块对 HashiCorp raft 进行了封装，实现了 Raft 节点的初始化及启动、Key-Value Store 的增删改查逻辑、集群节点的在线增删和快照的创建及恢复。Service 模块通过提供 RESTFul 接口，给外界提供了更加透明的 Key-Value 存储服务。下图清晰的描述了 Rastore 模块之间的关联。

![](/img/img-2018-02-05-Image-1.jpeg)

Rastore 实现了一个简化版的内存 Key-Value Storage System，源码放在了 [GitHub](https://github.com/tinylcy/rastore) 上。

### 附录

HashiCorp 对 Raft 的关键 term 进行了总结，我觉得写得不错，引用如下。

- Log - The primary unit of work in a Raft system is a log entry. The problem of consistency can be decomposed into a *replicated log*. A log is an ordered sequence of entries. We consider the log consistent if all members agree on the entries and their order.
- FSM - [Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine). An FSM is a collection of finite states with transitions between them. As new logs are applied, the FSM is allowed to transition between states. Application of the same sequence of logs must result in the same state, meaning behavior must be deterministic.
- Peer set - The peer set is the set of all members participating in log replication. For Consul's purposes, all server nodes are in the peer set of the local datacenter.
- Quorum - A quorum is a majority of members from a peer set: for a set of size `n`, quorum requires at least `(n/2)+1`members. For example, if there are 5 members in the peer set, we would need 3 nodes to form a quorum. If a quorum of nodes is unavailable for any reason, the cluster becomes *unavailable* and no new logs can be committed.
- Committed Entry - An entry is considered *committed* when it is durably stored on a quorum of nodes. Once an entry is committed it can be applied.
- Leader - At any given time, the peer set elects a single node to be the leader. The leader is responsible for ingesting new log entries, replicating to followers, and managing when an entry is considered committed.

### 参考

[1] Rastore, [https://github.com/tinylcy/rastore](https://github.com/tinylcy/rastore).

[2] Raft Consensus Algorithm, [https://raft.github.io/raft.pdf](https://raft.github.io/raft.pdf).

[3] HashiCorp raft, [https://github.com/hashicorp/raft](https://github.com/hashicorp/raft).

[4] hraftd, [https://github.com/otoolep/hraftd](https://github.com/otoolep/hraftd).

[5] HashiCorp Raft Protocol Overview, [https://www.consul.io/docs/internals/consensus.html#raft-protocol-overview](https://www.consul.io/docs/internals/consensus.html#raft-protocol-overview).