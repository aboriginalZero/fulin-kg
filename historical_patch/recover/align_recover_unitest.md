单测 StagingBlockInfoCid3 中 recover 的时候，有一个 regular write，recover 结束后 dst_chunk 上的数据会是最新的 regular write 数据，这个怎么实现的，还挺神奇的。

单测 StagingBlockInfoCid4 中，不能在 RecoverEnd 处停，因为这时候 recover 已经把 1024 个 block 做完了，这时候再来一个 regular io，由于 gen 不一样导致 recover 失败。

```
[ECGenerationNotMatch]: extent generation not match in recover end. extent: status: PEXTENT_STATUS_RECOVERING pid: 1 epoch: 0 generation: 2 bucket_id: 1 pblob_table_id: 1 pblob_group_id: 2097153 local gen: 2 request gen: 1
```

单测 StagingBlockInfoCid4 中，如果是 2 台 chunk，副本仅在 a 上，下发一条 recover cmd 到 b：

1. 如果让 dst_chunk 上写副本失败，会 cancel recover，因为 regular io 失败，这时候 recover 就算成功，也会因为部分被 regular io 写入的 block 失败而失败。
2. 如果让 src_chunk 上写副本失败，会 ECAllReplicaFail，因为源副本写失败，说明当前集群中没有健康的这个 pid 的副本，之后 ROLLBACK，在 zbs_client 那一侧重试。



lease 是怎么分配的，跟 recover 之间的关系？Access Server 会主动释放 lease 吗？在代码中的哪里体现

sync generation 就是去获取当前副本的最新状态，不一样取高的，剔除低的 gen 副本。

心跳就是拿来状态上报和命令下发两件事。



git review -d xxx

meta 切换 <30s 主要花在什么时间

pextent_table 之类的 meta leader 独有的表每次更新都会在 zk 中同步更新吧？是的。zk 只是存 kv，并且都常驻内存了，但实际上，zbs 场景下写比较多，大部分键值对可以持久化到硬盘而避免占用大量内存，只是之后组里要改进的一个点。

怎么控制一个 rpc 方法是同步还是异步的，比如 meta_rpc_server 的 RemovePExtentReplica 看上下文知道是同步的，但是在代码哪里体现？

#### 疑惑

* reocver_handler.cc 中把慢盘的情况单独统计出来的原因是为了好给用户反馈吗？

* 块存储白皮书中，本地 IO 路径部分

    > 每个写入请求都会写入一条 Journal，当写入请求大小小于 block 大小时，Journal 中包含完整的请求数据,。而当缓存命中（即缓存有足够的空间供相关数据写入或相关数据已经在缓存中存在）且请求大小为 block 大小时（常见于顺序写场景，接入层会将顺序 IO 合并至 block 大小下发至 Chunk），数据会直接写入缓存，并在稍后异步同步到 ExtentStore。此时 Journal 不携带数据，仅记录操作元数据，避免了写放大。

    这个 hping 讲过一次，找个机会记录下来。还有 CacheManager 部分，

    > 当数据第一次被写时，数据会直接写到 ExtentStore 上（如果是随机 IO，则会先写到 Journal 中）。

    为啥顺序 IO 不需要写 Journal

* 在没有 IO 的情况下，收到一条 recover 命令是怎么工作的

* 正在 recover 的时候，怎么保证新来的 write 与 recover 可以不冲突的正确写入，有 write 的时候，recover 是如何完成的

    读写锁，recover 是写锁，regular IO 是读锁。

    agile recover 跟 normal recover 还不一样，由于 agile recover 中两次 GetGeneration 间不能接受write，所以当 recover 处于 START 状态，regular write 需要等待。

    还有其他部分，待补充

#### Extent 生命周期

以一个 Thin Provision Volume 的 Extent 为例，整个 Extent 的生命周期包括如下阶段：

1. 创建 Volume 时， Extent 被初次分配，获得了 pid（此时还没有分配副本，只是在 VExtent Table 中写入 pid）；
2. Chunk 申请 VExtent 的 IO lease，Meta 为 Extent 分配 Replica；
3. Chunk 向获得的 lease 中的各个 Replica 写入数据，初次写入成功之后上报 Meta，Meta 记录 Extent 状态为存在；
4. Chunk 出现异常，Meta 通过心跳发现 Extent 活跃副本数不足，触发 Recover；
5. Chunk 执行 Recover 完成，Extent 出现在新的副本位置；
6. Volume 被删除，Extent 在 GC 流程中被回收；
7. Chunk 心跳上报 Extent 存在，Meta 认为 Extent 无效，下发 GC 命令；
8. Chunk 回收 Extent 所占用的空间；

