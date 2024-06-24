## ZBS 常用 CLI

### 本地编译

```shell
# alias 命令
zcp zbs; zc zbs3

# 格式化 + 非首次编译
cd /home/code/zbs3 && ./script/format.sh && cd build && mold -run ninja zbs_test

# 首次编译，默认是 DEBUG 模式，程序运行速率会比较慢（比如影响 load pextent table 的时间）
cd /home/code/zbs3 && rm -rf build/ && mkdir build && cd build && cmake -DBUILD_MOLD_LINKER=ON -DCMAKE_BUILD_TYPE=RELWITHDEBINFO -G Ninja ..
mold -run ninja zbs_test zbs-metad

# 清空缓存，显示缓存统计信息
ccache -z; 	ccache -s;

# 开启多种编译特性
cmake3 -DCMAKE_BUILD_TYPE=Release -DUSE_SANITIZE=address -DBUILD_BENCHMARKS=ON -GNinja ..; 
# 运行单测
cd /home/code/zbs3/build/src && ./zbs_test --gtest_filter="*FunctionalTest.WriteResize*"  --gtest_repeat=100
zrt zbs3 FunctionalTest.WriteResize
```

### docker 使用

用 docker 能比较好解决要切换多个 zbs 版本混跑的问题，zbs-buildtime:oe1-x86_64

```shell
# 首次编译/子模块如 spdk 更新，需要删除 build 目录，进到 Docker 内部执行
docker run --rm --privileged=true -it -v /home/code/zbs3:/zbs -w /zbs registry.smtx.io/zbs/zbs-buildtime:el7-x86_64
mkdir build && cd build && source /opt/rh/devtoolset-7/enable && cmake -DBUILD_MOLD_LINKER=ON -DCMAKE_BUILD_TYPE=RELWITHDEBINFO -G Ninja ..
# 编译时给定参数，比如要同时编译 bench，cmake -DBUILD_BENCHMARKS=ON -G Ninja ..
# d-DBUILD_TARGET_PLATFORM=hygon 编译海光下的
ninja zbs_test

# 屏幕中会提示出错处的日志信息，借助 newci 可以避免在本地配置 nvmf/rdma 环境跑单测
# 但是要配好 nvmf/rdma 的相关依赖包/服务
cd /home/code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"

# 运行后的测试日志默认保存在 /var/log/zbs/zbs_test.xxx 中
cd /home/code/zbs/build/src && ./zbs_test --gtest_filter="*FunctionalTest.WriteResize*"

# 显示指定 main 分支
git review main

# 非首次编译，自动修改格式后再编译
docker run --rm --privileged=true -v /home/code/zbs:/zbs -w /zbs registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 sh -c 'sh ./script/format.sh && cd build && ninja zbs_test'

# 在主分支上
git pull
# 将新的 URL 复制到本地配置中
git submodule sync --recursive
# 从新 URL 更新子模块
git submodule update --init --recursive
git submodule update --init --force --recursive --remote

# pyzbs 即 zbs-rest-server 调试，
```

### zbs-client-py 调试

```shell
# make 命令都是在项目根目录执行

# 编译项目，生成 proto 文件，如果报错有重复的文件，及时删除（以下 2 个步骤都需要这么做）
make build

# 在 ./dist/ 目录下生成的 zbs_client_py_xxx.whl 文件，可以 scp 到测试集群并通过以下命令替换
/usr/local/venv/zbs-client-py/bin/pip install /tmp/zbs_client_py-xxx.whl

# 另一种方式是进入配套的容器，在这里面跑单测和测试嵌套集群的效果
make docker
docker run -it -v $PWD:/zbs-client-py registry.smtx.io/zbs-py-dev/zbs-client-py3/runtime:el7-x86_64
source scripts/init_build_env.sh
# 跑单测
./scripts/run-test.sh
# 安装整个项目，就能在容器里使用 zbs 命令，可以实时改动代码马上执行验证
pip install -e .
# 指定自己嵌套集群的 meta leader ip，存储 IP 或管理 IP 都可以
# 如果 zbs-chunk 的命令，填的 chunk ip 如果下线，请求就会失败
zbs-meta --meta_ip <meta_leader_mgt_ip> migrate get_mode_info
zbs-chunk --chunk_ip <chunk_mgt_ip> internal_io get 
```

