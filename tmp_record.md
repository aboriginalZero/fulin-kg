3 节点集群，2 副本 prio-volume 持续进行写 IO，对 prio-volume 一直打快照，手动 gc-scan 一下，节点进入高负载后，一直未触发数据迁移。





策略类梳理（seq means prior）

1. 通过 rpc 显示指定的 recover cmd（这个不一定要支持）

2. 周期性扫描产生的 recover cmd

    待做 [ZBS-21199](http://jira.smartx.com/browse/ZBS-21199)，支持设置允许 recover 的时段，不在该时段内仅做 partial recover。在判断每个 pid 是否需要 recover 时，每个 pid 拿到 pentry 的时候就可以判断如果期望副本是 3 而目前副本是 2 时，不用触发 recover。

    往 UpdatableRecoverParams 中增加 2 个字段，start hour  end_hour。同时，通过这个 patch 改 recover 的触发策略，IsNeedRecover。

3. 预期内的节点下线

    ReGenerateMigrateForRemovingChunk()

    待做 [ZBS-21443](http://jira.smartx.com/browse/ZBS-21443)，节点移除还要考虑其他节点的负载情况。

    因为节点移除时，他上面的副本需要马上迁移完，否则那条 cli 就一直执行不完。从存储池中移除节点的时候，仅考虑 migrate dst cid 在不迁出副本的情况下是否可以容纳待迁移的数据，但这样可能导致 dst cid 超高负载或满载后，副本还没迁移完。

    例如节点 a b c d ，存在大量 extent 的 3 副本分布在 b c d，此时如果移除 c ，那么只能向 a 迁移，a 如果容量较小，可能会被 c 来的数据填满。

    这个过程中，当 a 进入高负载，可以考虑将部分副本移出，以腾出空间给 c 要移动过来的副本。需要确认一下 a 在作为 migrate dst 且进入高负载时，是否有机会把自己可迁移的数据迁移出去。

    要在这个逻辑里加上，到了超高负载时，先执行 ReGenerateMigrateForBalanceInStoragePool()

4. 通过 rpc 显示指定的 migrate cmd

    待做 [ZBS-20993](http://jira.smartx.com/browse/ZBS-20993)，允许 rpc 触发 migrate 命令，应该可以和预期内的节点下线合在一起做，因为他们的优先级都会更高，需要马上看到迁移效果

    可以对外做 2 个接口，一个是 pid 为粒度的，一个是 volume 为粒度的（MigrateForVolumeRemoving）

5. 低负载

    ReGenerateMigrateForLocalizeInStoragePool()，让副本位置符合 LocalizedComparator

6. 中高负载

    中高负载目前实际上的区别仅在：

    1. 中负载每 1h 扫描一次，高负载每 5min 扫描一次；
    2. 中负载不移动 local 和 parent 的 pextent，高负载会移动；
    
    如果 ReGenerateMigrateForRepairTopo 生成了 cmd，那么只生成这个目标的 cmd，否则试图去生成 ReGenerateMigrateForBalanceInStoragePool 的 cmd。需要对 ReGenerateMigrateForBalanceInStoragePool() 改进，先保证都有本地副本，再去做容量均衡。
    
    待做 [ZBS-13401](http://jira.smartx.com/browse/ZBS-13401)，让中高负载的容量均衡策略都要保证 prefer local 本地的副本不会被迁移，且如果 prefer local 变了，那么也要让他所在的 chunk 有一个本地副本（有个上限是保留归保留，但如果超过 95%，超过的 部分不考虑 prefero local 一定有对应的副本）。

7. 超高负载

    跳过 topo repair 扫描，只做 ReGenerateMigrateForBalanceInStoragePool()

所以，先做

1. ZBS-13401

   目前在高负载情况下，数据不再会遵循本地化分配原则，而是会尽量的均匀分布。这可能会造成部分虚拟机在迁移之后和原来的性能有较大的差异。需要考虑改善这个场景，也许有两个方向需要考虑：

   - 允许用户用命令行触发一个集中策略（向指定的节点聚集一个副本，不需要完整局部化，仅本地化即可），但是不能让指定节点进入超高负载状态（95%）
   - 调整平衡策略，在中高负载集群相对均衡后，尝试本地化聚集（不需要局部化，仅保证一个副本在 prefer cid 所在节点即可）

   prefer local 节点上没有副本的入口有且仅有这 2 个：

   1. 高负载情况下，prefer local 的数据会被迁移；
   2. 虚拟机热迁移（用户操作、无法干预）且处于高负载，此时不会做 prefer local 的副本迁移。

2. ZBS-21443、ZBS-20993

3. ZBS-21199





1. http://gerrit.smartx.com/c/zbs/+/53689 代码更新，并补充对应的 zbs cli
2. zbs cli 中加上 reposition cli，并添加 rpc，跟手动触发 mgirate rpc 指令一起做
    1. 能够观察 recover 真正 IO 的数据量，block 粒度的（比如如果有敏捷恢复，这个 pextent 就不会恢复 256 MB）
    2. 能够查看 generate/pending_recover 的数量
    3. 能够查看 need_migrate 的数量
3. 智能模式中，值变化的时候添加 log
3. ZBS-25666 pick 到 v5.5.x
3. 改 recover manager 中的函数名，比如 GenerateRecoverCmds 实际代表 DistributeRecoverCmds，还有计数相关的，recover 处有 2 个，可以精简的，把 recover 和 migrate 做到对称。



如果都给了 topology 且两个副本的 zone distance, topo distance 都相同的情况下，LocalizedComparator 和 TopoAwareComparator 区别在于：

1. 前者只有在 owner = 0 时，选第一个 cid 时会用 recover_comparator，否则用 ring id 比较；
2. 后者全部用 recover_comparator 比较，也就是先 recover cmd num 再容量。

就一个副本并且跟 prefer local 不在一个 zone，那么 recover_prefer = 剩下的这个副本。如果就一个副本，并且跟 prefer local 在一个 zone，又会怎么样呢？

comparator->UpdateChunkSet 这个地方，如果还剩的 2 副本并不符合 topo 安全，那么是会按照放进去的第 2 个副本来选择后面的第 3 个副本

```c++
LOOP(candidate_dst_chunks.size()) {
        LOG(INFO) << "yiwu candidate_dst_chunks " << i << " " << candidate_dst_chunks[i].id();
    }

LOOP(comparator->cids.size()) { LOG(INFO) << "yiwu push in cids " << comparator->cids[i].first; }

LOG(INFO) << (all_in_same_zone ? "all_in_same_zone" : "not");
LOG(INFO) << "all_zone_idx " << all_zone_idx;
LOG(INFO) << "prefer_zone_idx " << prefer_zone_idx;
```



为什么低负载下，replace_cid 要尽量跟 dst_cid 在一个域，考虑同时有 2 副本不符合拓扑安全的场景。

1. 如果已有副本和 dst 在不同 zone，replace 随便选

2. 如果已有副本和 dst 在同一个 zone，replace 也得选这个 zone 的，如果没有这个 zone 的，说明在这之前 3 个副本

只要 topo 不变，符合拓扑安全的副本位置也不变，LocalizedComparator 得到的排序结果是不变的，因此虽然选样本是连锁反应，但就算第 2 个副本位置不对，第 3 个副本位置还是正确的。

scan_extents_num 应该放在 ReGenerateMigrateForUnevenVolume 这个函数中去做统计，虽然实际上 all_pool 中就一个，且一次只会因为一种方式去 migrate(本地化/容量均衡)。

GenerateRecoverCmds 里面的 src 也可以有选择策略的，目前的写法太乱了。

低负载没有区分双活，中高负载开始有区分



ChunkTableEntry 的 last_succeed_heartbeat_ms 字段没用上

连续快照的 allocated_data_space 不对劲

chunk.chunk_space_info.thin_used_data_space 包含这个 chunk 最近一次上报的 thin extent 总空间消耗，用于在 meta 切换后进行集群 used_data_space 计算，在该 chunk 未再次上报 used space 前也能显示较为合理的集群空间消耗，空间分配改进中引入的字段。



实现命令行

多个参数用一个 DB、考虑升级兼容性问题、reposition 跟 recover migrate 分开、额外添加 rpc 用法

1. 能够观察 recover 真正 IO 的数据量，block 粒度的（比如如果有敏捷恢复，这个 pextent 就不会恢复 256 MB）
2. 能够查看 generate/pending_recover 的数量
3. 能够查看 need_migrate 的数量



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
   
   ZBS-25386 有描述一个当 lsm 读取速度过慢时，会导致大部分迁移命令超时的问题，因此并发度也可以根据 lsm 读取速率来调节



区分 recover 还是 recover_migrate

1. meta_grpc_server.h 需要同步添加新的 API 吗？不需要
2. 已有的 proto 中的 static recover limit 可以改名吗？会有升级兼容性问题吗？不会

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



migrate 和 recover 只是共用 RecoverCmd 这个数据结构，各自的命令队列（recover 是 std::set，migrate 是  std::list）、触发时机。


IO 下发的流程

NFS/iSCSI/nvmf -> ZBS Client -> access io handler -> generation syncor -> recover handler



低负载，ReGenerateMigrateForLocalizeInStoragePool，没有对是否双活显式区分，都被封装到 LocalizedComparator 里了

```c++
dst_cid
// dst_cid should meet: (seq means priority)
//   1. prefer local;
  
// dst_cid must meet:
//   - not failslow
//   - enough cmd quota
//   - enough remain valid space
replace_cid
// replace_cid should meet: (seq means priority)
//   1. same zone with dst_cid;
//   2. failslow;

// replace_cid must meet:
//   - not owner
src_cid
// src_cid should meet: (seq means priority)
//   1. same zone with dst_cid;
//   2. not failslow;

// src_cid must meet:
//   - enough cmd quota
```

节点移除

```cpp
dst_cid
// dst_cid should meet: (seq means priority)
//   1. not failslow
//   2. same zone with prefer local
//   3. comparator above

// dst_cid must meet:
//   - not in alive loc
//   - not in exclude_cids
//   - enough remain valid space
//   - not in the same zone where in stretch cluster and size(all_src_chunks) > 	
//     kMinReplicaNum, and all src chunks except replace_cid are located in
 
src_cid
// src_cid should meet: (seq means priority)
//   1. not failslow
//   2. same zone with dst_cid

// src_cid must meet:
//   - none
```

中高负载以 topo 安全为目的的迁移

```cpp
replace_cid
// replace cid must meet
//   - not prefer local
//   - engough cmd quota （replace_cid 应该不用考虑命令配额）

// replace cid should meet
//   1. not owner
//   2. failslow
//   3. lower available capacity

src_cid
// src_cid must meet:
//   - not failslow

// src_cid should meet:
//   - none

dst_cid
// dst_cid must meet:
//   - not failslow
//   - not src_cid
//   - enough remain valid space
//   - better topo safety than replace cid

// dst_cid should meet:
//   1. prefer local
  
```

中高负载，在 topo 安全不降级的情况下，优先选 prefer local，在这里我只需要调一下顺序就好，先 dst_cid 再 src_cid 再 replace_cid

中高负载，容量均衡迁移，其中，中负载可以容忍容量没那么均匀，并且允许此时最高负载和最低的如果 ratio 相差不超过 0.01 或者 extent 数量不超过 20 个。

超高负载，容量均衡迁移，

之前的迁移都是遍历 pid，而容量均衡迁移是先遍历 chunk 配对选出 replace_chunk 和 dst_chunk，然后才是从中选出合适的 pid。

```cpp
// replace_cid should meet:
//   1. bigger rx + tx		(值得商榷)
//   2. bigger cache use
//   3. bigger partition use

// replace_cid must meet:
//   - partition used ratio > cluster avg partition used ratio

// dst_cid should meet:
//   1. smaller rx + tx
//   2. smaller cache use
//   3. smaller partition use

// dst_cid must meet:
//   - not failslow
//   - not select by pre src_cid
//   - not >95% when cluster avg partition used ratio < 95%
```

如果选出的 replace_cid 和 dst_cid 不在一个 zone，直接返回了（这其实没有利用上跨 zone 节点的存储能力，但其实一开始就根据不同 zone 分开了，所以这个判断其实是无效的）

接下来选 src_cid 上的 pid，中负载不迁移 lease owner、parent、prefer local、even，高负载不迁移 lease owner、even，允许迁移 parent 和 prefer local。

```cpp
// pid should meet:
//   1. thick
//   2. thin and origin_id == 0
//   3. thin and origin_id != 0

// pid must meet (medium load):
//   - not temp replica
//   - src_cid not lease owner
//   - not thin and origin != 0		(allow in high load)
//   - src_cid not prefer local		(allow in high load)
//   - not even
//   - better and equal topo safety
//   - replace_cid and dst_cid have enough cmd quota

// src_cid 默认等于 replace_cid，
// 如果 failslow，那么从活跃副本中选出不是 failslow 的第一个，否则不生成 cmd


// src_cid should meet:
//   1. replace_cid
//   2. 还可以添加 same zone with dst

// src_cid must meet:
//   - not failslow
```



现有迁移策略：

1. 移除节点迁移；
2. 均匀分布迁移；
3. 据负载做不同迁移：
    1. 低负载：本地化分布迁移（包含本地化 + 局部化 + topo 安全）
    2. 中高负载 ：topo 安全迁移（仅包含 topo 安全）+ 容容量均衡迁移，如果 topo 安全迁移生成了 cmd，跳过容量均衡迁移，否则根据容量均衡生成 cmd；
    3. 超高负载：只做容量均衡迁移。

引入 prio-extent 后：

1. 移除节点上 prio-extent 的迁移；

2. 移除节点上 normal-extent 的迁移；

3. normal volume 均匀分布迁移（不论是 prio-extent 还是 normal extent）；

4. 据负载做不同迁移：
    1. allocated_prs > valid_cache_space：将在容量层的 prio-extent 优先迁移到 prs，再是性能层非 prs 空间上的 prio-extent 迁移到 prs；
    1. valid_cache_space > allocated_prs > planned_prs：将在性能层非 prs 空间上的 prio-extent 迁移到 prs；
    1. 若 allocated_prs < planned_prs，说明都在 prs 里，不需要迁移；
    
    在 pinperf 中分别定义了 high_prio_load、medium_prio_load、fine_prio_load 来对应这 3 种



现有负载的计算是只算 partition 的已用比例。

目前 migrate 中的逻辑是每次获取一个 pid 的 entry 都要通过 GetPhysicalExtentTableEntry 调用一次锁，但在 p rio-extent 的迁移中，可以批量获取 diff_pids 中所有的 pentry。



```cpp
/*
+--------------------------+----------------------------------------------+
| +-------------+		       |   																						|
| | planned_prs |          |																							|
| +-------------+          | 																							|
|   valid_cache_space      |               valid_data_space								|
+--------------------------+----------------------------------------------+

what prio_load of chunk depends on its allocated_prioritized_space(replace with x)
	1. low_prio_load_chunk    : x <= planned_prs
	2. medium_prio_load_chunk : planned_prs < x < valid_cache_space
	3. high_prio_load_chunk   : x > valid_cache_space

if a chunk is in high_prio_load_chunk:
	1. low_prio_data_space    = planned_prs
	2. medium_prio_data_space = valid_cache_space - planned_prs
	3. high_prio_data_space   = x - valid_cache_space

if a chunk is in medium_prio_load_chunk:
	1. low_prio_data_space    = planned_prs
	2. medium_prio_data_space = x - planned_prs
	3. high_prio_data_space   = 0

if a chunk is in low_prio_load_chunk:
	1. low_prio_data_space    = x - planned_prs
	2. medium_prio_data_space = 0
	3. high_prio_data_space   = 0
*/
```

1. planned_prs 是节点为 pinperf 预留的性能层空间，由 lsm 管理；
2. allocated_prs 总是等于 prio-replica * 256MiB，由 meta 管理；
3. downgraded_prs 是位于容量层的数据量，由 lsm 管理。



在通常分配，迁移和恢复副本的开始阶段都需要分配一个副本，对于 prio-replica，我们可以定义出三个副本放置级别：

1. 只有 allocated_prs + extent_size <= planned_prs && allocated_data_space + extent_size < valid_data_space 时允许放置 replica；

    说明此时在 prs 放得下，可以把在 2 3 场景下的副本迁过来

2. 在 allocated_prs + extent_size <= valid_cache_space && allocated_data_space + extent_size < valid_data_space 时允许放置 replica；

    说明此时在性能层的其他空间放得下，可以把 3 场景下的副本迁过来，这种场景下，prio-replica 可能会挤占节点的 normal-replica 的 cache 空间，造成 normal extent 的性能降级。

3. 在 allocated_data_space + extent_size <= valid_data_space 时允许放置；
    1. 放置到版本包含了 pinperf 特性的节点；
    2. 放置到低版本的节点（包含 LSM1 的节点）；
    
    prio-replica 可能会部分或全部地被放置到容量层设备上，可能造成 prio-replica 性能降级

称为 prio-replica 放置级别 1,2,3，3 细分为 3.a 和 3.b。除非特别指明，放置级别更高的副本放置策略仍然会首先尝试将副本按照低放置级别的策略放置 prio-replica，例如一个放置级别 3 的副本放置，会优先寻找 allocated_prs + extent_size <= planned_prs 的节点。



prio-extent 需要考虑 prefer local 和 lease owner、failslow、topo 安全、节点 prio 类容量、剩余 cmd 配额、pior 负载、是否双活、是否 even volume、是否有过克隆/快照行为。



现有代码在 EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，但如果 src_cid 跟 dst_cid 跨 zone，或者是 failslow，（他可能没有 cmd quota 了，不过这个条件在生成时是硬性规定的，派发时倒不用）这么选还是好事吗？

1. 后续可以改进容量均衡迁移中 replace chunk 和 dst chunk 1 1 配对，可以改成尽可能让多个 src_cid 参与进来，除非所有 under chunk 都不行，才退出循环。
2. http://gerrit.smartx.com/c/zbs/+/54622 patch 7 要拆分成几个小 patch



pin 的 3 个参数

1. meta 下发的 planned_prs 是 extent 粒度，值等于 valid_cache_space * prio_space_percentage，valid_cache_space 是 lsm 上一轮心跳汇报的结果，prio_space_percentage 是用户通过 rpc 设置的值；

   > 除了 rpc 更新会立即下发最新的 prio_space_percentage，在日常心跳中每轮也会通过 reserve_prio_space 字段下发给 lsm

2. meta 下发的 allocated_prs 是 extent 粒度，值等于 prioritized_pids * 256 MiB；

   > lsm 目前汇报的 allocated_prs = prior_extent_num * extent_size，不管实际数据使用量多少，也不管有没有共享 pblob，不过它汇报的这个值目前还没用，以后想用的话用 Meta - LSM 的就知道哪些空间还没有落到 LSM 上

3. lsm 汇报的 downg_prs 是 pblob 粒度，值等于 (all_prior_pblob - (planned_prs * 1024)) * 256 KiB，不管 prio pblob 是在 normal cache 还是 partition；

   > 由于会有共享的 pblob，所以即使汇报的 allocated_prs > planned_prs，downg_prs 也可能为 0



prio pblob 生命周期：

1. planned_prs 是在 meta 侧的值，也是用户视角的 prio 的空间大小，通过心跳下发给 chunk，LSM 为其在 cache 中预留 planned_prs * 1024 个 pblob 的空间（即 prio pblob，仅预留，未实际使用）；
2. 对 prio volume IO 时，卷内分配 prio extent，meta 下发请求到 lsm，lsm 根据这个 prio extent 实际使用的 pblob 数（extent 未必写满）在 prio pblob 预留空间里去分配 prio pblob；
3. 在 prio IO 一段时间后，如果预留的 prio pblob 空间都写完了，就会用 normal cache，此时 downg_prs 开始从 0 递增；
4. 继续分配 prio pblob 到 normal cache 都被用尽，那只能分配到 paritition 上，在 partition 上的 pblob 也算在 downg_prs 里面。

预期内的 planned_prs 都是小于 valid_cache_space 的，但如果出现掉 cache 盘，会导致 planned_prs > valid_cache_space，在掉盘而 planned_prs 还没更新的这段时间里，prio extent 会分配到 partition，但在一次心跳过后（request 传递 valid_cache_space，response 据此计算 reserved_prio_space 并下发作为 lsm 的 planned_prs）又可以保证 planned_prs < valid_cache_space。



在 lsm 视角，extent 只是 pblob 的数据集合，实际承担数据的是 256 KiB 的 pblob。

5.5.x 开始，prio extent 在 planned_prs 内的数据会常驻 cache 层，而 normal extent 是保留之前的 IO 行为：

1. IO 到 lsm 侧是先写 cache 层，cache 层的 pblob 会被 writeback 到 parition 层，在 cache 负载率超过 90% 时， writeback 完了之后变成 clean 的 pblob 会被淘汰；

   >  cache 负载率不高的话，writeback 了也不会被淘汰，淘汰的操作就是更改 pblob 的元数据，把 pblob 的 cache_blob_id 设为空，这样数据就不存在于 cache 上了，释放的 cache block 又可以被其他 pblob 重用。
   > writeback 一直在进行，只是限速会动态改变，当 cache 负载率高的话，writeback 速度会加快，当 APP IO 多的话，writeback 速度会调低。
2. 当 partition 层数据被多次访问后会 promote 到 cache 层，判断条件是 30 分钟内访问 3 次，此时数据在 cache 和 partition 中各有一份。



zbs 迁移策略（数字代表优先级）

1. chunk remove 迁移

   * prio extent

     1. 若 lhs 和 rhs 其中有不具备 prio 能力的，先选具备 prio 能力的，再选 v5.5.0 节点但 planned_prs 为 0，最后选老版本节点，否则说明都具备 prio 能力，执行下一步；
     2. 若 lhs 和 rhs 在容纳一个新副本后都是 low prio，按本地化 + 局部化 + topo 安全策略选；若只有一个是 low prio，选这个；否则说明都无法容纳，执行下一步；
     3. 若 lhs 和 rhs 在容纳一个新副本后都是 medium prio，选 valid_cache_space - allocated_prs 更大的；若只有一个是 medium prio，选这个；否则说明都无法容纳，执行下一步；
     4. 选 lhs 和 rhs 中 allocated_prs - valid_cache_space 更小的，选取结束。

   * normal extent
     1. 若集群是中低负载，按本地化 + 局部化 + topo 安全策略选，否则执行下一步；
     2. 若 prefer local 节点是中低负载，按本地化 + topo 安全策略选，否则执行下一步；
     3. 按 topo 安全策略 + 低容量优先选。

2. even volume 均匀分布迁移：prio 与 normal 一致，都是让 even volume 卷内 extent 先做 topo 安全，若满足 topo 安全，再做 topo 不降级 extent 数量均衡。

3. prior 超载迁移（存在 medium/high prio chunk）：在保证有足够的性能层和容量层空间前提下，按以下优先级做 topo 不降级节点 prio 负载均衡。
   1. high prio 到 low prio 
   2. high prio 到 medium prio 
   3. medium prio 到 low prio 
4. normal 低负载迁移（< 75%）：normal 和 prio extent 都按本地化 + 局部化 + topo 安全做迁移，normal 只需要保证有足够的容量层空间，而 prio 需要保证有足够的容量层和性能层空间，并且不引发 prio 超载。
5. normal 中高负载迁移（>= 75%）：先做 prio 和 normal extent 的 topo 安全，若满足 topo 安全，再做 normal extent 的 topo 不降级容量均衡。
6. normal 超高负载（>= 95%）：只做 normal extent 的 topo 不降级容量均衡。

prio 超载迁移的负载均衡终态只要求各 prio chunk 处于 low prio load，不要求 prio extent 在各节点均匀分布。另外，欠载 prio chunk 上的 extent 分布会在 normal 低、中高负载迁移中被调整。

chunk 级别的 prio-space 负载分为 3 种，计算方式：

1. 低负载（low_prior_load）： allocated_prs <= planned_prs；
2. 中负载（medium_prior_load）：planned_prs < allocated_prs <= valid_cache_space；
3. 高负载（high_prior_load）：valid_cache_space < allocated_prs；

cluster 级别的 prio-space 负载分为 2 种，计算方式：

1. 欠载（underload）：不存在中、高负载的 chunk
2. 过载（overload）：存在中负载或高负载的 chunk



如果节点上的 replica 发生 cow 的话，direct_prs 会瞬间减小而导致写入因为等待空闲 cache 而阻塞，也需要预先下刷以避免阻塞

pin 的 allocation 中的空间计算方式，以及初次写的 lease，怎么传递 prioritized 给 lsm





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







待整理

C++ 中为减少内存/读多写少的情况，可以用 absl::flat_hash_map 代替 std::unordered_map，https://zhuanlan.zhihu.com/p/614105687

C++ 中 map 嵌套使用，vector 添加一个 vector 中所有元素 https://zhuanlan.zhihu.com/p/121071760

protobuf 中 optional/repeated/ 等用法，https://blog.csdn.net/liuxiao723846/article/details/105564742

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610

git submodule ，https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97，https://zhuanlan.zhihu.com/p/87053283

protobuf 用法，https://bbs.huaweicloud.com/blogs/289568，参考我写的 reposition 中的 patch

遍历 repeat，https://blog.51cto.com/u_5650011/5389330

stl 容器迭代器失效问题，https://stackoverflow.com/questions/6438086/iterator-invalidation-rules-for-c-containers

对于 cmp

1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false



vscode 中用 vim 插件，这样可以按区域替换代码

一个遗留问题是，单测里面想要触发两次 recover cmd，怎么让 entry 的 GetLocation() 得到及时更新，试了 sleep(9) 不行，可能不止需要一个心跳周期，还有其他条件没触发。
