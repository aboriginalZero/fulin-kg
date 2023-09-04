1. 做 meta 的同学，互相 review 或者 CC，像 ec group 比如 fanyang / tengqiu，我跟 weipeng/shunkai；
2. 建议在设计文档中，最终 Owner 能用一个表格的形式体现出策略对比，清晰直接；
2. 生产代码应该少的出现断言，避免引起不必要的 coredump？
2. zhiwei 提议，测试版可以多点日志，等要上 df 的时候再去掉
2. 为啥 recover/migrate cmd queue 一个用 list 一个用 set
2. 如果要重构 recover manager 的思路





2023.6.15

1. 检查 topo obj name 查重的问题，yanlong 写测试脚本时发现 API / cli（zbs-meta topo list）中返回的 name 不一致，有些为 chunkX，有些为 hostname，预期应该是 chunkX（X 表示 cid）。

   调用发现，tower 没有这个问题，但是通过 fisheye 修改确实会同步设置 hostname 为 name，在 pyzbs 那每次调用 updateTopoObj 的时候都会必传 new_name（7 年前的代码），即使只是移动拓扑位置。这个 pyzbs 侧的问题其实 1 周前就发现了，但是当时评估 fisheye 不更新了，所以也就没改。但 smtxauto 那边集成测试还挺多是基于 fisheye 提供的接口，然后就在昨天 qe 的冒烟测试那暴露出这个 bug。

   这个过程主要是发现调用链 tower -> pyzbs -> zbs-client-py -> zbs rpc

   fisheye 调用 api 会映射到底层多个服务： 有些服务是 v2，有些是 v3，zbs 相关的 restful api 都是 v2 的

   tower 是集群上的一个虚拟机，用来管所有的虚拟机（多集群管理），fisheye 只是单机管理（虽然因为历史原因也提供了集群管理能力）。tower 调用 restful，pyzbs 上面起了个 nginx，上面还有 zbs 侧的代码没有剥离出来。

2. 拆分 AccessManager 中的 recover cmd 入队的逻辑，做这个事的时候顺带发现单测里面的 BUG，在 RecoverManagerTest.RebalanceMoveParentInHighLoadMode 单测中没有为 chunk 创建 Session，实际上触发 recover cmd alloc owner fail，但这条 recover cmd 还会被放入每个 extent 的命令列表中，所以单测能通过，相当于负负得正了。

   然后做这个事也顺带发现 RecoverManagerTest 这个单测类的 Setup 里面用 3402 的端口创建了个 session，但后续写单测的人应该并不清楚他们的 chunk 要用到 3401 的端口，以及创建 chunk 的 session 用的端口应该得是 chunk 的 data port。比如单测中如果注册 4 台 chunk 用的是 3402 3412 3422 3432 端口，其实一共调用了 5 次 CreateSession，创建了 5 个 session follower，但这里面其实有 2 个都是用的 3402，造成创建第二个 3402 时，将 session_ 中第一个 3402 的 session 覆盖掉了，进而导致测试退出时，遍历 session_，只退出了 4 个 session follower，而第一个 session follower 没有退出。出现了 memleak。

3. lianpeng 修的一个 bug，2 点那会你刚 +2，这个问题是 Session Lease 从 14s 升级到 7s 的升级兼容问题。在 SessionMaster 初始化的时候开一个定时器每 200ms 查一次所有 session 的 lease 租约，如果所有的 session lease 租约都是 7s，那么才把 7s 租约作为配置信息持久化到 db 中，关掉定时器，这里就出现了在定时器的执行函数中析构定时器他自己的情况，lianpeng 的做法是把析构定时器的操作放在主 loop 中，相当于有 2 个 coroutine，第一个执行完才会执行第二个。

   但我好奇的点是为啥要用定时器去做 lease 租约升级兼容问题，升级的过程是先重启 meta follower 最后重启 meta leader，然后再去重启 chunk，那等到 meta leader 重启的时候，其他所有 meta 都升到最新的了，他们此时就应该是 7s ？

   chunk 正常的情况下，meta 变更，chunk 能感知到，他会主动把之前的旧的 session 撤回的，

   故障、新旧混杂、ability manager

   升级的代码可以参考 iscsi 能力协商的升级过程、空间分配策略调整这两个 patch，这是集群粒度的升级（meta/chunk）更难，更具有参考性。

   因为如果升级一半，集群中出现既有 7s 又有 14s 的 lease 租约，会有什么影响？统一按照 14s 来处理，如果 meta 没有切换，每个 session 记录的 expired time 是对的。（疯狂切换的话是有可能出现 7s 工作，7s 不工作的情况，不过此时 meta 也没法工作，整个集群都没法工作，也没事）

   linux perf 可以看火焰图，看日志发现的 topo distance 计算那边算的慢（由于锁才慢），才做的缓存，是这么发现性能差的。

   因为可能出现升级完但老 session 还在的情况。

