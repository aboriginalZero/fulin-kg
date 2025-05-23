### Optimizing Resource Allocation in Hyperscale Datacenters: Scalability, Usability, and Experiences

问题的本质是什么？



今天给大家介绍这篇论文，meta 内部使用的资源分配器，他们叫做 rebalancer。



首先介绍一下一般系统领域中会遇到的资源分配问题。比如硬件放置问题，在大型数据中心中，经常需要对机器上电/下电，如何在满足网络/电力资源限制等前提下，规划哪些服务器机架在哪周上电和下电是 meta 他们遇到的实际问题。另外一个问题是服务分片问题，比如数据库服务的分片应该如何放置，怎么在满足故障域冗余、就近服务等前提下，迁移哪些服务分片到哪些服务器上让服务负载均衡。

这些问题抽象出来都是一类问题，怎么在满足给定约束条件下，把哪些物品放到哪些箱子里能够取得最大收益？



接下来看一道可能是中学或者大学时期的数学题，是一个类似问题。工厂有 2 种产品可以生产，每种产品所需的生产成本与可得利润如下，工厂现有 10 个工时和 20 个材料，问分别生产多少个产品 A 和 B 能够让总利润最大？ 

这个问题的约束条件是 2A + 4B <= 10，3A + 2B <= 20，A 和 B 是非负整数，目标函数是 Z = 5A + 7B，想要让利润最大化。这里的重点是 A 和 B 必须是整数，从计算机的视角看，这是一个 NP-hard 问题。

一般的求解方式是分支定界法、割平面法之类的，这个有专门的 MIP 求解器可以用，但在数据规模大了之后，求解所需时间会特别长，基本没法接受。



meta 为了解决他们自己遇到的硬件放置和服务分片等问题，开发了 Rebalancer，他的特点是高易用性和高可扩展性，并且能够在有限时间内得到精度足够高的解。



Rebalancer 包含 3 部分：第一部分是对外提供 spec 声明式 API 方便使用者描述他想要求解的资源分配问题。第二部分是将 spec 转换为表达式的递归组合，并得到表达式的有向无环图。最后是如何求解，小规模问题用 MIP solver，业界已经有现成的方案可以使用，数据规模大的时候用 local search 求解，这也是这篇论文的主要创新点。



Rebalancer 对外提供了多种 spec API，使用者可以组合使用这些 spec 来描述他想要求解的资源分配问题。比如这些的 CapacitySpec 可以用来限制资源的使用上限，GroupCountSpec 可以用来限制在一个 bin 中出现的同一类型的 object 的数量。AvoidMovingSpec 可以用来限制某些 object 被移动。这些 Spec 的实现代码都不多，基本都在 300 行以下，如果需要自定义一个新的 Spec，也比较方便添加。



接下来用一个简单的例子看一下如何组合使用这些 Spec 来描述资源分配问题。

比如这是一个有多个任务要分配到不同服务器上执行的问题，可以用 CapacitySpec 限制每个服务器的 CPU/存储使用上限，为了提高系统可靠性，可以用 GroupCountSpec 限制每个服务器机架只会分配一个 job 的一个 task，最后用 BalanceSpec 来限制服务器之间的 CPU/存储的负载均衡。

这里有一些基本概念，后续的介绍里也会提到。dimension 是 object 和 bin 之间的一种映射关系，表示使用者在意的点，bin scope 可以看成是一类 bin 的集合，类似的概念，object partition 是一类 obejct 的集合。



继续看一下怎么使用这些 Spec 解决前面提到的 Hardware Placement

机架作为 object，星期和物理位置作为 bin。

对于待使用的机架会被认为在一个 unsigned bin，对这个 bin 添加 ToFreeSpec，让其中的 object 必须被移出，也就是一定要把这些机架放到 MSB 里。

对 rack group 添加 ColocateGroupSpec，让属于同一个 group 的 rack 在同一周被处理，这应该是出于方便运维人员操作的角度加的限制。

