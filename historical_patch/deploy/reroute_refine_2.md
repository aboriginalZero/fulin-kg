

```
exsi
192.168.85.11 12 13

scvm
192.168.85.15 16 17
10.10.85.15 16 17
```



1. 去掉 shell 版代码；
2. 在每次 add route 之后，都主动调用一次 update_local_hypervisor_ip 去刷新 zbs 侧的 session 记录信息，以保证接入点的唯一性；
3. 试着重写一下 check_scvm_reroute_status，让路由切换优先级的定义更清晰，明确 session exist/a lie 的使用，考虑 chunk 会反复重启等特殊场景，减少不必要的路由切换。
4. 去掉 active_scvm_data_ip，只保留 active_scvm_data_ip_list



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

