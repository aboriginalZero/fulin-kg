### ChunkSpaceInfo 字段

1. total_data_capacity  = 94% * 所有 HHD 物理容量

   > ZBS 在写入数据时需要额外的空间保存数据的校验和信息以发现静默错误，比例大约为 512 checksum / (8 KiB data + 512 checksum) = 94%

2. total_cache_capacity < 94% * 所有 SSD 物理容量

   > 除了检验和占据数据空间，ZBS 中 SSD 会有部分空间用于装载 OS，ZBS 内部的 MetaData、Journal 与 Cache 自己的元数据都会占用部分 SSD 空间

3. v5.4.x 之后忽略 Used Space 和 Provisioned Space，只关注 allocated_data_space，该值等于以下 3 项累加：

   1. Thick Space：meta 侧在分配 thick extent 时得到的 extent 粒度 thick 占用；
   2. Thin Space：lsm 本次上报的 pblob 粒度 thin 占用 + 未包含在本次上报的 extent 粒度 thin 占用；
   3. RX：meta 侧在 reposition 时为 dst 预留的 extent 粒度占用，不论 thick/thin。

4. invalid_data_space 在 zbs-meta chunk list 中没有显示，可以在 zbs-meta cluster summary 中看到；

5. thin_used_data_space 包含这个 chunk 最近一次上报的 thin extent 总空间消耗，用于在 meta 切换后进行集群 used_data_space 计算，在该 chunk 未再次上报 used space 前也能显示较为合理的集群空间消耗，v5.4.x 引入的字段；

pin 开始在 ChunkSpaceInfo 引入的 3 个参数

1. meta 下发的 planned_prs 是 extent 粒度，值等于 valid_cache_space * prior_space_percentage，valid_cache_space 是 lsm 上一轮心跳汇报的结果，prior_space_percentage 是用户通过 rpc 设置的值；

    > 除了 rpc 更新会立即下发最新的 prior_space_percentage，在日常心跳中每轮也会通过 reserve_prior_space 字段下发给 lsm

2. meta 下发的 allocated_prs 是 extent 粒度，值等于 (prioritized_pids + prior_rx_pids) * 256 MiB；

    > lsm 目前汇报的 allocated_prs = prior_extent_num * extent_size，不管实际数据使用量多少，也不管有没有共享 pblob，不过它汇报的这个值目前还没用，以后想用的话用 Meta - LSM 的就知道哪些空间还没有落到 LSM 上

3. lsm 汇报的 downg_prs 是 pblob 粒度，值等于 (all_prior_pblob - (planned_prs * 1024)) * 256 KiB，不管 prior pblob 是在 normal cache 还是 partition；

    > 由于会有共享的 pblob，所以即使汇报的 allocated_prs > planned_prs，downg_prs 也可能为 0



### ListCacheResponse 字段

zbs-chunk cache list 显示结果

```shell
TOTAL ACTIVE:    768.0K ( 0.00%)
TOTAL INACTIVE:    2.7G ( 5.40%)
TOTAL CLEAN:         0B ( 0.00%)
TOTAL FREE:       38.3G (76.60%)
```

这 4 个字段目前都是普通缓存的统计，prior space 不算在内，因为 lsm 缓存使用 LRU 策略，所以 active 和 inactive 代表是否过期：

* dirty = active + inactive，即 dirty cache space，数据在 SSD cache 但未写入 HDD 时会被标记为脏；
* used = active + inactive + clean，即 used cache space，已使用的 cache 空间；
* total = active + inactive + clean + free，即 cache capacity，可使用的 cache 空间。



在 lsm 视角，extent 只是 pblob 的数据集合，实际承担数据的是 256 KiB 的 pblob。prior pblob 的生命周期：

1. planned_prs 是在 meta 侧的值，也是用户视角的 prior 的空间大小，通过心跳下发给 chunk，LSM 为其在 cache 中预留 planned_prs * 1024 个 pblob 的空间（即 prior pblob，仅预留，未实际使用）；
2. 对 prior volume IO 时，卷内分配 prior extent，meta 下发请求到 lsm，lsm 根据这个 prio extent 实际使用的 pblob 数（extent 未必写满）在 prior pblob 预留空间里去分配 prio pblob；
3. 在 prior IO 一段时间后，如果预留的 prior pblob 空间都写完了，就会用 normal cache，此时 downg_prs 开始从 0 递增；
4. 继续分配 prior pblob 到 normal cache 都被用尽，那只能分配到 paritition 上，在 partition 上的 pblob 也算在 downg_prs 里面。

预期内的 planned_prs 都是小于 valid_cache_space 的，但如果出现掉 cache 盘，会导致 planned_prs > valid_cache_space，在掉盘而 planned_prs 还没更新的这段时间里，prior extent 会分配到 partition，但在一次心跳过后（request 传递 valid_cache_space，response 据此计算 reserved_prior_space 并下发作为 lsm 的 planned_prs）又可以保证 planned_prs < valid_cache_space。