对 bin scope 比如 suite 添加维度是电量或网络流量的 CapacitySpec，来限制整个 suite 的电力和网络流量使用闪现

还使用了其他 Spec，这里不再一一列举了。



服务分片场景里，分片作为 object，服务器作为 bin，也可以灵活运用多种 Spec，这里不再一一说明了。

总而言之，meta 提供了一组简单易用的声明式 API，能够方便使用者用这些 API 去描述他们关心的资源分配问题。



当使用者定义了 Spec 之后，在模型表示环节，会将其转换成若干个表达式的递归组合，最终得到一个表达式图，便于后续在 model sloving 用 local search 求解。他们还提供了一个配套工具，可以显示这个表达式图，让使用者大概感受问题的规模、Spec 之间是否有冲突之类。



以前面提到的工厂生成产品为例，可以看一下这个表达式图是怎么构建的。这个问题里，object 是工时和材料，产品 A B 是 bin，约束条件 2A + 4B <= 10，目标函数 Z = 5A + 7B，A B 是非负数，那表达式图里就包含 2 个子树，蓝框部分表示约束条件构成的子树，根节点默认有值 <= 0 的限制，这里的根节点的值就是 2A + 4B - 10，期望他小于等于 0，红框表示目标函数构成的子树，根节点的值希望越大越好。



再以服务器资源使用为例，可以看下对应的表达式就是右图这样的，红线构成的子树是目标函数，也就是 2 个服务器之间要的容量负载均衡，黑线构成的是约束条件，也就是服务器 1 容量使用不能超过 L1，服务器 2 容量使用不能超过 L2。

这里的 Lookup 是比较特殊的叶节点，后面求最热箱子的时候也会用到，可以理解成 object 放入指定 bin 后的值，比如这里的 Lookup1 就是 server1 的存储用量。



有了表达式图后，接下来就是怎么求解问题。

主要的算法过程是这样的：

1. 在未被冻结的箱子里中找到最热的一个箱子，最热箱子的定义在下一页 PPT 会介绍；
2. 按一定的移动策略，比如逐个尝试添加和移除、或者先随机几次再贪心的尝试将某些 object 分配到最热箱子里；
3. 如果这样的一次 object 移动操作满足约束条件并且能让目标函数变大，代表他是有增益的，记录下来，找有最有增益的，应用到表达式图里，然后冻结这一轮选出的最热箱子，冻结起来。
4. 上面这 3 步循环执行，直到所有的箱子都被冻结，最终得到的 assigment 指明了哪些 object 应该放到哪些 bin 上

上面这个过程跟逐项配对尝试相对，牺牲了一定的精度，有些 obeject 和 bin 的组合可能没有尝试到，可能会错过最优解，但显著提高了速度。



刚刚的这个算法过程中，比较重要的一个环节是在每一轮找最热箱子。

最热箱子的定义是移入 object 到这个 bin 或者从这个 bin 移出 object 会最大程度提高目标值的 bin。在表达式图里，对根节点贡献最大的 Lookup 叶节点，它里面包含的 bin 就是最热箱子。



对于每个节点，按 potential 排序他的所有子节点，之后从该节点开始前序遍历，第一个叶节点就是 hottest bin，每个节点的 potential = current value - lower bound，不过论文中没有再展开介绍怎么计算的。不确定他具体是怎么定义 lower bound 的。



每一轮里，当选出了 best local change 时，也就是对目标值贡献最大的 object 放置方式后，只会去更改 bin 所在的 Lookup 叶节点到根节点路径上的所有节点。

这里应该还是依赖 DAG 的结构减少了很多计算量，但论文里只是提了一嘴，没有详细展开。



作者提出的这个 local search 算法主要是跟 MIP 类算法和启发式算法对比，local search 算法的主要特点就是快和容易演化，方便添加新规则。



最后是一个总结，资源分配框架里最主要的挑战就是易用性和可扩展性，这篇论文的创新点也就在这，spec 声明式 API 提供了易用性，表达式图和 local search 算法提供了可扩展性。