#### MetaServer 启动流程

1. MetaSMServer::Run() --> MetaSMServer::SMThreadFunc()、MetaSMServer::SetEnable() --> MetaSMServer::StartMetaServer() --> MetaSMServer::MetaServerThreadFunc() --> MetaServer::Initialize() 调用的是 QuorumServer 的方法
2. MetaServer::Run() 也是调用 QuorumServer 的方法，当 ENTER_RUNNING 事件被 zk 感知，调用 MetaServer::StartService()
    1. StartWatchDog()
    2. new MetaContext 并且 Initialize
    3. RunStatusServer()
    4. 如果是 meta leader ，StartLeader --> RunMetaRpcServer()、RunGCManager()，否则 StartFollower --> RunMetaRpcServer()
3. 而在 MetaContext::Initialize 中，包括整个系统主要组件的初始化，比如 CommonRpcServiceInitialize、ChunkManagerInitialize、PExtentTableInitialize、PExtentAllocatorInitialize、HTTPServerInitialize、GcManagerInitialize、VolumeManagerInitialize、MetaRpcServerInitialize、NFSServerInitialize 等。
    如果是 meta leader ，还会执行 RecoverManagerInitialize 和 AccessManagerInitialize，分别 new RecoverManager / AccessManager 并且 Initialize
4. 【RecoverManager】在 new 一个 RecoverManager 对象的时候就会 start 当前线程，开始 loop 每 4s 执行一次 RecoverManager::DoScan()
    1. CleanTimeoutAndFinishedCmd()、GenerateRecoverCmds()、GenerateMigrateCmds()
    2. AddRecoverCmdUnlock() 往 access_manager 中 EnqueueRecoverCmd，即将 active 队列中 extent 生成的 recover cmd 放入 chunk 队列中等待下发
        1. AllocOwner，为要 recover 的 pextent 分配 owner
        2. GenerateLease，为要 recover 的 pextent 生成 lease ，作为 recover_cmd 的 lease，owner 是先前生成的 owner
        3. 往 owner session 的 waiting_recover_cmds 放入 recover_cmd，随着 keepalive 发往目的 session
5. 【AccessManager】SessionMaster::KeepAlive 作为对 SessionFollower::KeepAliveLoop 暴露的 rpc 入口，调用了作为 keep_alive_cb_ 的 AccessManager::HandleKeepAlive 之后，调用 MasterSession::HandleKeepAlive 响应 follower 的心跳请求。
    1. AccessManager::HandleKeepAlive --> AccessKeepAliveResponse --> ComposeAccessResponse 这个方法把每个 AccessSession 的 waiting_recover_cmds 中的 recover cmd 放入 AccessKeepAliveResponse，跟随心跳下发到 SessionFollower
    2. 

待补充

具体可以参考[Meta in ZBS](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.esxx3wtrrxe)

#### DataChannel

DataChannel 用来处理从 Client 到 Server 的请求。定义的主要接口

```
ReadVolume/WriteVolume
ReadVExtent/WriteVExtent
ReadPExtent/WritePExtent
RecoverStart/RecoverEnd
GetGeneration
```

重要的是发包出去的包头 MessageHeader 中会指定类型，这样目的端 LocalIOHandler 就可以根据 MessageHeader 调用相应的处理函数，如 HandlePExtentRequest、 HandleRecoverRequest 等。

Data_channel_client_v2::ReadVolume() --> Data_channel_client_v2::DoRequest() --> Client_transport_manager::SendRequest()  --> Client_transport::SendRequest() -->  Socket_client_transport::TrySendRequest()  --> Socket_client_transport::DoSendRequestParts --> TcpSender::SendRequest()

#### MetaRpcServer::GetVExtentLease