4. 在 recover manager 中使用 topo 接口前先建立 topo 缓存，待 review；去重会让 extent 数量狂增，这边可能会有性能问题（是从未来的设计考虑，能够估计出这个问题）

   meta 中哪些需要精确的，哪些可以是模糊的（migrate/recover doscan），需要分开管理

   需要加单测/本地的 benckmark 体现出前后性能对比

5. 支持手动添加 recover/migrate 命令的 patch

   在 recover migrate 静态模式下的最大阈值检查调整，快改完了，今天会提交 patch；

   cmd slot 引入弹性模式，目前都是增量恢复，之前的经验值都是基于全量恢复给的，

   Zbs-25386 如果有个别慢盘，分到这个 chunk 上的 Recover cmd 会执行的慢，让 recover cmd 上限低一点。不然会达到 18min 的上限，造成 timeout

   支持手动添加 recover/migrate 命令是用于卸盘或啥时候想挪一下副本到指定文件时临时用一下，之后被 doscan 回去也没事。

   做完 prefer local 

   prefer local 节点上没有副本的入口有且仅有这 2 个：一个是高负载情况下，prefer local 的数据会被迁移。

   一个是虚拟机热迁移（用户操作、无法干预）且处于高负载，此时不会做 prefer local 的副本迁移。

   整体框架、整体原则、先不考虑细节，最终细节去满足原则，迭代的方式是一次次地明确需求。

   需求来源：1. 对未来功能的展望；2. 售后故事的展望。

   

   在中高负载情况下的迁移目前是允许迁移 prefer local 的，但根据 http://jira.smartx.com/browse/ZBS-13401 的需求，应该关掉。而不是在按目前的策略调整后，再去尝试本地化聚集（副本迁移来迁移去的，太动荡）；

   允许用户用命令行触发一个集中策略（向指定的节点聚集一个副本，不需要完整局部化，仅本地化即可），但是不能让指定节点进入超高负载状态（95%），这个事应该还得考虑不会被 doscan 给迁移回来吧？怎么做升级兼容性考虑呢？

   jiewei 提的需求是希望能够让用户指定对某个卷优先触发此类聚集操作，其实就是对这个卷的所有 pextent 做迁移，但这些迁移的 replace 和 src 要怎么选又要涉及到策略定制。

6. even volume 的 extent count 代表什么？为啥迁移时优先选择 count 大的 extent ？

   chunk_space 当支持 thick 模式时用 allocated_data_space，支持 thin 模式时用 provisioned_data_space

   * provisioned_data_space：Meta 已经分配给节点上持有的数据空间，Provisioned Space 和节点的实际可用空间可能存在差异，因为节点的实际使用空间有可能尚未回收，特别是 LSM 1 制作大量快照后删除时，但节点的实际使用空间最终会趋向于 Meta 分配给节点的数据空间；

       Provisioned Space，Chunk 属性，在 ZBS 中当一个存储对象（Extent）都按照其可能达到的最大体积计算时，不论数据是否写入，在物理节点上这部分空间已经归属于该存储对象，会被 ZBS 系统标记为已经消耗，不再挪作它用；

   * allocated_data_space：Allocated Space 虚拟卷/存储对象（Extent）/ Chunk 均有属性，代表已经真实分配的物理空间大小，已经分配给存储对象的空间将不再可以被分配给其他对象；

   * valid_data_space：健康的持久化存储层的空间（Partition），计算可用空间用到是这个参数；

   * used_data_space：Used Space，仅是 Chunk 的属性，在 ZBS 物理节点上真实存储的数据空间大小，如果一个 Volume 是 thick 模式创建的，那么在他真实写入数据之前，ZBS 系统会在关联的存储节点上为它分配物理空间，被标记为已预留和已分配。但是此时存储节点上只是预留了这部分空间给 Volume，直到 Volume 真实写入数据。存储节点上在对应的空间中真实的存储了 Volume 的相关数据之后这部分空间才会被标记为已使用；

   RecoverManager::GetMaxMoveForRebalance() 函数中的计算

   ChunkSpaceInfo 中每个字段的含义分别是什么？《数据存储空间概念》文档是否是最新的？我看上一次更新是在 19 年，你在 4.19 号有说会接下来会更新一版。chunk 节点内部的空间划分的两个图是否过时？

   ```protobuf
   message ChunkSpaceInfo {
       optional uint64 valid_data_space = 1 [default = 0];
       optional uint64 used_data_space = 2 [default = 0];
   
       // all provisioned space, for internal use, chunk should not set this field
       optional uint64 provisioned_data_space = 3 [default = 0];
       optional uint64 total_data_capacity = 14 [default = 0];
       optional uint64 failure_data_space = 15 [default = 0];
       optional uint64 allocated_data_space = 24 [default = 0];
       optional uint64 thin_used_data_space = 25 [default = 0];
   
       // id of the chunk, for internal use, chunk should not set this field
       optional uint32 id = 5;
   
       optional uint64 valid_cache_space = 11 [default = 0];
       optional uint64 used_cache_space = 12 [default = 0];
       optional uint64 dirty_cache_space = 13 [default = 0];
       optional uint64 failure_cache_space = 16 [default = 0];
       optional uint64 total_cache_capacity = 17 [default = 0];
       optional uint64 temporary_replica_space = 18 [default = 0];
   
       optional uint32 num_rx_pids = 21 [default = 0];
       optional uint32 num_tx_pids = 22 [default = 0];
       optional uint32 num_reserved_pids = 23 [default = 0];
   }
   ```

