# 分布式系统模式

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/

分布式系统给软件开发带来了一些特殊的挑战，要求数据有多个副本，且彼此间要保持同步。然而，我们不能保证所有工作节点都能可靠地工作，网络延迟会轻易地就造成不一致。尽管如此，许多组织依然要依赖于一系列核心的分布式软件处理，诸如数据存储、消息通信、系统管理以及计算能力等等。这些系统面临着共同的问题，可以采用类似的方案进行解决。本文将这些方案进行分类，进一步拓展成模式，以此我们获得更好的理解，改善对于分布式系统设计的理解、交流和传授方式。

**2020.8.2**

> Unmesh Joshi
>
> Unmesh Joshi 是 ThoughtWorks 的总监级咨询师。他是一个软件架构的爱好者，相信在今天理解分布式系统的原则，同过去十年里理解 Web 架构或面向对象编程一样至关重要。

## 这个系列在讨论什么

在过去的几个月里，我在 ThoughtWorks 内部组织了许多分布式系统的工作坊。在组织这些工作坊的过程中，我们面临的一个严峻挑战就是，如何将分布式系统的理论映射到诸如 Kafka 或Cassandra 这样的开源代码库上，同时，还要保持讨论足够通用，覆盖尽可能广泛的解决方案。模式的概念为此提供了一个不错的出路。

从模式的本质上说，其结构让我们可以专注在一个特定的问题上，这就很容易说清楚，为什么需要一个特定的解决方案。解决方案的描述让我们有了一个代码结构，对于展示一个实际的解决方案而言，它足够具体，对于涵盖广泛的变体而言，它又足够通用。模式技术还可以将不同的模式联系在一起，构建出一个完整的系统。由此，便有了一个讨论分布式系统实现非常好的词汇表。

下面就是从主流开源分布式系统中观察到的第一组模式。希望这组模式对所有的程序员都有用。

### 分布式系统：一个实现的视角

今天的企业架构充满了各种天生就分布的平台和框架。如果从今天典型的企业应用架构选取典型平台和框架组成列表，我们可能会得到类似于下面这样一个列表：

|**平台/框架的类型**|**样例**|
|:----|:----|
|数据库|Cassandra、HBase、Riak|
|消息队列|Kafka、Pulsar|
|基础设施|Kubernetes、Mesos、Zookeeper、etcd、Consul|
|内存数据/计算网格|Hazelcast、Pivotal、Gemfire|
|有状态微服务|Akka Actors、Axon|
|文件系统|HDFS、Ceph|

所有这些天生都是“分布式的”。对一个系统而言，分布式意味着什么呢？它包含两个方面：

* 这个系统运行在多个服务器上。集群中的服务器数量差异极大，少则两三台，多则数千台。
* 这个系统管理着数据。因此，其本质上是一个“有状态”的系统。

当多台服务器参与到存储数据中，总有一些地方会出错。上述所有提及的系统都需要解决这些问题。在这些系统的实现中，解决这些问题时总有一些类似的解决方案。以通用的形式理解这些解决方案，有助于在更大的范围内理解这些系统的实现，也可以当做构建新系统的指导原则。

好，进入模式。

#### 模式

