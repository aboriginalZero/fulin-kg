IMPI  ADMIN ADMIN

fisheye root 111111

prometheus prometheus HC!r0cks ，http://meta_leader_mgt_ip:9090

### 搭建开发机

1. 网卡随开机自启动

    ```shell
    ifup ens-192
    # 在/etc/sysconfig/network-scripts/ens-192 中
    ONBOOT=yes
    systemctl network restart
    ```

2. 挂载用于存放代码的磁盘（开发机选 2 块磁盘）

    ```shell
    # 查看还未挂载的磁盘，可以根据磁盘大小和有无分区来判断
    fdisk -l 
    # 创建硬盘分区，进入 fdisk 进程后，刻制成一个主分区就行，输入 w 保存退出
    fdisk /dev/sdb
    # 格式化分区
    mkfs.ext4 /dev/sdb1
    # 建立挂载目录并挂载
    mkdir /home && mount /dev/sdb1 /home
    # 设置开机自动挂载
    echo /dev/sdb1 /home ext4 defaults 0 0 >> /etc/fstab
    ```

3. 指定公司的 yum 源

    ```shell
    yum install -y wget
    # 用公司 yum 源替换系统自带的 yum 源
    mkdir /etc/yum.repos.d/bk && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bk/ && wget http://192.168.31.215/smartx.repo -O /etc/yum.repos.d/smartx.repo 
    yum clean all && yum makecache
    ```

4. 生成 ssh key 并手动上传到 gerrit

    ```shell
    ssh-keygen && cat ~/.ssh/id_rsa.pub
    ```

5. 安装依赖工具

    ```shell
    yum install -y git224 yum-utils rpm-build
    ```

6. 拉取 zbs 代码

    ```shell
    mkdir -p /home/code && cd /home/code
    git clone "ssh://yiwu.cai@gerrit.smartx.com:29518/zbs" && scp -p -P 29518 yiwu.cai@gerrit.smartx.com:hooks/commit-msg "zbs/.git/hooks/"
    
    cd zbs && git submodule update --init --recursive
    ```

7. 安装编译所需依赖

    ```shell
    # zbs.spec 更新，重新 yum-builddep 
    yum-builddep /home/code/zbs/rpm/zbs.spec
    # mold 用以加快链接，nasm 用于生成兼容多体系架构
    yum install mold nasm
    ```

    不想污染本地环境，也可以用 docker，https://blog.csdn.net/zzhongcy/article/details/131402389，安装 docker 并设置成随开机自启

8. 启动 dev-toolset7

    ```shell
    cat <<EOF >> ~/.bashrc
    source /opt/rh/devtoolset-7/enable
    alias cmake=cmake3
    export PATH=/usr/local/bin:\$PATH
    EOF
    
    source ~/.bashrc
    ```

