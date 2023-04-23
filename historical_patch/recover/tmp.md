1. 生成 ring id，初始值是 cid，跟现有的 topo 策略能够兼容
2. 对外提供 rpc 允许改 ring id，副本改用
3. elastic allocator 代码，topo
4. 每插入一次盘重新分配一次 ring id

zbs-meta 

看起来 topo_name 只跟 chunk 相关，不依赖于 node，所以把 topo_name 的信息放在 message chunk 上



// 用以拓扑环找点，TODO 怎么让默认值等于 topo_id

什么时候用 response->mutable_position()->CopyFrom(request->new_position()); 这种方式修改值







一致性哈希 虚拟节点 没有考虑拓扑，

ceph Crush 是一致性哈希的改进，考虑了拓扑，但没有没有成功，看能否对他改进



最终不要有复杂的代码实现

不需要考虑容量

Topo allocator

最多 255 个节点



每次新插入节点，重分配获取不会有太大消耗？

