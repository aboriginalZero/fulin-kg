meta1 中，每个节点已分配空间包含 4 部分，allocated space = thin space + new thin space + thick space + rx space。各项含义如下：

1. thin space：本次 lsm 心跳上报的所有 thin 分片占用空间，若只考虑 replica，那一定是 256 KiB 的倍数；
2. new thin space：两次 lsm 心跳上报之间新分配的 thin 分片占用空间，若只考虑 replica，那一定是 256 MiB 的倍数；
3. thick space：所有 thick 分片占用空间，若只考虑 replica，那一定是 256 MiB 的倍数；
4. rx space：节点作为恢复/迁移目的地的预留空间，若只考虑 replica，那一定是 256 MiB 的倍数，在没有恢复/迁移时总是为 0。

在这个模式下，不同 Chunk 在占用相同空间的情况下持有的精简置备的 Extent 数量是不同的。在极端场景下比如稀疏的 thin extent 突然写入大量数据，会造成某些节点快速过载，此时期望通过节点容量均衡策略来均衡各个节点的已分配空间。



thin space =  thin space (lsm report) + new thin space +  thin reserve space + thin rx space

thick space = thick space (calculate by pid set) + thick reserve space + thick rx space

对于 reserve / rx space，不论 thin / thick 都是按 256 MiB 来算。



ReserveSpaceForAllocation 

FreeSpaceForAllocation



meta2 中 chunk mgr 不清楚具体的 pid，只会有 pid_count 和 pid_space



chunk mgr 还需要维护一个 new_thin_pids，每次 chunk 跟 chunk mgr 做 keepalive 的时候，都把他清空。


recover 时，先找 chunk mgr 预留，预留时也需要携带 space version。


chunk mgr leader 为每个 exent mgr 都维护了：

1. 一个 <space version, space info> 的预留空间相关的 map，space info 的含义是这个 extent mgr 视角里的每个 chunk 持有各种类型 pid 的 count 和 space。这个预留空间的方式可能是为 allocation，也可能是为 reposition 预留的空间；

2. 一个 space version + space info，含义是 extent mgr 上一次 data report 时的 space version 和 space info

   extent mgr 在 data report 之前，要先让 space version + 1，然后等待之前 space version 的空间分配请求都完成后（不论是 allocation 还是 reposition，不论是成功还是失败，预期他们都在一个 rpc 超时时间内完成），才统计此时的 pid count 和 space 给到 chunk mgr。

3. 



典型交互流程

1. volume mgr 创建一个 thick volume；
2. extent mgr 创建 lextent 以及为了创建 thick pextent 向 chunk mgr 申请预留空间；
3. chunk mgr 根据当下各个 chunk 的负载情况，为这些 thick pextent 分配 loc，并且暂时记在预留空间里；
4. extent mgr 拿到 locs 后，持久化 pextent，更新 pextent table，然后更新 chunk space table；
5. extent mgr 汇总 chunk space table 中的信息，以 version = x 的 space info 汇报给 chunk mgr；
6. chunk mgr 据此更新自己的 space 视图，并删除 version < x 的预留空间信息。



> 在 Report Report 时，Extent Manager 会将 Space Version = x 立刻加一并记录 x + 1 时的 Chunk 空间信息。后续的分配请求使用 Space Version = x + 1 发送。此次 Report Space 则需要等待之前 Space Version 的空间分配请求完成，Report 的延迟取决于 RPC 设置的超时时间。

是否可能出现，在收到所有 version <= x 的 alloc resp 前，发出了 version = x + 1 的 alloc 请求并收到了回复，那么此时在 chunk mgr 侧的预留空间就多算了，总体空间会偏大。



等待之前 Space Version 的空间分配请求完成的方式

最差需要等待一个 rpc 超时时间。

每个 alloc for reposition / allocation request 之前先对这个 version 记录的 inflight req ++，

extent mgr 要维护一个 version, inflight req 的 ordered map，在完成 chunk space info 更新后才 inflight req -- 



