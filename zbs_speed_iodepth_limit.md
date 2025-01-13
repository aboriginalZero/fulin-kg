Cap IO Throttle

zbs 中长期存在以下 2 个事实：

1. 多个 Access 间、单个 Access 内都没有对所有类型的 IO 下发做统一并发度调控，这可能会导致不同 LSM 间的 IO 并发度、单个 LSM 内不同类型 IO 并发度差异都很大；
2. 为避免 IO 长时间不返回，每个发往本地的 LSM IO 若在默认 8s 内未完成，将被剔除分片。

这会导致若 LSM 先收到并发度很高的类型为 A 的 IO，会让后到的、类型为 B 的 IO 有很长的排队延迟，容易引发 B IO 超时、分片剔除，进而影响数据安全。在开启分层之前，大部分 IO 总是先落到性能较好的 SSD cache 上，影响还好，但在分层后，Cap IO 可以直接落在 SATA HDD 上，这个问题将难以忽视。

为了尽量避免出现此类现象，从 zbs 5.6.0 开始，zbs 通过工作在 LSM 外围的 pextent io handler / local io handler 的 Cap IO Throttle 来限制当容量层介质是 SATA HDD 时的 Cap IO 并发度。

对于所有的 Cap IO，不论是业务 IO 还是内部 IO，都需要经过 Cap IO Throttle，由 Cap IO Throttle 根据各类 Cap IO 当前并发额度决定放行/拦截。当 Cap IO 发往 LSM 时，并发度加一，当 IO 从 LSM 返回时，并发度减一。

默认配置下，各节点 Cap IO 总并发度上限 = 4 * SATA HDD 数量，每 100ms 根据各类 Cap IO 权重以及现有 IO 数量重新计算一次各类 Cap IO 的并发度上限，当某类 Cap IO 没有正在执行的 IO 时，本轮得到的并发度上限为 0，否则按各类 Cap IO 权重 recover : sink : elevate : migrate = 4 : 2 : 2 : 1 计算本轮得到的并发度上限。为了避免并发度突变、性能抖动过大，每一轮调整每类 IO 的并发度上限只会在上一轮的基础上加/减一。

实现上，Cap IO Throttle 不限制 APP Cap IO 的并发度，这是因为从旧版本升级到 5.6.x 的集群中的存量数据块会被认为是 Cap Pextent，但它们实际上有可能在性能层（取决于 lsm 内部 writeback 的进度），如果限制了它们的业务 IO 并发度，可能导致业务 IO 延时极高。为了避免 APP CAP IO 并发度很高时，内部 Cap IO 没有机会下发，会将 Cap APP IO 正在并发执行数量的一半作为内部 Cap IO 的并发度上限。

为了保证数据块的 gen 始终有序，Cap IO Throttle 中同一 pid 的 Cap IO 需要按 FIFO 顺序执行。在这个前提下，为了保证 IO 优先级（app > recover > sink / elevate > migrate），允许后到的、高优先级的 Cap IO 提升先到的、但被拦截在低额度队列的低优先级 Cap IO 到高额度队列中。

举个例子，若一开始 Cap IO Throttle 没有并发额度，

1. 来了 2 个 sink IO，那么这 2 个 IO 都会在 sink 队列中等待；
2. 来了 1 个非读写 IO（比如 get gen），那么这 3 个 IO 都会在 sink 队列中等待；
3. 来了 1 个 recover IO，那么这 4 个 IO 都会在 recover 队列中等待；
4. 来了 1 个 APP IO，那么这 5 个 IO 都会在 APP 队列中等待；

由于 APP 队列可以认为是个并发额度无限大的队列，所以最迟在 100ms 后，5 个 IO 都将被下发，且下发顺序是先 sink 再非读写 IO 接着 recover 最后是 APP IO。





> 不对 app cap io 限制并发度给的原因是：从旧版本升级到 5.6.x，所有现有的 PExtent 都会变成 Cap PExtent，根据 Writeback 的进度它们的数据可能在 Cache 磁盘或者 Partition 磁盘上。Local IO Handler 无法感知它们数据在哪种磁盘介质上，如果把它们的 APP IO 都做并发度限制，可能导致 APP IO 延迟极高。因此目前无法对 APP Cap IO 做并发度限制。可能需要 LSM 配合才能够更准确地判断 APP Cap IO 是否要限制并发度。

如果是新部署的 5.6.x 集群，是不是就可以对 app cap io 做限制了？集群是升级而来还是新部署的，tuna 可以支持。

