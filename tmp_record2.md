独立处理 prio-extent 的迁移逻辑，不跟 normal extent 搅和。

even prio-pextent 迁移

跟已有的 normal pextent 均匀策略保持一致

节点移除

replace_cid 是要被移除的节点，因此为该节点上的所有 prio-extent 选符合条件的 dst_cid 和 src_cid

```
dst_cid should meet: (seq means priority)
  1. not failslow
  2. same zone with prefer local
  3. PrioRecoverComparator 

dst_cid must meet:
  - not in alive loc
  - not in exclude_cids
  - enough remain valid space
  - not in the same zone where in stretch cluster and size(all_src_chunks) > 	
    kMinReplicaNum, and all src chunks except replace_cid are located in

src_cid should meet: (seq means priority)
  1. not failslow
  2. same zone with dst_cid

src_cid must meet:
  - none
```

这边如果用了 PrioRecoverComparator 除了支持 3 种 level 外，还会支持本地化 + 局部化 + topo 安全的。但后续的迁移又不支持局部化，我认为应该统一起来，都不支持局部化。

另外，目前代码里节点移除时的迁移跟副本恢复用的是同一个策略：ElasticAllocator::AllocRecoverReplica()

所以建议应该让 PrioRecoverComparator 继承 TopoAwareComparator，只保留本地化 +  topo 安全。



集群欠载：以本地化 + topo 安全为目的做迁移。

每 1h 触发一次迁移扫描，扫描所有的 pextent，过滤出符合条件的 candidate pextent，并按顺序选出符合条件的 dst_cid、replace_cid、src_cid，若三个都不是 kInvalidChunkId，生成 migrate cmd。

```
pextent must meet:
  - prio-extent (including thick)
  - not temp replica
  - not even

dst_cid should meet: (seq means priority)
  1. prefer local

dst_cid must meet:
  - not failslow
  - enough cmd quota
  - enough remain planned prs

replace_cid should meet: (seq means priority)
  1. same zone with dst_cid
  2. failslow
  3. not owner

replace_cid must meet:
  - not prefer local
  - not owner
  - topo safety replace_cid <= dst_cid

src_cid should meet: (seq means priority)
  1. same zone with dst_cid
  2. not failslow

src_cid must meet:
  - enough cmd quota
```

集群过载：

以容量均衡 + topo 安全为目的做迁移。

每 5 min 触发一次迁移扫描，先扫描所有的 chunk，从高负载 chunk 集合中选出符合条件的 replace_cid，从低负载 chunk 集合中选出符合条件的 dst_cid。

```
replace_cid should meet: (seq means priority)
  1. bigger allocated_prs (which means high_prior_load first and medium_prior_load second)
  
replace_cid must meet:
  - not low_prior_load (which means either medium_prior_load or high_prior_load)

dst_cid should meet: (seq means priority)
  1. smaller allocated_prs
  2. not lease owner （这个只能因 pid 而异，先不实现，有点复杂）

dst_cid must meet:
  - not failslow
  - not select by prevent src_cid
  
  // topo safety根据不同 pid 不同，后面 3 个是随着 pid 的数量而变化
  - topo safety replace_cid <= dst_cid
  - enough cmd quota
  - enough remain planned prs when cluster avg allocated_prs < cluster avg planned prs 
  - enough remain cache space when cluster avg planned prs <= cluster avg allocated_prs < cluster avg cache space
```

此时得到 replace cid 和 dst_cid，再从 chunk 本地过滤出符合条件的 prio-extent，并从活跃副本位置中选出符合条件的 dst_cid。

```
pextent must meet:
  - prio-extent (including thick)
  - not temp replica
  - not even

src_cid should meet:
  1. same zone with dst_cid
  
src_cid must meet:
  - not failslow
  - enough cmd quota
```

