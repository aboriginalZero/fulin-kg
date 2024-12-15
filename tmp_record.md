超高负载，直接跳过

中和高，需要保持 topo 和本地有副本



在 high load 的时候，分配并不保证会分配到 prefer local 上，但是 migrate for repair topo 会让此时的 prefer local 有副本，有可能导致一分配就迁移。

可以让 IsNeedMigrateForRepairTopo 只保证 topo 安全，如果已经 topo 安全，返回 already_match = true，否则走 IsNeedMigrateForPreferLocal，后者可用空间是到 0.85，前者是 0.95 - 0.05 = 0.9（perf layer 的话区分就很大了）



1. 怎么理解管理网络和虚拟机网络的区别？http://docs.fev.smartx.com/smtxos/6.1.1/elf_installation_guide/elf_installation_guide_21
2. VLAN ID = 0 的含义是什么？



1. 搞一个 RepositionPEntry，这样批量处理 reposition pentry 时能让内存命中率，除了 pentry 还可以有 chunk table 中的信息、lease owner、pid

    ```
    class RepositionPEntry {
        PhysicalExtentTableEntry entry;
        cid_t owner;
        uint8_t is_temporary : 1;
        uint8_t is_parent : 1;
    
        RepositionPEntry(const PhysicalExtentTableEntry& entry, bool is_temporary, bool is_parent) :
            entry(entry), is_temporary(is_temporary), is_parent(is_parent){};
        // 在 access manager 中赋值
        void SetOwner(cid_t cid) { owner = cid; }
    };
    ```

2. 把 RepositionPEntry 中的 pentry 替换成一个只供 recover 使用的数据结构，把 replica_ 开成栈变量，减少拷贝时的耗时；

3. 把批量处理 pids 的逻辑提出 1 或 2 个宏定义，在 7 个子 migrate 中使用。



感知硬件能力做成一个通用的能力，这样 internal flow ctrl / flow ctrl / sink 都能使用。



查看盘类型是 hdd 还是 ssd

```
cat /sys/block/sdb/queue/rotational
```



用 nvmf 测各种类型盘的上限，然后取其 50% 认为是 recover io 可用的上限

同一类型且同一型号的盘，容量不同，磁盘 iops/bps 的上限也不同（比如  P5620 NVMe SSD，1.6 TB 跟 6.4TB 的 4k iops 上限分别是 20w 和 30w）





先按照 pk 再按照 pid

recover cmd 中有了 is_thick 信息，所以 zbs cli 中可以区分 3 种 pk 类型展示（可以考虑在 zbs cli 中对 RecoverHandler::ListMigrateInfo / ListRecoverInfo 的结果排序，3 部分分开展示）

zbs-meta recover / migrate list、zbs-chunk recover / migrate list 都需要修改

1. 只有 lease owner 上的 Total Migrate Speed 才有值，而这也是会给到 meta 的值，才会有 reposition list，其中 STATE = INIT 的 pid 表示在 recover handler 的 pending 队列中，STATE = START/END/READ / WRITE 的 pid 表示正在执行，PAUSE 表示由于并发额度不足被阻塞。
2. From Local Speed 指的是该节点作为本地



已经 sync 过，所以 gen 是有效的



internal io throttle 的位置

1. 如果 pextent io handler 中把 throttle 放在 io 进入 lsm 前，那么需要 InterceptBusinessIO 来保 gen 序，否则可能出现，同一个 extent 的两个 block io。



如果是 special recover，给到 pextent io handler 的 data len 不一定是 kBlockSize。

replica recover src block 如果是 ELsmNotAllocData，block_not_alloc 置为 true，对 recover dst 的写会转换成 unmap



access_stats_->layer_stats 的统计未必准确：

1. data len，特别是  special recover 传递的
2. IsRead() / IsWrite()

影响到前端的展示。先不管，先让 LocalIOStats 准确。

recover write 也有可能 unmap 写，这部分不需要统计进来。

1. access_stats_->layer_stats 中是否需要忽略 ELSMNotAllocData

2. throttle_latency_ns_ 为啥只在 LocalIOHandler::LocalIOStart() 中更新

3. RecoverIOStats::from_local_recover_counter 的更新可以放在 pextent io handler 吗？

4. unmap 形式的 recover write 需要更新吗？

    ```
    layer_common_->UpdateCounter(ctx->replica_pextent_info, &RecoverIOStats::from_local_migrate_counter, ctx->cur_data_len);
    ```



下沉如果读到的是 ELsmNotAllocData，或是全 0 数据，且满足对其要求（replica 4k，ec 编码块对齐），则可以向 cap extent 发送 unmap，以缩减 cap space。





1. 一个 ever exist = false 的副本，在 lsm 上真的存在吗？如果是快照/克隆后被迁移到其他节点的 PExtent，此时虽然还是 non ever exist，但在目的节点上真实存在，其健康状态会被定期上报。
2. vextent no 对 FLAGS_meta_max_pextents 取余就是 lid。
3. lid 与 pid 的 gc 是独立运行的？
5. recover src 也有可能总是选到同一个，此时若 lease owner 与 recover src 网络失联，但 recover src 与 meta leader 是可以正常通信的，会导致 recover 一直无法完成。
6. 后续测试轮转调度是否有效，可以的方式是代码里指定给到 volume  A 的 io 一定带上 recover flag，B 的是 sink flag，然后用 fio 打到这两个 volume 上来模拟多种内部 IO 同时进行的场景，看此时的轮转调度是否有效。 



app write extent，是每写其中一个 block 就对 extent gen 加一次 1

所以处于 recovering 状态的 extent，如果在 recover 结束前有 app write（recover dst lsm 上处于 recovering 的 extent 是不允许被 app read），会对 track_gen ++，这样在 normal RecoverEnd 的 gen verify 才能对得上。

recover write 与 app write 带来的 gen 在 lsm 中是如何变化，access 怎么跟 lsm 校对的





1. sink io 的 dst 在 CheckAndGetLeaseForDrain 之后能知道，但它是从 perf 读，写到 cap

2. reposition io 的 dst 在 meta cmd 中知道，replica src 也可以从 meta cmd 中知道，但是 ec src 只能在最终的 io_res.succ_cids 中查到

3. 为啥 access 中允许并发写同一个 block？

   给到 lsm 内部执行时，有可能在 pblob 层面的并发。

   access 按一个 pextent 的 gen 不降序的方式顺序下发 io co 给 lsm（有这么个保序机制），lsm 收到后内部并发执行 io co，在 lsm 内部，如果这些 io co 写的是同一个 pblob，还是需要加锁，如果是不同 pblob，就可以并发执行。

   

fc 的 intercept io，是在 access io handler 处拦截 perf app io。

cap io throttle，是在 local io handler 处拦截 internal cap io + app cap io。





Flow Controller 运行在 Access Lease Owner 上，控制下发 NeedAlloc IO 的速度；Flow Manager 运行在 Local IO Handler 上，负责分配 Token

1. Data Channel 是怎么实现的保序性质，协调 Token 和 IO 的顺序？依赖 tcp 实现 gen 保序

2. 怎么说更恰当的方式是感知到当前 Recover 的进度？

3. 为啥普通写时，需要考虑到 recover dst，而 recover 写时就可以不考虑走 InterceptIO 接口？

4. 全闪不分层集群不会开启限流，这个在代码的哪里体现？

5. 为了避免内部 Cap IO 抢不到并发度，当 APP Cap IO 较高时，会采用 APP Cap IO 的并发度的一半作为内部 Cap IO 并发度的总上限。这个好像也解决不了内部 cap io 被饿死的问题？

   能不能把 FLAGS_cap_io_depth_limit_app_io 在升级之后开启呢？meta 目前没法感知是否在升级



一个普通 read / write 从 nfs server 到 lsm，过程中经历了多少次线程切换？如果是读取本地和读取远程数据参与的服务线程分别是谁？请求与包含的数据本身是如何在线程间安全的传递的？整个流程有多少次内存复制？ 承载数据的内存是谁申请的又是谁释放的？



Access 在 Sync perf extent 时，从 LSM 获取 perf extent valid bitmap，并以 256k 为粒度组织成一个个 BlockInfo。BlockInfo 也会加入到 BlockLRU。 IO 过程中， BlockLRU 感知 BlockInfo 的冷热。Access 在适当时机下沉冷的 BlockInfo 至 capacity extent。





1. flat_hash_map to btree_map，从内存访问的角度更好
3. cid map 改成 std::vector，这样从内存上更有顺序性



关注 zbs_chunk_cap_io_throttle_migrate_io_cur_io_depth metric 可以作证升级期间的 cap 慢是不是因为 migrate 抢了 recover 的 cap 并发度限制。



1. 从 5.0.5 升级到 5.6.0，感受一下敏捷恢复的触发效率（或者直接找 qe 借个环境）
5. chunk table 改成读写锁



vtable_id 就是 volume_id，vtable_size 就是这个 volume 持有的 lextent 的个数。lextent 先 gc，跟他对应的 cap / perf pextent 才会被 gc

gc 中的 ScanByPidRefs 会拷贝出一个 pid_perf_map 到 gc 线程中，migrate scan 也可以这么做，这样 next_scan_migrate_pid 才会比较准确，避免无效的访问。

这个函数不仅标记了可以被 gc 的 lextent，而且更新了所有 pid 的 ref_status，后续计算 unique_size 和 shared_size 也会有变化

GcManager::ScanAndProcessLExtents

计算 logical_size_bytes 相关的两个函数：GcManager::CountVolumeSize()、PhysicalExtentTable::ScanByPidRefs()



MLAG 集群中不同节点能力有差，有时候升级慢是在重启某个 chunk 后的恢复慢，这种情况下 meta 侧智能调节下发窗口就显得很有必要了。



1. 节点移除迁移中对 migrate src 的选择策略有问题，COW 后没写过的 pexent 迁移过的场景。
2. recover lease owner 上的 access metric 没有值，recover 路径上只对 counter 埋点，没有针对 metric 埋点；




1. perf 和 cap inernal io 除了考虑磁盘能力，还需要考虑他两加起来不能超过网络带宽的 50%，如果只有单层数据待恢复，那应该允许他用满 50%。

    目前的实现里，假设 cap / perf 都在满负载恢复，两边都 500 MB/s，那 app io 可能就抢不到网络带宽了。

2. recover dst 没有优选 topo safety，可能造成 recover 后要立马 migrate。

3. access reposition 的 Counter 改成 metric，否则影响前端展示、metric 使用，检查 recover/migrate speed 在前端界面和 prometheus 中的数值是否准确，meta 侧跟 chunk 侧的 total speed 和 local speed 和 remote speed；

4. access 在读 COW 出来还没写过的 pextent 时，如果读全部副本都失败，主动 refresh location 去读 parent 上的数据；

5. 在 133.171 上挂载 8 个 64T 的大卷做 ummap 试一下，如果还是慢，说明有可能是接入协议的问题。

    在 zbs 日志中看一下有没有 fail to ping 的日志，另外看一下多个卷做 unmap 的 zbs-chunk show_polling_stats 中 chunk-main 的 CPU 占用率。




tuna 自己去翻页查找 need recover 数据了。

zbs-client-py 中没有一个命令行可以给出精确的待恢复待恢复数据块个数，即使是隔 1min 调用 1 次 zbs-meta cluster summary 拿 ongoing / pending recover num，一共 2 次，在大规格容量（pid 数量超过 100w）下也会遗漏掉 100w 之后的那部分数据（目前 zbs 内部单次 recover 扫描上限是 50w 个 pid），比如最极端的场景有 800w 个 pid，需要调用 zbs-meta recover scan_immediate 16 次，如果连续 16 次看到的 ongoing / pending recover num 都是 0 并且 zbs-meta pextent find need_recover 也是 0，才认为集群真的没有待恢复数据。



刚刚那个日志显示 pid 239517 recover fail 报错 SYSENODATA，/var/log/message 中搜坏盘 sdg

cd /var/log/zbs && ll -rth zbs-chunkd.log* 按照日期排序找文件



副本读失败并不会触发 remove replica，但在副本读之前会有一次 sync，sync 失败的副本会被 remove replica

对于在被拔盘上的 extent，会读失败，但副本读失败并不会触发 remove replica，由于被拔盘了，所以他也不在 data report 里，meta 不会主动下发 gc cmd，直到 last report ms 超过 10 min 没更新，recover manager 扫描到它 not alive 了，才会下发 recover cmd。

对于在被拔盘上的 extent，会写失败，触发 remove replica，紧接放入待生成 recover cmd 队列中。



汇总一下临时副本，结合文档，https://docs.google.com/document/d/1L1I-_md5jE4GyqPItkioh1TzQXEgRhNFqIHtIwN-43k/edit#heading=h.moqcl2aq3auh