chunk mgr 能收到 space version = x 的 report space 说明在 extent mgr 那 space version < x 的操作都已经进到 pid set 了。



chunk mgr 视角里的 chunk 空间，等于最近一次 extent mgr 的 report space（其中的 space version = x） 中的 thick space + chunk 汇报的 thin_used_data_space + 以 x 为值，预留的部分

预留空间可以只算会让他变多的部分，需要指明 cap / perf 和 thin / thick

1. for allocation

   预留成功后，extent mgr 在收到 alloc resp 后，在处理完 pextent table 后，对于他的 chunk space table，如果是 thin，放入 new_thin_pids，如果是 thick，放入 thick pids

2. for reposition

   只影响 reposition dst，不影响 src / replace

   预留成功后，会放入 extent mgr 的 rx_pids



每个 extent mgr 有自己的 space version，

数据块真的在 extent mgr 分配了之后，chunk 给到 chunk mgr 的 keepalive 里才有意义



extent mgr 需要上报的内容是 



异常处理

extent mgr 重启

chunk 侧的预留信息还在，只是 session 断开重连，

chunk mgr 重启

所有预留空间信息都丢失了，在跟 extent mgr 建立 session 后，要求他们立即 report space 一次（这里可能会有较长的等待时间）





平均分 migrate quota，虽然最终总会收敛，但可能让 migrate 慢在 meta 侧



meta 改进设计

**Chunk Manager**

因为我们可以允许部分节点不运行 Chunk 服务（IOMesh 模式），也允许多个 Chunk 服务位于同一个 Node 上（5.7.0 开始支持的多 chunk 实例模式）。所以我们新增 Node 逻辑概念，代表每个可以部署 ZBS 服务的物理节点。物理节点本身的能力信息由 Service Manager 保存，Chunk Manager 仅保存它的拓扑信息和 IP 等连接信息。

整个集群内部的数据需要可以自由的在所有节点间流动，如果将 Chunk 分区管理，数据将需要分区分配，在出现节点异常，数据在单一分区内无法满足空间要求时（EC 模式下有可能会有较多的分片），节点的拓扑结构发生变化也许要调整分区时处理起来都会比较复杂。单实例全局空间管理服务会相对容易达成这个目标。因此在这个版本的设计中 Chunk Manager 保持唯一单体的形态，管理 Chunk table，与所有的 Chunk 连接，检查 Chunk 的健康状态并收集真实的空间信息，负责 Extent 的空间分配请求。

空间平衡类的 Migrate 请求（节点移除与节点容量均衡）将由 Chunk Manager 发起，但是命令执行过程将会交由 Extent Manager 完成。

**异常影响估计**

Chunk Manager 的异常会影响新的空间分配和变更类请求如新副本分配，Recover/Migrate/GC 等等，但是对于移除副本类型的请求，Remove Replica 后续可以考虑调整为让 Chunk Manager 延迟感知，降低对 IO 的影响，但是因为临时副本的分配也会涉及到 Chunk Manager 有可能也无法降低节点异常下电类型的影响。在调整完成之后需要单独针对具体的场景里的异常处理做对应讨论和设计。



Chunk 的信息与包含的 PExtent id 集合，考虑到单个 Chunk 上不太可能会出现 128 M 个 Pid 的情况，以 8 M pid （对应 2 P逻辑地址空间），估计，单个 Chunk 所需要的内存应当不会超过，64 M。在 40 个节点时，所有 Chunk 需要的总内存空间应该不会超过 3G



空间分配，从 thin / thick，prioritized / normal，cap / perf 角度展开。

分配过程中，使用的 cmp 是会变化的。



每个mgr 单独选举，获取单一服务异常后更小的故障影响范围，降低异常响应时间。

每个 chunk mgr 都有一个 raft 实例，不同节点上的 raft 实例组成一个 raft group。根据 service mgr 提供的成员列表来个调整自身 raft 实例所在位置（比如当前 chunk mgr 在节点 A B C  上，节点  C 下线/进入维护模式，那么 service mgr 可能发送新的成员列表 A B D，调整到这上面）

