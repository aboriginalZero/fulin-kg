## Internal IO Limit

在 zbs 集群中，除了用户 IO，系统内部也会产生用于恢复、下沉、抬升、迁移数据块的 IO，二者存在对集群有限的 IO 资源（网络/磁盘等）的竞争，为了避免内部 IO 影响业务 IO，需要有内部 IO 限流器，以指定速率拦截/放行内部 IO。

每个节点的容量层与性能层有各自独立的内部 IO 限速器，以 block 为粒度拦截/放行内部 IO，实现上分为 2 个组件：

* internal flow controller：工作在 access 侧、拦截/放行以自己作为 lease owner 收到的 internal io 的分布式限流器。
* internal io throttle：工作在 chunk 侧、拦截/放行以自己作为 internal src 或 dst 收到的 internal io 的单机限流器；

access 通过心跳从 meta 接受各种内部 IO 命令后（与其他内部 IO 不同，sink IO 可由 access 自行发起），以 block 为粒度从 internal src chunk 读取数据，写入 internal dst chunk，在 access 将内部 IO 派发到 chunk 前，若 internal flow controller 没有同时持有 internal src / dst chunk 提供的 internal token，IO 将被拦截，否则放行。大部分情况下，内部 IO 只要被 internal flow controller 放行，也会被 internal io throttle 直接放行。internal token 的颁发与消费参考 https://docs.google.com/document/d/128Lall-xo_EIPijNqf3IF4RclUhDGORWIgfY18h9cac/edit?tab=t.0#heading=h.qiis55i97kjd

对于同一类型的内部 IO 而言，以先进先出的方式放行，当同时有多种内部 IO 共存时，默认会以 recover > sink = elevate > migrate 的优先级轮转放行，保证高优先级 IO 尽快完成，同时也避免低优先级 IO 迟迟不被调度。影响放行效果的因素只有内部 IO 限速（internal io speed limit），不论当前限速器承载了多少数据量、已放行的 IO 是否完成。



为了避免内部 IO 影响业务 IO 以及充分利用 IO 资源，需要针对不同的业务 IO 负载自适应地调整内部 IO 限速，即智能调节（auto mode）。当业务处于高峰期，限制内部 IO 流量，而在业务相对空闲时，放开内部 IO 限速，让待变化的数据块尽快去到预期位置。

access 默认每 4s 检查自身是否需要调整内部 IO 限速，限速调整遵循快下降、慢上升的原则。对于每个节点来说，判断 IO 繁忙或空闲的指标是 iops 和 bps（延迟受 IO 大小、网络波动、磁盘故障影响明显，波动较大，并未采用），IO 流量包含本地（from local）流量和远程（from remote）流量两部分。

弹性调节规则如下：

1. 内部 IO 限速的默认值是最大值的 30%；
2. 当业务 iops / bps 超过繁忙阈值，每次将内部 IO 限速降低到当前限速的 50%，直到限速最小值；
3. 当业务 iops / bps 未超过繁忙阈值且内部 bps 超过当前限速的 80%，每次将内部 IO 限速提高到当前限速的 150%，直到限速最大值。

各节点的内部 IO 限速最小值是 1MiB/s，内部 IO 限速最大值与业务 IO 繁忙阈值与节点硬件能力有关，具体计算方式如下：

1. 内部 IO 限速最大值

    取决于磁盘限速和网络限速之间的最小值，其中网络限速最大值取决于集群是否开启 RDMA 以及存储网络带宽，磁盘限速最大值取决于所在层级已挂载且健康的磁盘类型和数量。

    网络

    * TCP：0.4 * 存储网带宽
    * RDMA： 0.5 * 存储网带宽

    磁盘

    * NVME SSD：每块提供 600 MiB/s 的 bps
    * SATA SSD：每块提供 250 MiB/s 的 bps
    * SATA HDD：每块提供 80 MiB/s 的 bps
    * Unkown：每块提供 100 MiB/s 的 bps

2. 业务 IO 繁忙阈值

    取决于所在层级已挂载且健康的磁盘类型和数量，大部分情况下，单一层级（容量层/性能层）中都会是相同类型的磁盘，如果混合部署了不同类型的磁盘，那么以最强类型的繁忙阈值为准（而非累加）。

    * NVME SSD：不论数量多少，只提供 500 MiB/s 的 bps、5000 的 iops
    * SATA SSD：不论数量多少，只提供 150 MiB/s 的 bps、1500 的 iops
    * SATA HDD：每块提供 20 MiB/s 的 bps、80 的 iops
    * Unkown：不论数量多少，只提供 100 MiB/s 的 bps、100 的 iops

zbs 默认运行在智能模式，由系统弹性调节内部 IO 限速，若需要人工设定，可通过 zbs cli 切换到静态模式（static mode）下进行，该模式下只允许对所有 chunk 设置相同的限速值，如果 chunk 自身硬件能力可提供的内部 IO 限速最大值小于静态设定值，只会使用内部 IO 限速最大值。



相同类型、型号的磁盘，容量不同，磁盘能提供的 iops/bps 上限并不相同，比如 P5620 NVMe SSD 1.6 TB 跟 6.4TB 的 4k iops 上限分别是 20w 和 30w，所以