什么时候会 verifyread 而不是普通的 read

有了临时副本，staging block info 的含义是啥？



gc 回收逻辑

普通读的时候是否会 sync，会的，在 AccessIOHandler::DoReadVExtent() 中调用，像写一样，也会剔除 gen 不符预期的副本、在 sync 失败时清理本地 lease， 读 COW 出来的 pentry 但 parent 不在本地的情况调用一次 RefreshChildExtentLocation rpc

普通读一个 pentry 会避免读正在 recover 的 dst 副本，因为 app read 没有加锁，如果去读，可能读到一个中间态的值。

replica sync gen 的时候，如果发现他有 temporary replica，也会一起 sync gen



remove replica 和 replace replica 这两个 rpc 很重要，理解形参各个字段的含义、副本被剔除/替换的时机、access 什么时候会调用

remove replica rpc 时会把那个 cid 从 pentry 中 clear 掉，这样在 PhysicalExtentTableEntry::UpdateReplicaInfo 的返回值就是 False，HandlePExtentInfo 也是 False，等到对应的临时副本先回收，她才被回收。

meta 认为的要回收的临时副本，会将这个 pentry garbage 设成 true，valid = 0；

等 chunk data report 的时候，对于每个副本，都通过 ReplicaIsValid 检查是否可以删除

失败副本里不会被 found

这两部分经过 ReplicaIsValid 判定后，meta 为其生成对应 gc cmd 下发给 lsm 执行，此时数据真正被丢弃。



pentry 的 gc 标志，是 pentry 粒度的是否回收，只在 PhysicalExtentTable::AppendGarbagePids 里被用到，gc manager 会调用他。

segment 粒度的是否回收，关注的是 pentry 中这个 cid 的 PExtentReplica 是否有值。

pentry 的 rim_cid 只会在 remove replica 的时候被设置。



corrupt 状态的 pxtent，读它的时候是在 sync 阶段就返回 ECAllReplicaFail 还是等到 read 的时候？

读的时候会去 sync 吗？

sync 过一次什么时候会再次 sync？看起来只有在 ENotFoundOrigin 时会 RefreshChildExtentLocation，并主动触发一次重新 sync。

special recover 不需要 sync 吗？



用 zbs-meta pextent find need_recover 可以显示。

目前的做法，没法确保一定没有数据。如果是因为选不出 recover src/dst，且待生成的数量少于 1024，比如 cmd slots 不足、可用节点数量不足、集群容量不足导致的，会造成通过 zbs-meta cluster summary 拿到的 ongoing / pending recover num 相加值为 0。



recover handler 中的执行队列，可否做成 ever exist = false 且 origin_pid = 0 的 pid 优先执行，其他 pid 按 FIFO 的顺序执行。

recover manager 对于没有实际分配的数据会跳过命令下发配额的限制，快速下发给 access，如果这部分数据是从本地读，这个 recover 很快就会完成，但 recover handler 的 pending_recover_cmds_ 是按 FIFO 的顺序进 running_recover_pids_ 执行的，所以后发的符合上述特点的 pid 也没法快速执行，可能被前面执行慢的 pid 拖慢。



4. 从一个  volume 可以拿到所有 vextent id，据此可以拿到 lid，接着

    ```c++
    Volume volume;
    // vextent table 直接存入 meta db，并不在内存里常驻，vextent no 中带有是否 cow 的信息
    vextent_id = volume.vextent_id(i);
    lid = to_pextent_id(vextent_id);
    
    // vextent 的 location 就是 cap extent 的 location
    std::vector<VExtent> vextents;
    meta->GetVTable(pool_name, volume_name, &vextents);
    
    // 从 lextent table 中获取 lextent 的 cap/perf pid 信息
    LExtent lextent;
    meta->GetLExtent(lid, &lextent);
    
    // 从 pextent table 中获取 pextent 的 location/preferred_cid/thin_provision 信息
    PExtent cap_pextent;
    meta->GetPExtent(lextent.cap_pid(), &cap_pextent);
    PExtent perf_pextent;
    meta->GetPExtent(lextent.cap_pid(), &perf_pextent);
    ```
    
    创建大量 volume 并删除后，pid 被消耗殆尽，
    
    打快照只是 SetCow，快照的 origin_id 是源卷的 origin_id
    
    分层之后，一个只有 cap pextents 的 normal thick volume COW，不仅会分配出新的 cap pextents，还会分配出新的 perf pextents。
    
    * 克隆时不指定 thin_provision 的话，不论源卷是否 thin，克隆卷都是 thin 的；
    
    * 克隆时不指定 prioritized 的话，不论源卷是否 prioritized，克隆卷都不是 prioritized 的；
    
      目前创建一个 prior volume 允许指定 thin_provision = true，这会让该 cap pextents 是 thin 的。
    
5. SetBitmap() 只在 2 个地方被调用，ReplicaIOHandler::SetStagingBlockInfo/UpdateStagingBlockInfo，

    1. SetStagingBlockInfo()

        1. TryRemoveWriteSlowReplicas()，暂时不管
        2. HandleWriteReplicasDone()，记录写失败的副本所在节点
    2. UpdateStagingBlockInfo()

        1. ReplicaIOHandler::UpdateDone()

            1. ReplicaIOHandler::DoUpdate()

                1. ReplicaIOHandler::DoUpdateAndTemporaryReplica
                2. ReplicaIOHandler::UpdateInternal()

6. 编译换回 docker，弄回 67.4

7. 若已有 lease owner，他可能跟 src/dst cid 不同，如果是由于 src/dst 单点 IO 性能差造成的 auto mode 下缩小 lease owner 命令下发窗口，看起来是误判，实际上这种情况，下一次关于这个 pid 的 cmd 大概率还是会选到这个 lease owner，所以也不算误判。

    若没有 lease owner，新分配的 lease owner 副本模式下大概率是 src cid（除了 lease owner 本身分配优先选 src cid 之外，在下发前也有根据 lease owner 调整 cmd  src cid 的逻辑），较大概率是 dst cid，然后才是其他节点。

    另外，命令下发窗口可能被自动调节的前提是要打满一个窗口，也就是要有满一个窗口大小的命令数大部分都失败才有可能引发窗口收缩，比如 lease owner = 1, src_cid = 2, dst_cid = 3，若集群中只是 cid 2 IO 性能性能差，基本上需要给到 1 的 cmd src 基本都是 2 才满足整个窗口命令基本超时的条件，而此时把 1 的窗口跳调小也算正常，因为后续这些超时 cmd 的 pextent 再生成 cmd 时，lease owner 大概率还是 1。

8. meta 侧智能调节可以依赖 access 侧的并发度，比如某个 access 对其他所有 chunk 的并发度中取个最大值，比如 3 节点，access 1 认为自己作为 lease owner 跟 2 的 reposition 并发度是 32，跟 3 的 reposition 并发度是 64，那么可以让 meta 侧给到 access 1 的命令下发窗口是 64。

    hdd 盘配比不均的话，meta 侧智能调节各个 access 的 cmd slot size 就能够避免 access 没喂饱。

9. 调整 business io 影响内部 IO 的 iops 和 bps 阈值。

    1. 理论上我应该拿到所有磁盘类型+数量的上限后，用 GLAGS 去定义能给到 internal io 用的磁盘性能比例（0.5）和网络带宽比例（0.4 / 0.5），并给出 app io busy 的判断准则（比如  0.3 的上限，这样预留 0.2 出来做缓冲）。
    2. ssd 的限速不能直接跟盘成正比，主要是考虑到 zbs 没法发挥出磁盘性能上限，比如 4 块 nvme ssd 跟 2 块性能差不多。ssd 的 app io busy 先保留目前是一个定值的做法，但应该是一个变化值，综合考虑网络带宽以及磁盘性能，盘多了之后，瓶颈可能在网络带宽上，而网络带宽这事儿没法直接给出 app io busy iops（不管下沉数据，直接用 bps / 256 KiB？）
    
11. 待做

       1. piror recover

           先把 recover 关于 prior 的部分做完，等有空再考虑把 topo distance 做好，zbs4，另外，空间充足可以先过滤，但是尽量不选 isolated 和双活需要 2 ：1 的特性需要特别考虑。

           1. recover / removing chunk dst 允许选 isolated ？允许，为了尽快恢复/迁出；

           2. 把 avail cmd slots 提前算好放 exclude_cids；

           3. GenerateMigrateCmdsForRemovingChunk 中 migrate_generate_used_cmd_slots 对 src / dst 的判断应该传入 AllocRecoverCap/PerfExtents；
    
              传入会有点麻烦，可能出现 removing chunk 的时候总是选某个 src / dst cid，但那个 dst cid 可生成的余额不足，还一直选他。但是影响最大也就造成一次 generate 过程中只选 1 个 src cid，用满他的 256 的配额，所以先不修复。
             
           4. reposition src 如果有多个可选，可以考虑随机选，避免总是选到一个没法执行的导致一直 recover / migrate 不掉

           这部分代码可以写到 recover manager，另外也可以总结出一个 recover 和 alloc 虽然大部分相同，但是存在的细微差别。

           agile recover 和 special recover 回头处理，都是利用到临时副本的，入口是 remove replica

11. 补一个同时有多个 removing cid 的单测；



命令行相关

1. zbs-meta migrate < volume id> <replace_cid> <dst_cid>，尽量从 replace_cid 上移除，并尽量放到 dst_cid 上，不保证严格执行；

    用于卸盘或其他临时移动卷到指定 dst，之后被 doscan 回去也没事，但如果这个要迁移的卷很大，无法快速完成就被 doscan 回去呢？

    配合关闭迁移扫描再执行这个指令，可以达到临时移动 volume 的效果，迁移想要迁移的部分，不过这样或许得在入口处把人工触发迁移和周期性系统自动触发迁移做一下简单的区分。

    这是要让 pid 进入每个判断逻辑吗？因为没法像 recover list 那样可以直接放到 waiting list 然后生成 recover cmd，还是需要按照负载把这个 pid 放到各个子策略中。

4. zbs-meta recover < volume_id> 想让这个 volume 优先被 recover；

    当有多个 volume 需要 recover，耗时太久时，可以优先 recover 指定卷上的 pextent

    貌似也可以提供 pid 粒度的优先 recover rpc，比如在 chunk 想要触发某个 pid 的 recover，这个优先级要比 recover doscan 扫描的高，更早被执行。

