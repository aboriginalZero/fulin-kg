5 节点集群，172.20.137.141



对于 pid src dst 有限定规则，外部 rpc 触发的 recover/migrate 优先级应该要更高，插到队列第一条，

recover/migrate 要分开讨论，migrate 要额外指定 replace chunk

做一次冲突检查，合法且和当前恢复不冲突，prefer local 和 topo 相关的不管



允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。

输入 pid, src, dst, replace

输出 pid, current loc, active loc, dst, owner, mode

搜索 AddRecoverCmdUnlock

1. AddMigrateCmd rpc 参考 RecoverManager::MakeMigrateCmd() 和 AddSpecialRecoverCmd() 的就行，再加些判断条件；

   RecoverManager::AddMigrateCmd() zbs-reader

2. AddRecoverCmd 参考 AddToWaitingRecover() 和 AddSpecialRecoverCmd() 写法。

// 人工指定后，后续还是有可能会被系统后台程序再次迁移回去

```shell
zbs-meta migrate create pid <pid> src_chunk <cid> dst_chunk <cid> replaced_chunk <cid>
```



非首次运行单测

```shell
# 首次编译需要进到 Docker 内部执行
docker run --rm --privileged=true -it -v /home/code/zbs3:/zbs -w /zbs registry.smtx.io/zbs/zbs-buildtime:el7-x86_64
mkdir build && cd build
source /opt/rh/devtoolset-7/enable
cmake -G Ninja ..
ninja zbs_test

# 屏幕中会提示出错处的日志信息，借助 newci 可以避免在本地配置 nvmf/rdma 环境跑单测
# 但是要配好 nvmf/rdma 的相关依赖包/服务
cd /home/code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"

# 运行后的测试日志默认保存在 /var/log/zbs/zbs_test.xxx 中 
cd /home/code/zbs/build/src && ./zbs_test --gtest_filter="*FunctionalTest.WriteResize*"

# 显示指定 main 分支
git review main

# 自动修改格式后再编译
docker run --rm --privileged=true -v /home/code/zbs:/zbs -w /zbs registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 sh -c 'sh ./script/format.sh && cd build && ninja zbs_test'
```





集群内部扫描副本状态，副本迁移相关代码

RecoverManager::DoScan()，正常情况下 60s 检查一次，也可以立即扫描如接受 rpc 请求

RecoverManager::ReGenerateWaitingMigrateList()，分别处理均匀分配的副本和有本地化偏好的副本，他们会有不同的初次分配和迁移策略

RecoverManager::ReGenerateMigrateForRebalance()，针对有本地化偏好的副本

1. 当处于低负载时 

   RecoverManager::ReGenerateMigrateForLocalizeInStoragePool()

   RecoverManager::RepairPExtentForLocalization()，满足如下任一条件不触发迁移：1. 副本的 prefer local cid 不是健康状态（status == CHUNK_STATUS_CONNECTED_HEALTHY）；2. 这个 pid 存在临时副本； 3. replace cid 和 dst cid 都是 0；4. dst_cid/src_cid 上被下发的 recover 命令超过 200 条；5. src_cid 上的有限空间不足 256 MB；

   否则通过 RecoverManager::GetRepairCid() 选取 replace_cid 和 dst_cid，初始值为 0，具体规则如下：

   1. 如果 prefer local 有该副本，并且其他活跃副本满足本地化/局部化的分配策略，说明副本满足分配规则，不触发迁移；
   2. dst_cid 的选取规则是优先选择没有副本且不处于 failsow 的 prefer local，次选符合期望分布（即符合 LocalizedComparator 分配策略） 的下一个副本；
   3. replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid； 

   选定 dst_cid 和 replaced_cid 后，src cid 的选择规则是默认值为 replace cid，但如果 replace_cid slowfail 或者和 dst_cid 不在同一个可用域，那么会从活跃副本中找跟 dst_cid 在同一个可用域且不是 slow fail 的 cid 作为 src_cid。

2. 当处于中高负载时

   1. 如果集群处于极高负载时，进行容量再均衡扫描

      RecoverManager::ReGenerateMigrateForBalanceInStoragePool()

   2. 否则进行拓扑安全扫描

      RecoverManager::RepairTopoInMediumHighLoad()

      RecoverManager::RepairPextentTopo()

      1. 非双活集群

         RecoverManager::RepairInZone() 

         RecoverManager::GetSrcAndReplace()

      2. 双活集群

         RecoverManager::RepairInStretched()

         RecoverManager::MoveToOtherZone()

         RecoverManager::GetSrcAndReplace()


经过以上步骤，生成的 Recover cmd 只是放入 passive_waiting_migrate 中，在等待 60s 或者 scan_recover_immediate = true 时会 swap(active_waiting_migrate, passive_waiting_migrate) 

关于 active_waiting_migrate / passive_waiting_migrate 这两个链表：

