### Optimizing Resource Allocation in Hyperscale Datacenters: Scalability, Usability, and Experiences

这篇论文跟我之前做过的工作比较接近。

https://grok.com/share/bGVnYWN5_475fb307-a291-4e2e-8894-21e67c9e391b



作者基于 Meta 私有云的实际经验，开发了 Rebalancer 框架，旨在提供一种可扩展且易用的资源分配解决方案，并在存储、计算和网络优化中取得了显著成果

Rebalancer 提供了一种声明式的语言，允许用户以高级方式定义分配策略。例如，存储管理员可以指定“确保每个数据分片至少有 3 个副本，且副本分布在不同机架”而无需编写复杂的数学公式。



在超大规模数据中心中，资源分配的目标是优化资源利用率、性能和成本，同时满足复杂的约束条件（如服务级别协议 SLA、故障容忍、能耗限制）。在存储领域，具体问题包括：

- **数据副本放置**：如何在分布式存储系统中分配数据副本，以平衡负载、降低延迟并确保高可用性？
- **存储与计算协同**：如何协调存储节点与计算节点的资源分配，以支持机器学习训练或实时分析？
- **动态迁移**：如何在不中断服务的情况下迁移存储实例（如数据库分片）以应对负载变化？



### FIFO Queues are All You Need for Cache Eviction

讲的是 S3-FIFO 队列，用来维护缓存队列，提高整体的缓存命中率。比 LRU 和一般的 FIFO 优秀





### A Cloud-Scale Characterization of Remote Procedure Calls

RPC 的调用链是宽而不深

为了尾延迟（少数请求的响应时间特别长），可以用请求对冲来缓解，即同时发送多个相同请求，取最快的结果，但缺点也很明显，比如浪费 CPU 资源。

zbs 可能在某个时刻卡住了，即使发多个请求，但是都慢。



### RackBlox: A Software-Defined Rack-Scale Storage System with Network-Storage Co-Design

网络-存储协同设计，将 SDN 和 SDF 集成，通过状态共享和全局管理优化机架级存储系统。

* 协调 I/O 调度：动态调整 I/O 调度以适应网络延迟，降低尾延迟。
  * SDN 交换机实时测量和预测网络延迟，并将这些信息传递给 SDF 控制平面。
  * SDF 根据网络延迟调整 I/O 优先级和调度策略，避免网络瓶颈影响存储性能。
  * 例如，当网络延迟较高时，RackBlox 可能推迟低优先级 I/O 请求，优先处理高优先级请求。
* 协调垃圾回收：在机架内协调 GC 活动，减少对 I/O 的干扰。
  * SDN 控制平面监控机架内所有 SSD 的 GC 状态（如空闲块数量、GC 触发频率）。
  * 当某个 SSD 需要执行 GC 时，RackBlox 协调其他 SSD 的 I/O 活动，优先将请求路由到未进行 GC 的 SSD。
  * 通过全局调度，减少 SSD GC 对尾延迟的影响。
* 机架级磨损均衡：通过数据交换平衡 SSD 磨损，延长 SSD 使用寿命。
  * SDN 跟踪每个 SSD 的擦写次数（erase count）和健康状态。
  * 定期在 SSD 之间交换高写入负载的数据块，平衡磨损程度。
  * 数据交换利用机架内高速网络，确保低开销。



### Combining Buffered I/O and Direct I/O in Distributed File Systems

Linux 提供了两种主要的 I/O 模式：

- **缓冲 I/O（Buffered I/O）**：数据通过 Linux 页面缓存（page cache）进行读写，适合小块、随机 I/O 操作，能减少对底层存储的直接访问。
- **直接 I/O（Direct I/O）**：绕过页面缓存，直接在用户缓冲区和存储设备间传输数据，适合大块、顺序 I/O 操作，能降低内存拷贝开销和 CPU 使用率。

然而，这两种 I/O 模式各有优劣：

- 缓冲 I/O 的优势与局限：
  - 优势：通过缓存减少磁盘访问，适合小 I/O 或高锁争用场景；支持预读（read-ahead）和延迟写（write-back）。
  - 局限：页面缓存可能导致内存压力，尤其在大数据量场景下；缓存管理开销可能增加延迟。
- 直接 I/O 的优势与局限：
  - 优势：减少内存拷贝和缓存管理开销，适合大 I/O 操作；降低 CPU 使用率。
  - 局限：需要满足严格的内存对齐要求（如 512 字节边界）；对小 I/O 或高锁争用场景效率较低。

在分布式文件系统中（如 Lustre），应用通常倾向于使用熟悉的缓冲 I/O，即使直接 I/O 在某些场景下可能更优。这是因为：

1. **用户选择困难**：分布式系统的复杂性使得用户难以判断哪种 I/O 模式更适合特定工作负载。
2. **动态环境**：I/O 大小、文件锁争用和内存压力等因素随时间变化，单一 I/O 模式难以始终最优。
3. **性能瓶颈**：现有文件系统缺乏动态切换 I/O 模式的机制，导致性能次优。