当 chunk mgr 感知到节点拓扑结构变化时，会通知新的 topo info 给 service mgr，service mgr 会据此调整 chunk mgr 的服务副本分布以满足拓扑安全。

不需要总是在心跳中传递 topo info version，当 chunk mgr 发现 topo 变化后，主动通知到对方（不同于目前在同一进程内用原子变量的 topo version，跨进程且大概率跨网络的场景，没法保证对方一定收到，所以需要周星期在心跳里传递 topo id）



1. node 信息管理

   1. topo info 的 CURD

      包含 zone / rack / brick / node 的拓扑信息

   2. 

2. chunk 信息管理

   1. state && status

      chunk 的状态管理

   2. pid set

      chunk 持有的 thin / thick，prioritized / normal，cap / perf pid set，用于空间计算

   3. isolated

      包含维护模式与网络亚健康管理

   4. data channel connectivity

      作为 lease owner hint，是否可以直接在 extent mgr 里维护，每个 extent mgr 会跟所有 chunk 建立连接吗？

   5. 集群中各 chunk 执行的 reposition cmd 数量（rx_pid）、transfer cmd 数量（作为 lease owner）

   6. 

3. 空间配额管理

   1. allocate && sink && elevate

      这些都算是新副本的分配

      ReserveSpaceForAllocation / FreeSpaceForAllocation

   2. recover && migrate

      reposition dst 的空间预留

      ReserveSpaceForReposition / FreeSpaceForReposition

   3. provision change

      将 pid 从各个 chunk 的 thick pid set 中删除，并添加到 thin pid set

      UpdatePExtentProvision

   4. delete garbage

      将 garbage pextent 的 pid 从各个 chunk 的 pid set 中删除

      RemovePExtent

   5. 

4. 内部 IO 命令配额管理

   每个 chunk 有个总限制，比如每个 chunk 一次只允许接受 32 个 recover cmd 同时运行，要在 meta 下发的时候做限制。

   elevate 跟其他不同，只限制了同时运行的个数不超过默认的 16 个，没有 chunk 粒度的限制，应该是考虑到 elevate 任务本身也比较少。

   每个 extent mgr 要记录自己视角的各个 chunk 正在运行的 recover cmd 数量，汇报给 chunk mgr。

   

   sink 限制的是每个 lease owner 发起的数量

   类似于 ifc 带宽的分配，

   chunk mgr 有个总额，

   

   extent mgr 给 chunk mgr 上报当前各个 chunk 正在执行的 cmd 数量、各个 chunk 等待执行的 cmd 数量

   chunk mgr 给 extent mgr 下发各个 chunk 允许执行的最大 cmd 数量

   

   每个 extent mgr 上对于每个 chunk 允许最少执行 k 个命令。好处是避免在心跳前完全不能动，坏处是可能存在浪费。或者可以把 k 设为默认的 1（0 应该也行，因为 internal io 并不着急）。

   跟 ifc 还不一样，要避免超过上限。

   

   方案一

   这里 recover 跟 migrate 需要分开讨论（目前 cmd slot 是会影响到 recover src/dst 选择，但 migrate src/dst 是提前选好的）

   假设 cid1 最多同时运行 30 个 cmd，extent mgr1 上的 cid1.executing_cmd_num = 10，cid1.waiting_cmd_num = 20，extent mgr2 上的 cid1.executing_cmd_num = 8，cid1.waiting_cmd_num = 30。

   那么需要先保证 executing_cmd_num 的预留，剩下可用的是 30 - 10 - 8 = 12，然后根据 waiting_cmd_num 决定给各个 extent mgr 的数量。cid1.allow_cmd_num = 10 + 12 * 20 / 50，cid2.allow_cmd_num = 8 + 12 * 30 / 50，有余数，直接给到 waiting_cmd_num 最大的。

   这样有可能出现饿死。可以考虑给出小于 executing_cmd_num 的 allow_cmd_num，这样在当前这一波执行的命令结束之后，就能腾出来了。但这里要避免给多了（比如这一轮发出去的总额就是少一些）。

   都给到一个 extent mgr 会有什么问题？

   

   方案二

   不同于 elevate 和 migrate，cmd avail slot 会影响到 sink 的 lease owner 选择、recover 的 src / dst 选择，所以没法直接得到 waiting cmd num，以下考虑只按 executing_cmd_num 去做分配。

   只根据已有 

   如果 extent mgr 某个分片移除，需要把预留给他的 min 删掉。

   需要配合提高尝试的次数（可以留个 flag，若存在因为 avail cmd slot 不足的情况导致 dsitribute 失败，那么后续可以立马尝试一轮）。

   参考 ifc 的文档补充，https://docs.google.com/document/d/1HtyePfeWVmKpy88YJyhoSxMGf9OqZWfzhwKcfM820vo/edit?tab=t.0

   

   都要保证一定不超过上限（可以加 chunk mgr 分配的地方加 log 检查）

   

   如果想限制 scan_limit_per_round，如果多个 extent mgr leader 都在同一个节点上，那需要考虑降低 scan_limit_per_round，这个信息应该由 service mgr 来传递？chunk mgr 没必要拿这个信息，不过 service mgr 自身的策略应该会保证 leader 尽可能散开，所以好像没必要特别调整？好处是可以减少 CPU 耗时。

   

   

   对于 drain cmd 来说，是按 lease owner 限制执行的命令数（但 sink 读和写的位置不一定在 lease owner 上，这么限制的好处是啥？如果 lease owner for sink 能分散开来，是不是就能提高集群层面的下沉效率），elevate cmd 目前没有这个限制，可以考虑加上（同一个 lid 上不会同时执行 sink 和 elevate，elevate 里有校验）。

   为什么 reposition 需要预留空间，而 transfer 就不需要？它是在实际执行的时候才去找 chunk 分配（sink 可能由 access 自行发起，就没法从 meta 层面做预留）。

   为了 sink 新分配的 lease owner 上一定有个 perf replica。

   

