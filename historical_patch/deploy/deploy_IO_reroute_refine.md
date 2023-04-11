### 任务要求

> ZBS-13105：IO reroute deployment change 

提供改进的部署模式，实现如下功能：

1. 对于新部署集群，一键安装与部署整个集群的 IOReroute 服务（需要提供命令行，因为 ESXi 关联信息可能未必在集群部署之后立即提供，有可能要独立触发）； 

2. 对于新节点加入，一个命令行执行单个节点的 IO Reroute 部署；

3. 对于 ZBS 版本升级，执行 Reroute 版本检查，如有不一致，则可以检测报告。后续可以通过另外一个命令行一键升级；

4. 测试验证获取 session 的方式是否可以调整为登陆节点后使用 wget 请求 rest service 而不是命令行（先需要比较二者的资源消耗）；

    timeout 会出现 

需要注意的是，对于新节点加入/版本升级的情况，在有可能的情况下，尽量不调整 ESXi 上现有的路由配置，避免 VM IO 受到影响。开始改进之前需要给出一个简单的描述/设计文档。

### 常用命令

1. 清理所有节点的 IO 重路由

    1. 所有 SCVM 上执行 zbs-deploy-manage clear-hypervisor
    2. 清除所有 ESXi 上的 33.2 的路由 esxcfg-route -d 192.168.33.2/32 10.1.73.104

2. 再执行安装流程

    1. 所有 SCVM 上执行 zbs-node collect_node_info

    2. 登录其中一台 SCVM 上执行 zbs-deploy-manage deploy-hypervisor --gen_ssh_key

        执行成功将会看到 Success to deploy key! 提示信息；

    3. 登录其他 SCVM 执行 zbs-deploy-manage deploy-hypervisor

        执行成功将会看到 Success to deploy key! 提示信息。

3. 检查状态

    在每个 ESXi 上使用命令 tail -f /var/log/scvm_failure.log ，确保一直在输出。在 IO 重路由脚本部署完成 3 分钟后，在 ESXi 上运行命令 esxcfg-route -l 。如果看到所有的 ESXi 上的 192.168.33.2 被路由重定向到本地 SCVM 的数据网络地址上，则为通过。

4. 拷贝流程

    ```shell
    scp -r tuna/deployment/ zbs_deploy/cmd.py smartx@192.168.30.40:/tmp
    yes | cp -r /tmp/deployment/tool/ /usr/lib/python2.7/site-packages/tuna/deployment
    yes | cp -r /tmp/deployment/script/scvm_failure_common/ /usr/share/tuna/script/
    yes | cp /tmp/cmd.py /usr/lib/python2.7/site-packages/zbs_deploy/cmd.py
    
    
    scp pyzbs/tuna/deployment/tool/deploy_hypervisor.py pyzbs/tuna/deployment/script/scvm_failure_common/scvm_failure_core.sh pyzbs/tuna/deployment/script/scvm_failure_common/gen_scvm_failure_loop.sh pyzbs/zbs_deploy/cmd.py smartx@192.168.30.40:/tmp
    yes | cp /tmp/deploy_hypervisor.py /usr/lib/python2.7/site-packages/tuna/deployment/tool/.;yes | cp /tmp/cmd.py /usr/lib/python2.7/site-packages/zbs_deploy/.;yes | cp /tmp/scvm_failure_core.sh /usr/share/tuna/script/scvm_failure_common/.
    ```

5. ESXi 常用指令

    ```
    esxcfg-route -l 	# 查看路由表
    esxcfg-vmknic -l	# 查看网卡
    esxcli vm process list	# 查看虚拟机
    esxcli network nic up -n=vmnic2 	# 开启关闭指定物理网卡
    esxcli network vm list # 获取虚拟机网卡 world_id
    esxcli network vm port list -w <world_id> # 获取各虚拟网卡的端口
    pktcap-uw --switchport <switch_port> -o 1.pcap # 抓取从该 vnic 发出的包
    pktcap-uw --switchport <switch_port> --capture PortOutput -o 2.pcap # 发往该 vnic 的包
    pktcap-uw --uplink vmnic0 --ip 192.168.18.82 --capture PortInput -o 55.pcap # 抓取物理网卡上指定 IP 的包
    ps -c | grep scvm_failure_loop.sh | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill
    ```

