### 任务要求

> ZBS-10973：taskd: support specified runtime

Timemachine 支持限定任务的运行时间，仅在指定的时间内运行跨站点备份任务。开始区间与结束区间，任务只能在这两个区间之内运行，超过结束区间后，就不能再分配新的任务。仅指定时间，不指定日期。

主要点是 RPC 来动态调整运行时状态的，dispatcher 直接发送 rpc 给 runner 来更新指定的运行时间。

### 从中所得

1. 理清 epoll + socket +  rpc + zk + leveldb 的使用逻辑，算是对分布式开发有一个具体的认识了。

2. coredump 可以显示 exit 现场的内存使用情况，有利于定位问题根源，如果只是打 LOG + stop，只能看到表面有这么个问题，无法去分析是造成这个问题可能的原因1 2 3 中的哪一个。

3. 判断是否需要加锁的 2 个条件

   1. 多个 thread 同时修改同一个 var；
   2. 一个 coroutine 访问 var 期间主动 yield，此时其他协程拿到 CPU 并修改 var

   如果代码逻辑是单向的，没有并发（不论是线程的真并发还是协程的逻辑并发）就不需要互斥。


### 残余问题

1.  只有 copy volume 会走 OnRequest、OnResponse，其他 StatusService 的方法就不会唉，这个跟总结的还有点出入？有空再整理一下

2. CoAsyncRpcClient::~CoAsyncRpcClient() 中为何要给 loop 添加 TimerFd 的事件？

3. RpcServer::OnRead 中当读取 header 超时，为什么要把 error_code 由 ETimedOut 改成 ESocket？

   代码注释说的是 header 缓冲区会被重用，ReadHeader 会用之前读到的部分 header 内容重写 body，还不是很理解，延伸一下，rpc 中的网络异常都有哪些？超时、socket 失去连接，还有啥？

4. TimerHandle 是个定时器，那么是在刚设置好就会执行第一次，还是在设置完过 n 秒才执行第一次？

5. zbs 中的 Coroutine 是串行、非抢占式的吗？是的

   Coroutine::Self() 的意义是什么？检查当前是不是在一个 coroutine 环境里运行，因为代码也可以直接在线程中没有套着 coroutine 跑，所以这个检查有意义。以及 CoMutex 是为了避免同一个资源被多个 coroutine 同时使用。原理是拿不到锁就主动 yield。

6. 单测中开一个 EventLoop 传给 TaskClient 表明要一个按需运行的 task rpc client，此时并没有给他一个 coroutine 环境，继续执行单测中后续的代码。当调用 CALL_TASK_SERVICE 时，coroutine 检查是 false，此时主动开一个 coroutine 并 enter，存在这个 coroutine 需要执行网络 IO 主动 yield 的情况，那就继续执行 enter 之后的代码。此时只要 co 还在，就会循环触发 loopOnce，直到 co 结束生命周期。

7. 单测中以及 task_client 中为什么要对 stub 套一个 unique_ptr 的壳呢？

   ```c++
   CopyVolumeService::Stub *stub = nullptr;
   ASSERT_STATUS(task_client->GetServiceStub(&stub));
   std::unique_ptr<CopyVolumeService::Stub> stub_ptr(stub);
   ```

   在这个简单的测试环境下也可以手动 delete，像这种内部申请内存，外部调用并释放的做法其实很常见。比如工厂方法，初始化模式（工厂）创建资源， 外部使用外部释放，资源申请和释放有固定的范式也不容易疏忽导致内存泄漏。

8. zk_db, task_db 干嘛用的？runner 中只有 ZkTaskDb，dispatcher 中有 TaskDb，其包含 ZkTaskDb。它们的作用是用来做缓存，提高查询效率。

   当 TaskServer 启动的时候，Dispatcher 会加载所有 task 到 TaskDb 中，也就是用 leveldb 构造的一个 local cache，由于它会跟 ZkTaskDb 同步，所以查询上能够为 ZkTaskDb 减少负载。

   Dispatcher 和 Runner 都在使用 ZkTaskDb，本质上就是为每个 task 在 /zbs/taskd/db/task_id 上创建一个临时节点，存放对应的 task 信息还有 runner 地址，以及可以为 task 设置 TaskWatcher，用于响应一些 zk 上的事件。

   * Runner：用于更新任务进度、所在 runner 地址

   * Dispatcher：查看任务进度、更新 cached task、runner 子节点变动调度到其他 runner 上。

     具体而言，在 TaskDispatcher::TaskWatcher 中：

     * 如果 task state 与 cached_task state 不一样的话，就把 ZkTaskDb 上的信息同步到 leveldb；

       > 插一句，这里同步的时候不需要加锁吗，不然无法保证从 leveldb 上总是读到最新的数据？包括 TaskDb 自己在 PutIntoCache 更新数据的时候也不用加锁吗？

     * 如果 task_id 路径下记录 runner 的子节点没了的话，Schedule到 waiting_task 上待分配新 runner