> 因此，APP Cap IO 的并发度可能很高。为了避免内部 Cap IO 抢不到并发度，当 APP Cap IO 较高时，会采用 APP Cap IO 的并发度的一半作为内部 Cap IO 并发度的总上限。


按这个策略执行的话，如果 app io 并发度很高，假设是 100 ，那么内部 io 并发度会是 50 ，这可能大大超过硬件提供的上限（每块 hdd 仅提供 4 个并发度），如果 150 个 io 同时下发到 hdd 上，那会比较容易触发 8s 的超时吧？那 cap io throttle 不就没发挥它的作用了。

是否可以考虑，让 cap io 还是有个上限，但是权重足够大（比如是 16）。

如果要保证内部 IO 并发度上限随，分层后，落到 cap 上的随机读





从理论上分析，也应该有个 perf io throttle，比如同一个 pid 的 perf app io 排队排在了 perf recover io 后面，需要等待 recover io 占满了并发度。

一个是 speed limit，一个是 depth limit，前者只管以一定的速率下发，不管 io 完成情况，后者关注 IO 完成情况，每完成一个才能新放行一个下去。

从另一个角度讲，因为有了 internal io throttle 和 cap io throttle，让 lsm 侧的演出可能是超过 8s 的。

为啥并发额度可以不随 app io 弹性变化？看起来没必要。

为什么 cap io throttle 需要保证 gen 有序？



1. from remote io stat 的 throttle_latency 可以用来在 prometheus 中查看 IO 在 local io handler 里被阻塞了多久。

2. 跟 internal io throttle 类似，pextent io handler 中用异步方法，local io handler 用同步的原因是啥？

   其实也是可以是异步的。只是我们以前都是用同步的写法，它一般更清晰，不容易出错。未来估计异步化改进后，会减少 co，都改成异步的

3. 为啥在 LandingRWIO 中还要调用一遍 PutCapIOCtx(cap_io_ctx)？

   一开始会获取一个凭证， 在 IO 做完的时候还回去

4. AllocNewIODepthLimits 里，app io 量很大时怎么避免了 Cap IO 被饿死？

   因为 app io 默认不限 iodepth，所以它现有 io 数量的一半，用来作为其他 3 种内部 IO 的

5. 每 100ms 重新分配  4 种类型可用的 iodepth 时平滑处理了，那如果插拔 SATA HDD 盘，增减 4 个 iodepth，SetIODepthLimit() 中是否也需要平滑处理？

   插拔盘这种运维场景出现的概率较低，能够忍受。

6. 后到的高优先级 IO 会提升低优先级的同一个 pid 的 io 到高优先级队列里，那之后消耗的是分给高优先级的 iodepth 额度，这个影响不大？

   是的，比如现有一些 pid a 的 recover io，然后来了一个 pid a 的 app io，后到的 app io 把前面的 recover io 都提升上去到 app io 的队列，而 app io 不限速，所以可能会让 recover io 突破他应该有的并发度上限。

7. 啥时候在进入 CapIOThrottle::Run 这个函数时，state 会是 LANDING？

   在 local io handler 里被 cancel 

8. Cap IO Throttle 的 queue 跟 internal io throttle 的应该比较像是？分别支持 pid 和 io type 链起来

9. 目前 reposition 是允许多个 extent 同时进行，但是每个 extent 内逐个进行。由于一个 reposition cmd 可能涉及不同的 src / dst，所以保持 extent 粒度的并发是有利于提高集群整体 reposition 进度，但如果也允许 block 粒度的并发，这样就能保证作为 reposition src / dst 的 chunk，能够更连续地处理，hdd 上性能可能会更好。比如 cap reposition 现在慢在 read 上，基本上是个 256k 随机读。

8. extent 粒度有 gen，但 block 粒度没有一个类似于 gen 的概念用于对账。jiewei 提供了一个思路是这会有 1024 倍 64 byte 的内存放开，不利于之后向大规模演进。





Local IO Handler 对收到的 Cap IO 会做并发度限制。Cap IO 可以分为 APP、RECOVER、SINK、MIGRATE 等。Access 目前没有对不同 IO 类型 Cap IO 的下发做统一并发度调控，多个 Access 之间也没有做协同，Local IO Handler 收到不同 IO 类型的并发度可能差异很大，低并发度的 IO 类型可能有很大的排队延迟。例如，APP IO 和 SINK IO 同时进行时，如果 SINK IO 并发度为 90，APP IO 并发度只有 1，那么 APP IO 就很难得到 Cap IO 并发度，排队延迟很高。

改进的方式是，为不同 IO 类型分配各自的 Cap IO 并发度。分配规则如下：

