Session Follower 

Session Follower 负责通过心跳循环维持 Session 的状态，它工作在独立的线程中，支持多种方式指定 Meta 中的 Session Master 位置：

1. Leader Loader，目前默认使用的方式，Loader 通过 ZK 集群获取 Meta Server Leader 地址，并会在 Leader 地址更新时同步更新 Lib Meta 中地址；
2. ZK Host，通过指定的 service name 直接从 ZK 中获取 Master 地址；
3. Master Address，直接指定 Master 地址；

当 Session 创建时，Meta Server 将返回 Session 有效时间（lease_interval_ns，相对时间，即仅代表 Lease 有效时间长度，而不是有效时间戳） 。Session Follower 记录后， 将进入无限 Keepalive 循环，每轮循环处理逻辑如下（对应代码 SessionFollower::KeepAliveLoop()）：

1. 检查 Session 是否已经异常（根据最新一次完成心跳的时间戳与当前时间戳比较确定）:
   1. Jeopardy，lease 已经失效，但是失效时长尚未超过 kJeopardyIntervalNS = 30s，上报事件继续处理；
   2. Expired，lease 失效超过 kJeopardyIntervalNS = 30s，退出循环，上报 Expire 事件，删除当前 Session，等待 Session 重建；
2. 如果之前出现了异常，将等待一段时间后，尝试重新连接 Master。重连将触发 Master 地址重新更新（如果使用 Master Leader Loader 模式）。等待的时间与当前重试次数有关，重试次数越大，等待时间越久。但最少是 kSessionConnectRetryIntervalMinMs = 1.5s，最长不超过 kSessionConnectRetryIntervalMaxMs = 6s。重连成功并且 Session 依然活跃则继续处理，否则回到步骤 1；
3. 通过 Session Client 向 Session Master 发送 Keepalive 信息，Session Client 的 timeout 会考虑当前 Lease 的存活时间，保证 timeout 之前 lease 是有效的：
   1. Master 回复正常，则更新本地的 Lease，继续步骤 1；
   2. 连接失败则重标记需要重连，继续步骤 1；
   3. Master 回复 Session 过期，则退出循环，上报 Expire 事件，删除当前 Session，等待 Sesion 重建；
   4. Master 回复 并非 Leader，标记重连，由重连过程更新 Leader 地址，进入步骤 1；
   5. Master 回复 Epoch 不匹配，则触发 Master Change 事件，继续步骤 1；

Session Follower 除在重连时可能做等待之外，心跳循环中均为直接处理没有等待，心跳的间隔周期由 Meta Leader 上的 Session Master 控制返回 Keepalive 请求的时机来实现，一般的周期是每 kReplyLeaseLeftNS = 5s 一次心跳，不过如果 Meta 刚重启或有立即下发 recover cmd 的需求，也会直接触发心跳。涉及代码包括：

1. Follower 中调用 AccessHandler::HandleKeepAlive() 来解析 response 中蕴含的 AccessKeepAliveResponse 部分的 recover/migrate/revoke lease/maintenance/clean chunk info/revoke client/volume update/config update cmd，ChunkKeepAliveResponse 部分的 gc cmd；
2. Follower 中调用 SessionMaster::KeepAlive() 来从 Session Master 那获取心跳结果，即一些待执行的命令放在 response 中。

Session Master 向 Session Follower 下发命令包含 2 种模式：（也可以理解成 Meta Leader 向 Access 下发）

1. 需要确认 Access 收到。例如 Revoke，iSCSI 的配置变更等同步 RPC。Meta 在发出命令后，一定会等待包含对应命令的 Keepalive 请求收到就Access 回应之后才返回命令（依赖心跳交互中 Session 的 Session Epoch 单调递增变化进行确认，例如下发命令时的 session epoch 是 1，当收到 access 回应的 epoch >=1 时即可代表之前产生的所有命令均已经正常接收）；
2. 无需确认。例如 GC，Recover 等，仅需要放入 Session 的通知队列中下发接口，Meta 中产生命令的逻辑不会等待