5. 内部 IO 模式管理

   根据集群负载指导 extent mgr 中 sink / reposition 行为

   1. elevate mgr 不需要控制信息；

   2. 复用 sink mgr，控制 drain mgr 的行为，包括根据集群平均负载设置下发 drain cmd 的频率、是否允许 drain parent / idle extent 之类的控制信息；

   3. reposition mgr，如果集群层面还有 need recover，不允许某个 extent mgr 发起 migrate，如果执行 migrate，reposition mgr 根据集群负载给出当前应该执行哪种迁移策略；

   4. extent mgr 需要先给出所在分片内有无 need recover 的信息到 chunk mgr，chunk mgr 拿到所有的 need recover 标志后，等到所有分片都不需要 recover 时，才通知各个 extent mgr leader 可以发起迁移，并给出当前负载以及各个 chunk 能用于迁移的空间大小。

      考虑到 recover 会引起节点的空间负载变化，migrate 不适合跟 recover 同时进行。

   5. chunk mgr 总是把集群负载给到所有 extent mgr leader，extent mgr leader 先判断

6. 能力协商

   代表 meta 整体与 chunk 间的能力协商，后续 chunk 有新能力还依赖这个。

   1. 新版本的 chunk 通过心跳告诉 meta 自己有一项新能力（但暂未开启），meta 给他们做标记
   2. 所有 chunk 都被标记上有新能力后，meta 通过心跳让各个 chunk 开启新能力

   原本在 access mgr 的能力协商部分，挪到 chunk mgr。

   不包含 meta 内部各个 mgr 之间的能力协商，这个比较适合放在 service mgr

7. 





1. 管理 chunk table，chunk 的 state，status，pid set，isolated，还要看下代码，需要考虑跟其他 mgr 的交互；

2. 与 chunk 建立连接，接受信息包括：chunk 的健康状态，chunk 的真实空间信息（chunk 粒度的，extent 粒度的交给 extent mgr 收集）、chunk 间连通性信息、负责 extent 的空间分配请求；

