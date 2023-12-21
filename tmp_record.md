access recover read 是 extent 粒度，write 是 block 粒度？



1. 卷 08e2d668-098d-4e30-b411-c55d1866353a，dd if=/dev/zero of=c.txt bs=1G seek=100 count=0

    刚创建时 alive location = location，过一会儿就变成 0（确认一下是不是 10 分钟，13:51:35）， ever exist 全程等于 false，

1. 从 esxi 上为虚拟机添加一个磁盘，对应的卷是 90abb21d-4a68-4cdc-be67-671517810896（14:41创建的），一开始都有 alive replica，其中头 2 个和最后 3 个 pextent 有 lease owner（创建 32 GB 就没有这个现象）

    没到 10 min，没有 lease owner 的 pextent 的 alive loc 都清空了，只有这 5 个 pextent lease owner 和 alive loc 还在，并且整体的 ever exist 都等于 false

    32 GiB 95a36f74-e904-468c-9ee2-e59ff9240e9b
    
    96 GiB 5f24a298-7d3a-4970-987d-0b2702d8aa73，Creation Time   2023-12-18 16:28:21.182458105
    
    pid 33642 在 chunk 上都有对应日志了，gen 为 0，所以 ever exist 还是 false？
    
    ```
    /var/log/zbs/zbs-chunkd.INFO:187212:I1218 16:28:21.604626 12869 lsm.cc:5134] [ALLOC EXTENT] status: EXTENT_STATUS_ALLOCATED pid: 33642 epoch: 35468 generation: 0 bucket_id: 874 einode_id: 4213113 thick_provision: true prior: false
    ```
    
    pid 33639 在 chunk 就没有写入的日志
    
1. 通过  zbs-meta volume report_access 90abb21d-4a68-4cdc-be67-671517810896 5 设置 volume 的 prefer local 之后，用 volume show_by_id 查看 volume 的 prefer local 没有变化，extent 有变化，但 extent 的 

1. 为什么 ever exist 不是 true？

    只是由读引入的 sync generation，gen 还是 0，所以 ever exist = false

    meta 会下发大概 （n = disk_size / 10 GiB）条 SETATTR，逐步扩大 size，并非一次直接申请那么大的。
    
    ```
    nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "7e7dcddf-bc3b-4492" sattr3 { size: 10737418240 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_4-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "7e7dcddf-bc3b-4492" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 10737418240 ctime { seconds: 1702910928 nseconds: 671267092 } mtime { seconds: 1702910928 nseconds: 671267092 } atime { seconds: 1702910928 nseconds: 608061606 } } volume_id: "7e7dcddf-bc3b-4492-90b6-60686fd569b1" xid: 1668525677, [TIME]: 39678 us.
    ```
    
    相应的，access 执行 n 次 AccessIOHandler::ReadVExtentInternal，其中，第 1 次会去申请 lease 并执行向各副本所在 chunk sync gen，后面 n - 1 次无需 sync gen，
    
    ```
    /var/log/zbs/zbs-chunkd.INFO:24617:I1218 22:48:50.713533 18732 meta.cc:570] yiwu Meta::GetVExtentLease pid 35609 my_chunk_id_ 2
    /var/log/zbs/zbs-chunkd.INFO:24618:I1218 22:48:50.713559 18732 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 35609
    /var/log/zbs/zbs-chunkd.INFO:24619:I1218 22:48:50.713564 18732 access_io_handler.cc:986] yiwu AccessIOHandler::DoSyncGeneration pid 35609
    /var/log/zbs/zbs-chunkd.INFO:24620:I1218 22:48:50.713573 18732 generation_syncor.cc:138] [SYNC GENERATION START] pid: 35609 loc: [3 1 6 ]
    /var/log/zbs/zbs-chunkd.INFO:24621:I1218 22:48:50.713586 18732 generation_syncor.cc:312] yiwu trigger GetGeneration pid 35609 cid
    /var/log/zbs/zbs-chunkd.INFO:24622:I1218 22:48:50.713605 18732 generation_syncor.cc:312] yiwu trigger GetGeneration pid 35609 cid
    /var/log/zbs/zbs-chunkd.INFO:24623:I1218 22:48:50.713627 18732 generation_syncor.cc:312] yiwu trigger GetGeneration pid 35609 cid
    /var/log/zbs/zbs-chunkd.INFO:24626:I1218 22:48:50.720468 18732 generation_syncor.cc:237] [SYNC GENERATION SUCCESS]  pid: 35609 loc: [3 1 6 ] gen: 0 io_hard: 0
    /var/log/zbs/zbs-chunkd.INFO:24627:I1218 22:48:50.721127 18732 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 35609
    ```
    
    
    
    
    
