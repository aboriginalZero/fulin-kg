1. 生成 ring id，初始值是 cid，跟现有的 topo 策略能够兼容
2. 对外提供 rpc 允许改 ring id，副本改用
3. elastic allocator 代码，topo
4. 每插入一次盘重新分配一次 ring id

zbs-meta 

看起来 topo_name 只跟 chunk 相关，不依赖于 node，所以把 topo_name 的信息放在 message chunk 上



// 用以拓扑环找点，TODO 怎么让默认值等于 topo_id

什么时候用 response->mutable_position()->CopyFrom(request->new_position()); 这种方式修改值

```c++
TopoObj topo_root;
// chunk_id 到 Chunk.topo_id 的 uuid 的映射
// chunk_id 到 obj.id() 的 uuid 的映射
std::unordered_map<cid_t, zbs_uuid_t> chunk_topos_;
// TopoObj id 到 TopoObj 的映射
std::unordered_map<zbs_uuid_t, TopoObj> topo_objs_;
// Chunk.topo_id 到 ChunkTopolopy 的映射
std::unordered_map<zbs_uuid_t, ChunkTopology> chunk_topo_cache_;

message Chunk {
  optional uint32 id = 1;
  // pid is the brick id for chunk, it could be zero
  optional uint32 v1_parent_id = 2 [deprecated = true];
  // topology info
  optional bytes topo_id = 41 [default = "", (zbs.labels).as_str = true];
  // only internal use, not valid in all scenario
  optional bytes zone_id = 42 [default = "", (zbs.labels).as_str = true];
};

message TopoObj {
    optional TopoType type = 1 [default = NODE];
  	// 经过 uuid 就是 obj_id，应该就是 Chunk.topo_id
    optional bytes id = 2 [(zbs.labels).as_str = true];
    // if no parent is set, the default is the root object called "topo"
    optional bytes parent_id = 3 [default = "topo", (zbs.labels).as_str = true];
    optional bytes name = 4 [default = "", (zbs.labels).as_str = true];

    optional uint64 create_time = 5;
    optional bytes desc = 6 [default = "", (zbs.labels).as_str = true];

    optional Position position = 12;
    optional Capacity capacity = 13;
    optional Dimension dimension = 14;
};

message ChunkTopology {
    optional bytes pod_id = 1 [default = "", (zbs.labels).as_str = true];
    optional bytes rack_id = 2 [default = "", (zbs.labels).as_str = true];
    optional bytes brick_id = 3 [default = "", (zbs.labels).as_str = true];
    optional bytes zone_id = 4 [default = "", (zbs.labels).as_str = true];
};

message ChunkSpaceInfo {
  // 包含这台 chunk 上已使用空间，有效 cach，正在接收/发送/预留的副本数
  optional uint32 id = 5;
}
```





一致性哈希 虚拟节点 没有考虑拓扑，

ceph Crush 是一致性哈希的改进，考虑了拓扑，但没有没有成功，看能否对他改进



最终不要有复杂的代码实现

不需要考虑容量

Topo allocator

最多 255 个节点



每次新插入节点，重分配获取不会有太大消耗？