pyzbs 原来是一个 monorepo，里面包含了很多模块，elf，network，tuna，deploy，他们之间的依赖关系非常的重，在 py2 -> py3 的升级过程中，将各个模块独立了 venv，现在 pyzbs 里面只有 zbs-rest-server，专门负责 zbs 相关的 http 接口，tuna 是 ops 相关的模块，里面有所有的硬件，部署，配置变更，集群变更等 API，有自己的 web server，早期整个 pyzbs 只有一个 web server，但是耦合太严重了，各个组件升级完全没办法独立运维。

如果要 zbs-chunk cli，把要验证的 chunk 节点上的 /etc/zbs/zbs.conf 复制到 docker 里（docker 本身没有 /etc/zbs 目录），并在 zbs-chunk --chunk_ip < mgt_ip> internal_io get。

### pyzbs 调试

即 zbs-rest-server 调试，目前 python3 所有 venv 环境都放在 /usr/local/venv/ 目录下，包含 ansible、elf、job-center、pyzbs、tuna、zbs-client-py，每个 venv 包含自己的 bin、lib 等子目录。

```shell
# 在 README.md 中有很清晰的使用说明
# make 命令都是在项目根目录执行

# 执行代码静态检查，输出格式错误，并自动修复代码格式问题
make lint/fix

# 生成 wheel 文件，输出路径在 .build/dist/wheel/tuna_xxx.whl
# wheel 包与 RPM 的区别在于，wheel 只更新了服务代码及其依赖，没有更新系统配置等。
make build/wheel APPS=tuna

# 添加注释输出到 /var/log/zbs/tuna-rest.INFO
logging.info()

# pyzbs 依赖 zbs-client-py，zbs-client-py 依赖 zbs-proto，要保证版本跟 zbs 用的一致
# 更新 pyzbs 依赖的 zbs-client-py 版本，修改 zbs-client-py 字段的值
vim venv/tuna/pyproject.toml

# 在 tuna 的 venv 同时更新 zbs-client-py wheel 和 tuna wheel
/usr/local/venv/tuna/bin/pip install /tmp/zbs_client_py-xxx.whl
/usr/local/venv/tuna/bin/pip install /tmp/tuna-xxx.whl

# 每次改动后需要重启对应的 rest-server
systemctl restart tuna-rest-server zbs-rest-server

# 如果要批量测试，需要借助 postman 批量访问 restful api，记得填 zbs token 和 smartx token
```

确认代码没问题，可以在commit message 里加入 `no-code-coverage-check` 绕过覆盖率检查。

### 测试集群调试

可参考，https://docs.google.com/document/d/1ctc_g51UC_yBsHOkUM4iRzjlYrN8y6buDuxJ6oLg4lU/edit#heading=h.efb25l4u0lco

替换 meta leader，先 stripe src/zbs-metad 再 scp，如果 restart 失败，可用通过 journalctl -u zbs-metad.service 找原因

查看指定列

```shell
# json 格式查看所有 chunk 已使用空间
zbs-meta -fjson chunk list | jq '.[] | {"ID", "Perf Valid Space", "Perf Allocated Space"}'
# 查看所有的 chunk 的 ring id
zbs-meta -fjson topo list | jq 'map(select(.type =="NODE")) | .[] | "\(.["description"]), ring id \(.["ring_id"])"'
# 查看一个 volume 中 prefer local = 2 的所有 cap pid
zbs-meta volume show_by_id 01576676-f8a0-40f2-88c0-434d05cddc8d --show_pextents | grep PT_CAP | awk '{print $1, $(NF-4)}' | grep -w '2'
```

查找 even volume

```shell
zbs-meta volume find even
```

查看一个卷是否有快照，需要给 pool_name + volume_name

