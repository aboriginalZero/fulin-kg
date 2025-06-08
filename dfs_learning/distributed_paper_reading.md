#### [GFS] 2003 SOSP The Google File System

GFS 是一个分布式文件存储系统。考虑到 GFS 主要是写少读多的场景，如适配 MapReduce 的话，写中间结果，后续归并的时候只会读。因此选用追加写，追加写依赖 OS 的原子写文件，不需要额外的锁同步，比覆盖写更轻量级。

* 在 GFS 中，并行覆盖写成功，所有的客户端看到同样的数据，但不一定是正确的数据。比如先写字符串 abc 再写 def，期望结果是 def，但如果 abc 第一次写失败，def 写成功，接着第一次写重试成功，最终的结果会是 abc。

* 在 GFS 中，并行追加写成功，由于 OS 的原子写文件操作，能够保证客户端要求写入的数据在每个副本中至少会出现一次，但是副本之间并不是完全一致的，例如副本 1 可能是 abc def，而副本 2 是 abc def def（之后可以通过操作 id 来过滤想要的数据版本）。

GFS 和 ZBS 的区别

1. GFS 是追加写。ZBS 是覆盖写，因为应用场景不同， ZBS 是通用块存储， GFS 写少读多。
2. GFS 没有逻辑的 Access。ZBS 中有Access，作为 IO 总控单独解耦，会让架构更加清晰。
3. generation 机制。GFS 在 master 和 chunk 签订 lease 的时候 generation++， 但 ZBS 是在 write 的时候才++。
3. GFS 如果写失败，会直接返回写失败，用客户端来决定是否重试，而 ZBS 有 剔除 + 重试 操作。

#### [BigTable] 2006 OSDI Bigtable: A distributed Storage System for Structured Data

BigTable 是一个用来管理结构化数据的分布式存储系统，可以看成是一个稀疏的、分布式的、可持久化的、多维的、有序的哈希表。其对应的映射关系是 (row, colume_family:qualifier, time) -> string

BigTable 是一个 NoSQL 数据库，并不支持完整的关系数据模型，仅支持单行事务

LevelDB 的有序性有利于相同域名的页面放在一起，www.baidu.com，与 www.taobao.com 都在 .com 域名下，并且可以做前缀压缩。

BigTable 根据行键的字典序组织行数据，动态地对 row range 进行切分，每个 row range 称为一个 tablet，这是数据分区和动态负载的基本单位。BigTable 中通过时间戳索引存储一个数据的多个版本



Tablet 的分配跟 LevelDB 有点像？LevelDB 就是单机版的 BitTable

https://spongecaptain.cool/post/paper/bigtable/#3-bigable-%E7%9A%84%E6%9E%B6%E6%9E%84

https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf

https://hardcore.feishu.cn/docs/doccnY21HwJO8LckMKEGMTzvm2g	



HBase 是 Hadoop 社区对 Bigtable 的开源实现

#### [Dynamo] 2007 SOSP Dynamo: Amazon's Highly Available Key-value Store

Dynamo 是一个用来管理键值对的分布式存储系统。

数据处理

* 数据分区（data partition）策略是一致性哈希 + 虚拟节点。每个节点在本地维护能够将请求直接路由到合适节点的路由信息。

* 数据复制（data replication）策略是每个 key，除了哈希计算出来的节点存一份外，还要在环上顺时针方向的 N - 1 个节点存一份。

* 数据版本化（data versioning）策略是允许系统中同时存在多个不同版本，Dynamo 优先保证可用性而非一致性，这种方式会导致数据冲突，需要检测并解决冲突。
    * 何时解决冲突？Dynamo 的做法是保证写永远不会被拒绝，在读操作时解决冲突。
    * 谁来解决冲突？一般来说是让应用自主选择对用户体验最好的冲突解决算法，例如购物车可以选择“合并”冲突版本，返回一个合并后的购物车。如果应用将解决冲突下放给数据仓库，那么数据仓库可以使用一些简单策略如最后一次写有效来解决更新冲突。

故障处理

* 短时故障处理：基于 hinted handoff 保证在节点或网络发生短时故障时读和写操作不会失败。
* 持久故障处理：基于 merkle tree 的 anti-entropy

成员管理

基于 gossip 算法通告成员变动情况，只要网络是连通的，任意一个节点就可以把消息散播到全网。消息扩散速度是 logN（这个 gossip 看着像是一个 BFS 遍历）

这几种策略，在 geektime 课中有介绍

http://blog.xiyoulinux.com/blog/104104788



#### [[zab] 2008 LADIS A simple totally ordered broadcast protocol](https://www.datadoghq.com/pdf/zab.totally-ordered-broadcast-protocol.2008.pdf)



#### [zk] 2010 USENIX ATC ZooKeeper: Wait-free Coordination for Internet-scale Systems



#### [[raft] 2014 USENIX ATC In Search of an Understandable Consensus Algorithm (Extended Version)](https://pages.cs.wisc.edu/~remzi/Classes/739/Spring2004/Papers/raft.pdf)



### 指导

按照从理论到实践的顺序，将经典的分布式系统论文分成了分布式理论基础、分布式一致性算法、分布式数据结构和分布式系统实战四类。

分布式理论基础