1. 为什么那 5 个 pid 10 分钟过后 alive loc 还在？

    因为 lsm 侧真的给这 5 个 pextent 分配空间了，经由心跳上报给 meta

1. 那 5 个 pid 的 lease owner 为什么 1 小时之后还在？1 个小时后释放了。

    先下发的 volume 级别的 revoke lease，5 个 access handler 主动清了自己本地的 lease 缓存，然后 meta 为那 5 个要被读的 pid 分配 read lease，

    观察发现他释放了 .....
    
    133.243，cid = 2
    
    ```c++
    210296:I1218 17:57:32.751230  3507 access_handler.cc:885] [REVOKE LEASE]: vtable: 453d9c62-16da-4649-bdef-18edd4d6c8c4
    210297:I1218 17:57:32.782191  3507 nfs3_server.cc:1745] [VAAI RESVSPACE]: inode: [HANDLE]: type: 1, inode_id: 5280147402817027397, volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4"
    
    // 连续相同的 5 条
    210354:I1218 17:57:33.023092  3507 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 5280147402817027397, volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4"
    
    // 有一个 Update volume config
    I1218 17:57:33.740621  3507 zbs_client.cc:1569] Update volume config : name: "453d9c62-16da-4649"
    
    // 跟了 5 条 pid sync 成功的日志
    I1218 17:57:33.745319  3507 generation_syncor.cc:138] [SYNC GENERATION START] pid: 33772 loc: [3 1 6 ]
    I1218 17:57:33.746989  3507 generation_syncor.cc:237] [SYNC GENERATION SUCCESS]  pid: 33772 loc: [3 1 6 ] gen: 0 io_hard: 0
    
    // 连续相同的 4 条
    210533:I1218 17:57:33.854576  3507 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 5280147402817027397, volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4"
    
    ```
    
    133.241，cid = 1
    
    revoke lease 
    
    ```c++
    366674:I1218 17:57:32.754055  6522 access_handler.cc:885] [REVOKE LEASE]: vtable: 453d9c62-16da-4649-bdef-18edd4d6c8c4
    ```
    
    133.242，cid = 3
    
    pextent 级别的日志
    
    ```c++
    [root@node2 18:07:48 ~]$ grep -wn "34155" /var/log/zbs/zbs-metad.INFO
    
    I1218 17:57:33.755331 20588 meta_rpc_server.cc:754] yiwu MetaRpcServer::GetVExtentLease
    /var/log/zbs/zbs-metad.INFO:7143:I1218 17:57:33.755450 20588 meta_rpc_server.cc:952] yiwu MetaRpcServer::GetLeaseForRead pid 34155
    /var/log/zbs/zbs-metad.INFO:7144:I1218 17:57:33.755466 20588 meta_rpc_server.cc:1204] yiwu MetaRpcServer::DoGetLease pid 34155 preferred session 56a6cdef-de32-4646-bd37-6b722db5635d local_cid 2
    /var/log/zbs/zbs-metad.INFO:7145:I1218 17:57:33.755486 20588 access_manager.cc:586] yiwu AccessManager::AllocOwner pid 34155 cid 2 cid 0
    /var/log/zbs/zbs-metad.INFO:7146:I1218 17:57:33.755506 20588 access_manager.cc:1857] yiwu GenerateLease pid 34155
    /var/log/zbs/zbs-metad.INFO:8052:I1218 18:58:46.704962 20588 access_manager.cc:1935] Release owner : pid: 34155 session: 56a6cdef-de32-4646-bd37-6b722db5635d
      
    [root@node2 18:40:49 sbin]$  grep -wn "34155" /var/log/zbs/zbs-chunkd.*
    /var/log/zbs/zbs-chunkd.log.20231218-175700.21577:26068:I1218 17:57:33.756036 21584 local_io_handler.cc:272] yiwu HandleGetGeneration pid 34155
    /var/log/zbs/zbs-chunkd.log.20231218-175700.21577:26069:I1218 17:57:33.756196 21585 lsm.cc:1501] yiwu VerifyAndGetGeneration pid 34155
    /var/log/zbs/zbs-chunkd.log.20231218-175700.21577:26070:I1218 17:57:33.756245 21585 lsm.cc:5135] [ALLOC EXTENT] status: EXTENT_STATUS_ALLOCATED pid: 34155 epoch: 35993 generation: 0 bucket_id: 1387 einode_id: 4213157 thick_provision: true prior: false
    ```
    
    volume 级别的日志
    
    ```c++
    [root@node2 18:41:03 sbin]$  grep -n "453d9c62-16da-4649" /var/log/zbs/zbs-chunkd.*
    /var/log/zbs/zbs-chunkd.log.20231218-175700.21577:24815:I1218 17:57:32.751410 21584 access_handler.cc:885] [REVOKE LEASE]: vtable: 453d9c62-16da-4649-bdef-18edd4d6c8c4
    
    // CREATE INODE 之后，会有连续 10 条的 SETATTR 的日志，每 10G 发一条，他是一步一步扩大 file size 的吗？
    7030:I1218 17:57:32.526930 20588 nfs_server.cc:337] [CREATE INODE]: [REQUEST]: parent_id: "60ca9357-9c5f-4818" name: "zyx1-test-vm03_2-flat.vmdk" type: FILE how: UNCHECKED sattr3 { mode: 384 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } } xid: 1668431687, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "453d9c62-16da-4649" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 0 ctime { seconds: 1702893452 nseconds: 519286486 } mtime { seconds: 1702893452 nseconds: 519286486 } atime { seconds: 1702893452 nseconds: 519286486 } } volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4" xid: 1668431687, [TIME]: 7804 us.
    7031:I1218 17:57:32.550280 20588 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "453d9c62-16da-4649" sattr3 { mode: 384 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "453d9c62-16da-4649" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 0 ctime { seconds: 1702893452 nseconds: 546426318 } mtime { seconds: 1702893452 nseconds: 519286486 } atime { seconds: 1702893452 nseconds: 519286486 } } volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4" xid: 1668431687, [TIME]: 3918 us.
    
    // 紧接着会有 revoke cmd，共有 5 条，应该是对应 5 个 chunk，3 + 3 的双活，但是有一个节点没起来
    7044:I1218 17:57:32.750852 20610 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4" } }
    
    // 创建 thick volume 一定会调用这个？
    7049:I1218 17:57:32.780071 20588 meta_rpc_server.cc:4154] [RESERVE VOLUME SPACE]: [REQUEST]: volume_id: "453d9c62-16da-4649-bdef-18edd4d6c8c4", [RESPONSE]: ST:OK, name: "453d9c62-16da-4649" size: 103079215104 created_time { seconds: 1702893452 nseconds: 519286486 } id: "453d9c62-16da-4649-bdef-18edd4d6c8c4" parent_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" replica_num: 3 thin_provision: false iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144, [TIME]: 29332 us.
    ```
    
    loc = 3 1 6，lease owner = 2
    
    Notify 这 5 条的时候，SessionState 是 PROCESS_KEEPALIVE，所以会立即下发，5 个 chunk 的 access handler 都收到了，他们会清空自己本地缓存的 lease
    