```
zbs-meta snapshot list target2 1a8df188-6d4d-4ed5-8759-6324b2dacc98
```

指定 meta leader 节点位置

```shell
zbs-tool service set_priority --force meta 10.0.0.222:10100:2
zbs-tool service set_priority meta ""
```

根据 pid 查 volume，elf 或 server san 都可以。对于 VMware 环境，由 pid 关联的 volume 可以找到 nfs file，文件名即对应 VM，所以没有提供额外的命令行查询

```shell
zbs-tool elf get_vm_by_pid [pid]
zbs-meta pextent getref <pid> 
```

观察 ELF 集群厚制备副本分配情况

```shell
# 查看 pextent 所属的 volume
zbs-meta pextent getref <pid>
# 查看 elf 集群中 2/3 副本、thin/thick 4 种情况的 target
# 到对应 target 里面找在 tower 界面上创建的指定大小的虚拟卷对应的 lun
zbs-iscsi target list
# 查看 lun 的副本分配情况
zbs-meta volume show_by_id 2cf94874-a119-49ad-8857-c5e5004174fb --show_pextents --show_replica_distribution
# 查看节点的 chunk 编号
zbs-meta chunk show 10.1.131.231 10200
# 查看集群节点容量
zbs-meta cluster summary
# 查看 chunk 健康情况
zbs-meta chunk list
```

写分区并观察 IO 流量和 recover 情况

```shell
fio -name=warmup -filename=/dev/sdd -bs=256k -direct=1 -numjobs=1 -iodepth=32 -rw=write -ioengine=libaio
# /dev/sdd 所对应的 volume 的实时 IO 流量
watch -n 1 zbs-perf-tools volume show 7c58b428-5dc0-4a7a-88d6-9025377c44a0
# 要确保写的副本有被分配到这台 chunk 上
watch -n 1 zbs-perf-tools chunk lsm summary
# 查看 Volume 中各个 extent 的副本分配情况
zbs-meta volume show_by_id --show_pextents --show_replica_distribution 7c58b428-5dc0-4a7a-88d6-9025377c44a0
# 强制格式化分区并挂载
zbs-chunk partition format --force /dev/sdb && zbs-chunk partition mount /dev/sdb
# 查看正在执行的 recover cmd
zbs-meta recover list
# 查看待执行的 recover cmd
zbs-meta pextent find need_recover
# 展示 zbs 版本信息
cat /etc/smtx-release.yaml && rpm -qa | grep zbs
```

数据存储 pool -> 存储卷 volume -> 数据块 extent

```shell
# 查看集群中连通的 chunk 列表
zbs-meta chunk list
# 查看集群中全部 pool 信息
zbs-meta pool list
# 查看某个 pool 中所有存储卷信息
zbs-meta volume listall <pool_name>
# 查看某个 volume 信息
zbs-meta volume show <pool_name> <volume_name> --show_pextents --show_replica_distribution
# 查看当前集群中的 extent 信息
zbs-meta pextent list [start_id] [end_id]
# 查看某个 extent 所属的 volume pool 信息
zbs-meta pextent getref <pextent_id>
# 查看某台 chunk 上的所有 pextent 的 ID 信息
zbs-meta chunk list_pid 10.x.x.x 10200
# 查看当前节点分区信息
zbs-chunk journal/cache/partition list
# 查看本地所有 extent 的信息
zbs-chunk extent list
# 查看当前集群的整体状态及 leader 节点的 IP
zbs-tool service list

# 三个接入协议 nfs, iscsi, nvmf
zbs-nfs export list
zbs-iscsi target list
zbs-iscsi lun list <target_name>
zbs-iscsi lun show --all <target_name> <lun_id>
zbs-nvmf subsystem list
zbs-nvmf ns list <subsystem_name>

# 创建 pool
zbs-meta pool create <pool_name>
# 创建 volume
zbs-meta volume create <pool_name> <volume_name> <volume_Gb_size>
# 默认集群搭建完应该都已经格式化并挂载完
zbs-chunk format <path> && zbs-chunk journal/cache/partition mount <path>
# 创建 nfs export（没有提供创建 dir 的命令）
zbs-nfs export create <export_name>
# 创建 iscsi target
zbs-iscsi target create <target_name>
# 创建 iscsi lun
zbs-iscsi lun create [--lun_name <LUN_NAME>] <target_name> <lun_id_0-256> <Gib_size>
# 创建 nvmf subsystem
zbs-nvmf subsystem create <name>
# 创建 nvmf namespace
zbs-nvmf ns create <subsystem_name> <ns_id_0-256> <Gib_size>
```

