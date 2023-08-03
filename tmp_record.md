pid 3133 的 prefer local 全程都是 4 吗？

I0802 21:45:27.201884 ， I0802 23:16:11.647876，这 2 个时间内没有过任何 topo 变化吗？

低负载、非双活、没有 failslow、topo 结构不变的话，topo 安全终态是 4 2 5，我看不明白为啥他想要去变成 4 2 1，发起一个 5 -> 1 的 migrate cmd。

```shell
# 之后没有对应的 access manager 侧的日志，同批次的 pid 2969 成功把 cmd 放到心跳回响，不过是发给 c2
22060:I0802 23:16:11.647876 58510 recover_manager.cc:414] add recover cmd for pid: 3133 src: 5 dst: 1 replace: 5 owner: 5

# 在把 cmd 放到心跳回响之前，发生了 topo 变更，chunk5 被移到 topo 了
I0802 23:16:12.213858  9843 chunk_manager.cc:992] [UPDATE TOPO OBJ]: [REQUEST]: id: "36e273e2-3406-4de1-b648-31dacb01e0c7" new_parent_id: "topo", [RESPONSE]: ST:OK, type: NODE id: "36e273e2-3406-4de1-b648-31dacb01e0c7" parent_id: "topo" name: "chunk-5-36e273e2-3406-4de1-b648-31dacb01e0c7" create_time: 1690797099 position { row: 2 column: 2 } dimension { row: 1 column: 1 } ring_id: 2, [TIME]: 19379 us.

# topo 变更会让 chunk5 RevokeRecoverCmds，不论是在 passive/active_waiting_migrate_、pid_cmds_ 还是 session 的 waiting_recover_cmds，都会被清空，所以 pid 3133 的 cmd 没有被放入心跳回响。

# 隔了7s 有 3314 的 cmd，顺利下发，且 recover 成功，c5 主动向 meta leader 发替换副本的 rpc
22207:I0802 23:16:19.662549 58510 recover_manager.cc:414] add recover cmd for pid: 3134 src: 5 dst: 1 replace: 5 owner: 5
22405:I0802 23:16:20.750061 58511 access_manager.cc:1060] [RECOVER]: pid: 3134 lease { owner { uuid: "afe6d52a-94dd-4534-bacd-e1a130382bbf" ip: "10.0.134.135" num_ip: 2273705994 port: 10201 cid: 5 secondary_data_ip: "20.0.134.135" zone: "default" scvm_mode_host_data_ip: "" alive_sec: 27028 machine_uuid: "93506396-2f6c-11ee-903c-525400bdd182" } pid: 3134 location: 328196 origin_pid: 0 epoch: 4354 origin_epoch: 0 ever_exist: true meta_generation: 1 expected_replica_num: 3 thin_provision: true chunks { id: 4 data_ip: 2223374346 data_port: 10201 rpc_ip: 2223374346 rpc_port: 10200 zone_id: "default" } chunks { id: 2 data_ip: 2206597130 data_port: 10201 rpc_ip: 2206597130 rpc_port: 10200 zone_id: "default" } chunks { id: 5 data_ip: 2273705994 data_port: 10201 rpc_ip: 2273705994 rpc_port: 10200 zone_id: "default" } } dst_chunk: 1 replace_chunk: 5 src_chunk: 5 is_migrate: true epoch: 4354 active_location: 328196 start_ms: 203965757
22682:I0802 23:16:43.267257  9843 meta_rpc_server.cc:1987] [REPLACE REPLICA]: [REQUEST]: session: "afe6d52a-94dd-4534-bacd-e1a130382bbf" pid: 3134 src_chunk: 5 dst_chunk: 1 epoch: 4354 reset_location_to_dst: false reset_generation: 18446744073709551615, [RESPONSE]: ST:OK, pid: 3134 location: 66052 expected_replica_num: 3 ever_exist: true origin_pid: 0 epoch: 4354 origin_epoch: 0 generation: 1 preferred_cid: 4 thin_provision: true alive_location: 66052 allocated_space: 47972352, [TIME]: 56069 us.
```

触发条件：在生成 cmd 并放入 session waiting cmd list 到通过心跳下发之前，cmd owner 拓扑变更。



1. 首先的问题是他为什么要生成 5 -> 1 的 migrate cmd，之前我得到的信息一直是 topo 安全都没变，会有这个 migrate cmd 就很蹊跷，但是如果有 topo 变更，那是有可能会有 migrate cmd 的；
2. 观察到 meta leader 没有给 c5 下发这条 migrate cmd （否则会有 [RECOVER] 开头的 access manager 侧日志），但 5 的 session 一直是正常且没替换过的，这个地方有问题。调查发现，在生成 cmd 并放入 session waiting cmd list 到通过心跳下发之前，cmd owner 拓扑变更，所有 cmd 中如果有 src 或 dst = 这个变更的 chunk 的话都需要被 revoke，不论 cmd 是在 passive/active_waiting_migrate、pid_cmds 还是 session 的 waiting_recover_cmds，都会被清空，所以 pid 3133 的 cmd 没有被放入心跳回响。
3. 但当 cmd 被放入 session 的 waiting_recover_cmds 之前，需要确认 cmd 的 lease owner（通过 owner uuid 才知道往哪个 session 的 cmd 队列发），因为在 1 个小时前，pid 3133 的 lease owner 被 release 了，所以会新生成一个让 migrate src (c5) 做 owner 的 pid owner。
4. 第 2 步中只 revoke cmd，没有去 revoke lease owner，后续也没有他的 app io 或 reposition io，所以 lease owner 一直在那。 



1. 如果后续有 app io / reposition io 都会去释放 lease 吧？看看代码确认下。
2. 并且在这之后没有去主动写 pid 3133，而 pid 的副本又符合本地化策略，不会触发 migrate，所以这个 pid 的 lease owner 就一直保留着。



