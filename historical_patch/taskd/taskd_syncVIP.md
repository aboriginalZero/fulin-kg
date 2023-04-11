### 任务要求

> ZBS-13104 taskd: sync ARP when task server becomes leader

在 Task Server 成为新的 Leader 之后，目前在配置 VIP 之后会主动的下发一次对应的免费 ARP 以刷新集群中的 ARP 缓存。但是有可能单个报文不足以及时刷新。需要做如下改进：

在成为 Leader 角色后启动一个周期性任务，持续 10s x 6 次，每次执行如下动作：

- 检查本机是否配置了所有应该有的 VIP ，如果没有，则重新配置；
- 尝试对每个持有的 VIP 重新发送一次免费 ARP；

### 从中所得

* 用 vscode 的全局查找来追踪代码，并写出与此业务相关的代码追踪流程

* 用 vscode 的 remote-ssh 插件来做远程开发

* Linux 命令行的快捷键等，从第一个 patch 中熟悉开发、提交流程、远程开发，Linux 的使用、c++ 生产项目的编译

* Primary IP 和 secondary IP 的区别

  一块网卡可以绑定多个 IP，在同一块网卡上的多个 IP，如果网段不一样，就都是 primary IP。如果网段相同，那么第一个 IP 是 primary，后面添加的都是 secondary，当 primary 被删掉时，secondary 默认会自动提升成 primary，本地发包的时候如果 socket 没有 bind 特定IP，会选出口网卡上的 primary IP 作为源 IP。

* ifconfig 跟 ip addr 的区别

  * ifconfig 通过 ioctl 查看 IP，内核会遍历 ip alias 列表，对每个网卡 alias 只显示第一个 IP（不一定是 Primary IP）

  * ip addr 通过 netlink 查看 IP，把所有 IP 都打印出来，包括 primary、secondary IP

* 本地接口指的是用 ip addr 查的本地网卡接口；本地路由指的是用 route 查的路由表项，毕竟一台计算机也能够作为路由器来用。

### 残余问题

* 有限次的定时任务能不能用 LoopOnce 来执行，开一次 coroutine，大致代码如下：

  ```c++
  for (int i = 0; i < N; i ++) {
    Coroutine co([=](void*){ SyncARP(); });
    co.Enter();
    while (!co.stopped()) {
      loop()->LoopOnce();
    }
    sleep(k);	// 每 k 秒执行一次，共执行 N 次
  }
  ```

* ConfigVIP 也放在 CheckVIP 中，那岂不是会每 3s 就发送一次 ARP，不需要 SyncARP？不是这样的，在ConfigVIP 调用的 SetServiceVIP 中逻辑是这样的：

  对于每一个 VIP，找到它对应的本地接口下标和路由表中的下标，判断：

  * 如果他们相等，说明 VIP 设置正确，直接返回 Status::OK ；
  * 如果本地接口中有 VIP 但是路由表中没有，说明这很可能是旧本地接口配置项，那么要从本地接口中删除 VIP；
  * 如果本地接口中没有 VIP，那么要往里添加并分发一次 ARP 包。

  正如其函数名，这是一个配置的过程，正确匹配的前提下不会继续发 ARP，所以需要一个 SyncARP 函数来做这件事。

  > 那有一个问题，ConfigVIP 为什么要每 3s 做一次，照理说成为 leader 时做那一次 ConfigVIP 不就好了，是为了防止人为误删除 route 表或 arp 表吗

* 如果 followers arp 表中 VIPs 还没更新 Mac 地址，用户就通过这个节点访问 VIP，那么这个 VIP 就会配上错误的 MAC 地址，而造成无法通信，好在新 leader 会先发送一次 arp 包才 StartDispatcher，之后通过 SyncARP 每 3s 发一次，共 10 次，想说的是，为了保证高可用，这个频率是不是高一点，比如每 1s 发一次，这样才能更好地保证 VIP 的高可用吧？

