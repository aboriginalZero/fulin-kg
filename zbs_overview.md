### ZBS 端到端 IO 链路

描述一个完整的从虚拟机到物理磁盘的 IO 链路，虚拟机执行 IO 时如何最终定位到应该访问哪个节点的哪个磁盘？

以 VMware ESXi  虚拟化平台为例，在一个分布式集群中，每台物理机上都装有 ESXi 作为虚拟机管理器（Hypervisor），在每台 ESXi 上，除了多台客户虚拟机之外还有一台运行 ZBS 的 SCVM，为了 ZBS 能够直接管理到虚拟磁盘，会将物理磁盘（SSD、HDD）直通给 SCVM。ESXi 用 33.2 作为 NFS Server 地址挂载 ZBS 对外提供的 NFS Export。

在客户虚拟机上的应用程序在读写文件的 IO 下发流程如下：

1. Common VM 上 APP 的 IO 请求，经过系统调用、内核处理下发到文件系统（如 Linux ext3）；
2. 文件系统将 IO 请求转换为对虚拟磁盘的 IO 请求并下发到 ESXi 虚拟设备驱动层；
3. ESXi 将 IO 请求转换为对 NFS Export 的 IO 请求（nfs_file, offset）并转发到 SCVM 的 NFS Server；
4. NFS Server 通过 ZbsClient 将 IO 请求转换成对 VExtent 的 IO 请求（volume_id, vextent_no）并转发到持有 VExtent Lease 的 Access Server；
5. Access Server 经过  Sync &Increase Generation  后，将 IO 请求转换为对 Extent 的 IO 请求（volume_id, extent_no）并转发到每个副本所在的 Chunk Server；
6. 各个 Chunk Server 以 Block 为粒度做本地磁盘读写。

虚拟机执行 IO 的时候根据 lease 找到副本所在节点，根据 volume_id，extent_no 找到具体数据块。

### ZBS 中 Meta、Access 作用

解析 zbs 中 Meta、Access 发挥的作用，即对上一节中的步骤 3 到步骤 5， IO 从接入点到转发给 LSM 之前的 IO 流程做进一步解析。

1. NFS3Server::write(nfs3_file, offset, count) 

2. ZbsClient::Write(volume_id, buf, len, offset)

3. ZbsClient::DoIO(volume_id, buf, len, offset)，通过 arena 构造带有 volume_ctx, volume_id, buf,  len, offset 的 uioctx，其中 volume_ctx = GetVolumeCtx(volume_id)。

4. 调用 ZbsClient::SplitIO(uioctx)，其中包括 2 个步骤。1. 调用 InternalIOClient::SplitIO(volume_ctx, uioctx) ，通过 arena 构造带有 uioctx, vextent_no, extent_offset 的 ioctx，其中 vextent_no 和 extent_offset 是根据条带化的策略算出来的。2. ZbsClient::ProcessIO(ioctx) 调用 InternalIOClient::SubmitIO(ioctx)

5. 调用 Meta::GetVExtentLease(volume_id, vextent_no)，如果本地 libmeta 有对应数据块的 vextent lease 则直接返回 lease 给 InternalIOClient，否则向 meta leader 请求 vextent lease 并缓存到本地。

6. 调用 InternalIOClient::IsLocal(ioctx->extent_lease->owner().uuid()) 判断本地 Access Session 是否是 vextent lease owner。

    1. 如果不是本地 Access，那么调用 InternalIOClient::DoRemoteIO(ioctx)，根据 vextent lease 的 IP + Port 通过 data channel manager 获取一个 dcc 下发 IO，当 IO 到达远程的 Access 后，调用 AccessIOHandler::SubmitWriteVExtent() 处理 IO

    2. 如果是本地 Access，那么调用 InternalIOClient::DoLocalIO(ioctx)，调用本地的 AccessIOHandler::WriteVExtent(volume_id, vextent_no, len, extent_offset, w_buf)

    至此，IO 从 ZbsClient 转发到 Access。
    
    > 由于缓存的 vextent lease 可能与 Meta Leader 处不一致，如果此时 ZBSClient  访问旧的 Lease Owner，Access 将返回一个 ENotOwner 错误，ZBSClient 收到该错误后会发 rpc 向 Meta Leader 获取最新的 Lease 并更新本地缓存，所以能够保证 ZBSClient 将 IO 从转发到正确的 Access
    
7. 调用 AccessIOHandler::GetVExtentLease(wctx) 获取 lease，因为如果是写有 COW 标记的 vextent，meta leader 在复制一个新的 vextent 之后，会通过心跳下发 Revoke Lease 指令要求 Access 释放 Lease，之后让 Access 主动去获取新的 vextent lease，这一步是为了保证拿到最新的 vextent lease 

8. 调用 AccessIOHandler::SyncGeneration(ioctx)，在 Access Session 成为 Lease owner 之后执行初次 IO 请求之前，会获取 Extent 所有有效副本的 Gen，最大的 Gen 会被视为有效的 Gen，如果这个 Gen 低于 Meta 认可的 Gen 则同步失败，数据进入不可 IO 的异常，否则那些没有在指定时间内响应的副本或者 Gen 低于有效 Gen 的副本将会被剔除。同步 Gen 之后，触发本地记录的 Gen + 1。

    > 为什么只需要初次 IO 时进行 SyncGeneration？
    >
    > 因为如果当前 extent 的 Lease Owner 不变，那么都要从这个 Access 处写入（唯一接入点），那么这个 Access 本地的 Gen 就永远是最新的，如果 Lease Owner 转移，那么在初次 IO 时又要执行 SyncGeneration 来保证从最新的（epoch 最大的）副本上开始写

9. 调用 AccessIOHandler::DoWriteVExtent(wctx)，根据 wctx->loc 遍历当前 extent 的每个副本，执行写操作，并通过 co_wait 来保证所有副本都写入才执行后续操作。也就是调用 AccessIOHandler::WriteVExtentDone() 统计写失败副本的个数、标记慢盘、调整失败副本上的 recover 行为、为写失败的副本创建敏捷恢复的 StagingBlockInfo

10. 调用 AccessIOHandler::WriteReplica(wctx, cid)，根据 cid 来判断是否是本地的 chunk。

    1. 如果不是本地 Chunk，调用 DataChannelClient::WritePExtent(pid, w_buf, len, extent_offset, gen, origin_pid)
    2. 如果是本地 Chunk，调用 LSMProxy::ScheduleWrite(pid, orgin_pid, len, extent_offset, gen, epoch, origin_epoch, w_buf) ，这个函数是一个典型的异步编程实现，通过回调机制而非显式的锁来实现线程同步。

    至此，IO 从 Access 转发到 Chunk。

在整个流程中，伴随着以下问题：

1. Access 的作用？

    Access Server 是 ZBS 的 IO 总控，接受不同的协议对象如 NFS/iSCSI Client 的 File/LUN 转义成 ZBS 内部的 Volume 与 Extent 对象。

    除了 IO 协议转换之外，Access 还通过心跳建立 Chunk 所在节点与 Meta 的联系，响应 Meta 下发的 Recover/Migrate Cmd 等工作。理论上 Access 能做的事 Chunk 也都能做，但为了简化 Chunk 的语义，在逻辑上抽象出 Access，不过还是同一个物理实体（跟 Chunk Server 在同一个进程）。

    从 zbs chunk 视角来看，有了 Access 它就不需要管跟 meta 的心跳，响应 Meta Cmd 等琐事，只需要处理本地读写，并且只认 volume extent，他才不管用户是用什么 IO 协议接入的。