### 从中所得

1. ssh 和 scp 的理解看文档【ssh_scp_use.md】，以及对 shell 的使用【shell_use.md】

2. 抓包 tcpdump -e -i ens256 dst 192.168.33.2 表明抓取 ens256 网卡上目的 IP 是 33.2 的数据包并显示 Mac 地址，如果需要更全的信息可以通过 -w output_path 保存 .pcap 文件并用 wireshark 打开

    Wireshark 过滤语法

    ```
    ip.src == 192.168.30.20 and ip.dst == 192.168.30.40
    http and icmp
    tcp.port == 80
    ```

3. 在使用 ping 时，可以指定网卡指定次数 ping 某个地址，如 ping 192.168.30.20 -c 300 -I vmk0 （大写的 i）

4. 一台机器上每张网卡都需要配置网关以及对外 IP，物理网卡应该只会处理广播报文和目的 MAC 是自己的报文。

### 残余问题

1. crontab 定时用 */20 就可以每 3s 执行一次，然后执行之前先 kill 掉旧的脚本（不过有可能存在 3s 不够执行的情况吧？），这样是不是比原本的方式好一些，起码输出日志会干净些
2. `zbs-meta session refresh_address $local_session_id $local_ip`的作用
3. Clear-hypervisor 不需要手动删除 esxi 上的路由，但是没考虑 rdma 的路由（存储网络指向网关）
4. 存在 scvm 挂掉但是 esxi 还在运行的情况吗，先判断 scvm 需不需要遍历发放公钥、清除 scvm_failure 脚本
5. 执行成功界面将显示 Success to deploy key!  提示信息，这个需要想一下怎么样判断成功？应该得做一个集群内部的联通判断以及 33.2 路由优先指向存储网，次等指向管理网？
6. zbs-node collect_node_info 以及 rdma 相关的，应该是可以合并到 deploy-hypervisor 一条指令的吧

### 需求分析

由于禁止用 root 账号 ssh scvm，且 scvm 暂未提供写 /root/.ssh/authorized_keys 的 RPC/ RESTful API（这是更理想的部署方式），所以还是无法做到一条指令部署所有的 scvm 机器。同时，由于 hypervisor scvm 之间需要共享一套公私钥，所以需要指定一台机器来生成 reroute_id_rsa 公私钥。

部署时每一台 scvm 仅操作他所在的 hypervisor，流程如下：

1. 指定 scvm 在本地生成 reroute 公私钥，其他 scvm 从任意一台 hypervisor 上拉取 reroute 公私钥
2. 为了让所有 ESXi 可以免密登录所有 scvm，添加 reroute 公钥到当前 scvm 的 /root/.ssh/authorized_keys
3. 生成 hypervisor 需要运行的脚本（包括 reroute 脚本），并上传到 hypervisor
4. scvm 通过 ZbsMeta session 拿到初始 cluster ips，并上传到 hypervisor
5. 清除 hypervisor 上正在运行 reroute 脚本的进程
6. 配置 hypervisor 的 crontab，定时执行 reroute 脚本。尽管 reroute 脚本 在进入主逻辑循环之前，会有一个 reroute 脚本进程数的检查，目的是保证当前就一个 reroute 脚本在运行。
7. 配置 rc.local.d 中的 local.sh，该目录下的 shell 脚本在每次开机时都会被执行一次，其执行内容是配置 hypervisor 的 crontab，定时运行 reroute 脚本
8. 把 reroute 公私钥上传到 hypervisor