* vip 怎么设置的，集群出错的时候怎么保证 vip 设置正确？这个集群出错怎么理解，是什么东西在什么情况下会出错？

* 随着 leader 的切换，watch -n 1 arp -a 192.168.87.101是会变化的，但是 tshark -i port-mgt arp -R "ip.addr eq 192.168.87.101" 观察不到，跟 yutian 讨论过，他的说法是 tcpdump 能抓到就行，tshark 主要用来分析，tcpdump 是拿来抓包的，但其实我还挺疑惑的。

* 用 tcpdump 来抓包好像不稳定？在抓包之前要先 arp -a 查一下有没有，初步排查是要先保证在本地 arp 表中有相关表项、但是照理说一个节点成为 leader 之后，其他节点都能收到 arp 包才是，但是目前没有观察到。

### 需求分析

task-center 提供 VIP 服务的实现方式：

* task server leader 节点启动时会从 zk 中获取 VIP 列表，将每个 VIP 依次添加到本地网络接口中。当 VIP 配置发生变化时，leader 节点会得到通知，更新本地配置。
* 非 leader 节点启动后会清空本地接口上的 VIP 地址。

VIP 的实质就是在 leader 节点上配置一个 secondary IP，当 leader 切换时，删除旧 leader 的 secondary IP 并在新 leader 上配置。

当集群中产生新 leader 后，在 follower 上的 arp 表仍然留有 VIP 的老 leader 的 Mac 地址，因此，刚成为新 leader 的节点要主动在集群中下发 arp 包，给 followers arp 表中 VIPs 新的 Mac 地址，这样才能保证 VIP 功能的高可用。原先的做法只下发了一次免费 arp，有可能不足以及时刷新，需要多发送几次。

### 解决思路

解决思路是在 VIPServer 类中新增成员变量 sync_arp_count_，其决定了用来刷新集群中的 ARP 缓存的 SyncARP 函数的执行次数。开发流程如下：

1. 检查 port-mgt 端口上是否有来自 vip 的 arp 包， tcpdump arp -i port-mgt | grep vip(192.168.87.101)

2. 在开发机上 build 目录下，ninja zbs-taskd 、 strip ./src/zbs-taskd、 scp ./src/zbs-taskd smartx@192.168.91.186:/tmp

3. 在节点192.168.91.186上 /usr/sbin 目录下，rm /usr/sbin/zbs-taskd、cp /tmp/zbs-taskd . 、systemctl restart zbs-taskd、tailf /var/log/zbs/zbs-taskd.INFO

### 过程疑问

* 当路由策略匹配失败，即没有设置默认路由，也没有匹配的子网路由时，task server 将通过 exit(1) 的方式退出，是否可以改成 Stop 的方式？除了已知的两种，也有可能会是其他原因导致的路由策略匹配失败，还是通过 exit 来生成 coredump 的方式来帮助之后的排错吧。
* 一开始不知道怎么复现问题，后来发现是观察 arp 表
* TaskDispatcher 的构造函数中，用 consensus::ZooKeeper *zk 初始化 VIPServer vip_server_，这是强制转换了吗？不是，而是在构造 VIPServer 对象时需要对成员变量 zk 赋值。

### 概念补充

* VIP 设置相关策略：

  VIP 服务仅需要配置 IP 地址，本地接口地址与子网掩码将从本地路由表中根据最长匹配原则获得。VIP 不改变本机的路由策略，如果没有更精确的路由，最终将匹配到默认路由上（如果 leader 节点上配置了默认路由）

  ```
  192.168.0.0		255.255.255.0	if1
  192.168.128.0	255.255.255.9	if2
  ```

  比如本地有以上两条路由，如果待配置的 VIP 地址为 192.168.128.x 时，最长匹配原则将得到对应的接口地址为 if2，子网掩码为 255.255.255.0；如果待配置的 VIP 地址为 192.168.1.x 时，最长匹配原则将得到对应的接口地址为 if1，子网掩码为 255.255.0.0；如果待配置的 VIP 地址为 192.101.x.x 时，因为没有设置默认路由，也没有匹配的子网路由时，task server 将通过 exit(1) 的方式退出，由 zk 选出的新 leader 将尝试配置 VIP