1. 整个流程经历了多少次线程切换？

    从 ZbsClient 开始算起到 LSM::Write 之前，经历了 AccessHandler、LSM 两个线程。其中，ZbsClient 和 AccessHandler 在一个线程，而 LSM 内部做具体读写的时候其实有多个线程。

2. 如果是读取本地和读取远程数据参与的服务线程分别是谁？

    本地：AccessHandler、LSM。远程：dcc、dcs

3. 请求与包含的数据本身是如何在线程间安全的传递的？

    Access 线程和 LSM 线程之间通过回调机制，保证在任意时刻仅有一个线程处理数据，使 Access 必须等待 lsm 的所有 IO 全部完成才会返回，所以可以保证 access 和 lsm 的线程安全问题。

    ```c++
    void LSMProxy::ScheduleWrite(Controller* ctrl, IOStage* stage,
                                 const Callable& done,
                                 const lsm_args_t& args, char* buf,
                                 flags_t flags /* =? */) {
        auto thctx = ThreadContext::Self();
        auto done_closure = new CommonClosure(done, true);
    
        auto closure = new LambdaSelfDeleteClosure([=]() {
            auto co = Coroutine::Create([=]() {
                // run inside v2_ thread
                bool is_done = false;
                // 用来保证 lsm 先处理 write 然后才是 access 线程运行 done_closure
                LambdaClosure fast_done([=, &is_done]() {
                    is_done = true;
                    thctx->Sched(done_closure);
                });
    
                auto st = v2_->Write(stage, args.pid, args.epoch,
                                     args.generation, buf, args.len,
                                     args.block_no * kBlockSize + args.block_offset,
                                     flags,
                                     &fast_done);
    
                if (UNLIKELY(!is_done)) {
                    if (!st.ok()) ctrl->SetStatus(st);
                    fast_done.Run();
                } else {
                    if (UNLIKELY(!st.ok())) {
                        LOG(WARNING) << "[LSM PROXY]"
                                     << " write request is failed while response has"
                                     << " been sent: "
                                     << st.str();
                    }
                }
            });
    
            co->Enter();
        });
        v2_->scheduler()->Sched(closure);
    }
    ```

    AsyncBatchHandleQueueImpl 中的锁是用来保证入队出队是原子操作，跟这个线程安全没关系。

4. 整个流程有多少次内存复制？ 承载数据的内存是谁申请的又是谁释放的？

    到 LSM 之前，就复制一次到 arena。 zbs client 的 arena_pool ，释放的话需要看 anera 的实现

    ```c++
    // internal_io_client.cc
    static WriteIOBufArg SetupWriteIOBufArg(WriteIOBufArg origin_wbuf, Arena* arena, size_t data_len) {
        if (LIKELY(IsAligned(reinterpret_cast<uint64_t>(origin_wbuf.addr()), SSD_ALIGNMENT_SIZE))) {
            return origin_wbuf;
        }
        auto owner = arena->CreateObject<AlignedMemIOBufOwner>(data_len);
        memcpy(owner->addr(), origin_wbuf.addr(), data_len);
        return WriteIOBufArg(owner->addr(), owner);
    }
    ```

    anera 可以理解成一段连续的空间，一同申请，一同释放，看代码这个内存空间是精简制备按需申请的（会考虑内存对齐）

5. 对于异常，上面简介中简单拆分的 5 个主体模块是否都可以重试还是最好仅向上传递错误？为什么？

    1. NFS Server 不重试。留给 zbsclient 重试，否则在多难以保证语义一致性。（其实不确定，问问同事）
    2. zbsclient 重试。因为会 splitIO 成多个 IO ，可以减少重试次数。
    3. AccessIOClient 读重试读另外的副本，写的话不重试而是剔除失败的副本。
    4. DataChannelClient。因为在 zbsclient 中就会根据 IsLocal 分类 extent，所以 zbsclient 的重试就包含了非本地的 IO
    5. LSM 不重试。单机上没有网络问题，在本地写失败基本也就失败了。

### 对外接口

ZBS 对内在 Meta 上有 256 MB 的 Extent，Chunk 上有 256KB 的 Block，对外提供用户指定大小的 Volume。Volume 是面向用户的存储对象，对应一个 iSCSI LUN / NFS File；Pool 是一个 Volume 集合的逻辑概念，对应一个 iSCSI Target / NFS Export。

```c++
/* pool API */
int zbs_pool_create(zbs_t zbs, const char *name, zbs_pool_t *pool,
                    const char* storage_pool_id);
int zbs_pool_delete(zbs_t zbs, const char *name);
int zbs_pool_list(zbs_t zbs, zbs_pool_t **pools);

/* volume API */
int zbs_volume_create(zbs_t zbs, const char* pool_name, const char* volume_name,
                      uint64_t size, uint32_t replica_num,
                      int thin_provision,
                      zbs_volume_t *volume);
int zbs_volume_delete(zbs_t zbs, const char* volume_id);
int zbs_volume_list(zbs_t zbs, const char* pool_id, zbs_volume_t **volumes);
int zbs_volume_clone_by_id(zbs_t zbs, const char* volume_id,
                         const char *dst_pool_name, const char *dst_volume_name);
int zbs_volume_truncate(zbs_t zbs, const char* volume_id, uint64_t size);

/* SnapShot API */
int zbs_snapshot_create_by_volume_id(zbs_t zbs, const char* volume_id,
                                     const char* snapshot_name,
                                     const char* snapshot_desc,
                                     zbs_snapshot_t* zbs_snapshot);
int zbs_snapshot_delete_by_id(zbs_t zbs, const char* snapshot_id);
int zbs_snapshot_list_by_pool_name(zbs_t zbs, const char *pool_name, zbs_snapshot_t **snapshots);
int zbs_snapshot_list_by_volume_id(zbs_t zbs, const char *volume_id, zbs_snapshot_t **snapshots);

/* IO API */
int zbs_read(zbs_t zbs, const char* volume_id, char* buf, uint32_t len,
                uint64_t offset, uint32_t flags);
int zbs_write(zbs_t zbs, const char* volume_id, char* buf, uint32_t len,
                 uint64_t offset, uint32_t flags = IOFlags_NO_STRIPE);

// Clients, e.g. qemu, could use asynchronous API
void zbs_read_async(zbs_t zbs, const char* volume_id, char* buf,
                    uint32_t len, uint64_t offset,
                    zbs_cb_t cb, void* data, uint32_t flags);
void zbs_write_async(zbs_t zbs, const char* volume_id, char* buf,
                     uint32_t len, uint64_t offset,
                     zbs_cb_t cb, void* data, uint32_t flags);

/* get cluster summary */
int zbs_get_cluster_summary(zbs_t zbs, zbs_cluster_summary_t *summary);
```

### Lease 机制

ZBS 中读写共用一个 Lease，iSCSI/NFS Client 会把 IO 请求转发到持有 Lease 的 Access 上，以此来保证 IO 请求是串行的，都是以相同的 IO 请求顺序去写多个副本，副本数据是一致的。

由应用层通过读写锁等机制来避免出现两个 ZBS Client 同时写同一个文件的情况，否则会出现数据错乱。

介绍

Lease 机制是一种维护分布式系统数据一致性的常用工具。 具有以下 3 个特点：

1. Lease 是颁发者对一段时间内数据一致性的承诺；
2. 颁发者发出 Lease 后，不管是否被接收，只要 Lease 不过期，颁发者都会按照协议遵守承诺；
3. Lease 的持有者只能在 Lease 的有效期内使用承诺，一旦 Lease 超时，持有者需要放弃执行，重新申请Lease。