```shell
$ rpm -q zbs
zbs-4.0.13-rc6.0.release.git.g933c12831.el7.SMTX.HCI.x86_64

$ curl http://192.168.91.19/s/zbs-debuginfo-4.0.13-rc6.0.release.git.g933c12831.el7.SMTX.HCI.x86_64

http://192.168.17.20/repo/pub/smartxos/el7/smtx/x86_64/zbs-debuginfo-4.0.13-rc6.0.release.git.g933c12831.el7.SMTX.HCI.x86_64.rpm
http://192.168.31.215/mirror/centos/7/smartxos-smtx/x86_64/zbs-debuginfo-4.0.13-rc6.0.release.git.g933c12831.el7.SMTX.HCI.x86_64.rpm
http://192.168.91.19/zbs-debuginfo-4.0.13-rc6.0.release.git.g933c12831.el7.SMTX.HCI.x86_64.rpm
```







抽空写一个脚本，能够进 docker 之后把所有环境配好并在 docker 中运行单测

vscode 中用 vim 插件，这样可以按区域替换代码

一个遗留问题是，单测里面想要触发两次 recover cmd，怎么让 entry 的 GetLocation() 得到及时更新，试了 sleep(9) 不行，可能不止需要一个心跳周期，还有其他条件没触发。





为什么低负载下，replace_cid 要尽量跟 dst_cid 在一个域，考虑同时有 2 副本不符合拓扑安全的场景。

1. 如果已有副本和 dst 在不同 zone，replace 随便选

2. 如果已有副本和 dst 在同一个 zone，replace 也得选这个 zone 的，如果没有这个 zone 的，说明在这之前 3 个副本

只要 topo 不变，符合拓扑安全的副本位置也不变，LocalizedComparator 得到的排序结果是不变的，因此虽然选样本是连锁反应，但就算第 2 个副本位置不对，第 3 个副本位置还是正确的。

scan_extents_num 应该放在 ReGenerateMigrateForUnevenVolume 这个函数中去做统计，虽然实际上 all_pool 中就一个，且一次只会因为一种方式去 migrate(本地化/容量均衡)。

GenerateRecoverCmds 里面的 src 也可以有选择策略的，目前的写法太乱了。

recover manager 代码中有些地方用 `pextent_table_`，有些用 `context_->pextent_table`，或许可以统一起来

低负载没有区分双活，中高负载开始有区分



对于 cmp
1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false



ZBS-25666

一个是希望 prefe local 尽快能够变成正常节点

另一个是根据老的 prefer local 得到的符合 LocalizedComparator 的 chunk list 的第 3 个希望能跟 prefer local 在一个可用域上，（要么遇到 prefer local 不存在的情况，放宽一次限制的阈值，不只 3 个，要么要去修改 LocalizedComparator 的逻辑，让它的第 4 个选跟 prefer local 在同一个可用域的）

话说双活的选，为啥是先 2 个 prefer local zone 再 1 个 secondary zone？是不是该 1 ：1 ：1 的挑？



写 Volume

1. ALLOC PEXTENT，分配 pid 和预留对应的空间，此时也指明了副本位置；
2. SYNC GENERATION START，写操作都需要触发 Sync Generation 用以保证数据一致性；
3. SYNC GENERATION SUCCESS
4. SET PEXTENT EXISTENCE ，第一次 IO 成功 Access 会用这个接口上报 Extent 存活。

创建快照

1. REVOKE LEASE
2. CREATE SNAPSHOT

写被快照的 Volume

1. COW PEXTENT，此时已经指明了根据被快照的 pextent 分配的 new pid；
2. SYNC GENERATION START，对 new pid 做 sync gen；
3. ALLOC EXTENT，给 new pid 分配 pextent；
4. COW PARENT EXTENT SUCCESS，此时指明其 parant_id 为 pid
5. COW EXTENT，
6. COW PARENT EXTENT SUCCESS，没懂 lsm 这边为啥会执行两遍，因为 inode_id 是 256 KB 为粒度？
7. COW EXTENT，
8. SYNC GENERATION SUCCESS
9. SET PEXTENT EXISTENCE

再次创建快照

1. REVOKE LEASE，meta 通过心跳主动下发给所有 chunk，会指明 vtable id，各个 chunk 上的 libmeta 据此清理 cache lease，在这个期间，触发了 migrate，
2. CREATE SNAPSHOT，

migrate 的流程

1. Migrate for xxx，指明 migrate 触发原因；
2. Enqueue recover cmd for session : xxx，可以根据 xxx 往回找日志，可以在 ACCESS SESSION INITIALIZE 的地方得到 cid；
3. add recover cmd for pid: 8 src: 4 dst: 5 replace: 1 owner: 4，指明 recover src dst replace owner
4. RECOVER，pid: 8 lease { owner { uuid: "f147d05a-bf65-4efd-955b-e562ec7782be" ip: "127.0.0.1" num_ip: 16777343 port: 10451 cid: 4 secondary_data_ip: "127.0.0.1" zone: "default" scvm_mode_host_data_ip: "" alive_sec: 4 machine_uuid: "test_machine_3" } pid: 8 location: 66562 origin_pid: 4 epoch: 12 origin_epoch: 8 ever_exist: true meta_generation: 1 expected_replica_num: 3 thin_provision: true chunks { id: 2 data_ip: 16777343 data_port: 10351 rpc_ip: 16777343 rpc_port: 10350 zone_id: "default" } chunks { id: 4 data_ip: 16777343 data_port: 10451 rpc_ip: 16777343 rpc_port: 10450 zone_id: "default" } chunks { id: 1 data_ip: 16777343 data_port: 10301 rpc_ip: 16777343 rpc_port: 10300 zone_id: "default" } } dst_chunk: 5 replace_chunk: 1 src_chunk: 4 is_migrate: true epoch: 12 active_location: 66562 start_ms: 1161051811
5. Get recover notification: pid: 8
6. RECOVER，Setup for pid: 8
7. SYNC GENERATION START，
8. FAIL TO SYNC GENERATION FROM，cid: 1 pid: 8 st: EShutDown



