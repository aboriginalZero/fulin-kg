ZBS-21375

在静态模式下，做了输入检查，限制为 500 MB/s。但是实际上让各个 Chunk 自己确定最大值即可。 Meta 侧不应该做检查。在 Static 模式下，Meta 调整为显示期望值与各个 Chunk 实际的取值（在硬件支持的最大与 Meta 下发的静态数字中取 Min）即可。因为不同硬件的能力不同，而 Meta 其实不太需要完整的感知到每个节点相对动态的硬件能力变化。 500 MiB/s 是实现恢复限流功能时基于万兆网卡和 2 个 SATA SSD 设置的静态上限。



静态模式下，用户设置的 migrate_limit 跟随心跳下发，chunk 上的 migrate_limit 被更新，chunk 自己每隔 4s 会根据硬件能力来更新 max_migrate_limit，在 UpdateThrottleConfig 时，会保证 migrate_limit <= max_migrate_limit。



ZBS 侧对外暴露 MetaRpcServer::SetRecoverModeInfo()

ZBS meta 侧

RecoverManager::SetStaticMigrateLimit()，如果用户传入的 migrate_limit > cluster_max_migrate_limit，会直接返回 EInvalidArgument，否则会随着心跳 AccessManager::UpdateRecoverMigrateLimit 下发更新到 chunk 侧 

cluster_max_migrate_limit = 所有 session 中的 current_max_migrate_limit() 的最大值

通过 AccessManager::SummarySessionMigrateLimit() 设置的 current_max_migrate_limit，这个值实际上等于 每个 AccessSession max_migrate_limit 中的最大值

RecoverManager::GetClusterMaxRecoverLimit() 不论是 static mode 还是 auto mode 都会使用，他是用来确定 cluster_max_recover_limit。



ZBS chunk 侧

chunk 上每 4s 调用一次 AccessHandler::AdjustRecoverMigrateLimit()，其中调用 AccessHandler::GetRecoverMigrateLimit() 根据硬件设备得到 limit，并用 limit.default_migrate_limit 更新 migrate_limit，用 limit.max_migrate_limit 更新 max_migrate_limit。这两个值会跟随心跳  AccessHandler::ComposeAccessRequest 上报给 meta leader。

其中，AccessHandler::GetRecoverMigrateLimit()，在这先计算各个磁盘支持的上限，然后算物理网卡支持的上限，（开启 rmda 情况下是 0.5 * 10GB = 500 MB/s），二者取最小值得到 max_migrate_limit，然后更新 default_migrate_limit = 0.1 * max_migrate_limit。

每个 chunk 通过 AccessHandler::HandleAccessResponse 处理 meta 侧跟随心跳下发的信息，其中将 meta 侧的 migrate_limit 设置到自身的 recover handler 的 migrate_limit 字段。（响应 meta 侧下发的上限速率）



ZBS-25522

目前弹性恢复策略中的智能模式（auto mode）会根据 APP IO 做快速增减 Recover IO 的限速，但这个智能调节策略目前在发现 APP IO 较高时一次性将限速调整到阈值等于 40MB/s 的做法过于粗暴。这个阈值的选定至少可以根据 1. 硬件能力 2. APP IO 来动态调节，而非一个固定值。

这个代码好写，但是验证代码的有效性需要上集群做实验，留到之后处理。

```
diff --git a/src/access/access_handler.cc b/src/access/access_handler.cc
index 44c04070d..b25795fbb 100644
--- a/src/access/access_handler.cc
+++ b/src/access/access_handler.cc
@@ -1506,6 +1506,8 @@ void AccessHandler::AdjustRecoverMigrateLimit() {
         if (total_iops > limit.normal_io_busy_iops_throttle ||
             total_bandwidth > limit.normal_io_busy_bps_throttle ||
             dirty_cache_ratio > FLAGS_recover_limit_high_cache_ratio) {
+            // TODO 直接设成固定值 40/50 MB/s，这个值是根据硬件能力得到的，此处还要考虑根据 APP IO 来调节
+            // 这个智能是不是应该考虑把硬件能力完全发挥出来，比如 APP IO 只用了 0.5 的硬件能力，但这边只是固定的 0.1
             recover_handler_.SetRecoverLimit(limit.default_recover_limit);
         } else if ((has_cache && dirty_cache_ratio < FLAGS_recover_limit_low_cache_ratio) || !has_cache) {
             auto recover_bandwidth = recov
```

