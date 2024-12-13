### 新创建数据块的 lease owner / prefer local 不在本地

同一个 target 上的其他 lun 在 chunk 重启后有 IO 导致的 iscsi 接入点变化，进而影响到这个 lun 上数据块的 lease owner 和 prefer local 位置。

1. iscsi initiator 在 cid 4 上，一开始 cid 4 正常，所以选的是 cid4 作为 iscsi 接入点，meta 上创建了一个 key = cid 4, value = initiator id + target id 的 iscsi session；
2. cid4 stop chunk 后，meta 会清理 cid4 的 iscsi session；
3. redirector 要为新来的访问这个 target 的 initiator 创建 iscsi connection（同一个 target 中还有其他 lun 在发 IO），为此需要先找 meta 要接入点（portal = data_ip + port 3261）；
4. cid 4 chunk 没了，meta 分配的接入点是在非本地的 iscsi session 中任选一个，若 local_access_ip 不变（10.0.131.196）且剩下的 iscsi session 个数不变（size = 3），那么就会一直选到同一个，这里就是 cid3
5. iscsi initiator 以 cid 3 作接入点访问 target，之后的 lease owner 以及 extent 的 prefer local 也就都是 3 了；
6. 后续即使 cid 4 恢复了，但是已有的 cid 3 的 iscsi connection 是稳定的，没有马上切换到 cid 4，所以 lease owner 和 prefer local 还是 3；
7. 在 hci 模式下，meta 每 60s 检查一次如果有存活超过 3 min 的非本地的 iscsi connection，meta 会让 chunk 主动 drop 它们，所以可以看到放置一段时间后，这个 initiator 又会用 cid4 作为接入点。



持续打印，观察 iscsi 接入点的变化

```
while true; do echo " " >> /tmp/1.log && date >> /tmp/1.log && zbs-meta session list_iscsi_conn >> /tmp/1.log; sleep 1; done;
```



iscsi redirector 没有 leader 的概念

```
I1211 18:04:34.719784 50497 iscsi_server.cc:1646] [CREATE ISCSI LUN]: [REQUEST]: lun_path { pool_path { pool_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" } lun_id: 114 lun_name: "2a207fb4-0220-41de-9004-988f53eab8a0" } size: 10737418240 single_access:false stripe_num: 4 stripe_size: 262144 prioritized: false, [RESPONSE]: ST:OK, lun_id: 114 pool_id: "e8a4315e-241d-4283-98ab-1e6198744a15" volume_id: "f74e78f0-a7d6-4de6-9b19-b9af2687f078" created_time { seconds: 1733911474 nseconds: 714625070 } size: 10737418240 description: "" replica_num: 2 thin_provision: true throttling { } name: "2a207fb4-0220-41de-9004-988f53eab8a0" naa: "33d4d6d7ad0f87e47" single_access: false stripe_num: 4 stripe_size: 262144 pr_info { gen: 0 } encrypt_method: ENCRYPT_PLAIN_TEXT, [TIME]: 6936 us.
I1211 18:04:34.996024 50497 iscsi_server.cc:2246] [UPDATE ISCSI LUN]: [REQUEST]: lun_path { pool_path { pool_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" } lun_id: 114 } new_allowed_initiators: "" single_access: true, [RESPONSE]: ST:OK, lun_id: 114 pool_id: "e8a4315e-241d-4283-98ab-1e6198744a15" volume_id: "f74e78f0-a7d6-4de6-9b19-b9af2687f078" created_time { seconds: 1733911474 nseconds: 714625070 } size: 10737418240 description: "" replica_num: 2 thin_provision: true throttling { } name: "2a207fb4-0220-41de-9004-988f53eab8a0" naa: "33d4d6d7ad0f87e47" allowed_initiators: "" single_access: true stripe_num: 4 stripe_size: 262144 pr_info { gen: 0 } encrypt_method: ENCRYPT_PLAIN_TEXT, [TIME]: 3223 us.
I1211 18:04:35.003139 50497 iscsi_server.cc:3513] [ADD ISCSI LUN ALLOWED INITIATORS]: [REQUEST]: lun_path { pool_path { pool_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" } lun_id: 114 } new_allowed_initiators: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0", [RESPONSE]: ST:OK, lun_id: 114 pool_id: "e8a4315e-241d-4283-98ab-1e6198744a15" volume_id: "f74e78f0-a7d6-4de6-9b19-b9af2687f078" created_time { seconds: 1733911474 nseconds: 714625070 } size: 10737418240 description: "" replica_num: 2 thin_provision: true throttling { } name: "2a207fb4-0220-41de-9004-988f53eab8a0" naa: "33d4d6d7ad0f87e47" allowed_initiators: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0" single_access: true stripe_num: 4 stripe_size: 262144 pr_info { gen: 0 } encrypt_method: ENCRYPT_PLAIN_TEXT, [TIME]: 5111 us.
I1211 18:04:35.008616 50497 iscsi_server.cc:3854] yiwu request: initiator: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0" target_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" initiator_ip: "127.0.0.1" conn_server_ip: "127.0.0.1" local_access_ip: "10.0.131.196", use_sec: 0, external_use: 0, first session cid: 3, conn_server_ip: 127.0.0.1
I1211 18:04:35.008667 50497 iscsi_server.cc:3862] yiwu request: initiator: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0" target_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" initiator_ip: "127.0.0.1" conn_server_ip: "127.0.0.1" local_access_ip: "10.0.131.196", already login portal: 10.0.131.199:3261
```

分配新的接入点

```
I1211 19:11:29.361781 26497 meta_rpc_server.cc:2140] [ALLOC PT_PERF PEXTENT] volume: name: "f8e5482a-10d8-46b7-9176-c4c5a402e88e" size: 10737418240 created_time { seconds: 1733915433 nseconds: 608325830 } id: "f8e5482a-10d8-46b7-9176-c4c5a402e88e" parent_id: "e8a4315e-241d-4283-98ab-1e6198744a15" replica_num: 2 thin_provision: true encrypt_method: ENCRYPT_PLAIN_TEXT iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144 prefer_cid: 0 resiliency_type: RT_REPLICA,  lid: 17156 lentry: {epoch: 17156, perf_pid: 34789, perf_epoch: 34789, cap_pid: 34749, cap_epoch: 34749, prioritized: 0, garbage: 0, valid: 1, staging: 0 }, perf_pentry: {loc: "[ 0:4 1:3 ]", alive loc: "[ 0:4 1:3 ]", epoch: 34789, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, even: 0, pt: "PT_PERF", rt: "RT_REPLICA", ec_param: "None", sinkable: 1, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0 }, cap_pentry: {loc: "[ ]", alive loc: "[ ]", epoch: 34749, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 0, even: 0, pt: "PT_CAP", rt: "RT_REPLICA", ec_param: "None", sinkable: 0, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0 }
I1211 19:11:32.296346 26578 session_master.cc:275] [LEAVE SESSION]: [REQUEST]: uuid: "68702494-9dc1-4405-a291-62a9a906a747" epoch: 1997833885, [RESPONSE]: ST:OK, , [TIME]: 10 us.
I1211 19:11:32.296455 26578 session.cc:205] [SESSION LEFT]: session_epoch { uuid: "68702494-9dc1-4405-a291-62a9a906a747" epoch: 1997833885 } group: "access" lease_expire_ns: 0
I1211 19:11:32.296514 26578 access_manager.cc:2678] [ACCESS SESSION EXPIRED]: session_epoch { uuid: "68702494-9dc1-4405-a291-62a9a906a747" epoch: 1997833885 } group: "access" lease_expire_ns: 0 uuid: "68702494-9dc1-4405-a291-62a9a906a747" ip: "10.0.131.196" num_ip: 3296919562 port: 10201 cid: 4 secondary_data_ip: "10.1.131.196" zone: "default" scvm_mode_host_data_ip: "" alive_sec: 412 machine_uuid: "133e6844-b231-11ef-a63f-52540062a3a8"
I1211 19:11:32.296666 26578 chunk_table.cc:1585] Chunk 4 status change: CHUNK_STATUS_CONNECTED_HEALTHY --> CHUNK_STATUS_SESSION_EXPIRED

I1211 19:11:32.323140 26497 iscsi_server.cc:3861] yiwu request: initiator: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0" target_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" initiator_ip: "127.0.0.1" conn_server_ip: "127.0.0.1" local_access_ip: "10.0.131.196", use_sec: 0, external_use: 0, first session cid: 3, conn_server_ip: 127.0.0.1
I1211 19:11:32.323194 26497 iscsi_server.cc:3891] yiwu PickAccessForNormalTarget
I1211 19:11:32.323205 26497 iscsi_server.cc:3940] yiwu cid: 3, session ip: 10.0.131.199, local_access_ip: 10.0.131.196
I1211 19:11:32.323215 26497 iscsi_server.cc:3940] yiwu cid: 2, session ip: 10.0.131.198, local_access_ip: 10.0.131.196
I1211 19:11:32.323225 26497 iscsi_server.cc:3940] yiwu cid: 1, session ip: 10.0.131.197, local_access_ip: 10.0.131.196
I1211 19:11:32.323231 26497 iscsi_server.cc:3952] yiwu PickAccessForNormalTarget random
I1211 19:11:32.323251 26497 iscsi_server.cc:4038] yiwu GetPortalFromSession, session.info.ip: 10.0.131.199, val: 10.0.131.199:3261
I1211 19:11:32.323559 26497 iscsi_server.cc:3861] yiwu request: initiator: "iqn.2013-11.org.smartx:c19cb419-35b0-4bfc-96a2-60b45f7ce90e.133e6844-b231-11ef-a63f-52540062a3a8.0" target_name: "zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613" initiator_ip: "127.0.0.1
" conn_server_ip: "127.0.0.1" local_access_ip: "10.0.131.196", use_sec: 0, external_use: 0, first session cid: 3, conn_server_ip: 127.0.0.1
I1211 19:11:32.323596 26497 iscsi_server.cc:3891] yiwu PickAccessForNormalTarget
I1211 19:11:32.323611 26497 iscsi_server.cc:3940] yiwu cid: 3, session ip: 10.0.131.199, local_access_ip: 10.0.131.196
I1211 19:11:32.323621 26497 iscsi_server.cc:3940] yiwu cid: 2, session ip: 10.0.131.198, local_access_ip: 10.0.131.196
I1211 19:11:32.323628 26497 iscsi_server.cc:3940] yiwu cid: 1, session ip: 10.0.131.197, local_access_ip: 10.0.131.196
I1211 19:11:32.323637 26497 iscsi_server.cc:3952] yiwu PickAccessForNormalTarget random
I1211 19:11:32.323647 26497 iscsi_server.cc:4038] yiwu GetPortalFromSession, session.info.ip: 10.0.131.199, val: 10.0.131.199:3261
```



