手动安装集群

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

    zbs-deploy-manage clear_deploy_tag

    rm -y /etc/zbs/uuid

    systemctl restart zbs-deploy-server nginx

9. 若部署成功，在所有机器上关机、打完快照再开始后续操作，便于操作失败回滚

详细安装见[安装部署指南](http://docs.fev.smartx.com/smtxzbs/5.4.1/zbs_installation_guide/zbs_installation_guide_27)

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
