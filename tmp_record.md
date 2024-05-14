一个是 tuna 那报的一个问题，从 5.0.3 升级到 5.0.7，有个节点 IO 重路由状态检查失败，上去看了下，有个现象是会删除本地存储 ip，然后又把他添加回来，看了下应该是这个版本里对 session alive 的判断逻辑有问题，当时没有 session alive 字段，他是自己写的一套判断逻辑，应该是有点 bug，还没来得及继续调查，还会出 5.0.8 吗？还需要更细致的调查吗？



五一期间 ESXi 升级后 SCVM 所在主机的 IO 重路由服务停止工作

8 个节点有 2 个出现问题，看 ESXi 的日志发现这两个节点在 4 月中旬做 scvm 的升级的时候 io reroute 就有问题了，scvm 升级之后，在 io reroute 升级时，scvm 会通过 ssh 的方式在每台 ESXi 上执行多个 cli 用以杀死旧进程，更换最新的 reroute 脚本，使用新的 reroute 文件并更新 crontab。

这个过程中需要获取 reroute 脚本所在的 datastore，这个 ssh 的过程中如果密码/密钥对不上或者网络 Timeout 时会将 datastore_path 认为是 None，然后直接用 None 去拼接路径字符串，填到 crontab 里，导致有问题。

这里没有做好 ssh 异常的处理，是 6 年前就有的问题了，不过现在才暴露，薛总说是这个客户机器物理环境很差，机房非常热，可能是导致 ssh timeout 的原因。





SCVM 升级之后，在 IO Reroute 升级时，SCVM 会通过 ssh 的方式在每台 ESXi 上执行多个 cli 用以杀死 reroute 旧进程，更换新版本 reroute 脚本，更新 crontab 中新 reroute 脚本位置并等待唤起新进程。

当 SCVM ssh ESXi Timeout 时，由于没有正确处理异常，会将 datastore_path 认为是 None，路径被拼接成一个错误的 "/vmfs/volumes/None/vmware_scvm_failure/reroute.py" 

crontab 没能正确找到 reroute 脚本所在位置，reroute 进程没起，引发 Tower 报警。





fio -ioengine=libaio -invalidate=1 -iodepth=128 -ramp_time=0 -runtime=300000 -time_based -direct=1 -bs=4k -filename=/dev/sdc -name=wrtie_sdc -rw=randwrite;



esxcfg-route -d 192.168.33.2/32 10.0.0.22; esxcfg-route -a 192.168.33.2/32 10.0.0.21; sleep 3; esxcfg-route -d 192.168.33.2/32 10.0.0.21; esxcfg-route -a 192.168.33.2/32 10.0.0.22; 



执行以下操作前，需要保证所有 ESXi 上的 192.168.33.2 的下一跳指向本地 SCVM 存储 IP：

在所有 SCVM 节点上执行：

1. 进入 /usr/share/tuna/script/scvm_failure_common 目录，在后面的操作中，不要切换目录。 
2. 备份已有 io reroute 脚本。命令：mv reroute.py reroute.py.bak
3. 将新的 reroute.py 脚本复制到同级目录；
4. 检查新的 reroute.py 脚本 md5sum = 60b0c13cd5680afb4f71c65a4785a07f
5. 将同级目录中的 rereoute_version 文件中记录的版本号从 2.2 改成 2.2.1；

在任一 SCVM 节点上执行：

1. zbs-deploy-manage update_reroute_version




1. zbs-chunk migrate list 中
    1. 只有 lease owner 上的 Total Migrate Speed 才有值，而这也是会给到 meta 的值，才会有 reposition list，其中 STATE = INIT 的 pid 表示在 recover handler 的 pending 队列中，STATE = READ / WRITE 的 pid 表示正在执行，
    2. From Local Speed 指的是该节点作为本地