- 如果用户指定了预留并发度，会按照预留值分配。预留值可能超过 HDD 提供的总并发度限制（4 HDD 默认一共 16 并发度）。
- 如果 HDD 总并发度还有剩余，且用户指定了份额，会按照份额的比例对剩余并发度分配；
- 如果没有指定份额，那么按照当前不同 IO 的队列深度比例对剩余并发度分配。

只有当 IO 类型当前有 IO 时，才会为它分配并发度。并发度分配调节的周期为 100ms 每次。

默认会为不同 IO 类型指定份额，APP、RECOVER、SINK、MIGRATE 的份额分别为 4:4:2:1。





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



总结一个  app write 可能被限制的地方，一个  internal io 可能被限制的地方。





flow control 的目标是，保证一个时间窗口内，空间释放与分配速度基本一致，避免集群 perf thin space 空间耗尽，LSM 返回 IO 失败，导致 IO Error 或分片剔除。



* flow controller (flow ctrl)：运行在 access io handler 中，定期向 Flow Manager 获取 Tokens，并根据 Tokens 控制 IO 下发。如果 IO 得到 Token，可以立即下发；如果得不到 Token，需要等超时后才能下发。通过立即下发避免 IO 等待；通过超时避免 Perf Space 快速增长。
* flow manager (flow mgr)：根据本地 lsm 提供的 perf thin space 使用率决定是否开启限流，以及按照一定策略分配 Tokens 给各个 Flow Controller 使用；

二者典型的交互流程是：

1. 各个 ifc 往各个 ifm 发出 req 并等待 ifm 返回；
2. 各个 ifm 按照一定周期（默认 100 ms）集中处理每个周期内收到的所有 ifm req 并发出一一对应的 resp；
3. 各个 ifc 收到 resp，立即发送下一个 req；
4. 以上过程循环反复。



flow ctrl 

1. io loc 组成

   对外使用上，仅暴露一个 InterceptIO，给定 io loc。

   类型为 cap 或者 IO 类型为 read，那么 io loc = 0；

   1. 如果是 perf thick，因为正常副本和  recover dst 都是写 perf thick，所以不需要参与限流，临时副本所有节点写 perf thin，所以要限流；
   2. 如果是 perf thin，临时副本、正常副本、recover dst 所在节点都需要拿 app token；

   需要临时副本参与的条件是：这是一个 256k 大小的 block io，因为 lsm 内部会分配新的 256k block（虽然会 gc 旧的 256k）

   promote 上来的

   ProcessWrite、ProcessUnmap、DoProcessPromoteIO

2. 超时时间设定

   每个需要 app token 的超时时间与这个 app token 要写的节点容量有关。

   如果 <= 0.9，timeout = 0.5s，0.9 - 0.98 之间是  0.5 ~ 5s，0.98 之后是 5s 

   fm 同时收到所有 fc 颁发的 app token 时，才能下发。

   另外，如果 Data Channel 已经失联，InterceptIO 会跳过等待 Tokens，加速 IO 完成；如果等待过程中 Data Channel 失联，也会唤醒等待的 IO。

3. token 消费顺序

   Flow Controller 会按照 IO 到来的顺序消费 Token，避免饥饿问题。例如 Flow Controller 有两个 IO 在等待，分别为写（1,2,3）和（1,2），其中 1 的 Token 充足；当 Flow Controller 收到来自节点 2 Flow Manager 的 Token，Flow Controller 会先让排在前面的（1,2,3）写得到 Token，即使他可能会因为缺失 3 的 token 没法立即下发。



flow mgr

1. 总可用 token

   开启限流条件：节点视角，当 perf thin 使用率超过 90% 开启限流，随后当低于 85% 关闭限流。

   每个 fm 生产的 avail token 跟 perf thin 使用率成正比，使用率越高，可用 token 越低，超过 98%，avail token 直接为 0（此时 fc 获取 token 超时后将继续下发，超时时间也是下发的）。也跟 cap 层盘的类型与数量有关。

2. 各 flow ctrl 可用 token 计算

   参考 ifc 中的描述





当集群平均负载处于中负载及以上时，允许新的用户 IO cap 直写，但不对 cap space IO 限流，空间耗尽返回 IO 失败即可。



meta 会根据集群平均负载决定使用何种下沉策略（下发给每个节点，每个节点使用的下沉策略一定是一致的）：

1. sink low：不允许 cap 直写；
2. sink medium：前一秒的 256k iops 若超过 500，允许接下来的 5s 做 256k 对齐的 cap 直写；
3. sink high：允许 256k 对齐的 cap 直写；
4. sink very high：允许 8k 对齐的 cap 直写。

















