用 uint64_t 来声明 ring_id

初始时生成符合 topo_id 次序的 ring_id

当有任意一种拓扑行动时，重新生成 ring_id

ring_id 的生成是按不同 brick/rack 中节点数量的比例配对

双活集群中优先可用域 2 副本，次级可用域 1 副本。

在重新生成 ring_id 时要保持原来的次序

UpdateTopoObj/RegisterChunk/RemoveChunk

以 brick 为粒度算节点数，找到 3 个 brick 中最小的节点数，记得 k，对其他两个 brick k 等分

按 brick 编号挑 3 个节点





文档中描述过程并说明算法可以收敛，稳定性





机器重启后，zk 要手动重启，selinux 要关闭



ZBS RPM 和 SMTX ZBS/SMTX OS/IOMesh 等不同产品的产品版本号从 5.0.0 开始已经分离，各有各的版本号

非首次运行单测

```shell
docker run --rm --privileged=true -v /code/zbs:/zbs -w /zbs/build registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 ninja zbs_test
cd /code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"
# 屏幕中会提示出错处的日志信息
```



IO 流

Access 提供的是外部客户端进入 ZBS 系统内的接入点功能。在数据请求达到 Access 后，它将负责把它转化为 ZBS 内部请求。处理的基本过程如下：

1. 直接接入的 Access 如果之前从未处理过对应数据块（Extent） 的数据请求，则会首先向 Meta 请求 Extent 的基本信息（副本分布，权限）。如果 Access 最近已经访问过（1 小时内），则无需这个步骤；
2. 在获得数据访问权限（如果本地的 Access 不持有 Extent 的访问权限，则会转发至持有权限的 Access 继续后续步骤）和基本信息之后：
    1. 读请求：本地 LSM 在副本列表中，Access 会优先访问本地的 LSM （无需经过网络，本地的内存交换即可），如果成功读取则直接访问，失败则继续尝试逐一其余副本直至成功或者全部失败；
    2. 写请求，并发的向所有副本发出写命令，确认所有副本均写入成功才返回，如果部分失败，则执行副本剔除，保证副本列表中的所有副本数据一致，如果全部失败，则不剔除，返回异常等待上层重试；



```shell
zbs-meta pextent read -o <output_file> <pextent_id> <offset> <length> 
zbs-chunk extent list 
```

显示 2023-04-07 14:58:47,878, ERROR, cmd.py:2967, write() argument must be str, not bytes

zbs-nfs 中难以查看 dir

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610



centos7 中将 pip 默认是 8.x，升级到 20.3.4（更高版本不支持 Python 2.7）

```
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
```

pip 根据 whl 文件安装包

```
pip install xx.whl
```

git_review 插件安装 1.28.0，进 pypi.org 找包的各个历史版本

```
pip install git-review==1.28.0
```

pip 更改镜像源

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



