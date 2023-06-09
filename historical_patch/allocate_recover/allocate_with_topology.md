用 uint64_t 来声明 ring_id

初始时生成符合 topo_id 次序的 ring_id，遍历一遍

当有任意一种拓扑行动时，重新生成 ring_id

ring_id 的重新生成是按不同 brick/rack 中节点数量的比例配对，并且要尽可能保持原来的相对次序

不管是首次按照 uuid 生成还是后面按照 ring_id 生成，都要重新考虑以 brick 为粒度算节点数



双活集群中优先可用域 2 副本，次级可用域 1 副本。

UpdateTopoObj/RegisterChunk/RemoveChunk



removeChunk 里面竟然没调用 DeleteTopoObj

Chunk_manager 对外暴露 CreateTopoObj / Delete / update / show / list

CreateTopoObj 只允许创建 rack/brick，不能直接创建 Node（在 RegisterChunk 中实现）

查看节点拓扑 GetChunkTopology（根据 cid 查的）、ShowTopoObj（根据 topo_id 查的） ；

UpdateTotoObj 是允许更新 Node 的 TopoObj，节点的拓扑位置移动也是在这里

1. 节点加入 RegisterChunk。此时没有 ring_id，需要生成；
2. 节点退出 RemoveChunk。此时删除了对应的 TopoObj，之后还需要引发重新生成 Ring_id；
3. 节点移动 UpdateTopoObj，Node 类型的 parent_id 被修改为其他 brick_id/CLUSTER，此时需要重新生成 ring_id；
4. Brick/Rack 位置移动 UpdateTopoObj，如 Brick 从一个 Rack 移动到另一个 Rack（parent_id 变化）也要重新生成 ring_id；

现在要考虑的是跟已有手动的 ring id 的兼容

1. 节点加入 RegisterChunk。此时没有 ring_id，需要生成；
2. 节点退出 RemoveChunk。此时删除了对应的 TopoObj，之后还需要引发重新生成 Ring_id；
3. 节点移动 UpdateTopoObj。Node 类型的 parent_id 被修改为其他 brick_id/CLUSTER，此时需要重新生成 ring_id；
4. Brick/Rack 位置移动 UpdateTopoObj。如 Brick 从一个 Rack 移动到另一个 Rack（parent_id 变化）也要重新生成 ring_id；
5. 删除 brick/rack DeleteTopoObj。

如果有不带有 topo 信息的节点，在自动生成 ring id 时是不纳入考虑，还是给他分配一个默认的 zone/rack/brick id，感觉后者更好一些，但这样就跟之前的情况不一样了，

如果用户没添加 topo 信息，所有节点被当作是都在最远的地方，那么让他们的 ring id 顺序递增就好；

如果一部分节点有 topo 信息，一部分没有，首先在 topo distance 那里就会把没有 topo 信息的节点优先级放在很后面，万一真分配到这部分节点，注意要让他们被随机选到（否则还是会出现聚焦在一个节点的情况）；



