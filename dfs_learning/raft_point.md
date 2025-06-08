把 block 例子看懂之后，根据 runLeader 等拆解 braft 中的实现





总结一下领导选举的部分，然后找宇珂问一下论文中的疑惑，看源码实现，再结合 geektime 评论和面试题



领导选举

raft 支持的成员身份即节点状态共有 3 种，leader、candidate、follower。

Leader 

* 向所有其他节点发送空的 AppendEntries RPC（心跳）来避免 follower 成为 candidate 产生不必要的选举；
* 当发现 last log index 大于 follower i 的 nextIndex，就往 follower i 发送 leader 从 nextIndex[i] 开始的日志条目；
* 接受所有客户端的读写请求，将客户端指令转化为日志条目并添加到本地日志，当这个条目应用到 leader 状态机后，leader 才向客户端返回应答；
* 如果收到更高任期 leader 的心跳，转到 follower 状态；
* 每次 AppendEntries RPC 之后更新 commitIndex、nextIndex 数组、matchIndex 数组

Candidate

* 选举流程包括自增 currentTerm、自投、重置选举超时时间、发送 RequestVote RPC 到所有其他节点，如果收到大多数节点的投票，转到 leader 状态；
* 如果收到 leader 的心跳，转到 follower 状态；
* 如果由于投票决裂等问题导致选举超时，发起新一轮选举。

Follower

* 不主动发起请求，只会向 leader 和 candidate 发送 RPC response；
* 如果等待 leader 心跳超时，转到 candidate 状态；

All Server

* 如果 commitIndex 大于 lastApplied，增加 lastApplied，应用 index 等于 lastApplied 的日志到状态机上；
* 如果收到任意一个 RPC 请求的 term 字段大于 currentTerm，那么让 currentTerm 等于 term 并将自身状态转到 follower；



日志复制

理想的日志复制阶段，日志完全一致

1. Leader 接收到客户端请求后，基于客户端请求中的指令创建一个新日志项附加到本地日志中；
2. Leader 通过 AppendEntries RPC 将该日志项复制到其他的节点；
3. 当大多数 follower 复制成功返回 RPC response 给 Leader 时，Leader 将这条日志项提交到它的状态机中，为这些 follower 更新 nextIndex 和 matchIndex 字段；
4. Leader 更新 commitIndex，具体规则是如果存在大多数 follower 对应的 matchIndex 大于等于 N，第 N 项日志的任期等于 currentTerm，N 大于 commitIndex，那么让 commitIndex 等于 N；
5. Leader 将执行的结果返回给客户端；
6. 当 follower 接收到空/新的 AppendEntries RPC 时，根据 leaderCommit 字段（leader 的 commitIndex）判断如果 Leader 已经提交了某条日志项而它还没有提交，那么 follower 就将这条日志项提交到本地的状态机中。

由于进程崩溃、服务器宕机等问题导致日志不一致

1. Leader 通过 AppendEntries RPC 的一致性检查，具体是根据 prevLogIndex 和 PreLogTerm 字段从后往前找到 follower 上与自己相同日志项的最大索引值；
2. Leader 强制 follower 覆盖不一致的日志项之后，为这些 follower 更新 nextIndex 和 matchIndex 字段。



成员变更

成员变更的问题，主要在于进行成员变更时，可能存在新旧配置的 2 个“大多数”，导致集群中同时出现两个领导者，破坏了 Raft 的领导者的唯一性原则，影响了集群的稳定运行。

联合共识除了 Logcabin 没有其他常见 raft 算法实现

单节点变更

假设我们有一个由节点 A、B、C 组成的 Raft 集群，现在我们需要增加数据副本数，增加 2 个副本（也就是增加 2 台服务器），扩展为由节点 A、B、C、D、E， 5 个节点组成的新集群：

集群配置从 [A, B, C] 到 [A, B, C, D]：

1. 领导者（节点 A）向新节点（节点 D）同步数据；
2. 领导者（节点 A）将新配置[A, B, C, D]作为一个日志项，复制到新配置中所有节点（节点 A、B、C、D）上，然后将新配置的日志项应用（Apply）到本地状态机，完成单节点变更。

集群配置从 [A, B, C, D] 到 [A, B, C, D, E]：

1. 领导者（节点 A）向新节点（节点 E）同步数据；
2. 领导者（节点 A）将新配置[A, B, C, D, E]作为一个日志项，复制到新配置中的所有节点（A、B、C、D、E）上，然后再将新配置的日志项应用到本地状态机，完成单节点变更。

在正常情况下，不管旧的集群配置是怎么组成的，旧配置的“大多数”和新配置的“大多数”都会有一个节点是重叠的。 也就是说，不会同时存在旧配置和新配置 2 个“大多数”。

在分区错误、节点故障等情况下，如果我们并发执行单节点变更，那么就可能出现一次单节点变更尚未完成，新的单节点变更又在执行，导致集群出现 2 个领导者的情况。如果你遇到这种情况，可以在领导者启动时，创建一个 NO_OP 日志项（也就是空日志项），只有当领导者将 NO_OP 日志项应用后，再执行成员变更请求。

