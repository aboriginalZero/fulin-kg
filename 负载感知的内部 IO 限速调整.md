敏捷恢复的目的地只会是 rim_cid 或者临时副本中记录的失败副本位置

rim_cid 是在 access 移除副本时，若发现该副本在维护模式节点上，会在 pentry 中标记该 cid

access 认为可以敏捷恢复必须满足以下所有条件：

1. lease 上记录了这个 dst cid 有 staging_block_info；
2. 这个 pid 在 dst cid 上的 gen 不为 0 或 kInvalidGen 或 staging_block_info 中记录的；



gen sync 期间，会拒绝新的写副本/临时副本的 IO



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





给出 prefer local 会变更的几种情况总结

1. 创建 volume 的时候，指定了 prefer local
2. get lease for read 时，若 lid = 0，prefer local 优先为 volume prefer local，其次为发起这个 rpc 请求的 cid，并用在分配 vextent 上
3. get lease for write 时，若 vextent_no < num_vextents 且  lid = 0，或者 vextent_no > num_vextents 时，prefer local 优先为 volume prefer local，其次为发起这个 rpc 请求的 cid，并用在分配 vextent 上
4. COW 跟第 3 点一样
5. get lease for sink 时，若需要为 cap pentry 分配 loc，prefer local 优先为 pentry 的 prefer local，其次是发起这个 rpc 请求的 cid
6. update volume 时，若是从普通卷转换成 prior volume 且 perf pid 存在时，会用 volume 的 prefer local 去分配数据块
7. create / update / resize / rollback / reserve volume space 时，thick extent 会直接用 volume 的 prefer local 当做自己的 prefer local 去分配数据块

8. 创建 even volume 或者更新快照时指定 even mode
9. 被 active access point 更改



总结一下会取消 reposition cmd 的几种方式：

1. meta 下发的 revoke reposition cmd；
2. access 中 pending 太久；
3. access 中在 app io 过程中发现的需要 cancel；