1. 如果是 Read

     MetaRpcServer::GetLeaseForRead ，通过 volume id 和 vextent_no 可以从 VExtentTable 中拿到对应的 pid，然后执行 MetaRpcServer::DoGetLease ，包括以下两个步骤：

     1.  PExtentTable::GetPExtentTableEntry ，通过 pid 可以从 PExtentTable 中拿到对应的 PExtentEntry，其中信息包括副本数、副本位置
     2. AccessManager::AllocOwner ，通过 pid 和 preferred_session 拿到对应 Lease 的 owner（是一个 SessionInfo ）

     > meta in zbs 中写的是，当发生 Snapshot/Clone/Rollback 等需要调整 vextent 中的 COW flag 操作时，Meta leader 会通知清理所有 Access 持有的 vextent lease 信息，以便 chunk 重新申请获取 lease 的最新状态。

2. 如果是 Write

    MetaRpcServer::GetLeaseForWrite ，通过 volume id 和 vextent_no 可以从 VExtentTable 中拿到对应的 pid，然后根据 thin/thick 和 COW 做不同处理：

    1. 如果这个 extent 已经被分配（比如 volume 为 thick 模式，或者 thin 模式但是之前已经被写过），不做任何处理
    
    2. 如果 Volume 为 thin 并且副本尚未分配，此时分配副本，持久化 pextent 副本信息，更新内存中的数据和 chunk 的数据空间信息，之后执行类 Read 操作，且操作的 pid 是新副本的 pid
    
    3. 如果此时 volume 的 pid 都还没分配（比如写一个 size 为 0 的 NFS file），那么在 GetLease 时需要申请 pids，分配副本，持久化 pextent 副本信息，通过 AllocPExtentTransaction 更新内存中的 PExtentTable  （没有更新 chunk 的数据空间和类 Read 操作吗？）
    
    4. 如果是写一个有 COW 标记的 Volume，那么需要调用 CowPExtentTransaction 进行 COW PExtent 的处理，包括申请一个新的 pid，在 origin_pid 的 location 上分配副本，用新的 pid 更新 VExtentTable，PExtentTable 中的 PExtentEntry，并在 meta db 上持久化 Volume 信息。
    
        > meta in zbs 中写的是，会修改 Volume 的 vtable 将对应带有 COW 标记的 vextent 替换成一个新的不带有 COW 标记的 vextent，新的 vextent 对应一个新的父级指向原 extent 的 child extent，ZBS Client / Access 会收到新的 vextent lease 以继续 IO 请求。
    
    执行 MetaRpcServer::DoGetLease 操作，从 PExtentTable 中获取 extent 详细信息，并查询 AccessManager 获取 owner

GenerateLease，通过 pid、MetaContext（包含这个 pid 的 PExtentEntry）和 owner 构造一个 Lease

#### libmeta::RemovePExtentReplica

在 zbs_code_intro_ans.md 中

#### AccessIOHandler::SyncGeneration

在 zbs_code_intro_ans.md 中

#### ZbsClient::Read

以 ZbsClient::Read 作为起点，一个 IO 下发到 ZBS 后各组件的交互过程

1. ZbsClient::Read() --> OP::VOLUME_READ 类型的 ZbsClient::DoIO(volume_id, buf, len, offset)

    如果有 cache， IOCacheManager::Cache()，否则 ZbsClient::SplitIO 以 extent 为粒度切分成一组 IO

    如果是正常的读写 IO，走的是 chunk 的内部线程，所以走 InternalIO 通道。InternalIOClient::SplitIO() --> ZbsClient::ProcessIO() --> InternalIOClient::SubmitIO()

    > 如果是做备份的 taskd，由于与 chunkd 不在一个进程，io clinet 是 ExtenrnallIO 

    1.  Meta::GetVExtentLease 根据 volume_id 和 vextent_no 向 libmeta 请求对应的 extent lease，如果 libmeta 中有缓存，那么直接将 lease 信息返回给 io client，否则向 meta leader 请求 lease 并缓存到本地 libmeta。

    2. InternalIOClient::IsLocal ，io client 根据拿到的 lease 中的 SessionInfo 中的 uuid，判断是否在本地。
       
        > io client 拿到的 lease 只通过里面的 uuid 判断 lease 是否在本地，其他信息没有被利用。io client 向 meta leader 申请 lease，如果这个 pid 已经被分配到其他 chunk，那么返回已有的 lease，否则 lease 分配到当前 io client 所在的 chunk
        
        1. InternalIOClient::DoLocalIO，如果 lease 在本地，那么执行本地的 AccessIOHandler::ReadVExtent
        2. InternalIOClient::DoRemoteIO，否则，根据 Lease 的 IP+port 通过 data channel manager 拿到一个 dc client，通过这个 client 下发 IO。当 IO 达到相应的 Access 后，由其 access io handler 处理 IO，aih 上的 DataChannelServerInterface 注册了 VEXTENT_READ 的处理函数为 AccessIOHanlder::SubmitReadVExtent