问题引入

资源分配问题

* Hardware placement

    决定何时何地在数据中心中添加或删除服务器机架，同时平衡相互竞争的目标，如员工工作计划、电力预算、跨故障域的分布和邻近的托管，例如需要高带宽网络的ML训练服务器。

* Service sharding

    对于数据库等分片服务，确定如何在数据中心区域内和跨数据中心区域迁移数据分片，以响应实时负载变化，同时确保跨故障域传播，防止过多并发更改，从而破坏系统稳定。

所有这些资源分配问题都有一个共同的模式，即我们希望将一组 object 分配到一组 bin 中，以优化特定目标，同时满足某些约束。



最常见的解法是混合整数规划（Mixed-Integer Programming ，MIP） ，解法有分支界定法和割平面法

时间复杂度是 O (|O| * |B|) ，当问题规模变大，时间复杂度就变大了

meta 遇到的问题规模是 100w 的 objects 和 10w 的 bins。



#### model specification

基本概念是物品和箱子，把特定物品放进特定箱子里，使得在满足给定限制下，整体收益最大 

对外提供了几个概念，比如 object 和 bin 之间的 dimension、bin scope

对外提供了一些 xxxSpec，比如 CapacitySpec 

支持自定义一个新的 xxxSpec，已有的单个 Spec 最复杂也是在几百行代码里完成

* dimension

    A dimension is a mapping of each object and bin to a number

    比如 memory dimension，一台服务器的 memory dimension 规定了它可以使用的内存上限，一个 task 的 memory dimension 规定了他运行所需的内存大小

* bin

    Rebalancer uses *scopes* to represent the hierarchical structure of bins

    scope 将所有 bins 划分成若干个 scope items。比如在 rack scope 下，scope item 比如 rack1和 rack2 表示这些特定机架中的服务器集合。

* object

    类似于 bin scope 的概念，对于 object 有个 object partition 的概念，但 object partition 之间可能不是正交的。

    object partition 中的每个集合计为 group

    For example, in the context of cluster management, all tasks are partitioned into jobs and a job is a group of tasks that run the same executable.

    一个 job 中有多个 tasks，一个 job 是一个 group

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

    NEW_util = AFTER_util - STAYED_util，刚被移入 bin_j 的 object 的利用率

    OLD_util = INITIAL_util - STAYED_util，刚被移出 bin_j 的 object 的利用率

    ANY_util = INITIAL_util + AFTER_util - STAYED_util，不论分配前后，只要在 bin_j 中的 object 的利用率

    NEW_util 和 OLD_util 可以度量系统稳定性，另外可以得出 util(b_j, D, STAYED)  = util(b_j, D_init, AFTER) 

定义作为 object 的 tasks 和作为 bin 的 servers 之间的关系

```
// 限制每个服务器的储存用量
addConstraint(CapacitySpec(scope="server", dimension="storage"))

// 限制一个机架上每个 job 最多运行一个 task
addConstraint(GroupCountSpec(scope="rack", dimension="ObjectCount", partition="job", limit=1))

// 优化目标是平衡服务器间的存储用量
addObjective(BalanceSpec(scope="server", dimension="storage"))
```

Rebalancer 会将这些 util 变量翻译成数学形式，INITIAL_util 可以被提前计算，AFTER_util 表示按维度 D 分配后的结果。比如用 UtilIncreaseCostSpec 来优先移动 CPU 利用率小于指定阈值 T 的 server，那么对每个 object 加一个惩罚表达式 Power(excessUtil_i, 2)，其中 excessUtil_i = Max(0, util(server_i, CPU, AFTER) - T)。



支持一次资源分配里有多个优化目标吗？



Local Search 的结果稳定吗？他们的 qe 怎么做测试？



有了这些 Spec 后，怎么解决 Hardware placement

datacenter region -> datacenter -> suite -> main switch board(MSB) -> server row -> server rack -> server

