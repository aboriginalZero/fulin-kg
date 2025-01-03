## 分片分配策略

在分配每个数据块 pextent 的分片位置时，ZBS 根据各个 pextent 的 prefer local 节点所处负载的不同，使用不同的分配策略，具体分片分配策略遵循如下原则：

1. 低负载：① 拓扑安全 ② 本地优先 ③ 局部化。
2. 中负载：① 拓扑安全 ② 本地优先 ③ 低载优先。
3. 高负载：① 拓扑安全 ② 低载优先。

序号越小的原则优先级越高，各个原则含义如下：

* 拓扑安全：不论哪种负载，分片分配策略总是优先满足拓扑安全。若拓扑安全暂时无法满足，允许先分配，后续通过迁移来修复；
* 本地优先：若 prefer local 节点处于健康状态且剩余空间充足，一定将分片分配给它；
* 局部化：若集群拓扑结构、节点健康状态不变且剩余空间充足，prefer local 相同的 pextent 分片的分配位置是固定不变的；
* 低载优先：若候选节点的拓扑等级一样，将分片分配到负载更低的节点。

用一个 3 节点集群来说明，perf thin 中高负载分别从 0.5 和 0.6 开始，prefer local = 1，各节点 perf valid = 400 GiB，创建一个 500 GiB 的 2-replica thin volume，并执行 256k 全盘顺序写，perf loc 变化如下：

| step | cid 1 ratio | cid 2 ratio | cid 3 ratio | perf loc       | prefer  cid load | note                                                         |
| ---- | ----------- | ----------- | ----------- | -------------- | ---------------- | ------------------------------------------------------------ |
| 1    | [0, 0.5]    | [0, 0.5]    |             | [1, 2]         | 低               | 遵循局部化分配                                               |
| 2    | [0.5, 0.6]  |             | [0, 0.1]    | [1, 3]         | 中               | 放弃局部化，遵循本地优先 + 拓扑安全 + 低载优先               |
| 3    |             | [0.5, 0.6]  | [0.1, 0.2]  | [2, 3]         | 高               | 放弃本地优先，遵循拓扑安全 + 剩余容量多，此时有 replace = 3，dst = 1 的迁移 |
| 4    |             |             |             | [3, 1], [3, 2] | 高               | cid 3 在负载跟 cid 1 和 2 相同前，总会被分配，perf loc 在 [3, 1]、[3, 2] 交替 |

ZBS 面向客户提供的最小存储对象是 Volume，但属于同一个 Volume 的 pextent 的 prefer local 并不一定相同。当 pextent 的 prefer local 为 0 或是一个不健康的节点，prefer local 的负载没有实际意义，本地优先原则被放弃，此时根据集群负载决定 pextent 的分配策略。

### 拓扑安全分配

拓扑安全分配指的是尽可能将分片分配到拓扑距离最远的节点上。在集群服役期间，除了存储节点可能发生物理故障，接入交换机（TOR，每个机箱一个接入交换机）、机柜 UPS 电源（每个机架一个独立电源）都有可能异常，为了提供更高水平的系统可靠性与可用性，需要将分片尽可能分布在物理拓扑的不同分支上。

在集群配置节点拓扑信息之后，就可以据此计算集群中节点之间的拓扑距离。一个节点要么因为没有配置拓扑信息被默认放在可用域中，要么一定被放在某个机箱中，两两节点之间的拓扑距离（topo distance）初始值为 0，依照如下方式计算：

* 若两个节点不在同一个可用域（zone），那么 topo distance - 256；
* 若两个节点不在同一个机架（rack），那么 topo distance - 16；
* 若两个节点不在同一个机柜（brick），那么 topo distance - 1；

分片间的拓扑距离值越小，认为分布越安全。举个例子，若一个 pextent 已有分片在节点 c1 和 c2，依照拓扑安全分配，第 3 个分片会被分配到跟 c1、c2 拓扑距离最小的节点上，若这样的节点有多个，再按照本地优先 / 局部化 / 低载优先原则来进一步筛选。

### 本地优先分配

本地优先分配指的是，若 pextent 的 prefer local 处于健康状态（healthy status && in_use state && not isolated）并且有剩余空间，那么一定将分片分配到 prefer local 节点。

pextent 的 prefer local 根据不一样的制备类型，会有不一样的初值，具体规则如下：