pid 1 : 5 1 2

pid 2 : 1 3 4

pid 3 : 3 2 5

pid 4 : 4 2 1

pid 5 : 5 1 2

pid 6 : 1 3 4

pid 7 : 3 2 5

pid 8 : 4 2 1

触发 3 条 migrate cmd，说是按照 localization 的策略，这个地方也很奇怪。

pid: 5 src: 5 dst: 3 replace: 2 owner: 5

pid: 7 src: 3 dst: 4 replace: 5 owner: 3

pid: 8 src: 4 dst: 5 replace: 1 owner: 4



vtable 看起来好像也是从 ptable 中复制来的？MetaRpcServer::GetVTable

Vtable 的内容就是若干个 VExtent，里面只有 3 个字段，vextent_id，location，alive_location，第一个字段是 volume 的 offset 与 pextent 的对应关系，后两个字段就是对应 pextent 的 location 和 alive_location，

快照会将 VTable 复制一份

我们的 COW PEXTENT 的触发时机是 GetVExtentLease rpc，如果 chunk 那里 lease 没有 expire，也就不会调用 GetVExtentLease，

COW 是先 revoke，然后打快照，保证了快照后，extent 无法写入的语意，如果不 revoke lease，快照是可写的

清理 cache lease 的方式有 3 种，一个是通过 session id，一个是通过 vtable id，一个是 pid，详见 Meta::ClearCachedLease

ZBS 目前（ <= 5.4.0 ）的快照实现方式分为两部分， Meta 在元数据层更新 COW 标记。即 Volume V [pid = 1, location = [x] ] 制作快照 Snapshot S 时，Snapshot S 会完整的复制一份 V 的 vTable，同时 V 和 S 的 vTable 中所有的 vExtent 都会打上 COW 标记变为 [ pid = 1+, location = [x] ]。当 V 对产生写入请求时，触发一次 COW，V 的 vtable 变化为 [pid = 2 , location = [x] ]，此时 2 的 Parent 为 1 ，并且 Location 需要和 1 完全一致。当 IO 被发往 X 时， X 上的 LSM 会使用 COW 的方式构建 2 -> 1 的关联关系。此时如果是写入请求，LSM 会按需从 1 中获取数据块进行 COW。如果后续有读取请求，则在 2 没有自身独立数据的情况下会访问 1 曾经持有的数据（在 LSM 1 中是以逐级查找 Block bitmap 标记位的方式实现，LSM 2 中以复制 Pblob Table 的方式实现，本质是一样的）。





ChunkTableEntry 的 last_succeed_heartbeat_ms 字段没用上

连续快照的 allocated_data_space 不对劲

chunk.chunk_space_info.thin_used_data_space 包含这个 chunk 最近一次上报的 thin extent 总空间消耗，用于在 meta 切换后进行集群 used_data_space 计算，在该 chunk 未再次上报 used space 前也能显示较为合理的集群空间消耗，空间分配改进中引入的字段。



把 COW 生命周期了解一下

实现命令行

多个参数用一个 DB、考虑升级兼容性问题、reposition 跟 recover migrate 分开、额外添加 rpc 用法

1. 能够观察 recover 真正 IO 的数据量，block 粒度的（比如如果有敏捷恢复，这个 pextent 就不会恢复 256 MB）
2. 能够查看 generate/pending_recover 的数量
3. 能够查看 need_migrate 的数量



http://meta/leader_mgt_ip:9090 账号 prometheus密码 HC!r0cks 

记录一下 chunkspaceinfo 上几个字段的意思，参考跟 wangsai yutian 的聊天