全球有几十个 datacenter region，每个 region 在几英里的范围内有多个 datacenter，每个 datacenter 有 4 个大房间 suite，每个 suite 中有 3 个 MSB，每个 MSB 为 1 ~2w 个 server 供电。

时不时会有整个机架的上电和下电。资源分配时需要考虑到员工的上班时间、故障域的冗余、网络、电力的资源限制。

rack 作为 object，(week, position) 作为 bin（前者是哪一周上电，后者是机架所在的物理位置，比如在哪个 MSB 之类的）。

在同一个 MSB 和同一周的 bin 集合是一个 scope item。同一类型的 rack 组成一个个 rack partition。

* 待添加的机架一开始认为在 unassgied bin，并对这个 bin 添加 ToFreeSpec，也就是必须都分配出去。
* 对 bin scope 如 MSB/suite 添加 CapacitySpec，来限制电力和网络流量。
* AI server rack 必须放在 AI zone 以使用高带宽的网络。为此引入一个 AiRack dimension，将使用 AiRack dimension 的 CapacitySpec 作用在 position scope 上。
* 在同一周要处理的 rack 通过把 rack 放在一个 group 并且应用 ColocateGroupSpec 到这个 group 在 week scope。

总的来说，由于机架的变化是渐进的，而且我们是分别为每个数据中心区域计算解决方案的，因此硬件布局问题的规模相对较小，涉及数百个 object 和数千个 bin。对于这些小问题，Rebalancer将表达式图转换为MIP问题公式，并采用MIP求解器，而不是局部搜索求解器，以确保高质量的结果。





有了这些 Spec 后，怎么解决 Service Placement

service placement 考虑的是每个 region 内的资源预留。

server 作为 obejct，reservations 作为 bin。

70w 个 servers 还能理解，不过 reservations 作为 bin，为啥会有 6k 个的量级？

为啥是 reservations 作为 bin，这个 reservation 像是资源池化后的使用？论文里说是作为一个动态的虚拟集群，用于承载业务团队的 job



有了这些 Spec 后，怎么解决 Service Sharding

什么时候使用分片？

* 应用程序数据量增长到超过单个数据库节点的存储容量
* 对数据库的读写量，超过单个节点或其只读副本可以处理的量，从而导致响应时间增加或超时
* 应用程序所需的网络带宽，超过单个数据库节点和任何只读副本可用的带宽，从而导致响应时间增加或超时。



一个  linux 进程上有 100 个 shard，每个 shard 有多个副本，

* Capacity limit

  在分片移动过程中，避免内部 IO 流量多大

  用 CapacitySpec 来保证 server 不过载，鉴于跨服务器分片移动不是即时的，使用 ANY_util

* Limit churns

  减少不必要的内部 IO 流量

  用 CapacitySpec，ObjectCount 作为 dimension，NEW_util 和  OLD_util 作为 util 来限制每个服务器或每个分片的移动次数

* Region preference 

  比如各个地区的使用者就近使用他所在的 region 上的 shard

  用 AssignmentAffinitiesSpec 来让某些 shard 分配到指定的 datacenter region

* Load banlancing

  用 BalanceSpec，分别以 bin / region 作为 bin scope 来做到 regional / global 级别的负载均衡

* Fault Tolerance

  属于同一个 shard 的多个副本要放在不同故障域上

  用 GroupCountSpec，以 replica 作为 partition，rack / MSB 作为 scope

最大的 sharding 问题涉及到 1.8M 个 object 和 27K 个 bin，并要求在 5 min 内给出分配结果。此时 MIP 无法应对，转为使用 local search



举例来说，假设您有一个数据库，其中有两个单独的分片，一个用于姓氏以字母A到M开头的客户，另一个用于名字以字母N到Z开头的客户。但是，您的应用程序为姓氏以字母G开头的人提供了过多的服务。因此，A-M分片逐渐累积的数据比N-Z分片要多，这会导致应用程序速度变慢，并对很大一部分用户造成影响。A-M分片已成为所谓的数据热点，在这种情况下，数据库分片的任何好处都被慢速和崩溃抵消了。数据库可能需要修复和重新分片，才能实现更均匀的数据分布。



