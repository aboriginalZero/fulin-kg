1. 领导选举流程、选举限制、什么时候发起选举、什么节点才可以做领导

2. 日志复制流程、日志结构及特点、日志不正常情况、如何保证日志的正常复制、网络分区下数据一致性怎么解决、raft 一致性怎么实现、

3. 如何保证安全性

4. 日志压缩流程、快照结构

5. 成员变更、联合共识流程、单节点变更流程

6. prevote 机制应对的场景、具体流程

    假设某个Follower节点出现网络分区，由于接收不到Leader的心跳包，所以会不断选举，Term会一直增加。加入原集群后会把原Leader降级为Follower，导致重新选举。但实际上它并不能成为Leader(没有最新日志)，造成disruption。

    Prevote 把选举也拆成一个两阶段提交的方式，先进行一轮prevote，这一轮prevote并不会更改自己的Term＋１，但是Vote请求里的Term是＋１的。如果收到大多数的赞成票，那么发起真正选举。否则继续等待下一轮。

7. Lease 机制应对的场景、具体流程

8. 节点故障后为何不重置为Candidate，而是Follower

9. 多个节点同时选举能否成功？ 选举时不首先投给自己呢？

10. Ｍaster在准备Commit某条Log的时候挂了，Raft如何保证一致性？

11. 假设Ａ, B, C三节点，A为Leader。A和B之间出现分区，但是A, C和B, C之间连接不受影响，会导致Leader频繁切换，有什么方法优化？

12. Raft协议的leader选举，正常情况下，网络抖动造成follower发起leader选举，且该follower的Term比现有leader高，集群中所有结点的日志信息当前一致，这种情况下会选举成功吗？

13. 假设硬件或软件错误破坏了 Leader 为某个特定 Follower 存储的 nextIndex 值。这是否会影响系统的安全？请简要解释你的答案

14. 假设你实现了 Raft，并将它部署在同一个数据中心的所有服务器上。现在假设你要将系统部署到分布在世界各地的不同数据中心的每台服务器，与单数据中心版本相比，多数据中心的 Raft 需要做哪些更改？为什么？

15. 每个 Follower 都在其磁盘上存储了 3 个信息：当前任期（currentTerm）、最近的投票（votedFor）、以及所有接受的日志记录（log[]）。未对算法做任何修改的前提下，假设 Follower 崩溃了，并且当它重启时，它最近的投票信息已丢失。该 Follower 重新加入集群是否安全？另一假设崩溃期间 Follower 的日志被截断（truncated）了，日志丢失了最后的一些记录。该 Follower 重新加入集群是否安全？

16. 即使其它服务器认为 Leader 崩溃并选出了新的 Leader 后，（老的）Leader 依然可能继续运行。新的 Leader 将与集群中的多数派联系并更新它们的任期，因此，老的 Leader 将在与多数派中的任何一台服务器通信后立即下台。然而，与此期间，它也可以继续充当 Leader，并向尚未被新 Leader 联系到的 Follower 发出请求；此外，客户端可以继续向老的 Leader 发送请求。我们知道，在选举结束后，老的 Leader 不能提交（commit）任何新的日志记录，因为这样做需要联系选举多数派中的至少一台服务器。但是，老的 Leader 是否有可能执行一个成功 AppendEntries RPC，从而完成在选举开始前收到的旧日志记录的提交？如果可以，请解释这种情况是如何发生的，并讨论这是否会给 Raft 协议带来问题。如果不能发生这种情况，请说明原因。

17. 在配置变更过程中，如果当前 Leader 不在 C-new 中，一旦 C-new 的日志记录被提及，它就会下台。然而，这意味着有一段时间，Leader 不属于它所领导的集群（Leader 上存储的当前配置条目是 C-new，它不包括 Leader）。假设修改算法，如果 C-new 不包含 Leader，则使 Leader 在其日志存储了 C-new 时就立即下台。这种方法可能发生的最坏情况是什么？

18. Raft 如何在 leader 变化的一瞬间也能保持一致性

19. raft 日志覆盖和回溯的关键点

20. raft 算法有什么不足

21. raft 出现双主时能对外提供服务吗

17. Raft选主过程、脑裂问题如何解决、网络分区恢复后发生什么。节点重启后如何恢复数据、日志如何记录、日志压缩。分片控制器的作用、是否是单节点部署、迁移分片数据是同步还是异步。从节点能否处理读请求。单点负荷怎么解决。一致性有哪些





先按答案来，再按这些题目去源码中验证，这个过程加深印象

https://www.jianshu.com/p/b28e73eefa88

https://blog.csdn.net/daaikuaichuan/article/details/98627822

https://juejin.cn/post/6902221596765716488

https://zhuanlan.zhihu.com/p/187506841

源码解读

https://zhuanlan.zhihu.com/p/169840204

https://zhuanlan.zhihu.com/p/169904153

https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md