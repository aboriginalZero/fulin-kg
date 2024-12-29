敏捷恢复的目的地只会是 rim_cid 或者临时副本中记录的失败副本位置

rim_cid 是在 access 移除副本时，若发现该副本在维护模式节点上，会在 pentry 中标记该 cid

access 认为可以敏捷恢复必须满足以下所有条件：

1. lease 上记录了这个 dst cid 有 staging_block_info；
2. 这个 pid 在 dst cid 上的 gen 不为 0 或 kInvalidGen 或 staging_block_info 中记录的；







判断 APP IO 空闲或者繁忙的依据，采用的是 IOPS 和 BPS，忽略 Latency，这是因为 Latency 跟 IO size 关系较大，需要分别统计，实现复杂，也受到不同硬件影响，阈值选择不大容易。另外，Latency 过大可能由于磁盘故障，应该加速 Recover/Migrate。