# 分布式系统模式

原文地址：https://martinfowler.com/articles/patterns-of-distributed-systems/

作者：[Unmesh Joshi](https://twitter.com/unmeshjoshi)

译者注：本人主要是为了学习、技术交流，翻译这篇文章，如有侵权，请及时联系本人删除。

分布式系统对编程提出了特定的挑战。分布式系统往往要求有多个保持同步的数据副本。然而我们又无法保证节点能可靠的工作，网络延迟也能导致数据不一致。尽管如此，很多组织依赖于很多核心的分布式组件来处理数据存储，消息传递，系统管理和计算能力。这些系统往往面临共同的问题，而他们的解决方案往往也类似。本文将整理这些解决方案，将之发展成模式（pattern），以便更好的理解、交流、学习分布式系统设计。

[TOC]

## 本文讲述什么？ 

过去几个月，我在ThoughtWorks组织了几次关于分布式系统的workshop。组织这些workshop所面临的主要挑战之一就是，在保持议题足够通用、能够涵盖解决方案的前提下，如何将分布式系统的理论映射到像是Kafka或是Cassandra这样的开源代码中。模式的概念给出了一个比较漂亮的方式。

模式结构（pattern structure），容许我们关注于一个特定的问题，明确为何需要这个特定的解决方案。之后，方案陈述容许我们给出一个代码结构，以便在保持方案足够通用、涵盖一定范围变种的前提下，足够具体的展示这个解决方案。模式技术（pattern technique）也容许我们去将多种模式连接在一起，以便创建一个完整的系统。这给出了一个非常优雅的词汇来讨论分布式系统设计。

接下来将给出一系列在开源分布式系统中所使用的框架，希望这些模式能给所有开发者带来帮助。

### 分布式系统设计 - 一个实现视角

从本质上讲，如今的企业架构充满各种分布式的平台和框架。下面给出一些经常被用于典型的企业架构中的框架与平台：

| 平台/框架的类型                           | 实例                                       |
| ----------------------------------------- | ------------------------------------------ |
| 数据库databases                           | Cassandra, HBase, Riak                     |
| 消息中间件message brokers                 | Kafka, Pulsar                              |
| 基础设施infrastructure                    | Kubernetes, Mesos, Zookeeper, etcd, Consul |
| 内存数据<br/>In Memory Data/Compute Grids | Hazelcast, Pivotal Gemfire                 |
| 有状态的微服务stateful microservices      | Akka Actors, Axon                          |
| 文件系统file systems                      | HDFS, Ceph                                 |

上面这些本质上都是`分布式`。对于一个系统，分布式意味着什么呢？主要有两方面：

- 系统运行与多个服务器上。集群中的服务器数量少则3个，多则几千。
- 系统需要管理数据。所以这些系统是`有状态`的系统。

当多个服务器参与存储数据时，会有多种情况导致问题的出现，而上面提到的这些系统都必须解决这些问题。针对这些问题，上面这些系统有一些不断出现的解决方案。理解这些解决的一般形式，将有助于理解这些系统的整体实现思路，并能作为创建新系统时的一个指引。



### 模式

[模式](https://martinfowler.com/articles/writingPatterns.html) 概念由Christopher Alexander引入，被软件开发社区广泛接受，以便描述用于创建软件系统的设计理念。模式提供了一个结构化的方式去观察重复出现、被证明行之有效的问题以及解决方案。有个有趣的使用模式的方式是将几个模式链接到一起，形成一个模式序列（pattern sequence）或是模式语言（pattern language），以便给出实现整个系统的一些指导意见。从一些模式的角度去观察分布式系统，也是一个很有用的了解系统实现的方式。



## 问题与解决方案

当数据被存储到多个服务器上时，会有多种情况会导致问题。

### 进程崩溃(process crashes)

进程可以在任意时间崩溃。由于硬件问题，或是软件问题。进程失败有多种方式：

- 日常维护系统管理员关掉进程
- 由于硬盘已满，或是异常没有被正确处理，导致程序在做文件IO时被kill
- 在云环境中，会更加魔幻，一些无关的时间可能导致服务器down掉

对于负责保存数据的进程，其底线是必须能为保存在服务器上的数据给出一个持久化保证。如果进程已通知用户他们的数据已经被保存，，即使进程突然崩溃也必须保证数据不丢。根据访问方式的不同，不同的存储引擎有着不同的存储结构，从简单的哈希表到复杂的图存储。由于将数据刷到硬盘是最消耗时间的操作之一，不可能每个对于存储介质的插入、更新操作都被刷到硬盘中。所以多数数据库会定义一个定期刷数据到硬盘的内存存储结构。这就意味着，如果进程突然崩溃，我们可能丢失所有数据。

一项被称作预写日志（Write-Ahead Log，简称WAL）的技术可以解决这个问题。服务器将每个状态变化作为一个指令，只以追加方式写入到硬盘上的一个文件中。追加文件操作通常非常快，不会影响到程序性能。单独使用一个可以顺序追加的日志文件保存每次的更新。在服务器启动时可以通过重放日志来恢复内存中的数据。

这就给出了持久化保证，即便服务器突然崩溃、重启，数据也不会丢失。但如果服务器没有备份，客户端还是无法获取或是保存数据。所以我们目前还欠缺应对服务器挂掉的能力。

而应对该问题的解决方案之一，就是将数据保存到多个服务器上。所以我们可以将预写日志（WAL）写到多个服务器上。

当多个服务器一起参与到系统中，将需要考虑更多的失效场景。

### 网络延迟（network delays）

在TCP/IP协议栈中，通过网络传输数据所引起的延迟并没有上限。延迟的长短会随网络的负载而变化。例如，在一个带宽1Gbs的网络中，当一个大数据任务被触发时，可能会将网络缓存区填满，网络被淹没（flooded），导致一些消息在传递给服务器时遇到任意长度的延迟。

在一个典型的数据中心，多个服务器通过机架（rack）打包在一起，多个机架通过机架交换器（switch）连接在一起。可能会有一棵交换器树将数据中心的不同部分连接起来。在一些场景下，可能出现一部分服务器可以连接另一部分服务器，但却无法连接其他服务器。这种情况被称作**网络分裂（network partition）**。在服务器通过网络通讯时，最需要解决的基本问题就是何时知道一个服务器已经挂掉了。

有两个问题已经被解决了：

- 一个特定的服务器不可能无限期地等待，以便知道是否另一台服务器已经崩溃。
- 不应该有2个服务器集合，每个集合都认为对方集合中的服务器已经挂掉、因而继续为不同客户端提供服务。这种现象被称作**脑裂（split brain）**。

为了解决第一给问题，每个服务器定时发送一个**心跳（heartbeat）**消息给其他服务器。如果没有心跳消息，那么发送这个心跳的服务器就被认为是已崩溃了。心跳发送间隔应该足够小以便保证不需要太多时间就可以感知到服务器失效。正如我们下面所要看到的，在最坏情况下，服务器可能已经重启、运行起来，但整个集群作为一个整体，仍认为这台服务器是失效的。这保证了提供给客户端的服务不会被打断。

第二个问题就是脑裂。在脑裂情况下，如果两个集合的服务器分别独立地接收了更新，不同客户端可能获取、设置不同的数据，当脑裂被解决时，就不可能自动解决这些数据冲突。

为了处理脑裂问题，我们必须保证不能相互连接的两个服务器集合，不能独立进行操作（注：make progress，不确定这样翻译是否准确）。为了保证这一点，在服务器上进行的每个操作，只有当所有服务器中的多数都确认该操作时，才能认为该操作执行成功。如果服务器不能获得多数确认，那他们就不能提供所要求的服务，部分客户端可能就无法接收到这个服务，但在集群中的服务器将永远保证一致的状态。集群中能达到多数（majority）的服务器数量被称作**法定人数（quorum）**。如何决定法定人数呢？这取决于集群可以容忍的失效数量。所以如果我们有一个包含5个节点的集群，法定人数就需要设置为3。通常，如果我们想要容忍f个节点失效，就需要使集群大小为2f+1。

法定任务确保我们有足够的数据副本来应对部分服务器的失效。但这无法为客户端提供强一致性的服务。

### 进程挂起（process pauses）

### 时钟不同步和有序事件（unsynchronized clocks and ordering events）

## 拼在一起 - 一个分布式系统实例

### 顺序模式实现共识（pattern sequence for implementing consensus）



## 下一步（next step）