（提一下没实现 iops 的原因）

内部 IO 命令在 meta 下发时，基本都比较多，不会保证慢在 meta 侧，即使多发，也就超时处理掉就好了。



一次 recover 完成流程描述（主要在 recover handler 里的逻辑）

一次 agile recover 的流程，与 normal recover 区别仅在于增量读取。



在数据进行恢复/再平衡时，ZBS 会感知应用的负载和硬件能力以及恢复/迁移任务的实际完成情况，动态的调节如下参数：Meta 扫描的频率；Access 执行任务的带宽限制；Access 执行任务的并行度；期望做到在硬件能力足够和负载相对空闲的情况下尽快达到平衡目标，在硬件能力较弱或者负载较高的情况下，适当调节确保任务的成功率并且尽可能少的对业务 IO 产生性能负面影响。

1. 一次 reposition 的耗时组成，会在哪些地方被阻塞
2. reposition 会被取消的几种场景
4. migrate 会将 replace cid 的 block 热点情况映射到 dst cid 上



每次写请求准备发往各个 Segment 进入异步等待前都会提升 Lease 中记录的 Generation，下一个请求即可使用新的 Generation。与不改变数据状态的读请求不同，写请求在 IO handler 中是不重试的，在 ZBS 链路中的 IO 重试集中在协议层（ZBS Client）避免多服务重试带来的前端协议链路失效后，内部重试带来的可能数据污染。



敏捷恢复的目的地只会是 rim_cid 或者临时副本中记录的失败副本位置

rim_cid 是在 access 移除副本时，若发现该副本在维护模式节点上，会在 pentry 中标记该 cid

access 认为可以敏捷恢复必须满足以下所有条件：

1. lease 上记录了这个 dst cid 有 staging_block_info；
2. 这个 pid 在 dst cid 上的 gen 不为 0 或 kInvalidGen 或 staging_block_info 中记录的；



gen sync 期间，会拒绝新的写副本/临时副本的 IO



staging block info 是个啥，什么时候被创建、使用、销毁

staging block info 什么时候被记录（RecordStagingBlockInfo 的调用位置）

1. replica generator sync 时





1. normal recover 是全量读一个正常副本，写到没有副本（或即使有失败副本但并不使用）的节点上
2. agile recover 是根据 staging block info 读正常副本中的指定片段，写到失败副本所在节点
3. special recover 是去读临时副本，写到失败副本所在节点

agile recover 可以跟 special recover 叠加吗？不会，agile recover 一定是从健康副本上读。

meta 下发的 agile recover 一定是会从健康副本读。

原本在节点重启或 lease 失效后，将无法再触发 agile recover，但在 zbs 5.3.0 启用了临时副本后，新的 Lease Owner 在 Sync Generation 阶段会获取有效的临时副本上的增量数据 bitmap 信息来重建 IO 记录并保存在 staging block info 中以保证更有效的触发敏捷恢复能力。



在以下 2 个副本剔除的位置记录 staging block info

1. sync gen 失败时
2. 用户 IO 写失败时，



1. reposition io 第一次写失败，access 并不会做什么剔除之类的操作，lsm 是怎么第二次 reposition io 的？第一次写失败的部分会被利用吗？
2. 代码上怎么体现的 agile recover 开始后，恢复过程中新产生的写入请求，都会同时写入恢复源和目标端？
3. 同一个 block，reposition io 是怎么跟 app io 和 sink io 做互斥的？
4. reposition 过程为啥是逐个 block 进行，而不允许并发呢？



meta in zbs 中少介绍了一节 agile recover 是怎么算 src 和 dst 的，可以加一节介绍 recover 选取策略，分别介绍 normal recover、agile recover、auto special recover。

分层之后的临时副本，在 perf layer 都是 thin，在 cap layer 都是 thick



会发起 special recover 的一定是个 dead pextent，不论是 auto 还是 manner，所有副本都 dead 了，所以肯定也没法发起 normal / agile recover。

* manner：这时去看他有没有临时副本，有的话，从中选一个临时副本执行，src cid 会是临时副本所在节点，dst cid 是与这个临时副本配对的失败副本，replace cid 会是 dead_segments[0]
* auto：自行指定使用哪个临时副本以及恢复方式是要 rollback failed replica 还是 force recover from temporary replica。





如果 meta 下发的 recover cmd 中设置了 replace cid，access 会先删除这个 replace cid 副本吗？不会，要删除也是在 recover 完成后调用的 replace pextent



什么情况下会出现给到 access 的 reposition cmd 中的 dst cid 已经在数据块的 loc 上了？ReplicaRecoverHandler::SetupRecover 里的判断



放在 recover 一节里

总结一下会取消 reposition cmd 的几种方式：

1. meta 下发的 revoke reposition cmd；
2. access 中 pending 太久；
3. access 中在 app io 过程中发现的需要 cancel；





一次 recover 完成流程描述（主要在 recover handler 里的逻辑）

先 sync gen， src / dst 是否在 lease 的 loc 上























IO Handler







perf thin space 承担之前的 cache 角色，当 lsm 





















































































































