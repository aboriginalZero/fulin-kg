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