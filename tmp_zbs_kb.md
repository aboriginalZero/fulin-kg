> 待整理的 zbs 零碎知识

VLOG(VLOG_INFO) 级别的日志怎么开启，zbs-meta vlog -h 的用法

```shell
# 在 meta 配置文件里加上
GLOG_vmodule="tcp=4, data_channel=4"
```

如果是 chunk 的 vlog，需要在 chunk 配置文件里加上吗？有了这个之后，可以通过 zbs-meta vlog -h 来不重启的状态下调节。

 zbs-meta memory heap_profiler_start /root/yiwu/heap_profiler && zbs-meta memory heap_profiler_stop 收集到的文件如何使用

```
# 在对应 docker （oe1 或 el7）里安装 pprof
./pprof_linux_amd64 -h
# 测试环境的话，可以直接在节点上 wget http://192.168.91.19/bin/x86_64/pprof_linux_amd64
./pprof_linux_amd64 -text /usr/sbin/zbs-metad heap_profiler.0002.heap
```

查看集群空间占用

```
zbs-tool space show 
```

可以通过在目标节点上的 /var/log/message 里搜索 Abort Task 来检查是否有物理磁盘 IO 异常的信息。如果有则通常是磁盘损坏或者磁盘固件版本不对，如果多个磁盘均有出现则有可能是 HBA 卡异常或者固件版本不对。

```shell
# 磁盘消除"隔离中"状态
zbs-chunk partition set-healthy /dev/sdx
# 磁盘消除"亚健康/不健康"状态
zbs-node set_disk_healthy sdx
```





### 疑惑

1. access handler 中为啥都以事件回调的形式来注册 Session Handler 的相关接口
2. ZBS 中的一个 extent，读的同时不允许写？为啥不用多版本机制来管理（块存储覆盖写的原因？还是块存储没必要提供）
3. gtest 如何开启 VLOG DLOG 部分的日志
4. CreateSession 这个过程 SessionMaster 做什么

zbs cli 如何快速查看集群负载情况？用 zbs-meta chunk list 看每个节点的负载然后自己手动算

