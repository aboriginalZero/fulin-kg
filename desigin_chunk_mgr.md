空间分配





service mgr 不负责拉起服务进程，那是什么角色来负责

每个服务加载自己的 Raft Lib，每个服务都有自己的选举策略。外部仅能通过访问服务本身获得选举信息



extent mgr 中的 chunk 状态存储，需要从 Chunk Manager 同步得到，如

1. 当前 Chunk 可以用做 Migrate Dst / Migrate Src 的空间配额；
2. 当前 Chunk 上可以同时执行的 Recover、Migrate、Drain、Elevate 命令的数量配额；
3. 当前的 Drain Policy。



为啥是由 volume mgr 发起 gc？只有他能给出 gc_version ？

extent mgr 负责实际下发 ElevateCmd，并由 extent mgr 向各个 extent mgr 分配 Elevate 任务的并发度（分配依据是各个 chunk 的 perf space）。



Extent Manager 会定期扫描所有 PExtent，检查其是否缺少 Segment，如果缺少则需发起 Recover。考虑到只有 Chunk Manager 有 Chunk 联通性、全局的 Recover 运行情况及空间预留信息，因此 Recover Src 和 Recover Dst 都由 Chunk Manager 选取，Chunk Manager 同时应提供 Lease Owner Hint（先要看下目前 lease owner 都有哪些 hint，chunk mgr 能提供什么），Extent Manager 若需分配新 Lease Owner 则可参考 Hint 分配。



每个 extent mgr 没有集群整体的空间信息（只有自己负责的 pid 的视图），因此需要由 chunk mgr 来发起。chunk mgr 在汇总多个 extent mgr 上报的空间信息后，最终形成集群整体的空间信息。这种方式也方便 Chunk Manager 统计到 ever exist == false 的 Thick PExtent 的空间占用，以及快照链的空间占用等。



chunk mgr 需要把 chunk 间连通性信息给到 extent mgr，分配 lease 的时候会用到