### IO Reroute 使用

1. hypervisor 命令 

    ```shell
    # 查看虚拟机进程
    esxcli vm process list
    # 类比 htop
    esxtop
    ```

    杀死对应进程

    1. 按 C 切换到 CPU 资源利用率屏幕。
    2. 按 Shift+V 将视图限定为虚拟机。这样会更容易在步骤 7 中找到 Leader World ID。
    3. 按 F 显示字段列表。
    4. 按 C 添加 Leader World ID 列。
    5. 通过目标虚拟机的名称和领导者域 ID (`LWID`) 来标识目标虚拟机。
    6. 按 K。
    7. 在 `World to kill` 提示符处，键入步骤 7 中获取的 Leader World ID，然后按 Enter。
    8. 等待 30 秒后验证该进程是否已不再列出。

2. 常用命令

    ```shell
    # esxi 中查看路由 
    esxcfg-route -l
    # xen 中查看路由
    route -n
    
    # xen 中删除 reroute 进程
    ps -ef | grep scvm_failure_loop.sh  |grep -v grep | grep -v vi | awk '{print $2}' | xargs /bin/kill
    # esxi 中删除 reroute 进程
    ps -c | grep scvm_failure_loop.sh | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill
    ps -ef | grep scvm_failure_loop.sh |  grep -v grep | grep -v vi | awk '{print $2}' | xargs /bin/kill
    
    ps -c | grep reroute.py | grep -v grep | grep -v vi | awk '{print $1}' | xargs /bin/kill
    ```

### NFS 使用

1. 在嵌套集群中使用 zbs-nfs export 命令创建一个 NFS export，例如 zbs-nfs export create my-export；
2. 在 Linux VM 中安装 nfs 组件 yum install nfs-utils，并创建目录用于挂载 NFS mkdir -p /mnt/my-dir
3. 在 Linux VM 中使用 mount -t nfs  <嵌套集群IP>:/zbs/my-export /mnt/my-dir 命令挂载 NFS export；
4. 在挂载成功后，进入挂载目录，尝试创建文件夹与文件，并写入数据；

### iSCSI 使用

搭建基于 ELF 的集群，通过 iSCSI 为在该嵌套集群所在的物理集群上创建的一个Linux VM 使用 ZBS 提供的存储。经黄萍讲解，NFS 与EXSi 搭配使用，该 Linux VM 通过 iSCSI 访问集群上 ZBS 的 chunk。

1. 集群默认已经关闭 SELinux，否则将`/etc/sysconfig/selinux`中的`SELINUX`改为`disabled`，使得chunk相关端口允许被访问，或 setenforce=0；

2. 在嵌套集群中使用 zbs-iscsi 命令行创建一个 Target ，并在 Target 中创建一个 LUN；

   ```shell
   ssh smartx@192.168.27.37
   zbs-iscsi target create zp-iscsi
   # 创建 zp-iscsi 下的 lun_id 为 1，size = 3 GB 的 LUN
   zbs-iscsi lun create zp-iscsi 1 3
   ```

