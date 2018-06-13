---
title: RocketMQ - 初探
---

五月，我回到杭州实习，做了一点微小的工作，为接下来的正式入职做了一定程度的预热。期间对公司内部的中间件有了些许实践，但也仅仅是浅尝辄止，想想有些心悸，所以决定针对 RocketMQ 做点源码剖析的工作。为什么选择 RocketMQ？理由有五：（1）RocketMQ 是公司开源的一款高性能、高吞吐量的分布式消息中间件（内部代号 MetaQ），方便写文章；（2）理解 Message Queue，知其然知其所以然；（3）理解分布式消息中间件如何实现高性能的网络通信及异常处理（网络传输存在第三种状态，超时）；（4）理解 RocketMQ 的消息的存储机制；（5）读点高质量的代码，权当开阔眼界罢。

本文是 RocketMQ 系列的首篇文章，暂不涉及源码，重点对 RocketMQ 中的核心概念进行梳理。

### 基本概念

**Producer**

消息生产者，负责创建消息，一般由业务系统负责产生消息。

**Consumer**

消息消费者，一般由后台系统异步消费消息。

**Push Consumer**

Consumer 的一种，应用通常向 Consumer 对象注册一个 Listener 接口，一旦接收到消息，Push Consumer 对象立刻回调 Listener 接口的方法。

**Pull Consumer**

Consumer 的一种，应用通常主动调用 Consumer 的拉取消息方法从 Broker 拉取消息，主动权由应用控制。

**Producer Group**

一类 Producer 的集合名称，这类 Producer 通常发送一类消息，且发送逻辑一致。

**Consumer Group**

一类 Consumer 的集合名称，这类 Consumer 通常消息一类消息，且消费逻辑一致。

**Broker**

消息中转角色，负责存储消息和转发消息。

**广播消息**

一个消息被多个 Consumer 消费，如果 Consumer 属于同一个 Consumer Group，那么消息会被该 Consumer Group 中的每个 Consumer 消费。

**集群消费**

一个 Consumer Group 中的 Consumer 平均分摊消费消息。例如一个 Topic 存在 9 条消息，并且 Consumer Group 中包含 3 个 Consumer，那么每个 Consumer 各消费 3 条消息。

**顺序消息**

消息被消费的顺序需要同消息非发送的顺序一致，在 RocketMQ 中，顺序消息指的是局部顺序。消息为满足顺序性，必须由 Producer 单线程发送，且发送至同一个队列，Consumer 根据 Producer 发送的顺序去消费消息。

**普通顺序消息**

顺序消息的一种。正常情况下可以保证完全的顺序消息，但是一旦发送通信异常，例如 Broker 重启，此时由于队列的总数发生了变化，哈希取模后选择的队列也会发生变化，由此产生短暂的消息顺序不一致。

**严格顺序消息**

顺序消息的一种。无论正常或是异常情况都能保证消息顺序，但是牺牲了分布式 Failover 特性，即 Broker 集群中只要有一台机器不可用，那么整个集群都不可用，服务可用性大大降低。在已知的应用场景中，数据库 binlog 同步强依赖严格消息顺序。

**Message Queue**

在 RocketMQ 中，所有的消息队列都是持久化、长度无限的数据结构。所谓长度无限是指队列中的每个存储单元都是定长，使用 offset 访问每个存储单元，offset 为 Java Long 类型（64 bit），理论上 100 年内不会溢出，所以可以认为是长度无限。RocketMQ 的消息队列一般保证保存一定时间内的数据，过期数据将会被删除。也可任务一个消息队列就是一个数组，offset 即为数组下标。

### RocketMQ需要解决哪些问题

**Publish/Subscribe**

发布/订阅是消息中间件最基本的功能。

**Message Priority**

由于 RocketMQ 所有的消息都是持久化存储的，如果需要按照优先级对消息进行排序，开销会非常大，因此 RocketMQ 没有特意支持消息优先级。但是可以通过变通的方式实现类似功能，即单独配置一个优先级高的消息队列和一个普通优先级的消息队列，将不同优先级的消息发送至相应的队列。

**Message Order**

消息有序指在消费一系列消息时需要按照这一系列消息的发送顺序进行消费。RocketMQ 可以严格保证消息消费的有序性。

**Message Filter**

消息过滤分为 Broker 端消息过滤和 Consumer 端的消息过滤。

Broker 端消息过滤的优点在于减少了对于 Consumer 无用消息的网络传输，缺点在于增加了 Broker 的负担，实现相对复杂。

Consumer 端消息过滤可以通过应用自定义实现，缺点在于可能存在大量的无用的消息被传输至 Consumer 端。

**Message Persistence**

一般而言消息中间件采用四种持久化的方式。（1）持久化至数据库，如 MySQL；（2）持久化至 KV 存储，如 LevelDB、Berkeley DB；（3）文件记录形式持久化，如 Kafka；（4）对内存出具做一个持久化镜像。

RocketMQ 参考了 Kafka 的持久化方式，充分利用 Linux 文件系统内存 Cache 提高性能。

**Message Reliability**

// TODO

**Low Latency Messaging**

在消息不堆积的情况下，消息到达 Broker 之后，能够立刻到达 Consumer。RocketMQ 采用长轮询 Pull 的方式，可保证消息非常实时，消息实时性不低于采用 Pull 的方式。

**At Least Once**