7. 梳理了中高负载模式下的 src/dst/replace 选取规则（也发现了一个 bug，中高负载模式下为了拓扑安全而做的迁移，replace_cid 永远不会选到 owner，报给 sijie 了），与低负载模式的并不相同。

   策略复杂的点在于要考虑 prefer local、lease owner、failslow、topo distance、节点容量、剩余 cmd 配额，场景的话要考虑 集群负载、是否双活、是否 even volume、是否精简制备、是否有过克隆/快照行为，暂时还不知道如何归类。

   如果要重构 recover manager，还需要考虑兼容 chunk 多线程（fengli 说之后让我做）、pin in perfermance 中高优先级卷（没看到 wenquan 的代码）、EC 单独写 recover manager 不纳入考虑，分层和去重不知道会不会影响（先不考虑）。

   must have ：过滤器

   shoud have ：排序

   根据影响范围从大到小写 if else，然后先过滤器再排序

   1. 集群负载、双活并列
   2. TBD

   恢复不满足

   可见、可调节、可人工干预

8. 配置中心的概念，应该怎么设计？给大家留什么接口呢？等 patch 都交完，最后完成。

   lease owner、prefer local、failslow



2023.5.23

1. 厚制备 elf 没有传递 Volume 所属的 Chunk id 给 ZBS，所以副本分配没法使用本地优先，此时第一副本会挑选为分配时集群空间比例最小的节点，然后其他 2 个副本在第一个副本的基础上再按照局部化原则选择。对应的 Extent 有真实 IO 之后会打上 prefer cid 的标记，再被定时扫描触发副本迁移；

2. 2 副本的数据分布，prefer local 是 2，副本分布一开始在 chunk 2 3，stop chunk2 之后，recover 从 3 到 4，副本分布是 3 和 4。此时在 chunk 3 上写 pid1，chunk4 上写 pid2，对应的 lease owner 分别是 3 和 4，chunk2 start，scan immediate，这个时候把 pid1 4 上的副本迁移到 2 上好理解，因为此时满足局部化/本地化期望副本分布是 2 3。

   对于 pid2，此时 lease owner 在 4，触发的副本迁移 dst 是 prefer local = 2（dst_cid 的选取规则是优先选择没有副本且不处于 failsow 的 prefer local，次选符合期望分布，即符合 LocalizedComparator 分配策略的下一个副本），replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid；那在这里因为 4 是 lease owner，3 又符合以 2 为 prefer local 的分布期望，所以 replace_cid 应该是 0，也就是不触发 pid2 的副本迁移才是，当 lease owner 过期（owner = 0），此时就可以将 pid2 在 4 上的副本迁移到 2 了。

   （跟着这个 case 理清了 src/dst/replace cid 的选取规则）

3. 加入和未加入机架的节点共存的场景，高负载做数据均衡的时候，数据理论上不会分配到未加入机架的节点，实在没有在机架中的节点可以选的时候会选到未加入机架的节点。把未加入节点视为同一个 rack 的一个 brick。

