ZBS RPM 和 SMTX ZBS/SMTX OS/IOMesh 等不同产品的产品版本号从 5.0.0 开始已经分离，各有各的版本号



非首次运行单测

```
docker run --rm  -v /code/zbs:/zbs -w /zbs/build registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 ninja zbs_test
cd /code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"
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