2. 至此，完成 IO 从 io client 转发到 access io handler 的过程。

    1. 获取 Lease，这次要使用其中的 pid, loc, uuid, cid, meta_generation 等字段信息，AccessIOHandler::GetVExtentLease

    2. 【Sync Generation】执行 Sync Generation 的操作，AccessIOHandler::SyncGeneration --> GenerationSyncor::Sync --> DoSync 

    3. 执行到这，说明 Sync Gen 成功，标记着这个数据存在有效副本。那么 AccessIOHandler::SetupIOCtx，然后 AccessIOHandler::DoReadVExtent --> AccessIOHandler::ReadReplica，只要有一个读成功就返回

        其中调用 AccessIOHandler::IsLocal ，access io handler 根据拿到的 lease 中的 SessionInfo 中的 cid，判断是否在本地
        1. 如果 lease 在本地，那么直接调用本地的 lsm 处理 IO，执行 LSM::ScheduleRead
        2. 否则，根据 Lease 的 IP+port 通过 data channel manager 拿到一个 dc client，通过这个 client 下发请求头为 PEXTENT_READ IO 请求，目的 chunk 上的 LocalIOHandler 收到这个请求后，根据 MessageHeader::PEXTENT_READ 注册的 LocalIOHandler::HandlePExtentRequest 调用它本地的 lsm 处理 IO

5. 
   至此，完成 IO 从 access io handler 到 lsm 的过程。

#### ZbsClient::Write

以 ZbsClient::Write 作为起点，一个 IO 下发到 ZBS 后各组件的交互过程

1. ZbsClient::Write() --> OP::VOLUME_WRITE 类型的 ZbsClient::DoIO(volume_id, buf, wbuf_owner, len, offset)。

    如果有 cache， IOCacheManager::Cache()，否则 ZbsClient::SplitIO 以 extent 为粒度切分成一组 IO。

    如果是正常的读写 IO，走的是 chunk 的内部线程，所以走 InternalIO 通道。InternalIOClient::SplitIO() --> ZbsClient::ProcessIO() --> InternalIOClient::SubmitIO()

    1. `Meta::GetVExtentLease`根据 volume_id 和 vextent_no 向 libmeta 请求对应的 extent lease，如果 libmeta 中有缓存，那么直接将 lease 信息返回给 io client，否则向 meta leader 请求 lease 并缓存到本地 libmeta。

    2. `InternalIOClient::IsLocal`，io client 根据拿到的 vextent lease 中的 SessionInfo 中的 uuid，判断是否在本地。
        1. `InternalIOClient::DoLocalIO`，如果 lease 在本地，那么执行本地的 AccessIOHandler::WriteVExtent，跟后者的区别在于 w_buf = SetupWriteIOBufArg，待细追区别
        2. `InternalIOClient::DoRemoteIO`，否则，根据 Lease 的 IP+port 通过 data channel manager 拿到一个 dc client，调用这个 client 的 DataChannelClient::WriteVExtent。当 IO 达到相应的 Access 后，由其 access io handler 处理 IO，aih 上的 DataChannelServerInterface 注册了 VEXTENT_WRITE的处理函数为 AccessIOHanlder::SubmitWriteVExtent --> AccessIOHandler::SubmitWriteVExtent

2. 至此，完成 IO 从 io client 转发到 access io handler 的过程。

    1. 获取 Lease，这次要使用其中的 pid, loc, uuid, cid, meta_generation 等字段信息， `AccessIOHandler::GetVExtentLease`

    2. 【Sync Generation】执行 Sync Generation 的操作，AccessIOHandler::SyncGeneration --> GenerationSyncor::Sync --> DoSync，具体判断逻辑暂不展开。

    3. 执行到这说明 Sync Gen 成功，标记着这个数据存在有效副本，那么执行 AccessIOHandler::SetupIOCtx。这边有个可以加速的特殊情况是如果 wctx.lease 的 gen 为 0 、没有 origin_pid 、写的是全 0 数据，那么无需经过 lsm 就可以直接返回 Status::ok

        1. wctx.lease 的 gen++，AccessIOHandler::DoWriteVExtent --> 等待多个副本 AccessIOHandler::WriteReplica 同步执行完毕。


        其中调用 AccessIOHandler::IsLocal ，access io handler 根据拿到的 lease 中的 SessionInfo 中的 cid，判断是否在本地
    
        1. 如果 lease 在本地，那么直接调用本地的 lsm 处理 IO，执行 LSM::ScheduleWrite
        2. 否则，通过 dc client 下发请求头为 PEXTENT_WRITE 的写请求，被目的 chunk 上的 LocalIOHandler::HandlePExtentRequest 处理

