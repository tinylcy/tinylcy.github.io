---
title: MapReduce论文阅读
date: 2017-01-07 21:03:08
tags: DistributedSystems
toc: true
---


大四时曾经粗略的阅读过这篇[论文](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)，并且已经写过不少的`MapReduce`程序，所以介绍性的内容不再赘述。再次阅读这篇论文的原因是为了更系统的学习分布式的相关知识，我开始跟进[MIT 6.824: Distributed Systems](http://nil.csail.mit.edu/6.824/2015/index.html)，而完成这门课程的第一个`lab`的前提便是阅读这篇论文。

这篇笔记重点分析了`MapReduce`的**执行流程**以及**容错机制**，因为是个人理解，若有分析不妥之处欢迎发送邮件至`tinylcy (at) gmail.com`讨论交流。

## Execution Overview

根据不同的环境，`MapReduce`的实现方式有多种，比如基于共享内存、基于`NUMA`多处理器环境等等。而`Google`内部实现的`MapReduce`基于如下环境。

* 双核`x86`处理器，`Linux`操作系统，每台机器有`2～4GB`的内存。
* 使用商用的网络设备，例如`100M`／`1G`带宽网卡。
* 集群是由成百上千台上述配置的设备组成的，因此集群中节点出现故障应该视为常态。
* 存储设备采用的是廉价的`IDE`硬盘。在这种不可靠的硬件上，`Google`实现了一个分布式文件系统`GFS`，通过备份和冗余来保证可靠性和可用性。
* 用户将作业（`job`）提交到调度系统，每个作业由多个任务（`task`）组成，调度系统负责将任务分配到集群空闲的节点上。

在`Map`阶段，输入数据会被自动划分为`M`个分片，这些分片可以在不同的节点上被并行处理。在`Reduce`阶段，根据一定的划分规则（例如`hash(key) mod R`），中间数据会被划分为`R`个分片，这`R`个分片也可以被多个节点同时处理。`Reduce`阶段的分片个数`R`和分片规则可以由用户指定。`MapReduce`的执行流程如下图所示。

![Alt text](/img/img-2017-01-07-Image 1.png)

整个执行流程可以划分为如下几个阶段，上图中的数字也标识了这几个阶段。

* 用户程序将输入数据切分为`M`个分片（分片的大小一般为`16～64MB`，用户可以设置分片大小），并把用户程序拷贝到集群中的多个节点。因为数据要比程序大得多，所以“拷贝程序”要比“拷贝数据”高效的多。
* 在拷贝程序到节点的过程中，有一个节点比较特殊：`master`节点。其余的节点都为`worker`节点，`worker`节点负责执行具体的任务，这些任务通过`master`节点来分配。任务又分为`map task`和`reduce task`。
* 被分配到`map task`的`worker（map worker）`会读取相应的输入分片，并将输入分片中的数据解析为一系列的`key/value pairs`，然后将这些`key/value pairs`输入到用户定义的`map function`，`map function`输出的`key/value pairs`会被缓存到内存中，而不是直接写入磁盘。
* 由`map function`输出的，缓存在内存中的`key/value pairs`会被划分为`R`个分区，并定期写入到**本地**磁盘中。写入磁盘的位置会被推送给`master`节点，`master`节点会将磁盘的位置信息转发给下一阶段执行`reduce`任务的节点（`reduce worker`）。
* `reduce worker`在接收到磁盘的位置信息后开始读取相应的磁盘中的数据，当所有的数据读取完毕后，`reduce worker`会在内存中按照`key`将所有的`key/value pairs`进行一次排序。论文认为这次排序是必要的原因是不同的`key`往往会映射到同一个`reduce worker`。
* `reducer worker`遍历已排好序的`key/value pairs`，每遇到一个不同的`key`，便将该`key`和对应的一系列`value`传递给用户定义的`reduce function`，这个过程同时解释了为什么在上一阶段`reduce worker`要对数据进行排序（论文`Section 4.2`提到了按照key排序的两个优势，一是支持高效随机按`key`的查找，二是已经排好序的数据可以方便用户的操作）。`reduce function`将输出数据`append`到最终的输出文件中。
* 当所有的`map task`和`reduce task`都完成了，`master`唤醒用户程序并返回。

在执行完整个流程后，会有`R`个输出文件，每个`reduce worker`对应一个。这`R`个输出文件一般不需要合并，因为它们往往是下一个`MapReduce`处理逻辑的输入数据。

## Master Data Structures

* `master`会记录每个`map task`和`reduce task`的状态，包括`idle`、`in-progress`和`completed`。同时，`master`还会记录`non-idle task`对应的`worker`的信息。
* 对于每个`map task`，`master`会记录下`map function`输出的`R`个分片的位置信息和大小。这些信息会被推送到处于`in-progress`状态的`reduce worker`上。

## Fault Tolerance

由于运行在规模庞大并且廉价的硬件上，因此容错性变得非常重要。

### Worker Failure

`master`会定期`ping` `worker`，如果`worker`没有响应并且超过了一定的次数，那么`master`就认为`worker`已经`failed`了。因此，所有在该`worker`上完成的`task`的状态将会被重置为初始的`idle`状态，并且这些`task`需要被重新分配到其它的`worker`上去。类似的，该`worker`上处于`in-progress`状态的`task`也会被重置为最初的`idle`状态，并被重新分配到其它`worker`上去。

对于已完成的`map task`，也需要重新被执行。因为`map task`的输出是在`worker`的本地磁盘上，因为`worker`已经失联了，所以`map task`的输出数据自然也获取不到。对于已完成的`reduce task`，不再需要重新执行。因为`reduce task`的输出是在全局的文件系统（`GFS`）上。

如果一个`map task`一开始运行在`worker A`上，接着由于`worker A` `failed`导致该`map task`迁移到`worker B`上。那么读取该`map task`输出数据并且处于正在执行的`reduce worker`会收到重新执行`reduce task`的通知，任何还未开始读取数据的`reduce task`也会收到通知。`reduce worker`接下来会从`worker B`上读取数据。

### Master Failure

可以通过定期建立检查点的方式来保存`master`的状态。但是，`Google`当时的做法是考虑到只有一个`master`，所以`master`出现故障的概率很小，如果出现故障了，重新开始整个`MapReduce`计算。

## Locality

网络带宽在计算环境中属于一种非常稀缺的资源，利用输入数据的特性可以减小网络带宽。

* 输入数据由`GFS`来管理，`GFS`把数据存储在集群节点的本地磁盘上，`GFS`将文件分割为`64MB`大小的块，并且针对每个块会做冗余（一般冗余`2`份）。`master`利用输入数据的位置信息，将`map task`分配给输入数据所在的节点。
* 如果在计算过程中出现了失败的情况，那么`master`会把任务调度给离输入数据较近的节点。

## Task Granularity

从上文我们可以得知，`map`阶段被划分成`M`个`task`，`reduce`阶段被划分成`R`个`task`，`M`和`R`一般会比集群中节点的个数大得多。每个节点运行多个`task`有利于动态的负载均衡，加速`worker`从失败中恢复。

在具体的实现中，`M`和`R`的大小是有实际限制的，因为`master`至少要做`O(M＋R)`次的调度决策，并且需要保持`O(M*R)`个状态。

通常情况下，`R`的大小是由用户指定的，而对`M`的选择要保证每个`task`的输入数据大小，即一个输入分片在`16MB～64MB`之间，这样可以最大化的利用数据本地性。

## Backup Tasks

导致整个`MapReduce`计算过程被延迟的原因之一是过多的时间花费在最后几个`map task`或`reduce task`上。导致这个问题的原因由很多，可能是因为`task`所在的节点硬盘的读写速度非常慢，同时`master`又有可能把新的`task`分配给了该节点，所以引入了更加激烈的`CPU`竞争、内存竞争。

一种通用的解决方案是在整个`MapReduce`计算快要结束时，`master`对当前处于`in-progress`状态的`task`进行备份，无论是原来的`task`执行完毕还是备份的`task`执行完毕，那么就认为该`task`完成了。

## 参考

* [MapReduce: Simplified Data Processing on Large Clusters](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)