3. 在 Linux VM 中使用 iscsiadm discovery 并 login 对应的 LUN；

   ```shell
   ssh root@192.168.27.30
   # 下载 iscsi 工具的
   yum install iscsi-initiator-utils -y
   # 查找目标，必须执行这条指令后面才能登入（st 是 sendtargets 缩写）
   iscsiadm -m discovery -t st -p 192.168.27.37:3260
   # 登入目标
   iscsiadm -m node -T iqn.2016-02.com.smartx:system:zp-iscsi -l
   # 重连 iscsi targets
   iscsiadm -m session -R
   # 查看存储设备，/dev/sdd 为连接的 iscsi 存储设备信息
   fdisk -l
   # 格式化设备
   mkfs.ext3 /dev/sdd
   # 挂载设备
   mkdir /mnt/iscsi
   chmod 777 /mnt/iscsi
   mount -o rw /dev/sdd /mnt/iscsi
   # 查看本机所有挂载设备
   df -h 或者 lsblk
   # 如果要在系统启动时自动挂载，启动守护进程，并在/etc/fstab 中加入一行
   service iscsi start
   /dev/sdd /mnt/iscsi ex3 default 0 0
   # 首先解除挂载，然后登出节点
   umount /mnt/iscsi
   iscsiadm -m node -T iqn.2016-02.com.smartx:system:zp-iscsi -u
   ```

4. 使用 dd 命令对新增的磁盘进行读写，分别在 Linux VM 和集群内的各个节点上使用 iostat 2 -xm （yum install sysstat）观察磁盘 IO 情况；

   ```shell
   # conv=fsync 执行写同步，实际刷入磁盘 默认的 (bs = 512 Bytes) * (count = 2048k) = 1GB
   dd if=/dev/random of=/dev/sdd count=1024k conv=fsync
   # 测试读能力
   dd if=/dev/sdd of=/dev/null bs=4k
   ```


### ZK 使用

* 连接 zk：`zkCli.sh -server 10.1.242.103:2181`
* 开启 / 查看 / 停止 / 重启 zk 服务：` zkServer.sh start / status / stop / restart`
* 创建节点：`create  [-s] [-e] path data acl`，如`create /zk-book content123`
* 删除节点：`delete path [version]`
* 修改节点：`set path data [version]`
* 查看节点：列出指定节点下的所有子节点`ls path [watch]`、查看指定节点的数据内容和属性信息`get path [watch]`
* zk 遵循环形选举，zbs 中各服务的 leader 之间不一定相同、且和 zk 也不一定相同（查看当前各服务的 leader，`zbs-tool service list`）

### 以 volume id 查询

1. iscsi，通过 volume id 找到对应的 iscsi target id 和 lun id 和 lun name，之后可以用 lun name 在 tower 中搜索就能找到对应的虚拟卷，进而可以看到该卷挂载了哪些虚拟机。

   ```shell
   [root@hygon-node-19-98 17:57:40 ~]$ zbs-iscsi lun show_target_and_lun_by_uuid 817e39de-41ee-49d5-a9bb-446951d4d607
   Target:
   ----------------------  ------------------------------------
   ID                      c50b577d-98cb-4187-836e-a967937e2d67
   Name                    zbs-iscsi-datastore-1711160779164l
   ...
   ----------------------  ------------------------------------
   Lun:
   ------------------  ----------------------------------------
   LUN Id              52
   LUN Name            841e69ac-2b1e-4080-8a7e-596de4b9aa43
   Volume id           817e39de-41ee-49d5-a9bb-446951d4d607
   ...
   ------------------  ----------------------------------------
   ```

2. nfs，volume id 前三节就是 inode id，之后可以用 inode id 在 tower 中搜索就能找到对应的虚拟卷。

### 在 tower 查询

