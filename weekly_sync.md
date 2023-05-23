1. 厚制备 elf 没有传递 Volume 所属的 Chunk id 给 ZBS，所以副本分配没法使用本地优先，此时第一副本会挑选为分配时集群空间比例最小的节点，然后其他 2 个副本在第一个副本的基础上再按照局部化原则选择。对应的 Extent 有真实 IO 之后会打上 prefer cid 的标记，再被定时扫描触发副本迁移；

2. 2 副本的数据分布，prefer local 是 2，副本分布一开始在 chunk 2 3，stop chunk2 之后，recover 从 3 到 4，副本分布是 3 和 4。此时在 chunk 3 上写 pid1，chunk4 上写 pid2，对应的 lease owner 分别是 3 和 4，chunk2 start，scan immediate，这个时候把 pid1 4 上的副本迁移到 2 上好理解，因为此时满足局部化/本地化期望副本分布是 2 3。

   对于 pid2，此时 lease owner 在 4，触发的副本迁移 dst 是 prefer local = 2（dst_cid 的选取规则是优先选择没有副本且不处于 failsow 的 prefer local，次选符合期望分布，即符合 LocalizedComparator 分配策略的下一个副本），replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid；那在这里因为 4 是 lease owner，3 又符合以 2 为 prefer local 的分布期望，所以 replace_cid 应该是 0，也就是不触发 pid2 的副本迁移才是，当 lease owner 过期（owner = 0），此时就可以将 pid2 在 4 上的副本迁移到 2 了。

   （跟着这个 case 理清了 src/dst/replace cid 的选取规则）

3. 加入和未加入机架的节点共存的场景，高负载做数据均衡的时候，数据理论上不会分配到未加入机架的节点，实在没有在机架中的节点可以选的时候会选到未加入机架的节点。把未加入节点视为同一个 rack 的一个 brick。

4. 双活集群情况下，初次分配副本的逻辑是先在 prefer local 所在可用域按照拓扑顺序分配 2 副本，然后再在远端可用域分配距离上一次选择更近的 chunk 分配 1 副本。副本迁移的话是分情况讨论：

   1. 副本如果被写过的话，是选择业务虚拟机当前接入点所在可用域作为数据优先写入的可用域，数据会自动向该可用域聚集，达到副本 2: 1 的效果；
   2. 如果是厚置备且从未写入的数据，将迁移到集群默认的优先可用域（用 zbs-meta session list 可以查看节点的 zone id，zone id = default 说明在集群默认的优先可用域）。（没有 prefer local 信息，所以给到 默认的优先可用域）

   这个是在双活文档中找到的，meta in zbs 等并没有（文档没有聚拢），代码里面这个逻辑在哪呢？

   双活集群的考虑感觉是一个挺复杂的事（双活集群是未来准备有，金融/医疗行业对其需求很高，目前我们的客户就 10+，规模比较小，只放核心数据）

5. migrate 时，replace_chunk 会避开 prefer_cid，即使 prefer_cid 处于 failslow。实际上 replaced_cid 的选取规则是从活跃副本中选出不满足期望分布且不是 lease owner 的 cid，如果有多个可选，则选择第一个 failslow 的 cid； 而 prefer_cid 一定是在期望分布里的，所以 prefer_cid 一定不会作为 replace_chunk。

   另外一个考量是，虽然此时 prefer_cid 是 failslow 的状态，但他此时只是网络差，还是能正常读写本地磁盘，那么此时如果有读操作，直接读本地磁盘就返回，如果有写操作，由于多副本机制需要走网络写其他节点，会造成写性能很差。而如果因为 failslow 把 prefer_cid 的副本踢掉，那么本地没有副本，此时读写操作都要走网络，会造成读写性能都差，比前面那种情况更糟糕。

   这个问题应该由 elf 那边来解，当节点 failslow 的时候，vm 应该要自动调度到非 failslow 的节点，这样 prefer local 后续也会变更（需求已经提给他们了）

   这个过程也把 chunk 服务隔离的文档看了，感觉基本能看懂，然后发现这只是网络故障处理的一部分，网络故障处理又只是 ZBS 故障处理机制的一部分（磁盘故障、OS 与节点异常、ZBS 相关的服务如 chunk/meta/zk/taskd/io reroute 异常），这个文档含金量很高。

6. access_manager 跟 recover_manager 中日志不一样的情况，recover cmd 最终是通过 access manager 的 keepalive 下发到 chunk 上执行，在这之前会给这条 recover cmd 指定 lease owner（分配的优先级是 1. 该 pid 已有的 lease owner；2. src_cid；3. dst_cid；4. 从非 slow_cids 中根据 [owner priority](https://docs.google.com/document/d/1Xro2919inu3brs03wP1pu5gtbTmOf_Tig7H8pfdYPls/edit#heading=h.2hivgtf3odem) 选一个 cid），

   并且如果此时 lease owner 跟 src_cid 不同，且跟 dst_cid 不同，且 lease owner 上有活跃副本（说明它是健康的），会把 recover cmd 的 src 修改成 lease owner。

   这个挺关键的会变更 recover cmd src 的逻辑被写在 AccessManager::EnqueueRecoverCmd() 方法里，比较隐秘， yutian 5 年前写的代码。

7. 允许 RPC 产生恢复/迁移命令，可以指定源和目的地和（可去掉副本），在运维场景或许会有用。

   输入 pid, src, dst 

   输出 pid, current loc, active loc, dst, owner, mode

   搜索 AddRecoverCmdUnlock

   对于 pid src dst 应该会有限定规则吧？外部 rpc 触发的 recover/migrate 优先级应该要更高吧？插到队列第一条？recover/migrate 应该要分开讨论吧？

   合法且和当前恢复不冲突，prefer local 和 topo 相关的不相关

   做一次冲突检查，

8. zbs cli 显示集群整体的负载情况？目前是用 zbs-meta chunk list 看每个节点的负载然后自己手动算

9. 自动拓扑提供一个开关 flag，自动拓扑是会去修改用户设置的 ring id，这个在用户侧要怎么解释？不用开关，允许 ring id，多加点单测。

10. 副本策略和 recover，access manger/ recover manager，

11. 在看敏捷恢复和临时副本的文档，2 年前看不懂现在才发现 hping 写的文档质量那么高。

12. 觉得我最近哪里不行？对我有什么建议？Recover 相关的 patch 修完开始做去重还是分层的工作？

13. 先做分层、然后做 EC、分布式改造、然后去重

14. 7 月会带上 pin in performance，

15. 7 8 月才能结束 recover 相关的东西吧，参与去重的子任务







未来开发重点：分层设计、EC、去重、元数据共识改造

分布式改造或分层

性能改进：单节点 55w、fengli、yuhua、haoqian、tianze

年底 EC：jiewei、sijie、lianpeng、kaiqi、haiwei

引擎：wangsai、dongdong、yangliang、shihao

元数据改造：fanyang 

分层：weipeng

warmup 作为入口，未来给分层改进

出一个大概的设计文档，然后接到的人出一个详细的设计文档

recover 策略改进，10 个 Jira，一个月之内做完，多了解前因后果

recover 看文档，

看一遍 recover 相关的代码

去重和 EC 依赖于分层结构

元数据改造的大文档 

预计到明年年底结束，今年再招聘 0-2 个人