3. 至此，完成 IO 从 access io handler 到 lsm 的过程。

4. 与读操作不同，当写操作执行完还要执行 AccessIOHandler::WriteVExtentDone

    统计写失败副本的个数，标记慢盘，调整失败副本上的 recover 行为，并为写失败的副本创建 StagingBlockInfo

#### Recover 基本流程

当在 Sync Generation 过程中出现剔除副本的操作后，meta leader 会向下发送 revoke 命令

1. AccessHandler 借助 access follower 创建跟 meta leader 的 session，并带有对应的 KeepAlive 回调 AccessHandler::HandleKeepAlive --> HandleAccessResponse，所以 recover 命令由 meta leader 发往对应的 session follower

    > meta leader 根据 pid lease 中的 owner 来判断 follower，如果这个 pid 当前没有 lease，那么分配给 src_cid 

2. RecoverHandler::NotifyRecover --> HandleRecoverNotification(recover_cmd) 
    这时构造 RecoverContext，其中指定了回调函数是 ctx->done = HandleRecoverEvent

3. RecoverHandler::DoRecover 
    做一些 recover 的准备工作，比如获取要 recover 的 pextent 的 Coroutine 排他锁、ctx->set_state(RecoverState::START)

    1. RecoverHandler::SetupRecover 做一些判断检验，比如 lease、location、generation、VerifyCmd

        1. lease 检验

            通过本地 libmeta 获取指定 pid 指定 epoch 的 lease，如果这个 lease owner 的 uuid 跟 libmeta 的 session_uuid 不同，那么说明是从缓存中获取的 lease 并且这个 lease 过期了。

        2. 同步 gen

            如果通过 lease 得知 dst_chunk 上已经包含指定 pid 的有效副本

            1. lease 中的 location 没有 src_chunk，那根本都无法从 src_chunk 中读数据，返回 EInvalidArgument

            2. 获取 src_chunk 的这个 pid 的 generation，这个获取具体过程看上边

                如果拿到的 generation 是 0 并且已经被写过（lease 的 ever_exist 为 true），这就已经自相矛盾，返回 ECBadExtentStatus

                否则，剔除 dst_chunk 上这个 pid 的副本 `Meta::RemovePExtentReplica`

            需要同步 gen，其中的 VerityAndGetGeneration 执行流程

            1. RecoverHandler::GetReplicaGeneration
            2. 通过 dcc 发往 src_chunk 上的 LocalIOHandler，由 LocalIOHandler::HandleGetGeneration 处理
            3. LSMInterface::VerifyAndGetGeneration --> LSMProxy::VerifyAndGetGeneration --> LSMv2::VerifyAndGetGeneration --> LocalIOHanlder::LocalIODone
        
    2. RecoverHandler::RecoverStart 通知 dst_cid 所在 chunk 的 lsm  做 RecoverStart，应该是为 dst_cid  创建对应的 extent，待细追
        1. dst_cid is local，通过本地 lsm 执行 RecoverStart，执行完调用 ctx->done
        2. dst_cid isn't local，通过 DataChannel 通知对方执行 RecoverStart，执行完调用 ctx->done
    
4. RecoverHandler::HandleRecoverEvent，其中的状态机有 4 种状态：
   
    * START
    
        如果 recover 做完，状态更改为 END，并执行 RecoverEnd，执行流程跟 RecoverStart 一样分本地和远程两种，RecoverEnd 之后回到 HandleRecoverEvent
    
        如果还没有做完，状态更改为 READ， 并执行 ReadFromSrc，执行流程跟 RecoverStart 一样分本地和远程两种，read 之后回到 HandleRecoverEvent
    
    * READ
    
        状态更改为 WRITE， 并执行 WriteToDst，执行流程跟 RecoverStart 一样分本地和远程两种， write 之后回到 HandleRecoverEvent
    
    * WRITE
    
        更新 recover_block_num、cur_block 等参数，如果是敏捷恢复则判断下一个要 recover 的 block，并且直接转到 START 的逻辑
        
    * END
    
        更新 meta  上的副本位置，ReplacePExtentReplica、DropLeaseIfNecessary 

