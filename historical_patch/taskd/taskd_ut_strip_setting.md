单测中条带化的设置不当，造成 CI 时间过长

如果不开启条带化，写 16K 的数据是在 1 块 extent 上写 16K。而在 CI 的设置中，stripe_num = 4，tripe_size = 4k，也就是说如果写 16K 的数据，需要到 4 块 extent 上各写 4K，因为每次发 4k 的包，所以 DataChannel 上 log 很多。



以下介绍条带化

默认 stripe_num = 1，stripe_size = 262144（256K）

#### 功能目标

截至目前版本（5.x ）ZBS 所实现的 Stripe 和 EC 里 Stripe 不同。EC 中的 Stripe 主要的作用是降低副本数，不同分片数据是关联： data + checksum。而 ZBS 的 Stripe 主要用途是帮助 LSM 将 Extent 分散至不同的 Partition 中，以便在顺序吞吐型 IO 击穿缓存后，获得尽可能高的带宽。Stripe 的属性包含 2 个参数 Stripe Size 和 Stripe Num。他们的基本工作方式是 Stripe Size X Stripe Num 的数据区域将成为一个基础的映射单元，被映射到 Stripe Num 数量的 Extent 中。

Stripe 能够帮助 LSM 尽量将连续的 Extent 分散至不同的 Partition的 原因是 Meta 在创建 Volume 时，不论是 Thin 还是 Thick 的 Extent 都会立即将所有的 pid 分配给 Volume，这样保证了 Volume 上的 pid 在分配顺序是严格且紧密的和它们在 Volume 上的逻辑顺序一致。而 Meta 在分配 pid 时会赋予 pid一个全局单调递增的 epoch（uint64 可能会翻转，但在集群生命周期内可以认为是单调递增的）。LSM 在为 Extent 分配空间时就可以根据相邻的 epoch 识别他们大概率来自一个 Volume ，在分配时会尽量（但是不保证一定）将他们分散至不同的磁盘中。这样对于 256k x 4 的默认配置，1 M 大小的连续 IO 就会被切分为 4 个 256k 的 IO 发往 4 个不同的 Partition，从而尽量充分的利用不同 Partition 的带宽，当然因为现在 HDD 的带宽即便是顺序 IO 也与 NVMe 相差悬殊，所以 IO 目前尽量是由 Cache 负责，我们实际上不太从这个策略中获得好处。并且如果 Volume 执行了 COW，部分 pid 相对随机的被替换为新的 pid，Stripe 也就失效了。

ZBS Client 在收到 UIO 之后会按照 Volume 的 Stripe 设置将其拆分为指定的 BIO。但是在目前看起来我们其实并不建议用户自己调整这个参数，在 HCI 模式下这个参数会被固定为 256K x 4，在 ZBS 模式下会被固定为 256K x 8。尽量减少拆分的层级和次数。



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