4. 双活集群情况下，初次分配副本的逻辑是先在 prefer local 所在可用域按照拓扑顺序分配 2 副本，然后再在远端可用域分配距离上一次选择更近的 chunk 分配 1 副本。副本迁移的话是分情况讨论：

   1. 副本如果被写过的话，是选择业务虚拟机当前接入点所在可用域作为数据优先写入的可用域，数据会自动向该可用域聚集，达到副本 2: 1 的效果；
   2. 如果是厚置备且从未写入的数据，将迁移到集群默认的优先可用域（用 zbs-meta session list 可以查看节点的 zone id，zone id = default 说明在集群默认的优先可用域）。（没有 prefer local 信息，所以给到 默认的优先可用域）

   这个是在双活文档中找到的，meta in zbs 等并没有（文档没有聚拢），代码里面这个逻辑在哪呢？

   双活集群的考虑感觉是一个挺复杂的事（双活集群是未来准备有，金融/医疗行业对其需求很高，目前我们的客户就 10+，规模比较小，只放核心数据）

5. migrate 时，replace_chunk 会避开 prefer_cid，即使 prefer_cid 处于 failslow。实际上 replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid； 而 prefer_cid 一定是在期望分布里的，所以 prefer_cid 一定不会作为 replace_chunk。

   另外一个考量是，虽然此时 prefer_cid 是 failslow 的状态，但他此时只是网络差，还是能正常读写本地磁盘，那么此时如果有读操作，直接读本地磁盘就返回，如果有写操作，由于多副本机制需要走网络写其他节点，会造成写性能很差。而如果因为 failslow 把 prefer_cid 的副本踢掉，那么本地没有副本，此时读写操作都要走网络，会造成读写性能都差，比前面那种情况更糟糕。

   这个问题应该由 elf 那边来解，当节点 failslow 的时候，vm 应该要自动调度到非 failslow 的节点，这样 prefer local 后续也会变更（需求已经提给他们了）

   这个过程也把 chunk 服务隔离的文档看了，感觉基本能看懂，然后发现这只是网络故障处理的一部分，网络故障处理又只是 ZBS 故障处理机制的一部分（磁盘故障、OS 与节点异常、ZBS 相关的服务如 chunk/meta/zk/taskd/io reroute 异常），这个文档含金量很高。

6. access_manager 跟 recover_manager 中日志不一样的情况，recover cmd 最终是通过 access manager 的 keepalive 下发到 chunk 上执行，在这之前会给这条 recover cmd 指定 lease owner（分配的优先级是 1. 该 pid 已有的 lease owner；2. src_cid；3. dst_cid；4. 从非 slow_cids 中根据 [owner priority](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.2hivgtf3odem) 选一个 cid），

   并且如果此时 lease owner 跟 src_cid 不同，且跟 dst_cid 不同，且 lease owner 上有活跃副本（说明它是健康的），会把 recover cmd 的 src 修改成 lease owner。

   这个挺关键的会变更 recover cmd src 的逻辑被写在 AccessManager::EnqueueRecoverCmd() 方法里，比较隐秘， yutian 5 年前写的代码。

7. 允许 RPC 产生恢复/迁移命令，可以指定源和目的地和（可去掉副本），在运维场景或许会有用。

   输入 pid, src, dst 

   输出 pid, current loc, active loc, dst, owner, mode

   搜索 AddRecoverCmdUnlock

   对于 pid src dst 有限定规则，外部 rpc 触发的 recover/migrate 优先级应该要更高，插到队列第一条，

   recover/migrate 要分开讨论，migrate 要额外指定 replace chunk

   做一次冲突检查，合法且和当前恢复不冲突，prefer local 和 topo 相关的不管

8. zbs cli 显示集群整体的负载情况？目前是用 zbs-meta chunk list 看每个节点的负载然后自己手动算

9. 副本策略和 recover，access manger/ recover manager，

10. 在看敏捷恢复和临时副本的文档，2 年前看不懂现在才发现 hping 写的文档质量那么高。

11. 觉得我最近哪里不行？对我有什么建议？Recover 相关的 patch 修完开始做去重还是分层的工作？

12. 先做分层、然后做 EC、分布式改造、然后去重

13. 7 月会带上 pin in performance，

14. 7 8 月才能结束 recover 相关的东西吧，参与去重的子任务



2023.4.16

未来开发重点：分层设计、EC、去重、元数据共识改造

分布式改造或分层

性能改进：单节点 55w、fengli、yuhua、haoqian、tianze

年底 EC：jiewei、sijie、lianpeng、kaiqi、haiwei

引擎：wangsai、dongdong、yangliang、shihao

元数据改造：fanyang 

分层：weipeng

warmup 作为入口，未来给分层改进

出一个大概的设计文档，然后接到的人出一个详细的设计文档

recover 策略改进，10 个 Jira，一个月之内做完，多了解前因后果

recover 看文档，

看一遍 recover 相关的代码

去重和 EC 依赖于分层结构

元数据改造的大文档 

预计到明年年底结束，今年再招聘 0-2 个人