2. set_mode_info  rpc 兼容性处理
2. 分开设置 recover 和 migrate 的单次 scan 上限，generate recover cmd 的过程中如果有了 migrate cmd，是可以打断他的。


    1. 把 recover 的分页跟 Migrate 的独立开来，并改大点。减少还有待恢复数据但却先下发 migrate cmd 把 cmd slot 等资源用满的情况；
    
    2. 如果 generate recover cmd ，可以清空待下发的 migrate cmd 吗？
    
        已下发的 migrate cmd，只要他还没完成，recover handler 收到同一 pid 的 recover cmd 会被直接丢弃，不会执行。
    
    3. recover handler 中的执行队列，可否做成 ever exist = false 且 origin_pid = 0 的 pid 优先执行，其他 pid 按 FIFO 的顺序执行。
    
    4. 进出维护模式，把 migrate cmd 清掉，防止他抢资源，让升级快点结束。进出维护模式的时间应该不长，这段时间内的 migrate 重要吗？
    
    4. recover manager 对于没有实际分配的数据会跳过命令下发配额的限制，快速下发给 access，如果这部分数据是从本地读，这个 recover 很快就会完成，但 recover handler 的 pending_recover_cmds_ 是按 FIFO 的顺序进 running_recover_pids_ 执行的，所以后发的符合上述特点的 pid 也没法快速执行，可能被前面执行慢的 pid 拖慢。

3. recover cmd 快速生成的逻辑在 arm 的环境中貌似没有起作用，因为分页 + pid 数量变大的原因，多扫一次的做法提升并不明显了，可能得多扫一轮？

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
    meta->GetLextent(lid, &lextent);
    
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

6. 编译换回 docker

7. 若已有 lease owner，他可能跟 src/dst cid 不同，如果是由于 src/dst 单点 IO 性能差造成的 auto mode 下缩小 lease owner 命令下发窗口，看起来是误判，实际上这种情况，下一次关于这个 pid 的 cmd 大概率还是会选到这个 lease owner，所以也不算误判。

    若没有 lease owner，新分配的 lease owner 副本模式下大概率是 src cid（除了 lease owner 本身分配优先选 src cid 之外，在下发前也有根据 lease owner 调整 cmd  src cid 的逻辑），较大概率是 dst cid，然后才是其他节点。

    另外，命令下发窗口可能被自动调节的前提是要打满一个窗口，也就是要有满一个窗口大小的命令数大部分都失败才有可能引发窗口收缩，比如 lease owner = 1, src_cid = 2, dst_cid = 3，若集群中只是 cid 2 IO 性能性能差，基本上需要给到 1 的 cmd src 基本都是 2 才满足整个窗口命令基本超时的条件，而此时把 1 的窗口跳调小也算正常，因为后续这些超时 cmd 的 pextent 再生成 cmd 时，lease owner 大概率还是 1。

8. 检查 recover/migrate speed 在前端界面和 prometheus 中的数值是否准确，meta 侧跟 chunk 侧的 total speed 和 local speed 和 remote speed

9. 调整 business io 影响内部 IO 的 iops 和 bps 阈值。

    1. 理论上我应该拿到所有磁盘类型+数量的上限后，用 GLAGS 去定义能给到 internal io 用的磁盘性能比例（0.5）和网络带宽比例（0.4 / 0.5），并给出 app io busy 的判断准则（比如  0.3 的上限，这样预留 0.2 出来做缓冲）。
    2. ssd 的限速不能直接跟盘成正比，主要是考虑到 zbs 没法发挥出磁盘性能上限，比如 4 块 nvme ssd 跟 2 块性能差不多。ssd 的 app io busy 先保留目前是一个定值的做法，但应该是一个变化值，综合考虑网络带宽以及磁盘性能，盘多了之后，瓶颈可能在网络带宽上，而网络带宽这事儿没法直接给出 app io busy iops（不管下沉数据，直接用 bps / 256 KiB？）
    2. 对 interval io 的判定除了 bps，是否需要把 iops 用起来？recover io 一定是 256 kb ，所以只关注 bps？sink io 有可能是 4k，所以应该关注 iops ？

10. 更新 recover / migrate 文档，看 zbs 已有临时副本相关文档，把 meta 的业务逻辑看懂之后，要看 zk 和 dbcluster 相关逻辑，access session 的建立/断开连接逻辑，看 meta2 文档，看 sink manager 和 drain manager 的区别。

