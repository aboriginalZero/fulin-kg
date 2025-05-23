### RDMA 信息

开启 RDMA 之后 33.2 和对应的存储 IP 预期都是要指向接入 IP

```
[root@87-21:~] esxcfg-route -l
VMkernel Routes:
Network          Netmask          Gateway          Interface
10.0.85.21       255.255.255.255  10.0.85.121      vmk1
192.168.33.2     255.255.255.255  10.0.85.121      vmk1
10.0.85.0        255.255.255.0    Local Subnet     vmk1
192.168.1.0      255.255.255.0    Local Subnet     vmk3
192.168.33.0     255.255.255.0    Local Subnet     vmk2
192.168.80.0     255.255.240.0    Local Subnet     vmk0
default          0.0.0.0          192.168.80.1     vmk0
[root@87-21:~] cat /vmfs/volumes/5fd798a7-2fdf1814-fbc6-b8cb299f10d1/vmware_scvm_failure/scvm_ip
local_scvm_data_ip=10.0.85.21
active_scvm_data_ip=10.0.85.23,10.0.85.22,10.0.85.21
local_scvm_manage_ip=192.168.85.21
active_scvm_manage_ip=192.168.85.23,192.168.85.22,192.168.85.21
local_scvm_vmware_access_ip=10.0.85.121
local_scvm_nfs_ip=192.168.33.2
active_remote_zone_scvm_data_ip=
active_remote_zone_scvm_manage_ip=
```

要手动切换时，只需要填存储 IP，代码里转了到接入 IP 的转换。

### 测试替换 reroute 脚本

执行以下操作前，需要保证所有 ESXi 上的 192.168.33.2 的下一跳指向本地 SCVM 存储 IP：

在所有 SCVM 节点上执行：

1. 进入 /usr/share/tuna/script/scvm_failure_common 目录，在后面的操作中，不要切换目录。 
2. 备份已有 io reroute 脚本。命令：mv reroute.py reroute.py.bak
3. 将新的 reroute.py 脚本复制到同级目录；
4. 检查新的 reroute.py 脚本 md5sum = 60b0c13cd5680afb4f71c65a4785a07f
5. 将同级目录中的 rereoute_version 文件中记录的版本号从 2.2 改成 2.2.1；

在任一 SCVM 节点上执行：

1. zbs-deploy-manage update_reroute_version

### esxi 常用命令

scvm 模拟丢包

```
iptables -A INPUT -p icmp --icmp-type echo-request -s 10.0.11.0/24 -j DROP
iptables -D INPUT -p icmp --icmp-type echo-request -s 10.0.11.0/24 -j DROP
```

esxi 启停网口

```
esxcli network ip interface set --interface-name=vmk2 --enabled=true
esxcli network ip interface set --interface-name=vmk2 --enabled=false
esxcli network ip interface list
```

删除/添加路由

```
esxcfg-route -d 192.168.33.2/32 10.0.0.22;
esxcfg-route -a 192.168.33.2/32 10.0.0.21;
```

删除进程

```
xen 中删除 reroute 进程
ps -ef | grep scvm_failure_loop.sh  |grep -v grep | grep -v vi | awk '{print $2}' | xargs /bin/kill
esxi 中删除 reroute 进程
ps -c | grep scvm_failure_loop.sh | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill
ps -c | grep reroute.py | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill
```

ESXi 常用指令

```
esxcfg-route -l 	# 查看路由表
esxcfg-vmknic -l	# 查看网卡（本地 IP）
esxcli vm process list	# 查看虚拟机
esxcli network nic up -n=vmnic2 	# 开启关闭指定物理网卡
esxcli network vm list # 获取虚拟机网卡 world_id
esxcli network vm port list -w <world_id> # 获取各虚拟网卡的端口
pktcap-uw --switchport <switch_port> -o 1.pcap # 抓取从该 vnic 发出的包
pktcap-uw --switchport <switch_port> --capture PortOutput -o 2.pcap # 发往该 vnic 的包
pktcap-uw --uplink vmnic0 --ip 192.168.18.82 --capture PortInput -o 55.pcap # 抓取物理网卡上指定 IP 的包
```

reroute 相关抓包

