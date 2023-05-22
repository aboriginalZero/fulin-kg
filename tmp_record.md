Access 在进行读/写 IO 前先进行一次 Sync Gen，确认所有副本当前的 Gen 是一致的。





// TODO 看一下这 3 个方法是怎么构造  recover cmd 的

RecoverManager::GenerateRecoverCmds()、RecoverManager::GenerateMigrateCmds()、RecoverManager::AddSpecialRecoverCmd()



RecoverManager::AddRecoverCmdUnlock()

1. 若这个 pid 上有 recover cmd，返回 false（每个 pid 上任一时刻只能有 1 条 recover cmd）；
2. 若这是一条临时副本的 recover cmd，且 src_tmp_replica 的正确版本（通过对比 epoch 确定）是否已经在 pid pextent 上，返回 false；
3. 若 src 中没有 pid 或者 migrate dst 已经有 pid，返回 false；
4. 确保 dst 上有足够的空间（能否再放一个 extent，区别对待 thin/thick），更新 dst 的 rx_pids、replaced 的 tx_pids、src 的 recover_src_pids 以及他们的占用空间；
5. 调用 AccessManager::EnqueueRecoverCmd() 生成 recover cmd 并放入对应命令队列中
   1. 通过 AccessManager::AllocOwnerForRecover() 分配 recover/migrate 的 lease Owner，与 AccessManager::AllocOwner() 由用户 IO 触发的 Owner Alloc 逻辑不同，分配的优先级是 1. 该 pid 已有的 lease owner；2. src_cid；3. dst_cid；4. 从非 slow_cids 中根据 [owner priority](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.2hivgtf3odem) 选一个 cid；
   2. 若此时 lease owner 跟 src_cid 不同，跟 dst_cid 不同，且 lease owner 上有活跃副本（说明它是健康的），为了避免 recover/migrate 的读走本地而非网络，会把 recover cmd 的 src 修改成 lease owner；
   3. 根据待恢复/迁移副本的 pid 和经过 1 2 步选出的 lease owner，构造 lease 并放入 recover cmd 中，接着将 recover cmd 放入 lease owner 的那个 recover cmd 队列（Access Manager 为每个 session 维护了一个  recover cmd 队列，通过 lease owner 的 uuid 获取。



集群内部扫描副本状态，副本迁移相关代码

RecoverManager::DoScan()，正常情况下 60s 检查一次，也可以立即扫描如接受 rpc 请求

RecoverManager::ReGenerateWaitingMigrateList()，分别处理均匀分配的副本和有本地化偏好的副本，他们会有不同的初次分配和迁移策略

RecoverManager::ReGenerateMigrateForRebalance()，针对有本地化偏好的副本

1. 当处于低负载时 

   RecoverManager::ReGenerateMigrateForLocalizeInStoragePool()

   RecoverManager::RepairPExtentForLocalization()，满足如下任一条件不触发迁移：1. 副本的 prefer local cid 不是健康状态（status == CHUNK_STATUS_CONNECTED_HEALTHY）；2. 这个 pid 存在临时副本； 3. replace cid 和 dst cid 都是 0；4. dst_cid/src_cid 上被下发的 recover 命令超过 200 条；5. src_cid 上的有限空间不足 256 MB；

   否则通过 RecoverManager::GetRepairCid() 选取 replace_cid 和 dst_cid，初始值为 0，具体规则如下：

   1. 如果 prefer local 有该副本，并且其他活跃副本满足本地化/局部化的分配策略，说明副本满足分配规则，不触发迁移；
   2. dst_cid 的选取规则是优先选择没有副本且不处于 failsow 的 prefer local，次选符合期望分布（即符合 LocalizedComparator 分配策略） 的下一个副本；
   3. replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid； 

   选定 dst_cid 和 replaced_cid 后，src cid 的选择规则是默认值为 replace cid，但如果 replace_cid slowfail 或者和 dst_cid 不在同一个可用域，那么会从活跃副本中找跟 dst_cid 在同一个可用域且不是 slow fail 的 cid 作为 src_cid。

2. 当处于中高负载时

   1. 如果集群处于极高负载时，进行容量再均衡扫描

      RecoverManager::ReGenerateMigrateForBalanceInStoragePool()

   2. 否则进行拓扑安全扫描

      RecoverManager::RepairTopoInMediumHighLoad()

      RecoverManager::RepairPextentTopo()

      1. 非双活集群

         RecoverManager::RepairInZone() 

         RecoverManager::GetSrcAndReplace()

      2. 双活集群

         RecoverManager::RepairInStretched()

         RecoverManager::MoveToOtherZone()

         RecoverManager::GetSrcAndReplace()

      

IO 下发的流程

NFS/iSCSI/nvmf -> ZBS Client -> access io handler -> generation syncor -> recover handler



后续读一下临时副本的内容，

允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。

输入 pid, src, dst 

输出 pid, current loc, active loc, dst, owner, mode

搜索 AddRecoverCmdUnlock





疑惑

