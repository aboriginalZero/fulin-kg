### Optimizing Resource Allocation in Hyperscale Datacenters: Scalability, Usability, and Experiences

现有问题



资源分配问题

* Hardware placement

    决定何时何地在数据中心中添加或删除服务器机架，同时平衡相互竞争的目标，如员工工作计划、电力预算、跨故障域的分布和邻近的托管，例如需要高带宽网络的ML训练服务器。

* Service placement 

    由于不同的服务在不同的服务器上表现出不同的性能，在将每个服务分散到不同故障域的同时，确定服务器对服务的分配，并优化服务和服务器代之间的匹配。

* Service sharding

    对于数据库等分片服务，确定如何在数据中心区域内和跨数据中心区域迁移数据分片，以响应实时负载变化，同时确保跨故障域传播，防止过多并发更改，从而破坏系统稳定。

所有这些资源分配问题都有一个共同的模式，即我们希望将一组 object 分配到一组 bin 中，以优化特定目标，同时满足某些约束。



最常见的解法是混合整数规划（Mixed-Integer Programming ，MIP） ，解法有分支界定法和割平面法

时间复杂度是 O (|O| * |B|) 但问题规模变大，时间复杂度就变大了

meta 遇到的问题规模是 100w 的 objects 和 10w 的 bins。



为了解决 usability 和 scalability，meta 开发了一个 Rebalancer 用来做资源分配

Rebalancer 包含 3 方面：

* model specification

    对外提供了一些 xxxSpec 供开发者使用

* model representation

    开发者通过 spec 定义一个分配问题后，rebalancer 将其转成表达式图，图上的每个节点表示由其子表达式构造的表达式。

* model solving

    分配问题一般是 NP-hard 问题，所以目标是在有限时间内找出一个高质量问题（不保证是最好的）

    小规模问题，用 MIP solver 处理，大规模问题，用 optimizer local search 处理





#### model specification

对外提供了几个概念，比如 object 和 bin 之间的 dimension、bin scope

对外提供了一些 xxxSpec，比如 CapacitySpec

支持自定义一个新的 xxxSpec，已有的单个 Spec 最复杂也是在几百行代码里完成

* dimension

    A dimension is a mapping of each object and bin to a number

    比如 memory dimension，一台服务器的 memory dimension 规定了它可以使用的内存上限，一个 task 的 memory dimension 规定了他运行所需的内存大小

* bin

    Rebalancer uses *scopes* to represent the hierarchical structure of bins

    scop 将所有 bins 划分成若干个 scope items。比如在 rack scope 下，scope item 比如 rack1和 rack2 表示这些特定机架中的服务器集合。

* object

    类似于 bin scope 的概念，对于 object 有个 object partition 的概念，但可能不是正交的。

    object partition 中的每个集合计为 group，

    For example, in the context of cluster management, all tasks are partitioned into jobs and a job is a group of tasks that run the same executable.

    一个 job 中有多个 tasks

* utilization

    利用率的主语是 bins 或者 scope items，他不是一个比例。

    一个 bin 的 ObjectCount dimension 等于被分配到这个 bin 的 object 的个数

    util(rack_r , job_j , ObjectCount) 统计了在 rack_r 上运行的在 job_j 中的 tasks 数量

    扩展 utilization 的使用，加一项时间

    AFTER 表示当前分配给 bin_j 的 object 集合

    INITIAL 表示一开始分配给 bin_j 的 object 集合

    STAYED 是他们的交集，里面的 object 满足分配前后都在 bin_j 中的特点

    计一个符号 TIME_util = util(b_j, D, TIME)，TIME 可以取 AFTER / INITIAL / STAYED

    

    这样可以定义新概念：

    NEW_util = AFTER_util - STAYED_util，刚被移到 bin_j 的 object 的利用率

    OLD_util = INITIAL_util - STAYED_util，刚被移出 bin_j 的 object 的利用率

    ANY_util = INITIAL_util + AFTER_util - STAYED_util，不论分配前后，只要在 bin_j 中的 object 的利用率

    NEW_util 和 OLD_util 可以度量系统稳定性，另外可以得出 util(b_j, D, STAYED)  =util(b_j, D_init, AFTER) 。

定义作为 object 的 tasks 和作为 bin 的 servers 之间的关系