```shell
# esxi 上抓包
tcpdump-uw -i vmk1 "tcp[tcpflags] & (tcp-ack|tcp-syn|tcp-fin) != 0" and host 10.0.0.21 -nnn -w /tmp/esxi1-yiwu.pcap
# 抓存储网卡上的 syn 和 rst 包，端口 2049 是 nfs server 监听端口
tcpdump -i ens224 "tcp[tcpflags] & (tcp-syn|tcp-rst) != 0" and port 2049 -w /tmp/scvm111-yiwu.pcap
# 读取抓包内容
tcpdump -r /tmp/scvm111-yiwu.pcap
# scvm 查看端口连接
netstat -anl | grep 2049
# esxi 查看端口连接
esxcli network ip connection list | grep 2049
# 在 scvm 上用丢 ping 包的方式模拟切出切回
while true; do iptables -A INPUT -i ens224 -p icmp --icmp-type echo-request -j DROP; sleep 4; iptables -D INPUT -i ens224 -p icmp --icmp-type echo-request -j DROP; sleep 60; done
# 查看 iptables 配置
iptables -L 
```

tcmpdump 抓包，-n 显示 ip 地址，而不是 hostname，-nn 显示端口号而不是协议名，过滤源和目的IP地址以及特定协议

```shell
tcpdump -i eth0 src host 192.168.1.2 and src port 8080 and dst host 192.168.1.3 and dst port 9090 and not icmp
```

wireshark 过滤 tcp 指定标志位的数据报，syn / ack / fin 同理

```
tcp.flags.reset == 1
```

wireshark 关闭相对 TCP 序号，在任一抓包上单点右键，选择协议首选项，然后在弹出菜单中出去 relative seq 前的勾选去掉 relative seq num

修改内核参数

```shell
echo "4" > /proc/sys/net/ipv4/tcp_orphan_retries 
# 在 /etc/sysctl.conf 中修改 net.ipv4.tcp_orphan_retries=4
sysctl -a | grep tcp_orphan_retries 查看配置 
```

### insight 行为

io reroute 多久没给 insight 心跳，他就会报警

判断 IO reroute 不工作的方式是没有按一定频率跟 insight 心跳，如果超过 n 次没有跟 insight 心跳，主动退出程序？

zbs-insight 每收到一次日志有可能打印一下吗？zbs-insight 如果没有收到心跳或者跟上一次收到的不一样，打印一下



insight metric

1. zbs_ioreroute_cluster_total_num 表示集群中的 scvm 个数，不论是否健康，只关注 SCVM 是否在集群中；
2. zbs_ioreroute_cluster_connected_num 表示 reroute 还在工作的 esxi 个数（还在工作指的是 reroute 还能跟 insight 发送心跳，33.2 不一定指向本地）；
3. zbs_ioreroute_cluster_normal_num 表示 33.2 指向 local scvm data ip 的 esxi 个数，一定小于等于 zbs_ioreroute_cluster_connected_num。

不过这些 metric 目前没有被用于报警，报警用的是 zbs_ioreroute_server_status，它的大小等于 zbs_ioreroute_cluster_total_num，初始值是 1，如果他的 reroute 还在工作且指向本地，那么值是 0，如果还在工作但没有指向本地，那么值是 2。zbs_ioreroute_server_status 中存在值为 1 或 2 的 item，tower 上就会有报警。

升级中心的判断依据是 zbs_ioreroute_cluster_connected_num < 存储节点个数（比如仲裁节点上就没有 chunk，不算是个存储节点）

查看 insight metric（insight leader data ip）