清除流程如下：

1. 清除 hypervisor 上正在运行 reroute 脚本的进程
2. 删除 hypervisor 上脚本以及配置文件所在的文件夹
3. 清除 hypervisor 上 crontab 定时任务以及清除开启启动
4. 清除 hypervisor 上 33.2 对应的路由
5. 清除 scvm 的 /root/.ssh/authorized_keys

ZBS 版本升级后的检查流程：

1. 检查 hypervisor 上 reroute 的版本号，如果与当前 ZBS 版本一致，直接结束
2. 生成当前版本的 reroute 脚本并上传到 hypervisor 覆盖旧脚本
3. 清除 hypervisor 上正在运行 reroute 脚本的进程，目的是为了让 crontab 运行新脚本

### 测试流程

1. 以下在 ESXI 6.7 18.80、18.82、19.35 三节点上测试（集群没有 rdma 特性）：
    1. 在所有 SCVM 上执行 zbs-deploy-manage clear-hypervisor，成功清除各 ESXi 上已有的 reroute script、crontab 和清除开机启动 reroute 脚本
    2. 在所有 SCVM 上执行 zbs-node collect_node_info
    3. 在其中一台 SCVM 上执行 zbs-deploy-manage deploy-hypervisor --gen_ssh_key
    4. 在其他 SCVM 执行 zbs-deploy-manage deploy-hypervisor
    5. 在所有 ESXI 上的 /var/log/scvm_failure.log 如果每隔 3s 就会有新内容时，此时查看路由表可以发现 192.168.33.2 指向本地 SCVM 的存储网
    6. 将其中任意一台 SCVM 的存储网断开，其所在的 ESXi 在一段时间内（粗略计算是 30s 内）将 33.2 重路由到其他可用的地址（这种情况下是其他 SCVM 的存储网），此时将这台 SCVM 的存储网恢复回来，发现在 3 分钟内，33.2 的路由恢复到本地存储网。
    7. 将其中任意一台 ESXi 的存储网断开，其所在的 ESXi 在一段时间内（粗略计算是 30s 内）将 33.2 重路由到其他可用的地址（这种情况下是 SCVM 的管理网），此时将这台 ESXi 的存储网恢复回来，发现在若干分钟内（没有记录），33.2 的路由恢复到本地 SCVM 的存储网。
2. 验证了当在任意一台 scvm 上执行 zbs-deploy-manage update_reroute_version 时，如果集群中任意一台 ESXI 上的 scvm_failure_loop.sh 脚本中的第二行 reroute_version 与 scvm 中的 deploy_version.py 指定的 reroute_version 不一致时，scvm 都会把最新的 change_route.sh 和 scvm_failure_loop.sh 更新到对应的 ESXi，如果版本一致，不做任何操作。
3. 验证了 ESXI 6.5/7.0 可以通过 wget 获取 zbs_session_list 和 zbs_local_session_id，但没有做进一步的安装部署和升级。

### 过程疑问

1. 在 ESXi 上会同时存在多个 scvm_failure_loop.sh 的进程。因为 timeout 是异步的，crontab 每一分钟执行一次，理想情况下把 timeout 去掉同一时间的进程数应该 <=2 个，待验证。在 SSH 连接成功，但命令执行过程中如果发生了网络异常。SSH 命令可能需要分钟级别才能返回，进而影响路由恢复时间，所以引入 timeout。

2. ESXi 上的 scp 出现 invalid argument/ file too large，但是由于 ESXi 是闭源的，无法查看源码排查原因。网上检索说是内存不足，但是  82 机器有 50 多个G，目前原因还是未知。不应该让这种事阻塞开发进度，应该早点绕过它，比如换一台 ESXi 之类

3. 由于 esxi 和 scvm 没有一一正确关联，导致在 sn miss ，无法收集到对应 esxi 的信息，这个应该早点请教 tuna 组同事，卡了两天不值得