* 分片用于处理大型数据库的数据划分和扩展，以提高性能和可扩展性，通常在不同的服务器上存储不同的数据

* 副本用于提高数据库的可用性、容错性和读取性能，通常在多个数据库服务器之间创建数据的副本。主数据库用于写入，从数据库用于读取。

  Replica 会带来写性能和吞吐的下降，一个分片可能会有 0 个或多个副本，这些副本中的数据保证强一致或最终一致。



k8s 的服务编排跟这个调度的区别是啥？基本上能实现 k8s 的调度，除了 k8s 的初始容量放置

特定节点pod的亲和度使用CapacitySpec指定，并使用自定义维度表示亲和度。

使用 GroupCountSpec 对副本组的 pod 间反亲和性进行建模。

使用 AvoidMovingSpec 将特定的 pods 固定到特定节点。



还是没有搞得很清楚，有了这些 spec 后，怎么定义我们想要的效果。这些 spec 貌似也不太易用

把分配问题转成 spec 的使用



#### model representation

Rebalancer的表达式API支持聚合操作符Max和Sum，以及转换操作符Step、Ceil、Log和Power。例如，如果x为正数，Step(x)的计算结果为1，否则为0。

一旦使用 spec 指定了优化问题，Rebalancer 会将规范转换为表达式的递归组合，重用常见表达式以获得紧凑的表达式图。比如用来保证多样性的 GroupDiversitySpec，转换成表达式是这样，含义是如果 b_j 包含来自 G_i 的 object 就加 1，要求值大于等于 k。
$$
GroupDiversitySpec = \sum_i{Step(util(b_j, O_i, count)) >= k}
$$
$$
CapacitySpec = {Max(util(b_j, O_i, count)) >= k}
$$

搞了几个 efficient custom expression 来把时间复杂度从  Θ（|O| * |B|）降低到  Θ（|O| + |B|）。

论文只介绍了其中一个 expression 叫 Lookup，用以高效实现 utilization。

可以用 Lookup 加速的原因是：在大多数情况下，object 的维度值不受它被分配到的 bin 的影响。例如，一个任务消耗的内存大小与它运行在哪个服务器无关，都是一样的值。这允许不同 bin 和 scope item 来共享和重用这些维度值。

obeject 分配到 bin1 和 bin2 在 dimension D 上的值都是一样的，那么就可以不用算 bin sum 次，每个 object 算一次就好。

具体来说，对于每个静态维度 D，建立一个对象向量 object vector，记为 VD，记录了从 object 到 dimension value 的映射关系。给定一个 object vector VD 和一个 scope item Si，一个 Lookup 表示对 Si 中的 bin 的 object-bin pair 进行的高效聚合操作。例如，util(Si, D) = Lookup(Si, VD) 只是通过查找汇总了 Si 中所有 bin 在给定维度 D 上的利用率。

请注意，查找本身的内存使用是固定大小的，因为它只保存了对 Si 和 VD 的引用，而 Si 和 VD 的表示大小分别为 O(|B|) 和 O(|O|)，在所有表达式中共享和重用。这导致整体问题规模是 Θ(|O|+|B|)。

还是没看懂为啥 Lookup 可以减少时间复杂度？只有涉及到成批 object 计算时才有用？



rebalancer 中的限制都可以用 fun(x) <= 0 来表示，那就可以转换成 Max 表达式

每个 bin 都可以有一个 CapacitySpec，比如 bin 1 限制 CPU 使用率不超过 10%，bin 2 限制不超过 20%，可以用一个统一的形式写成：
$$
Max_i(Lookup(b_i, V_D) - L_i) <= 0
$$
每个优化目标和限制都可以写成这种表达式的递归形式。那么可以把优化目标和限制在一个 DAG 上表示。