背景

缓存的使用提高了查询的效率和服务器的抗压能力，但也带来了数据一致性的问题。当有更改请求时，服务器修改了数据，但是缓存却还没来得及修改，就带来了数据一致性的问题。Lease 机制保证缓存的一致性，服务器发出 Lease 后，会保证在这个 Lease 的租期内，数据不会发生改变，客户端可以放心地在这段时间内使用数据，因为这个时间段内数据是和服务器上的数据是一致的。

存在问题及解决方法

1. 服务器修改元数据时，需要阻塞所有的读请求，此时服务器不能发出新的Lease。以防止新发出的Lease保证的数据与服务器刚才修改的数据不一致。

    解决方法：读请求到来时直接返回数据，不颁发 Lease。

2. 服务器需要等待直至所有的 Client 的 Lease 都过期后，才颁发新“修改”后的 Lease。因此，此时服务器上的数据修改了，生成了一个新的 Lease 版本，需要等到 Client 上所有的老 Lease 过期后，该新 Lease 版本才能颁布给 Client。

    解决方法：服务器主动通知持久 Lease 的 Client 放弃当前的 Lease，并请求新 Lease。

[双主问题](https://zhuanlan.zhihu.com/p/140260158)

### 数据生命周期

存储的数据生命周期管理是元数据管理的基本功能。在这个部分，可以按照以下过程理解数据块的分配与回收：

1. Thin Volume Create。/meta/meta_rpc_server.cc 中的 MetaRpcServer::CreateVolume；

2. Extent Replica Alloc in Meta。结合 IO 过程中 ZBS Client 的 lease 获取逻辑阅读 /meta/meta_rpc_server.cc 中 MetaRpcServer::GetVExtentLease；

3. Extent Replica Alloc in Chunk。结合 IO 过程中 Access 的 Sync Generation 逻辑阅读 /lsm/lsm2/lsm.cc 中的 LSM::VerifyAndGetGeneration；

4. Volume Delete。/meta/meta_rpc_server.cc 中的 MetaRpcServer::DeleteVolume；

    1. 获取 vextent table

    2. 如果没有 COW，可以被直接删除，pextent_table->MarkGarbage(to_pextent_id(vtable[i]))
    3. 无法被直接删除的 vextent
        1. meta_db_.DeleteVolume(volume)
        2. RevokeVTableLease(volume.id())
        3. volume_manager->Remove(volume.id())
        4. recover_manager->UnmarkVolumeAllocEven(volume.id())

5. Extent Recycle in Meta。/meta/gc_manager.cc 中的 GcManager::DoScan；

    1. 在 Meta DB 中删除 Extent 记录。meta_db_.DeletePExtents(pid_to_sweep)
    2. 释放 pid 关联的 lease。access_manager->ReleaseOwner(pid_to_sweep)，
    3. 在 PExtent Table 中清理相关 Extent 记录。pextent_table->DeletePExtents(pid_to_sweep)

6. Extent Recycle in Chunk。结合 Access 的 DataReport 过程阅读 /meta/access_manager.cc 中的 AccessManager::HandleChunkDataReportRequest 和 /access/access_handler.cc 中的     AccessHandler::HandleChunkResponse

    1. Meta 在心跳中发现 Chunk 上报了不该持有的 extent 信息。对应代码 AccessManager::HandleChunkDataReportRequest
    2. 向 chunk 下发 GC 命令。AccessManager::GenerateGcCmd，在这个过程中只回收没有孩子的 extent 
    3. Chunk 收到命令之后在本地清理 extent 回收物理空间。

在完成阅读之后尝试回答如下问题：

1. 整个流程的回收为什么不是即时的而需要是一个异步过程？

    GC 相较于正常 IO、Recover/migrate 等是一个优先级较低且允许分段操作的集群内部活动。其不仅涉及到 meta 更改、Lease 清除、PExtent Table 清理等 Meta 上的操作，还要在 Chunk 中回收实际的物理空间，IO 链路过长，重试成本过高。

2. 回收过程中是否不再可以有新的数据分配还是其实没有影响？如何保证正确的回收数据，而没有错误地将回收过程中产生的新数据标记为需要回收？

    回收过程中可以有新的数据分配。考虑到 GC 扫描过程耗时较长，不应该长时间锁住 VTable（存在 Volume -> Extent  映射关系），在 GC 时会为 Meta DB 中的 VTable 创建 DB SnapShot，即使 GC 根据 DB Snapshot 扫描得出的无效 pid 被重新分配，也会具有不同的 Epoch，不会错误地回收新分配的 extent。对应代码 AccessManager::HandleChunkDataReportRequest 中。且在 LSM 的回收中，不仅会用到 pid 还有 epoch，对应代码 AccessHandler::HandleChunkResponse

3. 对于快照&克隆产生的链式数据，对父 Extent 的回收是否会导致子 Extent 的数据不完整？

    不会，如果是有快照&克隆产生的链式数据，不会回收这个父 extent，只会在 MetaDB 中删除父 extent 对应的 vextent table，不需要更改子 extent 对应的 vextent table entry 上的 cow 标志位，因为还有可能有祖先 extent 也指着这个 chunk 中的那个 extent。对应代码 AccessManager::GenerateGcCmd

    但是为什么快照&克隆产生的链式数据的父 extent 会被当成垃圾？这个题目成立吗？

4. 数据块什么时候移动？

    节点上下线等各种事件的影响，集群中有可能出现数据分布不符合预定目标的现象，Recover Manager 在定期循环里执行 rebalance/migrate 策略来改善数据分布。

5. Extent 的生命周期是否与 Volume 在一起？克隆和快照对此是否有影响？

    extent 与 volume 生命周期不同步，取决于创建时是否设定 volume 的 thin_provision 属性。如果创建时制定精简配置（thin）的话，在没有发生写操作时只分配 pid，只有写的时候才会为 extent 申请内存空间；如果指定厚备配置（thick）的话，在卷创建的时候就会为 extent 分配空间。

    克隆和快照也会导致 extent 与 volume 生命周期不同步。由于 zbs 中的克隆/快照操作时刻遵循写时复制（COW），所以当克隆一个 volume 的 extent 时，不会立即复制出一个 extent。对 volume A 快照生成的 snapshot 也是一个 volume B，也需要 vtable，这个 vtable 是从 volume A 复制而来，并对 volume A B 的每一个 PExtentTableEntry 设置 cow 标志为 1。对应代码 MetaRpcServer::DoCreateSnapshot  中的 meta_db_.CreateSnapshotOrClone(vtable, vtable)

### Generation 相关

generation 代表 extent 副本的写入次数，不论是 ReadVExtent 还是 WriteVExtent，在 Access 将 IO 下发到 LSM 之前，都需要 SyncGeneration，以下仅分析 WriteVExent 的 SyncGeneration 过程。

1. 调用 AccessIOHandler::WriteVExtent(volume_id, vextent_no, len, extent_offset, w_buf)，通过 AccessIOHandler::GetVExtentLease 获取 vextent lease 之后，继续以下操作。

2. 调用 AccessIOHandler::SyncGeneration(wctx)，根据 wctx->lease->gen() 来判断是不是当前这个 vextent lease owner 的第一个 IO 请求。如果不等于 -1，说明已经同步过了，直接返回同步成功。如果等于 -1，需要触发同步机制，调用 GenerationSyncor::Sync(wctx->lease)，这个函数重点在建立 Sync Generation 的重试机制，具体执行在下一步。

3. 调用 GenerationSyncor::DoSync(wctx->lease)，根据 wctx->lease 得到每个副本的位置，构造 ReplocaCtx replicas 数组（其中每个 replicas 的 synced_gen 的初始值都是 0）。

4. 调用 GenerationSyncor::GetGenerations(pid,*wctx->lease, replicas, replicas_num)，对于每个 replica，通过 dcc 调用 DataChannelClient::Impl::GetGeneration(pid, gen_buf, lease.epoch, ....)  获取持有该副本的 chunk 上保存的该副本的 generation，具体而言，access 不经过 meta leader 而是借助 dcc 向拥有该副本的各个 chunk server 直接获取 extent 的 generation。各个 chunk server 有成员变量 LocalIOHandler，它针对 MessageHeader::GET_GENERATION 注册了 LocalIOHandler::HandleGetGeneration(creq)  回调函数，它会调用 LSMProxy::VerifyAndGetGeneration(args.pid,  args.epoch, timer_ctx->gen_buf)，紧接着调用 LSM::VerifyAndGetGeneration(args.pid, args.epoch, timer_ctx->gen_buf) ，在这个函数内部，通过 LSM::GetExtent(args.pid) 拿到指定 extent，当满足 args.epoch 等于 extent.epoch 并且这个 extent 处于 PEXTENT_STATUS_ALLOCATED 的状态时，gen_buf 会被赋值成 extent->generation()。由于同步回调，又会将 gen_buf 赋值给各个 replica 的 synced_gen 字段。

5. 调用 GenerationSyncor::ParseGenerations(pid, wctx->lease, replicas, replicas_num)，从每个不等于 -1 的 replica[i].synced_gen 中找到一个 max_generation，要保证它大于等于 wctx->lease->meta_generation() 。只有 synced_gen等于 max_generation 的 replica[i] 才是有效副本，其余的副本都要被剔除，之后更新 wtcx->lease 的 genmap_[pid] 为 max_generation、location 为剔除副本后的 location。

    其中，副本剔除操作由 Meta::RemovePExtentReplica(pid, failed_cids, gen = max_generation) 执行，具体分为 libmeta 和 meta leader 两部分操作：

    1. libmeta。调用 Meta::RemovePExtentReplica(pid, failed_cids, max_generation)  简单封装一个 request，通过 rpc 转发到 meta leader 的 MetaRpcServer::RemoveReplica(request) 执行。
    2. meta leader。调用 MetaRpcServer::DoRemoveReplica(request)，分别处理这个副本在 PExtentTable 和 MetaDb 中的记录：
        1. 调用 PExtentTable::RemoveReplica(info, response) ，删除 chunk_table 中指定 cid 指定 pid 的 ChunkTableEntry。然后更新这个 pid 的 PExtentTableEntry 的 generation() 为 max_generation()，将更新后的 PExtentTableEntry 更新到 response。
        2. 调用 MetaDb::PutPExtent(*response)，把经过上一步更新后的 extent 信息持久化到 MetaDB。
        3. 往 recover manager 下发一条这个 pid 的 recover cmd。
    3. libmeta。如果 meta leader 操作成功，libmeta 根据 reponse 中的 location 来设置 CachedLeasePtr->location，并用 max_generation 更新 CachedLeasePtr 的 meta_generation()

经过 Sync Generation 之后，调用 AccessIOHandler::SetupIOCtx(wctx) 让 wctx->gen 等于 wctx->lease->gen()， 接着让 wctx->lease->gen() +1，进入 AccessIOHandler::DoWriteVExtent(wctx)，在 IO 下发到各台 LSM 并写完，会调用 AccessIOHandler::WriteVExtentDone(wctx)，具体过程如下：

1. 调用 AccessIOHandler::WriteVExtentDone(wctx)，根据写失败的不同情况，下发取消 recover 的 cmd，并对在 wctx->lease->location 中的第一个 failed_cid 创建一个 staging_block_info，然后执行下一步。
2. 调用 Meta::RemovePExtentReplica(pid, failed_cids, gen = wctx->gen + 1)，副本剔除在上文描述过。
3. 调用 AccessIOHandler::UpdateStagingBlockInfo(wctx, failed_cids)，涉及到敏捷恢复，暂时不管。
4. 如果 (wctx->lease->meta_generation() == 0 || !wctx->lease->ever_exist())，调用 Meta::SetPExtentExistence(pid, wctx->lease->epoch(), gen = wctx->gen + 1)，否则直接返回 OK。
    1. libmeta。如果 lease && lease->meta_generation() > 0 && lease->ever_exist() ，直接返回 OK，否则根据 pid, epoch, gen 封装一个 request，通过 rpc 转发到 meta leader 的 MetaRpcServer::SetPExtentExistence(request) 执行
    2. meta leader。调用 MetaRpcServer::SetPExtentExistence(request) ，分别处理这个副本在 PExtentTable 和 MetaDb 中的记录：
        1. 调用 PExtentTable::SetPExtentExistence(pid, epoch, gen)，如果 gen 等于 0 说明要设置的 pextent 不存在，新建一个 generation = gen、ever_exist = false 的 PExtentTableEntry，如果 gen != 0 说明要设置的 pextent 已经存在了，那么把它对应的 PExtentTableEntry 的 ever_exist 设置为  true，并更新 generation = gen。
        2. 调用 MetaDb::PutPExtent(pextent)，把经过上一步更新后的 extent 信息持久化到 MetaDB。
    3. Libmeta。如果 meta leader 操作成功，libmeta 更新 CachedLeasePtr 的 ever_exist() 为 true，并用 gen 更新 CachedLeasePtr 的 meta_generation()

关于 generation 总结补充：

* 对 meta_generation 的更新只在两个地方，Meta::RemovePExtentReplica 和 Meta::SetPExtentExistence
* meta leader 接受各个 chunk server 心跳携带的 ChunkData 之后，只会去比较 info.generation 与 PExtentTableEntry 的 generation 的关系，如果前者小于后者，会打一条 INFO 级别的日志，如果前者大于后者，也并没有用前者更新后者，原因是不同 chunk server 心跳上报是异步的，对 generation 的更新存在滞后与副本间不一致的可能，所以不通过心跳来更新 PExtentTableEntry 中的 generation。相关代码在 AccessManager::HandleChunkDataReportRequest() --> PExtentTable::HandlePExtentInfo() --> PExtentTableEntry::UpdateReplicaInfo()
* 请求发起方在首次 IO 时都会执行 sync generation，其中会跟 meta_generation 比较，而每个副本变更的操作（剔除、移动、添加副本）都会去更新 meta_generation，所以如果副本变更，在 sync generation 阶段就会发现，并把不一致的副本剔除。
* 什么时候该判断 wctx->lease->expired()？好像是当会牵扯到 RemovePExtentReplica ？

### 相关问题

1. 快照/克隆时，要收回 lease，但快照也不更改数据，那支持读会有什么问题吗？如果一个 volume 快照后 write ，他到底在哪改了 cow = false？

2. 三副本集群，如果两个副本坏了被剔除，只剩下一个副本，然后这个副本还失败了，重试还失败，返回给客户失败之后，客户可能会怎么做？zbs 会有回滚吗？有的话，什么时候回滚？块磁盘为什么可以接受写失败数据处于未知状态？

    就返回给用户失败，因为 zbs 只能提供节点间的副本备份，如果多个节点上的副本都失败了，说明连本地的副本都有问题，那应该是磁盘都坏了，而这是 zbs 无法防范的。如果用户不能接受这种级别的失败，可以做磁盘 raid，从磁盘粒度提供容错性。Zbs 没有回滚，每次写操作只存了 meta log。

3. 为什么要在 extent 上引入 epoch 字段？

    在 chunk server 上插入已经存在数据的 HDD 磁盘时，需要鉴别磁盘上已有的 pextent 数据是否有效。目前使用的是会被循环利用的 pid （32位）标识 pextent。如果插入磁盘存在的部分老旧 pextent 的 pid 恰巧与现在正在使用的 pid 一致，就有可能发生老旧数据被当成有效数据处理的情况。因此引入 epoch，epoch 其实可以替换 pid 的，但是为了平滑升级吧，保留 pid 字段。

4. ZbsClient 为啥不跟 access 耦合起来？ZbsClient 负责将 NFS 等协议转换成内部的数据访问形式，那把这部分功能做到 access 里面会有什么问题？

    其实是耦合的，access handler 持有 zbs client 对象，两者都在一个线程里。实际上，chunk server、access handler、zbs client 共同复用 Chunk 进程中的 main_thctx 线程。

    存储协议层整体工作在 zbs 数据层外部不直接与内部 IO 逻辑交互，但 NFSZbsClientProxy、ISCSIMegaServer 也会拥有跟 access handler 持有的同一个 zbs client 对象，用来把协议对象如 File、LUN 转换为 Volume 后，通过 zbs client 以 Volume 作为对象执行 IO。

    Access 最重要的功能是完成 zbs 内部的 IO 组织，以 extent 为 IO 对象，在内部完成 extent 的多副本同步，并配合 meta 完成副本的再平衡与恢复工作。

5. Sync Generation 为啥在 Access 上做，而不是 meta leader 那边直接做了，还能减少 IO 路径？

    chunk server 向 meta leader 保持心跳的时候会上报 ChunkData，但由于不同 chunk server 心跳上报是异步的，对 generation 的更新存在滞后与副本间不一致的可能，所以不通过心跳来更新 PExtentTableEntry 中的 generation，让作为请求发起方的 Access 主动去 Sync Generation 通过 dcc 向其他 chunk server 索要 extent 的各个副本的 generation。

6. 在 nvmf 中连接 subsystem，挂载 ns，且在 lsblk 中可以看到暴露的 ns 对应的 disk。而 NFS 挂载的是 export ，在 lsblk 中无法看到挂载情况，用 zbs-meta volume list < pool-name > 发现它是一个 pool ？从 nvmf 类比过来，不应该是个 volume 吗？

    nfs 是文件系统而非块设备，不能用 lsblk 看，用 df -h。用户创建磁盘用 iscsi/nvmf，这时就可以看到。 

7. 为什么要两次 GetVExtentLease，一次是 libmeta 判断 extent lease 是否在本地，一次是 AccessIOHandler 使用其中的 loc、generation、cid 等字段？

    【第一次获取 Lease】Meta::GetVExtentLease 根据 volume_id 和 vextent_no 向 libmeta 请求对应的 extent lease，如果 libmeta 中有缓存，那么直接将 lease 信息返回给 io client，否则向 meta leader 请求 lease 并缓存到本地 libmeta。io client 拿到的 lease 只通过里面的 uuid 判断 lease 是否在本地，其他信息没有被利用。

    【第二次获取 Lease】调用 AccessIOHandler::GetVExtentLease(wctx) 去再一次获取 lease 是因为如果是写有 COW 标记的 vextent，meta leader 在复制一个新的 vextent 之后，会通过心跳下发 Revoke Lease 指令要求 Access 释放 Lease，之后让 Access 主动去获取新的 vextent lease，这一步是为了保证拿到的是最新的 vextent lease。 

8. 怎么保证集群中任意时刻都只会有一个 Meta Leader？

    Quorum Cluster 列出 zk 中 service 目录下的所有节点，选取节点中最早的创建者作为 leader，并通过 zk 接受处理 zk 传递的节点成员变化事件即可正常获知自身的角色。当一个 Leader 不论任何原因与 zk 失去连接都会导致它在 zk 中对应的临时节点被清理，新的最小 Sequence 节点会成为新的 leader。即使原有的 leader 节点再与 zk 恢复连接时创建的也是一个新的临时节点，Sequence 序列排在末尾，不会触发 leader 角色回归

9. ExternalIOClient::SplitIO 其实就是条带化，如果不开启条带化，写 16K 的数据是在 1 块 extent 上写 16K。如果让 stripe_num = 4，tripe_size = 4k，写 16K 的数据，就会到需要到 4 块 extent 上各写 4K，磁盘可以并行写。

10. vextent_no 在 splitIO 中计算得到，volume_id 是用户传入的

11. 看起来 vtable 好像也是从 ptable 中复制来的？MetaRpcServer::GetVTable

12. LSM 的底层写还是调用 memcpy / pmem_memcpy，所以还是要借助 OS 调用磁盘驱动程序来具体落盘？AIO Engine

### 备份快照

#### 应用场景

1. 容灾备份。平时定期创建快照，执行重要操作前创建一份快照以便回滚；
2. 环境复制。使用系统盘快照创建自定义镜像，再使用自定义镜像创建虚拟机，实现环境复制；

不同于基于 redo-log 文件和快照链结构的 VMware 快照技术，SMTX OS 快照拥有独立的元数据，避免遍历快照；快照元数据除了利用内存加速，同时做了持久化存储；使用更大的数据块进行存储，有效规避了多种影响快照性能的因素，降低时延、提升快照性能可恢复性。

定期快照的设置方式如下，基本上涵盖了误删文件的时间范围：

1. 每 1 小时快照：保留 1 周；
2. 每周周日快照：保留 1 月；
3. 每 1 月 1 日快照：保留 1 年；

#### 快照级别

1. 崩溃一致性快照（默认级别）

    崩溃一致性快照仅记录已写入虚拟硬盘的数据。快照中不会捕获内存或待处理 I/O 操作中的任何数据。因此，此类型的快照无法保证文件系统或应用程序的一致性，您可能无法还原具有崩溃一致性快照的虚拟机；

2. 文件系统一致性快照

    除了虚拟硬盘上的数据之外，文件系统一致性快照还会记录内存和待处理 I/O 操作中的所有数据。在生成文件系统一致性快照之前，用户 VM 中的文件系统会进入静默状态，内存中的所有文件系统缓存数据和待处理 I/O 操作都会刷新到硬盘之后再生成快照；

3. 应用一致性快照（内存快照）

    内存快照是指虚拟机执行快照时，除了对硬盘数据执行快照之外，虚拟机会进入静默状态，内存也会同时执行快照，并持久化保存内存数据；当执行虚拟机快照恢复时，可加载内存快照数据。

#### 快照实现

##### 写时复制 COW 快照

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208222353008.png" alt="image-20220822235334244" style="zoom: 33%;" />

1. 在创建快照时，会同时创建快照卷和快照数据指针表（元数据），快照卷只需要很少的存储空间用于存储跟源数据不同的数据，快照数据指针表记录指向相应源数据块的地址指针，在不做任何修改时，此时快照卷与源数据卷通过各自的指针表共享同一份物理数据；
2. 更改数据时，COW 在原始数据修改之前进行拷贝到快照卷中，然后将新数据写入到源数据块中覆盖原始数据，并且将原始数据在快照卷中的新地址更新到快照数据指针表记录中，使快照时间点后更新的数据不会出现在快照卷中，插入新数据自然是不会对快照卷有影响；
3. 再次创建快照，会再次拷贝源数据指针表，新的修改会影响到旧的快照卷和新的快照卷。

优点：原始卷物理块连续，没有碎片；源数据指针表的指向永远不会变化；如果有多次快照，那么对源数据的修改会有多次写操作，导致读写时延大；

缺点：降低源数据卷的写性能，每首次更新数据，至少进行两次写操作（把元数据拷贝一份到快照卷中，然后修改原数据）。

##### 重定向写 ROW 快照

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208282322725.png" alt="image-20220828232208937" style="zoom:70%;" />

1. 在创建快照时，会同时创建快照卷和快照数据指针表（元数据），此时快照卷与源数据卷通过各自的指针表共享同一份物理数据；

2. 更改数据时，ROW 直接写到快照卷中，然后更新源数据指针表的记录指向新数据所在的快照卷的地址；

3. 再次创建快照，会再次拷贝源数据指针表，新的修改会影响到新的快照卷。


ROW 方式下，源数据指针表上有上次快照的修改和新增数据，所以显然快照之间的关系是链式，恢复后面的快照需要源数据以及全面的快照作为基础。那么当需要删除快照时，需要根据数据块的引用计数来判断哪些数据块可以被真正删除，比较费时。

优点：写性能没有损耗，只是修改源数据指针表的指针；

缺点：没有一个个完整的快照卷，而是维护了一个快照链，如果快照层级过多，进行快照恢复/删除时的系统开销会比较大；

##### 对比

|                  | COW              | ROW              |
| ---------------- | ---------------- | ---------------- |
| 快照卷存放内容   | 原始数据         | 新数据           |
| 原始数据组织方式 | 连续             | 离散             |
| 适用场景         | 对原数据读多写少 | 对原数据写多读少 |

VMware vSphere 和阿里云 ECS 快照都是 ROW 模式，SMTX OS 是 COW 模式。

<img src="https://mmbiz.qpic.cn/mmbiz_jpg/NPFGvdyPsl6Lk9soPje7VS3sulYIvEBL7AwVSib8uXia3PFdB2GwkHEicnLQibNWmw1Jz5uMHxWdUltjcGG58KIaoA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1" alt="图片" style="zoom:50%;" />

#### ZBS 做法

SMTX OS 中虚拟机快照支持通过某个时刻的快照实现重建（克隆）虚拟机的操作。其采用 COW 的快照方式，同一个虚拟机的多个快照分别拥有自己独立的元数据如所有数据块的物理位置信息，该元数据保存在 zbs metaDB 中，持久化在 SSD 且常驻内存。

![图片](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208282343833.jpeg)

当写入 I/O 小于 256KB，例如需要写入 4KB 数据，那么需先从原 extent C 上读取对应 block 的数据，修改数据后，将数据最终写入新创建 extent C' 上对应的 block。当对齐写入 256K I/O 时，则无需读取原 vDisk 上 block，直接写入新位置。

快照容量大于文件系统内看到的数据量的可能原因？

1. 文件系统的元数据会占用磁盘空间；
2. 文件系统为了降低性能消耗，删除文件时只在文件属性中创建弃用标记。磁盘无法感知删除指令，数据块仍然是已分配状态，同时数据块会被拷贝到快照中导致快照容量大于文件系统。

## ZBS 实现

分布式存储分为分布式文件存储、分布式对象存储、分布式块存储。其中分布式块存储主要应用于

* 虚拟化：支撑虚拟机中的虚拟盘的存储
* 数据库：数据库的数据盘运行在一个共享的块存储服务上或者直接把数据库运行在虚拟机上
* 容器：容器的数据持久化，k8s 的 persistent volume 功能

分布式块存储包括三个方面：

### 元数据服务

> ZK 中存元数据操作日志（常驻内存），LevelDB 中存元数据（）
>
> ZBS 元数据规模不大，应该也是元数据全内存放置，写操作要写内存 + LevelDB，读操作只读内存

包括集群成员管理、数据寻址、副本分配、负载均衡、心跳、垃圾回收，要求：

* 可靠性：元数据必须是保存多份的，同时元数据服务还需要提供 Failover 的能力
* 高性能：数据分配等 IO 请求还是需要先访问元数据服务，为避免给实际 IO 造成瓶颈
* 轻量级：私有云场景便于运维（HDFS 的元数据服务 Namenode 就是一个重量级的模块，对硬件要求高且升级、主备切换麻烦）

实现：levelDB + zk。尽管 levelDB 的数据只能保存在单机上，无法提供可靠的数据保护和故障恢复能力，以及 zk 所有数据都缓存在内存中，能够存储的数据容量有限。但从轻量级、稳定性的角度考虑，还是选择用 levelDB + zk 的方式，并用 Log Replication 机制来更好的发挥它们的性能。

![image-20220821092450967](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210924012.png)

当 Leader 节点接收到元数据的更新操作后，会将这个操作序列化成一组操作日志，并将这组日志写入 Zookeeper。由于 Zookeeper 是多副本的，所以一旦 Log 数据写入 Zookeeper，也就意味着 Log 数据是安全的了。同时这个过程也完成了对 Log 的复制。

当日志提交成功后，Meta Server 就可以将对元数据的修改同时提交到本地的 LevelDB 中。这里 LevelDB 中存储的是一份全量的数据，而不需要以 Log 的形式存储。

对于非 Leader 的 Meta Server 节点，会异步的从 Zookeeper 中拉取 Log，并将通过反序列化，将 Log 转换成对元数据的操作，再将这些修改操作提交到本地的 LevelDB 中。这样就能保证每一个 Meta Server 都可以保存一个完整的元数据。

同时对 Zookeeper 中的 Log 定期执行清理。只要 Log 已经被所有的 Meta Server 同步完， Zookeeper 中保存的 Log 就可以被删除了，以节省空间。通常我们在 Zookeeper 上只保存 1GB 的 Log，已经足够支撑元数据服务。

### 数据存储引擎

包括单机上的数据存储、本地磁盘管理、磁盘故障处理，要求：

* 可靠性：数据丢失是绝对不允许接受的
* 高性能：用最少的 CPU 指令完成一次 IO 操作，因为目前最快的存储已经可以做到单次访问只需要 10 纳秒。而如果程序中加一次锁，做一次上下文切换，可能几百个纳秒就过去了。
* 易维护：如果发现问题，可以快速定位和修复

![image-20220821092532366](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210925414.png)![image-20220821092553485](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210925535.png)![image-20220821092612341](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210926388.png)

抛弃 LSM Tree 和 FileSystem 的原因：

* 尽管 LSM Tree 容易实现，且对小块数据的读写性能非常好，但是由于读写放大问题，还是不考虑使用。

* 首先，文件系统所提供的功能远远多于存储引擎的需求。例如文件系统提供的 ACL 功能，Attribute 功能，多级目录树功能，这些功能对于一个专用的存储引擎来说，都是不需要的。这些额外的功能经常会产生一些 Performance Overhead，尤其是一些全局锁，对性能影响非常严重。
* 其次，大部分文件系统在设计的时候，都是面向单一磁盘的设计方式，而不是面向多块磁盘的。而一般存储服务器上都会部署 10 块，甚至更多的磁盘，而且有可能是 SSD，有可能是 HDD，也可能是混合部署。
* 第三，很多文件系统在异步 IO 上支持的并不好，尽管支持异步 IO 的接口，但实际使用过程中，偶尔还是会有阻塞的情况发生，这也是文件系统里一个非常不好的地方。
* 最后一个问题，文件系统为了保证数据和元数据的一致性，也会有 Journaling 的设计。但这些 Journaling 也会引入写放大的问题。如果服务器上挂载了多个文件系统，单个文件系统的 Journaling 也无法做到跨文件系统的原子性。

zbs chunk 的实现中 IO Scheduler 负责接收上层发下来的 IO 请求，构建成一个 Transaction，并提交给指定的 IO Worker。IO Worker 负责执行这个 Transaction。Journal 模块负责将 Transaction 持久化到磁盘上，并负责 Journal 的回收。Performance Tier 和 Capacity Tire 分别负责管理磁盘上的空闲空间，以及把数据持久化到对应的磁盘上。

但这种方式最大的问题还是性能，Block Layer 和 Driver 都运行在 Kernel Space，User Space 的存储引擎的 IO 都会经过 Kernel Space，会产生 Context Switch。

本地存储引擎运行在用户态，并提供了类似文件系统的语义。本地存储引擎中保存有三种不同类型的数据：

* Local Meta Data：用于保存本地存储引擎的元数据
* Journal：用于保证本地数据的写入一致性
* Data：保存数据

### 一致性协议

用于保证隔离的存储引擎之间的数据一致性

保证 zk 一致性的 ZAB 协议是一种 two-phase 协议，相较于原生 Paxos，性能效率更好

### ZBS 具体实现

![image-20220821092638630](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210926661.png)

对外接口

![image-20220821092654152](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210926194.png)

TODO ZBS 对外提供了哪些接口

IO 数据路径

![image-20220821092707252](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210927294.png)

### 性能优化

#### virtio、vhost

在半虚拟化 Hypervisor 的解决方案中，guest vm 需要完成不同设备（如块设备、网络设备、PCI设备）的前端驱动程序，Hypervisor 配合 guest 完成相应的后端驱动程序。而 virtio 就规定了这个前后端之间的通用 API。是对半虚拟化 Hypervisor 中的一组通用 IO 设备的抽象，提供了一套上层应用与各 Hypervisor 之间的通信框架和编程接口。减少了跨平台所带来的兼容性问题，大大提高驱动程序开发效率，而半虚拟化方案本身就是通过减少不必要的 IO 指令虚拟化过程来提高 IO 性能。

![image-20220821095311146](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210953193.png)

* 前端：virtio driver， (virtio-blk、virito-net、virtio-scsi) ，存在于 guest os 中；

* 后端： virtio device，（qemu, spdk/vhost-user, host kernel vhost)，提供 IO 能力。