#### 数据结构

StagingBlockInfo、CachedLease、RecoverInfo、RecoverContext

```c++
class StagingBlockInfo {
  public:
    StagingBlockInfo(cid_t cid, uint64_t expect_valid_gen) : cid_(cid), expect_valid_gen_(expect_valid_gen) {
        memset(block_bitmap_, 0, sizeof(block_bitmap_));
    }

    cid_t cid() const { return cid_; }
    uint64_t expect_valid_gen() const { return expect_valid_gen_; }
    void set_expect_valid_gen(uint64_t gen) { expect_valid_gen_ = gen; }

    void SetBitmap(uint32_t bit);
    void ClearBitmap(uint32_t bit);
    uint32_t FindNextStagingBlock(uint32_t bit);

  private:
    cid_t cid_;
    // expect valid generation of dst extent, 
    // used for validation during agile recover
    uint64_t expect_valid_gen_;
    // 1024 个 bit，即一个 extent 中 block 的个数
    uint64_t block_bitmap_[16];
};

class CachedLease : public Lease,
                    public RefCounted<CachedLease> {
    bool expired_;
    bool cow_;
    bool io_hard;  
    uint64_t gen_;
    uint64_t last_access_;
    std::unique_ptr<StagingBlockInfo> staging_block_info_;
};

struct RecoverContext : public RecoverInfo {
    ~RecoverContext() {
        if (!coros.empty()) {
            while (!coros.empty()) {
                auto co = coros.front();
                coros.pop_front();
                co->Enter();
            }
        }
    }

    std::string ShortDebugString() const {}

    StopWatch              sw;
    Controller             ctrl;
    Callable               done;
    char*                  buffer;
    loc_t                  cmd_location;
    uint32_t               recover_block_num;
    std::deque<Coroutine*> coros;
    uint64_t               start_ms;

    // this is used to verify dst extent not change during agile recover check
    uint64_t               origin_gen;

    libmeta::CachedLeasePtr lease;
    std::unique_ptr<libmeta::StagingBlockInfo> staging_block_info;
    bool canceled;
    bool is_cross_zone;
    bool agile_recover;
};

message RecoverInfo {
    required uint32       pid          = 1;
    required RecoverState state        = 2;
    required uint32       cur_block    = 3;
    required uint32       src_cid      = 4;
    required uint32       dst_cid      = 5;
    required bool         is_migrate   = 6;
    required uint64       silence_ms   = 7;

    optional uint32       replace_cid  = 8 [default = 0];
    optional uint64       epoch        = 9;
}
```

Lease 、 VExtentLease 、RecoverCmd

```c++
message VExtentLease {
    required Lease lease = 1;
    optional bool cow = 2;
}

message Lease {
    // owner info
    optional SessionInfo owner        = 1;
    // corresponding pextent info
    // 使用 28 个 bit
    optional uint32 pid               = 10;
    optional uint32 location          = 11;
    // origin_pid 什么意思
    optional uint32 origin_pid        = 12 [default = 0];
    // 因为 pid （28 bit）会被回收利用，pid + epoch 的组合才可以唯一确定某个 Extent
    optional uint64 epoch             = 13 [default = 0];
    optional uint64 origin_epoch      = 14 [default = 0];
    // 如果为 true，说明这个 pextent 被写过（默认精简配置）
    optional bool   ever_exist        = 15 [default = false];
    // meta 记录的 generation
    // pextent 每次发生副本变更（recover/migrate）generation 都会 ++
    // 取最大的 generation 为正确 extent
    optional uint64 meta_generation   = 16 [default = 0];  // meta-recorded gen
    optional uint32 expected_replica_num = 17;
    repeated Chunk chunks             = 30;
}

message SessionInfo {
    required bytes uuid = 1;
    required bytes ip = 2;
    optional uint32 num_ip = 3;  // num representation of ip
    required uint32 port = 4;
    optional uint32 cid = 5;  // chunk cid if session is a chunk session
    optional uint32 local_cid = 6;  // closes chunk to this session (e.g. local chunk)
    optional bytes secondary_data_ip = 7;
    optional bytes zone = 8;
    optional bytes scvm_mode_host_data_ip = 9;
    optional uint64 alive_sec             = 10;
}

message RecoverCmd {
    required uint32 pid = 1;
    optional Lease lease = 2;  // lease for the pextent
    required uint32 dst_chunk = 3;
    optional uint32 replace_chunk = 4;
    optional uint32 src_chunk = 5;
    optional bool is_migrate = 6 [default = false];
    optional uint64 epoch = 7;
    // active_location when meta generation cmd, only for display now
    optional uint32 active_location = 8;

    // the following is for internal use by MetaServer
    optional uint64 start_ms = 21;
}
```

