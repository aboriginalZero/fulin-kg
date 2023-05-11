

允许 RPC 产生恢复/迁移命令，可以指定源和目的地，在运维场景或许会有用。

输入 pid, src, dst 

输出 pid, current loc, active loc, dst, owner, mode

搜索 AddRecoverCmdUnlock





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

疑惑

1. ZBS 中的一个 extent，读的同时不允许写？为啥不用多版本机制来管理（块存储覆盖写的原因？还是块存储没必要提供）
2. 代码中 [(zbs.labels).as_str = true] 的意思
3. gtest 如何开启 VLOG DLOG 部分的日志
4. meta in zbs 中 Volume 部分 origin_id 中示例继承树怎么看？
5. metaDB 中的 Vtable 表存储 Volume -> Extent 的关联关系
6. 对着 log 梳理一下 zbs 架构，角色在什么位置，哪些要持久化到 metaDB，关键数据结构的类型
7. CreateSession 这个过程 SessionMaster 做什么了



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