```
// 限制每个服务器的储存用量
addConstraint(CapacitySpec(scope="server", dimension="storage"))

// 限制一个机架上每个 job 最多运行一个 task
addConstraint(GroupCountSpec(scope="rack", dimension="ObjectCount", partition="job", limit=1))

// 平衡服务器间的存储用量
addObjective(BalanceSpec(scope="server", dimension="storage"))
```

Rebalancer 会将这些 util 变量翻译成数学形式，INITIAL_util 可以被提前计算，AFTER_util 表示按维度 D 分配后的结果。比如用 UtilIncreaseCostSpec 来优先移动 CPU 利用率小于指定阈值 T 的 server，那么对每个 object 加一个惩罚表达式 Power(excessUtil_i, 2)，其中 excessUtil_i = Max(0, util(server_i, CPU, AFTER) - T)。



有了这些 Spec 后，怎么解决 Hardware placement

datacenter region -> datacenter -> suite -> main switch board(MSB) -> server row -> server rack -> server

全球有几十个 datacenter region，每个 region 在几英里的范围内有多个 datacenter，每个 datacenter 有 4 个大房间 suite，每个 suite 中有 3 个 MSB，每个 MSB 为 1 ~2w 个 server 供电。

时不时会有整个机架的上电和下电。资源分配时需要考虑到员工的上班时间、故障域的冗余、网络、电力的资源限制。

rack 作为 object，(week, position) 作为 bin（前者是哪一周上电，后者是机架所在的物理位置）。

在同一个 MSB 和同一周的 bin 集合是一个 scope item。同一类型的 rack 组成一个个 rack partition。

* 待添加的机架一开始认为在 unassgied bin，并对这个 bin 添加 ToFreeSpec，也就是必须都分配出去。
* 对 bin scope 如 MSB/suite 添加 CapacitySpec，来限制电力和网络流量。
* AI server rack 必须放在 AI zone 以使用高带宽的网络。为此引入一个 AiRack dimension，将使用 AiRack dimension 的 CapacitySpec 作用在 position scope 上。
* 在同一周要处理的  rack 通过把 rack 放在一个 group 并且应用 ColocateGroupSpec 到这个 group 在 week scope。

总的来说，由于机架的变化是渐进的，而且我们是分别为每个数据中心区域计算解决方案的，因此硬件布局问题的规模相对较小，涉及数百个 object 和数千个 bin。对于这些小问题，Rebalancer将表达式图转换为MIP问题公式，并采用MIP求解器，而不是局部搜索求解器，以确保高质量的结果。



有了这些 Spec 后，怎么解决 Service placement

service placement 考虑的是每个 region 内的资源预留。

server 作为 obejct，reservations 作为 bin。

70w 个 servers 还能理解，不过 reservations 作为 bin，为啥会有 6k 个的量级？

为啥是 reservations 作为 bin，这个 reservation 像是资源池化后的使用？论文里说是作为一个动态的虚拟集群，用于承载业务团队的 job



k8s 的服务编排跟这个调度的区别是啥？基本上能实现 k8s 的调度，除了 k8s 的初始容量放置

特定节点pod的亲和度使用CapacitySpec指定，并使用自定义维度表示亲和度。

使用 GroupCountSpec 对副本组的 pod 间反亲和性进行建模。

使用 AvoidMovingSpec 将特定的 pods 固定到特定节点。



还是没有搞得很清楚，有了这些 spec 后，怎么定义我们想要的效果。这些 spec 貌似也不太易用



#### model representation



#### model solving



每个目标都指定了一个权重和优先级。Rebalancer使用加权和将所有目标具有相同的优先级。为竞争目标（例如objA， objB）找到合适的权重是通过首先对它们进行归一化，使它们的值具有可比性，然后根据它们在问题域中的相对重要性选择可乘性权重来完成的。一些用例为目标提供了严格的优先级，而再平衡器确保在解决低优先级目标时不会在高优先级目标上倒退。



ZBS 可能会怎么使用



用 DAG 的方式将模型尺寸从 O (|O| * |B|) 降低到 O (|O| + |B|) 





混合整数线性规划（MILP）问题

meta 也优化了他们的 MIP，将同类的 object 算到一起，将不会被改动的 bin 提前排除掉，这样大 O 虽然不变，但是系数变了。



资源分配问题可以抽象成将物品 object 放入箱子 bin 的问题



资源分配框架关注的是他的可用性和可扩展性。第一点对应是否足够抽象、透明，第二点对应是否足够高效，当问题规模变大，时间复杂度是否可接受。