4. vmware 自带的 wget 无法指定 PUT 方法，所以无法把所有的命令行都换成 http 请求

5. IO reroute 首次部署没有出现“每台 ESXi 上都有被路由重定向到本地 SCVM 的存储网上的 33.2 的地址“。

    现象：在 yiwu-esxi-cluster 上看，各节点之间可以 ping 通，但是只有 18.80 能 ssh 其他两个节点，但是其他两个节点无法 ssh 任何节点，连到自己上面开的 SCVM 也不行。原因是：

    * 没给所有的 ESXi 开 ssh 客户端和 ssh 服务器的连接选项。造成 ESXi 之间无法 ping 通
    * ESXi 的存储网和 SCVM 的存储网不在一个网段上。后来把 ESXi 上的79 改成 73
    * 一开始配 ESXi 网络有问题，正确配置如下：
        * 管理网络 Management 一个虚拟交换机，配在 1G 物理网卡上
        * 存储网络 ZBS 一个虚拟交换机，配在 10G 物理网卡上
        * 重定向网络 NFS 一个虚拟交换机，不接物理网卡

### 概念补充

**基本概念**

* VAAI 是 VMware 开放的一组 API，用于在 ESXi 和存储后端之间进行通信，vSphere Client/Center 是对 ESXi 机器的 Web 管理软件
* Hypervisor ，即虚拟机管理器，目前主流的包括 VMware vSphere/ESXi、Citrix Xenserver，Smartx  ELF-KVM，像 ESXi 就是一个阉割版的 linux，是VMware 虚拟化管理器，接管物理节点上的 CPU 等硬件资源，不带有 linux 的全部功能。
* Tuna 负责部署 Smartx 集群部署，其上运行的 MongoDB 是一个用 C++ 编写的基于分布式 NoSQL 数据库，存有一些部署信息。

**IO Reroute 背景**

因为 NFS 协议中不包含重定向的功能，只能直接访问在挂载 NFS 时指定的 IP 地址，所以在 ESXi 节点上部署 IO Reroute 服务，在 ESXi 的网络层（L3）直接修改数据流向实现 NFS 链路（L4+）的重定向，能够实现集群 IO 高可用和 IO 负载分担。

**IO Reroute 定义**

IO Reroute 是 ZBS NFS 重定向服务，为 ESXi 在集群中提供一个高可用的 SCVM Access 接入点。

SCVM 是运行在 VMware ESXi 上的一个装有 SMTX OS 的虚拟机，其中运行着 ZBS，并对外提供 NFS 访问接口。通过 PCI 设备直通的方式将磁盘控制器透传给 SCVM，从而用 SCVM 中的 ZBS 统一管理所有物理节点上的磁盘，包括 SSD 和 HDD，提高了对物理磁盘的访问速度。同时，每一个 SCVM 又对外提供一个监听地址在私有网络 192.168.33.2 上的 NFS 存储服务，VMware ESXi 通过这个地址访问到 NFS 服务，并创建数据存储，供其他的虚拟机使用。

在每个 ESXi 节点上都构建一个仅本地可见的私有网络（192.168.33.x/24 网段），每个 SCVM 都持有私有网络中的相同 IP 地址（192.168.33.2），因为这个网络是节点内部的私有网络，节点与节点之间并不联通，所以没有 IP 冲突的问题。每个 ESXi 均以 192.168.33.2 作为 NFS 服务端地址挂载 ZBS 提供的 NFS Export，在 vSphere 集群看来，整个集群每个节点都挂载了来自相同源（挂载点） 的 NFS 服务，而实际上每个 ESXi 所连接的均是各自本地的 SCVM 中的 Access 服务。

