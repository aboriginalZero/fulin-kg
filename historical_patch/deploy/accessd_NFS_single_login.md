## NFS 统一接入点

### 已有代码逻辑

nfs_server 保证接入点的唯一性代码流程

1. TcpZbsTransport::Listen --> Transport::AcceptClient --> NFSTcpZbsTransport::Accept
2. 一系列的 AcquireNFSLoginPermission 传递，NFSZbsClientProxy --> ZbsClient --> AccessHandler --> SessionFollower --> SessionClient --> SessionMaster
3. 当出现多个 session 时，SessionMaster::RevokeNFSClient，给指定 session 设置需要 revoke_client_cmd
4. MasterSession::Notify，并要求在 session_follower 的 keepalive 过来的时候，立即返回带有 revoke_client_cmd 的 keep alive
5. 在 access server 上，AccessHandler::HandleAccessResponse 在对返回的心跳包的处理中，通过 revoke_client_cb_ 回调处理 revoke_clients，而这个 revoke_client_cb_ 是 chunk StartServer 中通过 access_handler->RegisterSessionRevokeClientCb(BIND(&nfs::NFSMegaServer::RevokeClient, nfs_server, _1)) 注册的
6. NFSMegaServer::RevokeClient 分别调用 nfs_server.RevokeClient 和 mount_server.RevokeClient，最终调用到 NFSService::RevokeClient --> Transport::NotifyDestroy --> eventfd_write(destroy_efd_, 1) --> TcpZbsTransport::HandleDestroy 中关闭当前 TcpZbsTransport（也就是这个 tcp socket 连接） 



Nfs server 在 scvm 的 access/chunk 上，nfs client 在 ESXi 上，zbs session 是运行在 meta leader 和各台 chunk 之间，Nfs client 是感知不到的。正常情况下，每个 follower 会跟 master 维持一个 session。

session  的 uuid 属性唯一标识 session ，看 master 中 session_ 的 key 就是用的 uuid，那接下来看一下 epoch 属性是怎么更新的？在 Master 向 follower SendKeepAlive 的时候 +1，也就是说他们之间每 keepalive 一次，epoch 就会 +1，用来避免 master 响应 follower 的重复心跳或过期心跳。

每个 session follower 都持有一个 private_zbs_client 和 private_session，每次跟 master 通信都用 private_session 的 session_epoch() 的 uuid 和 epoch 来填充 request。这个 private_session 在 KeepAliveLoop 中会根据 keepalive 的 response 来更新自己。

因为 AcquireNFSLoginPermission 的 request 中会携带 session_epoch().uuid()，所以虽然会出现使用多个 IP 作为 request.client_ip()，但是用的是一个 session，即 session_epoch().uuid() 是一样。

client_ip 是建立连接的 socket ip，但 master 会根据 request  的 session_epoch().uuid() 来拿到当前这个 client_ip 对应的 session



* local_nfs_clients_ 是当前 scvm 的本地 ESXi 的非 33.1的 IP，nfs_client_ip_to_session_map_ 记录的是 ESXi 的 IP 跟 session 的映射关系， 

* login_nfs_clients_ 是当前 scvm 的 session 上登录的 IP，

* session_ 是 session master 记录的当前集群内所有 session

* kvs_ 记录了 client_ip 和 login_clients 的键值对

为什么要 CheckRequestSession 是因为 request 是指针，它的 session_epoch 会实时更新？

Follower CreateSession 的时候，response 携带这个 session 的 uuid，之后都用的这个 uuid



ZBSProxy 是为了自由重试，通过在主机侧运行 ZbsProxy 来替代 Linux 自带的 MPIO 软件，ZbsProxy 提供 ISCSI Target 和 NFS Client 接入功能，iscsi initiator 或 nfs client 连接本地的 ZbsProxy，访问集群的存储对象。在这种部署模式下，ZbsProxy 基于 ZBS 的 Session 协议，并且知晓自己是一个特定 I/O 访问的拥有者，不会有其他的 ZBSProxy 会转发此 I/O，因此 ZBSProxy 可以自由的重试。Session Client 中通过 Controller ctrl(kLeaseIntervalMS) 来设置重试时间

### 代码优化

先明确这个函数涉及到的 3 种 Session。调用 SessionMaster::AcquireNFSLoginPermission 所在的 Session follower 持有的 session 称为 local_session，其他 session 称为 remote_session，need_revoke_session 只可能是 remote_session。

对于 request 传入的 client_ip，总共就 3 种情况：

1. 33.1
2. hyper IP（集群中ESXi / Xen 的管理/存储网）
3. 其他 IP

为了复用不同情况 need_revoke_session 的处理逻辑，我们按以下 3 种情况分类：
假定现有一个3 节点集群，各节点存储 IP 为 Hyper A ~C :10.1.73.101~103、SCVM A～C：10.1.73.104~106，

1. 既不是 33.1 也不是集群中的任何一个 hyper ip。（单测的编写有这种情况，但是这种情况实际会发生吗？）
    需要 revoke remote session 上的 client_ip。
    比如用 10.1.73.222 登录存储网为 SCVM A，需要 revoke 73.222 在 SCVM B、C 上的登录。
2. 要么是 33.1 要么是 local_session 的 hyper ip。
    需要 revoke client_ip 登录的 local/remote session 上所有不在 local session 的 hyper ip。
    比如用 33.1 或者 Hyper A 存储网 登录 SCVM A，需要 revoke Hyper A 的存储/管理 IP 在其他 SCVM 上的登录。
3. 是一个非 local_session 的 hyper ip，也就是 remote_session hyper ip。
    需要 revoke client_ip 所在的 remote session 上的 33.1 和这个 remote session 不在 local_session 上登录的其他 hyper ip。
    比如用 Hyper A 存储网 登录 SCVM B，那么需要 revoke SCVM A 上的 33.1 和在 SCVM A、C 上的 Hyper A 的管理/存储 IP。

补充两点说明：

1. 由于以上实现，能够保证一个 Hyper 上至多 2 个 IP 短暂时间内同时登录，所以代码实现中需要 revoke 的只有一个 IP（ need_revoke_ips 和 need_add_ip 只有一个元素），被 yutian 举出反例了，还是会有多个 IP 的情况。
2. 假设 Hyper A 的存储网需要登录 SCVM B，如果 Hyper A 的管理网已经登录了 SCVM B，目前的策略是不去 revoke 的，因为它们连接的是同一个 NFS Server。

残留问题：

TODO 能不能把 revoke 模块中 local_session 的部分放在 single_login 里面做？进一步简化代码逻辑。可以，revoke 本身就是移除和添加的过程。只是中间因为移除是一个异步动作，在移除结束之后相比 single login 要补上一些有效性检查。

TODO single login 中 CheckRequestSession 返回 EBadSessionEpoch 时还可以 Update/Clean LoginBFSClient 吗？目前是仅在 ESessionExpired 时处理。当 revoke 成功的时候，这个时候即便是新的 session 也不应该再有记录；当 single login 成功的时候，这个时候新的 session 上应该要有记录。