chunk 日志

```
W1211 19:11:32.294697 15977 conn.cc:1418] [ISCSI] no need to enable epollin for conn id: 4294967317 in_flight_pdu_count: 0
W1211 19:11:32.294736 15977 conn.cc:1418] [ISCSI] no need to enable epollin for conn id: 4294967318 in_flight_pdu_count: 0
I1211 19:11:32.294798 15977 tgt_node.cc:358] destruct target iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-5e0022cd-b252-4ee1-9f5d-f2e33d14e36a safely
I1211 19:11:32.294847 15977 tgt_node.cc:358] destruct target iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-0913fc3f-a1a8-4f5f-bf89-3f631a44bd3d safely
I1211 19:11:32.294862 15977 tgt_node.cc:358] destruct target iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-7ad28c08-ec05-487b-850a-9e428733054f safely
I1211 19:11:32.294871 15977 tgt_node.cc:358] destruct target iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613 safely
I1211 19:11:32.294932 15977 tgt_node.cc:358] destruct target iqn.2016-02.com.smartx:system:043eb525-8ee4-4b6f-bf5d-3ac8a4418ae4 safely
I1211 19:11:32.294951 15977 iscsi_mega_server.cc:38] [ISCSI SERVER STOP]
```

关注的 target

```
ID                      e8a4315e-241d-4283-98ab-1e6198744a15
Name                    zbs-iscsi-datastore-5177d6ba-3d95-476e-826f-8b6f8dca0613
```

### 写一个超过单点容量的卷触发限流

meta 提供 2 种获取 lease 的 rpc

1. GetVExtentLease
2. GetLease



3 节点 cid 1 3 4 集群，lease owner = prefer local = 4，各节点 perf valid = 400 GiB，创建一个 500 GiB 的 2-replica thin volume，先执行 256k 全盘顺序写，紧接着跟上 4k 随机写，复现节点出现 enable flow ctrl 的场景。

perf loc 变化如下：

| step | cid 4 ratio | cid 3 ratio | cid 1 ratio | perf loc       | prefer  cid load | note                                                         |
| ---- | ----------- | ----------- | ----------- | -------------- | ---------------- | ------------------------------------------------------------ |
| 1    | [0, 0.5]    | [0, 0.5]    |             | [4, 3]         | 低               | 遵循局部化分配                                               |
| 2    | [0.5, 0.6]  |             | [0, 0.1]    | [4, 1]         | 中               | 放弃局部化，遵循本地优先 + 拓扑安全 + 剩余容量多             |
| 3    |             | [0.5, 0.6]  | [0.1, 0.2]  | [3, 1]         | 高               | 放弃本地优先，遵循拓扑安全 + 剩余容量多，此时有 replace = 3，dst = 1 的迁移 |
| 4    |             |             |             | [1, 3], [1, 4] | 高               | cid 1 在负载跟 cid 3 和 4 相同前，总会被分配，perf loc 在 [1, 3]、[1, 4] 交替 |

I1119 16:42:45.433437 第一次 ALLOC PT_PERF PEXTENT，此时各节点负载基本为 0

```
I1119 16:42:45.433437 55463 meta_rpc_server.cc:1566] [ALLOC PT_PERF PEXTENT] volume: name: "c0f40f0a-1a3f-4831-a87c-8448bb007396" size: 536870912000 created_time { seconds: 1732005758 nseconds: 707335736 } id: "c0f40f0a-1a3f-4831-a87c-8448bb007396" parent_id: "4b7928a6-9d98-4754-8e46-c90172c6da49" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 8 stripe_size: 262144 prefer_cid: 0 resiliency_type: RT_REPLICA,  lid: 5 lentry: epoch: 41509, perf_pid: 93673, perf_epoch: 93673, cap_pid: 91673, cap_epoch: 91673, prioritized: 0, garbage: 0, valid: 1, staging: 0  perf_pentry: replica: "[ 0:4 1:3 ]", epoch: 93673, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_PERF", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 1, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0  cap_pentry: replica: "[ ]", epoch: 91673, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 0, type: "PT_CAP", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 0, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0

I1119 16:42:45.433508 55463 meta_rpc_server.cc:1589][YIWU1 CHUNK INFO]
cid: 1, perf thin allocated: 1024 KiB, ratio: 0.000002, thin info: [perf_thin_used_data_space: (1024), perf_new_thin_pids: (0, 0), perf_rx_pids: (0, 0) KiB]
cid: 3, perf thin allocated: 263168 KiB, ratio: 0.000627, thin info: [perf_thin_used_data_space: (1024), perf_new_thin_pids: (1, 262144), perf_rx_pids: (0, 0) KiB]
cid: 4, perf thin allocated: 263168 KiB, ratio: 0.000627, thin info: [perf_thin_used_data_space: (1024), perf_new_thin_pids: (1, 262144), perf_rx_pids: (0, 0) KiB]
```

I1119 16:49:46.225734 第一次 ALLOC PT_CAP PEXTENT FOR SINK，此时集群平均负载过了 0.5

```
I1119 16:51:13.221374 55463 meta_rpc_server.cc:1675] [ALLOC PT_CAP PEXTENT FOR SINK]  lid: 1325 lentry: epoch: 42829, perf_pid: 94995, perf_epoch: 94995, cap_pid: 92993, cap_epoch: 92993, prioritized: 0, garbage: 0, valid: 1, staging: 1  perf_pentry: replica: "[ 0:1 1:3 ]", epoch: 94995, generation: 1, origin_pid: 0, origin_epoch: 0, ever_exist: 1, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_PERF", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 1, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0  cap_pentry: replica: "[ 0:4 1:3 ]", epoch: 92993, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_CAP", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 0, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0

I1119 16:51:13.221410 55463 meta_rpc_server.cc:1696][YIWU3 CHUNK INFO]
cid: 1, perf thin allocated: 181066496 KiB, ratio: 0.431697, thin info: [perf_thin_used_data_space: (95607552), perf_new_thin_pids: (8, 2097152), perf_rx_pids: (318, 83361792) KiB]
cid: 3, perf thin allocated: 259230208 KiB, ratio: 0.618054, thin info: [perf_thin_used_data_space: (258181632), perf_new_thin_pids: (4, 1048576), perf_rx_pids: (0, 0) KiB]
cid: 4, perf thin allocated: 259239680 KiB, ratio: 0.618077, thin info: [perf_thin_used_data_space: (258191104), perf_new_thin_pids: (4, 1048576), perf_rx_pids: (0, 0) KiB]
```

I1119 16:49:46.317569 55463 开始下沉后的第一次 ALLOC PT_PERF PEXTENT，此时只分配了 perf loc

```
I1119 16:49:46.317569 55463 meta_rpc_server.cc:1566] [ALLOC PT_PERF PEXTENT] volume: name: "c0f40f0a-1a3f-4831-a87c-8448bb007396" size: 536870912000 created_time { seconds: 1732005758 nseconds: 707335736 } id: "c0f40f0a-1a3f-4831-a87c-8448bb007396" parent_id: "4b7928a6-9d98-4754-8e46-c90172c6da49" replica_num: 2 thin_provision: true iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 8 stripe_size: 262144 prefer_cid: 0 resiliency_type: RT_REPLICA,  lid: 1173 lentry: epoch: 42677, perf_pid: 94843, perf_epoch: 94843, cap_pid: 92841, cap_epoch: 92841, prioritized: 0, garbage: 0, valid: 1, staging: 0  perf_pentry: replica: "[ 0:1 1:3 ]", epoch: 94843, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_PERF", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 1, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0  cap_pentry: replica: "[ ]", epoch: 92841, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 0, type: "PT_CAP", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 0, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0

I1119 16:49:46.317624 55463 meta_rpc_server.cc:1589][YIWU1 CHUNK INFO]
cid: 1, perf thin allocated: 139389440 KiB, ratio: 0.332331, thin info: [perf_thin_used_data_space: (94562816), perf_new_thin_pids: (1, 262144), perf_rx_pids: (170, 44564480) KiB]
cid: 3, perf thin allocated: 258778880 KiB, ratio: 0.616978, thin info: [perf_thin_used_data_space: (258516736), perf_new_thin_pids: (1, 262144), perf_rx_pids: (0, 0) KiB]
cid: 4, perf thin allocated: 258714112 KiB, ratio: 0.616824, thin info: [perf_thin_used_data_space: (258714112), perf_new_thin_pids: (0, 0), perf_rx_pids: (0, 0) KiB]
```