1. 创建不同大小的虚拟磁盘中有 lease owner 的 pextent 不同？

    创建小一点的，会调用 GetLeaseForWrite，96, 100, 128 G 会调用 GetLeaseForRead

    64, 90 GiB 会调用 GetLeaseForWrite，按 256 KiB 大小写，前一半的 pextent 有 lease owner，后一半没有，稳定复现

    16，32 GiB，所有都会有 lease owner，没有他的读或写日志

    256 GiB，前一些是 lease owner，走的是 Write

    但不论是哪种情况，ever exist = false，

    14b03eb7-4e79-4043-8cbf-c2a8f1672c6d，90 GiB，都有 alive replica

    69a5e412-95b9-414a-995a-163323dab337，128 GiB，只有 5 个有，

    5e27fa23-39e2-4db7-9c79-5406f104d925，16 GiB，都有 alive replica

    23869133-291b-45c8-832e-d633f364eaab，32 GiB，都有 alive replica

    7194d43c-d46d-401e-bf57-6f6f4976bdd5，64 GiB，都有 alive replica

    6d8d4150-cfa3-4237-9f7f-42ab80e7ef08，256 GiB，都有 alive replica

1. 主动设置卷的 prefer local 后，卷的属性上的 prefer local 为啥没更新？

1. 单测还需要补上 perf，prior，cap，空间不均，

    这些只用写一个单测，而不需要一组 case

    要模拟有 lease owner 可以参考 RepairTopoWithOwner



