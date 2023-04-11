### ZBS 常用 CLI

数据存储 pool -> 存储卷 volume -> 数据块 extent

```shell
# 查看集群中连通的 chunk 列表
zbs-meta chunk list 
# 查看集群中全部 pool 信息
zbs-meta pool list
# 查看某个 pool 中所有存储卷信息
zbs-meta pool list <pool_name>
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
zbs-tool serive list

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
   # 查找目标
   iscsiadm -m discovery -t st -p 192.168.27.37:3260
   # 登入目标
   iscsiadm -m node -T iqn.2016-02.com.smartx:system:zp-iscsi -l
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
   dd if=/dev/zero of=/dev/sdd count=1024k conv=fsync
   # 测试读能力
   dd if=/dev/sdd of=/dev/null bs=4k
   ```

   