对应的 cap loc 申请

```
I1119 16:49:46.324748 55463 meta_rpc_server.cc:1675] [ALLOC PT_CAP PEXTENT FOR SINK]  lid: 1173 lentry: epoch: 42677, perf_pid: 94843, perf_epoch: 94843, cap_pid: 92841, cap_epoch: 92841, prioritized: 0, garbage: 0, valid: 1, staging: 1  perf_pentry: replica: "[ 0:1 1:3 ]", epoch: 94843, generation: 1, origin_pid: 0, origin_epoch: 0, ever_exist: 1, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_PERF", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 1, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0  cap_pentry: replica: "[ 0:4 1:3 ]", epoch: 92841, generation: 0, origin_pid: 0, origin_epoch: 0, ever_exist: 0, garbage: 0, valid: 1, expected_replica_num: 2, staging: 0, thin_provision: 1, preferred_cid: 4, type: "PT_CAP", resiliency_type: "RT_REPLICA", ec_param: "None", sinkable: 0, allocated_space: 0, thin_uniq_size: 0, thin_shared_size: 0, cow_from_snapshot: 0
I1119 16:49:46.324784 55463 meta_rpc_server.cc:1696]

[YIWU3 CHUNK INFO]
cid: 1, cap allocated: 1024, ratio: 0.000000; perf thick allocated: 0 KiB, thin allocated: 141224448 KiB, ratio: 0.336706, thin info: [perf_thin_used_data_space: (94562816), perf_new_thin_pids: (8, 2097152), perf_rx_pids: (170, 44564480) KiB]
cid: 3, cap allocated: 2360320, ratio: 0.000257; perf thick allocated: 0 KiB, thin allocated: 259565312 KiB, ratio: 0.618853, thin info: [perf_thin_used_data_space: (258516736), perf_new_thin_pids: (4, 1048576), perf_rx_pids: (0, 0) KiB]
cid: 4, cap allocated: 2360320, ratio: 0.000257; perf thick allocated: 0 KiB, thin allocated: 259762688 KiB, ratio: 0.619324, thin info: [perf_thin_used_data_space: (258714112), perf_new_thin_pids: (4, 1048576), perf_rx_pids: (0, 0) KiB]
```



但 256k app io 会直写 cap，而 cap loc 又为空，所以 access 主动放弃 lease

```
W1119 16:49:53.375825 85152 access_io_handler.cc:1578] prepare cap layer for direct write not ok, lease: lease_id: 1173, lease_epoch: 42677, proxy_lid: 0, proxy_epoch: 0, owner: 4, cow: 0, expired: 1, version: "LV_LAYERED"  perf_pextent_info: pid: 94843, epoch: 94843, origin_pid: 0, origin_epoch: 0, ever_exist: 1, meta_generation: 1, expect_replica_num: 2, loc: "[1 3 ]", cow_from_snapshot: 0  capacity_pextent_info: pid: 92841, epoch: 92841, origin_pid: 0, origin_epoch: 0, ever_exist: 0, meta_generation: 0, expect_replica_num: 2, loc: "[]", cow_from_snapshot: 0 , st:
Traceback:
[EAgain]: cap segments is not allocated or synced, old lease: lease_id: 1173, lease_epoch: 42677, proxy_lid: 0, proxy_epoch: 0, owner: 4, cow: 0, expired: 1, version: "LV_LAYERED"  perf_pextent_info: pid: 94843, epoch: 94843, origin_pid: 0, origin_epoch: 0, ever_exist: 1, meta_generation: 1, expect_replica_num: 2, loc: "[1 3 ]", cow_from_snapshot: 0  capacity_pextent_info: pid: 92841, epoch: 92841, origin_pid: 0, origin_epoch: 0, ever_exist: 0, meta_generation: 0, expect_replica_num: 2, loc: "[]", cow_from_snapshot: 0 , new lease: lease_id: 1173, lease_epoch: 42677, proxy_lid: 0, proxy_epoch: 0, owner: 4, cow: 0, expired: 0, version: "LV_LAYERED"  perf_pextent_info: pid: 94843, epoch: 94843, origin_pid: 0, origin_epoch: 0, ever_exist: 1, meta_generation: 1, expect_replica_num: 2, loc: "[1 3 ]", cow_from_snapshot: 0  capacity_pextent_info: pid: 92841, epoch: 92841, origin_pid: 0, origin_epoch: 0, ever_exist: 0, meta_generation: 0, expect_replica_num: 2, loc: "[4 3 ]", cow_from_snapshot: 0
E1119 16:49:53.375845 85152 internal_io_client.cc:259] Failed to handle local IO: [IOCTX]: VEXTENT_WRITE seq: 1196082, volume: "c0f40f0a-1a3f-4831-a87c-8448bb007396", in_progress: 1, volume_offset: 313534709760, len: 262144, return_not_alloc: 0, resize: 0, retry_times: 0, next_retry_ms: 0 , vextent_no: 1168, data_len: 262144, extent_offset: 262144, lid: 1173, owner_ip: "10.168.66.64", owner_port: 10201  st:
Traceback:
[ELeaseExpired]:
W1119 16:49:53.375859 85152 zbs_client.cc:1005] [Add TO RETRY QUEUE]: [IOCTX]: VEXTENT_WRITE seq: 1196082, volume: "c0f40f0a-1a3f-4831-a87c-8448bb007396", in_progress: 1, volume_offset: 313534709760, len: 262144, return_not_alloc: 0, resize: 0, retry_times: 0, next_retry_
ms: 1711844953 , vextent_no: 1168, data_len: 262144, extent_offset: 262144, lid: 1173, owner_ip: "10.168.66.64", owner_port: 10201  queue size: 56 st:
Traceback:
[ELeaseExpired]:
```





### Reroute 进程卡住不工作

esxi 中的 ptrace 默认 disable 了，没法 gdb attach 一个正在运行的 python 进程



通过 ps -c| grep reroute.py | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill 来把所有的都 kill，但其实 1725208 还在，因为他不响应 15 信号，不过不影响新的 reroute 进程的拉起，这个进程变成僵尸进程，一直在了

主动 kill 1725208

为啥即使是正常的 reroute 进程，也会出现某个时刻会有多个线程的状态？这已经是协程使用模式了

有问题的 reroute 进程，总是有多个，这个也不太符合预期，预期只会有一个运行，手动把他 kill 掉就好了

这个 2115326，理论上应该是 

当子进程（WID）的 CID 不是 reroute 主进程的话，就会出现卡死的现象



可以验证文件锁是没有问题的，进程卡住时，crontab 新起的进程会被拦住，且主线程确实停了，没有打印每次循环开始都会打印的 log，ping 线程还在工作。

```
[PID:2631427] 2024-07-21 00:24:41 reroute.py:1853 INFO start reroute checking 12328
[PID:2631427] 2024-07-21 00:24:43 reroute.py:1853 INFO start reroute checking 12329
[PID:3060995] 2024-07-21 00:25:00 reroute.py:1816 WARNING lock file is locked, reroute service is already running, process 3060995 exit actively
[PID:3061516] 2024-07-21 00:26:00 reroute.py:1816 WARNING lock file is locked, reroute service is already running, process 3061516 exit actively
// 在主动 kill 之后，发现主进程继续工作了
[PID:2631427] 2024-07-21 04:09:49 reroute.py:732 WARNING child pid: 3060822 run cmd: ['esxcfg-route', '-l'], timeout: 2, cmd_retcode: -9, cmd_stdout: , cmd_stderr:
[PID:2631427] 2024-07-21 04:09:49 reroute.py:1879 WARNING dst ip is unknown, skip this round
[PID:2631427] 2024-07-21 04:09:51 reroute.py:1853 INFO start reroute checking 12330
```

但是诡异的是，此时多次连续执行 ps -c | grep reroute，可以看到很多主进程有超过 2 + 4 个的线程，不过先不管，不是重点。



处理售后 case，出问题的 3 个节点表现一致：ESXI 上 Reroute 进程仍存在，但不打印日志，跟 zbs insight 心跳失联。单节点的多个 Reroute 进程中大部分都能响应 SIGTERM 立马被 kill，但会剩一个 Reroute 进程需要通过 SIGKILL 才能完全杀死。上去排查日志看到退出栈停在 run_cmd 的 execute_child 上，Reroute 每个周期（2s）会通过起子进程的方式来在 ESXi 上执行 shell 命令如查看路由表、网卡信息，这里怀疑有可能是频繁创建/销毁子进程时卡住了。CPU 在不兼容性列表里



把 log size 调大些

下次如果还发现这个情况，试着执行 esxcfg-route -l 或 esxcfg-vmknic -l，看看是不是这 2 条命令卡在驱动上

