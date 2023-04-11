NFS 接入点的唯一性和高可用性

#### 残留问题

* iSCSI 的重定向功能是 iSCSI 协议有规定，然后咱们跟着实现？NFS 协议没规定，所以 IO reroute 需要自己实现？iscsi reroute 是 L5 层实现的，但 NFS 在 5 层是不清楚的。
* 用 rpc 的方式比用 http 慢的原因在哪？很消耗 CPU 资源，http server 是长连接，而 python zk lib 很慢，每次新建都要 1s。

#### 故事引入

> ESXi + NFS 的 IO 流程

SmartX 最有核心竞争力的产品就是分布式块存储系统 ZBS。作为一套存储系统，ZBS 支持 NFS 和 iSCSI 存储通信协议。但与 iSCSI 不同，NFS 协议中并不包含重定向或类似功能，为了保证 NFS 接入点的唯一性和高可用性，我们以 VMware ESXi  虚拟化平台为例，来介绍 ZBS 中的实现。

在一个分布式集群中，每台物理机上都装有 ESXi 作为虚拟机管理器（Hypervisor），在 ESXI 上，除了多台客户虚拟机之外还有一台运行 ZBS 的对外提供 NFSv3 访问接口的 SCVM。为了 ZBS 能够直接管理到虚拟磁盘，需要将物理磁盘（SSD、HDD）直通给 SCVM。最后，集群中多个 SCVM 通过万兆网络进行通信，组成 ZBS 集群为整个集群提供存储服务。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211015121.png" alt="image-20220821101509096" style="zoom:50%;" />

那么，在客户虚拟机上的应用程序在读写文件的 IO 下发流程如下：

1. Common VM 上 APP 的 IO 请求，经过系统调用、内核处理下发到文件系统（如 Linux ext3）；
2. 文件系统将 IO 请求转换为对虚拟磁盘的 IO 请求并下发到 ESXi 虚拟设备驱动层；
3. ESXi 将 IO 请求转换为对 NFS Export/File 的 IO 请求并转发到 SCVM 的 Access Server；
4. Access Server 将 IO 请求转换为对 ZBS Volume/Extent 的 IO 请求并下发到 Chunk Server；
5. Chunk Server 从具体的物理磁盘上读写数据。

在这个 IO 请求过程中，需要保证 SCVM 上 NFS Server 的高可用性和 ESXi NFS Client 接入点的唯一性。也就是说，需要保证 ESXi 上的一个 NFS Client 始终能够连接到健康可用的 NFS Server，以及同一时刻只会连接一个 NFS Server。

#### IO Reroute

在每个 ESXi 节点上都有一个仅本地可见的私有网络（192.168.33.x/24 网段），持有一个 IP 为 33.1 的私有地址，而各个 SCVM 以 33.2 对 ESXi 提供 NFS 服务。在网络联通正常的情况下，每个 ESXi 节点挂载的都是 33.2 的 NFS 服务，但每个 ESXi 连接的都是各自本地 SCVM 的 NFS Server。当本地 SCVM 异常，需要将 IO Reroute 到其他可用的 SCVM 上。

要做到这个效果，IO Reroute 需要动态配置路由规则。具体而言，根据 SCVM 的健康状况将 ESXi 上 33.2 的下一跳地址指向对应的 SCVM 存储/存储网 IP，路由配置有 3 种情况：

1. 当未配置静态路由时，ESXi 访问 33.2 会使用与 33.2 同网段的 33.1 作为源 IP；
2. 当配置 33.2 的下一跳是 SCVM 的存储网 IP，ESXi 访问 33.2 会使用自己的存储网络 IP 作为源 IP；
3. 当配置 33.2 的下一跳是 SCVM 的管理网 IP，ESXi 访问 33.2 会使用自己的管理网络 IP 作为源 IP。

所以同一个 NFS Client 在下发 IO 请求时会根据 SCVM 的健康状况使用不同的源 IP 连接 NFS Server。

这里面就有几个问题需要解决：

1. 为什么各节点的 NFS IP 都是 33.1 却不冲突？

    从 ESXI 角度来说：

    1. 如果部署了 IO Reroute，ESXi 可以通过存储/管理/NFS 网访问本地 SCVM，但是只能借助静态路由走存储/管理网访问其他 SCVM 的 33.2；
    2. 如果没有部署 IO Reroute，由于 NFS vSwitch 不与物理网卡相连，所以 ESXi 只能通过 NFS 网访问本地的 SCVM 的 33.2，无法访问其他 SCVM 的 33.2 。

    因此 ESXI 只有在访问本地 SCVM 的 33.2 时才会以 33.1 为源 IP 发包，也就是说 SCVM 向 33.1 发包的情况只有一种，即本地 ESXi 走 NFS 网访问 33.2 时的回包，因此 NFS vSwitch 上 33.1 的出口只有一个（本地 ESXI 的 MAC），所以各 ESXI 的 NFS IP 都是 33.1 却不冲突。