* 厚制备（thick）

    常驻缓存卷的 perf / cap pextent 和厚制备卷的 cap pextent 在 volume 刚创建时就会分配分片，此时 pextent 的 prefer local 等于创建请求中指定的 volume 的 prefer local，由于一般情况下不会显式指定，这些 pextent 的 prefer local 会是 0。

* 精简制备（thin）

    精简制备卷的 perf / cap pextent 和厚制备卷的 perf pextent 在数据块初次写时才会分配分片，此时 pextent 的 prefer local 优先以 volume 的 preferred_cid 为准，若该值为 0，以发起写申请的 lease owner 节点为准。

当出现以下 3 种情况时，pextent 的 prefer local 会被变更：

1. 主动更新

    虚拟卷更新时指定了不为 0 的 prefer local，一般是通过 zbs cli 来更新。

    大部分情况下，volume 的 prefer local 都是 0，持有的 pextent 的 prefer local 可能各不相同，但若指定了 volume 的 prefer local，那么该 volume 的所有 pextent 也会有相同的 prefer local。

2. 访问感知

    即快照卷模板化，被模板化的快照卷持有的 pextent 的 prefer local 将被重置为 0，有 2 种方式：

    1. 快照卷更新时指定了 alloc even。除了通过 zbs cli 更新，目前一般的行为是在 SMTX OS（ELF）中主动将虚拟机创建/转换为模板；
    2. 快照卷被大量克隆。克隆超过默认次数 （10）的快照卷即使 alloc even 属性还是 false，但会模板化。

    当一个快照卷被模板化，快照卷内的数据不应该集中存储，这是因为一个模板（不论上述哪种方式得来）预期都会有大量克隆，克隆之后的虚拟卷通常会分布到集群的多个节点上，集中存储无法获得本地化 IO 的好处，反而可能会造成局部过载。

    因此模板卷（even volume）的分配策略与一般的虚拟卷不同，只会遵循拓扑安全和完全均匀策略。完全均匀策略指的是，对于每个模板卷，集群中的每个节点都会尽可能持有相同数量的数据块分片。

3. 本地感知

    尽管以 pextent 为粒度的分片分配，不能保证同一个 volume 内的 pextent 分片位置相同，但作为一个块存储系统，通常一个 volume 内所有的 pextent 都会具备相同的本地访问（在 SMTX OS 下是数据卷关联的虚拟机所在物理节点，在 SMTX ZBS 中是 lun / namespace / ... 协议接入的物理节点），因此可以根据 volume 的接入点（access point）来变更其所持有 pextent 的 prefer local。

    
    
    集群中每个 access 每 1h 上报一次这段时间内 IO 次数超过 3600、平均大小超过 511Byte 的 volume（即 iops > 1，包含读写）给 meta，meta 会记录这些接入点，连续上报 6 次的接入点成为活跃接入点。当一个接入点初次变为活跃时会触发 local transfer 事件，此时若 volume 的接入点个数不超过默认数量（3）且 volume 没有显式指定 prefer local，那么会将该卷持有的所有非 COW 状态的 cap / perf pextent 的 prefer local 都变更成该活跃接入点，并撤回相关 lease。
    
    
    
    local transfer 通常发生在虚拟机开机或者热迁移到一个新物理节点上重新建立连接后稳定使用的场景。一般来说，一个 volume 只会有一个活跃接入点，zbs 采用稳定链路接入策略保证在计算端稳定时，总会从同一个 access 上访问数据以保证本地感知可以得到一个稳定的结果（尽管临时故障时可能会有接入点切出本地，但基本都能在 6h 内切回本地）。
    
    Oracle RAC 等小范围共享卷的场景可能会有 1 2 个活跃接入点，但处理方式不变。特别地，有些场景如 Xen 平台的 iSCSI DataStore 模式将一个 volume 作为一个池使用，不同节点上的虚拟机仅访问其中的部分数据，卷的活跃接入点可能会大于等于 3 个，此时持有的 pextent 只会在初次写入时用第一个接入节点作为 prefer local，后续不再变更。
    

### 局部化分配

局部化分配指的是将 prefer local 相同的 pextent 的分片分配到相同的、固定的节点上，以此来降低节点故障时的影响范围。考虑分片完全均匀分配在每个节点上的场景，此时每个 Volume 中的分片会均匀分散到整个集群，任意一个节点都持有集群中任一 Volume 的部分数据，这样当一个节点异常（节点下线、磁盘异常等）时，集群中所有的 Volume 都会受到影响，且节点故障的影响范围也会随着集群规模的扩张而增大。