通常 virtio 后端的驱动程序是实现在用户空间的 qemu 上，此时一个 IO 的路径包括：

```
+---------+------+--------+----------+--+
|         +------+        +----------+  |
|         |      |        |          |  |
|         |      |        | guest vm |  |
|         |      |        |          |  |
|    +----+ qemu |        | +--------+  |
|    |    |      |        | | virtio |  |
|    |    |      |        | | driver |  |
|    |    +------+        +-+--------+  |
|    |                          |       |
|    |       ^                  |       |
|    v       |  user space      v       |
|            |                          |
+-+-----+-----------------+----------+--+
| |tap  |                 |  kvm.ko  |  |
| +-----+                 +----------+  |
|             host  kernel              |
+---------------------------------------+
```

guest 设置好 tx ，发生一次内核切换陷入 kvm，又发生一次内核切换从 kvm 到 qemu，qemu 将 tx 数据投递到 tap 设备，共两次内核切换。vhost 对此进行优化，在内核中加入了 vhost-net.ko 模块，使得 guest 上的网络数据只需要一次内核切换就可以发送出去。

```
+---------+------+--------+----------+--+
|         +------+        +----------+  |
| user    |      |        |          |  |
| space   |      |        |  guest   |  |
|         |      |        |          |  |
|         | qemu |        | +--------+  |
|         |      |        | | virtio |  |
|         |      |        | | driver |  |
|         +------+        +-+---+----+  |
|                               |       |
|                               |       |
|                               |       |
|                               v       |
+-+-----+---+-+----+------+----+--+-----+
| |tap  |   | vhost-net.ko|    | kvm.ko |
| +---^-+   +------+----^-+    +----+---+
|     |-------|  kernel |-----------|   |
+---------------------------------------+
```

