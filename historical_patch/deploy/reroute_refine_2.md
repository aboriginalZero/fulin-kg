五一期间 ESXi 升级后 SCVM 所在主机的 IO 重路由服务停止工作

8 个节点有 2 个出现问题，看 ESXi 的日志发现这两个节点在 4 月中旬做 scvm 的升级的时候 io reroute 就有问题了，scvm 升级之后，在 io reroute 升级时，scvm 会通过 ssh 的方式在每台 ESXi 上执行多个 cli 用以杀死旧进程，更换最新的 reroute 脚本，使用新的 reroute 文件并更新 crontab。

这个过程中需要获取 reroute 脚本所在的 datastore，这个 ssh 的过程中如果密码/密钥对不上或者网络 Timeout 时会将 datastore_path 认为是 None，然后直接用 None 去拼接路径字符串，填到 crontab 里，导致有问题。

这里没有做好 ssh 异常的处理，是 6 年前就有的问题了，不过现在才暴露，薛总说是这个客户机器物理环境很差，机房非常热，可能是导致 ssh timeout 的原因。

SCVM 升级之后，在 IO Reroute 升级时，SCVM 会通过 ssh 的方式在每台 ESXi 上执行多个 cli 用以杀死 reroute 旧进程，更换新版本 reroute 脚本，更新 crontab 中新 reroute 脚本位置并等待唤起新进程。

当 SCVM ssh ESXi Timeout 时，由于没有正确处理异常，会将 datastore_path 认为是 None，路径被拼接成一个错误的 "/vmfs/volumes/None/vmware_scvm_failure/reroute.py" 

crontab 没能正确找到 reroute 脚本所在位置，reroute 进程没起，引发 Tower 报警。





整理一下 xen io reroute 中 meta leader 被 kill 的售后处理的流程，zk leader kill 包含 zk session、access session、db cluster 相关的内容。meta in zbs 中关于 db cluster 部分，DBCluster是一个通用的组件，用于各个节点间进行数据的同步。在有数据修改时，DBCluster会首先将journal提交到journal cluster（目前基于zookeeper实现），当提交到journal cluster完成后，数据修改就可以返回了，journal cluster保证修改的持久性，本地的LevelDb会异步的被修改。



shell io reroute，一开始 scvm 存储网都 down 的情况下，切到管理网，此时恢复其中一个 scvm 存储网，但就一个本地的 xen 的路由切回存储网，这是符合预期的，因为这个管理网的 session alive sec 正常，shell 版本中不会去选别的存储网。

smtxos 4.1.0 在 scvm 的 sshd 配置 /etc/ssh/sshd_config 中 ssh PermitRootLogin 默认为 yes，升级到 5.1.2 后会改为 no，这导致旧版本 ioreroute ssh scvm 失败，无法获取 session 信息

一个是 tuna 那报的一个问题，从 5.0.3 升级到 5.0.7，有个节点 IO 重路由状态检查失败，上去看了下，有个现象是会删除本地存储 ip，然后又把他添加回来，看了下应该是这个版本里对 session alive 的判断逻辑有问题，当时没有 session alive 字段，他是自己写的一套判断逻辑，应该是有点 bug，还没来得及继续调查，还会出 5.0.8 吗？还需要更细致的调查吗？不会出，不接着调查。



执行以下操作前，需要保证所有 ESXi 上的 192.168.33.2 的下一跳指向本地 SCVM 存储 IP：

在所有 SCVM 节点上执行：

1. 进入 /usr/share/tuna/script/scvm_failure_common 目录，在后面的操作中，不要切换目录。 
2. 备份已有 io reroute 脚本。命令：mv reroute.py reroute.py.bak
3. 将新的 reroute.py 脚本复制到同级目录；
4. 检查新的 reroute.py 脚本 md5sum = 60b0c13cd5680afb4f71c65a4785a07f
5. 将同级目录中的 rereoute_version 文件中记录的版本号从 2.2 改成 2.2.1；