局部化分配策略也有明显的劣于大部分分布式系统采用的完全均衡策略。就是在集群节点规模较大的时候，而单一节点上的磁盘能力较低的时候，单一虚拟卷能动员的硬件能力会被局限在有限的节点上而无法发挥整个集群的能力。但是考虑到：

* 即便数据均匀分散到所有节点上，性能上限也会受到网络的限制，技术的发展趋势是磁盘的能力远超过网络的发展能力，现代磁盘就不存在这样的瓶颈（ 2 x NVMe 能提供的 IOPS 和带宽超过 25 G 网卡的上限）；

* 即便是较差的磁盘，存储集群通常也不会只为单一虚拟卷服务，均衡分布带来的 IO 互相干扰的现象对集群整体能力上限更为不利。  


因此 ZBS 选择局部化策略作为空间分配的基础策略。



在实现上，存在多个候选节点跟已有节点的拓扑距离相等时，才会进一步根据局部化分配策略选择下一个分片的分配节点，具体来说，就是按照 ring id 的相对顺序选择跟上一个分片最接近的节点。

ring id 是 zbs 为每个节点添加的一个软件拓扑属性。当一个 ZBS 集群发生拓扑结构变化如 1. node 加入/退出、2. node 跨 zone/rack/brick 移动、3. brick 跨 zone/rack 移动、4. rack 跨 zone 移动时，集群内各节点的 ring id 都会重新生成。ring id 的自动生成规则如下：

1. 对各个 brick 级别的节点列表（每个 brick 中的所有节点构成一个列表）按 topo id 从小到大排序；
2. 遍历集群中每个 rack ：对同一个 rack 内部的 n 个 brick 级别的节点列表先执行粗合并，再执行细合并，得到一个 rack 级别的有序列表；
3. 遍历集群中每个 zone：对同一个 zone 内部的 m 个 rack 级别的节点列表先执行粗合并，再执行细合并，得到一个 zone 级别的有序列表；
4. 对 2 个 zone 级别的有序列表执行细合并（只有优先可用域/次级可用域 2 个节点列表，所以不用执行粗合并），得到一个 cluster 级别的节点列表，此时，每个节点在该列表中的位置就是它的 ring_id。

其中，粗合并指的是将给定的 n 个节点列表合并成 2 个长度尽可能接近的节点列表 lhs 和 rhs，目的是让副本分配也会落在长列表中的尾节点上，充分利用每个节点容量，保证整体副本分配尽可能均匀。细合并指的是将给定的 2 个节点列表划分成 k = min(len(lhs), len(rhs)) 并顺序按组配对，目的是让副本分配尽可能局部化，使得拓扑变动引起的副本迁移能够控制在与 k 正相关的节点规模内。

当集群拓扑结构、节点健康状态不变且剩余空间充足时，prefer local 相同的 pextent 分片得到的期望局部化分片列表是相同且固定的，这也意味着 prefer local 不同的 pextent 的分片位置的重叠概率是很低的，不同 Volume 之间占用的物理资源相对独立，这除了可以缩小故障影响范围，还便于按需调整各个 Volume 的 Qos 策略，让集群性能随节点数线性增长。

## 分片迁移策略

在 ZBS 集群运行过程中，不论集群此时处于何种负载，当集群中没有待恢复数据块时，meta leader 的一个后台线程会周期性地扫描所有副本/节点，按某种迁移策略选出待迁移数据块的 src / dst / replace cid 生成迁移命令，并为其选定/分配 lease 后，通过心跳下发给 access 执行。

迁移扫描在集群中有节点移除时，默认每 4s 触发一次。对于其他类型的迁移，当集群进入高负载前，默认每 15 min 触发一次，高负载后，每 2 min 触发一次。

迁移的触发条件包括以下 6 种：

1. 局部化修复。当集群进入中负载前，期望通过局部化修复让非均匀卷的数据块分片处在固定的节点列表，局部化修复包含本地化修复和拓扑修复。
2. 本地化修复。当集群进入高负载前，期望通过本地化修复让非均匀卷的数据块在 prefer local 上有分片，提供数据访问性能。
3. 拓扑修复。在集群进入超高负载前，期望数据块分片分布在物理拓扑的不同分支上以提高数据安全。
4. 节点容量均衡。在集群进入中负载后，期望各节点容量尽量均衡，充分发挥集群多节点存储上限。
5. 均匀块的数量均衡。初次模板化得来的均匀卷期望通过迁移将持有的数据块分散到各个节点。
6. 节点移除。待移除节点需要将自身数据迁移到集群中的其他节点上，在数据全部前移除，将会停止以上 5 种类型的迁移。