9. pyzbs 怎么使用 zbs 中的 RPC 服务？TaskDispatcher 怎么跟 CopyVolumeService 关联起来？

   * TaskDispatcher::Initialize 中 rpc_server.RegisterService(this)，说明把 StatusService 注册进 TaskRpcService；

   * TaskServer::RegisterDispatcherServices() 中 dispatcher->RegisterTaskService(copy_volume_dispatcher) ，copy_volume_dispatcher 是一个继承自 CopyVolumeService 的 TaskCopyVolumeDispatcher 对象，说明也把 CopyVolumeService 注册进 TaskRpcService 

   TaskDispatcher 根据被注册的 task_service 对外提供 RPC 服务，所以 pyzbs 通过 dispatcher IP + port 的 CopyVolume 服务来开启一个 CopyVolume 任务。

10. 任务怎么从 TaskCopyVolumeDispatcher 派发到 TaskCopyVolumeRunner 的？

   在 TaskDispatcher::ExecuteTask 中，通过 GetTaskClient 得到一个指向 runner 的 TaskClient。将 TaskCopyVolumeDispatcher 的 CopyVolume Task 通过  TaskClient->CallMethod() 转发到 TaskCopyVolumeRunner 的 CopyVolume 了。

   TaskClient 的 CallMethod 通过 CoAsyncRpcClient::CallMethod 实现，看了下代码，是比较裸的写法，自构建 header 和 body 通过 ProtoAsyncClient 发送出去，而不是获取 ServiceStub 再发送，这样做有什么优势吗？

   > 可能需要追 CoAsyncRpcClient 的代码，并跟用 ServiceStub 的方式比较，优缺点分别是啥？

### 需求分析

当 CopyVolume Task 创建的时候支持设定 runtime，同时支持后续用 SetRuntime 方法来修改。在 proto 中为 CopyVolume 增加一个 runtime 字段，并编写 grpc + protobuf 相关代码。

### 解决思路

**pyzbs 调用 zbs 提供的 CopyVolume 服务的执行流程**

1. TaskDispatcherRpcServer::OnRequest()，等待返回

2. TaskCopyVolumeDispatcher::CopyVolume()，参数校验

3. TaskDispatcherRpcServer::OnResponse()， 把 CopyVolume task 放进 waiting_tasks ，给调用方返回 response

   > 这边返回的 response 不一定能保证后面 task 被正确执行了，也就是存在返回给 pyzbs  CopyVolume task 创建成功的 response，但却在后面的步骤中出错 pyzbs 却不知道的情况，现有策略是怎么应对的呢？

4. TaskDispatcher::CheckTask()，每 2s 以 Coroutine 的方式运行 waiting_tasks 里合适的 task

5. TaskDispatcher::ExecuteTask()，分配一个 runner 并以此构造 TaskClient

6. TaskClient::CallMethod()，不用获取 ServiceStub 的 rpc 方法

7. CoAsyncRpcClient::CallMethod()，自构建 header 和 body 通过 ProtoAsyncClient 发送出去

8. CoAsyncRpcClient::SendRequest()，其中调用 AsyncSender::SendV() 将数据放在缓冲区上，然后通过 系统调用 eventfd_write 触发 send_efd 的写事件，数据就发送出去了。

**pyzbs 调用 zbs 提供的 SetRunTime 服务的执行流程**

