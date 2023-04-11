### 任务要求

> ZBS-20139：taskd: fix the speed of copy volume isn't as expected

在 dogfood 机器上 smartx@192.168.17.95｜smartx@2020，在 10.255.0.98 节点上，用一个有 1.5G 左右实际数据（存在于一个虚拟盘/volume 中）的虚拟机模版做了一下测试，完全拷贝模式下：

- 虚拟盘为 100G 时，用时 4 分 11 秒
- 虚拟盘为 200G 时，用时 7 分 56 秒
- 虚拟盘为 500G 时，用时 19 分 4 秒

需要修复 task center 中 copy volume 速度不及预期的问题。即当通过完全拷贝从模板创建虚拟机时，创建时间与虚拟机模板大小基本成正比关系，与 skip zero 功能相违背。

### 从中所得

* 在处理 bugfix 时，第一要义就是先稳定复现，能够稳定触发 error state 之后再加 log 判断。

* 用 grpc 是控制流数据，序列化/反序列化耗时长，而用裸 socket 做真正的数据 IO，不需要序列化/反序列化，性能更高。

* 单个 IO 的话，写不一定比读来的慢。LSM Tree、SSD 等结构。

* TaskCopyVolumeRunner::ReadData() 流程 

  ZbsClient::Read() --> OP::VOLUME_READ 类型的 ZbsClient::DoIO()

  1. 如果有 cache， IOCacheManager::Cache();
  2. 否则，ZbsClient::SplitIO()
     1. ExternalIOClient::SplitIO()
     2. ZbsClient::ProcessIO() --> ExtenrnalIOClient::SubmitIO()
        1. Data_channel_client_v2::ReadVolume() --> Data_channel_client_v2::DoRequest() --> Client_transport_manager::SendRequest()  --> Client_transport::SendRequest() -->  Socket_client_transport::TrySendRequest()  --> Socket_client_transport::DoSendRequestParts --> TcpSender::SendRequest()
        2. 

### 残余问题

1. DataChannel 传输 ENotAlloc 丢失，追踪一下传输过程

2. 怎么用 gdb 给正在运行的服务进程打断点

3. 他控制 CopyBlock Coroutine 的协程同步值得抽取出来学习，包括用协程之后就不用加锁。

4. 要学习一个 c++ 基本类可以看 status.h

5. 它就不是通过 controller 传 status 的，那如果有那些 ErrorRetryableStatus，该如何得知？

6. Internal IO Client 中通过 grpc 来做是会把 status 通过 controller 传递的。单测好像用的是 Internal IO Client ？没道理吧？

   InternalIOClient::SubmitIO()

   1. 如果有 access_handler，meta_->GetVExtentLease()
   2. 否则，借助 MetaClient，调用 MetaServer rpc 方法 CALL_ZBS(GetVExtentLease, ......)
      1. MetaRpcServer::GetVExtentLease()
         1. MetaRpcServer::GetLeaseForRead()
            1. MetaRpcServer::DoGetLease()
               1. AccessManager::AllocOwner()

7. 什么场景的 copy volume ，base_volume_id 不为 0 ？

8. Git review 的时候，忘记指定 zbs-20139，想要重新指定，该怎么操作？

9. 为什么要支持 nfs/iscsi 两种协议

10. 编译一个 C++ 项目

   1. 使用 cmake -G "Ninja" CMakeLists.txt_path 来生成 build.ninja
   2. 在 build.ninja 目录下执行 ninja/ ninja zbs_test 

### 解决思路

1. 先在自己的嵌套集群上稳定复现。在代码中加 log 查看 Copy Volume 中各部分时间占比（ReadData、WriteToDst 等），发现当通过完全拷贝从一个 2-100G 的模板创建虚拟机时，除了前 2 GB 需要真实拷贝之外，之后的 98G 在 CopyBlock 时没有触发 SKIP_EXTENT，而是 N 个 SKIP_BLOCK，这意味着虽然没有多余的写盘，但是读盘次数一次都没有省略。

   通过分析耗时，可以发现在上述条件下，前 8 个 Extent（2GB）有读有写耗时 8 * 1s = 8s，而后 392 个 Extent（392 GB）仅仅读耗时 392 * 0.15s = 58.8s。也就是说，其实耗时主要是 ReadData 占大头。因此当根据一个（2-200GB）的模板创建虚拟机，耗时是 8 * 1s + 792 * 0.15s = 127s 接近于前者 67 的两倍，符合我们看到的现象，耗时随模板大小接近线性增长。

2. 通过 zbs-task task list_by_status finished 可以看到 task cost time 和 src_volume id；通过 zbs-meta volume show zbs-iscsi-datastore-1629132402038s volume_id  --show_pextent 查看各个 extent 的 Unique Size 和 Shared Size，两者之和等于已分配的空间大小，发现 2-100G 和 2-200G 已分配的空间大小都是 2G，基本可以判断 meta/chunk 是没毛病的，但还是继续从 task center 中 skip extent 的判断逻辑处着手来实锤。