python 有没有办法不开子进程去 run cmd 

除了 CPU 的差异，节点间可能还有其他配置/网卡差异导致软件运行上的不一致。这么上层的问题直接怀疑 CPU 不太科学。

```
import os
import shlex
import signal
import subprocess
import time

from threading import Timer as timer

def kill_process(pid, signal_type=signal.SIGKILL):
    try:
        print('kill %s active' % pid)
        os.kill(pid, signal_type)
    except OSError:
        pass


def run_cmd(cmd, shell=False, timeout=2):
    if not shell:
        cmd = shlex.split(cmd)

    cmd_proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, shell=shell)

    timeout_handler = None
    if timeout > 0:
        timeout_handler = timer(timeout, lambda: kill_process(cmd_proc.pid))
        timeout_handler.setDaemon(True)
        timeout_handler.start()
    try:
        cmd_std_result = cmd_proc.communicate()
        cmd_retcode = cmd_proc.returncode
    finally:
        if timeout_handler:
            timeout_handler.cancel()

    cmd_stdout = cmd_std_result[0].decode("utf-8")
    cmd_stderr = cmd_std_result[1].decode("utf-8")
    if cmd_retcode != 0:
        print('execute cmd: %s, timeout: %s, parent pid: %s, child pid: %s' % (cmd, timeout, os.getpid(), cmd_proc.pid))
        print('cmd_retcode: %s, cmd_stdout: %s, cmd_stderr: %s' % (cmd_retcode, cmd_stdout, cmd_stderr))
    return cmd_retcode, cmd_stdout, cmd_stderr

for i in range(100000000):
    time.sleep(1)
    print(i, end='\t', flush=True)
    run_cmd("dd if=/dev/random of=/scratch/log/yiwu8m.log bs=1k count=8192", timeout=0)
    run_cmd("esxcfg-route -l")
    run_cmd("esxcfg-vmknic -l")

    run_cmd("cp -f /scratch/log/yiwu8m.log /scratch/log/yiwu8m.log.bak", timeout=0)
    run_cmd("esxcfg-vmknic -l")
    run_cmd("esxcfg-route -l")
```



### 集群升级过程中能力协商不兼容

access manager

1. MetaServer::StartService() --> MetaContext::Initialize() --> AccessManagerInitialize()
2. AccessManager::DoInit()，打印 Initializing AccessManager，然后按顺序执行 chunk 侧的 zk、session master、data report master 的初始化；
3. SessionMaster::Initialize()，先查 session db，如果其中存在还没过期的 session，那么会复用该 session，否则新建 session。接着调用 SessionInitializeCb()，如果是新建的 session，那么会在此时就通过 ComposeNegotiatedResponse() 在 keep alive 之前就先能力协商相关信息传入，access handler 会在它的 SessionCreateCb() 中处理。
4. 继续建立 session client info 并 Start session（lease 就会开始延长），最后调用 SessionCreatedCb()，打印  ACCESS SESSION CREATED；
5. access manager 对外暴露 HandleKeepAlive / HandleDataReport  rpc 用以接受每个 chunk 的 keep alive / data report，能力协商相关信息也会借助 KeepAliveResponse 传回去；

access handler

1. ChunkServer::Start() --> AccessHandler::Start() --> AccessHandler::CreateSession()
2. SessionFollower::CreateSession()，先调用 SessionClient::CreateSession()，创建成功后调用 SessionCreatedCb()，这里面会通过 HandleNegotiatedResponse() 在 keep alive 之前先协商一次，然后调用 KeepAliveLoop()
3. KeepAliveLoop 中通过 KeepAliveCb() 消化上一轮的包含能力协商信息的心跳返回内容（对应函数 HandleAccessResponse / HandleChunkResponse），并设置这一轮的包含能力协商信息的心跳上传内容（ComposeAccessKeepAliveRequest / ComposeChunkRequest），最后通过 SessionClient::KeepAlive 调用 session master 的 KeepAlive rpc，实际上就是 HandleKeepAlive rpc

光大集群从 505 升级到 513 recover 很慢

本身磁盘延迟也高，有 lsm slow io，大概在 100 - 300 ms，但还没触发 recover timeout，所以可以把 cmd slots 调高

recover dst 都选了同一个

瓶颈不在 chunk 侧，因为从 recover handler 开始执行 recover cmd 到结束，基本不超过 2 min，虽然有 lsm slow io，但基本上就 100 - 200 ms



17:44:24 想进维护模式

17:49:16 才下发 recover cmd，这些 recover cmd 的 replace cid 都是 2，在这之前，下发了一波 dst = 2 的 migrate cmd，把 cid 2 的 avail cmd slot 用满了。

replace cid 都是 2，比较像是他们真的 10 min 没有被 data report，数据本身是好的，gen 也对，所以 access handler 侧的 recover cmd 基本上也是执行失败，关键还会把健康的副本置为 invalid，这样副本就真没了，触发了一个 dst = 5 的 recover cmd，这次会成功，但是已经浪费了 3 min。

（不过为啥没有被 data report 很奇怪）

这些 recover cmd 的 replace 和 dst 都是 2，recover cmd 是因为有 dead replica，但给到 dst = 2 的这条 recover cmd 并没有成功，重试了 dst = 5 成功，前后耗时 3min

17:50:17 才开始下发 replace = 0 的 recover cmd（这中间 1min 下发了 15 次，都是 replace 和 dst = 2 的 recover cmd），但他们的 dst 又都是 5 了。

18:15:45 附近开始穿插一些 dst = 2 && replace = 5 的 recover cmd，且数量也很多，这两类 recover cmd 都有发

19:16:05 才真正进入维护模式



升级日志中搜 Wait data recover 和 Set chunk maintenance mode 和 Set data static recover limit

如果是敏捷恢复期间产生的 need recover，会被记录在 rim_pextent_num 中

除了读写初次 sync 会剔除 gen 不一样的 segment，ec 只有在写失败才会剔除 shard，并且不会创建临时副本。

进入维护模式后，会 revoke lease，那么下次读的时候，重新拿到 lease 后的初次读之前，会 sync，此时会剔除那些 gen 不一致的 segment 引发 recover



受限于 recover dst 基本都选 prefer local，prefer local 此时成为 recover cmd 下发的瓶颈，不过这个算预期内的

恢复过程中穿插了 migrate cmd，不过都是在 recover 完成之后的间隙里。影响不大

avail cmd slots 忽略 cmd src 会被 lease owner 替换的情况，即它限制的是 old src

只统计 agile recover 没有意义，因为 ec recover 是一定不会走 agile recover 的，需要看待恢复数据中 ec 和 replica 的比例

升级过程中查看 ongoing_recover 的值，如果没有降到 0，说明慢的瓶颈在 access 侧。



少量非预期日志的出现，晚点可以查一下这些 pid 的特点

```
W0712 14:06:47.404752 19128 physical_extent_table_entry.cc:147] [RECOVERING PEXTENT REPORTED]:  pid: 6052057 cid: 18
I0712 14:06:47.404778 19128 physical_extent_table_entry.cc:124] [IGNORE SMALL GEN]: wrong gen: 5 expected gen: 439 pid: 6052072 now_ms: 5187225614 cid: 18
W0712 14:06:47.404783 19128 physical_extent_table_entry.cc:147] [RECOVERING PEXTENT REPORTED]:  pid: 6052080 cid: 18
```

还是出现了 data report 超过 7s 没处理完的日志

```
W0712 14:06:58.241887 19128 rpc_server.cc:186] [RPC] Undone message socket: 0x564d28bf1ae0 OnReadContext: header: error_code: 0, service_id: 7002, method_id: 0, timeout_ms: 7000, message_id: 136, buf_len: 3157303 , offset: 1243800, elapsedTime: 108
W0712 14:06:58.351882 19128 rpc_server.cc:186] [RPC] Undone message socket: 0x564d28bf1ae0 OnReadContext: header: error_code: 0, service_id: 7002, method_id: 0, timeout_ms: 7000, message_id: 136, buf_len: 3157303 , offset: 2735240, elapsedTime: 218
```

### 快速切出切回路由出现 IO 重试

由于 IOReroute 在无网络故障时有一定概率会发生本地路由快速切出切回以及 ESXi NFS Client 有一定概率会复用之前的端口号导致出现超过 1min 的 IO 中断时间，触发 Oracle rac 虚拟机保护机制，自动重启。



SCVM 上的时间线

1. 12:15:25，scvm222 上新建 4 个连接，端口 903，899，900，901；
2. 路由从 10.22（scvm 222） 切出到 10.21（scvm 111）；
3. 12:16:27，scvm111 上新建 4 个连接，端口 903，904，905，906；
4. 12:16:27，scvm222 上关闭 4 个连接，端口 903，899，900，901，端口 903 对应的 socket 开始处于 FIN_WAIT1 的状态直到 12:18:13；
5. 路由从  10.21（scvm 111） 切回到 10.22（scvm 222）；
6. 12:16:30，scvm222 上新建 3 个连接，端口 907，908，909；
7. 12:16:30，scvm111 上关闭 4 个连接，端口 903，904，905，906，端口 903 对应的 socket 开始处于 FIN_WAIT1 的状态直到 12:17:32；

ESXi 上的时间线