针对这些问题，论文提出了一种名为 **AutoIO** 的自适应 I/O 方法，通过动态选择缓冲 I/O 和直接 I/O，结合系统状态（如 I/O 大小、锁争用、内存压力），优化分布式文件系统的性能。

**决策依据**：

- **I/O 大小**：大 I/O 操作（例如 >512KB）倾向于直接 I/O，以减少缓存开销；小 I/O 操作（例如 <4KB）倾向于缓冲 I/O，以利用页面缓存。
- **文件锁争用**：高锁争用场景下，缓冲 I/O 能通过缓存减少锁等待时间；低锁争用时，直接 I/O 可降低延迟。
- **内存压力**：内存紧张时，直接 I/O 优先，以避免页面缓存占用过多内存；内存充足时，缓冲 I/O 可提高缓存命中率。

**实现**：

- 在 Lustre 客户端的虚拟文件系统（VFS）层插入 AutoIO 模块，监控 I/O 请求和系统状态。
- 使用启发式算法根据上述因素动态选择 I/O 模式。

**实现细节**

- **平台**：在 Lustre 客户端和服务器端实现 AutoIO，修改 VFS 层和文件系统内核模块。
- **监控**：通过内核接口（如 /proc 或 sysfs）收集 I/O 大小、锁争用和内存压力等信息。
- **透明性**：AutoIO 对应用完全透明，无需修改用户代码或指定 O_DIRECT 标志。

当前实现基于 Lustre，未来需验证在其他文件系统（如 Ceph、HDFS）上的适用性。



### MinFlow: High-performance and Cost-efficient Data Passing for I/O-intensive Stateful Serverless Analytics

本文有源码，https://github.com/lt2000/MinFlow

在 I/O 密集型有状态无服务器分析（I/O-intensive stateful serverless analytics）如 MapReduce 中，函数间的数据传递（data passing）成为性能和成本的关键瓶颈，函数间数据通常通过远程存储（如 AWS S3）传递，涉及大量的 PUT/GET 操作，导致高延迟和高存储成本。

MinFlow 的设计围绕三个主要组件：**拓扑优化器（Topology Optimizer）**、**函数调度器（Function Scheduler）** 和 **配置建模器（Configuration Modeler）**。以下详细介绍其工作原理：

多层次拓扑优化 

**目标**：生成高效的数据传递拓扑，减少 PUT/GET 操作。

**方法**：

- MinFlow 分析工作流的 **有向无环图（DAG）**，识别函数间的数据依赖。
- 快速生成多种多层次拓扑，每种拓扑定义了函数间数据传递的路径和存储位置（本地内存、远程存储等）。
- 优先选择减少远程存储访问的拓扑，例如通过本地缓存或直接传递（direct passing）替代 S3 访问。
- 论文指出，优化的拓扑可将 PUT/GET 操作量减少 30-50%。



交错分区调度

**目标**：优化函数调度，减少远程存储的数据传输量。

**方法**：

- 将 DAG 分割为小规模的 **二分子图（bipartite sub-graphs）**，每个子图包含一组生产者函数和消费者函数。
- 使用交错分区策略（interleaved partitioning），根据数据局部性（data locality）和网络带宽分配函数到计算节点。
- 通过将生产者和消费者函数调度到同一节点或附近节点，减少远程存储访问，实验显示数据传输量可减少 50% 以上。



精准配置建模

**目标**：确定最优的函数并行度和资源配置，平衡性能和成本。

**方法**：

- 开发一个数学模型，基于历史执行数据预测数据传递时间和存储成本。
- 模型考虑因素包括：函数并行度、数据大小、网络带宽、存储延迟。
- 通过迭代优化，选择最小化作业完成时间（JCT, Job Completion Time）的配置。
- 例如，论文在 200GB Terasort 任务中，通过调整并行度将 JCT 降低 40%。



### μSlope: High Compression and Fast Search on Semi-Structured Logs

通过 Schema结构的简洁表示、数据分组、列式存储与压缩、支持在压缩状态下直接搜索四个步骤将半结构化日志数据“结构化”，从而提高对半结构化数据的压缩和搜索效率。



### Burstable Cloud Block Storage with Data Processing Units

DPU 是一种专为加速网络和存储任务设计的硬件，可以卸载 CPU 的部分工作，从而提升系统的整体效率。

在 BurstCBS 中，存储代理（SA）运行在 DPU，另外，设计了一个可突发的 I/O 调度器



### What’s the Story in EBS Glory: Evolutions and Lessons in Building Cloud Block Store

EBS 的演进，最新一版是 2024 年的 EBSX，用 PMEM 和去掉在线 EC

假设一个数据库应用（如 MySQL）部署在阿里云 EC2 实例上：

1. **EBS1**：VD 映射到单一 BlockServer，热点导致性能瓶颈，延迟高。
2. **EBS2**：Segment 分片和 SSD 后端提升 IOPS，日志结构支持压缩/EC，降低成本。
3. **EBS3**：FWE 和 FPGA 加速写入，减少网络流量，适合高并发查询。
4. **EBSX**：PMem 缓存实现 30 μs 延迟，满足实时分析需求。