> 我理解这两种模式区别在同步/异步上

在目前版本中，Access 实现的均是无状态推送，即便是确认推送才放回结果的模式，Meta 也不感知 Access 对命令的执行结果，仅确保送达。依赖这个模式的业务逻辑需要各自采用其他机制确保正确执行。

心跳是 Meta 给 Access 传递控制指令的过程，而 Chunk 状态信息（Extent 状态、Chunk 网络状态、Chunk 上活跃外部连接状态等）由于信息过于庞大，因此不在心跳中上报，而是剥离出来使用独立的固定间隔循环上报状态数据。（HDFS 中也是这么做的）







### session 机制

#### 背景

在 ZBS 中，有一类服务是需要和另外一类服务进行周期性地信息交换，进行资源锁定等。例如 Meta 服务的 Leader 需要管理 Chunk 服务的 Session 状态，向 Chunk 发布命令，实现 VExtent Lease 的分配等。

#### 定义

* SessionMaster：一个集群中只有一个 SessionMaster，在 Meta Leader 所在的节点。SessionMaster 是发出 revoke/recover 指令，并做中央决策的服务进程。

* SessionFollower：每台 Access Server 都是一个 SessionFollower ，需要周期性地向 SessionMaster 发送心跳更新状态，并且响应 SessionMaster 的命令；

SessionMaster 和 SessionFollower 之间通过 Session 连接，Session 一般有正常和过期状态，当 SessionFollower 和 SessionMaster 断开一段时间后，Session 会转为过期状态。对于任何一个 Session， Master 端发现 Session 断开的时间点晚于 Follower 发现 Session 断开的时间点。在分布式系统中，Master 通常需要在确认 Follower 的 Session 过期后，才能处理接下来的工作。

#### Session Master

SessionMaster通过事件的方式通知使用者，这些事件包括：

1. Session created

2. Session expired

3. KeepAlive request arrived

Master 可以通过 1 和 2 发现 Membership 的变化，通过 3 来自定义 keepalive response，从而 piggyback 命令。同时 SessionMaster 提供 NotifyFollower 的接口，SessionMaster 可以通过此接口向 SessionFollower立刻发送命令。

#### Session Follower

SessionFollower 通过事件的方式来通知使用者，这些事件包括：

1. Session Expired
2. KeepAliveResponse arrived

通过 1，follower 可以发现自己的 Session 丢失，进行相应的处理；通过 2，follower 可以处理自定义的 keepalive 消息；同时 SessionFollower 提供如下两个接口：

1. Start：开始创建 Session；
2. Stop：主动通知 Master 自己将离开，使得 Master 可以更快的侦测到 Session 的消失

#### Jeopardy Event

Master 失败后，重启到运行状态通常需要 <30s 的时间，但是一个 lease 的时间通常只有 12s，如果 Master发生 Failover，那么所有的 follower 的 Session 都会过期，这些过期对于 retry 间隔长的应用（例如 NFS Client）会造成较长时间的 hang。而这种情况下，follower 的 session 不必要过期，因为 master failover 后，将会恢复所有的 session。为了避免不必要的过期，参考 chubby 的实现，引入 Jeopardy 事件，当这个事件发生后，follower 再通知上层，和 master 已经失联，session 很可能会过期。在 Jeopardy 事件和Session expired 事件之间，follower 会不断和 master 重连，直到确认 session expired 或重连成功。Jeopardy 的时间间隔设定为 30s，即 Master Failover 的最长允许时间。

[Session机制](https://docs.google.com/document/d/1S_j7mtCCWT9x7Il6RmAp89fPhWHTQTaL-Iva3NtGtjQ/edit)





心跳机制 

access follower master handler 三个 keepalive 函数，跟 system_design 中的心跳服务对比

具体判断 session 过期的情况

https://docs.google.com/document/d/186FwmbmWvrT5xdsCHWCYvQrD_l6Oy-uVKoi3AGepsFM/edit#heading=h.4583j0aepbw3