具体而言，IO Reroute 服务会定期与 Meta 交互获取集群中所有可用的 Access 状态，并且检测它们在本地 ESXi 节点上是否可以 ping 通，在本地中通过指定通往 192.168.33.2 的静态路由来调整 NFS 链路的下一跳，确保 NFS 链路始终指向一个可用的 Access，在选取链路时，会优先选择本地 SCVM 的 Access。Access Server 提供 ZBS 的分布式数据接入服务，负责将各种存储协议如 NFS 转化为 ZBS 内部协议，完成数据 IO，Access Server 目前集成在 Chunk Server 内部，没有独立的实体。

**热迁移**

VM 在不同 ESXi 上迁移时，因为单个 ESXi 内 NFS 链路是共享的，所以 VM 的迁移并不会影响到 NFS 链路的状态。整个迁移过程对数据链路没有影响。在迁移完成之后，VM 的 IO 请求将由目标端 ESXi 上的 NFS Client 通过节点原有的 NFS 线路传递给本地 Access。

**异常重定向**

在 Access / SCVM 异常时的数据链路变化如下所示：

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211013341.png" alt="image-20220821101357310" style="zoom:50%;" />

1. IO Reroute 通过周期性的检查发现当前正在使用的 Access 不在 Meta 提供的健康 Access 列表中或者当前 ESXi 节点无法正常 Ping 通当前 Access 的 IP 地址，则触发 Reroute 动作。在 Meta 提供的健康 Access 列表中随机选择一个本地可以正常 Ping 通的 Access ，修改 ESXi 中的路由规则，将 192.168.33.2 指向该 Access 。ESXi 在转发指向 192.168.33.2 的数据流时，路由规则生效，所有的流量被引导向目标 Access，由目标 Access 提供 NFS 服务；
2. 在本地 SCVM 上的 Access 服务恢复正常之后，IO Reroute 监控到本地 Access 连续处于健康状态超过指定时间（默认 3 分钟），则重新调整 ESXi 的路由规则，将 192.168.33.2 指向本地 Access ，数据恢复本地访问；

**SmartX OS + VMware 安装部署**

1. 通过 IPMI 连接到对应的物理机，并挂载 EXSi 镜像到物理机上，通过该节点的 IPMI 网址可以远程登录该机器并进行 ESXi system install，系统安装之后用部署在 71.255 上的 vSphere Client 来管理；

2. 在每台 EXSi 上创建一个搭载 SmartX OS 的虚拟机 SCVM，通过 Web 访问虚拟机 IP 进入集群部署界面，为在不同 ESXi 上的 SCVMs 搭建一个超融合集群；

3. 部署完成后通过 Fisheye 集中管理平台管理集群和虚拟机。

4. 通过部署在 71.255 上的 vSphere Client 来管理刚刚部署好的机器

SmartX Halo部署架构图如下：

![image-20220821101417687](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211014730.png)

IP 地址规划：

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211014908.png" alt="image-20220821101431874" style="zoom:40%;" />

每台 ESXi 上有 2 张物理网卡，3 张虚拟网卡，3 台虚拟交换机

* Mgt vSwitch 与1G 物理网卡相连，ESXI 的 mgt 虚拟网卡通过 mgt vSwitch 与 SCVM 的 mgt 虚拟网卡相连。
* zbs vSwitch 与10G 物理网卡相连，ESXI 的 zbs 虚拟网卡通过 zbs vSwitch 与 SCVM 的 zbs 虚拟网卡相连。
* Nfs vSwitch 不与物理网卡相连，ESXI 的 nfs 虚拟网卡通过 nfs vSwitch 与 SCVM 的 nfs 虚拟网卡相连。

通过存储物理交换机来连通 ESXi 之间的存储网，通过管理物理交换机来连通 ESXi 之间的管理网



NFS 使用情况

超融合的（HCI）模式下，ESXi 使用 NFS 访问 ZBS，每个虚拟机对应一个目录，每个虚拟卷对应关联目录下的一组 NFS 文件，一个 ESXi 节点上的所有 VM 虚拟卷会共享同一个 NFS 数据连接。

