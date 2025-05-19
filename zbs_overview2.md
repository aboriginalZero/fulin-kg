





## 分片分配策略

zbs 保证每个节点最多持有 1 个 PExtent 的一个分片，不论冗余类型是 replica 还是 ec，在分配每个数据块 PExtent 的分片位置时, zbs 都会根据集群负载（集群中负载最高的节点负载）以及各个 PExtent 的 prefer local 节点所处负载的不同，使用不同的分配策略，具体分片分配策略遵循如下原则：

1. 集群低负载：① 拓扑安全 ② 本地优先 ③ 局部化
2. 集群非低负载 && 节点中低负载：① 拓扑安全 ② 本地优先 ③ 容量均衡
3. 集群非低负载 && 节点高负载：① 拓扑安全 ② 容量均衡

序号越小的原则优先级越高，各个原则含义如下：

* 拓扑安全：不论哪种负载，分片分配策略总是优先满足拓扑安全。若拓扑安全暂时无法满足，允许先分配，后续通过迁移来修复；
* 本地优先：若 prefer local 节点处于健康状态且剩余空间充足，一定将分片分配给它；
* 局部化：若集群拓扑结构、节点健康状态不变且剩余空间充足，prefer local 相同的 pextent 分片的分配位置是固定不变的；
* 容量均衡：若候选节点的拓扑等级一样，将分片分配到负载更低（空间利用率更低）的节点。

