单测中条带化的设置不当，造成 CI 时间过长

如果不开启条带化，写 16K 的数据是在 1 块 extent 上写 16K。而在 CI 的设置中，stripe_num = 4，tripe_size = 4k，也就是说如果写 16K 的数据，需要到 4 块 extent 上各写 4K，因为每次发 4k 的包，所以 DataChannel 上 log 很多。



以下介绍条带化

默认 stripe_num = 1，stripe_size = 262144（256K）

#### 功能目标

由于 ZBS 中 LSM 模块设计上的问题，单个 Extent 内部对于并发 IO 存在一定的限制：

1. 一个 Extent 的 IO 都需要经过同一个 Journal。一个 Journal 只存储在一个 SSD 磁盘上，无法发挥出多块 SSD 的性能。

2. 一个 Extent 的数据，在 HDD 上是连续分布的，只能发挥一个 HDD 的性能，无法发挥多块 HDD 的性能。

基于这两点，在顺序 IO 的场景下，性能受到很大的限制。而条带化的目的，就是在顺序 IO 的场景下，将 IO 分散到多个 Extent，利用多 Extent 之间的并发性，提高顺序 IO 的性能。

#### 具体介绍 

条带化是一种逻辑地址的映射关系，以下图为例，条带数为 4，每一个 extent 中包含 8 个 block

![image-20220821101835552](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211018575.png)

在进行条带化处理后，顺序 IO（V1-B1，V1-B2，V1-B3，V1-B4，V1-B5，V1-B6，V1-B7，V1-B8），则转换为（V1-B1，V2-B1，V3-B1，V4-B1，V1-B2，V2-B2，V3-B2，V4-B2）。这样，就可以利用到 V1，V2，V3，V4，4 个 Extent 的并发性。

以上只是一个示例。通用的地址映射关系为：

![image-20220821101851439](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211018464.png)

通过映射后，striped_vextent_no 与 striped_extent_offset 可以定位到新的逻辑地址。在添加条带化配置后，为满足条带化要求，一个 volume 的 size 将被限制为 kExtentSize * stripe_num 的倍数。