更换 lsm1

```
    1  2021-10-14 10:52:50 vim /etc/sysconfig/zbs-chunkd
    2  2021-10-14 10:52:58 cd /var/lib/zbs
    3  2021-10-14 10:53:01 mv lsm-meta lsm-meta.bak
    4  2021-10-14 10:53:04 systemctl restart zbs-chunkd
    5  2021-10-14 10:53:17 zbs-chunk journal mount /dev/sdb3
    6  2021-10-14 10:53:21 zbs-chunk journal mount /dev/sdc3
    7  2021-10-14 10:53:28 zbs-chunk cache format /dev/sdb4
    8  2021-10-14 10:53:33 zbs-chunk cache mount /dev/sdb4
    9  2021-10-14 10:53:37 zbs-chunk cache format /dev/sdc4
   10  2021-10-14 10:53:41 zbs-chunk cache mount /dev/sdc4
   11  2021-10-14 10:53:45 zbs-chunk partition format /dev/sdd1
   12  2021-10-14 10:53:49 zbs-chunk partition mount /dev/sdd1
   13  2021-10-14 10:57:45 history
```

#### 基本概念

* chunk 把节点所有磁盘（系统盘除外）划分成 Cache、Journal、Partition 三大块，Cache 做缓存，Journal 存日志，Partition 存数据。

* thin/thick 是存储卷的 thin_provision 属性，是创建卷的概念。如果设置为 thin，说明精简配置，在没有发生写操作时，只分配 pid，只有写的时候才会为 extent 申请内存空间。如果设置为 thick，说明是厚备配置，在卷创建的时候就会为 extent 分配空间。

* COW 指的是克隆卷时的写时复制。当 clone 一个volume 的 extent 时，不会立即复制出一个 extent，而仅仅只是复制 PExtentTableEntry，我们的克隆/快照操作时刻遵循写时复制。

* ChunkTable 中存的是 ChunkTableEntry，即记录每台 chunk 上 normal / recover / migrate / reserved pextent 的数量，还有上次往 meta leader 成功发送心跳的时间

* VExtentTable 中存的是 volume id + vextent_no 跟 pid 的映射关系

* PExtentTable 中存的是 pid 跟 PExtentEntry 的映射关系，PExtentEntry 中包含副本数、副本位置、Lease owner 等信息，举个例子，pid 为 776 的 PExtentEntry 如下：（根据 message PExtentInfo 得来）

    ```
     ID  						776
     Replica					[1, 3, 4]
    Alive Replica				[1, 3, 4]
    Ever Exist					True
    Is Garbage					False
    Origin PExtent				0
    Expected Replica num		3
    Epoch						1268
    Prefer Local				1
    Lease Owner					0
    ```