1. TaskDispatcherRpcServer::OnRequest()，等待返回
2. TaskDispatcher::SetRunTime()，更新 task_db
3. TaskClient::SetRunTime() ，通过 zk 获取到要修改的 task 所在的 runner 构造的 TaskClient
4. StatusService_Stub::SetRunTime()，调用 grpc 
5. TaskRunnerRpcServer::OnRequest()，立即返回
6. TaskRunner::SetRunTime()，透传
7. TaskRunnerRpcServer::SetRunTime()，更新 zk_db
8. TaskController::SetRunTime()，通过跟 task_id 绑定的 controller 来传递要修改的 runtime 信息，Copy Volume Task 在 runner 上运行的时候，会在每做完一个 ExtentCopy 都会判断一下当前时间是否在运行时间区间
9. TaskRunnerRpcServer::OnResponse()，不返回 response
10. TaskDispatcherRpcServer::OnResponse()，返回 response

**常用 zbs cli**

```shell
zbs-meta pool list
zbs-meta pool create <pool_name>
zbs-meta volume list <pool_name>
zbs-meta volume create <pool_name> <volume_name> <volume_size>
zbs-chunk partition format /dev/vdd1
zbs-chunk partition mount /dev/vdd1
zbs-task task list_by_status unfinished
zbs-task copy_volume copy <vola_id> <volb_id> 10.1.242.102:10201:10206 10.1.242.102:10201:10206 0 0 1 1
zbs-task task set_runtime <task_id> <start_hour> <start_min> <end_hour> <end_min>
zbs-task task cancel <task_id>
```

**pyzbs 开发流程**

```shell
# 开发机上安装好 protobuf2 并正确设置软链接和 python-protobuf 包
yum install autoconf automake libtool	gcc-c++ protobuf-compiler # 安装依赖
wget https://github.com/google/protobuf/archive/v2.6.1.zip
unzip v2.6.1.zip -d ./
cd protobuf-2.6.1
# 如果无法连接 google 网站，先下载再把 autogen.sh 脚本中相关部分注释掉
wget https://github.com/google/googletest/archive/release-1.5.0.tar.gz
tar xzvf release-1.5.0.tar.gz
mv googletest-release-1.5.0 gtest
./configure prefix=/opt/protobuf2 
make && make install
# 软连接地址取决于 pyzbs/gen_proto.sh 中的设置
ln -s /opt/protobuf2/bin/protoc /usr/local/bin/protoc

# 将 zbs 中的 task.poroto 复制到开发机的 /usr/share/zbs/proto
cp /zbs/src/proto/task.proto /usr/share/zbs/proto
# 生成 py_proto
sh pyzbs/gen_proto.sh
# 复制生成、修改的文件
scp pyzbs/zbs/proto/zbs_task_pb2.py smartx@192.168.91.186:/tmp
scp pyzbs/zbs/task/cmd.py smartx@192.168.91.186:/tmp
scp pyzbs/zbs/task/client.py smartx@192.168.91.186:/tmp
scp pyzbs/zbs_rest/api/v2/task/views.py smartx@192.168.91.186:/tmp
scp pyzbs/zbs/lib/ztype.py smartx@192.168.91.186:/tmp

yes | cp /tmp/zbs_task_pb2.py /usr/lib/python2.7/site-packages/zbs/proto 
yes | cp /tmp/cmd.py /usr/lib/python2.7/site-packages/zbs/task/
yes | cp /tmp/client.py /usr/lib/python2.7/site-packages/zbs/task/
yes | cp /tmp/views.py /usr/lib/python2.7/site-packages/zbs_rest/api/v2/task
yes | cp /tmp/ztype.py /usr/lib/python2.7/site-packages/zbs/lib/
# 重启 rest-sever
systemctl restart zbs-rest-server
```

### 过程疑问

1. 对 pyzbs 的 restful API 测试

   * Headers 中放入 X-SmartX-Token 字段，可以用网页访问 91.186 的 F12 Network 选项卡中找到该字段
   * Body 中选 raw JSON 模式，以 form-data 模式的 request body 跟 raw JSON 不一样。

   ```c++
   // 创建一个 copy volume task
   POST 192.168.91.186/api/v2/task_center/copy_volume/create
   {
       "src_hosts": "10.1.242.102:10201:10206",
       "dst_hosts": "10.1.242.103:10201:10206",
       "src_volume_id": "611702ea-17c6-48a8-939a-89c4296b2010",
       "dst_volume_id": "881ecb4c-3570-4a1e-9ba7-331fdf0192e4"
   }
   // 更改 runtime
   POST 192.168.91.186/api/v2/task_center/copy_volume/b981be69-3dcb-4e19-94c3-01a54f5a5baf/set_runtime
   {
       "start_hour":1,
       "start_min":1,
       "end_hour":23,
       "end_min":22
   }
   ```