图上的节点是表达式，每个优化目标和限制是一个子图，

Lookup 一定是叶子节点吗？目前来看，论文里的描述基本是的。但叶子节点不一定都是 Lookup 节点，只不过论文里只以 Lookup 为例。



一个分配问题就是要在多个 x 满足多个限制的前提下，找到使优化目标最大的各个 x 取值



#### model solving

用 optimized local search 来处理 model solving



因为 bin 比 object 的整体数量要少，所以先固定每个 bin 去找合适的 object，那么先找 hot bin。



生成 candidate move 包含 2 部分

* hot bin selection

  如果在 bin1 和 bin2 上有从一条从 node v 到 Lookup 的有向路径，那么将 object 移入/移出这些 bin 将提高 node v 的值。

  所以可以将 leaf node 对 object 的共享按 greedy order 排序来选出 hot bin

  当给定一个目标和当前任务时，移动物体到或离开这个箱子会最大程度地降低目标值，则认为这个箱子是最热的。从高层次上说，在每次迭代中，我们都会找到最热（也就是最坏）的箱子，并尝试通过根据移动类型进行局部更改来修复它，然后继续搜索，直到没有进展为止

  表达式图已经指明了优化目标被哪些 bins 和多少数量影响

  用贪心的方式找出对优化目标贡献最大的叶子节点，比如 Lookup，有了 leaf node 的排序后，就可以得到 bin set 的排序，这样就能找到 hot bin。

  对优化目标贡献最大的叶子节点，这里的贡献定义是什么？论文里没有细说，猜测是按叶子节点的入度来算？或者是跟根节点距离最近的叶子节点？

  图表示是该算法比过去的局部搜索算法更有效的根本原因

  

  只找最热的 bin

  哪些 object 移入或者移出这个 bin 能让目标函数最大化

  计算目标函数的收益只算动到的那些 Lookup 节点到 root 节点过程中涉及到的那个节点

  1M 节点可以只计算 tens of them

  

  lookup 是 utilization of a bin-group 的简洁表示

  utilization 可以认为是 contribution by objects of a bin

  

  current assignment: b1: o3 o4，b2: o2, o5

  

  如果一个节点 v 的 potential 是 0，说明以他为根节点的子图已经是 optimal 了。

  按 potential 排序每个节点的 child 节点，之后前序遍历（根左右）

  每个节点的 potential 等于 current value 和他的 lower bound 的差值。节点的 lower bound 怎么计算？

  如果一个 Lookup node 的所有 bin 都已经被 explored 过，那么认为他的 lower bound = 他的当前值。

  一旦所有叶子节点的 lower bound 被更新过，可以递归地计算所有表达式的 bound。

  如果 obejective root node 的值等于他的 lower bound，认为无法继续优化，程序结束。

  节点的当前值就是节点所在表达式的当前计算结果。

  

  潜在值最高的那个 node 中的 bin

  

  先前的工作里认为，一个 bin 的 potential 是当前的 obejective value 和移出 bin 上所有 object 后的 obejective value 之差，但这只适用于不会被移入 object 的 bin

  

  Lookup 跟 bin 的关系是啥？

  搞这个 DAG 图对性能有帮忙吗？这个图有啥好处。

  

  往 bin 中增删 object 会影响 bin 所在 Lookup 的值，进而影响到对优化目标的贡献

  从 hottest bin 增减 object 会让优化目标的

  

* move strategies

  在选定 hot bin 之后，可以用 SINGLE, RAMDOM, GREEDY, SWAP 移动策略将 object 移入/移出到 hot bin

  SINGLE 策略是穷举每个 object 到 hot bin  然后再接受最好的 move

  一般推荐是先 RANDOM 几次，当改进机会变的稀缺时，再用 GREEDY

move evaluation 的 3 种加速手段

