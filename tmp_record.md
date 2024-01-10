1. migrate / recover dst 要根据 ever exist = true 否 false 分别从 alive loc 和 loc 中选；

   git stash save zbs2，On ZBS-26732-2: filter stale reposition cmd before distributing

2. even migrate 暴露出来的还有 2 个问题；

3. prior migrate 设计；

一步一步来，最终可以考虑重写个 reposition manager，里面有把 cap replica， cap ec shard, perf replica 做成 3 个类。 但在此之前，需要先把 3 个 migrate 弄成统一的接口，这样才能一步步演进，让所有的 migrate 能共用一个 GetSrcCidForReplicaMigration。

1. ec migrate 目前的做法是 src_cid 一定等于 replace_cid，所以需要避免各个 ec migrate 的 replace cid 选 not healthy status/state 和 isolated 的 cid，等 ec access 支持用恢复的方式来做迁移，这个条件才能放开；

    另外，让各个 replica migrate 中的 replace cid should meet not healthy status/state，要除开 ec migrate；

    在 refactor migrate for repair topo 后，要添加 ec shard 的 replace 一定等于 src，在 healthy 等条件不同时的单测。

    把他跟 src 分开，src

2. refactor migrate for repair topo，从 GenerateMigrateCmdsForRepairTopo 开始改；

    1. 在去掉 GetDstCandidates 时要注意在 migrate for repair topo 中也算上 remain prior space，用它做约束；

    2. 待做 [ZBS-13401](http://jira.smartx.com/browse/ZBS-13401)，让中高负载的容量均衡策略都要保证 prefer local 本地的副本不会被迁移，且如果 prefer local 变了，那么也要让他所在的 chunk 有一个本地副本（有个上限是保留归保留，但如果超过 95%，超过的 部分不考虑 prefer local 一定有对应的副本）

        怎么判断是否会超过 95% 呢？

        如果 volume 的 prefer local 到新 chunk 后（不论是人为运维还是上层虚拟机被迁移到其他节点），现有的迁移策略能让新位置的 prefer local 有副本吗？

        如果不能，在 migrate for rebalance 之后，再有一个 migrate for prefer local，他的目的是保证让 prefer local 有副本，

        [ZBS-25949](http://jira.smartx.com/browse/ZBS-25949) 修改后的 migrate for repair topo 能够达到的效果是不会 replace prefer local，在 prefer local 满足 topo rank 不降级的情况下，dst 会优先选 prefer local，貌似能达到这个效果？双活下也可以吗？prefer local 从 prefer zone 迁移到 secondary zone。

3. refactor migrate for rebalance

    1. [ZBS-26042](http://jira.smartx.com/browse/ZBS-26042) 还缺一个 even volume 的 ut 验证 [ZBS-25847](http://jira.smartx.com/browse/ZBS-25847)

    2. rebalance 时能 recover jiewei 发现的问题，机架 A 有节点 1 2 3 4，机架 B 有节点 5 6 7 ，normal extent 容量均衡会去算一个 avg_load，B 上的节点负载都大于 avg_load，A 上的都小于 avg_load，5 容量不够了，只能往 1 2 3 4 迁，但是他们都在 A 上，由于 topo 降级所以都没法迁。改进使得 5 可以向 6/7 上迁。

        even volume 中的做法应该能实现这个效果，参考即可。

4. 改一下 migrate for even volume 的写法，指的是 vector 变 array ?

5. 在 migrate for prior extent 中引入 remain space map 来正确计算（不一定需要，目前看他好像没啥问题）；

    目前 migrate 中的逻辑是每次获取一个 pid 的 entry 都要通过 GetPhysicalExtentTableEntry 调用一次锁，但在 prior  extent 的迁移中，可以批量获取 diff_pids 中所有的 pentry，因此可以相应做优化。

6. recover / migrate for removing chunk 用到的函数，是否可以拿回 recover_manager.cc 中？

    暂时不需要，拿回来的好处是啥呢？

7. migrate for localization 中有 loose_medium_load_ratio 弹性边界的概念，其他 migrate 中不需要吗？

    检查一下其他 migrate 中是否都像 migrate for even volume 那样允许 4 个 pextent 级别的不均匀。

8. 因为后续的操作不会去操作 even pextent，所以 migrate for even volume 执行完，后续可以接着执行后续 migrate ，但开了分层后的 migrate for over load prior extent，假设分层之后的状态稳定，那 prior extent 作为 perf thick extent，也会参与后续的 migrate for rebalance 平衡，那好像就支持双活了

9. 在分层升级过程中，prior extent 还属于 cap，所以可能还是得保留，即使是升级之后，他属于 perf，也得让 perf thick extent 的优先级在所有 perf extent 里最高，所以还是得保留一个独立的 migrate 策略，因为算他的负载跟算 perf extent 整体的负载并不一致，如果他两在一次里触发的话，可能会有冲突；

10. migrate 策略复杂的地方在于代码写的太面向过程了，已经要做面向对象抽象的，只把 MigrateFilter 抽象出来；

11. migrate for localization

     1. 针对 ec 的 best topo distance 的计算，斯坦纳树问题，对应的 DP 做法 Dreyfus-Wagner 算法。yutian 说最新做法是从拓扑学的角度出发的，现有代码已经是最优解而非近似计算了
     2. 重构 recover manager 时，需要考虑 prior extent 不需要在低负载下支持局部化，master 分支上目前还是支持，但 5.5.x 已经不支持了



命令行相关

1. zbs-meta chunk list_pids，显示所有 chunk 的更细粒度的空间显示，把各个 pids 和他们的 space 显示出来，包括有关 reposition cmd 空间大小；

    zbs-meta chunk list 基本上把信息显示出来了，或者后续需要添加的，也应该放在那里。

    我应该针对 reposition 去设计一个 rpc 去显示详细的信息，放在 zbs-meta reposition 里

    zbs-meta reposition list_pids < cid>，如果传入了 cid，那么显示详细的他有哪些 pid，否则只显示一个 pid num，这样也就把第二个功能做了。

    只显示 src / replace / dst / prior_rx 的 pids 以及他们的 space，然后 cap / perf 都展示

    zbs-client-py 侧等待统一添加

2. zbs-meta chunk list_pid < cid>，看指定 chunk 持有哪些不同种类的 pid，除了 ip + port 还要支持直接给定 cid；

    对应 rpc ListPid，这个可以考虑补充下显示 thin/thick 个数，以及 reserve_pids 这样的，涉及到跟以往的兼容，这边先不修改

3. zbs-meta migrate < volume id> <replace_cid> <dst_cid>，尽量从 replace_cid 上移除，并尽量放到 dst_cid 上，不保证严格执行；

    用于卸盘或其他临时移动卷到指定 dst，之后被 doscan 回去也没事，但如果这个要迁移的卷很大，无法快速完成就被 doscan 回去呢？

    配合关闭迁移扫描再执行这个指令，可以达到临时移动 volume 的效果，迁移想要迁移的部分，不过这样或许得在入口处把人工触发迁移和周期性系统自动触发迁移做一下简单的区分。

    这是要让 pid 进入每个判断逻辑吗？因为没法像 recover list 那样可以直接放到 waiting list 然后生成 recover cmd，还是需要按照负载把这个 pid 放到各个子策略中。

4. zbs-meta recover < volume_id> 想让这个 volume 优先被 recover；

    当有多个 volume 需要 recover，耗时太久时，可以优先 recover 指定卷上的 pextent

5. zbs-meta recover set_runtime <start_hour> <end_hour>

    默认是 [0, 23]，左右闭区间，参考 taskd 中的实现，[ZBS-10973](http://jira.smartx.com/browse/ZBS-10973) 

    副本是期望 3，剩余 2 要恢复，EC 则是期望 m >= 2，允许丢失 m - 1 个 shard 

    是否也考虑做一个 zbs-meta migrate set_runtime <start_hour> <end_hour>，否则在业务高峰期带宽也有可能被 migrate io 抢占，但是 repair topo 应该是不希望关闭的吧？

    往 UpdatableRecoverParams 中增加 2 个字段，start hour  end_hour。同时，通过这个 patch 改 recover 的触发策略，IsNeedRecover。

    单测要写在 function_test 中



做一次冲突检查，合法且和当前恢复不冲突，prefer local 和 topo 相关的不管。

ZBS-20993，允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。



1. 把 avail cmd slots 提前算好放 exclude_cids
2. 让 cli 可以看到 avail cmd slots
3. 把 distributeRecoverCmds 中的生成部分函数抽出来
1. concurrency params 用起来
2. 自动调节 recover / migrate 变速，智能模式中，值变化的时候添加 log
3. summary recover perf 
6. io metrics 调整
    1. LocalIOHandler 中的 ctx->sw.Start() 应该放在所有会执行 LocalIODone 前？
    2. METRIC_INITIALIZE 中的 args 是怎么用起来的呢？
    2. 目前 Acccess IO Stats 中的统计是 app io 的流量，在 access handler 的调节中，后续要考虑 sink io，先不用做 recover io metrics；
    2. 在 access io handler 中做的 UpdateIOStats，对外展示有好处，但实际上没有流量，自动调节的话，可以忽略这部分。
5.  io 分成 app io, recover io 和 sink io 共 3 种，粗略理解，sink io 保性能，recover io 保安全，然后他们在不同场景下的优先级应该不一样：
    1. 比如 app io 流量小的话，应该让 recover io 高，sink io 小一些；
    2. 比如 app io 流量大的话，或许是可以允许 sink io 高，但是 recover io 得小一些这样的
6. 分层之后，cap 层还可以统计盘的数量，perf 层需要统计的是 perf space used rate
7. 需要拿到 sink io metrics 的统计



recover 自动限速调整

```c++
// 并发数提升条件：

// 用 app io metrics 比较
// 升速条件：
total_iops > limit.normal_io_busy_iops_throttle || 
total_bandwidth > limit.normal_io_busy_bps_throttle

// 降速条件：
recover_handler_.migrate_throughput_in_last_duration() > 
migrate_speed_limit * kRepositionIOPercentThrottle
```

recover > sink > migrate

分层之后，io 分成 app io, recover io 和 sink io 共 3 种。其中，app io 优先级最高，sink io 保其它副本的性能，recover io 保副本安全，在不同场景下的优先级应该不一样

1. 若 app io 流量较小，此时业务不应该让 recover io；
2. recover 的默认值是上限的 0.2，migrate 是 0.1；



存储分层模式，可以选择混闪配置或者全闪配置，其中全闪配置至少需要 1 块低速 SSD 作为数据盘，混闪配置至少需要 1 块 HDD 作为数据盘。

存储不分层模式，不设置缓存盘，除了含有系统分区的物理盘，剩余的所有物理盘都作为数据盘使用，只能使用全闪配置。

smtx os 5.1.1 中不论存储是否分层，都要求 2 块容量至少 130 GiB 的 SSD 作为 SMTX OS 系统盘（含元数据分区的缓存盘），为啥要 2 块做软 raid 1？



在恢复或者迁移任务结束时，新加入副本的状态被设置为未知，需要等待下一次心跳周期 LSM 上报副本后才可以确认副本为健康？allocation 的逻辑是马上会被设置为活跃副本，参考 Commit -> PersistExtents -> UpdateMetaContextWhenSuccess -> SetPExtents。



VIP 设计文档，https://docs.google.com/document/d/1M34zaIje2xkUSv9Q41waRH4GCPZ5yv7hwZqOK77vCq8/edit#heading=h.feb6l5x4y4vk

双活设计文档，https://docs.google.com/document/d/1z2cUXLrQ7pZnkJXiCPCLp-BPxkpZCSFIwrmvDxrUYX4/edit#heading=h.rxadnjfqdyav



rx_pids -> dst_pids，tx_pids -> replace_cids, recover_src_pids -> src_pids

那这里还有两个问题：

1. 卸载 partition 盘的时候，chunk 和 meta 分别会做哪些校验，分别用的哪个字段；
2. 卸载盘（或者拔盘）之后，可能会出现 allocated space > data capacity，进而导致数据迁移受到影响。meta 能否在集群数据恢复完成之后，保证 allocated space <= data capacity 呢？



策略类梳理（seq means prior）

1. 周期性扫描产生的 recover cmd

    待做 [ZBS-21199](http://jira.smartx.com/browse/ZBS-21199)，支持设置允许 recover 的时段，不在该时段内仅做 partial recover。在判断每个 pid 是否需要 recover 时，每个 pid 拿到 pentry 的时候就可以判断如果期望副本是 3 而目前副本是 2 时，不用触发 recover。

    往 UpdatableRecoverParams 中增加 2 个字段，start hour  end_hour。同时，通过这个 patch 改 recover 的触发策略，IsNeedRecover。

2. 通过 rpc 显示指定的 migrate cmd

    待做 [ZBS-20993](http://jira.smartx.com/browse/ZBS-20993)，允许 rpc 触发 migrate 命令，应该可以和预期内的节点下线合在一起做，因为他们的优先级都会更高，需要马上看到迁移效果

    仅支持 volume 粒度的就好（MigrateForVolumeRemoving）

    需要支持 prior volume 吗？

    可能需要把 std::list < RecoverCmd> 改成 hashmap

3. 低负载

    ReGenerateMigrateForLocalizeInStoragePool()，让副本位置符合 LocalizedComparator

4. 中高负载

    中高负载目前实际上的区别仅在：

    1. 中负载每 1h 扫描一次，高负载每 5min 扫描一次；
    2. 中负载不移动 local 和 parent 的 pextent，高负载会移动；

    如果 ReGenerateMigrateForRepairTopo 生成了 cmd，那么只生成这个目标的 cmd，否则试图去生成 ReGenerateMigrateForBalanceInStoragePool 的 cmd。需要对 ReGenerateMigrateForBalanceInStoragePool() 改进，先保证都有本地副本，再去做容量均衡。

    待做 [ZBS-13401](http://jira.smartx.com/browse/ZBS-13401)，让中高负载的容量均衡策略都要保证 prefer local 本地的副本不会被迁移，且如果 prefer local 变了，那么也要让他所在的 chunk 有一个本地副本（有个上限是保留归保留，但如果超过 95%，超过的 部分不考虑 prefero local 一定有对应的副本）。

5. 超高负载

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

2. ZBS-20993

3. ZBS-21199



改 Prefer Local / TopoAware / Localized 三个比较器名字，ZBS-25802

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



[ZBS-13059](http://jira.smartx.com/browse/ZBS-13059) 恢复数据允许识别原有数据块的冷热属性

[ZBS-25386](http://jira.smartx.com/browse/ZBS-25386) 修复节点数据迁移速度过慢导致access层无法迁移成功的问题

[ZBS-24563](http://jira.smartx.com/browse/ZBS-24563) 缓存命中率下降，副本恢复任务并发度过高，导致恢复任务执行过慢

4 个 ticket 4 件事

1. 冷热数据识别（梳理 lsm 侧的需求给到 lsm 同学做，识别冷热数据）

   recover src 读的热数据要写到 recover dst 上的 cache，冷数据直接写到 recover dst 上的 partition，避免恢复导致的缓存击穿

   需要修改 data channel 中的 message 中 max_message_id 的语义，换成 usercode，然后就可以带上返回的数据是否冷热的 flag

   参考 patch ZBS-21288

2. recover 每台 chunk 上执行的并发度默认 32，根据 recover extent 完成情况向上向下调节（auto mode）

   并发度最小应该是 2 4，而不是 1，就 1 个通道的容错率太差了

   recover extent 完成情况应该是本地的，从 lsm 侧获取到信息。

   这里权衡 Chunk 的负载指标可能采用的有：1）Chunk LSM 是否 Busy，即 LSM 的 IsLSMBusy，根据当前 Journal 可用数量来判别。2）Chunk Node 的各项性能指标，包括 cpu/memory/disk/network，见 node_monitor，但是现在的实现貌似只有全局的指标，例如 cpu 是所有的核心 summary，disk 不好区分是哪个盘需要做读写。不大好设置阈值。3）根据 Recover IO 与 正常 IO 之间的比例来判定，

   ZBS-25386 有描述一个当 lsm 读取速度过慢时，会导致大部分迁移命令超时的问题，因此并发度也可以根据 lsm 读取速率来调节

3. recover cmd slot 默认 128，根据 extent 的稀疏情况以及 recover extent 的完成情况向上向下调节（auto mode）

   怎么判断 extent 的稀疏情况？lsm 上报的心跳中有 thin pid 的信息

   recover extent 的完成情况可以是 meta 侧统计的下发出去的命令在单位时间内的完成数量

   jiewei 的想法是如果满足上一轮的 recover cmd queue 是满的，且当前这轮是空的，那么在下一个 4s 就触发扫描，而不需要等 5 分钟，这样来保证能喂满 chunk



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

现有负载的计算是只算 partition 的已用比例。



recover manager 中 recover 和 migrate 的不同之处：

1. migrate 和 recover 只是共用 RecoverCmd 这个数据结构，各自的命令队列（recover 是 std::set，migrate 是  std::list）、触发时机、选取 dst/src 的时机并不相同；
2. recover 的那个扫描只是一个非常浅的过滤 extent，分配 src dst 是在下发阶段，migrate 的 src dst 在扫描阶段就定下来了。不论是 recover 还是 migrate，下发阶段都会根据 avail_cmd_slots 过滤命令，另外在放到 session 的 queue 之前还有可能根据 lease owner 改 src，被 space 过滤
2. recover 是可以跨 zone，topo 降级的，但是 migrate 在 2 : 1 的情况下不会有跨域 migrate，且 migrate 需要满足 topo 安全
2. 理论上 migrate 也应该把 generate 和 distribute 合在一起，但 generate migrate cmd 相较于 migrate 策略  更复杂，计算复杂度更高，如果合在一起会卡很久。
2. chunk removing 要求尽快迁移，所以不考虑 topo 安全，跟 recover 用的同一个选择策略



避免某些 pid / volume migrate 迟迟不完成造成集群整体的 migrate 阻塞：

* 以 pid 为粒度扫描的，用 next_xxx_scan_pid_map 来避免让某些 pid migrate 影响到整体，next_xxx_scan_pid_map 保留现状，他其实不需要 generate_cmds_per_chunk_limit_，但为了便于理解，还是加上吧；

  migrate for removing chunk / localization / repair topo / even volume repair topo

* 以 chunk + pid 为粒度扫描的，用 randint + generate_cmds_per_chunk_limit_ 来避免某些 pid 阻塞导致其他 chunk / pid 没机会 migrate；

  migrate for over load prior extent / balance / even volume balance



剔副本的 4 种情况：

1. sync gen 失败的副本；
2. recover handler 在 SetupRecover 时遇到 lease 提供的 loc 中已经包含 dst cid 且 src cid 的 gen 是安全的；
3. access io handler 在 write replica done 时会剔除写失败的副本；
4. 临时副本重放完会被剔除。



做 migrate for repair topo 和 rebalance 时，需要考虑以 chunk 为粒度的遍历，其他的考虑以 pid 为粒度遍历就好。

xx 1. 不开分层的 replica ，2. 开分层后的 cap replica，3. 开分层后的 perf replica，4. 开分层后的 cap ec，他们的单测不适合合起来，因为如果之后 cap / perf 策略不同的话，还是得拆出来。

涉及到容量均衡的才要区分是否双活，比如 even / prior / normal rebalance



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

   meta 侧空间计算中的字段含义，[快照/克隆对空间参数的影响](https://docs.google.com/document/d/1oOZ6CENaLFBU_AG6tZ4nnxv1CFUNvv3ND_NWVGVN2PY/edit#heading=h.x0vh71hjzfds)
   
4. 目前遇到的高负载下不迁移：要么 topo 降级了，要么 lease owner 没释放，要么是双活只能在单 zone 内迁移

4. 后续写文档可以考虑的组织方式

   1. migrate for removing chunk
   2. migrate for no-removing chunk
      1. migrate for even volumes
         1. migrate for even volumes repair topo
         2. migrate for even volumes rebalance
      2. migrate for uneven volumes
         1. migrate for over load prior extents
         2. migrate for normal (cap / perf) extents ( 1. cap layer 2. perf layer )
            1. normal low
               1. migrate for localization
      
            2. normal mediun / high / very high
               1. migrate for repair topo
               2. migrate for rebalance
   
   
   先介绍所有 sub migrate strategies 中共有的限制条件：
   
   1. failslow
       1. must not be dst;
       2. should not be src/replace in ec migrate;
       3. should not be src, should be replace in replica migrate;
   2. src select 是共有的
   3. 介绍 src / dst / replace 通用的选择策略，然后再分别介绍各个部分中独有的
   4. EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，所以以上选到的 mgirate src 不一定就是最后给到 access 时的 src



遗留问题：

1. access recover read 是 extent 粒度，write 是 block 粒度？
2. 为什么 AllocRecoverForAgile 中一定不会有 prior extent？
3. 在 HasSpaceForCow() 为什么用的是 total_data_capacity 而不是 valid_data_space ？





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



### access point

zbs 当前行为：

1. zbs chunk 将每小时 io 大于 3600（iops > 1）的 zbs volume 视为 active volume。
2. zbs chunk 每隔 1h 上报一次 active zbs volume 的信息给 zbs meta，不会上报 inactive volume。
3. 对于一个 zbs volume，zbs meta 收到 6 次 volume 信息上报后（即 6h 后），会更新其 prefer cid 字段，同时更新其所有 pextent 的 prefer local 字段。



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

### sync gen

access 从 meta 拿到的 lease 中的 location 是 loc 而不是 alive loc，可参考 GenerateLayerLease()，在 sync gen  是对 loc 而不是 alive loc 上每个 cid 都 sync，实际上，让 access 做一下 sync 真正确定一下这个副本是否连通比 meta 给出的信息更靠谱，因为这个 chunk 有可能跟 meta 失联，但还跟其他 chunk 联通，此时的失联 chunk 还是可以被读写副本的。

### remove chunk

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

### 维护模式

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

### thin/thick 分配

分配一个 thick pextent，会马上分配 pid 的 location（此时的第一副本会挑选为分配时集群空间比例最小的节点，其他副本位置再按照局部化原则选择），然后在 transaction 的 Commit 中会让 location 上每个 replica 的 last_report_ms = now_ms，所以此时也马上会有 alive_location = location。

分配一个 thin pextent，直到初次写之前，他的 location 都是空的，所以 alive_location 也为空。

### COW 内容

COW 之后，child alive loc 不一定等于 parent alive loc。实际上，COW 在 Transaction Prepare 的 CowPExtent 时只会只会复制 parent 的 loc，然后在 Commit -> PersistExtents -> UpdateMetaContextWhenSuccess -> SetPExtents 时会将 loc 上的每一个副本的 last_report_ms 设为当前时间，所以 child alive loc = child loc = parent loc，但是不一定等于 parent alive loc。

### 560 空间计算

prioritized_pids 就是 perf_thick_pids，因为 perf 层只会有 perf thin 和 prior 两种类型的 extent，不会有非 prior 的 thick extent

prioritized_rx_pids 是 perf_rx_pids 的子集，一定被包含在 perf_rx_pids，除了在开启分层前的升级过程中产生  prior reposition 的话，prioritized_rx_pids 有部分 pid 是给到 cap_rx_pids 而不是 perf_rx_pids

有如下等价关系：

* allocated_prior_space = prioritized_pids + prioritized_rx_pids
* perf_pids = prioritized_pids + perf_thin_pids_with_origin + perf_thin_pids_without_origin

为啥没有 prioritized_tx_pids 和 prioritized_recover_src_pids ？因为不需要，要有一个单独的 prioritized_rx_pids 是为了在算 allocated_prior_space 的时候能把为 reposition 预留的算进去，而 tx 不加是因为 allocated_data_space 就算多算了也没事，过一段时间完成 reposition 之后 perf_tx_pids 也就被 erase 了。



cap_pids，除了 allocating / repositioning  的 cap 层 pids 都会被记入 cap_pids，cap_pids 一定包含 cap_tx_pids 和 cap_recover_src_pids（但不是仅由他们两组成的），一定不包含 cap_rx_pids ，与cap_reserved_pids 可能会有交集（取决于是否调用了 GetLeaseForRefreshLocation rpc，没调用的话是不会有交集的），也因此，求的 allocated_data_space 可能是略大的，因为有可能某个 pid 既在 cap_pids 又在 cap_reserved_pids。

cap_thick_pids，在 cap 层的 thcik pids；

cap_thin_pids_with_origin，在 cap 层的经过 COW 而来的 thin pids；

cap_thin_pids_without_origin，在 cap 层的没有 parent 的 thin pids；

cap_new_thin_pids，在 cap 层的还没被 LSM 上报真正空间占用的 thin pids；



有如下等价关系：

* cap_pids = cap_thick_pids + cap_thin_pids_with_origin + cap_thin_pids_without_origin

* thin_used_data_space 对应的 pids = cap_thin_pids_with_origin + cap_thin_pids_without_origin - cap_new_thin_pids

* cap_thin_pids_with_origin + cap_thin_pids_without_origin = cap_new_thin_pids + thin_used_data_space 对应的 pids

    > 这是因为 add thin pextent 的时候，要么放在 cap_thin_pids_with_origin，要么放在 cap_thin_pids_without_origin，必选其一。如果这个 thin 是刚创建的，还没有心跳上报真实空间，会被放在 cap_new_thin_pids 里面，如果空间上报了，thin_used_data_space 被更新，这个时间窗口内的 cap_new_thin_pids 被清空。具体看 ChunkTableEntry::UpdateThinUsedDataSpace()



cap_rx_pids / cap_tx_pids / cap_recover_src_pids，这三个分别对应 reposition dst / replace / dst 的空间大小，当有正在下发但还未完成的 repositon cmds 时，他们的值不为 0。

* add：下发 reposition cmd 时会调用 ReserveSpaceForRecover()，然后放入 pid_cmds_；
* delete：在 DoScan 时会调用 CleanTimeoutAndFinishedCmd()，满足 cmd finished or timeout 的条件时就会删除，然后从 pid_cmds_ 中删除；

这个空间大小也体现在 ongoing recover / migrate space，不过它的计算只计算了 src 的空间。算空间大小，不应该减去 cap_tx_pids，因为在这里的 pid 一定还在 cap_pids，并且当 reposition 成功，会有 RemovePextent 或者 ReplacePextent，到那时候会把 cap_pids 里面的相关 pid 去掉。



cap_reserved_pids，对应正在分配但还没分配成功的空间大小，先预留，把这部分空间占住，由于分配副本空间在 transaction 中很快完成，所以可以认为大部分时间 cap_reserved_pids 都是空的。

* add
    1. transaction 里面 CreateVolumeTransaction::Prepare() 会先调用 ReserveSpaceForAllocation() ，预留空间；
    2. 在 sync gen 时如果发现这是一个 parent 被迁移的 pextent，lease 上没有他的 location 等信息，会调用 MetaRpcServer::GetLeaseForRefreshLocation() 来从 parent pextent 中拷贝一份 location 信息出来，此时也会认为这个 pextent 没有 parent 了，那么他也要独立的占用空间，调用 ReserveSpaceForAllocation() 来预留空间，并马上通过 PhysicalExtentTable::SetPExtents() 把这部分预留空间删掉，即从 cap_reserved_pids 中删除，放入 cap_pids；
* delete
    1. add 里的两种情况会调用 FreeSpaceForAllocation()，对应 SpaceTransaction 的析构函数和 GetLeaseForRefreshLocation() 的 done 代码段的操作；
    2. 调用  ChunkTable::AddPExtent() 也会将 pid 从 cap_reserved_pids 中删除，放入 cap_pids，对外的接口是 PhysicalExtentTable::SetPExtents()，而这除了上面那个特殊的 rpc 外，就是在 transaction 中的 Commit 阶段的 UpdateMetaContextWhenSuccess() 中会调用



对于 cmp

1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false



vscode 中用 vim 插件，这样可以按区域替换代码

一个遗留问题是，单测里面想要触发两次 recover cmd，怎么让 entry 的 GetLocation() 得到及时更新，试了 sleep(9) 不行，可能不止需要一个心跳周期，还有其他条件没触发。

以一个 functional test 单测为例子展开看 zbs 系统的启动流程。
