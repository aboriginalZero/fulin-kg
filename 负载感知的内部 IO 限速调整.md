





在 zbs 集群中，除了用户 IO，系统内部也会产生用于恢复、下沉、迁移数据块的 IO，二者存在对集群有限的 IO /带宽资源的竞争。









在数据进行恢复/再平衡时，ZBS 会感知应用的负载和硬件能力以及恢复/迁移任务的实际完成情况，动态的调节如下参数：Meta 扫描的频率；Access 执行任务的带宽限制；Access 执行任务的并行度；期望做到在硬件能力足够和负载相对空闲的情况下尽快达到平衡目标，在硬件能力较弱或者负载较高的情况下，适当调节确保任务的成功率并且尽可能少的对业务 IO 产生性能负面影响。













一次 recover 完成流程描述（主要在 recover handler 里的逻辑）

一次 agile recover 的流程

内部 IO 自适应调节



1. 一次 reposition 的耗时组成，会在哪些地方被阻塞
2. reposition 会被取消的几种场景
3. recover 影响 lease 的释放，一次 recover 结束时
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



判断 APP IO 空闲或者繁忙的依据，采用的是 IOPS 和 BPS，忽略 Latency，这是因为 Latency 跟 IO size 关系较大，需要分别统计，实现复杂，也受到不同硬件影响，阈值选择不大容易。另外，Latency 过大可能由于磁盘故障，应该加速 Recover/Migrate。



1. normal recover 是全量读一个正常副本，写到没有副本的节点上
2. agile recover 是根据 staging block info 读正常副本中的指定片段，写到失败副本所在节点
3. special recover 是去读临时副本，写到失败副本所在节点

agile recover 可以跟 special recover 叠加吗？不会，agile recover 一定是从健康副本上读。

meta 下发的 agile recover 一定是会从健康副本读。



会发起 auto special recover 的 pid 一定是一个 dead pextent，所有副本都 dead 了（所以肯定也没法发起 normal / agile recover），这时去看他有没有临时副本，有的话，从中选一个临时副本执行，src cid 会是临时副本所在节点，dst cid 是与这个临时副本配对的失败副本，replace cid 会是 dead_segments[0]

人工发起 special recover 的就不一定了。



如果 meta 下发的 recover cmd 中设置了 replace cid，access 会先删除这个 replace cid 副本吗？



总结一下会取消 reposition cmd 的几种方式：

1. meta 下发的 revoke reposition cmd；
2. access 中 pending 太久；
3. access 中在 app io 过程中发现的需要 cancel；