1. 觉得我最近哪里不行？对我有什么建议？Recover 相关的 patch 修完开始做去重还是分层的工作？
2. ZBS 中的一个 extent，读的同时不允许写？为啥不用多版本机制来管理（块存储覆盖写的原因？还是块存储没必要提供）
3. 代码中 [(zbs.labels).as_str = true] 的意思，slack 中 qiuping 提过，
4. gtest 如何开启 VLOG DLOG 部分的日志
5. meta in zbs 中 Volume 部分 origin_id 中示例继承树怎么看？
6. metaDB 中的 Vtable 表存储 Volume -> Extent 的关联关系
7. 对着 log 梳理一下 zbs 架构，角色在什么位置，哪些要持久化到 metaDB，关键数据结构的类型
8. CreateSession 这个过程 SessionMaster 做什么了

zbs cli 如何快速查看集群负载情况？

Meta 与 chunk 如何交互？问题来源[减少数据恢复量](https://docs.google.com/document/d/1rDN0bNa-Dw6xo9yCN_gtVg1qrrDx1LVzc5_Dz_23dW8/edit#)

1. 为什么需要 Extent Lease
2. 为什么第一次拿到 Lease 之后需要 Sync Generation
3. 为什么 Meta 需要保存一份 generation，什么时候会使用和更新这个 generation
4. ZBS 如何保证多个副本之间的一致性
   1. Meta 如何分配副本
   2. ZBS 如何进行写操作
   3. 当有副本发生错误时，ZBS 如何保证数据安全
   4. Meta 中记录的副本和 LSM 中的本地副本之间的关系
   5. Meta 中记录的 location 和 alive_location 的含义
   6. 当 snapshot 和 clone volume 的时候 Meta 和 LSM 的元数据发生了什么
   7. Sync Gen 失败的副本是否必须立即从 meta 中剔除
   8. 写失败的副本是否必须立即从 meta 中剔除



可能有多个 zbs client，但一定只会有一个 access，一个 lease owner 来保证接入点的唯一性



命令行中 Replica 和 Alive Replica 的区别？前者是 pextent table  中记录的，后者是跟随心跳上报动态变动

```
# 在主分支上
git pull
# 将新的 URL 复制到本地配置中
$ git submodule sync --recursive
# 从新 URL 更新子模块
$ git submodule update --init --recursive
```

双活集群相关问题

虚拟机创建在次级可用域，应该是当前可用域 (次级可用域) 2 副本，远端可用域(优先可用域) 1 副本吧。

副本如果被写过的话，是选择业务虚拟机当前接入点所在可用域作为数据优先写入的可用域，数据会自动向该可用域聚集，达到副本 2: 1 的效果。
如果是厚置备且从未写入的数据，将选择集群默认的优先可用域（用 zbs-meta session list 可以查看节点的 zone id，zone id = default 说明在集群默认的优先可用域）。

这个是在双活文档中找到的，meta in zbs 等并没有，代码也不知道怎么看



用 uint64_t 来声明 ring_id

初始时生成符合 topo_id 次序的 ring_id，遍历一遍

当有任意一种拓扑行动时，重新生成 ring_id

ring_id 的重新生成是按不同 brick/rack 中节点数量的比例配对，并且要尽可能保持原来的相对次序

不管是首次按照 uuid 生成还是后面按照 ring_id 生成，都要重新考虑 以 brick 为粒度算节点数



双活集群中优先可用域 2 副本，次级可用域 1 副本。

UpdateTopoObj/RegisterChunk/RemoveChunk

以 brick 为粒度算节点数，找到 3 个 brick 中最小的节点数，记为 k，对其他两个 brick k 等分

按 brick 编号挑 3 个节点



removeChunk 里面竟然没调用 DeleteTopoObj

Chunk_manager 对外暴露 CreateTopoObj / Delete / update / show / list

CreateTopoObj 只允许创建 rack/brick，不能直接创建 Node（在 RegisterChunk 中实现）

查看节点拓扑 GetChunkTopology（根据 cid 查的）、ShowTopoObj（根据 topo_id 查的） ；

UpdateTotoObj 是允许更新 Node 的 TopoObj，节点的拓扑位置移动也是在这里

1. 节点加入 RegisterChunk。此时没有 ring_id，需要生成；
2. 节点退出 RemoveChunk。此时删除了对应的 TopoObj，之后还需要引发重新生成 Ring_id；
3. 节点移动 UpdateTopoObj，Node 类型的 parent_id 被修改为其他 brick_id/CLUSTER，此时需要重新生成 ring_id；
4. Brick/Rack 位置移动 UpdateTopoObj，如 Brick 从一个 Rack 移动到另一个 Rack（parent_id 变化）也要重新生成 ring_id；

加入新节点，由于存在没有 ring_id 的情况，所以按 topo_id 生成 ring_id

节点退出，由于此时一定是所有节点都有 ring_id，所以按照之前 ring_id 的相对顺序生成这一轮



机器重启后，zk 要手动重启，selinux 要关闭



ZBS RPM 和 SMTX ZBS/SMTX OS/IOMesh 等不同产品的产品版本号从 5.0.0 开始已经分离，各有各的版本号


非首次运行单测

```shell
# 编译
docker run --rm --privileged=true -v /home/code/zbs:/zbs -w /zbs/build registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 ninja zbs_test

# 屏幕中会提示出错处的日志信息，借助 newci 可以避免在本地配置 nvmf/rdma 环境跑单测
# 但是要配好 nvmf/rdma 的相关依赖包/服务
cd /home/code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"

# 运行后的测试日志默认保存在 /var/log/zbs/zbs_test.xxx 中 
cd /home/code/zbs/build/src && ./zbs_test --gtest_filter="*FunctionalTest.WriteResize*"

# 显示指定 main 分支
git review main

# 自动修改格式后再编译
docker run --rm --privileged=true -v /home/code/zbs:/zbs -w /zbs registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 sh -c 'sh ./script/format.sh && cd build && ninja zbs_test'
```



目前 ZBS 仅支持以多副本方式存储数据，适用于对读写延迟比较敏感的业务场景。EC 相比副本占用更少的存储空间，同时提供与副本同等甚至更高的容错能力，其代价是更新或者故障数据恢复的性能略差（注：写不一定比副本差，虽然需要多一次读，但数据量变少了。看最终实现的效果）。EC 非常适合归档、备份等数据量较大且更新很少的业务，也适用于对延迟不敏感而对带宽敏感的业务。需注意，EC 的目标是为了保证完整性而非正确性，只可以用于恢复丢失的数据，而无法修复位翻转等数据正确性问题。需要有其他机制对数据篡改进行检测，例如 CheckSum 检测，被篡改的数据可以视为丢失，再通过 EC 做数据恢复。

ZBS 副本机制采用 Lease Owner + Generation 方式实现数据一致性。1. 通过 Lease Owner，所有的 IO 都被顺序化，客户端的多读者多写者模型被转化为单读者单写者模型；2. 通过 Generation 判断多个副本是否一致，当一个副本成功写入一个 IO 后，Generation 自增，每个 IO 携带当前的 Generation，Chunk 只有在 Generation 匹配时才允许 IO 处理。

https://blog.csdn.net/qq_24406903/article/details/118763610


IO 流

Access 提供的是外部客户端进入 ZBS 系统内的接入点功能。在数据请求达到 Access 后，它将负责把它转化为 ZBS 内部请求。处理的基本过程如下：

1. 直接接入的 Access 如果之前从未处理过对应数据块（Extent） 的数据请求，则会首先向 Meta 请求 Extent 的基本信息（副本分布，权限）。如果 Access 最近已经访问过（1 小时内），则无需这个步骤；

2. 在获得数据访问权限（如果本地的 Access 不持有 Extent 的访问权限，则会转发至持有权限的 Access 继续后续步骤）和基本信息之后：

   1. 读请求：本地 LSM 在副本列表中，Access 会优先访问本地的 LSM （无需经过网络，本地的内存交换即可），如果成功读取则直接访问，失败则继续尝试逐一其余副本直至成功或者全部失败；

   2. 写请求，并发的向所有副本发出写命令，确认所有副本均写入成功才返回，如果部分失败，则执行副本剔除，保证副本列表中的所有副本数据一致，如果全部失败，则不剔除，返回异常等待上层重试；

      TODO 这部分代码可以看一下，是收集到所有副本的写结果后才下发指令还是只要收到任意一个副本写成功就可以？


FunctionalTest::SetUp()  --> new MiniCluster(kNumChunks);



gtest系列之事件机制
“事件” 本质是框架给你提供了一个机会, 让你能在这样的几个机会来执行你自己定制的代码, 来给测试用例准备/清理数据。gtest提供了多种事件机制，总结一下gtest的事件一共有三种：
1、TestSuite事件
需要写一个类，继承testing::Test，然后实现两个静态方法：SetUpTestCase方法在第一个TestCase之前执行；TearDownTestCase方法在最后一个TestCase之后执行。
2、TestCase事件
是挂在每个案例执行前后的，需要实现的是SetUp方法和TearDown方法。SetUp方法在每个TestCase之前执行；TearDown方法在每个TestCase之后执行。
3、全局事件
要实现全局事件，必须写一个类，继承testing::Environment类，实现里面的SetUp和TearDown方法。SetUp方法在所有案例执行前执行；TearDown方法在所有案例执行后执行。
例如全局事件可以按照下列方式来使用：
除了要继承testing::Environment类，还要定义一个该全局环境的一个对象并将该对象添加到全局环境测试中去
原文链接：https://blog.csdn.net/ONEDAY_789/article/details/76718463



linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610



centos7 中将 pip 默认是 8.x，升级到 20.3.4（更高版本不支持 Python 2.7）

```
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
```

pip 根据 whl 文件安装包

```
pip install xx.whl
```

git_review 插件安装 1.28.0，进 pypi.org 找包的各个历史版本

```
pip install git-review==1.28.0
```

pip 更改镜像源

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