这篇论文跟我之前做过的工作比较接近。

https://grok.com/share/bGVnYWN5_475fb307-a291-4e2e-8894-21e67c9e391b



作者基于 Meta 私有云的实际经验，开发了 Rebalancer 框架，旨在提供一种可扩展且易用的资源分配解决方案，并在存储、计算和网络优化中取得了显著成果

Rebalancer 提供了一种声明式的语言，允许用户以高级方式定义分配策略。例如，存储管理员可以指定“确保每个数据分片至少有 3 个副本，且副本分布在不同机架”而无需编写复杂的数学公式。



在超大规模数据中心中，资源分配的目标是优化资源利用率、性能和成本，同时满足复杂的约束条件（如服务级别协议 SLA、故障容忍、能耗限制）。在存储领域，具体问题包括：

- **数据副本放置**：如何在分布式存储系统中分配数据副本，以平衡负载、降低延迟并确保高可用性？
- **存储与计算协同**：如何协调存储节点与计算节点的资源分配，以支持机器学习训练或实时分析？
- **动态迁移**：如何在不中断服务的情况下迁移存储实例（如数据库分片）以应对负载变化？



aliyun 提供的 MindOpt 可以解混合整数线性规划（MILP）问题。



简单的问题，meta rebalancer 会将其转成  MIP 来处理。



object, bin, dimension, scope, object partition, task, job

提供了一个工具 rebalancer-explorer 可以看这个概念之间的关系，帮忙用户识别不同限制（specs）的权重、不同限制之间是否有冲突





如果有多个限制之间是冲突的，它会怎么排优先级呢？允许用户指定不同限制的权重

怎么跟别人的工作比较呢？设计了什么指标？DCM 是别人的工作，他怎么直接拿过来用

第二章给出了利用率的定义，第三章展示了怎么用，第四章展示了怎么实现



4.1 节中怎么从 o * b 通过 Lookup 优化成 o + b 的



6.1 节表明在准确度上会比 MIP 差一些，但是耗时远远短于 MIP，如果对计算时间没要求的话，MIP 这种遍历的方式更好，但是在任务规模稍微大一点的时候，都需要用 meta rebalancer 的计算方式，在规定时间内得出一个足够好的解。



按 hotness 对 bin 排序，可以快速减少 object value。如果时间充足，可以遍历所有 bin

MIP 是用形式化验证，DCM 支持 SQL 语句，Rebalancer 支持 object / bin 的概念



MIP 求解器不仅具有不可预测的执行时间，而且由于数值精度问题偶尔会遇到不可行性。

也试过用并行的方式优化，但在上升一个数量级的问题规模面前，还是比较



local search 和模拟退火的区别

为确保分配的稳定性，通常希望限制或尽量减少改进分配所需的对象移动次数。但模拟退火难以做到这一点



7.7 rebalancer 的限制

有些问题可以用 MIP 建模，但不能用 rebalancer 建模，因为它们不适合将 object 分配到 bin 的抽象。一个这样的例子是将网络流量分配给链路，其中 rebalancer 无法对链路拓扑中的依赖序列进行建模。然而，Rebalancer的低级表达式API和表达式图足够通用，可以支持这些问题，同时显著提高可用性。因此，我们扩展了表达式API和表达式图，以创建更灵活的框架，使MIP问题的建模超越了指派问题。



### FIFO Queues are All You Need for Cache Eviction

讲的是 S3-FIFO 队列，用来维护缓存队列，提高整体的缓存命中率。比 LRU 和一般的 FIFO 优秀

LRU： 基于访问时间，选择剔除最久未被访问的数据，通常基于 HashMap + Double-Linked List 实现，缺点在于对于大范围重复的 Scan 不友好；

LFU：基于访问频率，选择剔除访问频率最低的数据，通常基于最小堆实现，缺点在于若某一块数据在短时间内被大量访问后，则很难被换出；

 ARC ：由 IBM 在论文 A Self-Tuning, Low Overhead Replacement Cache 中提出。ARC 是一种平衡访问时间优先和访问频率优先的策略，ARC 可以自适应于工作负载。如果工作负载趋向于访问最近访问过的文件，将会有更多的命中发生在 LRU Ghost 链表中，也就是说这样会增加 LRU 的缓存空间。反过来一样，如果工作负载趋向于访问最近频繁访问的文件，更多的命中将会发生在 LFU Ghost 链表中，这样 LFU 的缓存空间将会增大。



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