分布式理论基础部分的论文，主要从宏观的角度介绍分布式系统中最为基本的问题，从理论上证明分布式系统的不确定、不完美，以及相互间的制约条件。研读这部分论文，你可以了解经典的 CAP 定理、BASE 理论、拜占庭将军问题的由来及其底层原理。有了这些理论基础，你就可以明白分布式系统复杂的根源。当再碰到一些疑难杂症，其他人不得其解时，你可以从理论高度上指明方向。以下就是分布式理论基础部分的论文：

* Time, Clocks, and the Ordering of Events in a Distributed System
* The Byzantine Generals Problem
* Brewer’s Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services
* CAP Twelve Years Later: How the “Rules” Have Changed
* BASE: An Acid Alternative
* A Simple Totally Ordered Broadcast Protocol
* Virtual Time and Global States of Distributed Systems

分布式一致性算法

只要脱离了单机系统，就会存在多机之间不一致的问题。因此，分布式一致性算法，就成了分布式系统的基石。在分布式一致性算法这一部分，我将与你推荐 2PC、Paxos、Raft 和 ZAB 等最知名的一致性算法。分布式算法的复杂度比普通算法要高出几个数量级，所以这部分论文是最为烧脑的一部分。搞明白这部分论文，你的空间想象力和统筹规划能力都会得到质的提升。

* A Brief History of Consensus, 2PC and Transaction Commit
* Paxos Made Simple
* Paxos Made Practical
* Paxos Made Live: An Engineering Perspective
* Raft: In Search of an Understandable Consensus Algorithm
* ZooKeeper: Wait-Free Coordination for Internet-Scale Systems
* Using Paxos to Build a Scalable, Consistent, and Highly Available Datastore
* Impossibility of Distributed Consensus With One Faulty Process
* Consensus in the Presence of Partial Synchrony

分布式数据结构

分布式数据结构部分的论文，将与你介绍管理分布式存储问题的知名数据结构原理。通过它们，你可以构建自己的分布式系统应用。这部分论文的涵盖范围大致包括两部分：一是，分布式哈希的四个著名算法 Chord、Pastry、CAN 和 Kademlia；二是，Ceph 中使用的 CRUSH、LSM-Tree 和 Tango 算法。和分布式一致性算法类似，分布式数据结构也极其考验空间想象力和统筹规划能力。不过，在经过分布式一致性算法的锻炼后，相信这些对你来说已经不再是问题了。

* Chord: A Scalable Peer-to-Peer Lookup Service for Internet Applications
* Pastry: Scalable, Distributed Object Location, and Routing for Large-Scale Peer-to-Peer Systems
* Kademlia: A Peer-to-Peer Information System Based on the XOR Metric
* A Scalable Content-Addressable Network
* Ceph: A Scalable, High-Performance Distributed File System
* The Log-Structured-Merge-Tree
* HBase: A NoSQL Database
* Tango: Distributed Data Structure over a Shared Log

分布式系统实战

分布式系统实战部分的论文，将介绍大量互联网公司在分布式领域的实践、系统的架构，以及经验教训。Google 的新老三驾马车，Facebook、Twitter、LinkedIn、微软、亚马逊等大公司的知名系统都会在这一部分登场。你将会领会到这些全球最大规模的分布式系统是如何设计、如何实现的，以及它们在工程上又碰到了哪些挑战。

* The Google File System
* BigTable: A Distributed Storage System for Structured Data
* The Chubby Lock Service for Loosely-Coupled Distributed Systems
* Finding a Needle in Haystack: Facebook’s Photo Storage
* Windows Azure Storage: A Highly Available Cloud Storage Service with Strong Consistency
* Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing
* Scaling Distributed Machine Learning with the Parameter Server
* Dremel: Interactive Analysis of Web-Scale Datasets
* Pregel: A System for Large-Scale Graph Processing
* Spanner: Google’s Globally-Distributed Database
* Dynamo: Amazon’s Highly Available Key-value Store
* S4: Distributed Stream Computing Platform
* Storm @Twitter
* Large-scale Cluster Management at Google with Borg
* F1 - The Fault-Tolerant Distributed RDBMS Supporting Google’s Ad Business
* Cassandra: A Decentralized Structured Storage System
* MegaStore: Providing Scalable, Highly Available Storage for Interactive Services
* Dapper, a Large-Scale Distributed Systems Tracing Infrast
* Kafka: A distributed Messaging System for Log ProcessingAmazon 
* Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases

如何高效地阅读论文？

一般来说，单篇论文大概会有 15 到 20 页的内容，如果你是第一次读论文可以把重点放在前面的背景介绍、相关工作和概要设计上。好的论文通常会很仔细地介绍背景知识，帮助你从宏观上先对整个问题有一个初步认识，了解当前现状。接下来，你可以再根据自己的兴趣，选择是否仔细阅读论文涉及的详细原理和设计。这一部分，通常是论文中最精华的部分，包含了最具创新的理念和做法，内容通常也会比较长，需要花费较多的时间和精力去研究。这时，你可以根据自己的情况，选择一批论文重点突破。论文最后通常是评测和数据展示部分。这部分内容对我们最大的参考价值在于，学习作者的评测方法、用到的测试工具和测试样例，以便将其运用到工作中。



VLDB 与 SIGMOD、ICDE 并称为全球数据库三大学术顶会