1. 只有 RecoverManager::GenerateMigrateCmds() / RevokeRecoverCmds() 会往 active_waiting_migrate 中 erase 元素；
2. 只有 RecoverManager::MakeMigrateCmd() 会往 passive_waiting_migrate 中 push 元素，调用 MakeMigrateCmd() 的有：
    1. RecoverManager::RepairPExtentForLocalization()，这是在集群处于低负载时的拓扑安全扫描；
    2. RecoverManager::RepairPextentTopo()，这是在集群处于中、高负载时的拓扑安全扫描；
    3. RecoverManager::DoMove()，这是在集群处于中、高负载时的容量再均衡扫描；
    4. RecoverManager::ReGenerateMigrateForRemovingChunk()，针对要退出的 Chunk 上的所有 pid 做迁移。

关于 active_waiting_recover / passive_waiting_recover 这两个链表：

1. 只有 RecoverManager::GenerateRecoverCmds() / RevokeRecoverCmds() 会往 active_waiting_recover 中 erase 元素；
2. 只有 RecoverManager::AddToWaitingRecoverIfNecessary() 会往 passive_waiting_recover 中 insert 元素，调用 AddToWaitingRecoverIfNecessary() 的有：
    1. RecoverManager::AddToWaitingRecover(pid_t pid)，recover 给定的 pid ，在 MetaRpcServer::DoRemoveReplica() 指定 pid 时会构建临时副本，并触发这个 pid 的 recover；
    2. RecoverManager::ReGenerateWaitingRecoverList()，recover pextent table 中所有需要 recover 的 pid，在 DoScan 中被定时执行；



RecoverManager::DoScan()，正常情况下 60s 检查一次，也可以接受 rpc 请求去立即扫描

在 Scan 的过程中：

1. 通过 RecoverManager::GenerateRecoverCmds() 为 active_waiting_recover 队列中元素构造 recover cmd；
2. 通过 RecoverManager::GenerateMigrateCmds() 为 active_waiting_migrate 队列中的元素构造 migrate cmd；
3. 调用 RecoverManager::AddRecoverCmdUnlock() 做 recover 相关检查；
4. 调用 AccessManager::EnqueueRecoverCmd() 生成 recover cmd 并放入对应 session 命令队列中；

另一种调用 RecoverManager::AddRecoverCmdUnlock() 的方式是 RecoverManager::AddSpecialRecoverCmd() ，接受通过 rpc 的方式来调用。

RecoverManager::AddRecoverCmdUnlock()

1. 若这个 pid 上有 recover cmd，返回 false（每个 pid 上任一时刻只能有 1 条 recover cmd）；
2. 若这是一条临时副本的 recover cmd，且 src_tmp_replica 的正确版本（通过对比 epoch 确定）是否已经在 pid pextent 上，返回 false；
3. 若 src 中没有 pid 或者 migrate dst 已经有 pid，返回 false；
4. 确保 dst 上有足够的空间（能否再放一个 extent，区别对待 thin/thick），更新 dst 的 rx_pids、replaced 的 tx_pids、src 的 recover_src_pids 以及他们的占用空间；
5. 调用 AccessManager::EnqueueRecoverCmd() 生成 recover cmd 并放入对应命令队列中
    1. 通过 AccessManager::AllocOwnerForRecover() 分配 recover/migrate 的 lease Owner，与 AccessManager::AllocOwner() 由用户 IO 触发的 Owner Alloc 逻辑不同，分配的优先级是 1. 该 pid 已有的 lease owner；2. src_cid；3. dst_cid；4. 从非 slow_cids 中根据 [owner priority](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.2hivgtf3odem) 选一个 cid；
    2. 若此时 lease owner 跟 src_cid 不同，跟 dst_cid 不同，且 lease owner 上有活跃副本（说明它是健康的），为了让 recover/migrate 的读走本地而非网络，会把 recover cmd 的 src 修改成 lease owner；
    3. 根据待恢复/迁移副本的 pid 和经过 1 2 步选出的 lease owner，构造 lease 并放入 recover cmd 中，接着将 recover cmd 放入 lease owner 的那个 recover cmd 队列（Access Manager 为每个 session 维护了一个  recover cmd 队列，通过 lease owner 的 uuid 获取）。



migrate 和 recover 只是共用 RecoverCmd 这个数据结构，各自的命令队列（recover 是 std::set，migrate 是  std::list）、触发时机、同时触发的命令数都是不同的。


IO 下发的流程

NFS/iSCSI/nvmf -> ZBS Client -> access io handler -> generation syncor -> recover handler

疑惑

1. FOREACH_SAFE 和 FOREACH 的区别？怎么体现 safe 了
2. access handler 中为啥都以事件回调的形式来注册 Session Handler 的相关接口
1. ZBS 中的一个 extent，读的同时不允许写？为啥不用多版本机制来管理（块存储覆盖写的原因？还是块存储没必要提供）
2. 代码中 [(zbs.labels).as_str = true] 的意思，slack 中 qiuping 提过，
3. gtest 如何开启 VLOG DLOG 部分的日志
4. meta in zbs 中 Volume 部分 origin_id 中示例继承树怎么看？
5. metaDB 中的 Vtable 表存储 Volume -> Extent 的关联关系
6. 对着 log 梳理一下 zbs 架构，角色在什么位置，哪些要持久化到 metaDB，关键数据结构的类型
7. CreateSession 这个过程 SessionMaster 做什么了
7. 生产代码应该少的出现断言，避免引起不必要的 coredump