9. 安装并配置 [ccache](https://ccache.dev/)，ccache 用于缓存编译结果，加速编译

    ```shell
    yum -y install ccache
    
    cp $(which ccache) /usr/local/bin/
    ln -s ccache /usr/local/bin/gcc && ln -s ccache /usr/local/bin/g++ && ln -s ccache /usr/local/bin/c++ && ln -s ccache /usr/local/bin/cc
    
    cat <<EOF > /root/.ccache/ccache.conf
    max_size=30G
    cache_dir=/home/.ccache/ccache
    EOF
    ```

10. 安装 Clang-Format 插件

    ```shell
    yum install llvm-toolset-11.0.x86_64
    yum install llvm-toolset-11.0-clang-tools-extra-11.0.1-1.el7_9.x86_64
    ```
    
11. 编译 zbs

     ```shell
     cd /home/code/zbs && rm -rf build/ && mkdir build && cd build && cmake -G Ninja ..
     cd /home/code/zbs && ./script/format.sh && cd build && mold -run ninja zbs_test zbs-metad
     strip ./src/zbs-metad
     ```

     其中，strip ./src/zbs-taskd 有选择地除去行号信息、重定位信息、调试段、typchk 段、注释段、文件头以及所有或部分符号表，后续难以调试，但能够有效减少二进制文件大小。

11. 单测运行

    ```shell
    yum -y install zookeeper rpcbind
    systemctl start zookeeper 
    
    # 创建所需目录
    mkdir -p /etc/zbs && mkdir -p /var/log/zbs && mkdir -p /var/lib/zbs/chunkd && mkdir -p /var/lib/zbs/iscsid && mkdir -p /var/lib/zbs/metad && mkdir -p /var/lib/zbs/registry && mkdir -p /var/lib/zbs/zbs-nfsd && chmod -R 1777 /var/log/zbs /var/lib/zbs
    
    # 创建 zbs 配置文件
    cat > /etc/zbs/zbs.conf <<  EOF
    [network]
    data_ip=127.0.0.1
    heartbeat_ip=127.0.0.1
    vm_ip=127.0.0.1
    web_ip=127.0.0.1
    
    [cluster]
    role=standalone
    members=127.0.0.1
    zookeeper=127.0.0.1:2181
    mongo=127.0.0.1:27017
    EOF
    
    # 拷贝出一份 zk 配置
    cp -v /etc/zookeeper/zoo_zbs.cfg /etc/zookeeper/zoo.cfg
    sed -i 's/cnxTimeout=2/cnxTimeout=10/g' /etc/zookeeper/zoo.cfg
    
    # 启动 rpcbind
    /sbin/rpcbind -w
    
    # 重启 zk
    zkServer.sh stop
    rm -rf /var/log/zookeeper
    mkdir -p /var/log/zookeeper
    chown -R zookeeper:zookeeper /var/log/zookeeper
    zkServer.sh start || die "Failed to start zookeeper"
    ```

    运行指令

    ```shell
    cd /home/code/zbs/build/src && ./zbs_test --gtest_filter="*FunctionalTest.WriteResize*"  --gtest_repeat=100
    ```

12. 安装 git-review 并提交新代码到 gerrit

    ```shell
    # centos7 中 pip 默认是 8.x，升级到 20.3.4（再更高的版本不支持 Python 2.7）
    wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
    python get-pip.py
    # pip 根据 whl 文件安装包
    pip install xx.whl
    # git_review 插件安装 1.28.0，进 pypi.org 找包的各个历史版本，python 2.7 还支持这个版本
    pip install git-review==1.28.0
    # pip 更改镜像源
    pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
    ```

13. 将终端机的 ssh key 添加到开发机上

    ```shell
     ssh-copy-id root@192.168.101.22
    ```

14. 开发机重启

    ```shell
    # 重启 zk
    zkServer.sh start
    # 关闭 selinux
    setenforce 0
    # 重启 docker 
    service docker start
    ```

    如果 start 失败，用 journalctl -u zbs-taskd 来查看报错信息

### 搭建物理集群

给物理裸机安装 smartxos，并创建集群

1. 通过 IMPI 地址（18.135） 给裸机挂载磁盘

    ```shell
    ssh root@192.168.17.20
    password: HC!r0cks
    cd /home/fengli/IPMI
    ./ipmi_start.sh 18.135  /data/distros/iso/smartos.xx.iso
    ```

2. 进[远程]([http://tower.smartx.com/#/vnc?meta={%22id%22:%22ckcljp2h15e4g0786l9rv6yfv%22,%22name%22:%22qiuping-win7%22}](http://tower.smartx.com/#/vnc?meta={"id":"ckcljp2h15e4g0786l9rv6yfv","name":"qiuping-win7"}))，在 IPMI View add new system 并Lauch KVM，进行系统安装，选定启动盘、存储盘之后就全是自动化操作

3. 访问 smartos 在eth0 / eth1 上分配的 IPv4，进入 web 控制台开始配置集群

通过 impi 部署 hypervisor

```
17.20 机器上，root HC!r0cks
locate zbs-5.5.0 会有路径，替换路径过去
这个节点为想要安装的节点
iso-uploader2 --server_addr 192.168.17.110 SMTXZBS-5.5.0-zhaokai-el7-2307121646-x86_64.iso
```

通常 ipmi (不管 html5 版本还是IMPI Viewer）都只允许一个用户使用。如果你能知道谁还打开了一个 session，一般让他关掉就行。如果不知道：

- 对于这种超微四子星机器：网页 maintenance 里有一个 reset ikvm 的选项
- 对于比较新的 ipmi console，一般可以找到一个 user session，可以强制将它注销掉

没有 UEFI 的话，直接选 IMPI Virtual CDROM 3000，ISO 上传既可以从本地，也可以用 17.20 上的 iso-uploader2

### 搭建嵌套集群

1. 在 tower.smartx.com 的 MLAG 或 SKS 中选择内存 / CPU 使用少的物理节点。主要是根据 CPU 架构如 intel_x86 kunpeng_aarch64 hygon_x86 和指令集不同选不同的集群。

2. 上传 iso 和 vmtools。17.20 机器上（镜像节点），root HC!r0cks 

    IP 是想要安装的物理节点 IP，iso 路径可以用 locate zbs-5.5.0 查找

    iso-uploader2 --server_addr 192.168.17.110 SMTXZBS-5.5.0-xxx-x86_64.iso

    初始安装可以保留比较低的 iso 版本，后续滚动升级上来

3. 挂两个 cdrom：系统镜像和 vmtools，磁盘总线都选 SCSI（除启动盘外的虚拟盘总线类型选择 SCSI，以确保可以在 ELF VM 中生成 Disk UUID），5 块盘（20G 启动盘、2 个 400 G 含元数据分区的缓存盘做 raid1，这个盘也是系统盘，1个 40G 缓存盘，1 个 100G 数据盘，分离模式 SMTXZBS 下的系统盘大于 330 GB），3 个网口（用以管理、接入、存储，第一个用 1GB/s vds，后两个可以共用一个 10GB/s vds），20G 内存，20 vCPU

4. 开机等待安装 OS 后，安装 vmtools

    mkdir /mnt/cdrom && mount -o loop /dev/sr1 /mnt/cdrom && sh /mnt/cdrom/SMTX_VM_TOOLS_INSTALL.sh

    systemctl status SVT

    刷新网页可以看到 tower/fisheye 上显示 ip 地址信息，此时在虚拟机中 rm /etc/zbs/uuid，然后打快照

5. 从刚刚这台机器上克隆 n 台机器出来，网页输入任一节点 IP，进入集群安装界面，根据 tower 上显示的单机 IP 选择要参与构建集群的节点，并重命名为 yiwu-x1-24.20，最后加上管理 IP 地址以做标识。

6. 管理 IP 可以保留原来的配置（掩码一般为 255.255.240.0），但是接入/存储网络需要保证 IP 与其他没有冲突，比如要让接入网使用 10.0.55.101.xxx 网段，那么可以进到任意一台机器上，为接入网口（eth1 或 eth2，使用 ethtool  eth0 查看每个网卡接口的速率）绑定 IP 如 ifconfig eth1 10.0.55.101.100/24，然后通过 nmap -v -sn -n 10.0.55.0/24 -oG - | awk '/Status: Down/{print $2}' 查看其他空闲的 IP 并填到表格中，子网掩码 255.255.255.0。存储网同理。

7. DNS 服务器可以选用 114.114.114.114，NTP 可以选用网关 IP，保证 zbs 中日志打印的时间跟系统时间一致，执行部署，集群部署日志在 /var/log/zbs-deploy.INFO，节点部署日志在 /usr/share/zbs_deploy/zbs_host_deploy.log

8. 若部署失败，清空重新部署

    zbs-deploy-manage clear_deploy_tag，有开 nvmf 需要执行，或者早期的 smtxos 4.0.14 也需要

    rm -y /etc/zbs/uuid，嵌套集群，有虚拟机克隆的情况需要执行

    systemctl restart zbs-deploy-server nginx，有开 nvmf 需要执行，或者早期的 smtxos 4.0.14 也需要

9. 若部署成功，在所有机器上关机、打完快照再开始后续操作，便于操作失败回滚

详细安装见[安装部署指南](http://docs.fev.smartx.com/smtxzbs/5.4.1/zbs_installation_guide/zbs_installation_guide_27)



搭 xen 7.0 + scvm smtx 4.0.14 solution v2 的坑

1. 基本只能在早期的超微节点上部署；
1. xen server 只安装在 SATA dom 上，一般是 sda，因为他的磁盘控制器跟其他磁盘并不一样，方便后续磁盘透传；
2. 一定要把除 xen server 系统盘以外的所有的磁盘透传给 SCVM，通过 ls -lh /sys/block/* | grep pci 可以看到各个磁盘对应的磁盘控制器；
3. 通过虚拟键盘按 F11 进入 BIOS，在引导界面通过 tab 或者 e 把 quiet 去掉，就可以看到安装流程了；
4. NTP 可以从外部获取，选网关那台服务器就行，网关上一般会配 NTP；
5. 第二套集群只有 2 个物理网口，所以不需要严格做 active-backup bonding（3 个及其上或许可以），ab bonding 不强求，客户环境也有单链路使用的；
6. 网络掩码给定 255.255.240.0 后，10.1.1.2 和 10.1.1.3 不需要指定网关理论上也能互相 ping 通，而管理网需要给定网关，这样方便从集群外部通过 ssh 登陆上去；
7. iSCSI/NFS 网络在 xen 环境下绑定的虚拟网卡必须是 disconnected，33.1 的下一跳如果是 33.2 不希望他从物理网卡发包出去；
8. 要通过 mac 地址，而不是 ip 来验证网络配置是否正确；
9. 在引导部署分布式系统前，要保证所有的 scvm 之间、所有 scvm 和 esxi 之间的 data ip 在同一网段且能互 ping，mgt ip 同理，否则部署一定会出错。mgt ip 的掩码可能是 255.255.240.0 也可能是 255.255.128.0；
9. xen io reroute 脚本没有清空 33.2 的下一跳路由。

---



192.168

* yiwu-b2-27-24
* yiwu-b3-23-74
* yiwu-b4-29-123
* yiwu-b5-28-93
* yiwu-b6-30-40



DNS 192.168.32.4，或者选网关机器，我们的网关机器基本都带有 DNS 功能

透明网关，实际上作为网关的机器有多台，通过 ip route | grep default 查看网关 IP 地址

北京：192.168.16.1

IDC：192.168.64.1

默认网关：31.215

内网网段分配

* 各 Office 私有  ｜192.168.0.0/20
* 北京 IDC 机房 1｜ 192.168.16.0/20
* 北京 IDC 机房 2 ｜172.20.0.0/16
* AWS 公网 ｜ 192.168.32.0/20
* 深圳机房 ｜ 192.168.48.0/20
* 北京机房 ｜ 192.168.64.0/20

- 上海机房 ｜ 192.168.80.0/20
- 成都机房 ｜ 192.168.112.0/20