1. 12:15:26，存在占用端口 903，899，900，901 的连接；
2. 12:16:28，存在占用端口 903，904，905，906 的连接；
3. 12:16:31，存在占用端口 903，907，908，909 的连接，并在这之后，端口 903 对应的连接的 Recv Q 始终有值且在累加，从创建以来就是 ESTABLISHED 状态，直到 12:17:33 该连接断开（此时内核日志中有 VSCSI Reset）。

```shell
# 抓存储网卡上的 syn 和 rst 包，端口 2049 是 nfs server 监听端口
tcpdump -i ens224 "tcp[tcpflags] & (tcp-syn|tcp-rst) != 0" and port 2049 -w /tmp/scvm111-yiwu.pcap
# 读取抓包内容
tcpdump -r /tmp/scvm111-yiwu.pcap
# scvm 查看端口连接
netstat -anl | grep 2049
# esxi 查看端口连接
esxcli network ip connection list | grep 2049
# 在 scvm 上用丢 ping 包的方式模拟切出切回
while true; do iptables -A INPUT -i ens224 -p icmp --icmp-type echo-request -j DROP; sleep 4; iptables -D INPUT -i ens224 -p icmp --icmp-type echo-request -j DROP; sleep 60; done
# 查看 iptables 配置
iptables -L 
```

scvm 上查看

```shell
# 为 volume 注入延迟
zbs-chunk volume inject_latency bc04f2bc-0558-4781-89be-bf1ecc53272f --readwrite_latency_ms 10000
# 用以查看有 IO 的 volume
zbs-perf-tools volume list
# 查看所有 export
zbs-nfs export list
# 查看 export 包含的文件，从 nfs 对应的 volume id 中取前 3 段
zbs-nfs inode list 8b2ccfaf-00a0-4e34
# 用 DIR 的 inode id 可以层层往下查看最终的 FILE
```

esxi 上

```shell
# 查看所有 datastore，包括本地的和通过 nfs 挂载的
cd /vmfs/volumes/ 

tcpdump-uw -i vmk1 "tcp[tcpflags] & (tcp-ack|tcp-syn|tcp-fin) != 0" and host 10.0.0.21 -nnn -w /tmp/esxi1-yiwu.pcap

esxcfg-route -d 192.168.33.2/32 10.0.0.21; esxcfg-route -a 192.168.33.2/32 172.20.249.54;
```

fio-test vm

```shell
fio --name a --filename=/dev/mapper/centos-root --bs=4k --rw=randwrite --ioengine=libaio --direct=1 --iodepth=128 --numjobs=1 --time_based --runtime=300000s
```



安装 yum install lz4-devel 失败，可以将 smartx.repo 传过来，另外也可以手动将 rpm 下载到本地，用 rpm -Uvh xxx.rpm 安装

缺失 pmem.h 等包后，可以在编译时 cmake -DBUILD_ENABLE_PMEM=OFF -G Ninja .. 来跳过检查

tcmpdump 抓包，-n 显示 ip 地址，而不是 hostname，-nn 显示端口号而不是协议名，过滤源和目的IP地址以及特定协议

```shell
tcpdump -i eth0 src host 192.168.1.2 and src port 8080 and dst host 192.168.1.3 and dst port 9090 and not icmp
```

wireshark 过滤 tcp 指定标志位的数据报，syn / ack / fin 同理

```
tcp.flags.reset == 1
```

wireshark 关闭相对 TCP 序号，在任一抓包上单点右键，选择协议首选项，然后在弹出菜单中出去 relative seq 前的勾选去掉 relative seq num

修改内核参数

```shell
echo "4" > /proc/sys/net/ipv4/tcp_orphan_retries 
# 在 /etc/sysctl.conf 中修改 net.ipv4.tcp_orphan_retries=4
sysctl -a | grep tcp_orphan_retries 查看配置 
```



### 不在机柜的节点跟在机柜的节点拓扑距离是 0

```c++
TEST_F(RecoverManagerTest, YIWU) {
    RecoverManager recover_manager(&(GetMetaContext()));

    ChunkTable* chunk_table = GetMetaContext().chunk_table;

    const int rack_num = 2;
    TopoObj racks[rack_num];
    ASSERT_STATUS(meta->CreateTopoObj(RACK, kDefaultZoneId, "rack0", &racks[0], 1, 2, 1, 1, 1, 1));
    ASSERT_STATUS(meta->CreateTopoObj(RACK, kDefaultZoneId, "rack1", &racks[1], 1, 2, 1, 2, 1, 1));
    const int brick_num = 2;
    TopoObj bricks[brick_num];
    ASSERT_STATUS(meta->CreateTopoObj(BRICK, racks[0].id(), "brick0", &bricks[0], 1, 2, 1, 1, 1, 1));
    ASSERT_STATUS(meta->CreateTopoObj(BRICK, racks[1].id(), "brick1", &bricks[1], 1, 2, 1, 2, 1, 1));

    uint64_t per_host_num = 100;
    constexpr int chunk_num = 7;
    cid_t cids[chunk_num];

    LOOP(chunk_num) {
        Chunk chunk;
        RegisterChunk("127.0.0.1", 3401 + i * 10, &chunk);
        cids[i] = chunk.id();
        ChunkSpaceInfo info;
        info.set_valid_data_space(per_host_num * kExtentSize);
        chunk_table->TEST_SetChunkSpaceInfo(cids[i], info);
        chunk_table->SetChunkStatus(cids[i], CHUNK_STATUS_CONNECTED_HEALTHY);
        if (i != 4) {
            ASSERT_STATUS(meta->MvTopoObj(chunk.topo_id(), (i <= chunk_num / 2 ? bricks[0].id() : bricks[1].id())));
        }
    }

    // make brick0 has 50 + 60 + 60 + 60 = 230 pids, brick1 has 96 + 67 + 67 = 230 pids,
    // and then, avg_ratio = (230 + 230) / (4 + 3) = 65.71.
    // cid:             1   2   3   4   5   6   7
    // brick id:        0   0   0   0   1   1   1
    // allocated pids:  50  60  60  60  96  67  67
    // valid pids:     100 100 100 100 100 100 100

    PExtentTable* pextent_table = GetMetaContext().pextent_table;
    IdMap *pid_map = GetMetaContext().pid_map;

    std::vector<cid_t> cap_segments;
    LOOP(230) {
        cap_segments.clear();
        if (i < 50) {
            cap_segments.push_back(cids[0]);
        } else if (i < 50 + 60) {
            cap_segments.push_back(cids[1]);
        } else if (i < 50 + 60 + 60) {
            cap_segments.push_back(cids[2]);
        } else {
            cap_segments.push_back(cids[3]);
        }
        if (i < 96) {
            cap_segments.push_back(cids[4]);
        } else if (i < 96 + 67) {
            cap_segments.push_back(cids[5]);
        } else {
            cap_segments.push_back(cids[6]);
        }

        loc_t loc = 0U;
        for (auto cid : cap_segments)  {
            loc = add_replica_to_location(loc, cid);
        }
        PExtent pextent;
        pextent.set_pid(i + 1);
        pextent.set_location(loc);
        pextent.set_expected_replica_num(2);
        pextent.set_ever_exist(false);
        pextent.set_thin_provision(false);
        pextent.set_preferred_cid(kInvalidChunkId);
        pextent_table->SetPExtent(pextent);
        pid_map->SetId(i + 1);
    }

    recover_manager.EnableScanMigrateImmediate();
    recover_manager.DoScan();
    std::vector<RecoverCmd> cmds;
    recover_manager.ListMigrateCmds(&cmds);
    ASSERT_TRUE(cmds.empty());
}
```



### lease owner 分散到各个节点

1. 抓包，明确通过 port-storage 接口，目的端口是 10100，源或目的主机是 10.10.51.75，抓 1000 个包自动结束，多次抓包主动删除文件。

    tcpdump -i port-storage dst port 10100 and host 10.10.51.75 -c 1000 -w /tmp/2.pcap

2. 查看新增磁盘数据块信息

    1. 界面上创建的磁盘，点击详情，里面的存储对象，iscsi://iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-1710222435766f/167；
    2. 通过 zbs-iscsi lun show zbs-iscsi-datastore-1710222435766f 167 查看它的 volume id 14fd95fb-9a32-4a5a-97cb-c2761db5ced1；
    3. 然后通过 zbs-meta volume show_by_id 14fd95fb-9a32-4a5a-97cb-c2761db5ced1 --show_pextents 查看副本分布。

3. fio 也可以远程运行，只要在该节点上运行 fui --server，然后通过 fio --client=192.168.61.110 firstwrite.cfg 来运行，其中 filename 可以替换到刚刚新增磁盘的路径，size 改成对应磁盘大小

    ```
    [root@localhost fioconfig]# cat firstwrite.cfg
    [global]
    ioengine=libaio
    direct=1
    thread=1
    norandommap=1
    randrepeat=0
    ##runtime=1h
    ramp_time=0
    size=30g
    filename=/dev/vdc
    
    [write1m-seq]
    ###time_based
    stonewall
    group_reporting
    bs=256k
    rw=write
    numjobs=1
    iodepth=128
    ```