Meta 与 chunk 如何交互？问题来源[减少数据恢复量](https://docs.google.com/document/d/1rDN0bNa-Dw6xo9yCN_gtVg1qrrDx1LVzc5_Dz_23dW8/edit#)

1. 为什么需要 Extent Lease

2. 为什么第一次拿到 Lease 之后需要 Sync Generation

3. 为什么 Meta 需要保存一份 generation，什么时候会使用和更新这个 generation

4. ZBS 如何保证多个副本之间的一致性

    1. Meta 如何分配副本

        Allocation

    2. ZBS 如何进行写操作

    3. 当有副本发生错误时，ZBS 如何保证数据安全

    4. Meta 中记录的副本和 LSM 中的本地副本之间的关系

    5. Meta 中记录的 location 和 alive_location 的含义

        前者是 pextent table  中记录的，后者是跟随心跳上报动态变动

    6. 当 snapshot 和 clone volume 的时候 Meta 和 LSM 的元数据发生了什么

    7. Sync Gen 失败的副本是否必须立即从 meta 中剔除

    8. 写失败的副本是否必须立即从 meta 中剔除

### iscsi io 流

1. submitter_receiver.cc 中的 eventfd_read / eventfd_write 使用
2. zbs client proxy 和 v2 的区别

一次写操作

一直走到 zbs client 侧

ZbsClient::DoIO --> ZbsClientProxyV2::DoIO --> IOReceiver::HandleIO（消费队列元素） --> ZbsClientProxyV2::IOSplit （放入一个 polling 队列中，之后被塞到 chunk 主线程） --> ZbsClientProxyV2::IOSubmit --> zbs_aio_write --> blockdev_zbs_writev -->  _blockdev_zbs_submit_request（在这区分 bdev io type，有 read / write / unmap / flush / reset / abort / CAW 等） --> blockdev_zbs_submit_request （注册在 zbs_fn_table 上） --> __submit_request --> spdk_bdev_io_submit --> spdk_bdev_writev --> spdk_bdev_scsi_readwrite --> spdk_bdev_scsi_process_block --> spdk_bdev_scsi_execute --> spdk_scsi_lun_execute_tasks --> spdk_scsi_dev_queue_task --> spdk_iscsi_queue_task --> spdk_iscsi_op_scsi / spdk_iscsi_op_data --> spdk_iscsi_execute --> spdk_iscsi_conn_handle_incoming_pdus --> spdk_iscsi_conn_execute --> spdk_iscsi_conn_full_feature_do_work --> spdk_iscsi_conn_full_feature_migrate （在这起了一个 timer 循环执行 do_work） 

初始化过程

spdk_iscsi_conn_full_feature_migrate --> spdk_iscsi_conn_login_do_work --> spdk_iscsi_conn_construct --> spdk_iscsi_portal_accept --> spdk_iscsi_portal_grp_open --> spdk_iscsi_portal_grp_open_all --> spdk_iscsi_setup --> ISCSIServer::SetupPortal --> ISCSIServer::UpdateConfig --> SetupChunkServerCtx（到了 zbs chunk 侧了）

ISCSIServer 在这执行注册多个回调，如 UpdateLuns / RefreshConfig / ListConnection。

iSCSI initiator 和 ZBS Chunk server 进行交互，iSCSI 配置信息要在 Chunk 上落地才算真正生效。

1. ZbsClientProxyV2::DoIO
2. ZbsClient::Write / Unmap
3. ZbsClient::DoIO
4. ZbsClient::SplitIO
5. ZbsClient::ProcessIO
6. ZbsClient::SubmitIO
7. InternalIOClient::SubmitIO
    1. InternalIOClient::DoLocalIO
        1. AccessIOHandler::WriteVExtent
    2. InternalIOClient::DoRemoteIO
        1. DataChannelClient::WriteVExtent
        2. DataChannelClient::Impl::WriteVExtent
        3. AccessIOHandler::SubmitWriteVExtent，Access IO Handler 中注册了该回调
        4. AccessIOHandler::WriteVExtent
8. AccessIOHandler::WriteVExtent
9. access io handler
10. replica io handler

### PExtent / Chunk 状态

chunk 视角的 PExtentStatus

```cpp
enum PExtentStatus {
  PEXTENT_STATUS_INIT = 99,
  PEXTENT_STATUS_INVALID = 0,
  PEXTENT_STATUS_ALLOCATED = 1,
  PEXTENT_STATUS_RECOVERING = 2,
  PEXTENT_STATUS_OFFLINE = 3,
  PEXTENT_STATUS_CORRUPT = 4,
  PEXTENT_STATUS_IOERROR = 5,
  PEXTENT_STATUS_UMOUNTING = 6
};
```

meta 视角的 PExtentStatus

```c++
enum PExtentStatus {
  PEXTENT_HEALTHY = 0,
  
  // 不是 staging/garbage 且写过真实数据且当前时刻活跃副本数为 0
  PEXTENT_DEAD = 1,
  
  // 不是 staging/garbage 且写过真实数据且当前时刻副本数为 0
  PEXTENT_BROKEN = 2,
  
  // 只看 garbage_ 字段是否为 true，不管其他的
  PEXTENT_GARBAGE = 4,
	
  // 
  PEXTENT_NEED_RECOVER = 3,
  
  PEXTENT_MAY_RECOVER = 5
};
```

ChunkState，用以表示 Chunk 视角的是否能够正常运行

```cpp
enum ChunkState {
  // 默认状态，当 Chunk 确定所属的 Storage Pool 后，就会进入 IN_USE 
  // 而 Chunk 在新加入集群时一定从属于某个 Storage Pool，所以这个状态存在时间很短
  CHUNK_STATE_UNKNOWN = 0;
  
  // 只有 Idle 状态的 Chunk 才允许加入新的 SP 或是从 ZBS 集群中退出，其不属于任何 SP
  // Meta 仅仅定期探测 Chunk 状态，不会再向 Chunk 分配任何 Extent
  CHUNK_STATE_IDLE = 1;
  
  // 只有这个阶段的 Chunk 可以被正常分配数据
  CHUNK_STATE_IN_USE = 2;
  
  // 当操作 Chunk 从 Storage Pool 中退出时， Chunk 将从 Inuse 切换至 Removing 
  // Meta 不会再向 Removing 状态的节点分配新的数据，其上的 extent 会被迁移到其他 Chunk
  // 当 Removing 状态下的 Chunk 已经没有任何 Extent 时，将把 Chunk 置为 Idle 状态
  CHUNK_STATE_REMOVING = 3;
}
```

ChunkStatus，用以表示 meta 感知的每个 Chunk 的连接状态

```cpp
enum ChunkStatus {
  // Chunk 加入集群/重新启动后的初始状态
  // Meta Leader 刚刚启动，从未获取过任何的 Chunk 状态信息时展示的状态，
  // 或者 Chunk 已经和 Meta 建立连接但是本地尚未完成初始化工作无法提供存储服务的状态
  CHUNK_STATUS_INITIALIZING = 1,
  
  // Chunk 在本地完成所有功能初始化并正常之后，通过心跳上报，将由 Initializing 进入 Healthy 状态
  // Chunk 正常与 Meta 建立连接，并处于可正常提供服务的状态
  CHUNK_STATUS_CONNECTED_HEALTHY = 2,
  
  // Chunk 正常与 Meta 建立连接，但是本地 LSM 处于异常状态（通常原因是没有可用的 Journal 分区）
  // 此时 Meta 不会向 Chunk 分配新的 Extent ，但也不会立即触发数据迁移动作；
  CHUNK_STATUS_CONNECTED_ERROR = 3,
  
  // 这 2 个实际未使用
  CHUNK_STATUS_CONNECTED_WARNING = 4, 	
  CHUNK_STATUS_CONNECTING = 5,		
  
  // 当前的 Meta Leader 生命周期内曾经和 Chunk 建立过健康连接，但此时已经和 Chunk 失去连接
  // Chunk 与 Meta 失去连接，其上的数据副本将因为长期未更新存活状态而触发 Meta 的数据恢复动作
  CHUNK_STATUS_SESSION_EXPIRED = 6
};
```



### zbs 端口使用

| 服务名            | 使用网络           | 使用端口         | 备注                       |
| ----------------- | ------------------ | ---------------- | -------------------------- |
| zookeeper         | 存储网络           | 2181、2888、3888 |                            |
| prometheus        | 管理网络、存储网络 | 9090             | 用 http 而非 https 访问    |
| zbs-deploy-server | 管理网络           | 10403            |                            |
| zbs-rest-server   | 管理网络、存储网络 | 10402            |                            |
| zbs-inspector     | 存储网络           | 10700、10701     | zbs-insight 也在同一进程   |
| zbs-taskd         | 管理网络、存储网络 | 10600、10601     | task dispatcher and runner |
|                   |                    |                  |                            |
| zbs-metad         |                    | 10100            | meta rpc server            |
| zbs-metad         |                    | 10101            | meta status server         |
| zbs-metad         |                    | 10102            | meta sm server             |
| zbs-metad         | meta leader        | 10103            | meta access manager        |
| zbs-metad         |                    | 10104            | meta http server           |
|                   |                    | 10105            | meta grpc server           |
|                   |                    |                  |                            |
| zbs-chunkd        |                    | 10200            | chunk rpc server           |
| zbs-chunkd        |                    | 10201            | chunk data channel         |
| zbs-chunkd        |                    | 10202            | chunk http server          |
| zbs-chunkd        |                    | 10203            | chunk perf server          |
| zbs-chunkd        |                    | 10206            | meta proxy rpc service     |
| zbs-chunkd        |                    | 10207            | meta proxy status service  |
| zbs-chunkd        |                    | 10208            | chunk grpc server          |

### ZBS Chunk 磁盘

chunk 侧有慢盘标记，早在网络亚健康之前就有了，副本分配策略会考虑节点容量，这个容量是有效数据空间比例，看的是比例，慢盘不算有效数据空间，如果有慢盘，这个值会比较低，所以这个 chunk 被选中的优先级会低一些。

[磁盘异常处理  II](https://docs.google.com/document/d/1NdsdRCPmNciLC8Vj70HEImQKRxLAPIVEzvtTbKqIkZI/edit)，在平均延迟 >= 2s 时，TUNA 将磁盘记录为慢盘 II ，并向本地 Chunk 提示慢盘进入隔离状态。LSM 2 将响应磁盘隔离请求，隔离磁盘：

1. 对于进入隔离状态的磁盘，所有的普通 IO 请求将被拒绝，关键 IO 需要被接受；
2. 隔离的磁盘上关联的 Extent 都将被标记为异常状态以快速让 Meta 触发数据恢复 （需要确认是否可以标记所有有部分间接 Block 在磁盘上的 Extent，如果实现代价较大，可以不标记这些 Extent）；
3. 隔离的磁盘不再会被分配新的 Extent，并且 GC 命令可以正常的清理系统中对隔离磁盘的数据状态；特别的，强制 IO 触发的 LSM 内部数据分配，在没有健康磁盘可用的情况下，要允许分配至隔离中的磁盘（比如处理 COW，COW 需要的副本空间只会在本地）；
4. 在隔离的磁盘上不再具备数据后，隔离磁盘将自动从 Chunk 中移除；
5. 隔离磁盘的空间将不再被标记为有效空间，需要在向 Meta 上报的有效空间容量中扣减；

隔离状态的磁盘与直接标记为慢盘 I & 坏盘的标记不同，在必要时依然需要处理 IO 请求。对于慢盘 I & 坏盘 & 磁盘消失，可以认为 Chunk 不具备读取数据的能力，在当前阶段只能直接放弃该副本。但是对于慢盘 II，数据是还可以被访问的。

理论上的最优效果是，保留该副本，且外部 IO 行为几乎等同于该磁盘行为。理论上，通过重置 HBA 卡、重启节点的手段来恢复该磁盘，很大几率地，磁盘能重新恢复正常响应，但需要引入较为复杂且不可控的恢复手段。但是在作为最后一个副本时，提供 IO 能力有可能避免用户业务立即中断。因为 LSM 本身并不具备识别副本是否最后一个副本的能力，因此数据副本在 LSM 2 中隔离后需要被标记为异常，由 Access （知晓副本状态）提供 IO 标记，要求 LSM 响应此类 IO。

即当慢盘 II / 健康盘上都有同一个 pid 副本且所有的副本都写失败，才会去写慢盘 II，对应代码 GenerationSyncor::MarkPidIOHard()



### ZBS Lease

Access 持有的 Lease 包括三种类型：

1. Session Lease，代表了 Session 的存活状态。由 Keepalive 过程维系，Lease 到期之后 Session 进入异常状态，Access 不再提供 IO 服务。Lease 在心跳过程中会不断续期。只要心跳正常，Lease 就始终有效；
2. VExtent Lease， 代表着虚拟卷对应数据块的访问权限，通常由 ZBS Client 和 Access 尝试对 Volume 执行 IO 时向 Meta 申请，存储 vExtent Lease 时也会缓存对应的 Extent lease；
3. Extent Lease，代表着 ZBS 内部的数据对象 Extent 的访问权限，在 Recover/Migrate 等系统内部 IO 事件发生时会如果需要生成 Lease 会产生于独立于 VExtent Lease 的 Extent Lease。

Extent Lease 为每个失败的副本维护了 Staging Block Info（class CachedLease 中持有），自副本第一个失败 IO 起所写过的 Block 都将记录在 Staging Block Info 中。敏捷恢复时，根据 Staging Block Info，读取数据增量恢复到失败副本中 ，Sync 临时副本时也将从临时副本记录的数据增量构建出 Staging Block Info。

VExtent Lease 和 Extent Lease 虽然有很多共同的属性，也都代表着访问权限。但是本质上是两个不同视角和逻辑对象的访问权限。目前版本因为 ZBS Client 和 Access 运行在同一个进程中，他们共享了缓存 Lease 的 Libmeta 结构。因此在大部分场景下， Client 对 Volume 执行 IO 时申请了 VExtent Lease ，Access 就不再需要额外申请一次。当然 Client 访问 Extent Lease owner 未必一定在本地 Chunk 中，此时就会通过网络访问远端 Access。

持有 Lease 的 holder 应该在 yield 之后重新检查 lease，因为在 yield 期间有可能被其他 holder 清理或自身过期

Lease 原理上应该是一个独立的具备时间有效性的机制，出于实现方便地考虑，我们没有对每个 VExtent Lease 和 Extent Lease 做独立的基于时间的生命周期管理。而是将他们的生命周期与 Session Lease 绑定在一起，Session 存活即保证 Lease 存活，仅在必要时通过命令处理逻辑或者异常处理逻辑清理部分 Extent 的 Lease。

Meta Lease 的授予对象是 Session，因此一个 Access 在上一次 Session 生命周期内获得的 Lease 并不可以沿用到下一个 Session 生命周期内。这是因为两个 Session 交替期间内一定发生过集群失连的事件，Lease 在此期间很有可能已经由 Meta 授予给其他 Access 上的 Session。

Access Server 通过 Session 机制与 Meta 建立连接，经过 Session 确保数据访问权限（Extent Lease）的唯一性，提供接入协议转换功能。

Access 中 Session 状态维护机制包括 2 个部分：维持生命周期心跳循环的 Session Follower 与响应状态变化事件的 Session Handler。Session 也有自己的 Lease， 代表 Session 的健康状态，每次 Access 收到 Meta 的心跳回复都会延续自身的 Lease 。如果长时间没有收到正常的心跳回复就会触发状态变化。状态有 Init、KeepAlive、Jeopardy、Expired。

Access 在进行读/写 IO 前先进行一次 Sync Gen，确认所有副本当前的 Gen 是一致的。



ZBS 保持副本一致性的方式的是在整体设计中采用了单一接入的策略，包含数据链接上的单一接入点、Access 上实现的数据块粒度的权限控制、Generation 机制。

分布式一致性接入仅是解决用户层面的多点访问带来的数据冲突问题的必要非充分条件。例如 A 和 B 两个程序同时写一个本地文件，文件系统可以保证按序处理 IO 并且返回给 A 和 B 一致的结果，但是如果 A 和 B 没有互相协作，不具备感知使用的存储是一个共享存储的能力，则数据状态很有可能不是他们中任意一个预期的结果。多点接入的数据正确性保证，还需要依赖于应用程序本身支持共享存储，例如 Oracle Rack，SQL Server Cluster，OCFS，VMFS, Hyper-V CSV 等。

块存储系统需要业务方来控制下发的 IO 是有序的，自身并不保证同时下发的 IO 有序。在 ZBS Client 中，不额外的处理 IO 重叠的保序问题，即不会严格的按照收到的 IO 请求顺序下发 IO。因为对于通用的块存储系统收到的并发重叠 IO，A 和 B，语义上并不保证最终结果是 A 或 B 或者 AB 交叠。大部分应用端希望确定某个结果时，会在发送时做合并或者严格的遵循收到 A 成功之后再下发 B 的原则。这样 IO 都成功之后结果就一定是 B。



### ZBS Client

ZBS Client 的核心功能是处理来自用户的 IO 请求。是 ZBS 内部数据对象 Volume 的访问入口，同时也集成了 Libmeta 作为集群 RPC 的访问代理（meta.cc，包含 Lease 管理和集群 RPC 接口两部分功能）。

可能有多个 zbs client，但一定只会有一个 access，一个 lease owner 来保证接入点的唯一性

在 IO 过程中，ZBS Client 不处理副本、数据校验等逻辑，他的主要逻辑是将对一个 Volume 的访问请求转化为对 Extent 的访问请求。ZBS Client 通过 Meta 将 Volume 指定区域（offset + len）映射到对应的 Extent，并从 Meta 返回的 VExtent Lease 中获得 Extent  Lease owner 的位置信息并将 IO 转发过去。

因为 IO 请求的频率过大，每一次都访问 Meta 获取位置信息将对 Meta 产生过大的压力。所以 Libmeta 收到 Lease 后将在本地缓存。ZBS Client 与 Access 不同，与 Meta 之间不会有心跳维护 Lease 的有效性，即 Meta 处的 Lease 变化是不会在意 Client 是否知道的，这样 ZBS Client 缓存的 Lease 可能与 Meta 存在不一致的现象。但是这并不影响正确性，因为 ZBS Client 的 Lease 仅用于路由向 Access Lease owner，只要 Meta 维持了 Access 的 Lease 正确性，如果 ZBS Client 访问了老旧的 Lease Owner，Access 将返回一个 ENotOwner 类型的错误。ZBS Client 收到此类错误后将丢弃本地缓存的 Lease 再向 Meta 获取新的 Lease ，中间过程数据是不会产生任何变化的。

Libmeta 也会监控自身的 Lease 使用情况，在长期未使用时会释放 Lease 以回收内存。但是因为 ZBS Client 自身其实不是 Lease 的持有者（非 Access owner），所以它的清理动作只需要涉及清理自身的缓存即可，不需要通知 Meta 也同步释放（但是因为和 Access 共享 Libmeta，所以会看到清理动作）。

他有两种存在形态：

1. Internal Mode

    Access Server 中集成的 ZBS Client，特点是本地 Client 所在的节点会持有一个 Session，并且本地有 Access 可以处理 IO 请求。Client 自身可以访问到集群内部的服务，例如 Meta，ZK 等。在这个模式下，ZBS Client 会通过 Access 受到 Meta 的控制和管理。

    在 Internal 模式下，IO Client 会将 UIO 按照 Volume 的 stripe 配置将 UIO 拆分为若干 BIO。在发送 BIO 时也会查询 VExtent Lease，将 IO 转发给 Access Owner。如是本地将直接调用 Access 接口将任务交给 Access 队列，如是远端，则通过 Datachannel Client 发送 VExtent IO。

2. External Mode

    集群外部（Taskd 也使用 External 模式，因为他并不在 IO 路径上，可以认为是集群外部）访问 ZBS 集群时使用的 Client 使用的模式，此时仅能访问集群中的 Blockd 服务，它将代理 IO 请求和元数据访问请求。Client 无法直接访问 ZK 或 Meta，在这个模式下，ZBS Client 不会收到任何来自 Meta 主动的控制命令下发。

    在 External 模式下，因为并不在当前的 Client 真的处理 IO，所以不会做拆分，只是简单的将 UIO 转化为 BIO。在发送时，将使用 Datachannel Client 将 Volume IO 发送给随机的一个 Blockd Server（在配置中指定的一组 Blockd Server 中随机选择），在与 Blockd Server 建立连接并且之后都正常通信的情况下，Volume IO 会始终的发往固定的 Blockd Server。需要注意的是，这个模式下通常是以 SDK 形态供外部集成，此时我们并没有一个强的单一接入点限制保证，需要 SDK 的使用方自己处理可能存在的多点接入情况下引发的正确性问题。这是因为这个模式下并没有天然可获得的客户端身份信息（iSCSI 的 initiator IQN， NVMF 的 initiator NQN，vHost 的 Machine ID），需要 SDK 调用者主动注入。如果后续有确切的需求场景我们会增加对应的限制策略。





目前 ZBS 仅支持以多副本方式存储数据，适用于对读写延迟比较敏感的业务场景。EC 相比副本占用更少的存储空间，同时提供与副本同等甚至更高的容错能力，其代价是更新或者故障数据恢复的性能略差（注：写不一定比副本差，虽然需要多一次读，但数据量变少了。看最终实现的效果）。EC 非常适合归档、备份等数据量较大且更新很少的业务，也适用于对延迟不敏感而对带宽敏感的业务。需注意，EC 的目标是为了保证完整性而非正确性，只可以用于恢复丢失的数据，而无法修复位翻转等数据正确性问题。需要有其他机制对数据篡改进行检测，例如 CheckSum 检测，被篡改的数据可以视为丢失，再通过 EC 做数据恢复。

ZBS 副本机制采用 Lease Owner + Generation 方式实现数据一致性。1. 通过 Lease Owner，所有的 IO 都被顺序化，客户端的多读者多写者模型被转化为单读者单写者模型；2. 通过 Generation 判断多个副本是否一致，当一个副本成功写入一个 IO 后，Generation 自增，每个 IO 携带当前的 Generation，Chunk 只有在 Generation 匹配时才允许 IO 处理。

用户可见的副本，是 Meta 认可的 Replica中 Gen 最低的副本，Gen 高不代表它里面的数据有用。



### IO 流

Access 提供的是外部客户端进入 ZBS 系统内的接入点功能。在数据请求达到 Access 后，它将负责把它转化为 ZBS 内部请求。处理的基本过程如下：

1. 直接接入的 Access 如果之前从未处理过对应数据块（Extent） 的数据请求，则会首先向 Meta 请求 Extent 的基本信息（副本分布，权限）。如果 Access 最近已经访问过（1 小时内），则无需这个步骤；

2. 在获得数据访问权限（如果本地的 Access 不持有 Extent 的访问权限，则会转发至持有权限的 Access 继续后续步骤）和基本信息之后：

    1. 读请求：本地 LSM 在副本列表中，Access 会优先访问本地的 LSM （无需经过网络，本地的内存交换即可），如果成功读取则直接访问，失败则继续尝试逐一其余副本直至成功或者全部失败；

    2. 写请求，并发的向所有副本发出写命令，确认所有副本均写入成功才返回，如果部分失败，则执行副本剔除，保证副本列表中的所有副本数据一致，如果全部失败，则不剔除，返回异常等待上层重试；

        TODO 这部分代码可以看一下，是收集到所有副本的写结果后才下发指令还是只要收到任意一个副本写成功就可以？



### ELF 行为

用户操作虚拟机行为

创建空白虚拟机：

从模板创建虚拟机：不存在虚拟机，根据模板创建一个虚拟机；

从快照重建虚拟机：不存在虚拟机，根据快照创建一个虚拟机；

从虚拟机克隆虚拟机：等效为对虚拟机配置做快照，磁盘是直接调用的 ZBS volume 克隆来实现；

导入 OVF 创建虚拟机：根据 OVF 创建一个虚拟机；

从快照回滚虚拟机：存在虚拟机，回到快照状态；

从虚拟机克隆为虚拟机模板：虚拟机还保留着，同时生成一个不可变更的虚拟机模板；

从虚拟机转化为虚拟机模板：虚拟机没了，同时生成一个不可变更的虚拟机模板；

虚拟机快照不会被快照/克隆。

当一个虚拟卷快照被克隆 10 次或一个虚拟机转化为虚拟机模板时，对应的副本会均匀分配。

ELF 克隆虚拟机有 2 种：

* 快速克隆

    COW 出来的 pextent 与 origin 的 loc 是一样的

    有的用户也会习惯于将应用装在系统盘，因此克隆后应用所使用的数据盘并不一定是新建的

    我们的 VM 盘会被局部化，写操作造成的 COW 又集中在这个母盘所在的机器上，导致性能差。

    调用 metad 提供的 Copy

* 完全克隆

    对 src volume 创建临时快照，调用 zbs taskd 提供的 CopyVolume(src_cid, dst_cid, src_snapshot, dst_volume)，全量备份后，删除临时快照。

    https://docs.google.com/document/d/1HyJ7uJpT0K3zSoAFKEX_m2gKNRYDlzWTXNoLNFTY0ac/edit#heading=h.umizx61uu46q