5.5.x 开始，prior extent 在 planned_prs 内的数据会常驻 cache 层，而 normal extent 是保留之前的 IO 行为：

1. IO 到 lsm 侧是先写 cache 层，cache 层的 pblob 会被 writeback 到 parition 层，在 cache 负载率超过 90% 时， writeback 完了之后变成 clean 的 pblob 会被淘汰；

    >  cache 负载率不高的话，writeback 了也不会被淘汰，淘汰的操作就是更改 pblob 的元数据，把 pblob 的 cache_blob_id 设为空，这样数据就不存在于 cache 上了，释放的 cache block 又可以被其他 pblob 重用。
    >  writeback 一直在进行，只是限速会动态改变，当 cache 负载率高的话，writeback 速度会加快，当 APP IO 多的话，writeback 速度会调低。

2. 当 partition 层数据被多次访问后会 promote 到 cache 层，判断条件是 30 分钟内访问 3 次，此时数据在 cache 和 partition 中各有一份。



卸载数据盘，会引发 lsm 的盘间数据迁移，从一个 parition 到另一个 partition，如果剩余的 partition 不够容纳就卸载不掉，除非强制拔盘，那就成丢副本了。



zbs 副本分配策略



prior extent 分配策略



### zbs 副本迁移策略

#### synopsis

chunk 级别的 prior-space 负载分为 3 种，计算方式：

1. 低负载（low_prior_load）： allocated_prs <= planned_prs；
2. 中负载（medium_prior_load）：planned_prs < allocated_prs <= valid_cache_space；
3. 高负载（high_prior_load）：valid_cache_space < allocated_prs；

cluster 级别的 prior-space 负载分为 2 种，计算方式：

1. 欠载（underload）：不存在中、高负载的 chunk
2. 过载（overload）：存在中负载或高负载的 chunk

（以下策略中，数字代表优先级）

1. chunk remove 迁移

    * prio extent

        1. 若 lhs 和 rhs 其中有不具备 prior 能力的，先选具备 prior 能力的，再选 v5.5.0 节点但 planned_prs 为 0，最后选老版本节点，否则说明都具备 prio 能力，执行下一步；
        2. 若 lhs 和 rhs 在容纳一个新副本后都是 low prior，按本地化 + 局部化 + topo 安全策略选；若只有一个是 low prior，选这个；否则说明都无法容纳，执行下一步；
        3. 若 lhs 和 rhs 在容纳一个新副本后都是 medium prior 且 valid cache space - allocated_prs 都大于 40GiB，按本地化 + 局部化 + topo 安全策略选，否则选 valid_cache_space - allocated_prs 更大的；若只有一个是 medium prior，选这个；否则说明都无法容纳，执行下一步；
        4. 选 lhs 和 rhs 中 allocated_prs - valid_cache_space 更小的，选取结束。
    * normal extent
        1. 若集群是中低负载，按本地化 + 局部化 + topo 安全策略选，否则执行下一步；
        2. 若 prefer local 节点是中低负载，按本地化 + topo 安全策略选，否则执行下一步；
        3. 按 topo 安全策略 + 低容量优先选。

    recover 与 chunk remove migrate 的 src/dst 选取逻辑一致。

2. even volume 均匀分布迁移：prio 与 normal 一致，都是让 even volume 卷内 extent 先做 topo 安全，若满足 topo 安全，再做 topo 不降级 extent 数量均衡。

3. prior 超载迁移（存在 medium/high prior chunk）：在保证有足够的性能层和容量层空间前提下，按以下优先级做 topo 不降级节点 prio 负载均衡。

    1. high prio 到 low prior 
    2. high prio 到 medium prior
    3. medium prio 到 low prior 

    如可容纳新副本且非 failslow 的 low prior chunk 有多个，那从中任选一个。

4. normal 低负载迁移（< 75%）：normal 和 prio extent 都按本地化 + 局部化 + topo 安全做迁移，normal 只需要保证有足够的容量层空间，而 prio 需要保证有足够的容量层和性能层空间，并且不引发 prio 超载。

5. normal 中高负载迁移（>= 75%）：先做 prio 和 normal extent 的 topo 安全，若满足 topo 安全，再做 normal extent 的 topo 不降级容量均衡。

6. normal 超高负载（>= 95%）：只做 normal extent 的 topo 不降级容量均衡。

prio 超载迁移的负载均衡终态只要求各 prio chunk 处于 low prio load，不要求 prio extent 在各节点均匀分布。另外，欠载 prio chunk 上的 extent 分布会在 normal 低、中高负载迁移中被调整。

为什么永远要保证 topo 安全是第一位？