2. 存在 BUG

   现象1：当 pyzbs 调用 dispatcher 的 SetRunTime 方法时，由于需要立即返回 Task 作为 response，但此时的 response 是从旧的 zk_task 中复制而来的，其 runtime 字段仍是更新前的值，于是造成无法返回最新值。

   因为 dispatcher 要立即返回，所以 SetRunTime 通过 client 传给 runner 就立即返回，但此时 runner 还未把它更新到 zk_db 上，所以造成返回的 task 上 runtime 不是最新的。解决方案是让 dispatcher 来更新 zk，这样会有个问题就是 runner 不一定成功就返回设定成功，比如考虑这样一种场景 dispatcher 返回 client 成功，但是 runner 在查询 zk 时，所在节点失效导致实际上 task 并没有按最新的 runtime 运行。

   解决1：SetRunTime 时由 dispatcher 来更新 zk 并返回最新 task，runner 仅设置 controller 的 runtime 就好。

   现象2： 每当 CopyVolume updateProgress 时，由于 TaskController updateProgress 方法传入的 task 的 runtime 字段总是该 task 刚创建时的值，于是造成 runtime 字段总会被更新回初始值。

   解决2：在 UpdateProgress 中保留 runtime 相关字段不被初始值覆盖，并且可以用 progress_task_ 的 runtime 字段来管理 latest runtime，去掉 TaskController 的 runtime_ 成员变量。注意要在 UpdateProgress 中对 CopyVolumeTask 额外判断。

3. TaskDispatcher 中提供的 CancelTask、SetRunTime 等写操作需要通过 TaskClient  转发到 runner 上执行，所以需要 TaskClient 提供这些写操作，这个好理解。但是 ShowTask、ListTaskByDate 等读操作通过 task_db 就可以实现，为什么还要在 TaskClient 上提供这些读操作？

   TaskClient 中的读写操作是通过 ServiceStub 来调用相应 dispatcher/runner 的方法，取决于这个 TaskClient 构造时传入的 IP+port，我觉得它还要提供那些读操作是为了方便吧，在 grpc ServiceStub 方法的基础上做了层封装，仅此而已，肉眼可见的好处是，单测里面这么用代码量会少很多。

   或者说， 那些读操作 TaskClient 也可以不提供，只不过这些每次调用的话，都得手动去创建 request、response，比较麻烦。那还有个问题，TaskClient 为什么不提供一个通过 ServiceStub 来调用相应方法的 CopyVolume 接口呢？我觉得是 CopyVolume 是一个比较重的操作，写代码时显示构造 request、response 也挺合理的，不然就算提供这个接口，参数列表也会长的一批。

   所以也跟 dispatcher 中通过 task_db 实现的读操作不冲突，举个例子，TaskClient 的 ListRunner 操作最终还是会调用到 dispatcher 中的 ListRunner，而它会查询 leveldb 得到答案。

4. 为什么要走 rpc controller 而不是直接操控 task，CopyVolume 中也能拿到包含 runtime 信息的 request？

   在 PrepareContext 中把 task、controller 的内容都塞给 CopyVolumeTaskContext 了，除了这两个大头，还有cur_extent、next_block 等跟拷贝相关的变量。我的理解是通过 CopyVolumeTaskContext 把所有这些变量同一到一个数据结构来管理。

5. 集群中的时间差了得有 8 个小时，可能跟 localtime 有关。但是保证了跟 tower_license 计时方式一致，应该不会有问题。

6. 每次编译 zbs 时造成跟 bj-amd 上的 dev 机器网络断开

   应该是开启并行编译之后，内存不足的原因。

   编译卡住的时候，机器没有 reboot，只是 OS 卡死了，造成 ssh 不响应，并且这个过程持续 3-5 分钟，猜测应该是 OS 的自动保护机制在操作，它主动去 kill 占用大内存的进程，如果被杀死的进程是 cc1plus，gcc 会将其解释为进程崩溃，因此看到的现象是 ssh 无响应之后断开、编译也会停止。

   我那台用来编译的开发机是 64核、12G 内存的配置，我现在开到 16G 就不会出现这个问题了。

   在 /var/log 目录下看 message 内核日志，有 sshd invoked oom-killer