```
[root@node130-71 22:30:46 ~]$ curl http://10.0.130.71:10700/api/v1/prometheus/insight
# HELP zbs_ioreroute_server_status the status of ioreroute server, 0: ioreroute is normal which means hypervisor route to data ip of local scvm, 1: ioreroute not working, 2: ioreroute is working but not normal
# TYPE zbs_ioreroute_server_status gauge
zbs_ioreroute_server_status{_data_ip="10.0.130.254"} 1.000000
zbs_ioreroute_server_status{_data_ip="10.0.130.71"} 1.000000
zbs_ioreroute_server_status{_data_ip="10.0.130.73"} 1.000000
zbs_ioreroute_server_status{_data_ip="10.0.130.72"} 1.000000
zbs_ioreroute_server_status{_data_ip="10.0.130.74"} 1.000000
# HELP zbs_ioreroute_cluster_normal_num the num of ioreroute servers which is normal, normal means hypervisor route to data ip of local scvm
# TYPE zbs_ioreroute_cluster_normal_num gauge
zbs_ioreroute_cluster_normal_num{_uuid="4ab5ad8e-bb19-4e89-ab00-cb541d20da50"} 0.000000
# HELP zbs_ioreroute_cluster_connected_num the num of ioreroute servers with heartbeat
# TYPE zbs_ioreroute_cluster_connected_num gauge
zbs_ioreroute_cluster_connected_num{_uuid="4ab5ad8e-bb19-4e89-ab00-cb541d20da50"} 0.000000
# HELP zbs_ioreroute_cluster_total_num total num of ioreroute servers
# TYPE zbs_ioreroute_cluster_total_num gauge
zbs_ioreroute_cluster_total_num{_uuid="4ab5ad8e-bb19-4e89-ab00-cb541d20da50"} 5.000000
```

### reroute 常用命令

1. 升级 reroute，执行的是 zbs-deploy-manage update_reroute_version，这个命令不会清理路由，所以没有这个问题。不过这里需要补充说明下，这个命令为了保持灵活性，是版本不匹配就做替换，比如 esxi 上已经是 2.2.1，scvm 里是 2.2，那么执行这个命令后版本是会回退到 2.2，不过预期从低到高升级，所以还好；
2. 清理 reroute，执行的是 zbs-deploy-manage clear-hypervisor，这个命令会清理路由，如果没有及时配上（30s 内），会有 NFS IO Retry；
3. 部署 reroute，执行的是 zbs-deploy-manage deploy-hypervisor，这个命令会拼接生成 reroute 脚本（把版本号填到 reroute.py 的第二行）、重置 contrab 配置、杀掉正在运行的 reroute 进程、等待 crontab 拉起新的 reroute 进程。

deploy-hypervisor && clear-hypervisor 这对命令预期只在新部署的机器上执行，节点还没投入使用，还没有业务 IO。

扩容节点部署 reroute 失败，应该只在新扩容节点上执行 clear-hypervisor && deploy-hypervisor。已经投入使用的节点，不建议执行 clear-hypervisor，只建议用 update_reroute_version。

### esxi 日志

1. /var/log/vmkernel.log 内核日志，用以查看路由设置情况、NFS IO 是否重试

2. /vmfs/volumes/66a0ba6a-6d9f4d64-ca9e-005056abb843/yiwu-fio-test/vmware.log 虚拟机日志， 用以查看虚拟机活动日志

3. /var/log/shell.log  shell 日志，记录已执行过的所有的 shell 命令

4. /var/log/auth.log 身份验证，用以查看 ssh 相关的日志是否执行，比如是否部署/清理 reroute 

5. /var/log/vmkwarning.log，VMKernel 警告，记录了虚拟机有关的活动

6. /var/log/syslog.log 消息信息，用以查看 crontab 定时任务是否执行



### 已知问题