11. 分配临时副本空间检查适配 pinperf in tiering，[ZBS-27272](http://jira.smartx.com/browse/ZBS-27272)，HasSpaceForTemporaryReplica 的修改，顺便把对 CowLExtentTransaction 的理解补充上

       1. prior pextent allocation

          升级到 560，但没有开启之前，不允许创建 prior pextent 的代码在哪里？

          replica_capacity_only 模式允许创建 prior pextent 吗？

          改动之后，可能的坑点：

          1. thick 有个最高 99%；
          2. temp pid 有个最高 95%；
          3. pid 分配 location 除了 ec 之外，并不会随机打乱 cid 在 loc 中的位置；

       2. piror recover

           先把 recover 关于 prior 的部分做完，等有空再考虑把 topo distance 做好，zbs4，另外，空间充足可以先过滤，但是尽量不选 isolated 和双活需要 2 ：1 的特性需要特别考虑。

           1. recover / removing chunk dst 允许选 isolated ？允许，为了尽快恢复/迁出；

           2. 把 avail cmd slots 提前算好放 exclude_cids；

           3. GenerateMigrateCmdsForRemovingChunk 中 migrate_generate_used_cmd_slots 对 src / dst 的判断应该传入 AllocRecoverCap/PerfExtents；

              传入会有点麻烦，可能出现 removing chunk 的时候总是选某个 src / dst cid，但那个 dst cid 可生成的余额不足，还一直选他。但是影响最大也就造成一次 generate 过程中只选 1 个 src cid，用满他的 256 的配额，所以先不修复。

           这部分代码可以写到 recover manager，另外也可以总结出一个 recover 和 alloc 虽然大部分相同，但是存在的细微差别。

           agile recover 和 special recover 回头处理，都是利用到临时副本的，入口是 remove replica

       3. prior migrate

          只有 replica 才会分配临时副本，所以 ec 不会有 agile recover

          临时副本在 perf layer 中一定是 thin 的，临时副本一定分配上

          有很多代码适合 pick 到 55x，但在 56x 中直接被删除了，见 [ZBS-27109](http://jira.smartx.com/browse/ZBS-27109)

          ```
          for (const auto& [cid, info] : healthy_chunks_map) {
          LOG(INFO) << "yiwu cid " << cid << " perf thick allocated "
          << GetAllocatedSpace(info, PK_PERF_THICK) / kExtentSize << " perf thick valid "
          << GetValidSpace(info, PK_PERF_THICK) / kExtentSize << " perf thin allocated "
          << GetAllocatedSpace(info, PK_PERF_THIN) / kExtentSize << " perf thin valid "
          << GetValidSpace(info, PK_PERF_THIN) / kExtentSize << " cap allocated "
          << GetAllocatedSpace(info, PK_CAP) / kExtentSize << " cap valid "
          << GetValidSpace(info, PK_CAP) / kExtentSize;
          }
          
          LOG(INFO) << "yiwu sp_load " << sp_load << " pk " << pk;
          ```

12. 对于仅被 thin volume / snapshot 引用的 capacity pextent，其 provision 将在 gc 扫描时被更新为 thin，随心跳下发给 lsm，如果有 pextent 被 thick volume 引用，那其 provision 将被更新为 thick，随心跳下发给 lsm，[ZBS-15094](http://jira.smartx.com/browse/ZBS-15094)。

13. 根据最新 lsm 设计文档大致了解 lsm2 

14. 补一个同时有多个 removing cid 的单测；

15. refactor migrate for repair topo，从 GenerateMigrateCmdsForRepairTopo 开始改；

        1. 待做 [ZBS-13401](http://jira.smartx.com/browse/ZBS-13401)，让中高负载的容量均衡策略都要保证 prefer local 本地的副本不会被迁移，且如果 prefer local 变了，那么也要让他所在的 chunk 有一个本地副本（有个上限是保留归保留，但如果超过 95%，超过的 部分不考虑 prefer local 一定有对应的副本）
        
            怎么判断是否会超过 95% 呢？
        
            如果 volume 的 prefer local 到新 chunk 后（不论是人为运维还是上层虚拟机被迁移到其他节点），现有的迁移策略能让新位置的 prefer local 有副本吗？
        
            如果不能，在 migrate for rebalance 之后，再有一个 migrate for prefer local，他的目的是保证让 prefer local 有副本，
        
            [ZBS-25949](http://jira.smartx.com/browse/ZBS-25949) 修改后的 migrate for repair topo 能够达到的效果是不会 replace prefer local，在 prefer local 满足 topo rank 不降级的情况下，dst 会优先选 prefer local，貌似能达到这个效果？双活下也可以吗？prefer local 从 prefer zone 迁移到 secondary zone。
        
            migrate for ec repair topo 中对 ec src 的选择过于宽松了，其实还可以选到更好的 src，但是目前不做处理，目前只根据 replace 来选 src

16. MgirateFilter 可以改成 allow, deny 都允许的，如果没要求，就传入 std::nullopt



命令行相关

1. zbs-meta chunk list_pids，显示所有 chunk 的更细粒度的空间显示，把各个 pids 和他们的 space 显示出来，包括有关 reposition cmd 空间大小；

    zbs-meta chunk list 基本上把信息显示出来了，或者后续需要添加的，也应该放在那里。

    我应该针对 reposition 去设计一个 rpc 去显示详细的信息，放在 zbs-meta reposition 里

    zbs-meta reposition list_pids < cid>，如果传入了 cid，那么显示详细的他有哪些 pid，否则只显示一个 pid num，这样也就把第二个功能做了。

    只显示 src / replace / dst / prior_rx 的 pids 以及他们的 space，然后 cap / perf 都展示

    zbs-client-py 侧等待统一添加

2. zbs-meta chunk list_pid < cid>，看指定 chunk 持有哪些不同种类的 pid，除了 ip + port 还要支持直接给定 cid；

    对应 rpc ListPid，这个可以考虑补充下显示 thin/thick 个数，以及 reserve_pids 这样的，

    涉及到跟以往的兼容，这边先不修改

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



1. 让 cli 可以看到 avail cmd slots
2. 把 distributeRecoverCmds 中的生成部分函数抽出来



存储分层模式，可以选择混闪配置或者全闪配置，其中全闪配置至少需要 1 块低速 SSD 作为数据盘，混闪配置至少需要 1 块 HDD 作为数据盘。

存储不分层模式，不设置缓存盘，除了含有系统分区的物理盘，剩余的所有物理盘都作为数据盘使用，只能使用全闪配置。

smtx os 5.1.1 中不论存储是否分层，都要求 2 块容量至少 130 GiB 的 SSD 作为 SMTX OS 系统盘（含元数据分区的缓存盘），为啥要 2 块做软 raid 1？



在恢复或者迁移任务结束时，新加入副本的状态被设置为未知，需要等待下一次心跳周期 LSM 上报副本后才可以确认副本为健康？allocation 的逻辑是马上会被设置为活跃副本，参考 Commit -> PersistExtents -> UpdateMetaContextWhenSuccess -> SetPExtents。



[ZBS-13583](http://jira.smartx.com/browse/ZBS-13583) ，考虑改进如下 Case

\1. extent 是一个空的 child ，自己没有数据，所以如果要读写一定要有 origin 
 \2. 这个 extent 初始状态在 LSM 上也没有存在，所以 parent 迁移的时候就没有数据给他继承把它变成一个实体，这就导致了 child 和 parent 的 location 不一致了。对于完全不一致的场景，我们已经直接把这种空 child 直接 refresh 到和 parent 一样。但是这个只有局部不一致，没有触发这个机制；
 \3. 他们相同的位置刚好坏了盘，盘被没剔除的时候 extent 就一直在 io error 状态，没有被 GC 不可以重建，导致 sync 的时候 2 个副本一个缺少 origin， 一个 IO error，从而导致无法 recover；
 \4. 剔除了异常之后， IO error 因为 extent 被彻底回收了，可以正常重建了，所以恢复就自然结束了。



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
   
   
   
   cmd slot 从 cid -> uint32 到 cmd -> pid set，能够被快速完成的 pid 如 ever exist = false 且 origin = 0 的不算在内
   
   
   
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

做 migrate for repair topo 和 rebalance 时，需要考虑以 chunk 为粒度的遍历，其他的考虑以 pid 为粒度遍历就好。



如果节点上的 replica 发生 cow 的话，direct_prs 会瞬间减小而导致写入因为等待空闲 cache 而阻塞，也需要预先下刷以避免阻塞

pin 中初次写的 lease，怎么传递 prioritized 给 lsm



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

4. 后续写文档可以考虑先介绍所有 sub migrate strategies 中共有的限制条件：

   1. isolated
       1. must not be dst;
       2. should not be src/replace in ec migrate;
       3. should not be src, should be replace in replica migrate;
       
   2. EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，所以以上选到的 mgirate src 不一定就是最后给到 access 时的 src
   
   3. 在介绍副本恢复策略时，可以着重介绍：
   
       1. recover src 和 dst 的选取规则；
   
       2. recover dst 黑名单机制；
   
       3. recover cmd 下发优先级；
   
           recover cmd 之间的优先级通过待修复 pextent 两两比较规则来体现：
   
           1. 若二者活跃副本冗余度不相等，优选最小的，否则执行下一步；
           2. 若二者活跃副本冗余度都不等于 0 并且其中有一个的副本在维护模式节点上，优选不在维护模式节点上的，否则执行下一步；
           3. 若二者有一个是 prior，优选它，否则执行下一步；
           4. 若二者有效副本冗余度不相等，优选最小的，否则执行下一步；
           5. 若二者期望副本冗余度不相等，优选最大的，否则执行下一步；
           6. 若二者活跃副本冗余度都等于 0 并且其中有一个的副本在维护模式节点上，优选不在维护模式节点上的，否则执行下一步；
           7. 选 pid 最小的。
   
           其中（活跃/有效/期望）副本冗余度根据冗余类型不同，计算方式如下：
   
           1. replica： (alive_segment_num / segment_num / expected_segment_num) - 1；
           2. ec shard： (alive_segment_num / segment_num / expected_segment_num) - k，k 是数据块个数；
   
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

1. 为什么 AllocRecoverForAgile 中一定不会有 prior extent？

2. lease owner 不释放的一个原因是 inspector 扫描到 extent generation 不一致而触发的读操作（借此剔除 gen 较低的 extent，再经由 recover 完成数据一致）

3. 为什么 LSM 写 4 KiB cache 需要写 journal，写 256 KiB cache 不需要？

    4k 写为了避免写放大，除了写一个 4k 的真实数据外，还需要写对应的 bitmap （一个 256 KiB 的 block 中有 64 个 4k ）并持久化到 journal。

    否则就需要写 4k 真实数据 + 1020k 实际为 0 的数据，这将引起写放大。

4. LSM2 中，采取 Journaling+Soft Update 的方式保证 crash consistency。默认采用 Journaling，所有数据和元数据的更新都需要通过 Journal 进行保护，然后再写入 BDev。当数据是初次写入时，允许采用 Soft Update 方式，先将数据写入磁盘，再把元数据写入 Journal，避免因数据写入 Journal 引起的写放大。

    初次写的数据如果丢了，元数据没来得及写入 Journal 会有什么后果？

5. 在元数据中，最消耗内存的是 PBlob ，我们必须实现 PBlob 不常驻内存。PBlobEntry 数量跟 PBlob 接近，但单个 PBlobEntry 较小，可全量放内存。对于独占数据，无需额外内存保存 Entry，只需要计算映射值即可；对于共享数据，需要在内存中保存 Entry，因此，快照数量的增加，会增加 LSM 的内存占用。

    为啥独占数据，无需额外内存保存 Entry，只需要计算映射值即可？

6. 为啥 class PBlobCtx 相较于 message PBlobPB 可以节省内存和操作开销？

7. 为啥只有 private pblob 可能触发 promote 的 IO 读，以及 shared pblob 为啥不会？

8. 普通 IO 读不需要修改 Journal，触发 promote 的 IO 读由于涉及到从 data block 到 cache block 的拷贝，所以需要写一条 amend pblob PB 的 Journal，然后才会按照普通 IO 读的流程走下去。

9. 只有 partition 中会使用 checksum，每 8 KiB + 512 B 的布局，每 8 KiB 计算出 CRC32 值，保存在随后的 512 B 里，所以 valid_data_space = 94% * total_data_capacity

    分层之后的 valid_data_space = 94% * total_data_capacity，perf_valid_data_space <= 90% * perf_total_data_capacity，10% 指的是至少为 cap read cache 预留 10% 的 perf space

10. SSD 和 HDD 的 Block（560 开始）

    1. SSD 的 Block 大小使用 16 KiB，而不是 4 KiB。主要出于两点考虑：1. 4 KiB 粒度过小，元数据量较大，且占用的内存过高；2. 4 KiB 粒度过小，对读不友好。单个大 IO 会切分成过多的小 IO。
    2. HDD 的 Block 大小使用 128 KiB，主要出于两点考虑：1. 128 KiB 粒度，占用的内存量可接受；2. HDD 的 IO 延时，大部分来自于寻道，写 IO 的放大对性能影响较小。

11. 256 KiB PBlob 描述

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

整理一下 xen io reroute 中 meta leader 被 kill 的售后处理的流程，zk leader kill 包含 zk session、access session、db cluster 相关的内容。meta in zbs 中关于 db cluster 部分，DBCluster是一个通用的组件，用于各个节点间进行数据的同步。在有数据修改时，DBCluster会首先将journal提交到journal cluster（目前基于zookeeper实现），当提交到journal cluster完成后，数据修改就可以返回了，journal cluster保证修改的持久性，本地的LevelDb会异步的被修改。

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

以一个 functional test 单测为例子展开看 zbs 系统的启动流程。



### reposition 性能测试

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

### perf & metric

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

CoM



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

### access io 并发控制

access 中 app io 写需要拿读屏障，recover io （recover io 读写是一体的）用写屏障，这会使得多个 app io 写之间不会互斥，而 recover io 会跟 app io 写以及其它 recover io 互斥。

1. app io 读不需要拿读屏障，所以 app io 读可以和 app io 写、recover io 并发；

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
   3. 另一个 recover block a 需要阻塞等待前一个 recover 完成才能执行（recover iodepth = 1，只有前一个  block read/write 都执行完才会执行下一个 block 的 read/write）；

具体搜 block_barrier_guard< true/ false> 代码，在 replica_io_handler 和 replica_recover_handler 之间起作用。

另外，access io handler 内部负责非 recover IO 的 3 种类型的 IO 互斥。sink io 会跟 app io 互斥，具体通过代码 LockBlock 和 WaitBlockUnlock 分析 sink io 的读还是写与 app io 的读还是写互斥，LockBlock 和 WaitBlockUnlock 可以看成同一个读写锁，前者像读锁，后者像读锁。

access 按一个 pextent 的 gen 不降序的方式顺序下发 io co 给 lsm（有这么个保序机制），lsm 收到后内部并发执行 io co，在 lsm 内部，如果这些 io co 写的是同一个 pblob，还是需要加锁，如果是不同 pblob，就可以并发执行。

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

transaction 中，判断 thin/thick 的依据，cap 用 thin_provision_ ，perf 用 prioritized_ ，因为一个 transaction 中既可能分配 perf 也可能分配 cap，他们的 thin 和 thick 可能不一样。



分配一个 thick pextent，会马上分配 pid 的 location（此时的第一副本会挑选为分配时集群空间比例最小的节点，其他副本位置再按照局部化原则选择），然后在 transaction 的 Commit 中会让 location 上每个 replica 的 last_report_ms = now_ms，所以此时也马上会有 alive_location = location。

分配一个 thin pextent，直到初次写之前，他的 location 都是空的，所以 alive_location 也为空。



待核实：

分配 pid 跟分配 location 的时期应该不同。

分层之后，创建一个 lextent 时，会马上分配 perf / cap pextent，一定会给 perf 分配 location，如果 cap 的 origin pid 是 0，而且是 thick pextent，也要为 cap 分配 location，否则不分配

### Clone, Snapshot, COW

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



对于 cmp

1. 若想让 l 的优先级更高，那就 return true，
2. 若想让 r 的优先级更高，那就 return false，
3. 其他情况想要保持相对顺序不变，那就 return false