4. 新数据块的空间分配，需要区别 thin / thick，prioritized / normal，cap / perf，还要考虑到临时副本；

    1. 对于新创建的 cap / perf thick pextent，extent mgr 需要向 chunk mgr 发出批量的空间分配请求，chunk mgr 检查、预扣除、持久化后返回 extent mgr 数据块的 loc 信息
    2. cap / perf thin pextent 在实际写入、申请 write lease 的时候，需要 chunk mgr 分配空间。

7. remove && replace 的处理

    Access 在 IO 异常或者在完成 Recover Migrate 任务要调整 Location 时，将向对应 Lid/Pid （取决于 Pid 是否有自己的 Lease）所属的 Extent Manager 发出 Segment 变化请求。Extent Manager 先更新自己的 Location 信息后异步的通过心跳通知 Chunk Manager Location 变化信息（Pid 上报通常是分片的，但是对于刚刚变化过的 Extent 可以考虑有一个独立的快速上报在下次心跳里直接携带，以让空间变化更为灵敏一些）
    
    remove replica 时还涉及到临时副本的创建，extent mgr 还需要跟 chunk mgr 交互一次，为 cap 临时副本预留空间
    
7. 对 recover / migrate 的处理

    1. chunk mgr 负责 Recover/Migrate 的模式切换

    2. Recover 将由每个 Extent Manager 自主的根据 Extent 健康状态发起。Recover 的目的地将通过向 Chunk Manager 请求得到，Chunk Manager 也会保留预留 Recover 完整空间的策略。Extent Manager 会在和 Chunk Manager 的周期性心跳中同步正处于 Recover 中的 cmd，帮助 Chunk Manager 及时的释放为 Recover 预留的空间。

        只有 chunk mgr 有 chunk 连通性、全局的 recover 运行情况及空间预留信息，因此 recover src 和 recover dst 都由 chunk mgr 选取（需要检查一下现有代码里选 recover src / dst 依赖的信息 chunk mgr 是否都有？）

    3. 迁移的总体流程是 Chunk Manager 周期性的通过心跳向 Extent Manager 同步当前应该采用的迁移策略。在需要主动触发空间类型的迁移时，计算后向对应的 Extent Manager 分发迁移 quota，Extent Manager 自主生成任务后执行
        1. 局部化/Topo 修复，如果由 Chunk Manager 发起，则 Chunk Manager 需要在内存中保留有 Extent Table 和对应的 Location 字段，这对管理全局的 Chunk Manager 来说 128 M 的 Pid 对应的空间太大了，还是交由 Extent Manager 来扫描比较合适，而为了做到这一点，Extent Manager 需要知晓拓扑信息
        2. 负载均衡/节点移除，Chunk Manager 可以直接生成命令交给 Extent Manager 也可以应的迁移 Quota 交付给 Extent Manager，由 Extent Manager 再次根据实际的空间占用情况生成迁移命令后交给 Chunk Manager 做记录和空间预留；考虑到命令生成流程统一有利于统一管理和负载分担。所以我们采用 Chunk Manager 产生 quota ，具体的迁移命令交给 Extent Manager 生成的方式

8. license 检查中Chunk 数量和可分配的物理空间检查，license 以 service mgr 保存的为准，chunk mgr 向他拷贝一份

    license 校验逻辑都放在 service mgr 中做， chunk mgr 给 service mgr 提供集群的 zone 数量、chunk 数量以及当前集群容量（需要放在心跳中吗？还是等 service mgr 主动获取）。

9. topo info 以 chunk mgr 保存的为准，在周期性心跳中跟其他 mgr 传递一个 topo info version，其他 mgr 收到后如果发现与自身的不同，那么主动找 chunk mgr 拉取最新的 topo info。

    可以设定 service mgr 收到的 topo version 与自身持有的 topo version 相等超过 5 min 后，才认为拓扑稳定，基于此去做服务分片调整。