7. 为什么仅指定时间，不指定日期，避开业务繁忙时间，每天固定一个时段来执行备份。这个时间是跟 runner 绑定，所有 runner 共享两个参数 startTime, endTime，这两个参数是可以被 dispatcher 动态修改的。

   1. 这个跨站点备份任务用的 Copy Volume Service，这个 service 就是拿来做远端备份的数据复制。Rsync Service 是理论上比较优雅的一种方式，先在本地计算与远程的差异，然后只传输有差异的那部分数据，实际上为了节省这一点点带宽，却耗费了大量的 CPU 资源，所以被舍弃了。

   2. 记得考虑输入合理性（满足 timestamp 格式、startTime < endTime，头天晚上到第二天凌晨怎么表示 等条件）

   3. 要写一个 dispatcher 通过 rpc 来更新 runner 上的 startTime 和 endTime，这个怎么做？不需要像 UpdateProgress 那样借助 zk_db 吧？zbs 中已有的函数有没有可以借鉴的？zk_db 中的对象是 task，而 startTime, endTime 是 runner 的属性，所以应该是在 class task_runner 中加上两个的私有变量，默认值设为 0.0-23.59（应该只需要细粒度到秒吧），然后在 dispatcher 中提供一个函数，参数为 Address、startTime、endTime 三个，修改 task_runners_ 中对应 Address 的 startTime、endTime。但是这个函数谁来调用啊，另一个代码库中有调用的。

      既然所有的 runner 共享一个可执行时间区间，那么只要在 dispatcher 设置这个时间属性就行了。不过也有个问题，如果 runner 跟 dispatcher 之间的通信断了，runner 就无法及时停止了 

   4. 在节点刚成为 dispatcher 调度 task 时，就需要判断 runner time 跟 [startTime, endTime] 的关系，只会把 copy volume task 调度到 runner time in [startTime, endTime] 中。如果找了一圈都没能找到满足条件的 runner，怎么处理这个 task 的状态呢？标注为 failed 之后都不可能被运行了，canceled 语义上又是用户主动取消的，跟这里不吻合，是要再定义一个 task state ？不用，由于所有的 runner 的  [startTime, endTime] 都是一样的，所以没必要调度，如果不在时间区间内，直接 sleep，等到明天的同一时段再执行就是了。

   5. runner 每完成一个 extent 复制，都会 UpdateSyncState，那应该在这加上查询 startTime ，endTime 是否更改的判断，如果当前 runner time 不在该区间，就停下来。

8. 备份的时候，为什么要先生成快照再传输数据？如果没有快照，备份执行期间有数据变动，那么备份的内容也就跟用户预想的不同，通过轻量级的快照，定格用户想要备份的那个状态下的数据，就可以避免这种常见情况

### 概念补充

##### 备份中的重要模块

SMTX OS 备份功能的设计目标是为 SMTX 集群提供更为强大的数据容灾、恢复和回溯能力。备份功能以 SMTX 的快照功能为基础，可以将数据在不同时刻的快照副本自动复制至其他城市的目标站点中。在需要时，可以使用目标站点的备份数据对原始站点的保护对象进行回滚操作，并且在原始数据所在站点崩溃时，可以迅速地从目标站点中利用最近时刻的快照重建业务。此外，备份功能还提供了数据对象跨站点克隆的能力。

Time Machine 是备份服务的主体模块，负责保护计划的维护、任务调度、资源监控等，也负责在异地备份站点中创建受保护对象的镜像对象。它按照预设的保护周期在保护站点中创建快照组，再将快照组内的数据写入至异地备份站点的镜像对象中。在数据传输完成之后，它再对备份站点中的镜像对象执行一次快照，得到镜像快照组。用户需要时可以从保护计划的执行记录中找到对应的快照组，选择回滚或者远程重建恢复当时的数据状态。在每次成功执行保护计划之后，Time Machine 还将按照指定的保留策略，清理过期快照组。Time Machine 部署在所有节点上，follower/leader 都可以接受外部请求，统一转发到由 zk 选出的唯一的 leader 节点执行。