ZBS 对外提供的最小存储对象是 Volume，但属于同一个 Volume 的 PExtent 的 prefer local 并不一定相同(比如克隆 Volume 和源 Volume 有共享的 PExtent，在克隆卷中部分 PExtent 发生 COW 拥有自己的 PExtent 和 prefer local 时）。当 PExtent 的 prefer local 为 0 或是一个不健康的节点，prefer local 的负载没有实际意义，本地优先被放弃，此时根据集群负载决定 PExtent 的分配策略。

从 zbs 5.6.0 开始引入层次化改造，节点空间由容量层、 性能层常驻区、性能层非常驻区三部分组成，cap / perf thin / perf thick 三种不同层级的各种负载阈值不同：

| 类型\负载  | Medium | High | Very High |
| ---------- | ------ | ---- | --------- |
| Cap        | 75%    | 85%  | 95%       |
| Perf Thick | 30%    | 50%  | 95%       |
| Perf thin  | 50%    | 60%  | 85%       |

数据块分片的分配、迁移、恢复策略，对于三种层级空间都是适用的，因为空间负载是相互独立计算的。

举个例子说明分片分配策略，假设一个集群包含 c1 c2 c3 三节点，节点都在同一个机架上，ring id 分别是 1 2 3，创建一个 2 副本精简制备卷，接入点在 c1 上，prefer local = c1，禁止 cap 直写和迁移，在执行全盘写的过程中，perf thin PExtent 分片分配位置随负载变化如下：

| step | cid 1 ratio | cid 2 ratio | cid 3 ratio | perf loc              | prefer  cid load | note                                                         |
| ---- | ----------- | ----------- | ----------- | --------------------- | ---------------- | ------------------------------------------------------------ |
| 1    | [0, 0.5]    | [0, 0.5]    |             | [1, 2]                | 低               | 遵循局部化分配                                               |
| 2    | [0.5, 0.6]  |             | [0, 0.1]    | [1, 3]                | 中               | 放弃局部化，遵循本地优先 + 拓扑安全 + 容量均衡，c3 负载低于 c2 所以被选择 |
| 3    |             | [0.5, 0.6]  | [0.1, 0.2]  | [2, 3]                | 高               | 从 step 3 开始放弃本地优先，遵循拓扑安全 + 容量均衡          |
| 4    |             |             |             | [3, 1], [3, 2]        | 高               | c3 在负载跟 c1 和 c2 相同前，总会被分配，loc 交替出现在 [3, 1]、[3, 2] |
| 5    |             |             |             | [1, 2], [2, 3],[3, 1] | 高               | 此时 c1 c2 c3 负载相同，loc 交替出现在各个节点上             |

### 拓扑安全分配

拓扑安全分配（topo safe allocation）指的是尽可能将分片分配到拓扑距离最远的节点上。在集群服役期间，除了存储节点可能发生物理故障，接入交换机（TOR，每个机箱一个接入交换机）、机柜 UPS 电源（每个机架一个独立电源）都有可能异常，为了提供更高水平的系统可靠性与可用性，需要将分片尽可能分布在物理拓扑的不同分支上。

在集群配置节点拓扑信息之后，就可以据此计算集群中节点之间的拓扑距离。一个节点要么因为没有配置拓扑信息被默认放在可用域中，要么一定被放在某个机箱中，两两节点之间的拓扑距离（topo distance）初始值为 0，依照如下方式计算：

* 若两个节点不在同一个可用域（zone），那么 topo distance - 256；
* 若两个节点不在同一个机架（rack），那么 topo distance - 16；
* 若两个节点不在同一个机柜（brick），那么 topo distance - 1；

分片间的拓扑距离值越小，认为分布越安全。举个例子，若一个 pextent 已有分片在节点 c1 和 c2，依照拓扑安全分配，第 3 个分片会被分配到跟 c1、c2 拓扑距离最小的节点上，若这样的节点有多个，再按照本地优先 / 局部化 / 低载优先原则来进一步筛选。

### 本地优先分配

本地优先分配（local prefer allocation）指的是，若 PExtent 的 prefer local 节点健康、非 isolated（[隔离状态](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.lbq4aoyc65wp)）并且有剩余空间，那么一定将分片分配到 prefer local 节点。

不同制备类型的 PExtent 的 prefer local，会有不一样的初值，具体规则如下：

* 厚制备（thick）

    常驻缓存卷的 perf / cap PExtent 和厚制备卷的 cap PExtent 在 volume 刚创建时就会分配分片，此时 PExtent 的 prefer local 等于创建请求中指定的 volume 的 prefer local，由于一般情况下不会显式指定，这些 PExtent 的 prefer local 都会是 0。

* 精简制备（thin）

    精简制备卷的 perf / cap PExtent 和厚制备卷的 perf PExtent 在数据块初次写时才会分配分片，此时 PExtent 的 prefer local 优先以 volume 的 prefer local 为准，若该值为 0，以发起写申请的 lease owner 节点为准。

当出现以下 3 种情况时，pextent 的 prefer local 会被变更：

1. 主动更新

    虚拟卷更新时指定了不为 0 的 prefer local，一般是通过 zbs cli 来更新。

    大部分情况下，volume 的 prefer local 都是 0，持有的 pextent 的 prefer local 可能各不相同，但若指定了 volume 的 prefer local，那么该 volume 的所有 pextent 也会有相同的 prefer local。

2. 访问感知（access mode perception）

    即快照卷模板化，被模板化的快照卷持有的 pextent 的 prefer local 将被重置为 0，有 2 种方式：

    1. 快照卷更新时指定了 alloc even。除了通过 zbs cli 更新，目前一般的行为是在 SMTX OS（ELF）中主动将虚拟机创建/转换为模板；
    2. 快照卷被大量克隆。克隆超过默认次数（10）的快照卷即使 alloc even 属性还是 false，也会被模板化。

    当一个快照卷被模板化，快照卷内的数据不应该集中存储，这是因为一个模板（不论上述哪种方式得来）预期都会有大量克隆，克隆之后的虚拟卷通常会分布到集群的多个节点上，集中存储无法获得本地化 IO 的好处，反而可能会造成局部过载。

    因此模板卷（even volume）的分配策略与一般的虚拟卷不同，只会遵循拓扑安全和完全均匀策略。完全均匀指的是所有模板卷的所有均匀数据块分片，将被均分到集群中的每个节点。

3. 本地感知（access point awareness）

    尽管以 PExtent 为粒度的分片分配，不能保证同一个 volume 内的 PExtent 分片位置相同，但作为一个块存储系统，通常一个 volume 内所有的 PExtent 都会具备相同的本地访问（在 SMTX OS 下是数据卷关联的虚拟机所在物理节点，在 SMTX ZBS 中是 lun / namespace / ... 协议接入的物理节点），因此可以根据 volume 的接入点（access point）来变更其所持有 PExtent 的 prefer local。

    集群中每个 access 每 1h 上报一次这段时间内 IO 次数超过 3600、平均大小超过 511Byte 的 volume（即 iops > 1，包含读写）给 meta，meta 会记录这些接入点，连续上报 6 次的接入点成为活跃接入点。当一个接入点初次变为活跃时会触发 local transfer 事件，此时若 volume 的接入点个数不超过默认数量（3）且 volume 没有显式指定 prefer local，那么会将该卷持有的所有非 COW 状态的 cap / perf pextent 的 prefer local 都变更成该活跃接入点，并撤回相关 lease。
    
    local transfer 通常发生在虚拟机开机或者热迁移到一个新物理节点上重新建立连接后稳定使用的场景。一般来说，一个 volume 只会有一个活跃接入点，zbs 采用稳定链路接入策略保证在计算端稳定时总会从同一个 access 上访问数据以保证本地感知可以得到一个稳定的结果（尽管临时故障时接入点会切出本地，但基本都能在 6h 内切回本地）。
    
    Oracle RAC 等小范围共享卷的场景可能会有 1 到 2 个活跃接入点，此时的 Local Transfer 不会变化 Prefer local 为当前正在接入的接入点的 PExtent，这意味着两个 RAC 实例健康的情况下总是各自持有一部分 PExtent 的 Prefer local）。特别地，有些场景如 Xen/VMware/OpenStack/CAS/FusionCompute 平台的 iSCSI DataStore 模式将一个 volume 作为一个池使用，不同节点上的虚拟机仅访问其中的部分数据，卷的活跃接入点可能会大于等于 3 个，此时持有的 PExtent 只会在初次写入时用第一个接入节点作为 prefer local，后续不再变更。
    

### 局部化分配

局部化分配（localization allocation）指的是将 prefer local 相同的 PExtent 的分片分配到相同的、固定的节点上，以此来降低节点故障时的影响范围。考虑分片完全均匀分配在每个节点上的场景，此时每个 Volume 中的分片会均匀分散到整个集群，任意一个节点都持有集群中任一 Volume 的部分数据，这样当一个节点异常（节点下线、磁盘异常等）时，集群中所有的 Volume 都会受到影响，且节点故障的影响范围也会随着集群规模的扩张而增大。

局部化分配策略也有明显的劣于大部分分布式系统采用的完全均衡策略。就是在集群节点规模较大的时候，而单一节点上的磁盘能力较低的时候，单一虚拟卷能动员的硬件能力会被局限在有限的节点上而无法发挥整个集群的能力。但是考虑到：

* 即便数据均匀分散到所有节点上，性能上限也会受到网络的限制，技术的发展趋势是磁盘的能力远超过网络的发展能力，现代磁盘就不存在这样的瓶颈（ 2 x NVMe 能提供的 IOPS 和带宽超过 25 G 网卡的上限）；

* 即便是较差的磁盘，存储集群通常也不会只为单一虚拟卷服务，均衡分布带来的 IO 互相干扰的现象对集群整体能力上限更为不利。  


因此 ZBS 选择局部化策略作为空间分配的基础策略。局部化策略逻辑希望达到的效果为，例如 4 节点 2 副本集群，A，B，C，D。A 上的 VM 关联数据均分配在 AB 两个节点，C 上 VM 的关联数据均分配在 CD 两个节点。这样 A 与 C 上 VM 的 IO 互相隔离，不会产生干扰。后续扩容添加 EF 等节点，E 上 VM 数据也分配在 EF 上。

在实现上，存在多个候选节点跟已有节点的拓扑距离相等时，才会进一步根据局部化分配策略选择下一个分片的分配节点，具体来说，就是按照 ring id 的相对顺序选择跟上一个分片最接近的节点。ring id 为 Chunk 的拓扑属性，代表它在拓扑环上的位置，它不是静态不变的，在每次集群的拓扑结构发生变化之后，chunk 的 ring id 都有可能发生改变以尽量适应副本分配规则达到一个相对均衡的分配结果。生成过程参见 [面向拓扑结构变化的副本分配策略调整](https://docs.google.com/document/d/1RgOG42tJZyNSEDLdpWgnJzPbKdSpDAbxvWEUGSbo-VM/edit) 。

举个例子，若有 ring id 为 1，2，3，4 的 4 个健康节点，四个节点之间拓扑距离相等，初始选择了 local uuid = 2，在所有节点都有剩余空间时，下一个被选中节点始终为 3。初始 local uuid = 4 时，下个被选中的节点始终为 1。如果空间不满足需求，则选择与 local uuid 最近的第一个有剩余空间的节点。

当集群拓扑结构、节点健康状态不变且剩余空间充足时，prefer local 相同的 pextent 分片得到的期望局部化分片列表是相同且固定的，这也意味着 prefer local 不同的 pextent 的分片位置的重叠概率是很低的，不同 Volume 之间占用的物理资源相对独立，这除了可以缩小故障影响范围，还便于按需调整各个 Volume 的 Qos 策略，让集群性能随节点数线性增长。

### 写时复制分配

zbs 在对卷发起快照/克隆时，会先将相关数据块标记成 COW 状态，只会有少量元数据拷贝，直到初次写时才通过 COW 行为复制数据。

目前 ZBS 在数据层面的 COW 是由 LSM 实现的，所以对于 PExtent（不论是 Perf 还是 Cap）的 COW 都一定只能在 Parent Extent 所在的节点上发生（这样可以保证 Child Extent 创建时和 Parent Extent 处于同一个 LSM 管理范围之内可以顺利完成数据继承），不会遵循前述空间分配规则。需要注意的是，此时 Child 和 Parent 共享的数据在它们的位置分离之后将会各自持有一份，带来空间放大的负面影响，因此在数据均衡时会尽量避免迁移此类 Extent 分片。

## 分片恢复策略

Meta leader 借助 Recover Manager 来生成恢复命令并放入待下发队列中，后续由 access manager 在回复心跳时下发给 access 执行。Recover Manager 运行在独立的 recover 线程中，内部维护冷热 2 个待恢复数据块队列：

1. passive list：周期性扫描所有数据块时发现的可能需要恢复的数据块队列；
2. active list：已经确认需要 recover，但尚未生成恢复命令的数据块队列，被 access 主动剔除分片的数据块将直接进入该队列。

perf thin / perf thick / cap 类型的数据块都有可能进入这两个队列，队列按以下规则选出优先生成恢复命令的数据块，即对于任意两个待恢复数据块：

1. 若二者活跃分片冗余度不相等，优选最小的；
2. 若二者活跃分片冗余度都不等于 0 并且其中一个数据块的分片在维护模式节点上，优选不在维护模式节点上的；
3. 若二者有一个是 perf thick（prioritized），即属于优先卷的数据块，优先它；
4. 若二者有效分片冗余度不相等，优选最小的；若二者期望分片冗余度不相等，优选最大的；
5. 若二者活跃分片冗余度都等于 0 并且其中一个数据块的分片在维护模式节点上，优选不在维护模式节点上的；
6. 优选 pid 最小的。

其中，活跃/有效/期望分片的数量定义如下：

* 活跃，在 alive_location 中的分片数量；
* 有效，在 location 中的分片数量（在大多数情况下等同于存活，除非出现了尚未处理的错误例如节点失联，当时尚未 IO 或通过恢复时间触发分片剔除时，则有效 > 存活）；
* 期望，用户设置的 PExtent 的期望冗余度，即 expect_segment_num。

根据冗余类型不同，他们的冗余度（代表可容忍同时异常的最大分片数量，计算为负数时取 0）计算方式如下：

* replica：(alive / existed / expected segment num) - 1；
* ec：(alive / existed / expected segment num) - k，k 是 ec 数据块的个数；

Recover 线程以默认每 4s （FLAGS_reposition_trigger_interval_ms）定时执行以下事件：

1. 缓存最新的所有数据块的 pid 信息、所有节点的状态/拓扑信息、清理无效 RecoverDstMgr；

   恢复命令是周期性生成并下发的，可以容忍这些信息不是实时更新的；

2. 清理执行失败/成功的恢复命令；

   1. 若 recover dst 持有这个数据块分片，说明执行成功；
   2. 若数据块已被标记为失效比如已经被回收了或 epoch 匹配不上，认为无需恢复，主动清理；
   3. 若命令执行超过默认的 18 min 还没完成，认为超时失败；
   4. 若 lease owner 上不再持有，表明在 Access/Chunk 侧执行失败，主动清理。

   不论成功还是失败，都需要释放恢复预留空间，若恢复失败，还需要更新对应的候选目的节点黑名单。

3. 若 active list 不为空，尝试生成恢复命令并将命令放入待下发队列；

   1. 每个节点作为 src/dst 每轮最多只能生成默认的 200 * 1.1 = 220 条 cap recover cmd 和 400 * 1.1 = 440 条 perf recover cmd，单轮所有节点最多只能生成默认的 1024 条 recover cmd（cap / perf 共计）；
   2. recover manager 会记录每个 PExtent 是否有恢复命令正在执行，以避免重复生成，生成命令时会用 Space Transaction 在相关节点上申请恢复预留空间（体现在源与目的节点的 tx/rx pid 列表中）；

4. 若满足扫描条件，根据 [8.4.1 Trigger Condition](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.46nmcqhhytdw) 将待恢复数据块放入 passive list，若 passive list 不为空，清空 active list 并交换 passive / active list，并且立即尝试生成恢复命令并将命令放入待下发队列。

Recover 可以通过 Meta 命令行关闭，默认使能。

### 触发条件

一般来说，当 Meta 发现 PExtent 的现有分片数量小于预期分片数量时，将会触发恢复动作（LExtent 不存在物理实体分片，所以没有恢复/迁移的逻辑）。根据 PExtent 是否 ever exist，具体的恢复触发条件并不相同：

* ever exist

  当且仅当活跃分片数满足恢复最低要求且活跃分片数小于预期分片数时恢复。

  活跃分片指的是在最近 10 min 内被 LSM 上报健康的存活分片。恢复最低要求指的是 replica 至少有 1 个、ec 至少有 k 个活跃分片时才能触发恢复，否则无法选出恢复源节点。

  常见的恢复场景是 Access 在 IO 过程中发现部分分片失败，比如初次读/写时 sync gen 时由于突然下电/程序崩溃等上报了较低的 gen 或者因为拔盘/磁盘损坏/网络失联等各种原因导致的写入失败，将会剔除分片并在随后恢复到健康节点上。

* non ever exist

  在有存活分片的前提下，若健康分片数小于预期分片数或异常分片数大于 0 时恢复。

  其中，健康分片指的是所在节点状态是健康（Connected Healthy）的分片，节点状态信息从 chunk table 中查询。异常分片指的是超过 10 min 未被上报或被上报 offline / error / corrupt 的存活分片。一般来说，刚分配的 non ever exist PExtent 不在 LSM 上真实存在，不会被上报，但快照/克隆后被迁移到其他节点的 PExtent，此时虽然还是 non ever exist，但在目的节点上真实承载数据，其健康状态会被定期上报。

### 维护模式

集群升级期间需要滚动重启各个 chunk，正在 IO 的 PExtent 在重启节点上的分片会失去响应并被剔除，在 IO 负载较高且分散的情况下，可能会带来大量数据恢复。在 5.1.x 之前，依赖集群升级模式来尽量减少这种预期内的运维操作带来的不必要恢复，在 5.1.x 之后，维护模式（[2.4.4 Maintenance](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.5nu389jru0gi)）取代了升级模式，推迟处于维护模式中的节点上的副本剔除动作。

在节点/chunk 下线前先进入维护模式，并在重新上线后退出维护模式，在此期间：

1. 若冷数据的失联分片在维护模式节点上，不论是 replica 还是 ec shards，都不会触发恢复；
2. 若冗余类型为 replica 的 PExtent 在维护模式节点上有数据更新且由于未能及时响应导致副本剔除，Meta 会将这个失联分片打上 RIM（Replica In Maintenance）标记，并根据 PExtent 的期望副本数与存活副本数有着不同的恢复策略：
   1. 2 副本数据块只有 1 个在维护模式节点上的副本失联：在节点进入维护模式超过 60s 或退出维护模式后才会触发恢复；
   2. 3 副本数据块只有 1 个在维护模式节点上的副本失联：在节点退出维护模式后才会触发恢复。
3. 若冗余类型为 ec 的 PExtent 在维护模式节点上有数据更新且由于未能及时响应导致分片剔除，会立即触发恢复（ec 暂不支持敏捷恢复和临时副本）。

带有 RIM 标记的失败副本在存活副本数量不足的情况下不会被 GC，而是留着敏捷恢复。具体来说，在节点/chunk 重新上线并退出维护模式后，Meta 会对带有 RIM 标记的副本下发恢复目的节点设置为 RIM 所示节点的 Recover 命令，通常情况下，因为不是存储介质故障，带有 RIM 标记的失败副本可以和对应的临时副本组成完整数据，所以 Access 可以触发敏捷恢复，仅重放运维期间产生的增量数据即可得到一个完整的数据副本。对于 EC 类型的数据分片，因为数据本身无法以增量形式存储，暂不支持临时副本创建和敏捷恢复，无需设置 RIM 标记。

### 恢复目标节点管理

从 zbs 5.3.x 开始，引入了 RecoverDstMgr 来管理 recover dst，它有以下 2 种作用：

1. zbs 5.3.x 引入了临时副本机制，为了发挥临时副本保护失败副本的作用，若待恢复数据块存在临时副本，RecoverDstMgr 需要优先将 recover dst 设置到临时副本记录的失败副本所在节点并优先尝试敏捷恢复，若敏捷恢复失败，再尝试发起普通恢复；
2. 考虑如下场景，对于某个待恢复数据块，所有候选目的节点中优先级最高的节点与 lease owner 无法通信，导致 recover 无法完成，但它跟 meta leader 通信正常，那么 meta leader 始终会下发同一个 recover dst 的恢复命令，recover 将一直无法完成。因此需要 RecoverDstMgr 为每个待恢复数据块维护一个候选目的节点黑名单，对于每个候选目的节点，若恢复失败将被加入黑名单，若所有候选节点都在黑名单中，那么重置黑名单并从头逐个尝试。

每个待恢复数据块有自己独立的 RecoverDstMgr，在各个 RecoverDstMgr 内部，敏捷恢复和普通恢复有各自独立的候选目的节点黑名单。一般来说，每个候选节点只要尝试一次失败，就会被加入黑名单。特别地，为了在本地拔盘时能优先将数据恢复到本地，会给 prefer local 多一次豁免权，即只有两次下发以 prefer local 为 recover dst 的普通恢复命令都执行失败时，才将其加入普通恢复的黑名单中。

每个 RecoverDstMgr 按照以下优先级给出 recover dst：

1. 若存在，以未被加入黑名单的、非 isolated（即没有被隔离的 chunk，详见 [Chunk 服务隔离](https://docs.google.com/document/d/1aBW5h1vkWEixB57QH--oIbddY5AC7EPJNQoyMRMNUJA/edit#heading=h.kt54xirw6sxy) ）、带有 RIM 标记的失败副本所在节点作为 agile recover dst；
2. 若存在，以未被加入黑名单的、非 isolated、temporary replicas 和 lossy temporary replicas 记录的失败副本所在节点作为 agile recover dst；
3. 若存在，以未被加入黑名单的、非 isolated、还有剩余空间的节点作为 normal recover dst；
4. 若以上 3 种情况都无法选出有效 recover dst，将忽略 isolated 限制再选一次，并仍然保持 agile recover 先于 normal recover；
5. 若以上 4 种情况都无法选出有效 recover dst，将忽略黑名单与 isolated 限制，在所有有剩余空间节点中选择 normal recover dst。

简而言之，敏捷恢复到非 isolated 节点 > 普通恢复到非 isolated 节点 > 敏捷恢复到 isolated 节点 > 普通恢复到 isolated 节点。

### 取消已下发迁移命令

从 smtx os 6.2.0 / zbs 5.7.0 开始，引入了 cancel migrate cmd 机制，让有限的 IO 资源尽量放在优先级更高的内部 IO 事件上（recover > migrate），保证数据安全，提高集群升级速度。当 recover manager 发现集群中存在如下情况时，将取消关联的已下发迁移命令：

1. 待恢复 pextent 因所属 lextent 存在已下发但未完成的迁移命令导致无法生成恢复命令时，先取消这个迁移命令；
2. 下发恢复命令时，取消同一 lease owner 的、已下发的、replace cid 还没到超高负载的迁移命令；
3. 节点拓扑变更或退出维护模式时，取消 replace / src / dst cid 是该节点的所有迁移命令（不论是否已下发）。

access 也会调整相应的优先级，在先收到的迁移命令和后收到的恢复命令之间，先执行后者。

## 分片迁移策略

在 ZBS 集群运行过程中，不论集群此时处于何种负载，当集群中没有待恢复数据块时，meta leader 复用 recover manager 来生成迁移命令并放入待下发队列中，后续由 access manager 在回复心跳时下发给 access 执行。recover manager 内部维护冷热 2 个待迁移数据块队列：

1. passive list：周期性扫描所有数据块时发现的需要迁移的数据块队列，与恢复不同，此时已经选出迁移命令的 src / dst / replace cid；
2. active list：在下发前会 double check 是否需要先恢复。

meta leader 的一个后台线程会周期性地扫描所有副本/节点，按某种迁移策略选出待迁移数据块的 src / dst / replace cid 生成迁移命令，并为其选定/分配 lease 后，通过心跳下发给 access 执行。

Recover 线程的周期性执行事件在上一节中有所描述，不再赘述。不同的是：

* 生成阶段：每个节点最多只能生成默认的 256 条 migrate cmd，所有节点最多只能生成默认的 1024 条 migrate cmd（cap / perf 共计）；
* 下发阶段：每个节点作为 src/dst 每轮最多只能下发默认的 200 条 cap migrate cmd 和 400 条 perf migrate cmd。

迁移扫描在集群中有节点移除时，默认每 4s 触发一次。对于其他类型的迁移，当集群进入高负载前，默认每 15 min 触发一次，高负载后，每 2 min 触发一次。

Migrate 可以通过 Meta 命令行关闭，默认使能。

### 触发条件

迁移的触发条件包括以下 6 种：

1. 局部化修复。当集群进入中负载前，期望通过局部化修复让非均匀卷的数据块分片处在固定的节点列表，局部化修复包含本地化修复和拓扑修复。
2. 本地化修复。当集群进入高负载前，期望通过本地化修复让非均匀卷的数据块在 prefer local 上有分片，提供数据访问性能。
3. 拓扑修复。在集群进入超高负载前，期望数据块分片分布在物理拓扑的不同分支上以提高数据安全。
4. 节点容量均衡。在集群进入中负载后，期望各节点容量尽量均衡，充分发挥集群多节点存储上限。
5. 均匀块的数量均衡。初次模板化得来的均匀卷期望通过迁移将持有的数据块分散到各个节点。
6. 节点移除。待移除节点需要将自身数据迁移到集群中的其他节点上，在数据全部前移除，将会停止以上 5 种类型的迁移。

与恢复一样，每个 volume 的 lextent 在一次迁移扫描过程中最多只会生成一条 PExtent 的迁移命令（要么 cap，要么 perf）。

### 非均匀卷迁移

集群中大部分数据卷（即非均匀卷）的数据块，依照集群负载不同，执行目标不同的迁移策略：

1. 低负载：① 局部化修复 ② 拓扑修复 ③ 本地化修复

   若数据块 prefer local 不健康或没有剩余空间，尽管无法执行局部化修复，但仍会为其尝试拓扑修复以保证拓扑安全。

2. 中负载：① 拓扑修复 ② 本地化修复 ③ 节点容量均衡

   若集群没有配置拓扑信息，拓扑修复将被直接跳过，执行本地化修复扫描，此时若生成了迁移命令，之后的节点容量均衡将会跳过。

3. 高负载：① 拓扑修复 ② 节点容量均衡

   此时放弃了本地化修复，若集群没有配置拓扑信息将直接执行节点容量均衡迁移扫描。

4. 超高负载：① 节点容量均衡

   此时放弃了本地化修复和拓扑修复，不会为了让本地有分片或达到更高的拓扑等级而触发迁移，仅有容量均衡迁移来避免节点过载（可能为了降低负载迁出本地分片，但不会让拓扑降级）。

补充说明：

1. 低负载时，若数据块 prefer local 不健康或没有剩余空间，尽管无法执行局部化修复，但仍会为其尝试修复拓扑以保证拓扑安全；
2. 中负载时，若集群没有配置拓扑信息，拓扑修复将被直接跳过，执行本地化修复扫描，此时若生成了迁移命令，之后的节点容量均衡将会跳过；
3. 高负载时，因为放弃了本地优先原则，所以若集群没有配置拓扑信息将直接执行节点容量均衡迁移扫描。



分片分配策略中，在集群进入中负载才放弃局部化，进入高负载才放弃本地优先，迁移策略也需要满足这个大的原则。不同的是，迁移策略中需要有一个弹性边界，用以避免节点容量在负载转换边界时因小流量 IO 流入出现迁移策略不断转化进而导致分片反复迁移。弹性边界是针对节点而非集群而言，以 cap layer 的默认参数 medium load = 75%、 high load = 85%、very high load = 95%、relaxation = 5% 为例，来说明弹性边界的策略：

1. 当集群处于低负载 [0, 75)，集群中处于严格低负载 [0, 70) 的节点可以保证相关数据块的分片分布满足局部化，但处于低中边界 [70, 75) 的节点无法保证；
2. 当集群处于中负载 [75, 85)，集群中处于严格中负载 [75, 80) 的节点可以保证相关数据块存在本地分片，但处于中高边界 [80, 85) 的节点无法保证；
3. 当集群处于高负载 [85, 95)，集群中处于严格高负载 [85, 90) 的节点可以保证相关数据块的分片分布满足拓扑安全（包含双活域间分片 2 ：1 的特点），但处于高超高边界 [90, 95) 的节点无法保证；

#### 局部化迁移

局部化迁移（migrate for localization） 指的是：当集群处于低负载，若集群拓扑结构、节点健康状态、数据块的 prefer local 发生变化，非均匀卷的数据块分片将会通过局部化修复迁移来达到新的期望局部化分布。

通过以下 3 个步骤选取分片放置节点构成的分布称为期望局部化分布：

1. 若 prefer local 处于健康状态且可以容纳新分片，那么 prefer local 作为第 1 节点，否则选取负载最低的节点作为第 1 节点；
2. 从剩余节点中选取与第 1 节点拓扑距离最近的节点作为第 2 节点，若存在多个，那么按照 ring id 的相对顺序从拓扑环上选取与第 1 节点最近的节点作为第 2 节点；
3. 与步骤 2 同理，从剩余节点中选取与第 k - 1 节点拓扑距离最近且在拓扑环上最近的节点作为第 k 节点，直到选出 n 个分片放置节点构成期望局部化分布。

对于给定 prefer local 的每个数据块而言，当集群中的拓扑结构和节点健康状态不变时，它的期望局部化分布也一定保持不变。



局部化迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足在局部化列表中，优选局部化列表中靠前的位置；
* replace cid 必须满足不在局部化分布中，replica 优选不健康、isolated 节点，ec 优选健康、非 isolated 节点；
* src cid 在 replica 时必须满足健康，优选非 isolated，在 ec 时必须是 replace cid 且健康。



补充说明：

1. 当数据块 prefer local 不处于健康状态，将不会触发该数据块的局部化迁移；
2. 数据块分片满足局部化后某个分片所在节点进入 isolated 状态如亚健康或维护模式，将不会触发局部化迁移；
3. 若集群中存在处于健康状态但没有剩余空间的节点或是 isolated 节点，由于无法作为 migrate dst，当它们处于数据块的期望局部化分布中，该数据块分片可能不会满足局部化分布，属于预期行为；
4. 在实现上，局部化迁移除了满足数据分布局部化，也能达到本地有分片、分片拓扑安全的目标。

#### 本地化迁移

本地化迁移（migrate for prefer local）指的是：当数据块的 prefer local 不超过严格中负载并且健康、非 isolated 时，将使其拥有分片以提高数据访问性能。

本地化迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须是 prefer local 且健康、非 isolated；
* replace cid 在 replica 时优选拓扑等级低、不健康、isolated 节点，在 ec 时优选健康、非 isolated 节点；
* src cid 在 replica 时必须满足健康，优选非 isolated，在 ec 时必须满足是 replace cid 且健康。

#### 拓扑修复迁移

拓扑修复迁移（migrate for repair topo）指的是：当集群负载不超过超高负载或者拓扑等级更高的节点不超过严格高负载时，将数据块分片迁移成更高拓扑等级的分布。

拓扑修复迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足健康、非 isolated 且让迁移后分片拓扑等级更高，优选 prefer local 节点；
* replace cid 必须满足不是 prefer local 且让迁移后分片拓扑等级更高，在 replica 时优选不健康、isolated 节点，在 ec 时优选健康、非 isolated 节点；
* src cid 在 replica 时必须满足健康，优选非 isolated，在 ec 时必须满足是 replace cid 且健康。



补充说明：

1. 集群中任一节点未配置拓扑信息，都会跳过拓扑修复迁移；
2. 如果 prefer local 作为 replace cid 能够不降低迁移后的分片拓扑等级，那么一定存在一个同级的其他节点可以作为 replace cid，因此可以直接规定 replace cid 不选 prefer local。
3. 拓扑修复迁移会导致节点容量发生变化，为避免可能的反复迁移，在一次迁移扫描中若生成拓扑修复迁移命令，那么不再执行节点容量均衡迁移；
4. 双活环境下的拓扑修复迁移可能产生跨域迁移以保证优先可用域与次级可用域副本比例 2 ：1。

#### 节点容量均衡迁移

节点容量均衡迁移（migrate for rebalance）指的是：当集群从中负载开始，将数据块分片从较高负载节点迁移到较低负载节点上，让集群内的数据块分片相对均衡地分布在各个节点上。

节点容量均衡迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足健康、非 isolated 且迁移后分片拓扑等级不降低，优选更低负载的节点；
* replace cid 必须满足迁移后分片拓扑等级不降低，优选更高负载、isolated 的节点；
* src cid 在 replica 时必须满足健康，优选非 isolated，在 ec 时必须满足是 replace cid 且健康。

根据负载对集群中的健康节点升序排序，经由以上规则选定的每一组 replace / dst cid，可以得到候选数据块集合，即 replace cid 与 dst cid 持有数据块的差集，另外也可以得到本组最大迁移容量。将 replace cid 的负载记为 replace_ratio，有效容量记为 replace_valid，dst cid 的负载记为 dst_ratio，有效容量记为 dst_valid，集群平均负载记为 avg_ratio，当 dst_ratio >= avg_ratio，本组最大迁移数量等于 (replace_ratio - dst_ratio) / 2 * min(replace_valid, dst_vaid)，否则等于 min(replace_valid * (replace_ratio - avg_ratio), dst_valid * (avg_ratio - dst_ratio))。

除了容量上的限制，根据 replace cid 的负载不同，可允许迁移的分片类型也不同：

* 中负载：不允许迁移作为 parent 的 pextent 分片、位于 prefer local 上的分片、lease owner 上的分片；
* 高负载：不允许迁移 lease owner 上的分片；
* 超高负载：所有分片都可能被迁移，另外此时还允许 src cid 是 isolated（但优先级是最低的）。

另外，不论处于何种负载，总是按照先 thick extent 再 thin extent without parent 最后 thin extent with parent 的顺序选取候选数据块。



补充说明：

1. 尽管节点容量均衡迁移关注的是各节点各种卷中数据块占用容量的相对一致，但为了简单处理，只有在非均匀卷的数据块会被迁移，一般来说集群中的均匀卷不会太多；
2. 一次迁移中若一个节点若作为 dst cid 已经跟某个 replace cid 生成迁移命令，那么该节点不会再作为其他 replace cid 的 dst cid；
3. 为避免来回迁移，当集群未处于超高负载且各节点之间的负载不超过指定大小（默认为 0.01）或者是各节点之间已使用容量不超过指定大小（默认为 5GiB），直接跳过不再触发节点容量均衡迁移；
4. 双活场景下的节点容量均衡迁移只会发生在各个可用域内部，即生成 migrate cmd 的 src / dst / replace cid 总是拥有相同的 zone id。

### 均匀卷迁移

一般来说，集群中的均匀卷数量较少，不会对节点负载太多影响，所以只尝试触发以 ① 拓扑修复 ② 数量均衡为目的的迁移。均匀卷的拓扑修复与非均匀卷策略一致，不再赘述。其中，数量均衡指的是所有均匀卷的数据块分片需要相对均匀地分布在整个集群，每个节点持有的均匀块数量大致相等。

均匀块数量均衡迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足健康、非 isolated 且迁移后分片拓扑等级不降低，优选持有均匀块更少的节点；
* replace cid 必须满足迁移后分片拓扑等级不降低，优选持有均匀块数量更多、isolated 的节点；
* src cid 在 replica 时必须满足健康，优选非 isolated，在 ec 时必须满足是 replace cid 且健康。

根据所持均匀块数量对集群中的健康节点升序排序，经由以上规则选定的每一组 replace / dst cid，可以得到候选数据块集合，即 replace cid 与 dst cid 持有均匀数据块的差集，另外也可以得到本组最大迁移数量。将 replace cid 持有的 even extent 数量记为 replace_num，dst cid 持有的记为 dst_num，集群中各个节点平均持有的 even extent 数量记为 avg_num，当 dst_num >= avg_num，本次最大迁移数量等于 (replace_num - dst_num) / 2，否则等于 min(replace_num - avg_num, avg_num - dst_num)。



补充说明：

1. 与节点容量均衡迁移不同，不管集群处于任意负载，都可以迁移各种类型的均匀块；
2. 为避免来回迁移，当集群中各节点之间持有的均匀块数量不超过指定大小（默认为 4）时，不再触发均匀块数量均衡迁移。

### 节点移除迁移

节点移除迁移（migrate for removing chunk）指的是：预期内的下线节点需要迁出自身所有数据直到不再持有任何数据块时才能从集群中注销。

节点移除迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足健康、有剩余空间，优选非 isolated、拓扑等级较高的节点；
* replace cid 一定是待移除节点；
* src cid 在 replica 时优选非 isolated、replace_cid 节点，在 ec 时一定是 replace cid；

节点移除迁移的选取规则基本上复用了分片恢复策略的选取规则。不同于其他迁移，节点移除迁移是有可能在其他节点容量不足时迁入拓扑等级更低但仍有剩余空间的节点上。

另外，尽管 zbs 允许多个节点同时进入 REMOVING state，但集群中一次只会迁出一个节点上的数据块，直到节点不再持有任何数据块后，才会进入 IDLE state，迁移下一个 REMOVING state 节点。

## 分片下沉/提升策略



提升貌似没啥策略



新写入数据会优先在缓存里，什么情况下会下沉到 HDD，比如多长时间不发生读写了。以及冷数据满足什么情况才会被 promote 到 SSD

分片回收跟下沉基本是配合的



## 分片回收策略

lextent / pextent 被 gc 的一般流程

1. ref status 被标记成 unused。根据 db snapshot 判断 pextent / lextent 是否被引用；
2. garbage 属性被置为 1。同时这个 pextent 也进入 valid = 0 的状态，后续一定不会被 stage，也就不会执行一些涉及到 pextent 内存变动的 rpc；
3. 在 meta 层面被 gc，即这个 lextent / pextent 在 meta db 和 pextent table 中被删除了，lextent 到这个阶段就结束了；
4. 在 lsm 层面被 gc，即 chunk 上报了 meta 认为它不该持有的 pextent，meta 给他发送一个 gc cmd。

### 标记 lextent garbage

每一轮扫描，lid_map 中除了被 volume 引用的每个 lextent 的 ref status 都是 unused。对于这些 lextent，如果它的 stage = 0 && valid = 1，并且它的 cap / perf pextent 的所有 child pextent 都是 ever exist 的（也就是读写 child 一定不需要访问 parent），它的 garbage 属性才会被标记为 1。

### 标记 pextent garbage

每一轮扫描，pid_map 中不满足以下 3 个条件的 pextent 的 ref status 才会是 unused。

1. 被 volume 引用的每个 lextent 的每个 cap / perf pextent 的 ref status 都不会是 unused；
2. 对于每个非 unused 且 ever exist = false 的 pextent 而言，溯源到第一个 ever exist = true 的祖先的这一路上的每个祖先 pextent，都不会是 unused（这样才能保证读 child 能读到数据），即使他们没有被任何有效 lextent 引用；
3. 对于每个 lextent 在这一轮可以被标记成 garbage 的 cap / perf pextent 而言，这一轮它们的 ref status 一定不会是 unused 的（这样可以保证 lextent 一定比它的 pextent 先 gc）；

对于 get reft = unused 的每个 pextent，如果它的 stage = 0 && valid = 1，并且自己不是临时副本的话（临时副本会在他关联的 pextent 被删除时一起删除），它的 garbage 属性才会被标记为 1。

如果一个 pextent 没有被标记成 garbage（说明他的 lextent 存在，只要有 lextent 引用，pextent 的 ref status 就不会是 unused），paired pextent 也不会被标记成 garbage。

### gc garbage

在 SweepGarbage 中，对于这些 garbage = 1 的 lextent / pextent，会先删除 db 中的相关数据，同步等待 access 放弃持有的相关 lease、删除 meta 持有的相关 lease 后，删除内存 lextent / pextent table 中的相关数据。

## IO 限流

### 业务 IO 限流

从 zbs 5.6.0 开始，非常驻缓存卷的大部分业务 IO 将会写在性能层非常驻缓存区（perf thin space），当 perf thin space 下沉腾出空间的速率赶不上业务 IO 的写入速率时，可能出现容量层仍有空间，但 perf thin space 空间耗尽，用户 IO 写入失败，导致分片剔除甚至返回 IO Error。

为了尽量避免出现此类问题，zbs 通过 flow control 机制来抑制用户 IO 的写入速率，为释放 perf thin space 的下沉 IO 争取时间。

flow control 机制中包含 2 类角色：

* flow manager (flow mgr)，工作在 LSM 侧，根据本地 LSM 提供的 perf thin 使用情况决定是否开启限流，按照一定的策略为各个 flow ctrl 颁发 app token；
* flow controller (flow ctrl)，工作在 Access 侧，按一定周期向各个 flow mgr 索要 app token，对于每个 block 粒度的业务 IO，若有可用 token，直接放行，否则经过一定的超时时间后再下发。

二者典型的交互流程如下：

1. 各个 flow ctrl 往各个 flow mgr 发出 token 请求并等待 flow mgr 回复；
2. 各个 flow mgr 以一定周期处理每个周期内收到的所有 flow ctrl 的请求，并发出一一对应的回复；
3. 各个 flow ctrl 收到回复后，放行相关用户 IO，立即发送下一个请求给 flow mgr；
4. 以上过程循环反复。

当节点的 perf thin space 使用率超过 90% 开启限流，随后在低于 85% 时关闭限流。

开启限流时，每个 flow manager 在每一轮可颁发的 app token 总数量与自身 perf thin space 使用率、容量层磁盘类型、磁盘数量相关。perf thin space 使用率越低、容量层磁盘数量越多、类型越好（NVME SSD > SATA SSD > SATA HDD），可颁发 token 总数量越多。定量计算参考 [Access for Tiering](https://docs.google.com/document/d/12P9oHQXEzRmOJ0RMdbdHoTF1_RkO7Ar_EUYoIAEKx08/edit?tab=t.0#heading=h.wth3yjpd4fxm) 。特别的，当 perf thin space 使用率超过 98%，可颁发 token 总数量为 0，这意味着用户 IO 只能等待 5s 超时后方能下发。

flow mgr 总是等待一个时间周期（默认 100ms）收到每个 flow ctrl 的请求后，再计算各个 flow ctrl 的可用 token 数量，这样有利于公平分配 token。每次交互，各个 flow ctrl 从 flow mgr 得到的 app token 数量（granted_token）与 2 个因素有关：

1. 本轮这个 flow ctrl 想要的 token 数量占所有 flow ctrl 想要的 token 数量的比例；
2. 过去这个 flow ctrl 的 token 消费数量占所有 flow ctrl 的 token 消费数量的比例；

对二者加权求和得到颁发比例，乘上可颁发 token 总数量，就是本轮这个 flow ctrl 能从 flow mgr 获得的 app token 数量。

flow ctrl 以用户 IO 到来的顺序消费从各个 flow mgr 获得的 app token。举个例子，若有 2 个 IO 被拦截，分别为 loc=[1, 2, 3] 的 IO A 和 loc=[1, 2] 的 IO B，其中只有节点 2 3 开启限流，当 flow mgr 收到节点 2 的 app token 时，会让先到的 IO A 消费，尽管 IO A 可能因为缺失节点 3 的 app token 无法立即放行。另外，若用户 IO 长时间无法获取所有想要的 app token，会被超时放行，超时时间与 loc 中 perf thin 空间最紧张节点的使用率负相关，具体来说，若其使用率低于 90%，等于 0.5s，若其高于 98%，等于 5s，其余在 0.5s ~ 5s 之间。

只有满足 perf、write、所在 block 未申请过 token 这 3 种条件的用户 IO 才需要 app token。特别的，若是一个 256 KiB 大小的 perf write（对应 LSM 行为是分配一个新的 pblob），那么认为一定没有申请过 token，本次写入需要 app token。另外，根据 IO 所在卷是否常驻缓存（perf thick or not），需要 app token 的节点列表 loc 组成并不相同，对于 perf thick write，仅临时副本所在节点需要 app token，对于 perf thin write，正常副本、临时副本所在节点、恢复/迁移目的地节点都需要 app token。

### 内部 IO 限流

#### 限流器

在 zbs 集群中，除了用户 IO，系统内部也会产生用于恢复、下沉、抬升、迁移数据块的 IO，二者存在对集群有限的 IO 资源（网络/磁盘等）的竞争，为了避免内部 IO 影响业务 IO，需要有内部 IO 限流器，以指定速率拦截/放行内部 IO。

每个节点的容量层与性能层有各自独立的内部 IO 限速器，以 block 为粒度拦截/放行内部 IO，实现上分为 2 个组件：

* internal flow control：工作在 Access 侧、拦截/放行以自己作为 lease owner 收到的 internal io 的分布式限流器。
* internal io throttle：工作在 LSM 侧、拦截/放行以自己作为 internal src 或 dst 收到的 internal io 的单机限流器；

对于同一类型的内部 IO 而言，以先进先出的方式放行，当同时有多种内部 IO 共存时，默认会以 recover > sink = elevate > migrate 的优先级轮转放行，保证高优先级 IO 尽快完成，同时也避免低优先级 IO 迟迟不被调度。另外，不论当前限速器承载了多少数据量、已放行的 IO 是否完成，影响放行效果的因素只有内部 IO 限速（internal io speed limit）。大部分情况下，内部 IO 只要被 internal flow controller 放行，也会被 internal io throttle 直接放行。

与 flow control 机制类似，internal flow control 机制中也有 internal flow controller（ifc）和 internal flow manager(ifm) 这 2 种角色。当 Access 通过心跳从 Meta 接受各种内部 IO 命令后（sink IO 特殊一些，可由 access 自行发起），以 block 为粒度，若 ifc 同时持有作为 ifm 的 internal src / dst chunk 颁发的 internal token，IO 将被放行，否则被拦截。更具体的，internal token 的颁发与消费规则参考 [ZBS 内部 IO 流控分析与改进](https://docs.google.com/document/d/128Lall-xo_EIPijNqf3IF4RclUhDGORWIgfY18h9cac/edit?tab=t.0#heading=h.qiis55i97kjd) 。

从 zbs 5.6.1+ 采用新的内部 IO 流控机制之后，大多数情况下业务写 IO 不再会因为所访问的 Block 正处于恢复、下沉、迁移被内部 IO 限流阻塞时出现很大的 IO 延迟现象（业务读 IO 一直以来都不会）。

#### 弹性调节

为了避免内部 IO 影响业务 IO 以及充分利用 IO 资源，需要针对不同的业务 IO 负载自适应地调整内部 IO 限速，即智能调节（auto mode）。当业务处于高峰期，限制内部 IO 流量，而在业务相对空闲时，放开内部 IO 限速，让待变化的数据块尽快去到预期位置。

Access 默认每 4s 检查自身是否需要调整内部 IO 限速，限速调整遵循快下降、慢上升的原则。对于每个节点来说，判断 IO 繁忙或空闲的指标是 iops 和 bps（延迟受 IO 大小、网络波动、磁盘故障影响明显，波动较大，并未采用），IO 流量包含本地（from local）流量和远程（from remote）流量两部分。

弹性调节规则如下：

1. 内部 IO 限速的默认值是最大值的 30%；
2. 若业务 IO 的 iops / bps 超过繁忙阈值，每次将内部 IO 限速降低到当前限速的 50%，直到限速最小值；
3. 若业务 IO 的 iops / bps 未超过繁忙阈值且内部 IO bps 超过当前限速的 80%，每次将内部 IO 限速提高到当前限速的 150%，直到限速最大值。

zbs 默认运行在智能模式，由系统弹性调节内部 IO 限速。若需要人工设定，可通过 zbs cli 切换到静态模式（static mode）下进行，该模式下只允许对所有 chunk 设置相同的限速值，如果 chunk 自身硬件能力可提供的内部 IO 限速最大值小于静态设定值，只会使用内部 IO 限速最大值，另外不支持将限速设置为 0 MiB/s。

不论哪种模式，各节点的内部 IO 限速最小值是 1MiB/s，内部 IO 限速最大值与业务 IO 繁忙阈值与节点硬件能力有关。

业务 IO 繁忙阈值取决于所在层级已挂载且健康的磁盘类型和数量，大部分情况下，单一层级（容量层/性能层）中都会是相同类型的磁盘，如果混合部署了不同类型的磁盘，那么以最强类型的繁忙阈值为准（而非累加）。

* NVME SSD：不论数量多少，只提供 500 MiB/s 的 bps、5000 的 iops
* SATA SSD：不论数量多少，只提供 150 MiB/s 的 bps、1500 的 iops
* SATA HDD：每块提供 20 MiB/s 的 bps、80 的 iops
* Unknown： 不论数量多少，只提供 100 MiB/s 的 bps、100 的 iops

内部 IO 限速最大值取决于磁盘限速和网络限速之间的最小值，其中网络限速最大值取决于集群是否开启 RDMA 以及存储网络带宽，磁盘限速最大值取决于所在层级已挂载且健康的磁盘类型和数量（虽然相同类型、型号的磁盘，在容量不同时磁盘能提供的 iops/bps 上限不同，比如 P5620 NVMe SSD 1.6 TB 跟 6.4TB 的 4k iops 上限分别是 20w 和 30w，但目前暂时还不需要如此精细的控制）。

* 网络
    * TCP： 40% * 存储网带宽
    * RDMA：50% * 存储网带宽
* 磁盘
    * NVME SSD：每块提供 600 MiB/s 的带宽
    * SATA SSD：每块提供 250 MiB/s 的带宽
    * SATA HDD：每块提供 80 MiB/s 的带宽
    * Unknown： 每块提供 100 MiB/s 的带宽

以目前最常见的硬件组合（2 * NVME SSD + 4 * SATA HDD + 10 Gbps 网卡，未开启 RDMA）为例：

* 性能层：业务 IO 繁忙阈值是 500 MiB/s 的 bps、5000 的 iops。内部 IO 限速最大值是 476.84 MiB/s（磁盘上限 = 2 * 600 = 1200 MiB/s，网络上限 = 40% * 10000 Mbps = 476.84 MiB/s，二者取最小值）；
* 容量层：业务 IO 繁忙阈值是 80 MiB/s 的 bps、320 的 iops。内部 IO 限速最大值是 320 MiB/s（磁盘上限 = 4 * 80 = 320 MiB/s，网络上限 = 40% * 10000 Mbps = 476.84 MiB/s，二者取最小值）。

### Cap IO 限流

zbs 中为了避免 IO 长时间不返回，每个发往本地 LSM 的 IO 若在默认的 8s 内未完成，相关分片将被剔除。另外，zbs 中多个 Access 间、单个 Access 内都没有对所有类型的 IO 下发做统一并发度调控，这可能会导致不同 LSM 间的 IO 并发度、单个 LSM 内不同类型 IO 并发度差异都很大。

以上 2 点事实可能会导致 LSM 先收到并发度很高的类型为 A 的 IO，让后到的、类型为 B 的 IO 有很长的排队延迟，容易引发 B IO 超时、分片剔除，进而影响数据安全。在开启分层之前，缓存击穿后 IO 将直接落在 SATA HDD 上，在分层之后，容量层与性能层相互独立，Cap IO 可以直接落在 SATA HDD 上，这个问题的出现概率大大增加了。

为了尽量避免出现此类问题，从 zbs 5.6.0 开始，zbs 通过 Cap IO Throttle 来限制各节点的 Cap IO 并发度。

当节点的容量层介质是 SATA HDD 时，对于所有的 Cap IO，不论是业务 IO 还是内部 IO，不论是读写 IO 还是非读写 IO，都需要经过工作在 pextent io handler / local io handler（位于 LSM 外围）的 Cap IO Throttle，由 Cap IO Throttle 根据各类 Cap IO 当前并发额度决定放行/拦截。当 Cap IO 发往 LSM 时，该类 IO 并发度加一，当 IO 从 LSM 返回时，该类 IO 并发度减一。

实现上，考虑到从旧版本升级到 5.6.x 的集群中的存量数据块会被认为是 Cap Pextent，但它们实际上有可能在性能层（取决于 LSM 内部 writeback 的进度），如果限制了它们的业务 IO 并发度，可能导致业务 IO 延时极高，因此 Cap IO Throttle 不限制业务 Cap IO 的并发度。在此基础上，为了避免业务 CAP IO 并发度很高时，内部 Cap IO 没有机会下发，会将当前正在执行的业务 Cap IO 数量的一半作为内部 Cap IO 的并发度上限（这个处理方式有待后续优化）。

默认配置下，各节点 Cap IO 总并发度上限 = 4 * SATA HDD 数量。每 100ms 根据各类 Cap IO 权重以及现有 IO 数量重新计算一次各类 Cap IO 的并发度上限。当某类 Cap IO 没有正在执行的 IO 时，本轮得到的并发度上限为 0，否则按各类 Cap IO 权重 recover : sink : elevate : migrate = 4 : 2 : 2 : 1 计算本轮得到的并发度上限。为了避免并发度突变时性能抖动明显，每一轮调整每类 IO 的并发度上限时只会在上一轮的基础上加/减一。

为了保证数据块的 gen 始终有序，Cap IO Throttle 中同一 pid 的所有类型的 Cap IO 需要按 FIFO 顺序放行。在此基础上，为了保证 IO 优先级（app > recover > sink / elevate > migrate），允许后到的、高优先级的读写 Cap IO 提升先到的、但被拦截在低优先级等待队列的低优先级 Cap IO 到高优先级等待队列中，避免高优先级 IO 因低优先级 IO 长时间没有可用并发额度而迟迟无法执行。

举个例子，假设 Cap IO Throttle 目前所有等待队列都没有可用并发额度，以下不同类型的 IO 都是同一个数据块的 IO，并有如下事件流：

1. 来了 2 个 sink IO，那么这 2 个 IO 都会在 sink 等待队列中阻塞；
2. 来了 1 个非读写 IO（比如 get gen），那么这 3 个 IO 都会在 sink 等待队列中阻塞；
3. 来了 1 个 recover IO，那么这 4 个 IO 都会在 recover 等待队列中阻塞；
4. 来了 1 个 APP IO，那么这 5 个 IO 都会在 APP 等待队列中阻塞。

默认情况下 Cap IO Throttle 不限制 APP Cap IO 的并发度，可以认为 APP 等待队列是个并发额度无限大的队列，所以最迟在 100ms 后，上述 5 个 IO 都将被放行，且放行顺序会是先 sink 再非读写 IO 接着 recover 最后是 APP IO。

Cap IO Throttle 具体实现参考 [Access for Tiering](https://docs.google.com/document/d/12P9oHQXEzRmOJ0RMdbdHoTF1_RkO7Ar_EUYoIAEKx08/edit?tab=t.0#heading=h.16o3ugr2s82p)

## ZBS IO 路径















