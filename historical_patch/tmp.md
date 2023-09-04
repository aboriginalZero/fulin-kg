1. 刚快照并对原卷写后，所有的 extent 都还是 prior 的，不先禁掉 migrate 的话，会引发 prior load migrate。

    禁掉之后，空间的值是符合预期的

    快照后写一次要 5s 左右

2. 在调用 GetStoragePoolSpace 时，cid1 给丢了，另外还有情况是他的空间值跟预期的也不符

    cid1 的 status 或 use_state 不符预期，cid1 session expired 了，要在 main thread 中 sleep，要不然 c1 提前退出了

3. 把 gc interval 调的很高，没有 gc-scan，快照并对原卷写后的，属于快照的 extent 竟然是 normal 的

4. 1 明明是最后一个，为啥不是从 1 开始做迁移（cid1 的 status 或 use_state 不符预期，cid1 session expired 了，要在 main thread 中 sleep，要不然 c1 提前退出了）

    ```
    I0830 12:05:10.748733  6940 recover_manager.cc:2089] yiwu cid 5 valid_data_space 37 allocated_data_space 36 valid_cache_space 39 used_cache_space 0 planned_prs 7 allocated_prs 4 downgraded_prs 0
    I0830 12:05:10.748735  6940 recover_manager.cc:2089] yiwu cid 1 valid_data_space 37 allocated_data_space 36 valid_cache_space 39 used_cache_space 0 planned_prs 7 allocated_prs 4 downgraded_prs 0
    I0830 12:05:10.748737  6940 recover_manager.cc:2089] yiwu cid 2 valid_data_space 39 allocated_data_space 15 valid_cache_space 39 used_cache_space 0 planned_prs 7 allocated_prs 0 downgraded_prs 0
    I0830 12:05:10.748739  6940 recover_manager.cc:2089] yiwu cid 3 valid_data_space 37 allocated_data_space 0 valid_cache_space 39 used_cache_space 0 planned_prs 7 allocated_prs 0 downgraded_prs 0
    I0830 12:05:10.748739  6940 recover_manager.cc:2089] yiwu cid 4 valid_data_space 37 allocated_data_space 0 valid_cache_space 39 used_cache_space 0 planned_prs 7 allocated_prs 0 downgraded_prs 0
    I0830 12:05:10.748742  6940 recover_manager.cc:1907] yiwu cid 5 planned_prs 2147273932 allocated_prs 1073741824 diff 1
    I0830 12:05:10.748745  6940 recover_manager.cc:1914] yiwu fine_prio_load_chunks cid 5
    I0830 12:05:10.748746  6940 recover_manager.cc:1907] yiwu cid 1 planned_prs 2147273932 allocated_prs 1073741824 diff 1
    I0830 12:05:10.748747  6940 recover_manager.cc:1914] yiwu fine_prio_load_chunks cid 1
    I0830 12:05:10.748749  6940 recover_manager.cc:1907] yiwu cid 2 planned_prs 2147273932 allocated_prs 0 diff 1
    I0830 12:05:10.748750  6940 recover_manager.cc:1914] yiwu fine_prio_load_chunks cid 2
    I0830 12:05:10.748757  6940 recover_manager.cc:1907] yiwu cid 3 planned_prs 2147273932 allocated_prs 0 diff 1
    I0830 12:05:10.748760  6940 recover_manager.cc:1914] yiwu fine_prio_load_chunks cid 3
    I0830 12:05:10.748764  6940 recover_manager.cc:1907] yiwu cid 4 planned_prs 2147273932 allocated_prs 0 diff 1
    I0830 12:05:10.748767  6940 recover_manager.cc:1914] yiwu fine_prio_load_chunks cid 4
    I0830 12:05:10.748795  6940 recover_manager.cc:907] yiwu ReGenerateMigrateForBalanceInStoragePool before sort
    I0830 12:05:10.748843  6940 recover_manager.cc:910] yiwu cid 5 allocated_data_space 36
    I0830 12:05:10.748847  6940 recover_manager.cc:910] yiwu cid 1 allocated_data_space 36
    I0830 12:05:10.748848  6940 recover_manager.cc:910] yiwu cid 2 allocated_data_space 15
    I0830 12:05:10.748849  6940 recover_manager.cc:910] yiwu cid 3 allocated_data_space 0
    I0830 12:05:10.748857  6940 recover_manager.cc:910] yiwu cid 4 allocated_data_space 0
    I0830 12:05:10.748875  6940 recover_manager.cc:917] yiwu ReGenerateMigrateForBalanceInStoragePool after sort
    I0830 12:05:10.748876  6940 recover_manager.cc:919] yiwu cid 3 allocated_data_space 0
    I0830 12:05:10.748917  6940 recover_manager.cc:919] yiwu cid 4 allocated_data_space 0
    I0830 12:05:10.748919  6940 recover_manager.cc:919] yiwu cid 5 allocated_data_space 36
    I0830 12:05:10.748919  6940 recover_manager.cc:919] yiwu cid 2 allocated_data_space 15
    I0830 12:05:10.748929  6940 recover_manager.cc:919] yiwu cid 1 allocated_data_space 36
    I0830 12:05:10.748989  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 18 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.748996  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 19 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.748998  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 20 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749011  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 21 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749014  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 22 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749018  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 23 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749022  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 24 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749042  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 25 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749053  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 9 src_cid 5 dst_cid 3 replace_cid 5
    I0830 12:05:10.749058  6940 recover_manager.cc:1111] yiwu generate migrate cmd pid 10 src_cid 5 dst_cid 3 replace_cid 5
    ```