* Storage Pool -> Pool -> Volume -> VExtent / PExtent -> Block

    * Storage Pool 指定用哪几台 Chunk

        ```c++
        message StoragePool {
            optional bytes id = 1;
            optional bytes name = 2;
        
            // chunks in the storage pool
            repeated Chunk chunks = 3;
        }
        ```

    * Pool 为 Volume 分类并提供默认存储策略。指定其中的 Volume 的精简配置、默认副本数、白名单、条带化策略等

        ```c++
        message Pool {
            required bytes name = 1;  // < 255B
            optional uint32 id_v1 = 2 [deprecated=true];
            optional bytes id = 3;
            optional TimeSpec created_time = 4;
        
            // if not specified, the system storage pool is used
            optional bytes storage_pool_id = 5;
        
            optional uint32 replica_num = 11;
            optional bool thin_provision = 13;
        
            optional bytes description = 21 [default = ""];  // 256B
        
            // deprecated: (v1 needed it)
            optional bool nfs_export = 42 [default = false, deprecated = true];
        
            // mod_verf is a guard against unintended modification/removal of
            // a pool, if set any future requestis attempting to alter the pool
            // needs to provide a match verification number
            optional uint32 mod_verf = 43;
        
            // ip v4: 192.168.10.11
            // ip/net-mask: 192.168.10.0/255.255.255.0
            // ip/mask: 192.168.10.0/24
            // */*
            optional bytes whitelist = 44 [default = "*/*"];
        
            // copied to every volume
            optional IOThrottleConfig throttling = 50;
        
            // copied to every volume
            optional uint32 stripe_num = 61 [default = 1];
            optional uint32 stripe_size = 62 [default = 262144];  // 256K
        }
        ```

    * Volume 面向用户，对应一个虚拟磁盘或存储卷

        ```c++
        message Volume {
            optional bytes name = 1;  // < 255B
            optional uint64 size = 2; //Bytes
            optional uint32 id_v1 = 3 [deprecated=true];
            optional uint32 parent_id_v1 = 4 [deprecated=true];
            optional TimeSpec created_time = 5;
            optional bytes id = 6;
            optional bytes parent_id = 7;  // parent id
            // where this volume is from: either a snapshot or a source volume (when
            // clone from a volume)
            optional bytes origin_id = 8;
        
            optional uint32 replica_num = 12;
            optional bool thin_provision = 14;
        
            // read only volume could be opened by multiple readers and can only be open
            // for read
            optional bool read_only = 21 [default = false];
        
            optional VolumeStatus status = 33 [default = VOLUME_ONLINE];
        
            optional bytes description = 41 [default = ""];  // 256B
            optional uint64 iops = 51 [default = 0, deprecated=true];
            optional uint64 iops_burst = 52 [default = 0, deprecated=true];
            optional uint64 bps = 53 [default = 0, deprecated=true];
            optional uint64 bps_burst = 54 [default = 0, deprecated=true];
            optional IOThrottleConfig throttling = 55;
        
            optional uint32 stripe_num = 61 [default = 1];
            optional uint32 stripe_size = 62 [default = 262144];  // 256K
        
            // used by snapshot or cloned volume
            optional bool is_snapshot = 70 [default = false];
            optional bytes snapshot_pool_id = 500;  // snapshot's pool id
        
            // diff size when cloning or snapshotting. Note this value is only created
            // when the volume is created. So it doesn't reflects the diff (unique) size of
            // the system later.
            optional uint64 diff_size = 71 [default = 0];
            optional NFSAttr nfs_meta = 72;
        
            // unique size means data size only used by this volume
            // unique_size == -1 means it has never be scaned
            optional int64 unique_size = 73 [default = -1];
            // shared size means data size shared with other volume
            // shared_size == -1 means it has never be scaned
            optional int64 shared_size = 74 [default = -1];
        
            repeated AccessPoint access_points = 80;
        
            optional bool alloc_even = 81 [default = false];
            // how many time has volume be cloned
            optional uint64 clone_count = 82 [default = 0];
        
            optional bool skip_all_zero_first_write = 83 [default = false];
        
            optional bytes secondary_id = 84;
            optional uint32 prefer_cid  = 85 [default = 0];
        
            repeated uint32 vextent_id = 100;
        
            optional bool file = 300 [default = false];
        
            // allocated virtual size
            optional uint64 alloced_virtual_bytes = 400;
        
            // allocated physical size = virtual size x replica num - missing
            // replicas size
            optional uint64 alloced_physical_bytes = 401;
        }
        ```

    * Extent

        ```c++
        message VExtent {
            required uint32 vextent_id     = 1; // vextent table, max size: 64KB
            required uint32 location       = 2;  // pextent locations;
            required uint32 alive_location = 3;
        }
        
        message PExtent {
            required uint32 pid      = 1;
            optional uint32 location = 2 [default = 0];
        
            optional uint32 alive_location = 21;
        
            optional uint32 expected_replica_num = 12;
            optional bool ever_exist = 13;
        
            optional bool garbage = 14 [default = false];
            optional uint32 origin_pid = 15;
        
            optional uint64 epoch = 16 [default = 0];
            optional uint64 origin_epoch = 17 [default = 0];
            optional uint64 generation = 18 [default = 0];
        
            optional uint32 preferred_cid = 19 [default = 0];
        }
        ```

    * Block

        参见 lsm.proto