4. 以二进制方式打印 header 和 buffer，与抓包内容比对。

    ```shell
    auto func = [](const char* data, int len) {
    	std::ostringstream oss;
    	for (int i = 0; i < len; i ++) {
    		oss << std::hex << std::setfill('0') << std::setw(2) << static_cast<int>(data[i]) << " ";
    	}
    	oss << std::dec << std::endl;
    	return oss.str();
    };
    
    # RpcServer::OnRead 服务端接收到的包
    LOG(INFO) << "yiwu [RPC] header: {" << " error_code: " << header->error_code << " service: " << sd->name() << " method: " << md_name << " timeout_ms: " << header->timeout_ms << " message_id: " << header->message_id << " message_len: " << header->buf_len << "}," << " request: {" << request->ShortDebugString() << "}" << " socket local addr: " << socket->GetLocalAddr() << " peer addr: " << socket->GetPeerAddr() << " header: " << func(reinterpret_cast<char*>(header), sizeof(RpcHeader)) << " context: " <<  func(context->GetBuffer().get(), header->buf_len);
    
    # CoAsyncRpcClient::CallMethod 客户端发送
    LOG(INFO) << "yiwu CoAsyncRpcClient::CallMethod request " << request->ShortDebugString() << " rpc client header " << req_hdr.DebugString() << " rpc client header " << func(reinterpret_cast<char*>(&req_hdr), sizeof(RpcHeader)) << " rpc client buffer " << func(req_buf, message_len);
    ```

### ESXi 上添加虚拟磁盘对应的副本分配

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

    只是由首次读引入的 sync generation，虽然会让 loc 上的各个 chunk 去真实分配 extent，但 gen 还是 0，所以 ever exist = false

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



[root@ESXI02:~] vmware -v
VMware ESXi 7.0.3 build-20328353

zbs-5.5.1-rc20.0.release.git.g06d26a37b.el7.SMTX.HCI.x86_64

cid2

volume 粒度

```c++
[root@node3 23:48:49 sbin]$ grep -wn "566adf10-fb26-467a-82d8-b490345a2fe3" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.INFO:5536:I1218 23:50:09.164472 12540 access_handler.cc:885] [REVOKE LEASE]: vtable: 566adf10-fb26-467a-82d8-b490345a2fe3
/var/log/zbs/zbs-chunkd.log.20231218-234846.12535:5536:I1218 23:50:09.164472 12540 access_handler.cc:885] [REVOKE LEASE]: vtable: 566adf10-fb26-467a-82d8-b490345a2fe3
You have new mail in /var/spool/mail/root
[root@node3 23:51:41 sbin]$ grep -wn "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.INFO:94873:I1218 23:57:19.930114 12540 access_handler.cc:885] [REVOKE LEASE]: vtable: 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:94874:I1218 23:57:19.967772 12540 nfs3_server.cc:1745] [VAAI RESVSPACE]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:94964:I1218 23:57:20.139151 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95034:I1218 23:57:20.243119 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95055:I1218 23:57:20.279748 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95076:I1218 23:57:20.304571 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95175:I1218 23:57:20.461102 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95220:id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95232:I1218 23:57:20.557519 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95233:I1218 23:57:20.557528 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95235:I1218 23:57:20.558077 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 0
/var/log/zbs/zbs-chunkd.INFO:95251:I1218 23:57:20.561120 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95252:I1218 23:57:20.561141 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95253:I1218 23:57:20.561151 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 4096
/var/log/zbs/zbs-chunkd.INFO:95260:I1218 23:57:20.569312 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95261:I1218 23:57:20.569335 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95263:I1218 23:57:20.569945 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268369920
/var/log/zbs/zbs-chunkd.INFO:95279:I1218 23:57:20.576017 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95280:I1218 23:57:20.576038 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95281:I1218 23:57:20.576049 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268427264
/var/log/zbs/zbs-chunkd.INFO:95288:I1218 23:57:20.578969 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95289:I1218 23:57:20.578989 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95290:I1218 23:57:20.578999 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 0
/var/log/zbs/zbs-chunkd.INFO:95297:I1218 23:57:20.579429 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95298:I1218 23:57:20.579450 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95299:I1218 23:57:20.579460 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 4096
/var/log/zbs/zbs-chunkd.INFO:95306:I1218 23:57:20.581504 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95307:I1218 23:57:20.581526 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95308:I1218 23:57:20.581537 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268431360
/var/log/zbs/zbs-chunkd.INFO:95315:I1218 23:57:20.584790 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95316:I1218 23:57:20.584811 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95317:I1218 23:57:20.584821 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268300288
/var/log/zbs/zbs-chunkd.INFO:95324:I1218 23:57:20.587162 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95325:I1218 23:57:20.587183 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95326:I1218 23:57:20.587198 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268402688
/var/log/zbs/zbs-chunkd.INFO:95333:I1218 23:57:20.587815 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95334:I1218 23:57:20.587836 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95335:I1218 23:57:20.587846 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268304384
/var/log/zbs/zbs-chunkd.INFO:95342:I1218 23:57:20.588296 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95343:I1218 23:57:20.588316 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95344:I1218 23:57:20.588326 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 4096 extent_offset 268230656
/var/log/zbs/zbs-chunkd.INFO:95351:I1218 23:57:20.592900 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95352:I1218 23:57:20.592921 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95362:I1218 23:57:20.593647 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268394496
/var/log/zbs/zbs-chunkd.INFO:95376:I1218 23:57:20.598439 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95377:I1218 23:57:20.598460 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95378:I1218 23:57:20.598471 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268349440
/var/log/zbs/zbs-chunkd.INFO:95385:I1218 23:57:20.600032 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95386:I1218 23:57:20.600052 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95387:I1218 23:57:20.600062 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268320768
/var/log/zbs/zbs-chunkd.INFO:95394:I1218 23:57:20.601107 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95395:I1218 23:57:20.601128 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95396:I1218 23:57:20.601138 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268230656
/var/log/zbs/zbs-chunkd.INFO:95403:I1218 23:57:20.603098 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95404:I1218 23:57:20.603121 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95405:I1218 23:57:20.603132 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268197888
/var/log/zbs/zbs-chunkd.INFO:95414:I1218 23:57:20.606535 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95415:I1218 23:57:20.606555 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95416:I1218 23:57:20.606565 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268189696
/var/log/zbs/zbs-chunkd.INFO:95425:I1218 23:57:20.607350 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95426:I1218 23:57:20.607370 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95427:I1218 23:57:20.607380 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268210176
/var/log/zbs/zbs-chunkd.INFO:95438:I1218 23:57:20.613101 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95439:I1218 23:57:20.613127 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95441:I1218 23:57:20.613878 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 381 data_len 4096 extent_offset 268165120
/var/log/zbs/zbs-chunkd.INFO:95452:I1218 23:57:20.616346 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95461:I1218 23:57:20.618173 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95462:I1218 23:57:20.618194 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95463:I1218 23:57:20.618204 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 262144
/var/log/zbs/zbs-chunkd.INFO:95470:I1218 23:57:20.620843 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95471:I1218 23:57:20.620867 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95472:I1218 23:57:20.620877 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 12288
/var/log/zbs/zbs-chunkd.INFO:95479:I1218 23:57:20.622473 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95480:I1218 23:57:20.622494 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95481:I1218 23:57:20.622505 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 28672
/var/log/zbs/zbs-chunkd.INFO:95488:I1218 23:57:20.624567 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95489:I1218 23:57:20.624589 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95490:I1218 23:57:20.624599 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 61440
/var/log/zbs/zbs-chunkd.INFO:95497:I1218 23:57:20.627243 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95498:I1218 23:57:20.627264 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95499:I1218 23:57:20.627274 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 8192
/var/log/zbs/zbs-chunkd.INFO:95505:I1218 23:57:20.627511 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95506:I1218 23:57:20.627532 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95507:I1218 23:57:20.627542 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 12288 extent_offset 16384
/var/log/zbs/zbs-chunkd.INFO:95513:I1218 23:57:20.627599 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95514:I1218 23:57:20.627611 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95515:I1218 23:57:20.627619 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 28672 extent_offset 32768
/var/log/zbs/zbs-chunkd.INFO:95523:I1218 23:57:20.627750 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95524:I1218 23:57:20.627763 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95525:I1218 23:57:20.627770 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 73728 extent_offset 65536
/var/log/zbs/zbs-chunkd.INFO:95542:I1218 23:57:20.635885 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95543:I1218 23:57:20.635906 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95544:I1218 23:57:20.635917 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 122880 extent_offset 139264
/var/log/zbs/zbs-chunkd.INFO:95551:I1218 23:57:20.639458 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95552:I1218 23:57:20.639479 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95554:I1218 23:57:20.640123 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 1 data_len 131072 extent_offset 0
/var/log/zbs/zbs-chunkd.INFO:95568:I1218 23:57:20.651880 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95569:I1218 23:57:20.651901 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95570:I1218 23:57:20.651911 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 1 data_len 131072 extent_offset 131072
/var/log/zbs/zbs-chunkd.INFO:95581:I1218 23:57:20.655007 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95582:I1218 23:57:20.655028 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95583:I1218 23:57:20.655040 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 16384 extent_offset 268173312
/var/log/zbs/zbs-chunkd.INFO:95589:I1218 23:57:20.655110 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95590:I1218 23:57:20.655122 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95591:I1218 23:57:20.655130 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 4096 extent_offset 268193792
/var/log/zbs/zbs-chunkd.INFO:95597:I1218 23:57:20.655222 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95598:I1218 23:57:20.655232 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95599:I1218 23:57:20.655241 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 8192 extent_offset 268201984
/var/log/zbs/zbs-chunkd.INFO:95605:I1218 23:57:20.655392 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95606:I1218 23:57:20.655412 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95607:I1218 23:57:20.655422 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 16384 extent_offset 268214272
/var/log/zbs/zbs-chunkd.INFO:95613:I1218 23:57:20.655534 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95614:I1218 23:57:20.655553 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95615:I1218 23:57:20.655562 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 69632 extent_offset 268234752
/var/log/zbs/zbs-chunkd.INFO:95629:I1218 23:57:20.657874 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95633:I1218 23:57:20.658442 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95634:I1218 23:57:20.658463 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95635:I1218 23:57:20.658473 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 16384 extent_offset 268304384
/var/log/zbs/zbs-chunkd.INFO:95641:I1218 23:57:20.658542 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95642:I1218 23:57:20.658553 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95643:I1218 23:57:20.658561 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 24576 extent_offset 268324864
/var/log/zbs/zbs-chunkd.INFO:95649:I1218 23:57:20.658651 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95650:I1218 23:57:20.658670 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95651:I1218 23:57:20.658680 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 40960 extent_offset 268353536
/var/log/zbs/zbs-chunkd.INFO:95657:I1218 23:57:20.658778 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95658:I1218 23:57:20.658797 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95659:I1218 23:57:20.658807 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 382 data_len 36864 extent_offset 268398592
/var/log/zbs/zbs-chunkd.INFO:95669:I1218 23:57:20.661958 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95670:I1218 23:57:20.661979 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95671:I1218 23:57:20.661993 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 57344 extent_offset 268173312
/var/log/zbs/zbs-chunkd.INFO:95678:I1218 23:57:20.662801 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95679:I1218 23:57:20.662822 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95680:I1218 23:57:20.662832 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 65536 extent_offset 268234752
/var/log/zbs/zbs-chunkd.INFO:95687:I1218 23:57:20.663722 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95688:I1218 23:57:20.663754 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95689:I1218 23:57:20.663765 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 61440 extent_offset 268308480
/var/log/zbs/zbs-chunkd.INFO:95695:I1218 23:57:20.663908 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95696:I1218 23:57:20.663938 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95697:I1218 23:57:20.663949 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 28672 extent_offset 268374016
/var/log/zbs/zbs-chunkd.INFO:95703:I1218 23:57:20.664031 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95704:I1218 23:57:20.664044 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95705:I1218 23:57:20.664053 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 383 data_len 20480 extent_offset 268406784
/var/log/zbs/zbs-chunkd.INFO:95714:I1218 23:57:20.664778 12540 internal_io_client.cc:126] yiwu InternalIOClient::SubmitIO volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95715:I1218 23:57:20.664799 12540 internal_io_client.cc:141] yiwu InternalIOClient::SubmitIO GetVExtentLease volume id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-chunkd.INFO:95716:I1218 23:57:20.664809 12540 access_io_handler.cc:423] yiwu AccessIOHandler::ReadVExtent volume_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 vextent_no 0 data_len 4096 extent_offset 524288
/var/log/zbs/zbs-chunkd.INFO:95792:I1218 23:57:20.842830 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
/var/log/zbs/zbs-chunkd.INFO:95804:I1218 23:57:20.855486 12540 nfs3_server.cc:1811] [VAAI STATX]: inode: [HANDLE]: type: 1, inode_id: 10109580989325272214, volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8"
```