2. 怎么判断 SCVM 是否健康？

    IO Reroute 服务会定期（3s）向 Meta Leader 查询集群中所有可用的 Access Server。但是如果有某个 Access Server 异常，Meta Leader 最坏情况下需要等待一个 KeepAlive Time(8s) 才能知道，所以对于 ESXi 从 Meta Leader 上拿到的 Access Server List 还需要检测它是否可以 ping 通。只有这两个条件都满足，才认为 SCVM 健康可用。

3. 你主要干了啥？

    为了 ESXi 能够随意登录任意一台 SCVM，需要在整个集群统一一套为 IO Reroute 而用的公私钥，各台 SCVM 需要将 ESXi 上的公钥写入自己的免密登录列表中并在本地保留对应的私钥。原有的做法是指定一台 SCVM 生成公私钥，然后复制到集群中的每一个节点，有 N 个 ESXi 节点，就要执行 N 条部署指令，操作复杂。

    由于在每台 SCVM 上开机就会启动一个已经运行了 rest-server 的服务，可以对外提供访问 ZBS 存储卷的 RESTful API。那么就可以在这个基础上提供一个处理公私钥的 API，通过 curl 就能实现在任意一个节点上部署整个集群的 IO Reroute 服务。

    > 不用 rpc 的原因是因为通过 python 调用 rpc 的速度远不如用 curl 访问 HTTP JSON

    这个工作有营养的点还是在网络部署上，由于虚拟化，网络规划跟我之前理解的还是很不一样，ESXi 集群 IP 地址规划：

    <img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211015720.png" alt="image-20220821101528697" style="zoom:40%;" />

    每台 ESXi 上有 2 张物理网卡，3 张虚拟网卡，3 台虚拟交换机

    * Mgt vSwitch 与1G 物理网卡相连，ESXi 的 mgt 虚拟网卡通过 mgt vSwitch 与 SCVM 的 mgt 虚拟网卡相连。
    * zbs vSwitch 与10G 物理网卡相连，ESXi 的 zbs 虚拟网卡通过 zbs vSwitch 与 SCVM 的 zbs 虚拟网卡相连。
    * Nfs vSwitch 不与物理网卡相连，ESXi 的 nfs 虚拟网卡通过 nfs vSwitch 与 SCVM 的 nfs 虚拟网卡相连。

    通过存储物理交换机来连通 ESXi 之间的存储网，通过管理物理交换机来连通ESXi 之间的管理网。

#### Login Revoke 

尽管 ESXi 会使用自实现的文件锁协议来阻止对 NFS 文件的并发访问，但还需要在 ZBS 层面做一个唯一性保证，ESXi NFS Client 使用 NFS 的一个常见场景是，当本地 SCVM 网络异常时，ESXi 上 NFS Server IP (192.168.33.2) 会被 IO Reroute 服务重路由到其他可用的 SCVM 存储网。但是由于 TCP 链路切换需要一定时间，所以在重路由后的短暂时间内，可能存在一个 NFS Client 借助两个 NFS Server 同时写入一个 NFS 对象而破坏数据一致性的情况。

在 ESXi 集群中，每个 Access Server 要么是 Session Master 要么是 SessionFollower。

* SessionMaster：一个集群中只有一个 SessionMaster，在 Meta Leader 所在的节点。SessionMaster 是发出 revoke/recover 指令，并做中央决策的服务进程。

* SessionFollower：每台 Access Server 都是一个 SessionFollower ，需要周期性地向 SessionMaster 发送心跳更新状态，并且响应 SessionMaster 的命令；

SessionMaster 和 SessionFollower 之间通过 Session 连接，Session 一般有正常和过期状态，当 SessionFollower 和 SessionMaster 断开一段时间后，Session 会转为过期状态。

为了避免 NFS Client 多点登录 NFS Server，在 Access Server 间引入 Login Revoke 机制。当位于 Access 上的 NFS Server 在收到一个 ESXi NFS Client 连接请求时，Access 作为 Session Follower 向 Meta Leader 上的 Session Master 查询这个 Client IP 是否已经在其他 Session 中登录（正常情况每个 Access 跟 Meta Leader 间只有一个 Session），如果是，那么 Session Master 将通过 KeepAlive 立即下发一条 Revoke 命令到那些跟这个 Client 建立连接的 NFS Server，要求它们立即断开与这个 Client 的连接。只有当 Client 与其他 Server 没有任何连接时，才能跟当前 Server 建立连接，完成登录。

例如当 ESXi B 上 33.2 的下一跳从 SCVM A 的存储网指向 SCVM C 时，SCVM C 上的 Session Follower 通过 Session Master 将 ESXI B 上的 NFS Client 与 SCVM A 建立的 TCP 连接断开。只有成功断开之后，才允许在 SCVM C 上的 NFS Server 登录。

这样就保证了在任意时刻一个 NFS Client 对一个 NFS Datastore 的访问始终由同一个 NFS Server 处理，避免了多点接入现象。当然了，NFS Server 允许跟每个 NFS Client 之间有多个连接，前提是这些连接都建立在一个 NFS Server 上，在 Meta Leader 看来，这些连接都在同一个 Session 中。

这里面有几个问题需要解决：

1. 待补充