10. 提供 metric，基于节点/chunk 聚合的属性，比如节点的空间利用率，节点的 IOPS 以及聚合之后的 storage pool，zone，cluster 粒度的属性，将由 chunk manager 聚合后对外提供

    升级到 meta2，chunk 怎么知道应该找哪个 mgr，这个应该还是需要能力协商一次。

11. 与其他节点的 rpc 交互都需要携带集群 uuid 以供验证，避免节点长期离开集群后 IP 复用后部署新集群带来的数据混乱。

12. 运维动作如扩缩容

      这两部分关注下 servcie mgr 的设计文档

      目前拓扑的设计是，必须通过创建 chunk 来创建 node，在删除 chunk 后，node 会被自动删除。iomesh 下有不运行 chunk 的管理节点，没办法关联到 chunk/node 获取拓扑。chunk manager 最好能允许没有 chunk 的 node 存在，service manager 就能够通过 ip 将管理节点关联到 node 拓扑。或者把 chunk 这一级拓扑重命名为 service，元数据服务和数据服务共同使用这一拓扑层级，这样可以同时展示一个 node 是否运行元数据服务以及运行的数据服务的数量。chunk manager 在管理时需要能进行对这两种服务进行区分。

      

      是否需要为 iomesh 场景添加 node mgr 的概念？

      

      meta1 的节点注册（实际上仅有 chunk 注册）

      1. tuna 行为
         1. 调用 zbs-meta chunk register <rpc_ip> <rpc_port> [data_port]
      2. chunk mgr 行为
         1. 检查 license，最大允许物理节点个数、集群已运行时间；
         2. 创建 chunk topo obj，若有需要也创建 node topo obj，chunk 的 parent 一定是 node；
         3. 持久化 chunk、chunk topo obj；
         4. 更新 topo version、node map version。

      

      meta2 的节点注册

      区分 meta 注册和 chunk 注册

      

      meta2 的节点注册

      1. tuna 行为
         1. 
      2. chunk mgr 行为

      

      不是所有的 Node 上都同时有 Chunk 和 Meta，比如存储节点上只有 Chunk、IOMesh 场景下的管理节点可能只有 Meta。Chunk 的注册还是保留

      

      meta2 里 tower 上需要暴露有元数据服务的概念吗？

      比如也展示有元数据服务的位置吗？

      

      meta / chunk 需要有自身属于哪个物理节点的标志，这样物理节点移动了，作为 child 的所有 chunk 和 meta 都跟着移动。

      

      搞个 node mgr，管理节点层级的资源，为了保持兼容，还是放在 chunk mgr 里。

      这样

      

      chunk mgr 的

      貌似在 5.7.0 里就可以有 node mgr 和 node table

      在 tower 上不暴露一个节点里有哪些 

      每一个 node （每个 data ip 确定一个 node）有哪些

      

     是不是应该有个 node mgr，记录 node 的 data ip 跟 cid 的关系

     node mgr 也集成到 chunk mgr 中

     

     搞个 node mgr，有如下好处：

     1. 一个节点可能只有 meta，或者只有 chunk，或者同时有 meta 和 chunk，现有的基于 chunk 的设置未必适用；
     2. 节点有是否维护模式的标记（持久化），若节点进入维护模式，该节点上的元数据服务 leader 需要提前迁出，数据服务需要进入 chunk 维护模式；
     3. node 粒度的 topo info，同一个节点上的 meta serivce / chunk 拥有相同的 topo 信息，ring id 应该分开设置（元数据服务需要有 ring id 吗？）；

     

     

     为了便于兼容，node mgr 应该运行在 chunk mgr 里？

      

      

      1. 增加节点的接口由 Service Manager 提供， Service Manager 在增加自身的节点流程也会调用 Chunk Manager 注册新的节点。节点增加后，新节点上的 Chunk 服务由外部的扩容流程（TUNA 提供）注册到 Chunk Manager 上
      2. 移除节点，chunk mgr 应该还是复用之前提供的接口，需要看看代码，移除节点还是做了很多清理工作，需要理一遍。

      

13. 需要考虑一下对外提供哪些 rpc

     1. 修改节点拓扑信息
     2. 