Task Server 是 SMTX OS 的块存储服务（ ZBS） 的异步长任务处理模块。提供诸如存储池间移动/复制数据（Storage Pool Service）、远端备份的数据复制（Copy Volume Service）等耗时较长的长任务处理服务。在备份服务中，Task Server 负责保护计划任务的具体执行。保护站点的 Task Server 将通过 Envoy 提供的远端接入服务来访问异地备份站点的块（block）服务， 直接对数据进行操作。此模块只处理存储卷和快照的备份、克隆、回滚等数据层的任务，不处理关联的元数据信息。Task Server 部署在所有节点上，所有的 task server 都作为 runner 实际运行长任务，而经由 zk 选出的唯一的 leader 负责任务的调度分发。

##### 备份基本原理

备份的目标是将数据的当前状态完整地复制到指定的位置。 SMTX OS 的备份模块借助 ZBS 的快照功能来达成这个目标。备份被拆分为三个主要步骤：

1. 获取当前数据状态。通过快照功能，备份模块可以获取当前的所有保护对象的在同一时刻内的状态（存在微秒级误差），后续单个数据对象的更新将不会影响到需要备份的数据快照；

2. 配置同步。备份功能所备份的不仅仅是数据本身，也包括了数据的组织状态，例如虚拟机的配置信息、文件的目录结构等等。在备份开始之前，会首先在保护站点和目标站点之间进行保护对象的配置信息同步，确保在目标站点内创建出所有保护对象的镜像对象；

3. 数据同步。在保护站点与目标站点间传输数据快照。

备份模块的克隆与回滚功能的实现原理与备份类似，基本流程均是从快照组中重建数据对象。

##### 传输速率与空间优化

备份通常运行在两个距离较远的站点之间，它们之间的带宽通常较小。为了尽可能充分地利用带宽，减少传输时间，备份功能采用如下策略组合来降低传输的数据量：

1. 增量备份：如果保护站点和目标站点曾经完成过一次备份任务，后续任务执行时，都会采用增量备份的方式进行。每次传输的数据仅是当前需要传输的快照组与目标站点已有快照组之间的增量，在数据更新不频繁或更新区域较为集中的场景下可以很大程度地减少传输量；

2. Rsync 同步模式：在变化的数据量中，也会在固定区块内（256MB， ZBS extent 大小）采用 Rsync 模式同步增量，尽可能地利用目标站点已有的数据；

3. 空洞避免：在传输时将跳过未写入过的虚拟卷空间，避免传输大量无用数据；

同时为降低备份空间成本，目标站点中的数据组织形态为快照链。对于不同时刻快照组间的重复数据将仅保留一份逻辑数据，以减少不必要的空间损耗。同时各个时刻的快照组在逻辑上依旧完整地持有所有数据。删除某个快照对其他快照的数据完整性不会产生影响。

##### Copy Volume Service 

Copy Volume Service 用于提供不同集群间存储对象（VM / Volume / File／LUN）的复制。由于数据拷贝耗时较长，需要先对源对象创建 Snapshot，后续复制这个 Snapshot。为减少数据传输流，CopyVolumeService 使用增量复制。

复制的任务包括如下 2 个阶段：

1. PrepareContext：复制前准备，初始化参数，根据 extent_id 的不同，整理出与基准 Extent 有差异的 Extent 用于复制。

2. CopyExtent：逐个 Extent 复制。对于每个 Extent，逐个遍历 Block，对比当前 Block 与基准 Block 的数据差异，只复制有差异的 Block。如果 Extent/Block 没有被真实分配（Thin Provision 时）或全零时，将快速跳过。


每个步骤执行完成时，Runner 均将向 ZK Task DB 更新当前步骤。在 Copy Extent 阶段，每完成一个 Extent 复制，均会更新状态。当 Runner 异常 Task 被重新调度时，新接手的 Runner 可以从上一个复制完成的 Extent 位置继续任务。（以上与 Storage Pool Service 相同）

用户也可以提前在 Block 层面暂停或取消任务。当任务被暂停或取消后，Runner 将最新的状态更新到 ZK Task DB 中。暂停的任务可以被重启，重启后的任务将在上一个复制完成的 Extent 位置继续。