* Bottom-up change propagation

  维护一个从 object 到 leaf node 的 map，一个从 bin 到他涉及到的 leaf node 的 map，对于给定的 candidate move，通过这两个 map 找到受影响的叶子和根节点，这条路径上的所有节点都需要被重新计算

* Minimal computation during a node update

  假设要计算类型为 max 的 node，要区分 changed / unchanged child nodes，对 child nodes 维护一个有序列表，这样可以快速计算 unchanged child nodes 中的最大值，并且有序列表的更新只在应用这个 move 的时候，并且也只是 O(unchanged child nodes len * log(all child nodes))

* Parallelizing Move Evaluations 

  在 apply 之前，找 candidate move 是可以并行找的，因为 move evaluation 并不改变当前状态



潜在值 = 当前值 - 下界值。

对于每个节点，用 child 的潜在值作为排序依据，得到一个有序列表。

如果一个节点的潜在值是 0，说明以该节点为根节点的子图是有优化空间的。

叶子节点，比如 ObjectLookup 被一组 bin 参数化，改变 bin 的内容就会影响这个叶子节点的值。

识别出等价对象，比如属于的同一个 job 的每个 task 对住资源的占用是一样的，只要算其中一个 task 在移动前后的对比。事实上，这样的功能可以非常强大地进一步减少搜索空间，因为它允许我们放弃在探索时等效的移动（如果两个移动都将等效的对象从源箱移动到目标箱，则两个移动是等效的）。

对于给定的限制和 object 表达式（也就是只关心本次问题中的 spec，不需要 cover 每一种 spec），如果 A 和 B 算出相同的值，那么他们是等价对象。

在表达式图中的每个节点，哪些 object 集合等价于这个节点，这些 object 集合之间是等价的。



根据 algorithm 3，怎么体现出时间复杂度是 O(O + B) ？

搞了几个 efficient custom expression 来降时间复杂度，论文只介绍了一个 expression 叫 Lookup，用以高效实现 utilization。





最耗时的 3 个操作

* 找 hot bin
* 对于给定的 bin 和 move type，找到能让优化目标效果最好的 best local change
* 在表达式图上应用 best local change，并且更新赋值情况





每个目标都指定了一个权重和优先级。Rebalancer使用加权和将所有目标具有相同的优先级。为竞争目标（例如objA， objB）找到合适的权重是通过首先对它们进行归一化，使它们的值具有可比性，然后根据它们在问题域中的相对重要性选择可乘性权重来完成的。一些用例为目标提供了严格的优先级，而再平衡器确保在解决低优先级目标时不会在高优先级目标上倒退。

在使用MIP时，如果有两个目标值完全相同的解，则再平衡器可能会任意选择其中一个，造成多个解之间的不稳定。



ZBS 可能会怎么使用？

了解下 MIP 的一般解法，这样有个直观概念。



用 DAG 的方式将模型尺寸从 O (|O| * |B|) 降低到 O (|O| + |B|) 





混合整数线性规划（MILP）问题

meta 也优化了他们的 MIP，将同类的 object 算到一起，将不会被改动的 bin 提前排除掉，这样大 O 虽然不变，但是系数变了。

https://www.zhihu.com/question/25560707

使用 MindOpt 解混合整数线性规划  https://opt.aliyun.com/doc/1.0.1/cn/html/model/index.html



https://docs.google.com/presentation/d/1SIjfipamWTSzqBA_VYzREbvkUeI1HsfwfWnkY4qtWjg/edit?slide=id.g354cdc775c5_0_603#slide=id.g354cdc775c5_0_603

参考格式：https://docs.google.com/presentation/d/1uBEqs2V9Sme-yDb6gTxpQaHahc3xl69t4m3qFeZpgFg/edit?slide=id.g33789ec6c07_1_1#slide=id.g33789ec6c07_1_1



资源分配问题可以抽象成将物品 object 放入箱子 bin 的问题



资源分配框架关注的是他的可用性和可扩展性。第一点对应是否足够抽象、透明，第二点对应是否足够高效，当问题规模变大，时间复杂度是否可接受。