pid 粒度

```c++
[root@node3 23:58:02 sbin]$ grep -wn "37840" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.INFO:95262:I1218 23:57:20.569917 12540 meta.cc:570] yiwu Meta::GetVExtentLease pid 37840 my_chunk_id_ 2
/var/log/zbs/zbs-chunkd.INFO:95264:I1218 23:57:20.569962 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95265:I1218 23:57:20.569967 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 18446744073709551615
/var/log/zbs/zbs-chunkd.INFO:95266:I1218 23:57:20.569970 12540 access_io_handler.cc:991] yiwu AccessIOHandler::DoSyncGeneration pid 37840
/var/log/zbs/zbs-chunkd.INFO:95267:I1218 23:57:20.569979 12540 generation_syncor.cc:138] [SYNC GENERATION START] pid: 37840 loc: [2 1 6 ]
/var/log/zbs/zbs-chunkd.INFO:95268:I1218 23:57:20.569994 12540 generation_syncor.cc:312] yiwu trigger GetGeneration pid 37840 cid
/var/log/zbs/zbs-chunkd.INFO:95269:I1218 23:57:20.570013 12540 generation_syncor.cc:312] yiwu trigger GetGeneration pid 37840 cid
/var/log/zbs/zbs-chunkd.INFO:95270:I1218 23:57:20.570034 12540 generation_syncor.cc:312] yiwu trigger GetGeneration pid 37840 cid
/var/log/zbs/zbs-chunkd.INFO:95271:I1218 23:57:20.570127 12540 local_io_handler.cc:272] yiwu HandleGetGeneration pid 37840
/var/log/zbs/zbs-chunkd.INFO:95272:I1218 23:57:20.570161 12541 lsm.cc:1501] yiwu VerifyAndGetGeneration pid 37840
/var/log/zbs/zbs-chunkd.INFO:95273:I1218 23:57:20.570216 12541 lsm.cc:5135] [ALLOC EXTENT] status: EXTENT_STATUS_ALLOCATED pid: 37840 epoch: 39699 generation: 0 bucket_id: 976 einode_id: 4208154 thick_provision: true prior: false
/var/log/zbs/zbs-chunkd.INFO:95274:I1218 23:57:20.572376 12540 generation_syncor.cc:237] [SYNC GENERATION SUCCESS]  pid: 37840 loc: [2 1 6 ] gen: 0 io_hard: 0
/var/log/zbs/zbs-chunkd.INFO:95275:I1218 23:57:20.572404 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95276:I1218 23:57:20.572410 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95278:I1218 23:57:20.572448 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95282:I1218 23:57:20.576056 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95283:I1218 23:57:20.576061 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95284:I1218 23:57:20.576064 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95285:I1218 23:57:20.576068 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95287:I1218 23:57:20.576117 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95309:I1218 23:57:20.581544 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95310:I1218 23:57:20.581548 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95311:I1218 23:57:20.581552 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95312:I1218 23:57:20.581555 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95314:I1218 23:57:20.581625 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95318:I1218 23:57:20.584827 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95319:I1218 23:57:20.584832 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95320:I1218 23:57:20.584836 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95321:I1218 23:57:20.584839 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95323:I1218 23:57:20.584910 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95327:I1218 23:57:20.587205 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95328:I1218 23:57:20.587208 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95329:I1218 23:57:20.587213 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95330:I1218 23:57:20.587215 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95332:I1218 23:57:20.587368 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95336:I1218 23:57:20.587853 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95337:I1218 23:57:20.587857 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95338:I1218 23:57:20.587862 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95339:I1218 23:57:20.587867 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95341:I1218 23:57:20.587921 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95345:I1218 23:57:20.588332 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95346:I1218 23:57:20.588335 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95347:I1218 23:57:20.588338 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95348:I1218 23:57:20.588342 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95350:I1218 23:57:20.588395 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95672:I1218 23:57:20.661998 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95673:I1218 23:57:20.662003 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95674:I1218 23:57:20.662006 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95675:I1218 23:57:20.662009 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95677:I1218 23:57:20.662063 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95681:I1218 23:57:20.662840 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95682:I1218 23:57:20.662843 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95683:I1218 23:57:20.662849 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95684:I1218 23:57:20.662853 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95686:I1218 23:57:20.662904 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95690:I1218 23:57:20.663774 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95691:I1218 23:57:20.663777 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95692:I1218 23:57:20.663781 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95693:I1218 23:57:20.663785 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95698:I1218 23:57:20.663960 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95699:I1218 23:57:20.663964 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95700:I1218 23:57:20.663969 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95701:I1218 23:57:20.663975 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95706:I1218 23:57:20.664059 12540 access_io_handler.cc:459] yiwu AccessIOHandler::ReadVExtentInternal pid 37840
/var/log/zbs/zbs-chunkd.INFO:95707:I1218 23:57:20.664062 12540 access_io_handler.cc:982] yiwu AccessIOHandler::SyncGeneration pid 37840 gen 0
/var/log/zbs/zbs-chunkd.INFO:95708:I1218 23:57:20.664067 12540 access_io_handler.cc:757] yiwu AccessIOHandler::DoReadVExtent pid 37840
/var/log/zbs/zbs-chunkd.INFO:95709:I1218 23:57:20.664069 12540 access_io_handler.cc:897] yiwu AccessIOHandler::ReadReplica pid 37840 cid 2
/var/log/zbs/zbs-chunkd.INFO:95711:I1218 23:57:20.664083 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95712:I1218 23:57:20.664103 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
/var/log/zbs/zbs-chunkd.INFO:95713:I1218 23:57:20.664336 12540 access_io_handler.cc:1253] yiwu AccessIOHandler::ReadVExtentDone pid 37840
```



cid3

volume 粒度