3. 由于 task center 中用的是 External IO Client，而它的 SubmitIO 方法是通过 DataChannelClientV2 的 ReadVolume 进行的，这个跟踪代码到最后是通过 tcp/rdma 下发的 read request，但是这整个过程中并没有设置 controller，所以导致每次 CopyVolumeRunner::ReadData 的 Status 都是默认的 OK。

   因为 task center 是在 IO 路径之外的服务，跟 Access 是两个进程，所以必须用 External IO Client，所以不走 grpc 那一套的话也就没办法设置 controller ，那我们接下来得想想怎么更改 skip extent 的判断逻辑。

4. 发现在 PrepareContext 中有通过 ShowVolumeById 获取到 src_volume 的 pextent 信息，所以可以根据它的返回值中 location 字段判断是否已分配副本，借此来判断 extent 是否被分配，借助 diff_extents 来跳过未分配的 extent 拷贝。

   ```
   55 I0906 10:11:57.664176 15696 task_copy_volume_service.cc:238] pid: 14189 location: 770 aliva_location: 770 expected_replica_num: 2 ever_exist: True preferred_cid: 2
   56 I0906 10:11:57.671435 15696 task_copy_volume_service.cc:238] pid: 14190 location: 770 aliva_location: 770 expected_replica_num: 2 ever_exist: True preferred_cid: 2
   63 I0906 10:11:57.680850 15696 task_copy_volume_service.cc:238] pid: 14197 location: 0 aliva_location: 0 expected_replica_num: 2 ever_exist: False preferred_cid: 0
   64 I0906 10:11:57.690454 15696 task_copy_volume_service.cc:238] pid: 14198 location: 0 aliva_location: 0 expected_replica_num: 2 ever_exist: False preferred_cid: 0
   ```

### 过程疑问

1. 增量备份：如果保护站点和目标站点曾经完成过一次备份任务，后续任务执行时，都会采用增量备份的方式进行。每次传输的数据仅是当前需要传输的快照组与目标站点已有快照组之间的增量，在数据更新不频繁或更新区域较为集中的场景下可以很大程度地减少传输量；

3. 用 zbs-task task show task-id 看 task 的各字段：

   * base_volume_id 是基准 volume，在 copy volume task 中没有用到，是默认值 0；
   * sync_speed_Bps 是扫描速率，代表完成一个 extent （不论会不会 skip）的速度，而且能看到的是最后一个 extent 的扫描速率，所以没啥参考性；
   * synced_volume_bytes 在最后一个 extent 时肯定等于 volume_size_bytes，所以没啥参考性；
   * net_speed_Bps 为啥两个都是 0，应该是因为最后一个 extent，两个都没有实际拷贝；
   * 两个 transferred_volume_bytes 都等于 940048384L（1.5G），所以真实需要传输的数据是一样的。transferred_volume_bytes 初始为 0，每做完一个 CopyExtent 后就 + bytes_transferred_in_extent。 bytes_transferred_in_extent 的计算方式如下：
     * 如果 BlockCopyState = SKIP_EXTENT，复制跳过整个 Extent，那么 bytes_transferred_in_extent 为 0，只会有 io_depth 个 ReadData 的开销；
     * 如果每个 copy block coroutine 都有 k 个 BlockCopyState = SKIP_BLOCK，复制跳过 k 个 Block，那么 bytes_transferred_in_extent = io_depth * (1024 - k) * kBlockSize（256KB），会有 io_depth * 1024 个 ReadData 的开销 + io_depth * (1024 - k)  个 WriteToDst 的开销；
     * 剩下的情况就是 full copy，也就是 bytes_transferred_in_extent = io_depth * kBlockSize，会有 io_depth * 1024 个 ReadData 的开销 + io_depth * 1024  个 WriteToDst 的开销。
   * preferred_cid 代表 preferred chunk id ，Extent 倾向的副本，通常为 Extent 当前访问者所在的 Chunk，Meta 进行副本分配时会尽量的靠近该 Chunk，默认为 0。这边是 12 和 9，在 WriteToDst 时用到这个参数，好像跟副本分配有关，这里会有影响吗？好像是没关系。

### 补充知识

Copy Volume Service 用于提供不同集群间存储对象（VM / Volume / File／LUN）的复制。由于数据拷贝耗时较长，需要先对源对象创建 Snapshot，后续复制这个 Snapshot。为减少数据传输流，CopyVolumeService 使用增量复制。

复制的任务包括如下 2 个阶段：

1. PrepareContext：复制前准备，初始化参数，根据 extent_id 的不同，整理出与基准 Extent 有差异的 Extent 用于复制。

2. CopyExtent：逐个 Extent 复制。对于每个 Extent，逐个遍历 Block，对比当前 Block 与基准 Block 的数据差异，只复制有差异的 Block。如果 Extent/Block 没有被真实分配（Thin Provision 时）或全零时，将快速跳过。


每个步骤执行完成时，Runner 均将向 ZK Task DB 更新当前步骤。在 Copy Extent 阶段，每完成一个 Extent 复制，均会更新状态。