[ZBS-13059](http://jira.smartx.com/browse/ZBS-13059) 恢复数据允许识别原有数据块的冷热属性

[ZBS-25386](http://jira.smartx.com/browse/ZBS-25386) 修复节点数据迁移速度过慢导致access层无法迁移成功的问题

[ZBS-24563](http://jira.smartx.com/browse/ZBS-24563) 缓存命中率下降，副本恢复任务并发度过高，导致恢复任务执行过慢

[ZBS-13041](http://jira.smartx.com/browse/ZBS-13041)允许在线调整恢复/迁移的命令数量

4 个 ticket 4 件事

1. 冷热数据识别（梳理 lsm 侧的需求给到 lsm 同学做，识别冷热数据）

   recover src 读的热数据要写到 recover dst 上的 cache，冷数据直接写到 recover dst 上的 partition，避免恢复导致的缓存击穿

   需要修改 data channel 中的 message 中 max_message_id 的语义，换成 usercode，然后就可以带上返回的数据是否冷热的 flag

   参考 patch ZBS-21288

2. recover 每台 chunk 上执行的并发度默认 32，根据 recover extent 完成情况向上向下调节（auto mode）

   并发度最小应该是 2 4，而不是 1，就 1 个通道的容错率太差了

   recover extent 完成情况应该是本地的，从 lsm 侧获取到信息。

   这里权衡 Chunk 的负载指标可能采用的有：1）Chunk LSM 是否 Busy，即 LSM 的 IsLSMBusy，根据当前 Journal 可用数量来判别。2）Chunk Node 的各项性能指标，包括 cpu/memory/disk/network，见 node_monitor，但是现在的实现貌似只有全局的指标，例如 cpu 是所有的核心 summary，disk 不好区分是哪个盘需要做读写。不大好设置阈值。3）根据 Recover IO 与 正常 IO 之间的比例来判定，

3. recover cmd slot 默认 128，根据 extent 的稀疏情况以及 recover extent 的完成情况向上向下调节（auto mode）

   怎么判断 extent 的稀疏情况？lsm 上报的心跳中有 thin pid 的信息

   recover extent 的完成情况可以是 meta 侧统计的下发出去的命令在单位时间内的完成数量

4. recover 并发度、recover cmd slot 支持通过命令行手动调整该值（static mode）

   recover cmd slot 是在 recover manager 上的参数，可以立即生效。max_recover_cmds_per_chunk

   recover 并发度跟限速一样，跟随心跳下发，所以不保证立即生效。
   
   用一个 metadb 放下所有相关的参数
   
   目前的写法好像都是先更新内存再更新 metaDB，需要调整顺序吗？



区分 recover 还是 recover_migrate

1. meta_grpc_server.h 需要同步添加新的 API 吗？
2. 已有的 proto 中的 static recover limit 可以改名吗？会有升级兼容性问题吗？

meta 侧的参数在尽可能让 recover 变快的同时，要考虑自身一次的扫描时间，若扫描周期过长，无法立即触发数据的恢复和迁移



chunk recover 执行的慢可能原因：慢盘、缓存击穿、normal instead of agile recover、

考虑到如果是稀疏的 Extent，恢复命令执行的会比较快。所需要的恢复命令会相对较多。如果是 Extent 数据相对饱和。则恢复没有那么快。所需要的命令会较少。

过去一段时间的恢复速率过慢、recover cmd 完成数量过少、还是 timeout 标记、lsm 侧缓存剩余比例（clean + free 的比例，如果太高的话，说明缓存基本没用上，recover handler 目前已经用了这个值来避免缓存击穿，zbs-chunk cache list 可以看到）、路径很多，先列举出来。PartitionList 有感知 SlowIO 的数量。



作用于 meta 侧的 recover IO 超时相关的 FLAGS

* recover_timeout_ms = 18 min；

作用于 chunk 侧 IO 超时相关的 FLAGS

* chunk_recover_io_timeout_ms = 9s，chunk recover io timeout ms，这个参数实际上没用到；
* local_chunk_io_timeout_ms = 8s，local chunk io timeout ms；
* remote_chunk_io_timeout_ms = 9s，remote chunk io timeout ms，（ZBS 对网络有限制，如果 ping 大包来回超过 1s，认为网络严重故障，系统不工作）。



敏捷恢复为减少内存使用，是有单次最大数量的限制。不过 100G 的写盘应该不会触发这个上限。

调查为啥升级时触发的敏捷恢复数量不及预期可以从维护模式时是否 lease 没清空的角度出发调查。



缓存击穿后，大量的恢复任务争抢 IO，恢复任务容易超时被取消，导致实际恢复速率不足 10MB/s。副本恢复的并发度默认是固定值 32，应该作为一个自适应缓存命中率的值

日志中看到的下发 recover 命令和 EXTENT GC CMD ，两者其实并不是我们想象的相互影响，导致谁也无法推进下去，即不是因为 recover 任务在进行时收到了 EXTENT GC CMD，导致 recover 被中止。这里实际发生的情况是：

1. meta leader 下发 recover 命令到 chunk。
2. chunk 收到命令开始执行，首先创建一个新 pextent（称为 A）作为恢复的目标端。
3. 17 分钟后，chunk 上 recover 命令未执行完成，报错超时。
4. meta leader 发现自己下发的 recover 命令已经超时结束，下发 GC 命令将临时 pextent A 给回收掉。
5. 等待临时 pextent A 回收掉后，meta leader 再次下发新的 recover 命令。（重复步骤 1）

日志上看到的现象：

01:18 下发 recover 命令

01:35 recover 命令超时，收到了 gc 命令

01:36 再次下发 recover 命令

01:53 recover 命令超时，收到了 gc 命令

这里问题的根源在于一个 recover 命令未能在 17 分钟内完成。

---



ZBS-13401

目前在高负载情况下，数据不再会遵循本地化分配原则，而是会尽量的均匀分布。这可能会造成部分虚拟机在迁移之后和原来的性能有较大的差异。需要考虑改善这个场景，也许有两个方向需要考虑：

- 允许用户用命令行触发一个集中策略（向指定的节点聚集一个副本，不需要完整局部化，仅本地化即可），但是不能让指定节点进入超高负载状态（95%）
- 调整平衡策略，在中高负载集群相对均衡后，尝试本地化聚集（不需要局部化，仅保证一个副本在 prefer cid 所在节点即可）

prefer local 节点上没有副本的入口有且仅有这 2 个：

1. 高负载情况下，prefer local 的数据会被迁移；
2. 虚拟机热迁移（用户操作、无法干预）且处于高负载，此时不会做 prefer local 的副本迁移。



改进迁移策略，在节点上待回收数据较多时（已经使用的数据空间占比超过 95%），如果集群没有进入极高负载状态（整体空间分配比例达到 90%），不向该节点迁移数据以保证回收顺利进行。

lsm1 回收空间的速率非常慢，所以如果删除一个 extent，存在 chunk 的 provisioned space 会减少（分配数据空间比例在减小），但 used space 可能仍然很高的情况，如果这时集群上的其他节点向它迁移数据，会进一步降低回收速率。

在 Migrate 时进行检查，如果集群整体尚有可用空间时比如整体 provisioned 比例在 90% 以下，不向 Used Space 比例大于 95% 的节点迁移数据，即便 Provsioned 比较低。



RecoverManager::GetDstCandidates() 中关于 allocated_data_space、valid_data_space、provisioned_data_space 的用法。



得到一台 chunk 上指定 storage pool 中的 pid 的 prefer local

```c++
for (pid_t pid = context_->pid_map->GetNextSetId(next_repair_scan_pids_[storage_pool_id]);
     pid != IdMap::kInvalidId; pid = context_->pid_map->GetNextSetId(pid + 1)) {
  PExtentTableEntry entry = context_->pextent_table->GetPExtentTableEntry(pid, nullptr);
  cid_t prefer_local = entry.PreferredCid();
}
```

ZBS-20993

允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。

支持手动添加 recover/migrate 命令是用于卸盘或啥时候想挪一下副本到指定文件时临时用一下，之后被 doscan 回去也没事。

输入 pid, src, dst, replace

输出 pid, current loc, active loc, dst, owner, mode

搜索 AddRecoverCmdUnlock

1. AddMigrateCmd rpc 参考 RecoverManager::MakeMigrateCmd() 和 AddSpecialRecoverCmd() 的就行，再加些判断条件；

   RecoverManager::AddMigrateCmd() zbs-reader

2. AddRecoverCmd 参考 AddToWaitingRecover() 和 AddSpecialRecoverCmd() 写法。

人工指定后，后续还是有可能会被系统后台程序再次迁移回去，如何应对？如果这个要迁移的卷很大，无法快速完成就被定时线程扫描到，或者卡在超高负载的状态。

```shell
zbs-meta migrate create pid <pid> src_chunk <cid> dst_chunk <cid> replaced_chunk <cid>
```

外部 rpc 触发的 recover/migrate 优先级应该要更高，插到队列第一条（应该不需要）

recover/migrate 要分开讨论，migrate 要额外指定 replace chunk

做一次冲突检查，合法且和当前恢复不冲突，prefer local 和 topo 相关的不管

```c++

Status RecoverManager::AddMigrateCmd(pid_t pid, cid_t src, cid_t dst, cid_t replace) {
    // 手动 rpc 下发的 migrate cmd 会不会被 doscan 又平衡回来？
    // 不确定是否需要加锁
    // 是否需要先把其他 recover/migrate 撤掉呢？因为如果这个 migrate cmd
    if (PExtentHasCmdUnLock(pid)) {
        return Status(EDuplicate) << "pid: " << pid << " already has migrate cmd";
    }

    // 因此此时存在并未执行 doscan，access_manager_ 还没初始化的可能
    if (UNLIKELY(!access_manager_)) access_manager_ = context_->access_manager;
    PExtentTableEntry pentry = context_->pextent_table->GetPExtentTableEntry(pid, nullptr);
    auto now_ms = GetMetaTimeMS();
    if (pentry.IsDead(now_ms)) {
        return Status(EBadRequest) << "pid: " << pid << " is dead, maybe can try recover from temporary replica.";
    }
    if (pentry.NeedRecover(now_ms)) {
        return Status(EBadRequest) << "pid: " << pid << " is recovering.";
    }
    if (context_->pextent_table->IsTemporaryReplica(it->pid())) {
        return Status(EBadRequest) << "pid: " << pid << " is temporary replica, does not trigger migration."
    }
    // 检查 cmd_slots，避免下发过量的 migrate cmd，因为可能同时运行多条命令行
    std::unordered_map<cid_t, int> avail_cmd_slots;
    context_->chunk_table->GetRecoverInfo(&avail_cmd_slots);
    FOREACH_SAFE(avail_cmd_slots, it) {
        if ((it->second) >= FLAGS_max_recover_cmds_per_chunk) {
            avail_cmd_slots.erase(it);
        } else {
            it->second = FLAGS_max_recover_cmds_per_chunk - it->second;
        }
    }
    if (avail_cmd_slots.find(src) == avail_cmd_slots.end() || avail_cmd_slots.find(dst) == avail_cmd_slots.end()) {
        return Status(EBadRequest) << "src chunk " << src << " or dst chunk " << dst << " does not have enough cmd slots.";
    }

    PExtent extent;
    context_->pextent_table->GetPExtent(pid, &extent);
    if (!location_contains(extent.location(), src) || !location_contains(extent.location(), replace)) {
        return Status(EBadRequest) << "given src/replace chunk does not have the replica of pid: " << pid;
    }
    if (location_contains(extent.location(), dst)) {
        return Status(EBadRequest) << "given dst chunk already has the replica of pid: " << pid;
    }
    if (!context_->chunk_table->ReserveSpaceForRecover(pid, src, replace, dst, context_->enable_thick_extent)) {
        return Status(EBadRequest) << "do not have enough space for this migration.";
    }

    RecoverCmdPtr cmd_ptr = std::make_shared<RecoverCmd>();
    cmd_ptr->set_pid(pid);
    cmd_ptr->set_epoch(pentry.Epoch());
    cmd_ptr->set_src_chunk(src);
    cmd_ptr->set_dst_chunk(dst);
    cmd_ptr->set_replace_chunk(replace);
    cmd_ptr->set_is_migrate(true);
    cmd_ptr->set_active_location(pentry.GetAliveLocation(now_ms));
    cmd_ptr->set_start_ms(0ULL);

    SessionInfo owner;
    if (!access_manager_->AllocOwnerForRecover(cmd_ptr->pid(), cmd_ptr->src_chunk(), cmd_ptr->dst_chunk(), extent.alive_location(), &owner)) {
        return Status(EBadRequest) << "fail to alloc migrate owner for recover cmd: " << cmd_ptr->ShortDebugString();
    }

    access_manager_->EnqueueRecoverCmd(cmd_ptr, owner);
    pid_cmds_[cmd.pid()] = cmd_ptr;

    LOG(INFO) << "add recover cmd by rpc for pid: " << cmd_ptr->pid() << " src: " << cmd_ptr->src_chunk()
              << " dst: " << cmd_ptr->dst_chunk() << " replace: " << cmd_ptr->replace_chunk()
              << " owner: " << cmd_ptr->lease().owner().cid();

    avail_cmd_slots[src_replica]--;
    avail_cmd_slots[dst_replica]--;
    if (avail_cmd_slots[src_replica] <= 0) {
        avail_cmd_slots.erase(src_replica);
    }
    if (avail_cmd_slots[dst_replica] <= 0) {
        avail_cmd_slots.erase(dst_replica);
    }
    return Status::OK();
}
```

---

1. node3 上没有 meta 服务是正常的吗？4 节点集群中只有 3 个 meta。目前支持的 meta 是 3 或 5 个。

2. 打快照的行为是在被快照的副本的原地去分配新副本吗？3

3. 7-3 12:04 之前没有在日志中找到 migrate 的标记，fanyang 删除快照后各节点容量回归正常，然后才触发 migrate for localization

4. 快照计划对虚拟机做快照，会有优先分配到虚拟机所在节点的偏好吗？（prefer local）
    我算了一下他一分钟产生一次的快照大概独占空间是 ～20G，高负载情况下 migrate 是 5 min 现

5. 在 7-4 12 点及之前（fanyang 未删除集群中所有快照），分配的 pid 期望 3 副本，结果只分配到 4 和 2（1 和 3 空间不足），触发 recover，但 recover 的 dst 也会选 1 和 3，还是由于空间不足 recover 失败，猜测此时由于 recover cmd 数量过大导致集群没有触发 migrate。

6. 从 0703 22:01:03.398327 开始出现副本分配 2 < 预期副本数 3，pid: 363490，在这前后 2 小时里都没有 migrate，

7. 实际上，更早的时候出现第一条显示空间不足的日志是 chunk 3 ，时间是 22391:E0703 20:59:24.579273，找 pid: 363459 的日志，

    grep -n "FAIL TO COW PEXTENT" zbs-metad.log.20230703-191821.5689 -C 100 | head -n 200

    grep -n "pid: 363459" zbs-metad.log.20230703-191821.5689 -C 5 | head -n 500

    ```c++
    22391:E0703 20:59:24.579273 12378 pextent_allocator.cc:49] There is no enough space on chunk 3 to cow pextent.
    22392-I0703 20:59:24.581032 12378 meta_rpc_server.cc:989] [COW PEXTENT]: pid: 363401 new pid: 363459 loc: [1 ] volume: name: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" size: 214748364800 created_time { seconds: 1681060403 nseconds: 513532843 } id: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" parent_id: "6599b64b-9fea-4597-8733-f55097464673" origin_id: "ffd2a616-2deb-4671-b4a5-b8d60c02576f" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144 access_points { cid: 3 }
    ```

    这个时间点前后最近的 migrate 是 I0703 18:57:56.005661，但也间隔了 2 小时（migrate 正常 1小时触发一次）

    ```
    zbs-metad.log.20230702-112658.5689:I0703 18:57:56.005661 12635 recover_manager.cc:1232] Migrate for localization in storage pool: system
    zbs-metad.log.20230704-120523.5689:I0704 13:04:09.009789 12635 recover_manager.cc:1232] Migrate for localization in storage pool: system
    ```

    跟踪一下 pid: 363459 的生命历程：

    1. 此时快照，对 pid: 363401 COW 得到  new pid: 363459，期望副本在 [3, 1]，由于 chunk 3 空间不足，只分配到 [1]；
    2. 过了 40s ，下发 recover cmd 从 1 到 4；
    3. 离上一次快照 1min，下一个快照就来了，对 pid: 363459 COW 得到 new pid: 363568，期望副本在 [3, 1]，由于 chunk 3 空间不足，只分配到 [1]，此时上面那条 recover cmd 还没完成（不知道做的慢还是根本就没做）；
    4. 过了 8s，又触发从 1 到 2 的 recover（前一次应该是被取消掉了，recover 并没有完成）；
    5. 又过了 15s，触发从 chunk 0 到 chunk 2 的副本迁移，这个日志也很奇怪，应该是从 1 到 3？不过 3 都满载了，为啥会替换到 3 ？
    6. 过 1h，对 pid: 363459 释放 lease owner。

    ```c++
    22391-E0703 20:59:24.579273 12378 pextent_allocator.cc:49] There is no enough space on chunk 3 to cow pextent.
    22392:I0703 20:59:24.581032 12378 meta_rpc_server.cc:989] [COW PEXTENT]: pid: 363401 new pid: 363459 loc: [1 ] volume: name: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" size: 214748364800 created_time { seconds: 1681060403 nseconds: 513532843 } id: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" parent_id: "6599b64b-9fea-4597-8733-f55097464673" origin_id: "ffd2a616-2deb-4671-b4a5-b8d60c02576f" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144 access_points { cid: 3 }
    22397:I0703 20:59:24.595217 12378 meta_rpc_server.cc:2079] [SET PEXTENT EXISTENCE]: [REQUEST]: pid: 363459 origin_pid: 0 epoch: 409375 origin_epoch: 0 generation: 1, [RESPONSE]: ST:OK, , [TIME]: 1359 us.
    22522:I0703 21:00:04.847577 12635 recover_manager.cc:419] add recover cmd for pid: 363459 src: 1 dst: 4 replace: 0
    22805:I0703 21:00:56.738984 12378 meta_rpc_server.cc:989] [COW PEXTENT]: pid: 363459 new pid: 363568 loc: [1 ] volume: name: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" size: 214748364800 created_time { seconds: 1681060403 nseconds: 513532843 } id: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" parent_id: "6599b64b-9fea-4597-8733-f55097464673" origin_id: "53a5c9e4-ff85-4b17-89cc-3ba7317f2f85" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144 access_points { cid: 3 }
    22825:I0703 21:01:04.891676 12635 recover_manager.cc:419] add recover cmd for pid: 363459 src: 1 dst: 2 replace: 0
    23126:I0703 21:01:19.121897 12378 meta_rpc_server.cc:1989] [REPLACE REPLICA]: [REQUEST]: session: "0cdce1ad-b4c3-4412-af9f-90383dd901bc" pid: 363459 src_chunk: 0 dst_chunk: 2 epoch: 409375 reset_location_to_dst: false reset_generation: 18446744073709551615, [RESPONSE]: ST:OK, pid: 363459 location: 513 expected_replica_num: 2 ever_exist: true origin_pid: 363401 epoch: 409375 origin_epoch: 409317 generation: 1 preferred_cid: 3 thin_provision: true alive_location: 513 allocated_space: 108003328, [TIME]: 1361 us.
    150704:I0703 22:02:15.651685 12378 access_manager.cc:1905] Release owner : pid: 363459 session: 0cdce1ad-b4c3-4412-af9f-90383dd901bc
    ```

8. tower 上显示是 0703 15:49 开始每分钟一次快照，第一条显示空间不足的日志是 chunk 3 ，时间是 22391:E0703 20:59:24.579273

    ```
    22391:E0703 20:59:24.579273 12378 pextent_allocator.cc:49] There is no enough space on chunk 3 to cow pextent.
    22392-I0703 20:59:24.581032 12378 meta_rpc_server.cc:989] [COW PEXTENT]: pid: 363401 new pid: 363459 loc: [1 ] volume: name: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" size: 214748364800 created_time { seconds: 1681060403 nseconds: 513532843 } id: "edca71ef-83a6-4f2e-b137-f9be9edb3feb" parent_id: "6599b64b-9fea-4597-8733-f55097464673" origin_id: "ffd2a616-2deb-4671-b4a5-b8d60c02576f" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144 access_points { cid: 3 }
    ```

    这个时刻往前 1 小时，没有任何 migrate

    grep -n -i "recover" -e "I0703 19:5" zbs-metad.log.20230703-191821.5689 -B 200 | head -n 200

    这个时刻往前 2 小时，0703 18:57:56，有 migrate for localization，下发了 59 条命令，其中大量从 2 读，写到 1，根据 session id 不同，replace 分别是 2 或 3。

    chunk 1 3 满，2 4 特别空

    在 0703 18:57:56 到 20:59:24 之间，chunk2 上没有任何的 recover，

    ```c++
    // 这是上一次 migrate 的最后一条，行号 223494
    I0703 18:58:33.277525  5052 recover_handler.cc:1024] [NORMAL RECOVER END] pid: 347167 state: END cur_block: 1024 src_cid: 2 dst_cid: 1 is_migrate: true silence_ms: 0 replace_cid: 3 epoch: 393035 gen: 7
    ```

    应该看 chunk3 的 chunk 日志

    21:00 开始有 recover 1 -> 2/4 的 Failed to recover epextent，ELeaseExpired

    chunk3 从 0703 14:58:09 开始就没有出现 recover extent start/end 的日志 

---



集群内部扫描副本状态，副本迁移相关代码

RecoverManager::DoScan()，正常情况下 60s 检查一次，也可以立即扫描如接受 rpc 请求

RecoverManager::ReGenerateWaitingMigrateList()，分别处理均匀分配的副本和有本地化偏好的副本，他们会有不同的初次分配和迁移策略

RecoverManager::ReGenerateMigrateForRebalance()，针对有本地化偏好的副本

1. 当处于低负载时 

   RecoverManager::ReGenerateMigrateForLocalizeInStoragePool()

   RecoverManager::RepairPExtentForLocalization()，满足如下任一条件不触发迁移：1. 副本的 prefer local cid 不是健康状态（status == CHUNK_STATUS_CONNECTED_HEALTHY）；2. 这个 pid 存在临时副本； 3. replace cid 和 dst cid 都是 0；4. dst_cid/src_cid 上被下发的 recover 命令超过 200 条；5. src_cid 上的有限空间不足 256 MB；

   否则通过 RecoverManager::GetRepairCid() 选取 replace_cid 和 dst_cid，初始值为 0，具体规则如下（选择顺序 dst_cid --> replaced_cid --> src_cid）：

   1. 如果 prefer local 有该副本，并且其他活跃副本满足本地化/局部化的分配策略，说明副本满足分配规则，直接返回 replace dst 的初始值，不触发迁移；
   2. dst_cid 的选取规则：首选没有副本且不处于 failsow 的 prefer local，次选符合期望分布（即符合 LocalizedComparator 分配策略） 的下一个副本；
   3. replaced_cid 的选取规则：从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid，如果都不是 faislow，那么会选到 location 中的最后一个； 
   3. src_cid 的选取规则：默认值为 replace cid，在选定 dst_cid 和 replaced_cid 后，如果 replace_cid slowfail 或者和 dst_cid 不在同一个可用域，那么会从活跃副本中找跟 dst_cid 在同一个可用域且不是 slow fail 的 cid 作为 src_cid。

   选定 3 要素后调用RecoverManager::MakeMigrateCmd()。

2. 当处于中高负载时

   1. 先做目的为拓扑安全的扫描，如有必要（条件是经过以下步骤选出了非 0 的 src/dst/replace）会产生迁移命令

      RecoverManager::RepairTopoInMediumHighLoad()

      RecoverManager::RepairPextentTopo()

      1. 非双活集群：RecoverManager::RepairInZone() 
      2. 双活集群：RecoverManager::RepairInStretched()、RecoverManager::MoveToOtherZone()
   
      以上两种情况的 src/dst/replace cid 选取规则是一样的，通过 RecoverManager::GetSrcAndReplace() 选取 src 和 replace、通过 RecoverManager::GetDst() 获取 dst，不过中高负载与低负载情况下的选取规则是不同的，具体规则如下（选择顺序 replace_cid --> src_cid --> dst_cid）：
   
      1. replace_cid 的选取规则：将已有副本按照拓扑距离排序后，优先把 failslow 节点放在列表头部，然后如果没有 failslow 节点或者存在多个 failslow 节点，这些节点之间按照节点容量从大到小排序。
   
         做好以上准备工作后，从左到后遍历，选择第一个不是 prefer_local 且不是 owner 且命令数未满的节点，如果所有副本所在的节点都不满足这个条件，选择 owner 所在的节点作为 replace_cid，如果副本没有在 owner 上的，那么说明没选到 replace_cid，replace_cid = 0。
   
      2. src_cid 的选取规则：默认是 replace_cid，如果 replace_cid = 0，那么 src_cid = 0，如果 replace_cid != 0 且 failslow，那么从所有副本中任选一个不是 failslow 的节点作为 src_cid。
   
      3. dst_cid 的选取规则：
   
         非双活集群：首选 1. 不处于 failsow 且 2. 没有副本且 3. 还有待生成 cmd 配额且 4. 能让拓扑结构更安全的 prefer local，否则对没有副本的 chunk 按照添加之后能让拓扑结构更安全的方式排序并从中选出第一个满足这 4 个条件的 cid 作为 dst_cid。
   
         > 现有逻辑如果 prefer local 不能让拓扑结构更健康，迁移的 dst 是不会选 prefer local 的
   
         双活集群：与非双活集群略有不同，需要考虑 prefer local 所在的 zone，然后不需要考虑让拓扑结构更安全，详细见 RepairInStretched()
   
      如果 replace_cid|src_pid = 0，那么不会生成 cmd 的。
   
      选定 3 要素后调用 RecoverManager::MakeMigrateCmd()
   
   2. 如果集群不处于极高负载并且做过拓扑安全的迁移（生成了 migrate cmd）直接 continue，否则进行目的为容量再均衡的迁移。
   
      RecoverManager::ReGenerateMigrateForBalanceInStoragePool()，把高负载节点的副本迁移到低负载节点。
   
      RecoverManager::Move()，被迁移副本的优先级：1. 厚制备副本；2. 不是被克隆/快照的精简制备副本（没有 origin pid）；3. 被克隆/快照的精简制备副本（有 origin pid）。
   
      RecoverManager::DoMove()
   
      RecoverManager::ShouldMove()
   
      RecoverManager::MakeMigrateCmd()


经过以上步骤，生成的 Recover cmd 只是放入 passive_waiting_migrate 中，在等待 60s 或者 scan_recover_immediate = true 时会 swap(active_waiting_migrate, passive_waiting_migrate) 

关于 active_waiting_migrate / passive_waiting_migrate 这两个链表：

1. 只有 RecoverManager::GenerateMigrateCmds() / RevokeRecoverCmds() 会往 active_waiting_migrate 中 erase 元素；
2. 只有 RecoverManager::MakeMigrateCmd() 会往 passive_waiting_migrate 中 push 元素，调用 MakeMigrateCmd() 的有：
    1. RecoverManager::RepairPExtentForLocalization()，这是在集群处于低负载时的拓扑安全扫描；
    2. RecoverManager::RepairPextentTopo()，这是在集群处于中、高负载时的拓扑安全扫描；
    3. RecoverManager::DoMove()，这是在集群处于中、高负载时的容量再均衡扫描；
    4. RecoverManager::ReGenerateMigrateForRemovingChunk()，针对要退出的 Chunk 上的所有 pid 做迁移。

关于 active_waiting_recover / passive_waiting_recover 这两个链表：

1. MetaRpcServer::DoRemoveReplica() 移除副本会触发这个 pid 的 recover，调用 RecoverManager::AddToWaitingRecover(pid_t pid) ，它会接着调用 AddToWaitingRecoverIfNecessary() 往 active_waiting_recover 中 insert 元素；
1. RecoverManager::GenerateRecoverCmds() / RevokeRecoverCmds() 会往 active_waiting_recover 中 erase 元素；
3. RecoverManager::ReGenerateWaitingRecoverList() 会 recover pextent table 中所有需要 recover 的 pid，在 DoScan 中被定时执行，它调用 RecoverManager::AddToWaitingRecoverIfNecessary() 往 passive_waiting_recover 中 insert 元素；



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
2. 代码中 [(zbs.labels).as_str = true] 的意思，标记一个可以被安全转换为str 的 bytes field，对于之后向 zbs-proto 提交的新字段，除了明确需要使用 bytes类型存放的字段，其他字符串类型都建议直接使用 “string” 关键字标记，减少不必要的 bytes 类型的使用。引入这个功能的原因是 python2 中 string 为 bytes 的 alias, 但 python3 中 string 和 bytes 则完全不同。 zbs-client-py 的下游用户如果直接适配这种改变需要编写很多针对性的垃圾代码。
3. gtest 如何开启 VLOG DLOG 部分的日志
7. CreateSession 这个过程 SessionMaster 做什么

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



开发机重启后：zk 要手动重启，selinux 要关闭，docker 要手动重启，zkServer.sh start、setenforce 0、service docker start

ZBS RPM 和 SMTX ZBS/SMTX OS/IOMesh 等不同产品的产品版本号从 5.0.0 开始已经分离，各有各的版本号



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

用户可见的副本，是 Meta 认可的 Replica中 Gen 最低的副本，Gen 高不代表它里面的数据有用。




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



C++ 中为减少内存/读多写少的情况，可以用 absl::flat_hash_map 代替 std::unordered_map，https://zhuanlan.zhihu.com/p/614105687

C++ 中 map 嵌套使用，vector 添加一个 vector 中所有元素 https://zhuanlan.zhihu.com/p/121071760

protobuf 中 optional/repeated/ 等用法，https://blog.csdn.net/liuxiao723846/article/details/105564742

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610

git submodule ，https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97，https://zhuanlan.zhihu.com/p/87053283

protobuf 用法，https://bbs.huaweicloud.com/blogs/289568，参考我写的 reposition 中的 patch 

遍历 repeat，https://blog.51cto.com/u_5650011/5389330





网卡随开机自启动

挂载用于存放代码的磁盘

指定公司的 yum 源

生成 ssh key 并上传到 gerrit

yum install -y git224 yum-utils rpm-build zookeeper rpcbind



安装 docker，https://blog.csdn.net/zzhongcy/article/details/131402389，设置成随开机自启



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