避免某些 pid / volume migrate 迟迟不完成造成集群整体的 migrate 阻塞

1. migrate for chunk removing，没必要添加，因为只要有 chunk removing，就会一直执行这种 migrate，其他 mgirate 没有机会执行，直到他的数据迁移完了，他的 state / status 状态变更后，才会跳过这个 migrate；

   > 这个情况，不对 replace cid 做限制，但对 src 和 dst 做限制

2. migrate for even volume，需要添加，如果一个 even volume 占用了所有的 quota 且一直没法完成，其他 even volume 没有机会做 migrate 的情况，甚至下面的几种 migrate 也一直没有机会；

   需要有一个 chunk 的上限，避免一个 chunk 一直无法完成，一直产生给他的 migrate cmd 把 quota 占满；

   1. migrate for even volume repair topo，不需要有，但调用了 GetSrcCidForReplicaMigration() 默认就会有，看起来还是需要在 GetSrcCidForReplicaMigration() 里加个是否启用的 flag，因为后续 for localization 也会用到；
   2. migrate for even volume rebalance，需要有 chunk 上限的限制；

3. migrate for over load prior extent，需要添加，因为如果一个节点负载比较高的节点消耗了所有 quota，且一直难以完成，那么其他节点都没有机会做 migrate；

4. migrate for localization，不需要添加，因为按 pid 为粒度扫描，当前几个无法完成不会影响到其他 pid migrate cmd 下发；

5. migrate for repair topo，不需要添加，因为按 pid 为粒度扫描，当前几个无法完成不会影响到其他 pid migrate cmd 下发；

6. migrate for rebalance，需要添加，因为如果一个负载比较高的节点消耗了所有 quota，且一直难以完成，那么其他节点都没有机会做 migrate；

结论：给所有 migrate 都添加 generate_cmds_per_chunk_limit_ ，但他的值不支持自调节，只是 generate_cmds_per_round_limit_ / 2，引入 generate_cmds_per_chunk_limit_ 的目的是避免一个节点用完所有的 generate quota 而其他节点 / migrate 没有机会生成。

* 以 pid 为粒度扫描的，用 next_xxx_scan_pid_map 来避免让某些 pid migrate 影响到整体，next_xxx_scan_pid_map 保留现状，他其实不需要 generate_cmds_per_chunk_limit_，但为了便于理解，还是加上吧；
* 以 chunk + pid 为粒度扫描的，用 randint + generate_cmds_per_chunk_limit_ 来避免某些 pid 阻塞导致其他 chunk / pid 没机会 migrate；



1. 为什么 AllocRecoverForAgile 中一定不会有 prior extent？

2. 在 HasSpaceForCow() 为什么用的是 total_data_capacity 而不是 valid_data_space ？

3. 为什么对节点进入超高负载的判断用的是 used_data_space 而不是 allocated_data_space？

    改进迁移策略，在节点上待回收数据较多时（已经使用的数据空间占比超过 95%），如果集群没有进入极高负载状态（整体空间分配比例达到 90%），不向该节点迁移数据以保证回收顺利进行。

    lsm1 回收空间的速率非常慢，所以如果删除一个 extent，存在 chunk 的 provisioned space 会减少（分配数据空间比例在减小），但 used space 可能仍然很高的情况，如果这时集群上的其他节点向它迁移数据，会进一步降低回收速率。

    在 Migrate 时进行检查，如果集群整体尚有可用空间时比如整体 provisioned 比例在 90% 以下，不向 Used Space 比例大于 95% 的节点迁移数据，即便 Provsioned 比较低。



做 migrate for repair topo 和 rebalance 是，需要考虑

xx 1. 不开分层的 replica ，2. 开分层后的 cap replica，3. 开分层后的 perf replica，4. 开分层后的 cap ec，他们的单测不适合合起来，因为如果之后 cap / perf 策略不同的话，还是得拆出来。