3 节点集群，2 副本 prio-volume 持续进行写 IO，对 prio-volume 一直打快照，手动 gc-scan 一下，节点进入高负载后，一直未触发数据迁移。


看着不像是等 gc-scan 才从 prior 变 normal，而是快照+IO后，快照引用的那部分 pextent meta 认为他是 normal，然后通过心跳传给 chunk 的

把 prior over load 关了之后，现象是正常的

3 节点集群，2 副本 prio-volume 持续进行写 IO，对 prio-volume 一直打快照，手动 gc-scan 一下，节点进入高负载后，一直未触发数据迁移。

---

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







---

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

我们的 COW PEXTENT 的触发时机是 GetVExtentLease rpc，如果 chunk 那里 lease 没有 expire，也就不会调用 GetVExtentLease，所以需要通过 revoke 主动让他 expire

COW 是先 revoke，然后打快照，保证了快照后，extent 无法写入的语意，如果不 revoke lease，快照是可写的

清理 cache lease 的方式有 3 种，一个是通过 session id，一个是通过 vtable id，一个是 pid，详见 Meta::ClearCachedLease

ZBS 目前（ <= 5.4.0 ）的快照实现方式分为两部分， Meta 在元数据层更新 COW 标记。即 Volume V [pid = 1, location = [x] ] 制作快照 Snapshot S 时，Snapshot S 会完整的复制一份 V 的 vTable，同时 V 和 S 的 vTable 中所有的 vExtent 都会打上 COW 标记变为 [ pid = 1+, location = [x] ]。当 V 对产生写入请求时，触发一次 COW，V 的 vtable 变化为 [pid = 2 , location = [x] ]，此时 2 的 Parent 为 1 ，并且 Location 需要和 1 完全一致。当 IO 被发往 X 时， X 上的 LSM 会使用 COW 的方式构建 2 -> 1 的关联关系。此时如果是写入请求，LSM 会按需从 1 中获取数据块进行 COW。如果后续有读取请求，则在 2 没有自身独立数据的情况下会访问 1 曾经持有的数据（在 LSM 1 中是以逐级查找 Block bitmap 标记位的方式实现，LSM 2 中以复制 Pblob Table 的方式实现，本质是一样的）。

把 COW 生命周期了解一下







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