* VIP 的作用？VIP 的引入，是为了实现 smartos web 控制台、SCSI 服务的高可用

  * 通过命令行为 ISCSI 配置一个 VIP 地址，供上层应用作为 iSCSI 的服务地址
  * 将域名绑定到VIP，或者让用户通过VIP访问控制台服务、当集群某节点故障时可以自动切换到当前集群中能提供服务的其他节点。目前的部署模式下每个 ZBS 节点都会启动 fisheye 服务。假设部署了 192.168.1.101 ~ 103 三个节点。 用户可以通过任一 IP 地址访问 fisheye 页面。假如用户 使用 192.168.1.101 地址直接 访问 fisheye。 在101 节点故障时候（例如宕机），该地址将无法访问。用户需要改用其他 ip 地址访问服务。 如果为 fisheye 服务配置了 VIP， 如192.168.1.100。只要有一个节点存活，用户通过 192.168.1.100 VIP 均可以正常访问 fisheye 服务。不再需要更换 IP 地址。在利用DNS 通过域名来访问 fisheye 的场景下，可以将域名绑定至该 VIP。实现 fisheye 服务的高可用。

* 对 TaskServer 的理解。TaskServer 是 ZBS 的异步长任务处理模块。提供诸如存储池间移动/复制数据、远端备份的数据复制等耗时较长的长任务处理服务。与 MetaServer 类似，多台 TaskServer 组成 Task 集群。经由 ZooKeeper 选出唯一的 Task Leader 负责任务的调度分发，其余 TaskServer 只作为 Runner 实际运行长任务。当 Task Leader 故障时，集群会重新选举出新的 Leader，而已在运行中的任务不会受到影响。新的长任务请求将由新的 Leader 处理。 当 Task Runner 异常时，Leader 将会感知到任务异常，重新将任务调度至新的可用 Task Runner 处理。新的 Task Runner 将会从获取任务的当前进度状态，继续任务。此外，Task Runner 利用 ZooKeeper 来确保每个任务在任意时刻只有至多一个 Runner 在处理。VIP Service 是 ZBS TaskServer 提供的虚拟 IP 服务。当为某一个服务配置了 VIP 后，集群内有且仅有一个物理节点（即 Task Leader）对应这个虚拟的 IP 地址。VIP 服务使得可以通过一个不变的 IP 地址使用接入服务。例如为 iSCSI 指定 VIP 地址之后，即可通过不变的 VIP 地址访问 ZBS 提供的 iSCSI 接入服务，而无需为每个 iSCSI 目标根据连接配置不同的服务地址。TaskServer 保证仅有 Task Leader 节点持有 VIP。在 Leader 异常切换时，老 leader 会主动清理 VIP，在新 leader 部署 VIP 配置。

* VIP Service 为集群提供虚拟 IP 配置服务，允许用户为不同的 Service 配置 VIP。Task Center 保证 VIP 将出现且仅出现在 Task Leader 所在节点上。主要的用途是分离部署场景下提供统一的 iSCSI 接入点与管理界面（FishEye） 接入地址服务。VIP Service 与其他 Service 不同，不需要 Runner 参与。实现的方式为在 Task Server 中加载 VIP Server，当前节点为 Leader 节点时将从 ZK 中读取 Service 与对应的 VIP 列表，尝试在通过 NetLink 接口在本地配置 VIP。如果配置失败将重启节点，触发 Leader 节点转移，以在其他节点尝试配置。如果 Task Server 启动时发现自身并非 Leader 节点，则会检查节点是否有 VIP 残留，如果有将尝试清理配置。Leader 节点上的 Dispatcher 将对外提供 VIP 设置与清理接口。Dispatcher 收到请求后将更新 ZK 上对应配置节点。VIP Server 通过监听对应节点的状态，感知到 VIP 配置更新后刷新本地配置。