为了便于 access 执行迁移命令时保证数据正确性，每个 volume 的 lextent 在一次迁移扫描过程中最多只会生成一条 pextent 的迁移命令（要么 cap，要么 perf），在分片迁移策略的设计中，也会避免反复迁移，争取一步迁移到位。

### 非均匀卷迁移

集群中大部分数据卷（即非均匀卷）的数据块，依照集群负载不同，执行目标不同的迁移策略：

1. 低负载：① 局部化修复 ② 拓扑修复 ③ 本地化修复
2. 中负载：① 拓扑修复 ② 本地化修复 ③ 节点容量均衡
3. 高负载：① 拓扑修复 ② 节点容量均衡
4. 超高负载：① 节点容量均衡

补充说明：

1. 低负载时，若数据块 prefer local 不健康或没有剩余空间，尽管无法执行局部化修复，但仍会为其尝试修复拓扑以保证拓扑安全；
2. 中负载时，若集群没有配置拓扑信息，拓扑修复将被直接跳过，执行本地化修复扫描，此时若生成了迁移命令，之后的节点容量均衡将会跳过；
3. 高负载时，因为放弃了本地优先原则，所以若集群没有配置拓扑信息将直接执行节点容量均衡迁移扫描。



分片分配策略中，在集群进入中负载才放弃局部化，进入高负载才放弃本地优先，迁移策略也需要满足这个大的原则。不同的是，迁移策略中需要有一个弹性边界，用以避免节点容量在负载转换边界时因小流量 IO 流入出现迁移策略不断转化进而导致分片反复迁移。弹性边界是针对节点而非集群而言，以 cap layer 的默认参数 medium load = 75%、 high load = 85%、very high load = 95%、relaxation = 5% 为例，来说明弹性边界的策略：

1. 当集群处于低负载 [0, 75)，集群中处于严格低负载 [0, 70) 的节点可以保证相关数据块的分片分布满足局部化，但处于低中边界 [70, 75) 的节点无法保证；
2. 当集群处于中负载 [75, 85)，集群中处于严格中负载 [75, 80) 的节点可以保证相关数据块存在本地分片，但处于中高边界 [80, 85) 的节点无法保证；
3. 当集群处于高负载 [85, 95)，集群中处于严格高负载 [90, 95) 的节点可以保证相关数据块的分片分布满足拓扑安全（包含双活域间分片 2 ：1 的特点），但处于高超高边界 [85, 95) 的节点无法保证；

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

尽管 zbs 允许多个节点同时进入 REMOVING state，但集群中一次只会迁出一个节点上的数据块，直到节点不再持有任何数据块后，进入 IDLE state。

节点移除迁移命令的 dst / replace / src cid 选取规则：

* dst cid 必须满足健康、有剩余空间，优选非 isolated、拓扑等级较高的节点；
* replace cid 一定是待移除节点；
* src cid 在 replica 时优选非 isolated 节点，在 ec 时一定是 replace cid；

节点移除迁移的选取规则基本上复用了分片恢复策略中的选取规则。不同于其他迁移，节点移除迁移是有可能在其他节点容量不足时迁入拓扑等级更低但仍有剩余空间的节点上。

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

    1. 每个节点作为 src/dst 每轮最多只能生成默认的 200 条 cap recover cmd 和 400 条 perf recover cmd，单轮所有节点最多只能生成默认的 1024 条 recover cmd（不区分 cap / perf）；
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
2. 若冗余类型为 replica 的 PExtent 在维护模式节点上有数据更新且由于未能即使响应导致副本剔除，Meta 会将这个失联分片打上 RIM（Replica In Maintenance）标记，并根据 PExtent 的期望副本数与存活副本数有着不同的恢复策略：
    1. 2 副本数据块只有 1 个在维护模式节点上的副本失联：在节点进入维护模式超过 60s 或退出维护模式后才会触发恢复；
    2. 3 副本数据块只有 1 个在维护模式节点上的副本失联：在节点退出维护模式后才会触发恢复。

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

## 分片回收策略

描述 gc 机制







常用数值的计算





https://docs.google.com/document/d/1CTxMn6pyajujO7u7sbsggz3gbDV7YtDWKgH8NRgB5ZA/edit?tab=t.0

https://docs.google.com/document/d/17H6WVHB3fWr_JnyNutRkycpXSIZbWKujS6oybFZhR64/edit?tab=t.0