在任一 SCVM 节点上执行：

1. zbs-deploy-manage update_reroute_version



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



替换所有 esxi 节点 reroute 脚本的方式

1. 任选一个 scvm 节点，修改 /usr/share/tuna/script/scvm_failure_common/reroute.py 和 reroute_version
2. 执行 zbs-deploy-manage update_reroute_version
3. 如果 SCVM 未开启 / 网络不稳定 / 开启防火墙，该命令会返回一个错误的 volume path

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



reroute 2.2.1 之后，因为 ping 不通而切换路由的可能原因

1. esxi 的 dmsg -T 里会显示是否跟其他 ip 冲突；

   ```
   arp: 00:50:56:6d:f4:6e is using my IP address 10.20.127.137 on vmk2
   ```

2. 没有开启防火墙的 ssh 服务。

   ```
   # 开启
   esxcli network firewall ruleset set --ruleset-id=sshClient --enabled=true
   esxcli network firewall refresh
   
   # 检查是否生效
   esxcli network firewall ruleset list | grep ssh 
   ```





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



1. 怎么看在哪里调用了 pyzbs 中的 update_reroute_version 函数？这个是对外提供的命令，代码中如果搜不到的话，可以在文档中搜索，http://docs.fev.smartx.com/

2. 怎么看 /api/v3/sessions 对应的 pyzbs 中的方法？可以在 https://newgh.smartx.com/cluster-platform/harbor/blob/master/swagger/yaml/crab.yaml

3. http://remote_ip/api/v2/zbs_session/session/session_id 是用于更新 session 的做法，具体实现是 zbs 中的  SessionMaster::RefreshLocalNFSClientIPs()，

4. xen 平台还有人在使用吗？xen 要保证可以工作

5. reroute.py 中提供了 LOOP 和 ROUTE 模式，后者应该是用于人工指定一个 target i，相当于一个智能模式，一个静态模式，对应原来的 change_route_route.sh

6. shlex.split(cmd) 是用来当 shell=false 时，把 shell cmd 进行分割，期望传入一个数组，比如 ["ls", "-a"]。

7. 从 shell 切 python，reroute version 从 1.4 变成 1.5， 1.6 变成 2.1

   1. 1.4 版本开始把 zbs cli 都替换成 restful 的形式，并设有 timeout 机制
   2. 1.5 版本开始使用 reroute.py，1.5 到 1.6 之间修了一些小 bug，所以对外形式可以理解成，1.5 开始一定是用的 reroute.py
   3. smtx 5.x 开始用的 reroute 脚本都是 2.x，这样 4.x 再打 patch 的话可以从 1.6 开始接着用，另外，2.1 到 2.2 变化并不大

8. 为啥要对 target_ip send heartbeat？

   为了让 insight 可以感知到 reroute 状态，if reroute other host，insight 是 zbs 中的模块，用以感知探测 + 报警 zbs 状态。

9. ping 发 icmp 包为啥要自实现一个 ICMPHandler？

   客户的 vmware 环境里 scvm vNIC 开启了 MTU check，网络包大小如果超过了 MTU 值会被 drop 掉

10. 日志在 /scratch/log/scvm_failure.log，为什么还要软链接一份到 /var/log/scvm_failure.log，   check_lock_and_log_file() 

11. 为啥有了 session 还要用 ping 来判断？

    When SCVM is down, meta leader will need some time to detect the session lost, and reroute should not rely on it since IO may hang in this period. 

12. 什么时候应该 refresh_local_hypervisor_ip，在 shell 版本是在 refresh_local_hypervisor_ip 成功之后，才会去 del_route/add_route，但在 python 版本中并没有，怀疑可能会影响到 NFS 接入点的唯一性保证。

    通过 lease owner 也可以保证接入点的唯一性，所以在 NFS 接入点这边没有处理也不会导致数据污染？

13. back to normal 限制要 180s ，是否可以减少？





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