分支定界法

首先，将问题松弛，即忽略整数约束，将生产数量x和y视为可以取任何实数值。通过线性规划方法（如单纯形法）求解这个松弛问题，得到一个非整数解。

假设松弛问题的最优解为x=4.81,y=1.82，目标函数值Z=356。此时，356成为上界，而任何整数解的目标函数值将构成下界。

由于x和y不是整数，选择一个变量进行分支，比如以x=4.81为基准，分成两个子问题：

分支

- **分支1**：x≤4（假设向下取整）
- **分支2**：x≥5（假设向上取整）

定界与剪枝

- 对于**分支1**，添加约束x≤4，求解新的线性规划问题，得到一个解，并更新下界。
- 对于**分支2**，添加约束x≥5，同样求解并更新下界。
- 每个子问题的解如果包含整数解，则更新下界；如果子问题的最优解的目标函数值小于当前的上界，则该分支被剪枝，因为不可能包含全局最优解。

迭代与细化

- 如果某个子问题的下界大于或等于已知的上界，那么这个子问题的解已经是全局最优解或者可以排除。
- 对于每个未被剪枝的子问题，重复分支和定界的过程，直到所有可能的分支都被探索，或者找到一个整数解，其目标函数值等于上界。

最终，通过不断分支和定界，找到一个整数解，使得目标函数达到最大值，且这个解满足所有的整数约束和原始的线性约束条件。例如，如果在某一步中发现一个整数解x=4,y=5，且目标函数值为340，且没有其他分支能提供更高的目标函数值，那么这个解就是最优解。



这篇论文跟我之前做过的工作比较接近。

https://grok.com/share/bGVnYWN5_475fb307-a291-4e2e-8894-21e67c9e391b





传统优化工具：如线性规划求解器（Gurobi、CPLEX），虽然功能强大，但在超大规模场景下计算开销高。Rebalancer 的表达式图和启发式算法更适合动态、实时优化。



作者基于 Meta 私有云的实际经验，开发了 Rebalancer 框架，旨在提供一种可扩展且易用的资源分配解决方案，并在存储、计算和网络优化中取得了显著成果

Rebalancer 提供了一种声明式的语言，允许用户以高级方式定义分配策略。例如，存储管理员可以指定“确保每个数据分片至少有 3 个副本，且副本分布在不同机架”而无需编写复杂的数学公式。



在超大规模数据中心中，资源分配的目标是优化资源利用率、性能和成本，同时满足复杂的约束条件（如服务级别协议 SLA、故障容忍、能耗限制）。在存储领域，具体问题包括：

- **数据副本放置**：如何在分布式存储系统中分配数据副本，以平衡负载、降低延迟并确保高可用性？
- **存储与计算协同**：如何协调存储节点与计算节点的资源分配，以支持机器学习训练或实时分析？
- **动态迁移**：如何在不中断服务的情况下迁移存储实例（如数据库分片）以应对负载变化？



aliyun 提供的 MindOpt 可以解混合整数线性规划（MILP）问题。



规模小的问题，meta rebalancer 会将其转成 MIP 来处理。



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

也试过用并行的方式优化，但在上升一个数量级的问题规模面前，时间复杂度还是太高了。



local search 和模拟退火的区别

为确保分配的稳定性，通常希望限制或尽量减少改进分配所需的对象移动次数。但模拟退火难以做到这一点



7.7 rebalancer 的限制

有些问题可以用 MIP 建模，但不能用 rebalancer 建模，因为它们不适合将 object 分配到 bin 的抽象。一个这样的例子是将网络流量分配给链路，其中 rebalancer 无法对链路拓扑中的依赖序列进行建模。然而，Rebalancer的低级表达式API和表达式图足够通用，可以支持这些问题，同时显著提高可用性。因此，我们扩展了表达式API和表达式图，以创建更灵活的框架，使MIP问题的建模超越了分配问题。



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