数据安全永远大于性能，这是做存储的第一要务。

在几年前出现过副本在高负载下分配到同一个 brick，有一天 brick 背板坏了，这个 brick 上所有的节点都异常，导致部分数据的所有副本都无法访问，停机了一天。性能也好，容量也好，都是可以简单通过规划和告警避免的问题，客户想获得安全可靠的服务可以通过提前规划，重视告警完成。如果拓扑安全不在代码的设计里作为第一位，客户是没有任何办法的。

只有一种情况下允许拓扑降级，就是新数据分配的时候，即便得不到集群理论拓扑的最高分数。因为如果这个时候不允许分配，业务系统也是直接停止，但是集群内部也会不断的通过拓扑安全扫描尽可能的主动发现和修复这种潜在风险。

#### scan extent

低负载

ReGenerateMigrateForLocalizeInStoragePool，没有对是否双活显式区分，都被封装到 LocalizedComparator 里了。

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
//   - enough cmd quota （replace_cid 应该不用考虑命令配额）

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

中高负载，在 topo 安全不降级的情况下，优先选 prefer local，如果 prefer local 不能让拓扑结构更健康，迁移的 dst 是不会选 prefer local 的。

> TODO 可以改进成：先 dst_cid 再 src_cid 再 replace_cid

期望改成

replace_cid 跟 dst_cid 的选择是互相制约的，因为 topo distance better 的可能有多种组合

```shell
replace_cid
// replace cid must meet
//   - worse or equal topo safety than replace cid

// replace cid should meet
//   1. not owner
//   2. not prefer local
//   3. failslow
//   3. lower available capacity

dst_cid
// dst_cid must meet:
//   - not in alive loc
//   - not failslow 
//   - enough remain valid space （这 2 个情况只是限速器导致的，啥都不干，过段时间也清出来了）
//   - enough cmd quota 
//   - better topo safety than replace cid

// dst_cid should meet:
//   1. prefer local
//   2. comparator above (more valid space)

src_cid
// src_cid must meet:
//   - enough cmd quota 

// src_cid should meet:
//   1. not failslow
//   2. same zone with dst_cid （这两个顺序值得商榷）
```

1. 若有 prefer local 且 topo dis 最优，不迁移，否则执行下一步；
2. 若 dst_cid 是 prefer local 能让 topo 不降级就迁移，若dst_cid 不是 prefer local 能让 topo 升级才迁移。



对 src 的选取是通用的，默认为 replace，优先选 

局部化找到的副本其实考虑了双活场景，因为副本分配也用的 LocalizationComparator 这个比较器。



1. 考虑到副本处于 FailSlow 节点上的数据安全风险，要显著高于处于一个正常但是拓扑接近的节点。因此，在选择迁移的目的节点时，选择健康节点的优先级，要高于拓扑安全；
2. 对于源端节点选择来说，如果一次迁移可以提高拓扑安全等级，即使没有健康的源端节点，也应该选择一个 Failslow 节点做尝试



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

其中，thin and origin_id != 0 指的是被克隆/快照的精简制备副本。

#### generate cmd

经过以上步骤，生成的 Recover cmd 只是放入 passive_waiting_migrate 中，在等待 5min/1h 或者 scan_recover_immediate = true 时会 swap(active_waiting_migrate, passive_waiting_migrate)

关于 active_waiting_migrate / passive_waiting_migrate 这两个 std::list ：

1. 只有 RecoverManager::GenerateMigrateCmds() / RevokeRecoverCmds() 会往 active_waiting_migrate 中 erase 元素；
2. 只有 RecoverManager::MakeMigrateCmd() 会往 passive_waiting_migrate 中 push 元素。

关于 active_waiting_recover / passive_waiting_recover 这两个 std::set ：

1. MetaRpcServer::DoRemoveReplica() 移除副本会触发这个 pid 的 recover，调用 RecoverManager::AddToWaitingRecover(pid_t pid) ，它会接着调用 AddToWaitingRecoverIfNecessary() 往 active_waiting_recover 中 insert 元素；
2. RecoverManager::GenerateRecoverCmds() / RevokeRecoverCmds() 会往 active_waiting_recover 中 erase 元素；
3. RecoverManager::ReGenerateWaitingRecoverList() 会 recover pextent table 中所有需要 recover 的 pid，在 DoScan 中被定时执行，它调用 RecoverManager::AddToWaitingRecoverIfNecessary() 往 passive_waiting_recover 中 insert 元素；

#### distribute cmd

在 Scan 的过程中：

1. 通过 RecoverManager::GenerateRecoverCmds() 往 active_waiting_recover 队列中添加 recover cmd；
2. 通过 RecoverManager::GenerateMigrateCmds() 往 active_waiting_migrate 队列中派发 migrate cmd；
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