```c++
[root@node2 23:51:34 sbin]$ grep -wn "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.log.20231218-234815.30236:360716:I1218 23:57:19.929402 30241 access_handler.cc:885] [REVOKE LEASE]: vtable: 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8
/var/log/zbs/zbs-metad.INFO:15469:I1218 23:57:19.722865 31136 nfs_server.cc:337] [CREATE INODE]: [REQUEST]: parent_id: "60ca9357-9c5f-4818" name: "zyx1-test-vm03_2-flat.vmdk" type: FILE how: UNCHECKED sattr3 { mode: 384 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } } xid: 1668748745, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 0 ctime { seconds: 1702915039 nseconds: 712888125 } mtime { seconds: 1702915039 nseconds: 712888125 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 10148 us.
/var/log/zbs/zbs-metad.INFO:15470:I1218 23:57:19.742895 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { mode: 384 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 0 ctime { seconds: 1702915039 nseconds: 734441488 } mtime { seconds: 1702915039 nseconds: 712888125 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 8513 us.
/var/log/zbs/zbs-metad.INFO:15471:I1218 23:57:19.778373 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 10737418240 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 10737418240 ctime { seconds: 1702915039 nseconds: 756774326 } mtime { seconds: 1702915039 nseconds: 756774326 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 21666 us.
/var/log/zbs/zbs-metad.INFO:15472:I1218 23:57:19.787062 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 21474836480 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 21474836480 ctime { seconds: 1702915039 nseconds: 779199380 } mtime { seconds: 1702915039 nseconds: 779199380 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 7918 us.
/var/log/zbs/zbs-metad.INFO:15473:I1218 23:57:19.804730 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 32212254720 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 32212254720 ctime { seconds: 1702915039 nseconds: 787824793 } mtime { seconds: 1702915039 nseconds: 787824793 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 16962 us.
/var/log/zbs/zbs-metad.INFO:15474:I1218 23:57:19.829738 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 42949672960 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 42949672960 ctime { seconds: 1702915039 nseconds: 806571031 } mtime { seconds: 1702915039 nseconds: 806571031 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 23225 us.
/var/log/zbs/zbs-metad.INFO:15475:I1218 23:57:19.843844 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 53687091200 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 53687091200 ctime { seconds: 1702915039 nseconds: 831702363 } mtime { seconds: 1702915039 nseconds: 831702363 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 12198 us.
/var/log/zbs/zbs-metad.INFO:15476:I1218 23:57:19.854727 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 64424509440 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 64424509440 ctime { seconds: 1702915039 nseconds: 844364282 } mtime { seconds: 1702915039 nseconds: 844364282 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 10421 us.
/var/log/zbs/zbs-metad.INFO:15477:I1218 23:57:19.867600 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 75161927680 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 75161927680 ctime { seconds: 1702915039 nseconds: 855439709 } mtime { seconds: 1702915039 nseconds: 855439709 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 12218 us.
/var/log/zbs/zbs-metad.INFO:15478:I1218 23:57:19.875872 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 85899345920 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 85899345920 ctime { seconds: 1702915039 nseconds: 868271537 } mtime { seconds: 1702915039 nseconds: 868271537 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 7658 us.
/var/log/zbs/zbs-metad.INFO:15479:I1218 23:57:19.891204 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 96636764160 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 96636764160 ctime { seconds: 1702915039 nseconds: 877350204 } mtime { seconds: 1702915039 nseconds: 877350204 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 13918 us.
/var/log/zbs/zbs-metad.INFO:15480:I1218 23:57:19.914664 31136 nfs_server.cc:1107] [SETATTR]: [REQUEST]: id: "9664c6d0-5872-4c8c" sattr3 { size: 103079215104 atime_how: DONT_CHANGE atime { seconds: 0 nseconds: 0 } mtime_how: DONT_CHANGE mtime { seconds: 0 nseconds: 0 } }, [RESPONSE]: ST:OK, name: "zyx1-test-vm03_2-flat.vmdk" pool_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" id: "9664c6d0-5872-4c8c" parent_id: "60ca9357-9c5f-4818" attr { type: FILE mode: 384 uid: 0 gid: 0 size: 103079215104 ctime { seconds: 1702915039 nseconds: 893154384 } mtime { seconds: 1702915039 nseconds: 893154384 } atime { seconds: 1702915039 nseconds: 712888125 } } volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" xid: 1668748745, [TIME]: 21566 us.
/var/log/zbs/zbs-metad.INFO:15483:I1218 23:57:19.928862 31159 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" } }
/var/log/zbs/zbs-metad.INFO:15484:I1218 23:57:19.928915 31159 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" } }
/var/log/zbs/zbs-metad.INFO:15485:I1218 23:57:19.928941 31159 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" } }
/var/log/zbs/zbs-metad.INFO:15486:I1218 23:57:19.928961 31159 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" } }
/var/log/zbs/zbs-metad.INFO:15487:I1218 23:57:19.928982 31159 session.cc:246] Notify in PROCESS_KEEPALIVE state.[zbs.meta.AccessKeepAliveResponse.response] { revoke_cmd { vtable_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" } }
/var/log/zbs/zbs-metad.INFO:15488:I1218 23:57:19.967679 31136 meta_rpc_server.cc:4157] [RESERVE VOLUME SPACE]: [REQUEST]: volume_id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8", [RESPONSE]: ST:OK, name: "9664c6d0-5872-4c8c" size: 103079215104 created_time { seconds: 1702915039 nseconds: 712888125 } id: "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" parent_id: "057ac9c5-9c26-41ec-a5ad-643034ea7c3f" replica_num: 3 thin_provision: false iops_burst: 0 bps_burst: 0 throttling { } stripe_num: 4 stripe_size: 262144, [TIME]: 38922 us.
```

pid 粒度

```c++
[root@node2 23:58:01 sbin]$ grep -wn "37840" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.log.20231218-234522.21577:168621:I1218 23:48:01.539799 21584 access_io_handler.cc:1763] [GET TEMPORARY REPLICA GENERATION] pid: 1721 epoch: 1941 temporary_pid: 35987 temporary_epoch: 37840
/var/log/zbs/zbs-lsm2db.log.20231210-161333.12848:83793:2023/12/16-10:07:12.531906 13160 Delete type=0 #37840
/var/log/zbs/zbs-metad.INFO:11968:I1218 23:52:30.706779 31164 gc_manager.cc:44] [MARK GC PIDS (pid:epoch)] 35977:37830 35978:37831 35979:37832 35980:37833 35981:37834 35982:37835 35983:37836 35984:37837 35985:37838 35986:37839 35987:37840 35988:37841 35989:37842 35990:37843 35991:37844 35992:37845
/var/log/zbs/zbs-metad.INFO:15581:I1218 23:57:20.569731 31136 meta_rpc_server.cc:952] yiwu MetaRpcServer::GetLeaseForRead pid 37840
/var/log/zbs/zbs-metad.INFO:15582:I1218 23:57:20.569749 31136 meta_rpc_server.cc:1205] yiwu MetaRpcServer::DoGetLease pid 37840 preferred session 664f5c05-b622-4d3a-abdb-42c9c72c9087 local_cid 2
/var/log/zbs/zbs-metad.INFO:15583:I1218 23:57:20.569768 31136 access_manager.cc:586] yiwu AccessManager::AllocOwner pid 37840 cid 2 cid 0
/var/log/zbs/zbs-metad.INFO:15584:I1218 23:57:20.569814 31136 access_manager.cc:1919] yiwu GenerateLease pid 37840 ever exist 0 meta gen 0
```

cid1

```c++
[root@node1 23:52:41 ~]$ grep -wn "9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.INFO:279726:I1218 23:57:19.929340 17741 access_handler.cc:885] [REVOKE LEASE]: vtable: 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8

[root@node1 23:59:58 ~]$ grep -wn "37840" /var/log/zbs/zbs-*
/var/log/zbs/zbs-chunkd.log.20231218-235331.17726:280540:I1218 23:57:20.572149 17741 local_io_handler.cc:272] yiwu HandleGetGeneration pid 37840
/var/log/zbs/zbs-chunkd.log.20231218-235331.17726:280541:I1218 23:57:20.572185 17742 lsm.cc:1501] yiwu VerifyAndGetGeneration pid 37840
/var/log/zbs/zbs-chunkd.log.20231218-235331.17726:280542:I1218 23:57:20.572232 17742 lsm.cc:5135] [ALLOC EXTENT] status: EXTENT_STATUS_ALLOCATED pid: 37840 epoch: 39699 generation: 0 bucket_id: 976 einode_id: 4215102 thick_provision: true prior: false
```

volume

```c++
[root@node3 23:57:35 ~]$ zbs-meta volume show_by_id 9664c6d0-5872-4c8c-b68d-3f85fbb3c7c8 --show_pextents

...
    
37835  [3, 2, 6]  [3, 2, 6]        []                   False         False                        0                       3    39694               0              0  768.00 MiB
37836  [1, 3, 5]  [1, 3, 5]        []                   False         False                        0                       3    39695               0              0  768.00 MiB
37837  [2, 1, 6]  [2, 1, 6]        []                   False         False                        0                       3    39696               0              0  768.00 MiB
37838  [3, 2, 6]  [3, 2, 6]        []                   False         False                        0                       3    39697               0              2  768.00 MiB
37839  [1, 3, 5]  [1, 3, 5]        []                   False         False                        0                       3    39698               0              2  768.00 MiB
37840  [2, 1, 6]  [2, 1, 6]        []                   False         False                        0                       3    39699               0              2  768.00 MiB
```