[模式](https://martinfowler.com/articles/writingPatterns.html)，这是 Christopher Alexander 引入的一个概念，现在在软件设计社区得到了广泛地接受，用以记录在构建软件系统所用的各种设计构造。模式提供一种“从问题到解决方案”的结构化方式，它可以在许多地方见到，并且得到了证明。使用模式的一种有意思的方式是，采用模式序列或模式语言的形式，将多个模式联系在一起，这为实现“整个”或完整的系统提供了指导方向。将分布式系统视为一系列模式是一种有价值的做法，可以获得关于其实现更多的洞见。

## 问题及可复用的解决方案

当数据要存储在多台服务器上时，有很多地方可能会出错。

### 进程崩溃

进程随时都会崩溃，无论是硬件故障，还是软件故障。进程崩溃的方式有许多种：

* 系统管理员进行常规维护时，进程可能会挂掉
* 因为磁盘已满，异常未能正确处理，做一些文件 IO 时进程可能被杀掉
* 在云环境中，甚至更加诡异，一些无关因素都会让服务器宕机

如果进程负责存储数据，底线是其设计要对存储在服务器上的数据给予一定的持久性（durability）保证。即便进程突然崩溃，所有已经通知用户成功存储的数据也要得以保障。依赖于其存储模式，不同的存储引擎有不同的的存储结构，从简单的哈希表到复杂的图存储。因为将数据写入磁盘是最耗时的过程，可能不是每个对存储的插入或更新都来得及写入到磁盘中。因此，大多数数据库都有内存存储结构，其只完成周期性的磁盘写入。这就增加了“进程突然崩溃丢失所有数据”的风险。

有一种叫[预写日志（Write-Ahead Log，简称 WAL）](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)的技术就是用来对付这种情况的。服务器存储将每个状态改变当做一个命令记录在硬盘上的一个只追加（append-only）的文件上。在文件上附加是一个非常快的操作，因此，它几乎对性能完全没有影响。这样一个可以顺序追加的日志可以存储每次更新。在服务器启动时，日志可以回放，以重建内存状态。

这就保证了持久性。即便服务器突然崩溃再重启，数据也不会丢失。但是，客户端在服务器恢复之前是不能获取或存储数据的。因此，在服务器失效时，我们缺乏了可用性。

一个显而易见的解决方案是将数据存储在多台服务器上。因此，我们可以将 WAL 复制到多台服务器上。

一旦有了多台服务器，就需要考虑更多的失效场景了。

### 网络延迟

在 TCP/IP 协议栈中，跨网络传输消息的延迟并没有一个上限。由于网络负载，这个值差异会很大。比如，由于触发了一个大数据的任务，一个 1Gbps 的网络也可能会被吞噬，网络缓冲被填满，一些消息到达服务器的延迟可能会变得任意长。

在一个典型的数据中心里，服务器并排放在机架上，多个机架通过顶端的机架交换机连接在一起。可能还会有一棵交换机树，将数据中心的一部分同另外的部分连接在一起。在某些情况下，一组服务器可以彼此通信，却与另一组服务器断开连接。这种情况称为网络分区。服务器通过网络进行通信时，一个基本的问题就是，知道一个特定的服务器何时失效。

这里有两个问题要解决。

* 一个特定的服务器不能为了知晓其它服务器是否已崩溃而无限期地等待。
* 不应该出现两组服务器，彼此都认为对方已经失效，因此，继续为不同的客户端提供服务，这种情况称为脑裂。

要解决第一个问题，每个服务器都要以固定的间隔像其它服务器发送一个[心跳（HeartBeat）](https://martinfowler.com/articles/patterns-of-distributed-systems/heartbeat.html)消息。如果心跳丢失，那台服务器就会被认为是崩溃了。心跳间隔要足够小，确保检测到服务器失效并不需要花太长的时间。正如我们下面将要看到的那样，在最糟糕的情况下，服务器可能已经重新启动，集群作为一个整体依然认为这台服务器还在失效中，这样才能确保提供给客户端的服务不会就此中断。

第二个问题是脑裂。一旦产生脑裂，两组服务器就会独立接受更新，不同的客户端就会读写不同的数据，脑裂即便解决了，这些冲突也不可能自动得到解决。

要解决脑裂问题，必须确保两组失联的服务器不能独立地前进。为了确保这一点，服务器采取的每个动作，只有经过大多数服务器的确认之后，才能认为是成功的。如果服务器无法获得多数确认，就不能提供必要的服务。某些客户端可能无法获得服务，但服务器集群总能保持一致的状态。占多数的服务器的数量称为 [Quorum](https://martinfowler.com/articles/patterns-of-distributed-systems/quorum.html)。如何确定 Quorum 呢？这取决于集群所容忍的失效数量。如果有一个 5 个节点的集群，Quorum 就应该是 3。总的来说，如果想容忍 f 个失效，集群的规模就应该是 2f + 1。

Quorum 保证了我们拥有足够的数据副本，以拯救一些服务器的失效。但这不足以给客户端以强一致性保证。比如，一个客户在 Quorum 上发起了一个写操作，但该操作只在一台服务器上成功了。Quorum 上其它服务器依旧是原有的值。当客户端从这个 Quorum 上读取数据时，如果有最新值的服务器可用，它得到的就可能是最新的值。但是，如果客户端开始读取这个值时，有最新值的服务器不可用，它得到的就可能是一个原有的值了。为了避免这种情况，就需要有人追踪是否 Quorum 对于特定的操作达成了一致，只有那些在所有服务器上都可用的值才会发送给客户端。在这种场景下，会使用[领导者和追随者（Leader and Followers）](https://martinfowler.com/articles/patterns-of-distributed-systems/leader-follower.html)。领导者控制和协调在追随者上的复制。由领导者决定什么样的变化对于客户端是可见的。[高水位标记（High Water Mark）](https://martinfowler.com/articles/patterns-of-distributed-systems/high-watermark.html)用于追踪在 WAL 上的项是否已经成功复制到 Quorum 的追随者上。所有达到高水位标记的条目就会对客户端可见。领导者还会将高水位标记传播给追随者。因此，当领导者出现失效时，某个追随者就会成为新的领导者，所以，从客户端的角度看，是不会出现不一致的。

### 进程暂停

但这并非全部，即便有了 Quorum、领导者以及追随者，还有一个诡异的问题需要解决。领导者进程可能会随意地暂停。进程的暂停原因有很多。对于支持垃圾回收的语言来说，可能存在长时间的垃圾回收暂停。如果领导者有一次长时间的垃圾回收暂停，追随者就可能会失联，领导者在暂停结束之后，会不断地给追随者发消息。与此同时，因为追随者无法收到来自领导者的心跳，它们可能会重新选出一个领导者，以便接收来自客户端的更新。如果原有领导者的请求还是正常处理，它们可能会改写掉一些更新。因此，需要有一个机制检测请求是否是来自过期的领导者。[世代时钟（Generation Clock）](https://martinfowler.com/articles/patterns-of-distributed-systems/generation.html)就是用于标记和检测请求是否来自原有领导者的一种方式。世代，就是一个单调递增的数字。

### 不同步的时钟和定序问题

检测消息是来自原有的领导者还是新的领导者，这个问题是一个维护消息顺序的问题。一种显而易见的解决方案是，采用系统时间戳为一组消息定序，但是，我们不能这么做。主要原因在于，跨服务器使用系统时钟是无法保证同步的。计算机时钟的时间是由石英晶体管理，并依据晶体震荡来测量时间。

这种机制非常容易出错，因为晶体震荡可能会快，也可能会慢，因此，不同的服务器会产生不同的时间。跨服务器同步时钟，可以使用一种称为 NTP 的服务。这个服务周期性地去查看一组全局的时间服务器，然后，据此调整计算时钟。

因为这种情况出现在网络通信上，正如我们前面讨论过的那样，网络延迟可能会有很大差异，由于网络原因，时钟同步可能会造成延迟。这就会造成服务器时钟彼此偏移，经过 NTP 同步之后，甚至会出现时间倒退。因为这些计算机时钟存在的问题，时间通常是不能用于对事件定序。取而代之的是，可以使用一种简单的技术，称为 Lamport 时间戳。[世代时钟](https://martinfowler.com/articles/patterns-of-distributed-systems/generation.html)就是其中一种。

## 综合运用——一个分布式系统示例

理解这些模式有什么用呢？接下来，我们就从头构建一个完整的系统，看看这些理解会怎样帮助我们。下面我们以一个共识系统为例。分布式共识，是分布式系统实现的一个特例，它给予了我们强一致性的保证。在常见的企业级系统中，这方面的典型例子是，[Zookeeper](https://zookeeper.apache.org/)、[etcd](https://etcd.io/) 和 [Consul](https://www.consul.io/)。它们实现了诸如 [zab](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html#sc_atomicBroadcast) 和 [Raft](https://raft.github.io/) 之类的共识算法，提供了复制和强一致性。还有其它一些流行的算法实现共识机制，比如[Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))，[Google Chubby](https://research.google/pubs/pub27897/)  把这种算法用在了锁服务、视图戳复制和[虚拟同步（virtual-synchrony）](https://www.cs.cornell.edu/ken/History.pdf)上。简单来说，共识就是指，一组服务器就存储数据达成一致，以决定哪个数据要存储起来，什么时候数据对于客户端可见。

### 实现共识的模式序列

共识实现使用[状态机复制（state machine replication）](https://en.wikipedia.org/wiki/State_machine_replication)，以达到对于失效的容忍。在状态机复制的过程中，类似于键值存储这样的存储服务是在所有服务器上进行复制，用户输入是在每个服务器上以相同的顺序执行。做到这一点，一个关键的实现技术就是在所有服务器上复制[预写日志（WAL）](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)，这样就有了可复制的 WAL。

我们按照下面的方式可以将模式放在一起去实现可复制的 WAL。

为了提供持久性的保证，要使用[预写日志（WAL）](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)。使用[分段日志（Segmented Log）](https://martinfowler.com/articles/patterns-of-distributed-systems/log-segmentation.html)可以将与写日志分成多个段。这么有助于实现日志的清理，通常这会采用[低水位标记（Low-Water Mark）](https://martinfowler.com/articles/patterns-of-distributed-systems/low-watermark.html)进行处理。通过将预写日志复制到多个服务器上，失效容忍性就得到了保障。在服务器间复制由[领导者和追随者（Leader and Followers）](https://martinfowler.com/articles/patterns-of-distributed-systems/leader-follower.html)保障。[Quorum](https://martinfowler.com/articles/patterns-of-distributed-systems/quorum.html) 用于更新[高水位标记（High Water Mark）](https://martinfowler.com/articles/patterns-of-distributed-systems/high-watermark.html)，以决定哪些值对客户端可见。所有的请求都严格按照顺序进行处理，这可以通过[单一更新队列（Singular Update Queue）](https://martinfowler.com/articles/patterns-of-distributed-systems/singular-update-queue.html)实现。领导者发送请求给追随者时，使用[单一 Socket 通道（Single Socket Channel）](https://martinfowler.com/articles/patterns-of-distributed-systems/single-socket-channel.html)就可以保证顺序。要在单一 Socket 通道上优化吞吐和延迟，可以使用[请求管道（Request Pipeline）](https://martinfowler.com/articles/patterns-of-distributed-systems/request-pipeline.html)。追随者通过接受来自领导者的[心跳（HeartBeat）](https://martinfowler.com/articles/patterns-of-distributed-systems/heartbeat.html)以确定领导者的可用性。如果领导者因为网络分区的原因，临时在集群中失联，可以使用[世代时钟（Generation Clock）](https://martinfowler.com/articles/patterns-of-distributed-systems/generation.html)检测出来。

![分布式系统模式](../image/patterns-of-distributed-system.svg)

这样，以通用的形式理解问题以及其可复用的解决方案，有助于理解整个系统的构造块。

## 下一步

分布式系统是一个巨大的话题。这里涵盖的这套模式只是其中的一小部分，它们覆盖了不同的主题，展示了一个模式能够如何帮助我们理解和设计分布式问题。我将持续在添加更多的模式，包括分布式系统中解决的下列主题：

* 成员分组以及失效检测
* 分区
* 复制与一致性
* 存储
* 处理