5. zbs-meta recover set_runtime <start_hour> <end_hour>

    默认是 [0, 23]，左右闭区间，参考 taskd 中的实现，[ZBS-10973](http://jira.smartx.com/browse/ZBS-10973) 

    副本是期望 3，剩余 2 要恢复，EC 则是期望 m >= 2，允许丢失 m - 1 个 shard 

    是否也考虑做一个 zbs-meta migrate set_runtime <start_hour> <end_hour>，否则在业务高峰期带宽也有可能被 migrate io 抢占，但是 repair topo 应该是不希望关闭的吧？

    往 UpdatableRecoverParams 中增加 2 个字段，start hour  end_hour。同时，通过这个 patch 改 recover 的触发策略，IsNeedRecover。

    单测要写在 function_test 中



做一次冲突检查，合法且和当前恢复不冲突，prefer local 和 topo 相关的不管。

ZBS-20993，允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。



存储分层模式，可以选择混闪配置或者全闪配置，其中全闪配置至少需要 1 块低速 SSD 作为数据盘，混闪配置至少需要 1 块 HDD 作为数据盘。

存储不分层模式，不设置缓存盘，除了含有系统分区的物理盘，剩余的所有物理盘都作为数据盘使用，只能使用全闪配置。



在恢复或者迁移任务结束时，新加入副本的状态被设置为未知，需要等待下一次心跳周期 LSM 上报副本后才可以确认副本为健康？allocation 的逻辑是马上会被设置为活跃副本，参考 Commit -> PersistExtents -> UpdateMetaContextWhenSuccess -> SetPExtents。



[ZBS-13583](http://jira.smartx.com/browse/ZBS-13583) ，考虑改进如下 Case

\1. extent 是一个空的 child ，自己没有数据，所以如果要读写一定要有 origin 
 \2. 这个 extent 初始状态在 LSM 上也没有存在，所以 parent 迁移的时候就没有数据给他继承把它变成一个实体，这就导致了 child 和 parent 的 location 不一致了。对于完全不一致的场景，我们已经直接把这种空 child 直接 refresh 到和 parent 一样。但是这个只有局部不一致，没有触发这个机制；
 \3. 他们相同的位置刚好坏了盘，盘被没剔除的时候 extent 就一直在 io error 状态，没有被 GC 不可以重建，导致 sync 的时候 2 个副本一个缺少 origin， 一个 IO error，从而导致无法 recover；
 \4. 剔除了异常之后， IO error 因为 extent 被彻底回收了，可以正常重建了，所以恢复就自然结束了。



VIP 设计文档，https://docs.google.com/document/d/1M34zaIje2xkUSv9Q41waRH4GCPZ5yv7hwZqOK77vCq8/edit#heading=h.feb6l5x4y4vk

双活设计文档，https://docs.google.com/document/d/1z2cUXLrQ7pZnkJXiCPCLp-BPxkpZCSFIwrmvDxrUYX4/edit#heading=h.rxadnjfqdyav



那这里还有两个问题：

1. 卸载 partition 盘的时候，chunk 和 meta 分别会做哪些校验，分别用的哪个字段；
2. 卸载盘（或者拔盘）之后，可能会出现 allocated space > data capacity，进而导致数据迁移受到影响。meta 能否在集群数据恢复完成之后，保证 allocated space <= data capacity 呢？



如果都给了 topology 且两个副本的 zone distance, topo distance 都相同的情况下，LocalizedComparator 和 TopoAwareComparator 区别在于：

1. 前者只有在 owner = 0 时，选第一个 cid 时会用 recover_comparator，否则用 ring id 比较；
2. 后者全部用 recover_comparator 比较，也就是先 recover cmd num 再容量。

就一个副本并且跟 prefer local 不在一个 zone，那么 recover_prefer = 剩下的这个副本。如果就一个副本，并且跟 prefer local 在一个 zone，又会怎么样呢？

comparator->UpdateChunkSet 这个地方，如果还剩的 2 副本并不符合 topo 安全，那么是会按照放进去的第 2 个副本来选择后面的第 3 个副本。

只要 topo 不变，符合拓扑安全的副本位置也不变，LocalizedComparator 得到的排序结果是不变的，因此虽然选样本是连锁反应，但就算第 2 个副本位置不对，第 3 个副本位置还是正确的。





4 个 ticket 4 件事

1. recover 每台 chunk 上执行的并发度默认 32，根据 recover extent 完成情况向上向下调节（auto mode）

   并发度最小应该是 2 4，而不是 1，就 1 个通道的容错率太差了

   recover extent 完成情况应该是本地的，从 lsm 侧获取到信息。

   这里权衡 Chunk 的负载指标可能采用的有：1）Chunk LSM 是否 Busy，即 LSM 的 IsLSMBusy，根据当前 Journal 可用数量来判别。2）Chunk Node 的各项性能指标，包括 cpu/memory/disk/network，见 node_monitor，但是现在的实现貌似只有全局的指标，例如 cpu 是所有的核心 summary，disk 不好区分是哪个盘需要做读写。不大好设置阈值。3）根据 Recover IO 与 正常 IO 之间的比例来判定，

2. recover cmd slot 默认 128，根据 extent 的稀疏情况以及 recover extent 的完成情况向上向下调节（auto mode）

   怎么判断 extent 的稀疏情况？lsm 上报的心跳中有 thin pid 的信息

   

   生成的自动调节机制
   
   1. 缓存队列不为空，那么下一个 1min 才生成下发；
   2. 缓存队列为空
      1. 首次为空，下一个 4s 生成下发；
      2. 至少两次为空，下一个 1min 才生成下发；
   
   下发的自动调节机制
   
   调节可允许下发命令的窗口大小，这个需要到 chunk 粒度（lease owner），目前是一个值作用到所有 chunk 上，但 auto mode 要能让各个 chunk 变动的值。
   
   
   
   被 access 标记为超时的单独拎出来：
   
   1. 在 sync 之后发现 ENoNeedRecover；
   2. 在每一个 block read/write 之间，都有可能判断到 EShutDown，ELeaseExpired，ETimedOut，ECancelled，其中 ETimedOut 指的是超过 17 min 的。
   3. 在 agile recover 中发现 ECGenerationNotMatch 或者 ELeaseExpired
   
   这些里面，lease 过期、被 cancel 之类的不该影响到下发个数，针对超时单独汇报。



meta 侧的参数在尽可能让 recover 变快的同时，要考虑自身一次的扫描时间，若扫描周期过长，无法立即触发数据的恢复和迁移



chunk recover 执行的慢可能原因：慢盘、缓存击穿、normal instead of agile recover、

考虑到如果是稀疏的 Extent，恢复命令执行的会比较快。所需要的恢复命令会相对较多。如果是 Extent 数据相对饱和。则恢复没有那么快。所需要的命令会较少。

过去一段时间的恢复速率过慢、recover cmd 完成数量过少、还是 timeout 标记、lsm 侧缓存剩余比例（clean + free 的比例，如果太高的话，说明缓存基本没用上，recover handler 目前已经用了这个值来避免缓存击穿，zbs-chunk cache list 可以看到）、路径很多，先列举出来。PartitionList 有感知 SlowIO 的数量。



作用于 meta 侧的 recover IO 超时相关的 FLAGS

* recover_timeout_ms = 18 min；

作用于 chunk 侧 IO 超时相关的 FLAGS

* local_chunk_io_timeout_ms = 8s，local chunk io timeout ms，返回的是 ELSMCanceled
* chunk_recover_io_timeout_ms = 9s，chunk recover io timeout ms，recover 远程 IO，这个远程指的是 access (pextent io handler) 给到非本地的 lsm (local io handler)。
* remote_chunk_io_timeout_ms = 9s，remote chunk io timeout ms，非 recover 远程 IO （ZBS 对网络有限制，如果 ping 大包来回超过 1s，认为网络严重故障，系统不工作）。
* chunk_lsm_recover_timeout_sec = 10 min，在 lsm 侧每 60s 检查一次 recover pextent，如果 recover 时间超过 10 min 都没有结束，会将 extent 标记为 EXTENT_STATUS_INVALID，dst cid 上的这个 pextent inode 会被 lsm gc，之后接着 recover 会抛出 ENotFound（这个 pid 后续跟随 data report  给到 meta，不过 meta 没有对 EXTENT_STATUS_INVALID 特别处理）





敏捷恢复为减少内存使用，是有单次最大数量的限制。不过 100G 的写盘应该不会触发这个上限。

调查为啥升级时触发的敏捷恢复数量不及预期可以从维护模式时是否 lease 没清空的角度出发调查。



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

做 migrate for repair topo 和 rebalance 时，需要考虑以 chunk 为粒度的遍历，其他的考虑以 pid 为粒度遍历就好。



如果节点上的 replica 发生 cow 的话，direct_prs 会瞬间减小而导致写入因为等待空闲 cache 而阻塞，也需要预先下刷以避免阻塞



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

4. 目前遇到的高负载下不迁移：要么 topo 降级了，要么 lease owner 没释放，要么是双活只能在单 zone 内迁移

4. 后续写文档可以考虑先介绍所有 sub migrate strategies 中共有的限制条件：

   1. isolated
       1. must not be dst;
       2. should not be src/replace in ec migrate;
       3. should not be src, should be replace in replica migrate;
   2. EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，所以以上选到的 mgirate src 不一定就是最后给到 access 时的 src
   4. 在介绍 reposition 时，可以介绍：
   
       1. 扫描、生成、下发命令参数
           1. 单轮扫描命令上限；
           2. 单轮生成命令上限、单轮每个 chunk 生成命令数上限；
           3. 单轮下发命令上限、单轮每个 chunk 下发命令数上限；
       2. repostion stale cmd 的判定；
       3. reposition lease owner 的选取规则（replica 和 ec 不同）；
           1. 若已有 lease owner，沿用之前的 lease owner，否则
           1. 先 src 再 dst；
           2. ChooseOwnerForReposition；
   5. 在介绍副本分配策略时，可以介绍：
   
       1. thick / thin 分配，结合 transaction，什么时候分配 pid，location，预留空间；
       2. chunk space 计算方式、字段含义；



遗留问题：

1. lease owner 不释放的一个原因是 inspector 扫描到 extent generation 不一致而触发的读操作（借此剔除 gen 较低的 extent，再经由 recover 完成数据一致）

2. 为什么 LSM 写 4 KiB cache 需要写 journal，写 256 KiB cache 不需要？

    4k 写为了避免写放大，除了写一个 4k 的真实数据外，还需要写对应的 bitmap （一个 256 KiB 的 block 中有 64 个 4k ）并持久化到 journal。

    否则就需要写 4k 真实数据 + 1020k 实际为 0 的数据，这将引起写放大。

3. LSM2 中，采取 Journaling+Soft Update 的方式保证 crash consistency。默认采用 Journaling，所有数据和元数据的更新都需要通过 Journal 进行保护，然后再写入 BDev。当数据是初次写入时，允许采用 Soft Update 方式，先将数据写入磁盘，再把元数据写入 Journal，避免因数据写入 Journal 引起的写放大。

    初次写的数据如果丢了，元数据没来得及写入 Journal 会有什么后果？

4. 在元数据中，最消耗内存的是 PBlob ，我们必须实现 PBlob 不常驻内存。PBlobEntry 数量跟 PBlob 接近，但单个 PBlobEntry 较小，可全量放内存。对于独占数据，无需额外内存保存 Entry，只需要计算映射值即可；对于共享数据，需要在内存中保存 Entry，因此，快照数量的增加，会增加 LSM 的内存占用。

    为啥独占数据，无需额外内存保存 Entry，只需要计算映射值即可？

5. 为啥 class PBlobCtx 相较于 message PBlobPB 可以节省内存和操作开销？

6. 为啥只有 private pblob 可能触发 promote 的 IO 读，以及 shared pblob 为啥不会？

7. 普通 IO 读不需要修改 Journal，触发 promote 的 IO 读由于涉及到从 data block 到 cache block 的拷贝，所以需要写一条 amend pblob PB 的 Journal，然后才会按照普通 IO 读的流程走下去。

8. 只有 partition 中会使用 checksum，每 8 KiB + 512 B 的布局，每 8 KiB 计算出 CRC32 值，保存在随后的 512 B 里，所以 valid_data_space = 94% * total_data_capacity

    分层之后的 valid_data_space = 94% * total_data_capacity，perf_valid_data_space <= 90% * perf_total_data_capacity，10% 指的是至少为 cap read cache 预留 10% 的 perf space

9. SSD 和 HDD 的 Block（560 开始）

    1. SSD 的 Block 大小使用 16 KiB，而不是 4 KiB。主要出于两点考虑：1. 4 KiB 粒度过小，元数据量较大，且占用的内存过高；2. 4 KiB 粒度过小，对读不友好。单个大 IO 会切分成过多的小 IO。
    2. HDD 的 Block 大小使用 128 KiB，主要出于两点考虑：1. 128 KiB 粒度，占用的内存量可接受；2. HDD 的 IO 延时，大部分来自于寻道，写 IO 的放大对性能影响较小。

10. 256 KiB PBlob 描述

     ```protobuf
     message PBlobPB {
     	// 按 4k 粒度记录是否有真实数据
         optional fixed64 bitmap = 1 [default = 0];
         optional fixed32 cache_block_id = 2;
         optional fixed32 data_block_id = 3;
     
         // cache_clean is meaningful only when has_cache_block && has_data_block
         optional bool cache_clean = 4 [default = false];
     }
     ```

     PBlob 有如下 5 种状态：

     1. 未分配：cache_block_id 不存在、data_block_id 不存在；
     2. 冷数据或全闪：cache_block_id 不存在、 data_block_id 存在；
     3. 热数据：cache_block_id 存在、 data_block_id 存在、二者数据不一致；
     4. cache bock 可被回收的热数据：cache_block_id 存在、 data_block_id 存在、二者数据一致；
     5. 新写且未触发过回写：cache_block_id 存在、data_block_id 不存在；

     有如下几种操作：

     1. 创建一个 PBlob，例如 Extent 需要分配空间时；
     2. 更新 PBlob 内容，包括：
         1. 修改 Bitmap，例如当有 IO 写入 PBlob 时；
         2. 修改 Cache Block ID，例如当发生分配、Promotion 或 Eviction 时；
         3. 修改 Data Block ID，例如当发生分配或 Writeback 时。
     3. 删除一个 PBlob，例如当没有任何 PBlob Table （这个概念现在可能是 ExtentInode？）引用这个 PBlob 时。



待整理

下一个 smtxos 开始使用 yq，了解 yq 的用法，https://github.com/mikefarah/yq 



unmap 是针对精简配置的存储阵列做空间回收，提高存储空间使用效率，应用于删除虚拟机文件的场景。VMware 向存储阵列发送 UNMAP 的 SCSI 指令，存储释放相应空间。

https://blog.51cto.com/xmwang/1678350

TRIM 是一种由操作系统发出的命令，用于告知 SSD 哪些闪存单元包含已删除文件的数据，可以被标记为空闲状态

SSD 从不像 HDD 那样直接将新数据覆盖写入旧数据。在所有存储单元都被擦干净之前，无法用新数据对存储单元进行写入，且擦除必须在块级别进行，而写入则在页级别（更小的粒度）进行，这意味着对 SSD 进行写入比擦除要快得多。

- `discard` 是一个由操作系统在删除文件时自动发送给 SSD 的命令，它是实时执行的。
- `fstrim` 是一个由用户手动调用的命令，用于释放整个文件系统中的未使用空间，也可以被自动调度为定期任务执行。



内核实时线程的说明，https://access.redhat.com/documentation/id-id/red_hat_enterprise_linux_for_real_time/9/html/understanding_rhel_for_real_time/assembly_scheduling-policies-for-rhel-for-real-time_understanding-rhel-for-real-time-core-concepts

C++ 中为减少内存/读多写少的情况，可以用 absl::flat_hash_map 代替 std::unordered_map，https://zhuanlan.zhihu.com/p/614105687

C++ 中 map 嵌套使用，vector 添加一个 vector 中所有元素 https://zhuanlan.zhihu.com/p/121071760

stl 容器迭代器失效问题，https://stackoverflow.com/questions/6438086/iterator-invalidation-rules-for-c-containers

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610

git submodule ，https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97，https://zhuanlan.zhihu.com/p/87053283



vscode 中用 vim 插件，这样可以按区域替换代码

一个遗留问题是，单测里面想要触发两次 recover cmd，怎么让 entry 的 GetLocation() 得到及时更新，试了 sleep(9) 不行，可能不止需要一个心跳周期，还有其他条件没触发。



### 临时副本

临时副本的有效性是两方面的，一方面是临时副本自己有效，另一方面是临时副本的失败副本有效。因为临时副本的数据 + 失败副本的数据才是完整的数据，如果有一方不完整，那整体都是不完整的。



只有在做 special recover 且 rollback_failed_replica 和 force_recover_from_temporary_replica 其中一个为 true 的时候才会在 replace replica request 中设置 reset_location_to_dst 和 reset_generation，那么 meta 会要求这个 pentry 必须所有副本都 dead，把这个 pentry 的 location 设置成只有一个 dst cid， gen 设置成 reset_generation，rim cid 设置成 0，清空这个 pentry 所有的临时副本。



有损临时副本的 temporary_pid 和 temporaray_epoch 都是 0，但他的 failed_cid 是个有意义的值，能保证避免在 failed_cid 上的失败副本被 lsm 回收。

special recover 的 src cid 是（有损）临时副本所在 chunk，dst 是失败副本所在 chunk。

由于允许发起 special recover 的前提是所有副本都 dead，所以 special recover 中的 replace cid 一定会被填充。当使用的临时副本是 lossy 时，必须要让 force_recover_from_temporary_replia 和 rollback_failed_replica 其中一个为 true。



所有副本都 dead 的时候才允许 special recover rpc 执行

1. normal special recover like agile recover
2. force_recover_from_temporary_replica, base on normal special  recover, but we ignore the validity check of  temporary replica 
3. rollback_failed_replica, just set temporary replica's failed_cid  as pextent's location

一般情况下，如果不能直接通过 normal special recover 恢复的，需要分析日志再决定采用强制恢复还是回滚。

force recover from tmeporary replica 直接从临时副本上读数据，不管他的  gen 是多少，然后写入到失败副本，然后把这个副本当成正常副本来用。

rollback_failed_replica 是丢弃临时副本上的数据，直接把失败副本当正常副本来用，rollback 是一种更兜底的做法，大部分情况是在集群因为空间不足没法为临时副本分配副本的时候用的。



临时副本不会产生迁移命令，且由于 temporary pid 不在 chunk table 的 cap / perf pids，所以移除节点迁移时，即使不迁移临时副本，但最终也认为他移除完了，实际上这个移除节点上的临时副本全都丢失了，不过一般来说不会在有数据恢复的情况下移除节点，所以应该还好。



临时副本以一个单副本的形式保存了失败副本（通过 RemoveReplica rpc 被剔除的副本）在剔除之后增量 IO，在失败副本恢复可用之后（常见于节点升级、服务重启，网络中断等存储介质本身没有损失的场景），失败副本中的原始数据和临时副本中的增量数据能够组合成一份完整数据。

在分配临时副本时（AllocTemporaryReplica），会在集群中所有的健康节点中选一个节点（是的，因为是单个副本）

must meet

1. 不是这个 pextent 的其他临时副本所在 chunk；
2. 不是这个 pextent 的失败副本所在 chunk；
3. 不是这个 pextent 的 location 中的 chunk；
4. 已使用空间没有超过 95% 的 chunk

should meet

1. 不是 isolated 节点；

2. 跟这个 pextent 的失败副本所在 chunk 还有 location 中的 chunk 的 topo distance 最远的；

    双活下的规则特殊点：

    1. 健康副本剩余 2 个：若在同一个可用域，则优先选择与当前存活副本相同可用域的节点存放 Temporary Replica，否则优先选择 extent 自身 prefer local 所在可用域的节点存放 Temporary Replica；
    2. 健康副本剩余 1 个，则优先选择与当前健康副本在同一个可用域内的节点；

3. 正在 reposition 数量最少的；

4. 剩余空间最大的。

分配临时副本时，先默认分配一个 lossy 临时副本，如果空间充足，分配成功了会在 transaction commit 中把它设成 false 的，否则还是分配一个 lossy 临时副本，不会不分配（因为想尽可能保留失败副本，有临时副本的失败副本，不会在 lsm 被 GC）。

除了集群无法为临时副本分配空间时它的 lossy 属性是 True，已分配的临时副本有 IO 错误时为 True（在 RemoveReplica rpc 移除副本时可能也会夹带着移除有 IO 错误的临时副本 ），有损临时副本不参与 IO，仅可用作恢复，多为有损恢复。除开这 2 种情况，临时副本的  lossy 都是 False。

在 RemoveReplica rpc 时，如果在 request 里也指定了要移除的临时副本，那么只是把这些临时副本 lossy 设为 true，而不去标记待 gc，这是因为 IO 过程出错的临时副本已经写了一部分数据，想保留这部分数据（临时副本这一功能的核心是尽力保留所有已经写入的数据），不到万不得已不丢弃数据。

失败副本及其临时副本什么时候才认为可以被 gc？比如 3 副本的 extent，降级为 1 副本+ 2 临时副本，等这个 extent 恢复成 2 副本后，调用 ReplaceReplica rpc 告知 meta，其中 request 的 src_chunk 是 recover cmd 中的 replace cid ，这个 rpc 中会调用 RemoveTemporaryReplicaOnReplace 来移除 1 个临时副本（dst cid 上的优先，否则是 lossy 和 ever exist = false 的临时副本，最后才是 gen 最大的那个副本）



失败副本在写失败时就在 meta 侧设成待 gc 了，但是在 lsm 侧，只有对应的临时副本 gc 后，这个失败副本才会被 gc（代码体现在一个副本如果有对应的临时副本，那么 meta 不会下发 gc cmd）。



临时副本的 pentry 被删除一定发生在他所附属的那个普通 pentry 被删除

```
// PhysicalExtentTable::ScanByPidRefs
{
	...
	MarkAndClearGarbageUnlocked(pid);
	MarkAndClearAllTemporaryReplicaGarbageUnlocked(pid);
}

```

### 集群升级

日志见升级控制节点的 /var/log/zbs/cluster_upgrade.log 日志，/usr/share/upgrade/file 路径下有待升级的目标 ISO 节点是升级控制节点。



### Lease 相关

meta 会 revoke 整个 volume lease 的 5 种情况

1. create snapshot
2. rollback volume、move src volume to dst（rollback 的老接口）
3. reclaim volume temporary replica（命令行调用，但是有必要回收整个 volume lease 吗？）
4. delete volume
5. resize volume

### prometheus 使用

4k app io 没被统计在 local io handler ，access handler 中显示 app iops / bps = 0，显示在 perf layer，因为 4k 会先写 perf layer

开启 prometheus：在 meta leader 上执行 nc -k -l -p 9093 -c "nc 10.0.180.183 9090"

prometheus 语法

```shell
# 过滤掉不想看的 volume 
zbs_volume_logical_size_bytes{_volume!~"7697f.*|56e7e.*|130478.*|6ac3588.*"}
# 按值过滤
zbs_volume_logical_size_bytes{} > 1 and zbs_volume_logical_size_bytes{} < 536870912000
```

tower 首页的存储性能图标对应 zbs 的哪些 metric？



prometheus 里可以从 2 个角度来观察值

1. zbs_chunk_access 开头的，比如 zbs_chunk_access_cap_replica_reposition_read_iops from_chunk 1 to_chunk 2 表示以 1 为 lease owner read 2 上数据的 iops。
2. zbs_chunk_local_io_from_local 开头的，比如 zbs_chunk_local_io_from_local_cap_ec_app_write_latency_ns 表示这个节点的 local io handler 接受到的从本地 access 来的 ec app write 的延迟
3. zbs_chunk_local_io_from_remote 开头的，比如 zbs_chunk_local_io_from_remote_cap_replica_reposition_write_speed_bps 表示这个节点的 local io handler 接受到的从远端 access 来的 cap replica write 的带宽

prometheus 中支持多种 IO 类型的 metric 相加，比如二者相加可以观察这个 chunk 收到的所有 cap replica reposition write 的带宽，zbs_chunk_local_io_from_remote_cap_replica_reposition_write_speed_bps + zbs_chunk_local_io_from_local_cap_replica_reposition_write_speed_bps 

### 均匀卷

* zbs cli 是只有 snapshot 能更新 new_alloc_even 
* zbs iscsi / nvmf / meta client 是可以更新 lun / volume / snapshot 的 new_alloc_even 属性
* meta iscsi / nvmf server 支持在  UpdateLun / UpdateSnapshot 里更新
* meta rpc server 里支持在 UpdateSnapshot / UpdateVolume里更新

如果 volume 更新了 even，尽管把 volume 的 prefer local 请求了，但是如果还会写，那之后

虽然 elf 用 lun_create 的接口，但是之后不会再去写这个 volume，但是这个 volume 会被下沉，而下沉分配出来的 cap extent 的 prefer local 在 cap pentry 没有 prefer local 时由发起下沉的 lease owner 决定。



elf  一直以来创建虚拟卷模板，创建的是 volume 而不是 snapshot。我理解是因为它那边有虚拟机模板转化为虚拟机，以及反向操作的需求（tower 上有入口），但如果是 volume，理论上总是有可能被写，可以加上考虑在 UpdateVolume 的时候，除了将 new_alloc_even 设置为 true，也同时把 read_only 设置成 true（没问题的话把这个需求给到 elf）



均匀卷由 even 属性被更新成 True 的卷/快照卷和克隆超过一定次数（默认为 10）的快照卷两部分组成（不支持直接创建出 even volume / snapshot）

### 网络相关日志

查看网络的方式有哪些？

* [网卡异常探测](https://docs.google.com/document/d/1cNja_rnJ3fQglBqfouPvx8Ss82qzIiObN6cN_wYUs3s/edit#heading=h.xjzjaz3p8u9t)（节点层面） /var/log/zbs/netbouncer/l2ping@storage.INFO

* [网络亚健康探测](https://docs.google.com/document/d/1MK0VRK5WcRF14N36PpJ_O0Y-HUHG-fxe9q-usfAAevk/edit#heading=h.jz7vtdo3hm60)（集群层面） /var/log/zbs/network-monitor.log

    ```
    fping <data_ip> <mgt_ip> -C 30 -t 199 -i 1 -r 1 -p 400 -q 
    ```

    发送 30 个 ping 包，超时时间为 100ms，发送间隔为 1ms，如果第一次 ping 失败会重试一次，两次 ping 之间的间隔为 400 毫秒。

    符合 failslow 条件会被隔离（/var/log/zbs/netbouncer/netreactor.INFO）

    ```
    I1125 10:35:17.013400 scoredb.go:374] - Got scores: map[10.0.132.231:88.48718606407681 10.0.132.232:4.319999999999999 10.0.132.233:15.069390336000003 10.0.132.234:1.1]
    I1125 10:35:17.013507 scoredb.go:529] - Found outlier score for the worst node: 10.0.132.231, score: 88.487186
    I1125 10:35:17.013522 scoredb.go:378] - Found slow node ip: 10.0.132.231
    I1125 10:35:17.028179 reactor.go:148] - Ban chunk id: 2 done
    ...
    I1125 10:42:17.000876 scoredb.go:374] - Got scores: map[10.0.132.231:1 10.0.132.232:1 10.0.132.233:1 10.0.132.234:1]
    I1125 10:42:17.000932 scoredb.go:384] - Found recovered node ip: 10.0.132.231
    I1125 10:42:17.009688 reactor.go:157] - Unban chunk id: 2 done
    ```

    对应的 meta 日志

    ```
    I1125 10:35:17.013777 32630 chunk_isolate_manager.cc:429] Start isolating chunk. cid: 2, policy name: FAILSLOW_HCI, overwrite_policy: 1
    ...
    I1125 10:42:17.001516 32630 chunk_isolate_manager.cc:489] Start deisolating chunk. cid: 2
    ```

    假设集群内有 3 个节点 A、B、C，其中 A 与 B 之间发生了网络失联。A 和 B 的互相打分均会按照 1->2->4->8->16->32->64->100 迅速到达 100。但由于 C 和 A、B 的网络状况仍正常，所以 C 对 A 和 B 的打分仍维持 1。对应的 IPScores 如下：

    ```
    A: {B: 100, C: 1}
    B: {A: 100, C:1}
    C: {A: 1, B:1}
    ```

    以 30% 分位的分数作为特征值，最终 A、B、C 的特征值都是 1。

* data channel 探测（chunk 层面）/var/log/zbs/zbs-chunkd.INFO

网络延迟发生在中间网络或者发生在 L2 层以上，网络亚健康探测的 fping 发现异常，而网卡异常探测 L2Ping 没有发现异常，此时也只能依靠 Fail Slow 机制隔离节点。

recover 慢，要检查网络情况，看看是否有大包丢包，如果不是所有节点都差的情况下，可以适当选择切换 bond。怎么查看是否 bonding 以及切换 bond 的方式？

rdma 的网络环境测试由自己的 ib 测试方法，不能只看 ping 的结果。

### app io 性能调查

1. 看 fio 给出的延迟
2. zbs-perf-tools volume show < volume id > 给出的延迟
3. zbs-perf-tools chunk access summary 给出的延迟
4. zbs-perf-tools chunk lsm summary 看存储引擎给出的延迟
5. iostat -xm 1 看物理磁盘给出的延迟

### reposition 性能测试

fio -ioengine=libaio -invalidate=1 -iodepth=128 -ramp_time=0 -runtime=300000 -time_based -direct=1 -bs=4k -filename=/dev/sdc -name=wrtie_sdc -rw=randwrite;



考虑一个被写满的 extent，从理论上分析：

ec

recover：从 k - 1 个节点读，每个节点读 256 MiB / k；写到 1 个节点上，写 256 MiB / k；

* recover 读：
* recover 写：写到 1 个节点上，写 256 MiB / k；

* migrate 读：从 1 个节点读，读 256 MiB / k；
* migrate 写：写到 1 个节点上，写 256 MiB / k；

replica

不论 migrate / recover，不论读还是写，都是从 1 个节点到另一个节点，数据量是 256 MiB



* 如果是 ec，recover 读的数据总量是 256 MiB / k * (k - 1)，从 k - 1 个节点上读，写是 256 MiB / k；migrate 读写都是 256 MiB / k。
* 如果是 replica，recover / migrate 读写都是 256 MiB。

读写



实验设定：

ring id 一开始是 1 4 2 3 的顺序（值要分散点），副本超时时间配成 1 min，fio 在 cid1 上做，对应 ip 213

1. 一开始 segment 在 [1, 4, 2]，修改 cid 3 的 ring id 到 4 和 2 之间，产生 src = 2，dst = 3，replace = 2 的 migrate cmd（因为 ec src = ec replace），统计时间，之后 loc = [1, 4, 3]

    日志里搜 UPDATE TOPOOBJ c26160d8-3dd1-4fca-b895-294d612c91b2 new ring id 5，看之后第一条 migrate cmd 下发时间到最后一个 replace replica 接受时间

    0723 22:25:44 - 0723 22:28:38，174s

    Current generate cmds per round limit: 4096，Current cap distribute cmds per chunk limit: 500

    0724 00:03:08 - 0724 00:04:40，92s

2. 把 cid 4 的 chunk stop 掉，统计时间，之后 loc = [1, 2, 3]

    日志里搜 cid 4 session expired， 要减去 1 min，因为分片要 1min 才超时

    0723 22:29:52 - 0723 22:31:45，113s

    Current generate cmds per round limit: 4096，Current cap distribute cmds per chunk limit: 500

    0724 00:06:48 - 0724 00:08:45，117s

3. ring id 改成 1 4 2 3 的顺序，把 ec 卷删掉，再开始创建一个 prefer local 也是 1 的 replica 卷，fio 写全盘后，主动多次 sink；

4. 一开始 segment 在 [1, 4, 2]，修改 cid 3 的 ring id 到 4 和 2 之间，产生 src = 1，dst = 3，replace = 2 的 migrate cmd（因为 replica replace 会优选 lease owner），统计时间，之后 loc = [1, 4, 3]

    0723 22:53:09 - 0723 22:56:24，195s

    Current generate cmds per round limit: 4096，Current cap distribute cmds per chunk limit: 500

    0723 23:36:43 - 0723 23:39:47，184s

5. 把 cid 4 的 chunk stop 掉，统计时间，之后 loc = [1, 2, 3]

    0723 22:57:53 - 0723 23:01:12，199s 

    Current generate cmds per round limit: 4096，Current cap distribute cmds per chunk limit: 500

    0723 23:42:31 - 0723 23:45:36，185s





1. ec volume 的下沉比 replica volume 来的快
2. 2 + 1 ec volume 在 perf 层只会有 2 个 replica



顺序写 nvme 盘

cgexec -g cpuset:. taskset -c 11 fio -ioengine=libaio -invalidate=1 -iodepth=128  -direct=1 -bs=256k -filename=/dev/nvme4n1 -name=write_128_4k_fio -rw=write



测试 auto mode 给的默认值是否合适

1. 没有执行 zbs-meta reposition update --static_cap_distribute_cmds_per_chunk_limit 256，跑不满 internal IO 上限
2. 需要执行 zbs-meta sink update --cap_direct_write_policy_map '0, 3; 1, 3; 2, 3; 3, 3'  来直写 cap，且要保证这个 volume 没有 perf 副本
3. zbs-perf-tools volume show 的值跟 fio 的基本能对上
4. watch zbs-meta internal_io show
5. zbs-meta topo update 77d91b44-abb8-43ce-832f-5bf632b50601 --new_ring_id 2 / 4 && zbs-meta migrate scan_immediate
6. watch zbs-meta reposition summary
7. rm -f /usr/sbin/zbs-chunkd && cp /tmp/zbs-chunkd /usr/sbin/ && systemctl restart zbs-chunkd



在一个脚本中跑多个 fio 任务，lun 更改之后，计算端怎么感知到，目前是先退出再进去。

看下 zbs-meta session list_iscsi_conn 给出的 session id 和 cid

```shell
[root@Node57-63 17:14:05 ~]$zbs-meta session list_iscsi_conn
Initiator                           Initiator IP    Target ID                               Client Port  Session ID                              CID  Transport Type
----------------------------------  --------------  ------------------------------------  -------------  ------------------------------------  -----  ----------------
iqn.1994-05.com.redhat:9350c647c49  127.0.0.1       7cac873b-8bbe-464d-9748-74c7cab63698          47466  283bea99-7056-440e-9083-c0a0ca3ef141      3  TCP
iqn.1994-05.com.redhat:9350c647c49  127.0.0.1       b34f5079-25f3-460e-a16f-de1b7272fb47          47544  283bea99-7056-440e-9083-c0a0ca3ef141      3  TCP
iqn.1994-05.com.redhat:a14f743d372  10.0.57.63      12499d25-24db-46ee-bd11-3c85be84e8e0          57464  6d8747ee-f3c3-43f0-8428-221d096739a8      1  TCP
```

133.173 节点上跑 /dev/sdh，并在这个节点上开个 iostat -xm 1

hdd 磁盘参数如下

```shell
Model Family:     Seagate Constellation.2 (SATA)
Device Model:     ST91000640NS
User Capacity:    1,000,204,886,016 bytes [1.00 TB]
Sector Size:      512 bytes logical/physical
Rotation Rate:    7200 rpm
Form Factor:      2.5 inches
ATA Version is:   ATA8-ACS T13/1699-D revision 4
SATA Version is:  SATA 3.0, 6.0 Gb/s (current: 6.0 Gb/s)
```

往 /etc/sysconfig/zbs-metad 中追加如下参数并重启，以保证 fio 节点上始终有本地副本

若还是没有本地副本（没有感知到 access point），可以 zbs-iscsi lun update target-yiwu 1 --prefer_cid 2 来手动设置。

```shell
CAP_MEDIUM_RATIO = 0.997
CAP_HIGH_RATIO = 0.998
PERF_THIN_MEDIUM_RATIO = 0.997
PERF_THIN_HIGH_RATIO = 0.998
VERY_HIGH_LOAD_RATIO= 0.999
META_DISABLE_RECOVER=true
META_DISABLE_MIGRATE=true
```

往  /etc/sysconfig/zbs-chunkd 中追加如下参数并重启，以保证不会写到 ssd 盘

```shell
ENABLE_IO_CACHE_V2=false
CHUNK_LSM2_CACHE_FIXED_RATIO=0
```

在 cat /etc/cgconfig.d/cpuset.conf 中选一个在 group machine.slice 中的 CPU 或者不在任何 group 中的，比如 10，绑到没被 polling 的 CPU 核心上，10 是 CPU num

```shell
cgexec -g cpuset:. taskset -c 10 fio yiwu.fio

cat yiwu.fio
[global]
ioengine=libaio
direct=1
time_based
filename=/dev/sdh

[test]
rw=randread
bs=256k
numjobs=1
size=10G
iodepth=8
```

创建一块 10G Lun，在集群任一节点上用以上配置跑 fio，iSCSI 跑性能显著差于 nvmf，如果性能上不去上，可能是被接入协议限制的。

如果想要批量测试的话，fio 脚本应该怎么写

200 G 的 LUN，2 副本都在有 4 块 hdd 的节点上，其中一个是本地副本。

在 vi /etc/cgconfig.d/cpuset.conf 把 12 去掉了

nvmf 

 zbs-nvmf ns create yiwu-sub1 5 16 --nqn_whitelist="nqn.2014-08.org.nvmexpress:uuid:f97082f5-2092-42f6-a223-14ecdc6996d0" --replica_num=1

lsblk | grep nvme4 带后缀 5

cgexec -g cpuset:. taskset -c 11 fio -ioengine=libaio -invalidate=1 -iodepth=128 -ramp_time=0 -runtime=300  -direct=1 -bs=256k -filename=nvme4n5 -name=write_128_4k_fio -rw=write -randrepeat=0

| Case(hdd num + mode + iodepth + bs) | IOPS         | BW                   |
| ----------------------------------- | ------------ | -------------------- |
| 4_randread_8_256k                   | 801          | 200MiB/s             |
| 4_randread_16_256k                  | 1081         | 270MiB/s             |
| 4_randread_32_256k                  | 1027         | 257MiB/s             |
| 4_randread_64_256k                  | 1069         | 267MiB/s             |
| 4_randread_128_256k                 | 1104         | 276MiB               |
|                                     |              |                      |
| 3_randread_8_256k                   | 698          | 175MiB/s             |
| 3_randread_16_256k                  | 892          | 223MiB/s             |
| 3_randread_32_256k                  | 837          | 209MiB/s             |
| 3_randread_64_256k                  | 864          | 216MiB/s             |
| 3_randread_128_256k                 | 842          | 211MiB/s             |
|                                     |              |                      |
| 2_randread_8_256k                   | 549          | 137MiB/s             |
| 2_randread_16_256k                  | 661          | 165MiB/s             |
| 2_randread_32_256k                  | 640          | 160MiB/s             |
| 2_randread_64_256k                  | 603          | 151MiB/s             |
| 2_randread_128_256k                 | 560          | 140MiB/s             |
|                                     |              |                      |
| 4_write_8_256k                      | 2109 \| 3233 | 527MiB/s \| 808MiB/s |
| 4_write_16_256k                     | 1748 \| 3227 | 437MiB/s \| 807MiB/s |
| 4_write_32_256k                     | 1544 \| 2927 | 386MiB/s \| 732MiB/s |
| 4_write_64_256k                     | 1574 \| 1999 | 394MiB/s \| 500MiB/s |
| 4_write_128_256k                    | 1577 \| 1613 | 394MiB/s \| 403MiB/s |
|                                     |              |                      |
| 3_write_8_256k                      | 2088 \| 2199 | 522MiB/s \| 550MiB/s |
| 3_write_16_256k                     | 2144 \| 2185 | 536MiB/s \| 546MiB/s |
| 3_write_32_256k                     | 1747 \| 2052 | 437MiB/s \| 513MiB/s |
| 3_write_64_256k                     | 1278 \| 1153 | 320MiB/s \| 288MiB/s |
| 3_write_128_256k                    | 1102 \| 1119 | 276MiB/s \| 280MiB/s |
|                                     |              |                      |
| 2_write_8_256k                      | 1601         | 400MiB/s             |
| 2_write_16_256k                     | 1581         | 395MiB/s             |
| 2_write_32_256k                     | 1163         | 291MiB/s             |
| 2_write_64_256k                     | 834          | 209MiB/s             |
| 2_write_128_256k                    | 836          | 209MiB/s             |

### recover 性能统计流程

recover 性能统计


1. StatusServer::CollectMetrics()
2. ClusterSummary summary;
3. AccessManager::GetClusterPerf(summary.mutable_cluster_perf())
4. SummaryChunkPerf(const std::vector< ChunkPerf>& chunk_perfs, ChunkPerf* summary, bool has_cache)，传入的是 ClusterSummary.mutable_cluster_perf()
5. SummaryRecoverPerf(const std::vector< const RecoverPerf*>& perfs, RecoverPerf * summary)，传入的是 ClusterSummary.mutable_cluster_perf()->mutable_recover_perf()
6. 在 SummaryRecoverPerf 中，把 perfs 数组中的值做了个汇总，累加到 ClusterSummary.mutable_cluster_perf()->mutable_recover_perf() 中，这里为啥可以直接累加？

心跳上报到 access mgr 里的 session 的 chunk_perf

1. AccessManager::HandleAccessDataReportRequest
2. AccessHandler::HandleAccessDataReport()
3. AccessHandler::ComposeAccessDataReportRequest(meta::AccessDataReportRequest *request)
4. RecoverHandler::ListRecoverAndMigrateInfo(&recover_list, true)，其中 ListRecoverResponse recover_list;
5. ComposeRecoverPerf(ListRecoverResponse* recover_list, RecoverPerf* perf)

AccessHandler::ComposeAccessPerf(AccessDataReportRequest* request, bool only_summary) 这个统计的是普通 io，ComposeRecoverPerf 统计的 reposition io 相关的。

### 副本剔除

引发数据恢复一般是副本失联或副本剔除，副本失联可能是 Chunk 服务中断、节点异常（关机/网络隔离）、Chunk 上包含 Extent 的磁盘故障。剔副本的 4 种情况：

1. sync gen 失败的副本，比如突然下电/程序崩溃等现象造成的目标副本 gen 落后于其他副本；

2. recover handler 在 SetupRecover 时遇到 lease 提供的 loc 中已经包含 dst cid 且 src cid 的 gen 是安全的；

3. access io handler 在 write replica done 时会剔除写失败的副本，比如可能是磁盘损坏/拔盘等物理故障引起的副本状态异常，access 与目标 chunk 之间的链路异常；

    对于磁盘异常造成的错误，可以在目标 chunk 的日志中查找对应的 pid ，通常会伴随 local io failed 的日志会显示无法 IO 的原因。可以通过在目标节点上的 /var/log/message 里搜索 Abort Task 来检查是否有物理磁盘 IO 异常的信息。如果有则通常是磁盘损坏或者磁盘固件版本不对，如果多个磁盘均有出现则有可能是 HBA 卡异常或者固件版本不对。

4. 临时副本重放完会被剔除。

### access point

zbs 当前行为：

1. 每 1 min 判断一次，如果这个期间内卷的外部 io count > 3600 （除下来就是 iops 要大于 1）并且 io len > 511 * io count（用以跳过 nfs hearbeat io），那么认为它是 active volume，ZbsClient::CheckVolumeActive。
2. zbs chunk 每隔 1h 上报一次 active zbs volume 的信息给 zbs meta，不会上报 inactive volume，Meta::ReportVolumeAccess。
3. 对于一个 zbs volume，zbs meta 收到 6 次 volume 信息上报后（即 6h 后），会更新其 prefer cid 字段，同时更新自身持有的那些 prefer local 不符合预期的数据块，MetaRpcServer::RefreshVolumeAccessPoints。



允许在 server san 模式下人工指定 access point，融合模式下，接入点总是快速的自动切换至本地，因此对于融合模式下的 Target 执行对应的配置不会产生期望的效果。

当一个共享卷被大于等于 3 个接入点同时访问时（Xen 平台的 iSCSI DataStore 模式，Xen 将一个 LUN（Volume） 作为一个池，不同节点上的 VM 将仅访问其中的部分数据） ，将不会触发 Local Transfer 事件，相关的 Extent 会保持初次写入时的接入节点作为 prefer local。

在副本分配策略 in ZBS 中搜 prefer local。

每个数据链路会尽量分散给不同的 Access 处理，尽量避免链路资源竞争（在这里就可以把数据链路理解成 access point）

https://docs.google.com/document/d/1rcpxCZDNb7YFnEYVIJg-WzZVCzI8rArTFuzLNjJLNhc/edit#heading=h.gye9t51u3igb

从计算端查看数据接入点有两种方式

```shell
netstat -antp | grep 3261
# zbs 中同一 client 连接多个 target 一般会分散在不同节点接入
iscsiadm -m session -P3 | less
```

为 Volume 增加 access point （cid 列表）属性， Extent 增加 Prefer Local （cid）的属性，用于表示数据的访问点。



iscsi access point 3 部分策略：iscsi 建立连接、异常重定向、动态平衡

修改 iscsi 接入点选择策略：双活集群下，Target 开启外部接入模式时（非 qemu 发起的 iscsi 连接），如果主可用域至少有一个节点可用，则必须选择在主可用域中的节点作为接入点。如果主可用域没有可用节点，则返回次级可用域的接入点。

修改 iscsi 接入点平衡策略：（每 3 分钟执行一次接入点平衡检查，每次检查最多移动一个接入点）
平衡策略目标是：使得每个接入点的数量在主可用域的节点之间内尽可能平均，次可用域内 iscsi 接入点数量应当为 0。如果 iscsi 接入点已经在主可用域上，即使主可用域所有节点都宕机，也不允许主动迁移和修改 access record 到次可用域。客户端发起重试后，会返回次可用域的临时接入点， access record 仍然在主可用域。

临时异常回切：对于临时分配到次可用域的接入点，一旦主可用域有一个节点恢复，且该节点对应的 access session 存活超过一定时间（3分钟），则自动平衡检查时应当尝试将其迁移回主可用域。

https://docs.google.com/document/d/1t14uKF6YCaijgXAq-bS-WR_I1SaLhYxbOnKXhspBtlQ/edit#heading=h.iidguj2la1

### zbs 线协程模型

zbs 内部的线程基本都用 ThreadContext 来代替 std::thread 这种裸线程，也就是说

Timer 和 TimerHandle 是有区别的，TimerHandler 是 Coroutine 和 timer 的结合，是具备 co 语义的。

```c++
void TEST_DoScan() {
	  // 若调用方没跑在 ThreadContext 中，比如单测使用的没有 zbs co 语义的线程或者
    if (ThreadContext::Self() == nullptr) {
        Sync sync;
        thctx_->Sched(new CoroutineClosure([&]() {
            DoSomething(); // 业务代码
            sync.Run();
        }));
        sync.WaitAndReset();
    } else {
        // 若调用方跑在 ThreadContext，可以通过 ThreadContextGuard 把 co 调度到 thctx_ 所在线程运行
        ThreadContextGuard tcg(thctx_.get());
        // 为了避免调度过来的 co 被 yield
        CoLockGuard l(&co_mutex_);
        DoSomething();		// 业务代码
    }	
}
```

RWLock 是个线程间的读写锁，如果需要互斥的 2 个线程有 co 语义（ThreadContext），其中一个线程 co 因为拿不到锁而阻塞，那么其内部的 coroutine 也没法调度，可能出现一个 rpc 阻塞着，其他 rpc 也没法响应的情况。

CoRWLock 是协程间的读写锁，



SessionExpiredCb 是个 Timer 触发的，不是在一个 co 里，所以它调用的函数里，如果要

```c++
void RecoverManager::ClearSessionCmd(const std::string& session_id) {
    Sync sync;
    thctx_->Sched(new CoroutineClosure([&session_id, &sync, this]() {
        CoLockGuard guard(&co_mutex_);
        // 业务代码 ....
        sync.Run();
    }));
    sync.WaitAndReset();
}
```





### zbs io 流

zbs app io trace

1. --> ZbsClient::Write() --> ZbsClient::DoIO<>() --> ZbsClient::SplitIO() --> ZbsClient::SubmitIO() --> InternalIOClient::SubmitIO()
2. --> 判断走网络还是本地，VExtent 粒度，路由到合适的 AccessIOHandler，此时有获取一次 lease，根据 lease owner uuid 跟本地 session uuid 判断是否相等
3. --> AccessIOHandler::SubmitWriteVExtent() --> AccessIOHandler::WriteVExtent() 
4. --> Meta::GetVExtentLease() -->Meta::CacheVExtentLease() --> Meta::CacheLease() ，此时也有一次获取 Lease，是根据 volume_id 和 vextent_no 拿到 pextent 的 location 等信息，对 location 上的每一个 cid 执行下面操作
5. --> ECIOHandler / ReplicaIOHandler::Write() --> PextentIOHandler::Write() 
6. --> 判断走网络还是本地，PExtent 粒度，本地的话直接在 PextentIOHandler 塞入队列，如果是走网络，是通过远程 Chunk 上的 LocalIOHandler 塞入队列
7. --> LSMCmdQueue::Submit，队列中的元素会被 lsm 根据不同的 op code 执行不同的操作，比如 LSM_CMD_RECOVER_START 等；
8. --> LSM::DoIO() --> LSM::RecoverWrite() --> LSM::DoRecoverWrite() --> ExtentInode::SetupRecoverWrite() ，此时会有 pblob 层面的操作。



zbs reposition io trace

1. --> RecoverManager::GenerateRecoverCmds() --> RecoverManager::AddRecoverCmdUnlock() --> AccessManager::EnqueueRecoverCmd()

2. --> AccessManager::ComposeAccessResponse() --> 通过心跳下发给 access handler --> AccessHandler::HandleAccessResponse()

3. --> RecoverHandler::NotifyRecover() --> RecoverHandler::TriggerMoreRecover() --> RecoverHandler::ScheduleRecover() --> RecoverHandler::DoScheduleRecover()

4. 根据 resiliency_type，分别给到 ECRecoverHandler / ReplicaRecoverHandler

5. --> ReplicaRecoverHandler::HandleRecoverNotification() --> ReplicaRecoverHandler::DoRecover() 

6. --> ReplicaRecoverHandler::RecoverStart() 

   1. --> PExtentIOHandler::SyncRecoverStart() --> PExtentIOHandler::RecoverStart()
   2. --> ReplicaRecoverHandler::GetReplicaGeneration() ，向 dst_cid 获取 gen 并校验；

7. --> 按 256 KiB 为粒度，做 1024 次 ReadFromSrc() + WriteToDst() 

   1. --> ReplicaRecoverHandler::ReadFromSrc() --> PExtentIOHandler::RecoverRead() 

   2. --> 判断走网络还是本地，PExtent 粒度，本地的话直接在 PextentIOHandler 塞入队列，如果是走网络，是通过远程 Chunk 上的 LocalIOHandler 塞入队列

   3. 读是没法避免的，而写的话，如果读的时候 LSM 返回 ELSMNotAllocData 且支持敏捷恢复和 unmap 时，会通过 unmap 来完成这次 recover block write；如果走普通恢复并且 ELSMNotAllocData 或用 all_zero_buf 写 dst cap layer，那么直接完成本次 recover lock write，不会有真实的 IO 流量。

      > lsm 会记录 perf 层有效数据 bitmap，所以如果用 all_zero_buf 写 dst perf layer 不能直接跳过

   4. --> ReplicaRecoverHandler::WriteToDst() --> ReplicaRecoverHandler::DoRecoverWrite() --> PExtentIOHandler::SyncRecoverWrite() --> PExtentIOHandler::RecoverWrite()

   5. --> 判断走网络还是本地，PExtent 粒度，本地的话直接在 PextentIOHandler 塞入队列，如果是走网络，是通过远程 Chunk 上的 LocalIOHandler 塞入队列

8. -->ReplicaRecoverHandler::RecoverEnd() 

9. --> ReplicaRecoverHandler::ReplacePExtentReplica()



LSMCmdQueue 中的元素在哪被消费呢？

1. EpollLSMIOContext 的 Flush() 把 LsmCmd SubmitIO 给 lsm；
2. LSM::Init() ，将 cmd_event_poller_ 设为 LSM::HandleCmdAndEvents()，在这里会用 coroutine 执行 LSM::DoIO()；
3. LSM::DoIO()，其中，根据不同的 op code 执行不同的操作，比如 LSM_CMD_RECOVER_START 等；
4. LSM::RecoverWrite()，LSM::DoRecoverWrite()；
5. ExtentInode::SetupRecoverWrite()，会有 pblob 层面的操作。

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

### access io 并发控制

access 中 app io 写需要拿读屏障，recover io （recover io 读写是一体的）用写屏障，这会使得多个 app io 写之间不会互斥，而 recover io 会跟 app io 写以及其它 recover io 互斥。

1. app io 读不需要拿读/写屏障，所以 app io 读可以和 app io 写、recover io 并发；

2. 允许多个 app io 写并发是因为块设备不需要保证同一个 lba 上 inflight io 的先来后到；

3. recover io 需要保证跟 app io 写以及其它 recover io 互斥，这是因为需要保证多个副本之间的内容是一样的。一个 recover io 包含一次读和一次写，如果 recover io 读后有 app io 写，那recover io 写可能会覆盖 app io 写，导致多副本间内容不一致。

互斥/并发都是按 block 粒度的，据此可以得到：

1. 若当前正在 app read block a，不论再来什么 io，都无需阻塞即可执行；
2. 若当前正在 app write block a：
   1. app read block a 无需阻塞即可执行；
   2. 另一个 app write block a 无需阻塞即可执行；
   3. recover block a 需要阻塞等待 normal write 完成才能执行；
3. 若当前正在 recover block a：
   1. app read block a 无需阻塞即可执行；
   2. app write block a 需要阻塞等待 recover 完成才能执行；
   3. 另一个 recover block a 需要阻塞等待前一个 recover 完成才能执行（不过 recover iodepth = 1，只有前一个  block read/write 都执行完才会执行下一个 block 的 read/write，理论上不存在这种情况）；

具体在代码里搜 extent_io_barrier，在 replica_io_handler 和 replica_recover_handler 之间起作用。

另外，access io handler 内部负责非 recover IO 的 3 种类型的 internal IO 以及 app io 间互斥。sink io 会跟 app io 互斥，具体通过代码 LockBlock 和 WaitBlockUnlock 分析 sink io 的读还是写与 app io 的读还是写互斥，LockBlock 和 WaitBlockUnlock 可以看成同一个读写锁，前者像读锁，后者像读锁。

access 按一个 pextent 的 gen 不降序的方式顺序下发 io co 给 lsm（有这么个保序机制），lsm 收到后内部并发执行 io co，在 lsm 内部，如果这些 io co 写的是同一个 pblob，还是需要加锁，如果是不同 pblob，就可以并发执行。

### sync gen

access 从 meta 拿到的 lease 中的 location 是 loc 而不是 alive loc，可参考 GenerateLayerLease()，在 sync gen  是对 loc 而不是 alive loc 上每个 cid 都 sync，实际上，让 access 做一下 sync 真正确定一下这个副本是否连通比 meta 给出的信息更靠谱，因为这个 chunk 有可能跟 meta 失联，但还跟其他 chunk 联通，此时的失联 chunk 还是可以被读写副本的。

### remove disk

卸载盘调的是 chunk rpc server 的 UmountCache/UmountPartition rpc，没做空间校验，meta 侧没参与，tuna 那边也是放行的，所以变成 tower 在做。

tower 卸载物理盘限制如下（zbs 5.6.x）：

1. 存在数据恢复： 获取 [zbs_cluster_pending_recover_bytes](https://smartx1.slack.com/archives/C06T7HMAV5J/p1717136802475749?thread_ts=1716789334.665009&cid=C06T7HMAV5J) metric 数据，不为 0 则说明有数据恢复；

    1. 判断待恢复数据是否存在，使用 zbs-meta pextent find need_recover；
    2. 判断待恢复数据量，使用 metric zbs_cluster_pending_recover_bytes（zbs status server 对外暴露的时候提前加了，这个 metric = pending_recovers_bytes + ongoing_recovers_bytes），大部分情况下，能反映真实待恢复数据量，当比如剩余空间/节点/可用下发命令额度不足导致选不出 recover src/dst，这个值就会有偏差。

2. 存在其它卸载的盘，需等待卸载完成；

3. 不能卸载所属主机最后一块包含数据分区的物理盘，数据分区判断：partitions有 usage 为partition；

4. 不能卸载启动盘 ，启动盘判断：usage 为 boot；

5. 空间不够不可卸载，从 /v2/storage/storage_pools 和 /v2/cluster/storage api 处获取空间字段；

    对于任意一个待卸载盘上的 cache 分区大小记为 disk perf size，partition 分区大小记为 disk cap size。tower 卸盘空间检验需同时满足以下 3 个条件：

    1. disk perf size < cluster perf valid space - cluster perf allocated space，含义是待卸载盘上 cache 分区空间不该超过集群 perf 层的剩余可用空间；
    2. disk cap size < cluster cap valid space - cluster cap allocated space，含义是待卸载盘上 partition 分区空间不该超过集群 cap 层的剩余可用空间；
    3. disk perf size * prs_ratio < cluster perf thick valid space - cluster perf allocated space，含义是卸盘后，集群 perf thick 的使用量不该超过总量；

    在收到卸盘请求后，lsm 将本轮的 perf valid - disk perf size 作为下一轮心跳上报的 perf valid 值，cap valid 同理，同时执行盘间迁移（限流 flow control 使用的负载判断字段需要调整）。

    若卸盘后该节点的 perf/cap allocated > perf/cap valid，需要等待节点间的容量均衡迁移迁出部分数据才能完成盘间迁移。但容量均衡迁移可能因为拓扑降级或双活场景下单 zone 内空间已满而无法执行，导致盘一直卸不掉。作为已知问题，此时需要人工介入，zbs cli 提供了 zbs chunk cache / partition cancel-umount 命令。

    这个空间检查只在卸盘前，卸盘过程中可能会有新数据写入，所以还需要多加个 2% - 5% 预留空间来做更严格的判定？目前未执行。

前端在存储健康状态的判定基础上加了 job center 的判断（connected），如果卸载磁盘时，jc-worker 异常，不加这个判断的话，任务可能可以提交成功，但迟迟不会开始执行，直到 jc-worker 恢复正常。

chunk 日志中搜 migrate pblob skip 会显示卸载盘卸不掉的 pblob，有 MIGRATE BLOCK DEVICE SUCCESS 表明卸载成功，搜 LSM::MigrateBlockDevice() --> SubmitBlockDeviceMigrate() --> MigratePBlobsOnBlockDevice() 可以看到卸盘逻辑

### remove chunk

1. zbs-deploy-manage storage_pool_remove_node < storage ip> 
    1. 这个命令会调用 zbs 侧的 RemoveChunkFromStoragePool rpc，只做剩余空间检查，检查通过后，chunk 状态改成 REMOVING，日志里出现 REMOVE CHUNK FROM STORAGE POOL；
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

### lsm 行为

zbs 5.6.x 中将 cap 层数据读到 cap read cache 的条件是 40 分钟内读 3 次，每次间隔 15 秒以上。要特别注意的是，一定是读的下沉后的数据。



LSM 在处理写 IO 时，有三个性能拐点：

1. LSM 以 8KB 为单位存储 Checksum，这意味着任意一个小于 8KB 的写操作，都需要从硬盘中读取整个 8KB，更新 Checksum 后再重新写入磁盘，这种读后写会损害小 IO 的性能。**如果我们的编码块小于 8K，每次对数据块的更新都会有读后写问题，对性能损害是非常大的**。
2. 当写入的块为 256KB 时，LSM 可以将 IO 直接写入磁盘而不用先写入 Journal。
3. LSM 支持 512e 磁盘，写入大小小于 4k 时，物理磁盘会有读后写问题。



lsm 中单个 Extent 内部对于并发 IO 存在一定的限制：

1. 一个 Extent 的 IO 都需要经过同一个 Journal。一个 Journal 只存储在一个 SSD 磁盘上，无法发挥出多块 SSD 的性能。
2. 属于同一个 Extent 的 pblob 会被优先分配到一个盘上，无法发挥多块磁盘的性能（特别是 hdd cap layer），假设有 4 块盘，容量都充足，分配 extent 1 2 3 4，是会分别给到盘 1 2 3 4，数据盘已使用容量标准差小于 0.2 的时候轮转，否则按容量均衡分配 extent 所在盘。

基于这两点，在顺序 IO 的场景下，性能受到很大的限制。而条带化的目的，就是在顺序 IO 的场景下，将 IO 分散到多个 Extent，利用多 Extent 之间的并发性，提高顺序 IO 的性能。

假设条带数为 4，每一个 extent 中包含 8 个 block，在进行条带化处理后，顺序 IO（V1-B1，V1-B2，V1-B3，V1-B4，V1-B5，V1-B6，V1-B7，V1-B8），则转换为（V1-B1，V2-B1，V3-B1，V4-B1，V1-B2，V2-B2，V3-B2，V4-B2）。这样，就可以利用到 V1，V2，V3，V4，4 个 Extent 的并发性。

### elf 行为

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



### thin/thick 分配

transaction 中，判断 thin/thick 的依据，cap 用 thin_provision_ ，perf 用 prioritized_ ，因为一个 transaction 中既可能分配 perf 也可能分配 cap，他们的 thin 和 thick 可能不一样。



分配一个 thick pextent，会马上分配 pid 的 location（此时的第一副本会挑选为分配时集群空间比例最小的节点，其他副本位置再按照局部化原则选择），然后在 transaction 的 Commit 中会让 location 上每个 replica 的 last_report_ms = now_ms，所以此时也马上会有 alive_location = location。

分配一个 thin pextent，直到初次写之前，他的 location 都是空的，所以 alive_location 也为空。

分配一个 prefer local = 0 的 thick volume 时，会立马分配 cap loc，在低负载且 topology 不为空时，没有 prefer local 后，也不是按节点容量最低的作为第一个节点，而是会随机选第一个节点（只要剩余空间大于 pextent size），然后按 ring id 找他后续的节点。



待核实：

分配 pid 跟分配 location 的时期应该不同。

分层之后，创建一个 lextent 时，会马上分配 perf / cap pextent，一定会给 perf 分配 location，如果 cap 的 origin pid 是 0，而且是 thick pextent，也要为 cap 分配 location，否则不分配

分配 pid 是先一个 lextent 中的所有 cap pid 都分配出来，再在有需要时分配 perf pid





### Clone, Snapshot, COW

如果一个 vextent 被打了 cow flag：vtable 上面会变成 cow=1 .... lid = 1[perf_pid = 1, cap_pid = 2]。

- 如果对这个 vextent 发起写，会把这个 vtable[vno] 改成 cow = 0 ... lid = 2[perf_pid = 3, perf_origin_pid = 1, cap_pid = 4, cap_origin_pid = 2]，获取到的 lease 是 lid = 2 的。

- 如果对这个 vextent 发起读，vtable[vno] 不变，获取 lease 是 Lid = 1 的，Child 还没出现，也不会 RefreshChild。



这方面的内容应该通过在 tiering_test.cc 中写单测来感受？直接看代码理不顺。



CreateSnapshot 只是将

快照一定是个非 prior volume，且它的所有 vextent COW flag 一定都是 true，这样可以保证快照只读的假设。因为对这个快照的写，要申请 is_cow = true 的 GetLeaseForWrite，那么一定会 COW 出一份新的 vtable 和分配新的 pid，在新的上面写。



假设 A 的 origin id = B，发生快照/克隆后 origin id 的变更情况（MetaRpcServer::CreateSnapshot / CloneVolumeTransaction）：

1. 从 A 克隆出 C，当 A 是快照，那么 C 的 origin id = A，A 的 origin id = B；
2. 从 A 克隆出 C，当 A 是卷，那么 C 的 origin id = 0，A 的 origin id = B；
3. 从 A 快照出 C，不区分 A 是快照/卷，C 的 origin id = B，A 的 origin id = C；
4. 从 A 回滚到 C，不区分 A 是快照/卷，C 的 origin id 不变，A 的 origin id = C；



明确以下分层之后，转换/克隆出一个普通卷的流程，包括 lextent, pextent 分配等。 

追踪克隆一个普通卷的流程

 lsm 现在上报 pextent info 的时候，如果 pextent 在 perf layer，不论 thick/thin，都会将 prioritized 字段设为 true，所以会出现单测创建 perf thin 之后，出现 FOUND REPLICA NEEDS CHANGE PERFHINT，但 LSMProxy::SetExtentsPriority 目前是个空方法，所以没啥影响。

 分层之后，创建一个 volume：

 1. 在 CreateVolume rpc 中先做各类参数校验，通过后执行 CreateVolumeTransaction

    1. cap pextent：不论是否开启分层，不论是什么类型的 volume 都会马上分配 cap pid，但只有 prior 或 thick 才会马上分配 cap location；

       > 如果 thick volume 的 lextent 的 cap pextent 不马上分配 location 来预留空间的话，没法保证有足够的下沉空间。

    2. perf pextent：开启分层并且是 prior volume 才会马上同时分配 perf pid 和 location。

 2. 在 CloneVolume 时，CloneVolumeTransaction 根据源卷是快照或普通卷有不同行为

    1. src volume 是快照 
       1. 从 meta db 中拿到 src volume 的 vtable 并拷贝一份给 dst volume ；
       2. 设置 dst volume 的 origin id 为 src volume id；
    2. src volume 是普通卷
       1. 加锁
       2. 从 metadb 中拿到 src volume 的 vtable；
       3. revoke src volume 的 vtable lease；
       4. 将 src vtable 上各个 vextent cow flag 设置为 true，
       5. 解锁
       6. 清空目的卷的 origin id（因此源卷是普通卷，很快就有新的写数据，那就跟 dst volume 不一样了）

    二者共同操作包含：

    1. 将目的卷标记成不是快照；
    2. 从 src volume 复制一份 vtable 给 dst volume；
    3. 若 dst volume 尺寸大于 src volume，按照 CreateVolume 中的原则分配新的 cap/perf pextent，这些新的 vextent 对应的 vextent 上 cow flag = false。

 3. CowLExtentTransaction

    待补充

 4. AllocPExtentTransaction，已有 lid，且 COW = 0 时被调用

 5. AllocVExtentTransaction，在  (vextent_no < num_vextents && lid = 0) || (vextent_no >= num_vextents)   时被调用，只会发生在 NFS overflow write 时；

 6. 

在分层之后，更新一个 Volume：



快照会将 VTable 复制一份，Vtable 的内容就是若干个 VExtent，里面只有 3 个字段，vextent_id，location，alive_location，第一个字段是 volume 的 offset 与 pextent 的对应关系，后两个字段就是对应 cap pextent 的 location 和 alive_location，不同于 pextent table 常驻内存，vextent table 常驻 meta db，需要时从 meta db 中读。

COW PEXTENT 的触发时机是 GetVExtentLease rpc，如果 access/chunk 那里 lease 没有 expire，也就不会调用 GetVExtentLease，所以需要通过 revoke 主动让他 expire。COW 是先 revoke，然后打快照，保证了快照后，extent 无法写入的语意，如果不 revoke lease，快照只读假设将被打破。

COW 之后，child alive loc 不一定等于 parent alive loc。实际上，COW 在 Transaction Prepare 的 CowPExtent 时只会只会复制 parent 的 loc，然后在 Commit -> PersistExtents -> UpdateMetaContextWhenSuccess -> SetPExtents 时会将 loc 上的每一个副本的 last_report_ms 设为当前时间，所以 child alive loc = child loc = parent loc，但是不一定等于 parent alive loc。

### 560 空间计算

volume 级别的 metric 中各空间字段含义

* logical_size = vtable_size * 256 MiB，vtable_size 就是一个 volume 里 lid 的数量，每个 lextent 只算单个分片，且是固定的 256 MiB。

* logical_used_size = 对每个 lid，取他的 cap / perf pextent 中 logical_used_size 最大的那个。

    对于每个 pextent 的 logical_used_size：（PhysicalExtentTableEntry::GetLogicalUsedSpace）

    * thick，不论 replica 还是 ec shard，都是 256 MiB；
    * thin，对于 replica 是 lsm 汇报的所有 replica space / get_replica_num(location)  ，对于 ec shard 是  lsm 汇报的所有 ec shard space / 数据块个数（不是数据块 + 检验块），算的平均每个分片的 lsm 汇报的空间占用，是 256 KiB 的倍数。

* perf_unique_size = 持有的所有 perf pextent 独占的所有分片的 allocated size 总和

* perf_shared_size = 持有的所有 cap pextent 共享的所有分片的 allocated size 总和，如果一个卷不曾发生过克隆/快照，shared_size 始终为 0。

logical 不考虑分片数量，physical 考虑分片数



5.6.0 中，lsm 对外暴露 GetPerfSpaceInfo，其中引入 PerfSpaceInfo 给 access 限流/下沉使用（meta 没用）

```c++
struct PerfSpaceInfo {
    size_t thin_used;
    size_t thin_free;
    size_t thin_valid;
    size_t thick_reserved;
};
```

其中，

-  space_info.thin_free + space_info.thin_used 不等于 space_info.thin_valid；
-  space_info.thin_used 可能大于 space_info.thin_valid，当发生拔盘，卸载盘，或者快照克隆时会发生；
-  access 限流 block 分配时，使用 space_info.thin_free / space_info.thin_valid 来判断是否要限流，值越小，越需要限流；
-  access 是否加速下沉，使用 space_info.thin_used / space_info.thin_valid 判断，是否需要加速下沉，值越大，越需要加速下沉。

#### space

1. perf_total_data_capacity = (1 - GFLAGS_chunk_lsm2_cache_fixed_ratio) * total_cache_capacity

   默认等于 0.8 * total_cache_capacity，total_cache_capacity 表示 cache 分区总容量；

2. perf_valid_data_space =  perf_total_data_capacity - perf_failure_data_space

   当不存在坏盘时，perf_valid_data_space = perf_total_data_capacity，即不增减物理盘且不处于升级过程中，这个值应该是固定的。

   当集群升级期间开启分层后一段时间内，可能存在大量的 dirty space 需要 writeback，此时 LSM 无法保证这些空间被性能层及时使用，所以在这段期间，perf_valid_data_space = perf_allocated_data_space + valid_free_cache_space + cap_cache_clean_space，随着回写完成，valid_free_cache_space 变大，perf_valid_data_space 也跟着变大。

3. perf_planned_space = min(perf_total_data_capacity, (perf_allocated_data_space + GFLAGS_chunk_lsm2_perf_fixed_free_ratio * total_cache_capacity))

   默认等于 0.1 * total_cache_capacity + min(perf_allocated_data_space, 0.7 * total_cache_capacity)

4. cap_cache_capacity = total_cache_capacity - perf_planned_space

   算的是 cap cache 理论可占用容量上限，默认等于 0.9 * total_cache_capacity - min(perf_allocated_data_space, (0.7 * total_cache_capacity))，未在 ChunkSpaceInfo 中提供

   1. 当 perf 实际使用空间为 0 时，容量层读缓存空间取得最大值，0.9 * total_cache_capacity；
   2. 当 perf 实际使用空间超过 0.7 * total_cache_capacity 时，容量层读缓存空间取得最小值，0.2 * total_cache_capacity；

   cap cache 理论可占用容量上限不会低于 20%，但 cap cache 实际可能只占用 10%，那么此时可以用 used_cache_space - perf_used_data_space 得到这个实际占用容量。据此，也可以得到实际 cap read cache 使用率等于 dirty_cache_space / (used_cache_space - perf_used_data_space)。

5. planned_prioritized_space = prio_space_percentage * perf_valid_data_space

#### 持有 pextent 特点

prioritized_pids 就是 perf_thick_pids，因为 perf 层只会有 perf thin 和 prior 两种类型的 extent，不会有非 prior 的 thick extent

prioritized_rx_pids 是 perf_rx_pids 的子集，一定被包含在 perf_rx_pids，除了在开启分层前的升级过程中产生  prior reposition 的话，prioritized_rx_pids 有部分 pid 是给到 cap_rx_pids 而不是 perf_rx_pids

有如下等价关系：

* allocated_prior_space = prioritized_pids + prioritized_rx_pids
* perf_pids = prioritized_pids + perf_thin_pids_with_origin + perf_thin_pids_without_origin

为啥没有 prioritized_tx_pids 和 prioritized_recover_src_pids ？因为不需要，要有一个单独的 prioritized_rx_pids 是为了在算 allocated_prior_space 的时候能把为 reposition 预留的算进去，而 tx 不加是因为 allocated_data_space 就算多算了也没事，过一段时间完成 reposition 之后 perf_tx_pids 也就被 erase 了。



cap_pids，除了 allocating / repositioning  的其他属于 cap 层的 pid 都会被记入 cap_pids，cap_pids 还一定包含 cap_tx_pids 和 cap_recover_src_pids（但不是仅由他们两组成的），一定不包含 cap_rx_pids ，与cap_reserved_pids 可能会有交集（取决于是否调用了 GetLeaseForRefreshLocation rpc，没调用的话是不会有交集的），也因此，求的 allocated_data_space 可能是略大的，因为有可能某个 pid 既在 cap_pids 又在 cap_reserved_pids。

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

这个空间大小也体现在 ongoing recover / migrate space，不过他们只算了 src 的空间。算节点已分配空间大小，不应该减去 cap_tx_pids，因为在这里的 pid 一定还在 cap_pids，并且当 reposition 成功，会有 RemovePextent 或者 ReplacePextent，到那时候会把 cap_pids 里面的相关 pid 去掉。



cap_reserved_pids，对应正在分配但还没分配成功的空间大小，先预留，把这部分空间占住，由于分配副本空间在 transaction 中很快完成，所以可以认为大部分时间 cap_reserved_pids 都是空的。

* add
    1. transaction 里面 CreateVolumeTransaction::Prepare() 会先调用 ReserveSpaceForAllocation() ，预留空间；
    2. 在 sync gen 时如果发现这是一个 parent 被迁移的 pextent，lease 上没有他的 location 等信息，会调用 MetaRpcServer::GetLeaseForRefreshLocation() 来从 parent pextent 中拷贝一份 location 信息出来，此时也会认为这个 pextent 没有 parent 了，那么他也要独立的占用空间，调用 ReserveSpaceForAllocation() 来预留空间，并马上通过 PhysicalExtentTable::SetPExtents() 把这部分预留空间删掉，即从 cap_reserved_pids 中删除，放入 cap_pids；
* delete
    1. add 里的两种情况会调用 FreeSpaceForAllocation()，对应 SpaceTransaction 的析构函数和 GetLeaseForRefreshLocation() 的 done 代码段的操作；
    2. 调用  ChunkTable::AddPExtent() 也会将 pid 从 cap_reserved_pids 中删除，放入 cap_pids，对外的接口是 PhysicalExtentTable::SetPExtents()，而这除了上面那个特殊的 rpc 外，就是在 transaction 中的 Commit 阶段的 UpdateMetaContextWhenSuccess() 中会调用

### Oracle RAC

1. Oracle RAC 节点重启
2. 节点未重启但数据库无法正常响应



触发原因

1. ocssd 仲裁磁盘上的 IO 心跳超时并且在 ocssd.bin 重启后无法恢复（默认的 disktimeout=200s）
2. ocssd 网络心跳超时（默认的 misscount=30s）
3. Oracle ASM disk group 被卸载 + 客户端 vip 配置错误



可能由 Oracle RAC 节点间 Private 网络中断，或 Oracle RAC 节点上的仲裁磁盘、根分区磁盘 IO 中断引起。

如下两种情况可能导致 Oracle RAC 节点重启：

- ocssd 仲裁磁盘上的 IO 心跳超时并且在 ocssd.bin 重启后无法恢复（默认的 disktimeout=200s）

- ocssd 网络心跳超时（默认的 misscount=30s）

    

数据库无法正常响应，

可能有如下原因 Oracle ASM disk group 被卸载 + 客户端 vip 配置错误（）：



- 磁盘组中的磁盘无法响应 PST 心跳（超时时间 _asm_hbeatiowait）。如果 ASM 无法向磁盘组中多数副本发出 PST 心跳，磁盘组将被卸载。分为下面三个情况：
    - 正常冗余（normal redundancy）的磁盘组无法更新一个副本，磁盘组将被卸载（umount）；
    - 高冗余（high redundancy）的磁盘组无法更新两个副本，磁盘组将被卸载；
    - 灵活（flex redundancy）的磁盘组提供两副本或三副本，同上；
    - 外部冗余（external redundancy）的磁盘组遇到任意的写入错误，磁盘组都将被卸载；不建议设置为 external redundancy。



对于 cmp

1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false
