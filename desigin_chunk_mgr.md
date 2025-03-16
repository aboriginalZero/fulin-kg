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



先理出一个大纲，需要考虑升级过程

1. 管理 chunk table，chunk 的 state，status，pid set，isolated，还要看下代码，需要考虑跟其他 mgr 的交互；

2. 与 chunk 建立连接，接受信息包括：chunk 的健康状态，chunk 的真实空间信息（chunk 粒度的，extent 粒度的交给 extent mgr 收集）、chunk 间连通性信息、负责 extent 的空间分配请求；

3. 发起空间平衡类的 migrate cmd（节点移除和节点容量均衡）到 extent mgr 执行

4. 新数据块的空间分配，需要区别 thin / thick，prioritized / normal，cap / perf，还要考虑到临时副本；

    1. 对于新创建的 cap / perf thick pextent，extent mgr 需要向 chunk mgr 发出批量的空间分配请求，chunk mgr 检查、预扣除、持久化后返回 extent mgr 数据块的 loc 信息
    2. cap / perf thin pextent 在实际写入、申请 write lease 的时候，需要 chunk mgr 分配空间。

5. 响应 gc， 包括数据块制备类型转换、garbage petent 的删除，把 pid 从 chunk table 中的 pid set 删除；

6. sink mgr，控制 drain mgr 的行为，包括根据集群平均负载设置下发 drain cmd 的频率、是否允许 drain parent / idle extent 之类的控制信息；

    Sink 事件本身保持 Access 自主触发的机制。但是 Sink 模式，即是否需要加速 Sink 需要有全局视野的模块决定。因为 Chunk Manager 负责管理收集和管理集群中的空间状态。所以基于空间产生的策略调节部分由它负责比较自然

7. 对 recover / migrate 的处理，待补充

    1. chunk mgr 负责 Recover/Migrate 的模式切换
    2. Recover 将由每个 Extent Manager 自主的根据 Extent 健康状态发起。Recover 的目的地将通过向 Chunk Manager 请求得到，Chunk Manager 也会保留预留 Recover 完整空间的策略。Extent Manager 会在和 Chunk Manager 的周期性心跳中同步正处于 Recover 中的 cmd，帮助 Chunk Manager 及时的释放为 Recover 预留的空间。
    3. 迁移的总体流程是 Chunk Manager 周期性的通过心跳向 Extent Manager 同步当前应该采用的迁移策略。在需要主动触发空间类型的迁移时，计算后向对应的 Extent Manager 分发迁移 quota，Extent Manager 自主生成任务后执行
        1. 局部化/Topo 修复，如果由 Chunk Manager 发起，则 Chunk Manager 需要在内存中保留有 Extent Table 和对应的 Location 字段，这对管理全局的 Chunk Manager 来说 128 M 的 Pid 对应的空间太大了，还是交由 Extent Manager 来扫描比较合适，而为了做到这一点，Extent Manager 需要知晓拓扑信息
        2. 负载均衡/节点移除，Chunk Manager 可以直接生成命令交给 Extent Manager 也可以应的迁移 Quota 交付给 Extent Manager，由 Extent Manager 再次根据实际的空间占用情况生成迁移命令后交给 Chunk Manager 做记录和空间预留；考虑到命令生成流程统一有利于统一管理和负载分担。所以我们采用 Chunk Manager 产生 quota ，具体的迁移命令交给 Extent Manager 生成的方式

8. license 检查中Chunk 数量和可分配的物理空间检查，license 以 service mgr 保存的为准，chunk mgr 向他拷贝一份

9. 提供 metric，基于节点/chunk 聚合的属性，比如节点的空间利用率，节点的 IOPS 以及聚合之后的 storage pool，zone，cluster 粒度的属性，将由 chunk manager 聚合后对外提供

10. meta 与 chunk 的能力协商部分， enable_unmap / write_compress_enabled 之类。

11. 与其他节点的 rpc 交互都需要携带集群 uuid 以供验证，避免节点长期离开集群后 IP 复用后部署新集群带来的数据混乱。

12. 需要考虑一下对外提供哪些 rpc





service mgr 不负责拉起服务进程，那是什么角色来负责

每个服务加载自己的 Raft Lib，每个服务都有自己的选举策略。外部仅能通过访问服务本身获得选举信息

物理节点本身的能力信息由 Service Manager 保存，Chunk Manager 仅保存它的拓扑信息和 IP 等连接信息。

多实例之后，是通过 ip + port 确定每个实例的。



extent mgr 中的 chunk 状态存储，需要从 Chunk Manager 同步得到，如

1. 当前 Chunk 可以用做 Migrate Dst / Migrate Src 的空间配额；
2. 当前 Chunk 上可以同时执行的 Recover、Migrate、Drain、Elevate 命令的数量配额；
3. 当前的 Drain Policy。



为啥是由 volume mgr 发起 gc？只有他能给出 gc_version ？

extent mgr 负责实际下发 ElevateCmd，并由 extent mgr 向各个 extent mgr 分配 Elevate 任务的并发度（分配依据是各个 chunk 的 perf space）。



Extent Manager 会定期扫描所有 PExtent，检查其是否缺少 Segment，如果缺少则需发起 Recover。考虑到只有 Chunk Manager 有 Chunk 联通性、全局的 Recover 运行情况及空间预留信息，因此 Recover Src 和 Recover Dst 都由 Chunk Manager 选取，Chunk Manager 同时应提供 Lease Owner Hint（先要看下目前 lease owner 都有哪些 hint，chunk mgr 能提供什么），Extent Manager 若需分配新 Lease Owner 则可参考 Hint 分配。



每个 extent mgr 没有集群整体的空间信息（只有自己负责的 pid 的视图），因此需要由 chunk mgr 来发起。chunk mgr 在汇总多个 extent mgr 上报的空间信息后，最终形成集群整体的空间信息。这种方式也方便 Chunk Manager 统计到 ever exist == false 的 Thick PExtent 的空间占用，以及快照链的空间占用等。



chunk mgr 需要把 chunk 间连通性信息给到 extent mgr，分配 lease 的时候会用到