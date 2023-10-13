[ZBS-26042](http://jira.smartx.com/browse/ZBS-26042) 还缺一个 even volume 的 ut 验证 [ZBS-25847](http://jira.smartx.com/browse/ZBS-25847)

perf_distribute_cmds_per_chunk_limit 或许得改成 perf_generate_cmds_per_chunk_limit 更符合语义。



1. recover 支持分批扫描

   比较纠结的是，目前 recover_manager 中 scan_extents_per_round_limit 这个参数只限制了 recover / migrate for localization / migrate for repair topo 一次扫描的 pextent 数量，而对其他类型的 migrate （如 migrate for even volume/ prior extent / rebalance）并没有做限制，这让他的语义并不完整。

   根据 review 意见修改

   http://gerrit.smartx.com/c/zbs/+/8495

   http://gerrit.smartx.com/c/zbs/+/26377

   http://gerrit.smartx.com/c/zbs/+/16123/3

2. 区分 distribute_cmds_per_chunk_limit 和 generate_cmds_per_chunk_limit，目前把他两混用了，或者考虑下是否需要引入新的字段；在 recover / migrate 中的限制并不相同。

   ```c++
   // GenerateRecoverCmds() 中用法可能有问题
   if (recover_cmd_nums >= generate_cmds_per_round_limit_) {
               break;
           }
   ```

3. 改 recover manager 中的函数名，比如 GenerateRecoverCmds 实际代表 DistributeRecoverCmds，还有计数相关的，recover 处有 2 个，可以精简的，把 recover 和 migrate 做到对称。

4. 互斥的做法

   ```
   // 这个写法有起到互斥的作用吗？
       {
           LockGuard l(&mutex_);
           wait_recover_count = active_waiting_recover_count_;
           wait_recover = active_waiting_recover_;
       }
   ```

5. 为啥没有 active_waiting_migrate_count_ 和 passive_waiting_migrate_count_，这是以 chunk 为粒度的，记录的 generated recover cmd，

6. reposition params 支持分层，相对修改 zbs-client-py 的代码，两个 patch 一起







单测里面

```cpp
// 验证双活生效
RecoverManager* recover_manager = GetMetaContext().recover_manager;

// 验证 GFLAGS 更改生效
RecoverManager recover_manager(&(GetMetaContext()));
```





1. 怎么看在哪里调用了 pyzbs 中的 update_reroute_version 函数

2. xen 平台还有人在使用吗？

3. reroute.py 中提供了 LOOP 和 ROUTE 模式，后者应该是用于人工指定一个 target i，相当于一个智能模式，一个静态模式。

4. RerouteConfig.LOOP_FINISHED 貌似没有 true 的时候

5. 为啥要软链接一份日志 check_lock_and_log_file

6. smartx token 为啥每 2 秒获取一次，token 过期时长是 7 天？refresh_service_token，或许可以改成如果 bearer_token 为 None 时才获取，不为 None 验证过期了再获取。

7. 怎么看 /api/v3/sessions 对应的 pyzbs 中的方法

8. 每 200 次判断一下是否 enable reroute 和 enable secondary data channel？对 enable 的响应会不会太慢？200 * 2 = 400s 才能变更状态。不过其实也不会有多经常变更，所以也还好。

9. 从 shell 切 python，reroute version 从 1.6 变成 2.1

10. 为啥要对 target_ip send heartbeat

11. ping 发 icmp 包为啥要自实现一个 ICMPHandler？

    客户的 vmware 环境里 scvm vNIC 开启了 MTU check，网络包大小如果超过了 MTU 值会被 drop 掉

12. ioreroute 需要在非常明确的场景下才进行路由切换，https://cs.smartx.com/cases/detail?id=1050822

13. 需要预防 python 中的 shuffle 不会吞掉 ip_list 中的值，参考 http://gerrit.smartx.com/c/pyzbs/+/40696



一个 extent 只要写过 1 次真实数据，EverExist 就是 true，如果从没有写过并且不是来自 COW ，那它就没有任何有效数据。在副本迁移或者恢复的时候可以有一些简化的特殊处理。



存储分层模式，可以选择混闪配置或者全闪配置，其中全闪配置至少需要 1 块低速 SSD 作为数据盘，混闪配置至少需要 1 块 HDD 作为数据盘。

存储不分层模式，不设置缓存盘，除了含有系统分区的物理盘，剩余的所有物理盘都作为数据盘使用，只能使用全闪配置。

smtx os 5.1.1 中不论存储是否分层，都要求 2 块容量至少 130 GiB 的 SSD 作为 SMTX OS 系统盘（含元数据分区的缓存盘），为啥要 2 块做软 raid 1？



在恢复或者迁移任务结束时，新加入副本的状态被设置为未知，需要等待下一次心跳周期 LSM 上报副本后才可以确认副本为健康。

在心跳开始之前，如果执行 find need recover extent 命令，将会把这样的 extent 返回，这会导致升级过程中，如果有数据迁移发生，则会被判定会产生了恢复，导致升级过程退出。

修复方式为调整新加入的副本状态，从未知（需要修复）调整为不需要修复，但是最近并不活跃，并不能直接清理恢复命令。 



```
ssh -i /vmfs/volumes/65128d50-5afed7ff-1661-005056ab5edf/vmware_scvm_failure/.smartx_key/smartx_reroute_id_rsa -o ServerAliveInterval=1 -o BatchMode=yes -o ConnectTimeout=1 -o StrictHostKeyChecking no -o UserKnownHostsFile /dev/null -o GSSAPIAuthentication no root@10.97.60.236 timeout 2 curl -H 'Content-Type: application/json' -X PUT -d '{"local_clients_ips":"192.168.77.36,10.97.60.235"}' http://10.97.60.236/api/v2/zbs_session/session/86dccf70-f0c8-4a02-ab6d-ca9a49ffcd74
```



MetaRpcServer::MarkAllocEvenIfNecessary、MetaRpcServer::ResetVolumeAllocEven 会调用



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





VIP 设计文档，https://docs.google.com/document/d/1M34zaIje2xkUSv9Q41waRH4GCPZ5yv7hwZqOK77vCq8/edit#heading=h.feb6l5x4y4vk

双活设计文档，https://docs.google.com/document/d/1z2cUXLrQ7pZnkJXiCPCLp-BPxkpZCSFIwrmvDxrUYX4/edit#heading=h.rxadnjfqdyav



重构 recover manager 的话，可以不用再考虑支持 Storage Pool 了吧？我看 CheckUpgradeThinProvision 里面都没有去遍历各个 StoragePool

rx_pids -> dst_pids，tx_pids -> replace_cids, recover_src_pids -> src_pids

那这里还有两个问题：

1. 卸载 partition 盘的时候，chunk 和 meta 分别会做哪些校验，分别用的哪个字段；
2. 卸载盘（或者拔盘）之后，可能会出现 allocated space > data capacity，进而导致数据迁移受到影响。meta 能否在集群数据恢复完成之后，保证 allocated space <= data capacity 呢？



如果允许出现 allocated_data_space 大于 valid_data_space，感觉空间计算部分都会有负数的情况





thick_pids insert 的位置是 ChunkTable::ReplacePExtent 和 ChunkTable::AddPExtent

PhysicalExtentTable::SetPExtentUnlocked() 和 PhysicalExtentTable::AddReplicaUnlocked() 忽略

PhysicalExtentTable::SetPExtents()

SpaceTransaction::PersistPExtents() 和 ReserveVolumeSpaceTransaction::Commit() 和 UpdateVolumeTransaction::CommitInternal()

感觉是没有触发迁移。

CowPExtentTransaction::Commit()

CowPExtentTransaction，UpdateVolumeTransaction，ReserveVolumeSpaceTransaction





1. zbs-deploy-manage storage_pool_remove_node < storage ip> 
   1. 这个命令会调用 zbs 侧的 RemoveChunkFromStoragePool rpc，只做剩余空间检查，检查通过后，chunk 状态改成 REMOVING，日志里出现 REMOVE CHUNK FROM STORAGEPOOL；
   2. recover manager 有个 4s 定时器会为状态为 REMOVING 的 chunk 生成迁移命令并下发，而对 migrate dst 的选取，如果是在集群 normal low/medium load，会按本地化 + 局部化 + topo 安全策略选，如果是 normal high load，优先考虑 topo 安全，然后才是剩余容量；
   3. 等待这个 chunk  pextent 全被 remove（命令行看 provisioned_data_space 为 0），chunk manager 有个 4s 的定时器会将状态为 REMOVING 且它上面的 pextent 全被 remove 的 chunk 改成 IDLE；
2. zbs-deploy-manage meta_remove_node < storage ip>
   1. 这个命令会调用 zbs 侧的 RemoveChunk rpc，此时要求 chunk 处于 IDLE；
   2. 把 chunk 的各种持久化信息从 metaDB 中删除；
   3. 清空 meta 内存里各种表（chunk_table, chunk_id_map,  topo_objs_map, ）中的记录；
   4. 清空 meta 侧这个 chunk 相关 session，在 iscsi_table/nvmf_table 中把这个 chunk 标记为 inactive（避免新的数据接入），通过心跳异步告知其他 chunk 这个 chunk session 失效；
   5. 日志里出现 REMOVE CHUNK；



[ZBS-25686](http://jira.smartx.com/browse/ZBS-25686) 前，只在 recover 里用上了 maintenance cid， 代码是 RecoverManager::NeedRecover 中的 IsChunkInMaintenanceMode，[ZBS-25686](http://jira.smartx.com/browse/ZBS-25686) 后，在 migrate 中也用上了 maintenance cid，是借助 isolated 来实现的，isolated 包括 maintenance 和 failslow。

维护模式的 chunk 上的副本存在以下 2 种情况：

- 被修改的数据，一定需要恢复。离线节点上的数据已经不再是有效的最新数据，这种类型的数据恢复如果希望减少会是一个比较大的结构性改动，暂时不会考虑；

  > 如果在维护期间 Extent 上**没有发生写请求**，则 Meta 在检测副本状态时，可以识别到当前节点正处于存储维护模式中，属于预期内的离线，不会因为副本失联而触发数据恢复，在 Chunk 退出存储维护模式后直接恢复到预期副本数，也不会触发相关数据恢复

- 未被修改的数据，默认触发恢复的逻辑是超时（默认 10 分钟）没有上报数据健康就会认为数据需要恢复。在节点进入维护模式后，如果一个数据的损失的所有副本都是维护节点上的失联副本，则不会触发恢复；

  - 如果数据的当前总副本数小于期望（发生 IO 剔除），会触发恢复；
  - 如果数据的所有失联副本中有部分不在维护模式的节点上，会触发恢复；

  > 如果在维护期间 Extent 上**发生写请求**，由于当前副本不是最新的，因此需要进行数据恢复，但 ZBS 将通过使用更小恢复单元的方式，将恢复粒度从 Extent 变为 Block，从而减少数据恢复量（敏捷恢复）。
  >
  > ZBS 发现此副本处于存储维护模式中，会进行如下操作：
  >
  > 1. 先将此副本从有效副本中剔除，将相关的写请求内容暂时记录在 Access 的内存中，并正常完成写操作，过程中不会触发数据恢复；
  > 2. 当 Chunk 退出存储维护模式后，Meta 会再次检测到此副本信息，此时会触发数据恢复，将维护期间在内存中记录的写操作数据恢复到原始副本中。

维护模式和目前的节点状态可叠加，处于维护模式的节点可能是健康的，也可能是失去连接的。

维护模式会在集群中持久化，即维护模式过程中即便集群重启，也不会改变节点的维护模式状态。并且维护模式只能由用户动作触发变化，不会超时自动退出维护模式。

集群中最多仅能有一个节点进入维护模式，进入维护模式后，所有失联的数据依然会展示在待恢复数据中，只是不会真的触发恢复。在节点状态恢复正常后，将自动的从待恢复数据中清理。

敏捷恢复设计文档，https://docs.google.com/document/d/1JZ6trjE_D1ewfWbaSuewPoFzUksbfno8AyS1wj2Lkio/edit



策略类梳理（seq means prior）

1. 通过 rpc 显示指定的 recover cmd（这个不一定要支持）

2. 周期性扫描产生的 recover cmd

    待做 [ZBS-21199](http://jira.smartx.com/browse/ZBS-21199)，支持设置允许 recover 的时段，不在该时段内仅做 partial recover。在判断每个 pid 是否需要 recover 时，每个 pid 拿到 pentry 的时候就可以判断如果期望副本是 3 而目前副本是 2 时，不用触发 recover。

    往 UpdatableRecoverParams 中增加 2 个字段，start hour  end_hour。同时，通过这个 patch 改 recover 的触发策略，IsNeedRecover。

3. 预期内的节点下线

    ReGenerateMigrateForRemovingChunk()

    待做 [ZBS-21443](http://jira.smartx.com/browse/ZBS-21443)，节点移除还要考虑其他节点的负载情况。

    因为节点移除时，他上面的副本需要尽快迁移完，否则不会执行下一条命令（zbs-deploy-manage meta_remove_node < storage ip>）。从存储池中移除节点的时候，仅考虑 migrate dst cid 在不迁出副本的情况下是否可以容纳待迁移的数据，但这样可能导致 dst cid 超高负载或满载后，副本还没迁移完。

    例如节点 a b c d ，存在大量 extent 的 3 副本分布在 b c d，此时如果移除 c 上的 2 副本，那么只能向 a 迁移，a 如果容量较小，可能会被 c 来的数据填满。

    这个过程中，当 a 进入高负载，可以考虑将部分副本移出，以腾出空间给 c 要移动过来的副本。需要确认一下 a 在作为 migrate dst 且进入高负载时，是否有机会把自己可迁移的数据迁移出去。

    要在这个逻辑里加上，到了超高负载时，允许在 removing chunk 的过程中先执行 replace_cid 为 a 的 ReGenerateMigrateForBalanceInStoragePool

4. 通过 rpc 显示指定的 migrate cmd

    待做 [ZBS-20993](http://jira.smartx.com/browse/ZBS-20993)，允许 rpc 触发 migrate 命令，应该可以和预期内的节点下线合在一起做，因为他们的优先级都会更高，需要马上看到迁移效果

    可以对外做 2 个接口，一个是 pid 为粒度的，一个是 volume 为粒度的（MigrateForVolumeRemoving）

    需要支持 prior volume 吗？

    

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

     看起来得单独另其一个策略函数，在 migrate for rebalance 之后，以 pid 为粒度去遍历，仅靠以 cid 为粒度的两两匹配做不到这个。

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



1. 改 Prefer Local / TopoAware / Localized 三个比较器名字，[ZBS-25802](http://jira.smartx.com/browse/ZBS-25802)



如果都给了 topology 且两个副本的 zone distance, topo distance 都相同的情况下，LocalizedComparator 和 TopoAwareComparator 区别在于：

1. 前者只有在 owner = 0 时，选第一个 cid 时会用 recover_comparator，否则用 ring id 比较；
2. 后者全部用 recover_comparator 比较，也就是先 recover cmd num 再容量。

就一个副本并且跟 prefer local 不在一个 zone，那么 recover_prefer = 剩下的这个副本。如果就一个副本，并且跟 prefer local 在一个 zone，又会怎么样呢？

comparator->UpdateChunkSet 这个地方，如果还剩的 2 副本并不符合 topo 安全，那么是会按照放进去的第 2 个副本来选择后面的第 3 个副本。

只要 topo 不变，符合拓扑安全的副本位置也不变，LocalizedComparator 得到的排序结果是不变的，因此虽然选样本是连锁反应，但就算第 2 个副本位置不对，第 3 个副本位置还是正确的。

```c++
LOOP(candidate_dst_chunks.size()) {
        LOG(INFO) << "yiwu candidate_dst_chunks " << i << " " << candidate_dst_chunks[i].id();
    }

LOOP(comparator->cids.size()) { LOG(INFO) << "yiwu push in cids " << comparator->cids[i].first; }

LOG(INFO) << (all_in_same_zone ? "all_in_same_zone" : "not");
LOG(INFO) << "all_zone_idx " << all_zone_idx;
LOG(INFO) << "prefer_zone_idx " << prefer_zone_idx;
```



实现命令行

reposition 跟 recover migrate 分开、额外添加 rpc 用法

1. 能够观察 recover 真正 IO 的数据量，block 粒度的（比如如果有敏捷恢复，这个 pextent 就不会恢复 256 MB）
2. 能够查看 generate/pending_recover 的数量
3. 能够查看 need_migrate 的数量



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
   
   jiewei 的想法是如果满足上一轮的 recover cmd queue 是满的，且当前这轮是空的，那么在下一个 4s 就触发扫描，而不需要等 5 分钟，这样来保证能喂满 chunk。





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

在 lsm 侧如果超过 chunk_lsm_recover_timeout_sec = 10 min， recover 都没有结束，会主动通过心跳上报给 meta，meta 通过将对应命令的 start_ms 设置为 1 来让该命令超时去除， AccessManager::HandleAccessRequest()



敏捷恢复为减少内存使用，是有单次最大数量的限制。不过 100G 的写盘应该不会触发这个上限。

调查为啥升级时触发的敏捷恢复数量不及预期可以从维护模式时是否 lease 没清空的角度出发调查。



缓存击穿后，大量的恢复任务争抢 IO，恢复任务容易超时被取消，导致实际恢复速率不足 10MB/s。副本恢复的并发度默认是固定值 32，应该作为一个自适应缓存命中率的值

改进迁移策略，在节点上待回收数据较多时（已经使用的数据空间占比超过 95%），如果集群没有进入极高负载状态（整体空间分配比例达到 90%），不向该节点迁移数据以保证回收顺利进行。

lsm1 回收空间的速率非常慢，所以如果删除一个 extent，存在 chunk 的 provisioned space 会减少（分配数据空间比例在减小），但 used space 可能仍然很高的情况，如果这时集群上的其他节点向它迁移数据，会进一步降低回收速率。

在 Migrate 时进行检查，如果集群整体尚有可用空间时比如整体 provisioned 比例在 90% 以下，不向 Used Space 比例大于 95% 的节点迁移数据，即便 Provsioned 比较低。



ZBS-20993

允许 RPC 产生 recover cmd，考虑到如果 need recover 的 volume 很多，那么可以优先恢复某些卷的

所以先知考虑允许 RPC 产生 migrate cmd，然后做 volume 级别的，允许有 replace 和 dst 的偏好，但不保证严格执行。



那还得允许查询哪些还在执行中



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



现有负载的计算是只算 partition 的已用比例。



recover manager 中 recover 和 migrate 的不同之处：

1. migrate 和 recover 只是共用 RecoverCmd 这个数据结构，各自的命令队列（recover 是 std::set，migrate 是  std::list）、触发时机、选取 dst/src 的时机并不相同；
2. recover 的那个扫描只是一个非常浅的过滤 extent，分配 src dst 是在下发阶段，migrate 的 src dst 在扫描阶段就定下来了，下发阶段最多根据 lease owner 改一下 src
2. recover 是可以跨 zone，topo 降级的，但是 migrate 在 2 : 1 的情况下不会有跨域 migrate，且 migrate 需要满足 topo 安全



1. 后续可以改进容量均衡迁移中 replace chunk 和 dst chunk 1 1 配对，可以改成尽可能让多个 src_cid 参与进来，除非所有 under chunk 都不行，才退出循环。
2. http://gerrit.smartx.com/c/zbs/+/54622 patch 7 要拆分成几个小 patch
2. 现有代码在 EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，但如果 src_cid 跟 dst_cid 跨 zone，或者是 failslow，（他可能没有 cmd quota 了，不过这个条件在生成时是硬性规定的，派发时倒不用）这么选还是好事吗？
2. 目前 migrate 中的逻辑是每次获取一个 pid 的 entry 都要通过 GetPhysicalExtentTableEntry 调用一次锁，但在 p rior  extent 的迁移中，可以批量获取 diff_pids 中所有的 pentry，因此可以相应做优化。
2. 机架 A 有节点 1 2 3 4，机架 B 有节点 5 6 7 ，normal extent 容量均衡会去算一个 avg_load，B 上的节点负载都大于 avg_load，A 上的都小于 avg_load，5 容量不够了，只能往 1 2 3 4 迁，但是他们都在 A 上，由于 topo 降级所以都没法迁。改进使得 5 可以向 6/7 上迁。
2. scan_extents_num 应该放在 ReGenerateMigrateForUnevenVolume 这个函数中去做统计，虽然实际上 all_pool 中就一个，且一次只会因为一种方式去 migrate(本地化/容量均衡)。
2. GenerateRecoverCmds 里面的 src 也可以有选择策略的，目前的写法太乱了。



如果节点上的 replica 发生 cow 的话，direct_prs 会瞬间减小而导致写入因为等待空闲 cache 而阻塞，也需要预先下刷以避免阻塞

pin 的 allocation 中的空间计算方式，以及初次写的 lease，怎么传递 prioritized 给 lsm



FunctionalTest::SetUp()  --> new MiniCluster(kNumChunks);

gtest系列之事件机制

“事件” 本质是框架给你提供了一个机会, 让你能在这样的几个机会来执行你自己定制的代码, 来给测试用例准备/清理数据。gtest提供了多种事件机制，总结一下gtest的事件一共有三种：

1. TestSuite事件：需要写一个类，继承testing::Test，然后实现两个静态方法：SetUpTestCase方法在第一个TestCase之前执行；TearDownTestCase方法在最后一个TestCase之后执行。
2. TestCase事件：是挂在每个案例执行前后的，需要实现的是SetUp方法和TearDown方法。SetUp方法在每个TestCase之前执行；TearDown方法在每个TestCase之后执行。
3. 全局事件：要实现全局事件，必须写一个类，继承testing::Environment类，实现里面的SetUp和TearDown方法。SetUp方法在所有案例执行前执行；TearDown方法在所有案例执行后执行。

例如全局事件可以按照下列方式来使用：除了要继承testing::Environment类，还要定义一个该全局环境的一个对象并将该对象添加到全局环境测试中去。原文链接：https://blog.csdn.net/ONEDAY_789/article/details/76718463



1. 我需要整理一下 prefer local，lease owner 的变更情况，怎么生成，怎么变更，怎么释放的。

   Meta 的 Lease 过期策略，出于性能的考虑，Meta 未将对外授权的 Lease 信息持久化在 MetaDB 中。因此新的 Meta Leader 无法知晓上一任 Leader 分发的 Extent Lease Owner 是谁 ，因此它在开始服务之前需要确保之前授予的所有 Lease 都被清空，重新由自身进行授予。此时如果有一个 Access（Chunk） 失联，为了确保已经失联的 Access 将所有从上一任 Leader 中获得的 Lease 丢弃，需要等待失连的 Session 一定超时（ **12 s**）, ZBS 5.2.0 之后调整为 **7s**

   分配 lease 代码，AccessManager::AllocOwner、GenerateLease、

2. 把 prior 快照 + IO 的 functional test 单测流程记下来，便于后续排查问题

   块存储对外接口并不多，一般就是快照/克隆之后的 COW 让问题变复杂，性能变慢

3. lsm 测跟 dongdong 的聊天内容

   meta 侧空间计算中的字段含义，
   [快照/克隆对空间参数的影响](https://docs.google.com/document/d/1oOZ6CENaLFBU_AG6tZ4nnxv1CFUNvv3ND_NWVGVN2PY/edit#heading=h.x0vh71hjzfds)

4. migrate 这段时间的跟 zhiwei 的聊天，smtxos 和 pin test 频道中的 case 整理

   目前遇到的高负载下不迁移：要么 topo 降级了，要么 lease owner 没释放



zbs 当前行为：

zbs chunk 将每小时 io 大于 3600（iops > 1）的 zbs volume 视为 active volume。
zbs chunk 每隔 1h 上报一次 active zbs volume 的信息给 zbs meta，不会上报 inactive volume。
对于一个 zbs volume，zbs meta 收到 6 次 volume 信息上报后（即 6h 后），会更新其 prefer cid 字段，同时更新其所有 pextent 的 prefer local 字段。



允许在 server san 模式下人工指定 access point，融合模式下，接入点总是快速的自动切换至本地，因此对于融合模式下的 Target 执行对应的配置不会产生期望的效果。

根据 access point 去变化 prefer local 的，MetaRpcServer::ReportVolumeAccess

当一个共享卷被大于等于 3 个接入点同时访问时（Xen 平台的 iSCSI DataStore 模式，Xen 将一个 LUN（Volume） 作为一个池，不同节点上的 VM 将仅访问其中的部分数据） 。将不会触发 Local Transfer 事件。相关的 Extent 会保持初次写入时的接入节点作为 prefer local

在副本分配策略 in ZBS 中搜 prefer local。

每个数据链路会尽量分散给不同的 Access 处理，尽量避免链路资源竞争（在这里就可以把数据链路理解成 access point）

https://docs.google.com/document/d/1rcpxCZDNb7YFnEYVIJg-WzZVCzI8rArTFuzLNjJLNhc/edit#heading=h.gye9t51u3igb



为 Volume 增加 access point （cid 列表）属性， Extent 增加 Prefer Local （cid）的属性，用于表示数据的访问点。



iscsi access point 3 部分策略：iscsi 建立连接、异常重定向、动态平衡

修改 iscsi 接入点选择策略：双活集群下，Target 开启外部接入模式时（非 qemu 发起的 iscsi 连接），如果主可用域至少有一个节点可用，则必须选择在主可用域中的节点作为接入点。如果主可用域没有可用节点，则返回次级可用域的接入点。

修改 iscsi 接入点平衡策略：（每 3 分钟执行一次接入点平衡检查，每次检查最多移动一个接入点）
平衡策略目标是：使得每个接入点的数量在主可用域的节点之间内尽可能平均，次可用域内 iscsi 接入点数量应当为 0。如果 iscsi 接入点已经在主可用域上，即使主可用域所有节点都宕机，也不允许主动迁移和修改 access record 到次可用域。客户端发起重试后，会返回次可用域的临时接入点， access record 仍然在主可用域。

临时异常回切：对于临时分配到次可用域的接入点，一旦主可用域有一个节点恢复，且该节点对应的 access session 存活超过一定时间（3分钟），则自动平衡检查时应当尝试将其迁移回主可用域。

https://docs.google.com/document/d/1t14uKF6YCaijgXAq-bS-WR_I1SaLhYxbOnKXhspBtlQ/edit#heading=h.iidguj2la1



待整理



unmap 是针对精简配置的存储阵列做空间回收，提高存储空间使用效率，应用于删除虚拟机文件的场景。VMware 向存储阵列发送 UNMAP 的 SCSI 指令，存储释放相应空间。

https://blog.51cto.com/xmwang/1678350

TRIM 是一种由操作系统发出的命令，用于告知 SSD 哪些闪存单元包含已删除文件的数据，可以被标记为空闲状态

SSD 从不像 HDD 那样直接将新数据覆盖写入旧数据。在所有存储单元都被擦干净之前，无法用新数据对存储单元进行写入，且擦除必须在块级别进行，而写入则在页级别（更小的粒度）进行，这意味着对 SSD 进行写入比擦除要快得多。

- `discard` 是一个由操作系统在删除文件时自动发送给 SSD 的命令，它是实时执行的。
- `fstrim` 是一个由用户手动调用的命令，用于释放整个文件系统中的未使用空间，也可以被自动调度为定期任务执行。



C++ 中为减少内存/读多写少的情况，可以用 absl::flat_hash_map 代替 std::unordered_map，https://zhuanlan.zhihu.com/p/614105687

C++ 中 map 嵌套使用，vector 添加一个 vector 中所有元素 https://zhuanlan.zhihu.com/p/121071760

stl 容器迭代器失效问题，https://stackoverflow.com/questions/6438086/iterator-invalidation-rules-for-c-containers

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610

git submodule ，https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97，https://zhuanlan.zhihu.com/p/87053283



对于 cmp

1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false



vscode 中用 vim 插件，这样可以按区域替换代码

一个遗留问题是，单测里面想要触发两次 recover cmd，怎么让 entry 的 GetLocation() 得到及时更新，试了 sleep(9) 不行，可能不止需要一个心跳周期，还有其他条件没触发。

以一个 functional test 单测为例子展开看 zbs 系统的启动流程。