1. 通过 tower 界面找 volume id

   iscsi，smtxos 5.0.5 之前，tower 界面上的虚拟卷点开详情，其中的存储对象 "iscsi://iqn.2016-02.com.smartx:system:zbs-iscsi-datastore-1711160779164l/52" 指明 Target name 和 lun id

   ```shell
   [root@hygon-node-19-98 18:07:08 ~]$ zbs-iscsi lun show zbs-iscsi-datastore-1711160779164l 52
   ------------------  ------------------------------------
   LUN Id              52
   LUN Name            841e69ac-2b1e-4080-8a7e-596de4b9aa43
   Volume id           817e39de-41ee-49d5-a9bb-446951d4d607
    ...
   ------------------  ------------------------------------
   ```

   iscsi，smtx os 5.0.5 之后，在 tower 界面上的虚拟卷点开详情，其中的 UUID "841e69ac-2b1e-4080-8a7e-596de4b9aa43" 可以用来查看 volume id

   ```shell
   [root@hygon-node-19-98 18:04:23 ~]$ zbs-tool elf get_zbs_vol_by_elf_vol_uuid 841e69ac-2b1e-4080-8a7e-596de4b9aa43
   -------------  ------------------------------------
   Volume UUID    841e69ac-2b1e-4080-8a7e-596de4b9aa43
   Zbs Volume ID  817e39de-41ee-49d5-a9bb-446951d4d607
   -------------  ------------------------------------
   ```

   nfs，在 tower 界面上的文件点开详情，其中的 ID "d9310c3c-c1ad-42f3" 可以用来查看 volume id

   ```shell
   [root@node51 18:19:59 ~]$zbs-nfs inode show d9310c3c-c1ad-42f3
   -----------  ------------------------------------
   id           d9310c3c-c1ad-42f3
   name         sijie-test-flat.vmdk
   pool         9667cc1e-67e9-4f67-8bb1-e04792b7670e
   volume       d9310c3c-c1ad-42f3-8869-6a73c5630456
   type         FILE
   ...
   -----------  ------------------------------------
   ```

2. 

### 以 pid 查询

1. 通过 pid 找所有引用他的 volume

   ```shell
   [root@hygon-node-19-98 21:10:39 ~]$ zbs-meta pextent getref 3662796
   -----------------------  ------------------------------------
   Pool Name                zbs-iscsi-datastore-1711160779164l
   Volume ID                817e39de-41ee-49d5-a9bb-446951d4d607
   Volume Name              817e39de-41ee-49d5-a9bb-446951d4d607
   Volume Is Snapshot       False
   Volume Snapshot Pool ID
   Snapshot Name
   -----------------------  ------------------------------------
   ```

2. iscsi，通过 pid 找所有使用它的虚拟机（从 SMTX OS 5.0.5 ELF 模式开始）

   ```shell
   [root@hygon-node-19-98 21:10:54 ~]$ zbs-tool elf get_vm_by_pid 3662796
        ID  Replica    Alive Replica    Temporary Replica    Ever Exist    Is Garbage      Origin PExtent    Expected Replica num     Epoch    Prefer Local  Allocated Space    Affected ZBS Volumes                      Affected ZBS NFS Files    Affected VMs
   -------  ---------  ---------------  -------------------  ------------  ------------  ----------------  ----------------------  --------  --------------  -----------------  ----------------------------------------  ------------------------  --------------------------------------------------------------------------------------------
   3662796  [2, 3]     [2, 3]           []                   True          False                        0                       2  41818550               2  512.00 KiB         ['817e39de-41ee-49d5-a9bb-446951d4d607']  []                        ['sfs-fffebc73-46a9-4991-80ee-9cda40c62ff5-0', 'sfs-fffebc73-46a9-4991-80ee-9cda40c62ff5-2']
   ```

3. nfs，通过 pid 找所有使用它的虚拟机，对于 VMware 环境，由 pid 关联的 volume uuid 的前 3 节即 inode id，据此可以找到 inode name 即对应 VM name。

4. 查看 pid 副本都在哪些节点的哪些盘上

   ```shell
   # 查看他所处的 volume
   zbs-meta pextent getref 186992
   # 查看 segment cid
   zbs-meta volume show_by_id b89004e0-de62-401a-b239-afcefbd31dac --show_pextents | grep 186992
   # 到副本所在节点上执行，就可以看到具体在哪个数据盘上
   [root@zyx-node1 13:51:33 ~]$ zbs-chunk extent show 186992 --show_disk
   pid: 186992
   epoch: 187038
   generation: 5530
   status: 1
   private_blob_num: 322
   shared_blob_num: 0
   disk_refs: /dev/sdc4(322)
   ```

   