guest 设置好 tx ，发生一次内核切换陷入 kvm，kvm 借助 vhost-net 将 tx 数据投递到 tap 设备。整个过程 qemu 只需要负责，一些控制层面的事情，比如和 KVM 之间的控制指令的下发等。大大提高了虚拟网卡的性能。

> 一个 guest VM 或者 guest 指的是在物理计算机上安装、执行和托管的 VM。托管 guest VM 的计算机称之为 host，它为 guest VM 提供资源。Guest VM 通过 Hypervisor 在 host OS 之上运行独立的 OS。例如，host 将为 guest 提供虚拟的 NIC，guest 感觉好像是在使用真实的 NIC，而实际上使用的是虚拟的 NIC。
>
> - KVM：kernel based virtual machine，允许 Linux 充当 Hypervisor，以便 host 可以运行多个隔离的虚拟环境（guest）。KVM 基本上为 Linux 提供了 Hypervisor 功能。这意味着 Hypervisor 组件，如内存管理、调度程序、网络堆栈等作为 Linux 内核的一部分提供。VM 是由标准 Linux 调度程序通过专用虚拟硬件（例如网络适配器）调度的常规 Linux 进程。
> - Qemu：一个托管的虚拟机监视器，可以通过仿真为 guest 提供一组不同的硬件和设备模型。qemu 可以和 KVM 一起使用，以利用硬件扩展使得 guest 达到接近 host 的速度。guest 通过 qemu 命令行执行，CLI 提供了为 qemu 指定所有必须的配置选项和能力。