At Least Once 指任意一条消息至少投递一次。RocketMQ Consumer 首先 Pull 消息至本地，当消费逻辑完成后，才会向 RocketMQ 集群返回 ACK。如果没有完成消费一定不会 ACK 消息，RocketMQ 会继续投递消息。

**Broker  Buffer**

对于一般的消息队列，Broker Buffer 通常指在 Broker 中的一个队列的内存 Buffer，这类 Buffer 通常有大小限制。

RocketMQ 没有内存 Buffer 的概念，RocketMQ 的队列都是持久化磁盘，数据定期清除。或者说，RocketMQ 的内存 Buffer 被抽象成一个无限长度的队列，但是前提是 Broker 会定期清除过期的数据。

**回溯消费**

回溯消费指 Consumer 已经消费成功的消息，由于业务上的需求需要重新消费消费，为了支持回溯消费，Broker 在向 Consumer 成功投递消息后，消息仍然需要保留。回溯消费一般是按照时间维度，例如由于 Consumer 系统故障，恢复后需要重新消费 1 小时前的消息，因此 Broker 需要提供一种机制，允许按照时间维度来回退消息消费进度。

RocketMQ 支持按照时间回溯消费，时间维度精确至毫秒，支持向前回溯，也支持向后回溯。

**消息堆积**

消息中间件的主要功能是异步解耦，还有一个重要功能是抵挡前端的数据洪峰，保证后端系统的稳定性，这要求消息中间件具备一定的消息堆积能力，消息堆积分为两种情况。

（1）消息堆积至内存 Buffer，一旦超过内存 Buffer，可以根据一定的丢弃策略来丢弃消息，适用于能够容忍丢弃消息的业务场景。这种情况的消息堆积能力主要取决于内存 Buffer 的大小，消息堆积后性能不会发生明显下降。

（2）消息堆积至持久化存储系统中，例如 DB、KV 存储和文件记录形式。当消息不能在内存 Cache 命中时，需要访问磁盘，此时会产生大量的读 IO，因此读 IO 的吞吐量直接决定了消息堆积后的访问能力。

**分布式事务**

分布式事务设计到两阶段提交问题。在数据存储方面一般需要 KV 存储的支持，因为第二阶段的提交回滚需要修改消息状态，此时会涉及到根据 Key 查找 Message 的动作。RocketMQ 在第二阶段没有通过 Key 去查找 Message，而是在第一阶段发送 Prepared 消息时就获取了消息的 offset，在第二阶段通过 offset 访问消息，并修改状态，offset 就是数据的地址。RocketMQ 采用 offset 实现分布式事务，可能会导致系统的脏页过多，需要特别关注。

**定时消息**

定时消息指消费被发送至 Broker 后，不被 Consumer 立即消费，而是要在特定的时间点或者等待确定的时候后才能被消费。如果需要支持任意的时间精度，在 Broker 层面就需要做消息的排序，同时 RocketMQ 持久化存储消息的特性，势必会造成巨大的性能开销。

RocketMQ 支持定时消息，但是不支持任意的时间精度，支持特定的 leve，如 5s、10s、1min 等。

**消息重试**

Consumer 消费消息失败后，消息中间件需要提供一种重试机制，令消费再被消费一次。Consumer 消费消息失败分为两种情况。

（1）由于消息本身的原因，如反序列化失败，消息数据本身无法被处理。这种情况通常需要跳过这条消息，转而消费其他消息，并且失败消息即使立刻重新被消费往往也不会成功，所以一般提供一种定时重试机制。

（2）由于应用服务器不可用导致消息消费失败。这种情况下即使去消费其他消息往往也会报错，所以建议应用 sleep 一段时间再消费下一条消息，减轻 Broker 重试消息的压力。

### RocketMQ物理部署结构

![](/img/img-2018-06-09-rmq-basic-arc.png)

**NameServer Cluster**

NameServer 提供轻量级的服务发现和路由。每一个 NameServer 记录了所有的路由信息，提供对应的读写服务。NameServer 是一个几乎无状态的节点，可集群部署，节点之间无任何信息同步。

**Broker Cluster**

Broker 负责消息的存储。Broker 部署相对负责，分为 Master 和 Slave，一个 Master 可以对应于多个 Slave，但是一个 Slave 只能对应于一个 Master。Master 和 Slave 的对应关系通过指定相同的 BrokerName 和不同的 BrokerId 来定义。每个 Broker 与 NameServer Cluster 中的所有节点建立长连接，定时注册 Topic 信息至所有的 NameServer。

**Producer Cluster**

Producer 与 NameServer Cluster 中的一个节点（随机选择）建立长连接，定期从 NameServer 获取 Topic 路由信息，并向提供 Topic 服务的 Master 建立长连接，且定时向 Master 发送心跳。Producer 完全无状态，可以集群部署。

**Consumer Cluster**

Consumer 与 NameServer Cluster 中的一个节点（随机选择）建立长连接，定期从 NameServer 获取 Topic 路由信息，并向提供 Topic 服务的 Master 和 Slave 建立长连接，且定时向 Master 和 Slave 发送心跳。Consumer 既可以从 Master 订阅消息，也可以从 Slave 订阅消息，订阅规则由 Broker 配置决定。

### 参考

[1] [Apache RocketMQ](https://rocketmq.apache.org/)

[2] [RocketMQ 原理简介](http://alibaba.github.io/RocketMQ-docs/document/design/RocketMQ_design.pdf)



 