14. 与其他 mgr 的心跳往来

     1. extent mgr 
         1. 短周期
             1. remove/replace replica 时的 loc 变化
             2. chunk 状态变化时，及时更新 lease owner hint，影响 extent mgr 的 lease 选择

         2. 长周期
             1. pid report （虽然 chunk 也会 report，但是这个应该也是需要的？比如 ever exist = false 的 pid）

     2. service mgr

15. 升级考虑

     1. 旧的 Meta 需要和所有 Chunk 先完成是否支持平滑升级的能力协商之后才会开始启动升级 meta2 过程的请求
     2. 感受一下 chunk mgr 在这个过程中需要参与的部分

16. 空间配额管理

     空间信息由 2 部分构成，1 个是 extent mgr 汇报的（包含 non-ever exist thick extent），1 个是 chunk 汇报的（thin 的实际空间），怎么做到 2 个的有效结合？

     1. 新分配 && 下沉 && 提升
     2. 恢复 && 迁移

17. 内部 IO 命令配额管理

     同样的，chunk mgr 才有所有 extent 分片的信息，所以必须它来承担全局的分配。

18. 展开 recover 过程



service mgr 不负责拉起服务进程，那是什么角色来负责

每个服务加载自己的 Raft Lib，每个服务都有自己的选举策略。外部仅能通过访问服务本身获得选举信息。

物理节点本身的能力信息由 Service Manager 保存，Chunk Manager 仅保存它的拓扑信息和 IP 等连接信息。

多实例之后，是通过 ip + port 确定每个实例的。



为啥是由 volume mgr 发起 gc？只有他能给出 gc_version。

extent mgr 负责实际下发 ElevateCmd，并由 chunk mgr 向各个 extent mgr 分配 Elevate 任务的并发度（分配依据是各个 chunk 的 perf space）。



Extent Manager 会定期扫描所有 PExtent，检查其是否缺少 Segment，如果缺少则需发起 Recover。考虑到只有 Chunk Manager 有 Chunk 联通性、全局的 Recover 运行情况及空间预留信息，因此 Recover Src 和 Recover Dst 都由 Chunk Manager 选取，Chunk Manager 同时应提供 Lease Owner Hint（先要看下目前 lease owner 都有哪些 hint，chunk mgr 能提供什么），Extent Manager 若需分配新 Lease Owner 则可参考 Hint 分配。



每个 extent mgr 没有集群整体的空间信息（只有自己负责的 pid 的视图），因此需要由 chunk mgr 来发起。chunk mgr 在汇总多个 extent mgr 上报的空间信息后，最终形成集群整体的空间信息。这种方式也方便 Chunk Manager 统计到 ever exist == false 的 Thick PExtent 的空间占用，以及快照链的空间占用等。



chunk mgr 需要把 chunk 间连通性信息给到 extent mgr，分配 lease 的时候会用到





运行 tokcpp

```shell
docker exec -it registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 bash 

# 在容器中启用 gcc 14
scl enable gcc-ztoolset-14 bash 

# cmake 3.20 的二进制预编译包
wget https://github.com/Kitware/CMake/releases/download/v3.20.0/cmake-3.20.0-linux-x86_64.tar.gz
tar -xzvf cmake-3.20.0-linux-*.tar.gz
mv cmake-3.20.0-linux-* /opt/cmake-3.20.0

# 创建符号链接（全局可用）
ln -sf /opt/cmake-3.20.0/bin/* /usr/bin/

# 应输出 "cmake version 3.20.0"
cmake --version  

git clone https://github.com/google/googletest.git
cd googletest
git checkout release-1.12.0

mkdir build && cd build
cmake .. -DCMAKE_CXX_STANDARD=11  # 显式指定C++11标准
make -j$(nproc)  # 并行编译加速

# 默认安装到/usr/local
make install  

# 若未自动安装，手动配置，这样编译 tokcpp 时才能找到这个头文件
cp lib/*.a /usr/lib64/
mkdir -p /usr/include/gtest
cp -r ../googletest/include/* /usr/include/gtest/
```