zbs cli 如何快速查看集群负载情况？用 zbs-meta chunk list 看每个节点的负载然后自己手动算

Meta 与 chunk 如何交互？问题来源[减少数据恢复量](https://docs.google.com/document/d/1rDN0bNa-Dw6xo9yCN_gtVg1qrrDx1LVzc5_Dz_23dW8/edit#)

1. 为什么需要 Extent Lease

2. 为什么第一次拿到 Lease 之后需要 Sync Generation

3. 为什么 Meta 需要保存一份 generation，什么时候会使用和更新这个 generation

4. ZBS 如何保证多个副本之间的一致性

   1. Meta 如何分配副本

   2. ZBS 如何进行写操作

   3. 当有副本发生错误时，ZBS 如何保证数据安全

   4. Meta 中记录的副本和 LSM 中的本地副本之间的关系

   5. Meta 中记录的 location 和 alive_location 的含义

      前者是 pextent table  中记录的，后者是跟随心跳上报动态变动

   6. 当 snapshot 和 clone volume 的时候 Meta 和 LSM 的元数据发生了什么

   7. Sync Gen 失败的副本是否必须立即从 meta 中剔除

   8. 写失败的副本是否必须立即从 meta 中剔除

```shell
# 在主分支上
git pull
# 将新的 URL 复制到本地配置中
git submodule sync --recursive
# 从新 URL 更新子模块
git submodule update --init --recursive
```

开发机重启后，zk 要手动重启，selinux 要关闭，zkServer.sh start、setenforce 0

ZBS RPM 和 SMTX ZBS/SMTX OS/IOMesh 等不同产品的产品版本号从 5.0.0 开始已经分离，各有各的版本号



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




IO 流

Access 提供的是外部客户端进入 ZBS 系统内的接入点功能。在数据请求达到 Access 后，它将负责把它转化为 ZBS 内部请求。处理的基本过程如下：

1. 直接接入的 Access 如果之前从未处理过对应数据块（Extent） 的数据请求，则会首先向 Meta 请求 Extent 的基本信息（副本分布，权限）。如果 Access 最近已经访问过（1 小时内），则无需这个步骤；

2. 在获得数据访问权限（如果本地的 Access 不持有 Extent 的访问权限，则会转发至持有权限的 Access 继续后续步骤）和基本信息之后：

   1. 读请求：本地 LSM 在副本列表中，Access 会优先访问本地的 LSM （无需经过网络，本地的内存交换即可），如果成功读取则直接访问，失败则继续尝试逐一其余副本直至成功或者全部失败；

   2. 写请求，并发的向所有副本发出写命令，确认所有副本均写入成功才返回，如果部分失败，则执行副本剔除，保证副本列表中的所有副本数据一致，如果全部失败，则不剔除，返回异常等待上层重试；

      TODO 这部分代码可以看一下，是收集到所有副本的写结果后才下发指令还是只要收到任意一个副本写成功就可以？


FunctionalTest::SetUp()  --> new MiniCluster(kNumChunks);





gtest系列之事件机制

“事件” 本质是框架给你提供了一个机会, 让你能在这样的几个机会来执行你自己定制的代码, 来给测试用例准备/清理数据。gtest提供了多种事件机制，总结一下gtest的事件一共有三种：
1、TestSuite事件
需要写一个类，继承testing::Test，然后实现两个静态方法：SetUpTestCase方法在第一个TestCase之前执行；TearDownTestCase方法在最后一个TestCase之后执行。
2、TestCase事件
是挂在每个案例执行前后的，需要实现的是SetUp方法和TearDown方法。SetUp方法在每个TestCase之前执行；TearDown方法在每个TestCase之后执行。
3、全局事件
要实现全局事件，必须写一个类，继承testing::Environment类，实现里面的SetUp和TearDown方法。SetUp方法在所有案例执行前执行；TearDown方法在所有案例执行后执行。
例如全局事件可以按照下列方式来使用：
除了要继承testing::Environment类，还要定义一个该全局环境的一个对象并将该对象添加到全局环境测试中去
原文链接：https://blog.csdn.net/ONEDAY_789/article/details/76718463



C++ 中 map 嵌套使用，vector 添加一个 vector 中所有元素 https://zhuanlan.zhihu.com/p/121071760

protobuf 中 optional/repeated/ 等用法，https://blog.csdn.net/liuxiao723846/article/details/105564742

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610



centos7 中 pip 默认是 8.x，升级到 20.3.4（更高版本不支持 Python 2.7）

```
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
```

pip 根据 whl 文件安装包

```
pip install xx.whl
```

git_review 插件安装 1.28.0，进 pypi.org 找包的各个历史版本

```
pip install git-review==1.28.0
```

pip 更改镜像源

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