1. shell io reroute，一开始 scvm 存储网都 down 的情况下，切到管理网，此时恢复其中一个 scvm 存储网，但就一个本地的 xen 的路由切回存储网，这是符合预期的，因为这个管理网的 session alive sec 正常，shell 版本中不会去选别的存储网。
2. smtxos 4.1.0 在 scvm 的 sshd 配置 /etc/ssh/sshd_config 中 ssh PermitRootLogin 默认为 yes，升级到 5.1.2 后会改为 no，这导致旧版本 ioreroute ssh scvm 失败，无法获取 session 信息
3. 一个是 tuna 那报的一个问题，从 5.0.3 升级到 5.0.7，有个节点 IO 重路由状态检查失败，上去看了下，有个现象是会删除本地存储 ip，然后又把他添加回来，看了下应该是这个版本里对 session alive 的判断逻辑有问题，当时没有 session alive 字段，他是自己写的一套判断逻辑，应该是有点 bug，还没来得及继续调查，还会出 5.0.8 吗？还需要更细致的调查吗？不会出，不接着调查。
4. 当 SCVM ssh ESXi Timeout 时，由于没有正确处理异常，会将 datastore_path 认为是 None，路径被拼接成一个错误的 "/vmfs/volumes/None/vmware_scvm_failure/reroute.py" ，crontab 没能正确找到 reroute 脚本所在位置，reroute 进程没起，引发 Tower 报警。
5. 如果 SCVM 未开启 / 网络不稳定  开启防火墙，该命令会返回一个错误的 volume path
6. SMTX OS 升级到 5.0.6 之后可以将 APD 统一设置为 1 。在 5.0.5 及之前的版本需要保持 0 ，补充 VMware 官方针对 [APD 介绍](https://docs.vmware.com/cn/VMware-vSphere/7.0/com.vmware.vsphere.storage.doc/GUID-54F4B360-F32D-41B0-BDA8-AEBE9F01AC72.html)如下：
    - ESXi 主机上的存储全部路径异常 (APD) 处理功能为 1 时处于开启状态。如果启用了该功能，主机会在有限的一段时间内持续向处于 APD 状态的存储设备重试非虚拟机 I/O 命令。时限到期后，主机将停止重试尝试，并终止所有非虚拟机 I/O。
    - 如果禁用了 APD 处理功能也就是设置为 0 时，主机将无限期持续重试发出命令，尝试重新连接到 APD 设备。该行为可能导致主机上的虚拟机超过其内部 I/O 超时值而无响应或发生故障。主机可能与 vCenter Server 断开连接。



如果在同一个网络平面中，由于大流量的 I/O 切换而导致某些数据包在传输过程中出现延迟、丢失或重新排序，TCP 协议可能会根据内部的超时定时器或接收方的 ACK 触发重传。但是，通常情况下，网络设备的内部处理（例如内部的缓冲、队列、调度机制等）不会触发 TCP 的重传机制。总的来说，TCP 协议更关注于网络的端到端通信质量，对于网络设备内部的 I/O 切换并不敏感。如果这种切换并未导致数据包在传输路径中出现丢失或延迟，TCP 通常不会主动触发重传机制。

网络平面切换可能导致短暂的丢包或延迟，主要是因为切换过程中的一些处理步骤可能会引起数据包的暂时丢失或延迟传输：

1. **处理延迟**：在网络设备进行平面切换时，可能涉及到数据包从一个处理单元切换到另一个处理单元的过程。在这个切换过程中，数据包可能需要经过额外的处理步骤，这可能导致一些短暂的处理延迟；
2. **内部队列处理**：在网络设备中，数据包通常会经过内部的队列或缓冲区进行处理。在平面切换期间，由于切换路径的改变，数据包可能需要等待进入新的队列或缓冲区，这可能会导致一些暂时的延迟；
3. **路由表更新**：网络设备在进行平面切换时可能需要更新路由表或其他相关的网络状态信息。这个更新过程可能会导致一些数据包在切换期间暂时找不到正确的路由路径，从而产生短暂的丢包或延迟；
4. **上下文切换**：在切换不同的处理单元或处理逻辑时，可能涉及到操作系统级别的上下文切换。这种切换可能会带来一些处理延迟和资源重新分配，可能会影响数据包的传输速度和时序
5. **设备状态同步**：在切换过程中，可能涉及设备状态的同步或数据同步的过程，这可能会导致某些数据包在切换时暂时无法正确处理

```
exsi
192.168.85.11 12 13

scvm
192.168.85.15 16 17
10.10.85.15 16 17
```



1. 怎么看 /api/v3/sessions 对应的 pyzbs 中的方法？可以在 https://newgh.smartx.com/cluster-platform/harbor/blob/master/swagger/yaml/crab.yaml

2. http://remote_ip/api/v2/zbs_session/session/session_id 是用于更新 session 的做法，具体实现是 zbs 中的  SessionMaster::RefreshLocalNFSClientIPs()，

3. xen 平台还有人在使用吗？xen 要保证可以工作

4. reroute.py 中提供了 LOOP 和 ROUTE 模式，后者应该是用于人工指定一个 target i，相当于一个智能模式，一个静态模式，对应原来的 change_route_route.sh

5. shlex.split(cmd) 是用来当 shell=false 时，把 shell cmd 进行分割，期望传入一个数组，比如 ["ls", "-a"]。

6. 从 shell 切 python，reroute version 从 1.4 变成 1.5， 1.6 变成 2.1

   1. 1.4 版本开始把 zbs cli 都替换成 restful 的形式，并设有 timeout 机制
   2. 1.5 版本开始使用 reroute.py，1.5 到 1.6 之间修了一些小 bug，所以对外形式可以理解成，1.5 开始一定是用的 reroute.py
   3. smtx 5.1.x 开始用的 reroute 脚本都是 2.x，这样 4.x 再打 patch 的话可以从 1.6 开始接着用，另外，2.1 到 2.2 变化并不大

7. ping 发 icmp 包为啥要自实现一个 ICMPHandler？

   客户的 vmware 环境里 scvm vNIC 开启了 MTU check，网络包大小如果超过了 MTU 值会被 drop 掉

8. 日志在 /scratch/log/scvm_failure.log，为什么还要软链接一份到 /var/log/scvm_failure.log，   check_lock_and_log_file() 

9. 为啥有了 session 还要用 ping 来判断？

    When SCVM is down, meta leader will need some time to detect the session lost, and reroute should not rely on it since IO may hang in this period. 

10. 什么时候应该 refresh_local_hypervisor_ip，在 shell 版本是在 refresh_local_hypervisor_ip 成功之后，才会去 del_route/add_route，但在 python 版本中并没有，怀疑可能会影响到 NFS 接入点的唯一性保证。

    通过 lease owner 也可以保证接入点的唯一性，所以在 NFS 接入点这边没有处理也不会导致数据污染？

11. back to normal 限制要 180s ，是否可以减少？可以但没必要，3 min 是一个大致的阈值

### shell 版本 reroute 手动切换路由

shell 版本 reroute 手动切换路由，用于已知的运维动作（比如 SCVM 节点下线修改内存值后重启）前的主动切换

1. 停止 crond

    ```
    # 待重启节点上
    # 在 /var/spool/cron/crontabs/root 文件中注释 scvm_failure_loop.sh 所在行
    cat /var/run/crond.pid | xargs /bin/kill;
    crond
    ```

2. 停止当前正在运行的 reroute；

    ```
    ps -c | grep scvm_failure_loop.sh | grep -v grep | grep -v vi | awk '{print $2}' | xargs /bin/kill -9
    ```

3. 执行  change_route.sh 到指定节点

    ```
    find / -name "*route.sh"
    sh /<path>/change_route.sh <no-local scvm data ip>
    # 查看路由表上 192.168.33.2 的下一跳是否是 <no-local scvm data ip>
    esxcfg-route -l
    ```

4. 重启 SCVM

5. 把 reroute 添加回 crontab 中

    ```
    # 解除步骤 1 中的注释
    cat /var/run/crond.pid | xargs /bin/kill;
    crond
    ```

    等待 1 min，观察 tail -f scvm_failure.log 有持续输出，单个节点操作完毕。

### xen io reroute 获取/更新 session 的方式从 ssh 改成 wget

xen io reroute 获取/更新 session 的方式从 ssh 改成 wget 的安装包，操作步骤：

1. 下载补丁包到各个 scvm 的 /tmp 目录，其 md5sum = 71c440126f7e6176b80326e06b555a7c

    ```shell
    # 检查是否存在 
    ls /tmp/scvm_failure_common.tar
    ```

2. 对各个 scvm 节点

    ```shell
    # 在所有 scvm 节点上，进入到此目录
    cd /usr/share/tuna/script
    
    # 在所有 scvm 节点上，将已有 io reroute 脚本备份
    rm -rf /home/smartx/scvm_failure_common.bak && mv scvm_failure_common /home/smartx/scvm_failure_common.bak
    
    # 在所有 scvm 节点上，解压到 /usr/share/tuna/script 目录
    tar -xvf /tmp/scvm_failure_common.tar -C .
    
    # 在所有 scvm 节点上，确保补丁包的 reroute version 是 1.5.1，执行后续操作
    cat scvm_failure_common/reroute_version
    
    # 在所有 scvm 节点上，清空已有 ioreroute 脚本及服务
    zbs-deploy-manage clear-hypervisor
    
    # 重新部署 ioreroute 脚本并启动服务
    # 任选一个 scvm 上节点执行
    zbs-deploy-manage deploy-hypervisor --gen_ssh_key
    # 在剩下的 N - 1 个 scvm 上节点执行
    zbs-deploy-manage deploy-hypervisor
    ```

3. 对各个 xen server 节点

    ```shell
    # 在所有的 xen server 节点上，观察 io reroute 输出日志
    tail -f /var/log/scvm_failure.log
    ```

    若出现 “the session of local scvm is not stable”，那么等待 3 分钟， 查看 192.168.33.2 的下一跳是否能指向本地 scvm 存储 ip，预期出现如下日志：

    ```
    Thu Jan 11 12:40:01 CST 2024 [+] local scvm active, try to route to local scvm
    Thu Jan 11 12:40:01 CST 2024 [+] try to delete route
    Thu Jan 11 12:40:01 CST 2024 [+] trying to del route 192.168.33.2 --> 10.10.130.219
    Thu Jan 11 12:40:01 CST 2024 [+] delete route success
    Thu Jan 11 12:40:01 CST 2024 [+] try add route:
    Thu Jan 11 12:40:01 CST 2024 [+] trying to add route from 192.168.33.2 to 10.10.130.218
    Thu Jan 11 12:40:01 CST 2024 [+] add route succss
    ```



整理一下 xen io reroute 中 meta leader 被 kill 的售后处理的流程，zk leader kill 包含 zk session、access session、db cluster 相关的内容。meta in zbs 中关于 db cluster 部分，DBCluster是一个通用的组件，用于各个节点间进行数据的同步。在有数据修改时，DBCluster会首先将journal提交到journal cluster（目前基于zookeeper实现），当提交到journal cluster完成后，数据修改就可以返回了，journal cluster保证修改的持久性，本地的LevelDb会异步的被修改。



### 异步 ping

Python 3.2 引入了concurrent.futures。 3.4版本引入了asyncio到标准库， python3.5以后使用async/await语法。



ESXi 6.7.0 build-14320388，Python 3.5.7

ESXi 7.0.3 build-19482537，Python 3.8.8

ESXi 8.0.1 build-21813344，Python 3.8.16



```
[root@ESXi13:/vmfs/volumes/6537862b-750b1f8f-cfb7-0cc47aa5d914/vmware_scvm_failure] cat /var/spool/cron/crontabs/root
#min hour day mon dow command
1    1    *   *   *   /sbin/tmpwatch.py
1    *    *   *   *   /sbin/auto-backup.sh ++group=host/vim/vmvisor/backup.sh
0    *    *   *   *   /usr/lib/vmware/vmksummary/log-heartbeat.py
*/5  *    *   *   *   /bin/hostd-probe.sh ++group=host/vim/vmvisor/hostd-probe/stats/sh
00   1    *   *   *   localcli storage core device purge
*/10 *    *   *   *   /bin/crx-cli gc
* * * * * /bin/sh /vmfs/volumes/6537862b-750b1f8f-cfb7-0cc47aa5d914/vmware_scvm_failure/scvm_failure_loop.sh &

改完记得 crontab reload
```

ip num 20，PING_SELECT_TIMEOUT_S = 0.01，PING_SELECT_COUNT = 100，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 4800 - 4900 ms，now = 2500 ms

ip num 20，PING_SELECT_TIMEOUT_S = 1，PING_SELECT_COUNT = 1，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 1000 - 2000 ms，now = 2000 ms

ip num 206，PING_SELECT_TIMEOUT_S = 1，PING_SELECT_COUNT = 1，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 1000 - 3000 ms，now = 400 ms

ip num 506，PING_SELECT_TIMEOUT_S = 1，PING_SELECT_COUNT = 1，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 1000 - 3000 ms，now = 3000 - 4000 ms



PING_THREAD_NUM =cpu_num() // 2，now

PING_SELECT_TIMEOUT_S/PING_SELECT_COUNT 用 1 1 的效果

* ip num 26， 50 ms
* ip num 206，400 ms
* ip num 506，1394 ms

PING_SELECT_TIMEOUT_S/PING_SELECT_COUNT 用 0.1 10 的效果

* ip num 26， 1500 ms
* ip num 206，xxx ms
* ip num 506，6000 7000 ms



11 的性能虽好，但是太敏感了，在性能差的嵌套集群上，会认为大部分 ip unreachable，导致路由频繁切换，还是选用 0.1 和 10



ip num 20，PING_SELECT_TIMEOUT_S = 0.1，PING_SELECT_COUNT = 10，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 2900 - 3100 ms，now = 1000 - 1300 ms
* PING_THREAD_NUM =cpu_num()，PING_TIMEOUT =  2，previous 1100 - 1400 ms，now = 1200 - 1400 ms
* PING_THREAD_NUM =cpu_num() * 2，PING_TIMEOUT =  2，previous 1100 - 1400 ms，now = 2000 - 2000 ms（timeout 的多）

ip num 206 ，PING_SELECT_TIMEOUT_S = 0.1，PING_SELECT_COUNT = 10，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 4，PING_TIMEOUT =  2，previous 29000 - 30000 ms，now = 800 - 2100 ms
* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 10000 - 11000 ms，now = 1200 - 1800 ms
* PING_THREAD_NUM =cpu_num()，PING_TIMEOUT =  2，previous 1700 - 2400 ms，now = 800 - 2400 ms
* PING_THREAD_NUM =cpu_num() * 2，PING_TIMEOUT =  2，previous 500 - 1400 ms，now = 900 - 1900 ms
* PING_THREAD_NUM =cpu_num() * 3，PING_TIMEOUT =  2，previous 600 - 1300 ms，now = 1600 - 2600 ms
* PING_THREAD_NUM =cpu_num() * 4，PING_TIMEOUT =  2，previous 850 - 2500 ms，now = 16400 - 2200 ms

ip num 506，PING_SELECT_TIMEOUT_S = 0.1，PING_SELECT_COUNT = 10，cpu num = 32，观察 10 组 ping 的结果

* PING_THREAD_NUM =cpu_num() // 2，PING_TIMEOUT =  2，previous 39000 - 39000 ms，now = 6900 - 8600 ms
* PING_THREAD_NUM =cpu_num()，PING_TIMEOUT =  2，previous 12000 - 13000 ms，now = 5600 - 10000 ms
* PING_THREAD_NUM =cpu_num() * 2，PING_TIMEOUT =  2，previous 1400 - 2100 ms，now = 5200 - 7700 ms



相比于同步做法，异步做法下，与线程数的关系是先增后降，且线程数越多，timeout 的概率越大，怀疑是线程调度影响



ip num 212，嵌套集群上

* PING_THREAD_NUM = cpu_num() * 2，PING_TIMEOUT =  2，previous 600 -  3931 ms，now = 800 ~ 4887 ms
* PING_THREAD_NUM =cpu_num()，PING_TIMEOUT =  2，previous 500 - 1800 ms，now = 
* cpu_size = cpu_num() * 2，when timeout of each ping is 2，previous  ms，now = 800 ~ 3400 ms
* 

origin timeout = 1, cost 47000

new time



在开发机上用 Python 3.5.6 和 Python 3.5.7 都可以正常使用 asyncio，但 esxi 上报错如下：

```
Traceback (most recent call last):
  File "yiwu_test.py", line 65, in <module>
    run(main)
  File "yiwu_test.py", line 60, in run
    loop.run_until_complete(main())
  File "/build/mts/release/bora-14320388/bora/build/esx/release/vmvisor/sys-boot/lib64/python3.5/asyncio/base_events.py", line 454, in run_until_complete
  File "/build/mts/release/bora-14320388/bora/build/esx/release/vmvisor/sys-boot/lib64/python3.5/asyncio/base_events.py", line 421, in run_forever
  File "/build/mts/release/bora-14320388/bora/build/esx/release/vmvisor/sys-boot/lib64/python3.5/asyncio/base_events.py", line 1389, in _run_once
  File "/build/mts/release/bora-14320388/bora/build/esx/release/vmvisor/sys-boot/lib64/python3.5/selectors.py", line 445, in select
OSError: [Errno 22] Invalid argument
```

还需要对比一下，原本的线程池做法跟用了异步的做法，有什么区别