[Zbs 中的使用](https://docs.google.com/document/d/1EBi3d2GD4RzNUmpqXQQTyCdmIsSdZIuse6cN6cfljKk/edit)

#### SPDK

SPDK（Storage Performance Development Kit）是 Intel 开源的基于 NVMe SSD 的一个高性能存储框架。在用户空间中运行的应用程序能够直接与网络设备打交道，避免了内核上下文切换和中断处理带来的开销。该套件包含 NVMe driver、I/OAT driver、MVMF 等。

DPDK 与 SPDK 的关系。DPDK 由于做用户态轮询模式，需要对线程模型、内存管理等系统级的资源做定制的管理。如 DPDK 中利用 CPU 亲和性为线程绑定 CPU，让单个线程独享一个 CPU 内核。如大页内存管理，预先分配一块内存，进行独立的内存管理。SPDK 和 DPDK 的区别在于，SPDK 重点在于提高存储性能尤其是 NVME 驱动，而 DPDK 重点在于提供网络性能。但它们都需要对系统资源做独特管理。为了资源复用，SPDK 在 EAL 层统一采用 DPDK 的实现，无须再去自己实现一套内存管理、线程模型等底层机制。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210953635.png" alt="image-20220821095330594" style="zoom:60%;" />

主要特性

- 用户态驱动

    和常见的驱动在内核中实现不同，SPDK 借助于 UIO 或者 VFIO 将 NVMe 驱动移到了用户态，应用可以在用户态直接和 SSD 进行数据传输，这样相对于内核驱动带来的好处是，消除了 IO 调用时的上下文切换且去除了用户空间和内核空间的数据拷贝操作，有效降低了 IO 延迟。

- 轮询
    在内核驱动中，当 IO 提交到设备时，进程会进入睡眠状态，当数据传输完毕，设备会发起中断从而将进程唤醒，是一种同步方式；在 SPDK 中，并没有使用中断的方式，在提交 IO 之后继续执行之后的指令，通过使用轮询的方式检查 IO 是否完成，当 IO 完成时，使用异步的方式进行回调，消除了中断的 CPU 消耗和避免 IO 延迟抖动。

    > 这个轮询占用的时间开销比中断来的小？避免 IO 延迟抖动又是怎么理解的呢？

- 无锁

    SPDK 中的进程使用了绑核无锁机制，进程间使用无锁队列进行通信，避免了锁资源竞争导致 IO 延迟抖动。

    > 这个无锁用法值得深究

基本架构

最底层是最核心的用户态 NVMe 驱动，往上一层是基于用户态驱动程序构建的存储服务，这部分主要是统一抽象的块设备层，包含了用户空间块设备语义的抽象和多个不同后端存储实现，在块设备之上，SPDK 提供了标准存储协议的实现，使得 SPDK 可以为通用存储客户端提供高性能的存储服务。

* 驱动层

    <img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210953541.png" alt="image-20220821095357516" style="zoom:60%;" />

    用户态驱动是 SPDK 构建其他服务的基础，主要实现了的驱动有：

    * 基于 PCIe NVMe 协议的驱动，用于连接本地 NVMe SSD 设备；
    * 基于 RDMA / TCP NVMe-over-Fabric(NVMF) 协议的驱动，用于连接网络的 NVMe SSD 设备；
    * 基于 virtio PCIe / vhost user Virtio 的驱动，用于加速虚拟机 IO；
    * 基于 I/OAT 的驱动，用于提高数据拷贝效率。

* 存储服务层

    <img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210954332.png" alt="image-20220821095423305" style="zoom:67%;" />

    存储服务层主要在用户空间对块设备语义进行了统一封装抽象，并开发不同的实现用于支持不同的后端存储。

    * NVMe bdev 用于管理本地 NVMe SSD；
    * NVMe-oF 用于管理远程的 NVMe SSD；
    * Ceph RBD 用于管理 Ceph 块存储；
    * AIO / uring 用于管理内核块设备。

* 存储协议层

    <img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210954359.png" alt="image-20220821095441328" style="zoom:67%;" />

    SPDK 在存储协议层主要实现了基于网络的块存储协议，将 bdev 暴露到网络中供其他服务进行使用，除了支持 NVMF 协议之外，还支持 iSCSI、nbd 等协议；由于 bdev 层屏蔽掉了后端存储的实现，所以可以按需使用不同的协议将 bdev 进行暴露，如将 Ceph RBD暴露成 NVMe 设备给客户端使用。

#### RDMA

RDMA（Remote Direct Memory Access）指远程计算机通过网络直接读写本地内存的一种网络技术。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208210954188.png" alt="image-20220821095457152" style="zoom:40%;" />

区别于 TCP/UDP Socket，RDMA 的 IO 接口基本是异步和非阻塞的。同时，如果追求高性能的话，不能使用基于 linux signal 的通知机制（中断的耗时在这个场景下难以接受），需要使用轮询模式主动地从 RDMA 的相关队列中获取 IO 的信息。我们将重复非阻塞地查询 IO 状态并执行动作的工作模式称为 busy-polling，简称 polling。

RDMA 在一些操作上完全 bypass CPU，而在另一些操作上能做到不触发系统调用，这在网络带宽提高到 25 GbE 甚至更高的时候可以很大程度上提高网络的性能。在省掉 CPU 的复制开销的同时，还能够提高网络吞吐量、降低网络延迟。

目前 zbs 采用的 InfiniBand 网络架构提供了一整套 RDMA 解决方案，包含编程接口，链路层、网络层和传输层的协议，网卡及网卡接口和交换机等。InfiniBand 的编程接口也是 RDMA 编程接口的事实标准，RoCE 和 iWARP 都使用 InfiniBand 的接口进行编程。

在使用 RDMA 通信之前，提供方需要预留好虚拟地址空间，并通过 ibv_reg_mr 将其注册到 memory region 中，防止其内存页被交换到硬盘，之后将内存映射关系写入网卡，并将虚拟地址信息通知对端。

> RDMA 与 TCP 的区别可以深究一下。

[zbs 中的使用](https://docs.google.com/document/d/1BQRV60l9qoZRcSqRE508eyf1CQvr8T1oZ_HGPKcCuo0/edit#)

#### PMem

PMem（非易失内存）是一种新的存储介质，插在内存插槽（DIMM）上，性能介于内存与 NVME SSD 之间，具备接近内存的超低访问延迟，同时具有持久化的存储能力。基于 PMem 的高性能、低延迟和非易失等特性，smartx 希望打造以 PMem 作为缓存，NVMe NAND SSD 作为存储介质的全闪超融合解决方案。使用该方案，SmartX 超融合一体机三个节点的最小系统在 4K 随机读中可达到 120 万 IOPS，而且虚拟机端的 IO 延迟可从 ms 级别降低至 μs 级别，大幅度改善业务系统延迟。

开启 PMem 之前需要确保有以下硬件配置：

* 持久内存：PMem 支持单条 128G、256G 及 512G 三种容量，目前我们的一体机中使用均为 128G 的规格。需集群中所有主机的 PMem 设备的固件版本相同。
* 存储：需配置为全闪模式，且磁盘需全部使用 NVMe SSD。
* CPU：需支持 I/OAT DMA。通过 I/OAT DMA 引擎将内存拷贝任务交给 DMA 而非 CPU 处理。
* 网卡：由于需配合 RDMA 使用，因此需使用支持 RDMA 的特定网卡如迈络思 Mellanox 
* 交换机：由于需配合 RDMA 使用，因此需要交换机支持 L3 DSCP 和 Global Pause 中的一种流控配置（可通过交换机手册检查）。

### 数据一致性

1. Meta 管理的元数据。Meta DB 是什么？是如何工作的？

3. 为什么 zbs 选用集中式元数据管理方式？

    [链接](https://docs.google.com/document/d/1rqegbfDWQ_9ruUBfaN_jBskdFuOcQ1CQ1-BqqjU0pPE/edit)，分析了集中式、基于自动分片的分布式、基于哈希的分布式三种优缺点，并从超融合的角度出发，分析了为啥要选用集中式。主要还是私有云超融合场景下元数据的规模不大，集中式的单机内存存储完全可以满足需求。

5. 服务器故障，怎么使用周期性的快照和快照之后的日志恢复？具体流程？zk 论文的 4.3 节

6. Zookeeper 是一个开源的分布式协调服务框架 ，主要用来解决分布式集群中应用系统的一致性问题和数据管理问题

    1. 怎么做选举，怎么保证正确性，
    2. Meta db , task db 不同的 db ，怎么实现的，他们两个实现的差异，为什么这么设计两个不同的 db
    3. meta db 怎么保证集群一致的，要从故障、异常的角度考虑
    4. Zk 的完整解读，paper 解析 zk 如何实现，有一篇博士论文
    5. raft 要求一定会，zk 的原始论文要读

7. Chunk 管理的用户真实数据。Jornal 的作用是什么？它是如何工作的？

### 副本分配策略

1. 为什么采用局部化策略作为副本分配策略，优缺点是啥？
1. 快照，https://developer.aliyun.com/article/716202

### 论文阅读

1. Google Big Table
2. Fackbook Dynamo，
3. Ceph CRUSH，没有中心节点，根据 crush 算法计算副本所在节点
4. Google Chubby，为分布式系统提供一个可靠的粗粒度的锁服务
5. Standford Raft，Paxos 的简易实现
6. Nutanix Bible