1. 待做 [ZBS-13401](http://jira.smartx.com/browse/ZBS-13401)，让中高负载的容量均衡策略都要保证 prefer local 本地的副本不会被迁移，且如果 prefer local 变了，那么也要让他所在的 chunk 有一个本地副本（有个上限是保留归保留，但如果超过 95%，超过的 部分不考虑 prefer local 一定有对应的副本）

   怎么判断是否会超过 95% 呢？

2. rebalance 时能 recover jiewei 发现的问题，机架 A 有节点 1 2 3 4，机架 B 有节点 5 6 7 ，normal extent 容量均衡会去算一个 avg_load，B 上的节点负载都大于 avg_load，A 上的都小于 avg_load，5 容量不够了，只能往 1 2 3 4 迁，但是他们都在 A 上，由于 topo 降级所以都没法迁。改进使得 5 可以向 6/7 上迁。

   even volume 中的做法应该能实现这个效果，参考即可。

3. 后续可以改进容量均衡迁移中 replace chunk 和 dst chunk 1 1 配对，可以改成尽可能让多个 src_cid 参与进来，除非所有 under chunk 都不行，才退出循环。（其实下一轮就会用上的）



1. 把 migrate for repair topo 和 rebalance 替换之后再做 generate_cmd_per_chunk_limit 的统一修改

2. ever exist = false 且 origin_pid = 0 的 pextent 在下发 reposition cmd 时才可以不用受 avail cmd slots 限制

3. ec migrate 目前的做法是 src_cid 一定等于 replace_cid，所以需要避免 ec migrate 的 replace cid 选 not healthy status/state 和 isolated 的 cid，等 ec access 支持用恢复的方式来做迁移，这个条件或许才能放开；

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

4. zbs-meta recover < volume_id> 想让这个 volume 优先被 recover；

    当有多个 volume 需要 recover，耗时太久时，可以优先 recover 指定卷上的 pextent



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



[ZBS-26042](http://jira.smartx.com/browse/ZBS-26042) 还缺一个 even volume 的 ut 验证 [ZBS-25847](http://jira.smartx.com/browse/ZBS-25847)

针对 ec 的 best topo distance 的计算，斯坦纳树问题，对应的 DP 做法 Dreyfus-Wagner 算法。yutian 说最新做法是从拓扑学的角度出发的，现有代码已经是最优解而非近似计算了



1. recover 支持分批扫描

   比较纠结的是，目前 recover_manager 中 scan_extents_per_round_limit 这个参数只限制了 recover / migrate for localization / migrate for repair topo 一次扫描的 pextent 数量，而对其他类型的 migrate （如 migrate for even volume/ prior extent / rebalance）并没有做限制，这让他的语义并不完整。

   需要在其他 migrate 中加上这个限制

6. reposition params 支持分层

    分层之后，perf speed limit 与 cap speed limit 必然不同，但是要怎么限制呢？

    分层之后，盘就是分开用了，perf 层用 ssd，cap 层用 hdd

    补充对应的单测

    access_io_stats.h 中 access_io_perf_t 和 access_io_stats_t 的区别？前者在 zbs 内部用，后者好像是给 zbs 外部用的

    变量  to_submit_iocbs_ 看起来不是线程安全的，还是说他不需要保证线程安全，access_handler 和 local_io_handler 和 temporary_replica_io_handler 等都是在一个线程？

3. 重构 recover manager 时，需要考虑 prior extent 不需要在低负载下支持局部化，master 分支上目前还是支持，但 5.5.x 已经不支持了



单测里面

```cpp
// 验证双活生效
RecoverManager* recover_manager = GetMetaContext().recover_manager;

// 验证 GFLAGS 更改生效
RecoverManager recover_manager(&(GetMetaContext()));

改用 CO_TEST_F 开头好像就可以了
```



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

    可能需要把 std::list < RecoverCmd> 改成 hashmap

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



改 Prefer Local / TopoAware / Localized 三个比较器名字，[ZBS-25802](

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
2. chunk removing 要求尽快迁移，所以不考虑 topo 安全，跟 recover 用的同一个选择策略，



1. 现有代码在 EnqueueCmd 的时候会将 migrate src_cid 设置成 lease owner，但如果 src_cid 跟 dst_cid 跨 zone，或者是 failslow，（他可能没有 cmd quota 了，不过这个条件在生成时是硬性规定的，派发时倒不用）这么选还是好事吗？
2. 目前 migrate 中的逻辑是每次获取一个 pid 的 entry 都要通过 GetPhysicalExtentTableEntry 调用一次锁，但在 prior  extent 的迁移中，可以批量获取 diff_pids 中所有的 pentry，因此可以相应做优化。
3. GenerateRecoverCmds 里面的 src 也可以有选择策略的，目前的写法太乱了。



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

   目前遇到的高负载下不迁移：要么 topo 降级了，要么 lease owner 没释放，要么是双活只能在单 zone 内迁移





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
