## HDFS Client

### 读

HDFS 目前实现的读操作有三个层次，分别是网络读、短路读（short circuit read）以及零拷贝读（zero copy read），它们的读取效率依次递增。  

1. 网络读：网络读是最基本的一种 HDFS 读，DFSClient 和 DN 通过建立 Socket 连接传输数据。 

    一个 Block 被切分成若干 Chunk 检验块；一个 Chunk 包含一个校验和 + 所验证的数据；一个数据包 packet 包含一个头域 + 一组 Chunk 的检验和 + 一组 Chunk，Block 的最后一个 packet 是一个空 packet。网络读是一个个 packet 通过 DataTransferProtocol 传输的，这种方式中 DN 需要为每个读取 Block 的 Client 维持一个线程和 TCP Socket，占用系统开销。

2. 短路读：当 DFSClient 和保存目标数据块的 DN 在同一个物理节点上时， DFSClient 可以直接打开数据块副本文件读取数据，而不需要 DN 进程的转发（DN 进程只转发块文件以及该块文件的元数据文件的 fd）。

    当客户端向 Datanode 请求数据时，Datanode 会打开块文件以及该块文件的元数据文件，将这两个文件的文件描述符通过 domainSocket 传给客户端，客户端拿到文件描述符后构造输入流，之后通过输入流直接读取磁盘上的块文件。采用这种方式，数据绕 过了 Datanode 进程的转发，提供了更好的读取性能。由于文件描述符是只读的，所以客户端不能修改收到的文件；同时由于客户端自身无法访问块文件所在的目录， 所以它也就不能访问数据目录中的其他文件了，从而提高了数据读取的安全性。

3. 零拷贝读：当 DFSClient 和缓存目标数据块的 DN 在同一个物理节点上时， DFSClient 可以通过零拷贝的方式读取该数据块，大大提高了效率。而且即使在读取过程中该数据块被 DN 从缓存中移出了，读取操作也可以退化成本地短路读， 非常方便。

    缓冲中的副本已经执行过校验操作，不需要再检验所以可以走零拷贝（应用程序不需要读取）

网络读的流程如下：

![image-20220831160305864](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605601.png)

1. 打开 HDFS 文件。HDFS 客户端首先调用 DistributedFileSystem.open()方法打开 HDFS 文件，这个方法在底层会调用 ClientProtocol.open()方法，该方法会构造一个 DFSInputStream 但返回一个 HdfsDataInputStream 输入流）用于读取数据块。

    > 从继承的父类来看，后者也可以是前者的子类，但他两最多算兄弟类。DFSInputStream 支持基于位置读取功能，支持使用 ByteBuffer 接收数据、读取后删除缓存数据、预读取数据以及零拷贝读取。

2. 从 NN 获取 DN 地址。在 DFSInputStream 的构造方法中调用 ClientProtocol.getBlockLocations(String path, long offset, long len) 方法向名字节点获取该 HDFS 文件指定范围内所有 Block 的位置信息，然后调用 ClientDatanodeProtocol.getReplicaVisibleLength() 方法获取该 HDFS 文件最后一个 Block 在任意一个 DN 中的长度。NN 返回的 Block 的存储位置是按照与客户端的距离远近排序，所以 DFSInputStream 可以选择一个最优的 DN 节点，然后与这个节点建立数据连接读取数据块；

    > 由于文件的最后一个数据块可能处于构建状态（正在被写入）或者整个 HDFS 集群重启的时候有些 DN 还没有向 NN 进行完整的数据块汇报，NN 上的 BlockMaps 数据不全，那么 Namenode 命名空间中保存的数据块长度就有可能小于 Datanode 实际存储数据块的长度，所以需要 Client 与 Datanode 通信以确认文件最后一个 Block 的真实长度。
    >
    > 如果用户想要得到确定性的长度，那么需要调用 sync/flush 函数，那么任意一个 DN 上的最后一个 Block 的长度都是对的，如果用户没有调用 flush 就重新读文件，HDFS 语义不保证返回的文件长度是正确的。

3. 连接到 DN 读取数据块。HDFS 客户端通过调用 DFSInputStream.read() 方法从最优的 DN 读取数据块，数据会以数据包 packet 为单位从 DN 上通过流式接口传送到客户端。当达到一个数据块的末尾时，DFSInputStream 就会再次调用 ClientProtocol.getBlockLocations() 获取文件下一个数据块的位置信息，并建立和这个新的数据块的最优节点之间的连接，然后 HDFS 客户端就可以继续读取数据块了；

4. 关闭输入流。当客户端成功完成文件读取后，会通过 DFSInputStream.close()方法关闭输入流。

数据块的应答包中不仅包含了数据还包含了校验值。 HDFS 客户端接收到数据应答包时，会对数据进行校验，如果出现校验错误，也就是数据节点上的这个数据块副本出现了损坏，HDFS 客户端就会通过 ClientProtocol.reportBadBlocks() 向 NN 汇报这个损坏的数据块副本，同时 DFSInputStream 会尝试从其他的数据节点读取这个数据块。

### 写

![image-20220831163548435](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605945.png)

1. 创建 HDFS 文件。HDFS 客户端首先调用 DistributedFileSystem.create() 方法在 HDFS 文件系统中创建一个新的空文件。这个方法在底层会通过调用 ClientProtocol.create() 方法通知 Namenode 执行对应的操作，Namenode 会首先在文件系统目录树中的指定路径下添加一个新的文件，将创建新文件的操作记录到 editlog ，完成 ClientProtocol.create() 调用后 DistributedFileSystem.create() 方法就会返回一个 DFSOutputStream 的继承类 HdfsDataOutputStream 对象用于写入数据块；

2. 建立数据流管道。由于只是在目录树中创建一个空文件，所以 DFSOutputStream 需要调用 ClientProtocol.addBlock() 向 NN 申请一个新的空 Block，NN 依靠副本分配策略如机架感知等分配不同 DN 上可用的 Block，DFSOutputStream 此时就可以与这些 Block 所在 DN 如 DN 1 2 3 上面的 TCPServer 建立数据流管道写 Block 了；

    > 感知机架的副本分配策略 
    >
    > 如果 Client 在 DN1 节点上，那么副本 1 存在 DN1，副本 2 存在不同于副本 1 的机架上的某个节点上，副本 3 存到与副本 2 同一机架的不同节点上，若还有其他副本则随机挑选节点存储；如果 Client 不在某个 DN 上，那就随机选一个节点存储副本 1，其他副本的选取策略与之前一致。
    >
    > 之所以有这样的存储规则，实际上是充分利用了网络带宽，同一个机架 Rack 上的服务器节点一般通过二层交换机相连，不同机架 Rack 上的服务器节点可能汇聚到三层交换机相连。

3. 通过数据流管道写数据。写入 DFSOutputStream 中的数据会先被缓存在数据流中，之后被切分成一个个 64KB 的数据包 packet  通过 pipeline 依次发送给所有数据节点。每个 DN 本地存完 packet 后会向上一层发送确认包，确认包逆序通过数据流管道回到输出流。

    输出流在确认了所有数据节点已经写入这个数据包之后，就会从对应的缓存队列删除这个数据包，循环执行这个操作直到写完一个 Block，当客户端写满一个数据块之后，会调用 addBlock() 申请一个新的数据块，循环执行这个操作直到写完一个文件；

    > 1. 每传输一个 block 都会新建一个 pipeline，如果 pipeline 中不超过一半的 DN 写入异常，那么剩下正常的节点重新建立 pipeline 写 Block；
    > 2. Client 在建立 pipeline 之后，可以在一个窗口内并发多个 packet。以 packet1 为例，Client 将 packet1 发给 DN1，DN1 在本地存完以后会把 packet1 发给 DN2，DN2 在本地存完以后会把 packet1 发给 DN3，当 DN3 在本地存完以后发 ACK 给 DN2，DN2 发 ACK 给 DN1，DN1 发 ACK 给 Client。Client 收到前几个的 ACK 才可以把滑窗向后移动发送后面的 packet；
    > 3. 当 Client 收到一个 Block 中所有的 packet 的 ACK 后会发送一个空 pakcet 标识当前数据块发送完毕，准备关闭 pipeline。

4. DN 实时增量上报。在写文件的过程中，当 DN 成功地接收一个新的数据块后， DN 通过 DatanodeProtocol.blockReceivedAndDeleted() 方法实时而非默认的每 300s 向 NN 汇报，NN 更新内存中的数据块与数据节点的对应关系；

5. 关闭输入流并提交文件。当 HDFS 客户端完成整个文件中所有 Block 的写操作后就可以调用 close() 方法关闭输出流，并调用 ClientProtocol.complete() 方法通知 NN 提供这个文件中的所有数据块，这样才算完成一个文件的写操作。

### 追加写

1. 打开已有的 HDFS 文件。HDFS 客户端调用 DistributedFileSystem.append() 方法打开一个已有的 HDFS 文件，这个方法在底层调用 ClientProtocol.append() 方法获取文件最后一个 Block 的位置信息（如果该 Block 写满会返回 null）。然后 append() 方法会调用 DFSOutputStream.newStreamForAppend() 方法创建一个到这个数据块的 HdfsDataOutputStream 对象并返回；
2. 建立数据流管道。DFSOutputStream 类的构造方法会判断文件最后一个数据块是否已经写满。如果没有写满，则根据 ClientProtocol.append() 方法返回的该数据块的位置信息建立到该数据块的数据流管道；如果写满了，则调用 ClientProtocol.addBlock() 向 NN 申请一个新的空数据块之后建立数据流管道；
3. 通过数据流管道写数据。同前；
4. DN 实时增量上报。同前；
5. 关闭输入流并提交文件。同前。

## NameNode

### 整体结构

NN 上持有的元数据主要包括：

1. Namespace Mgr。管理文件系统目录树以及每个目录/文件的属性（常驻内存 + 持久化到 FSImage ）；
2. Block Mgr。管理每个文件对应的 Block 列表以及处理 DN 块汇报来管理 Block 的状态（常驻内存 + 持久化）；
3. Datanode Mgr。管理文件的每个 Block 对应的副本所在 DN 列表（非持久化），与 DN 相关的如添加、删除 DN，与 DN 在启动过程的交互，处理 DN 发来的心跳；
4. Lease Mgr。管理能够实现 HDFS Client 互斥写的 Lease 的添加、更新、删除、检查、恢复；
5. Cache Mgr。管理 HDFS Client 保存到 HDFS 缓存的文件/目录。

### Namespace Mgr

Namespace 上保存了文件系统目录树以及每个目录（InodeDirectory）/文件(INodeFile)的属性。这部分元数据除了常驻内存外，定期 flush 到持久化设备上生成一个新的 FsImage 镜像文件，方便 NN 重启时从 FsImage 及时恢复整个 Namespace。主要数据结构如下：

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605478.png" alt="INode继承关系" style="zoom:67%;" />

1. INodeWithAdditionalFields 的 features 除了用来添加扩展属性如 Quota，Snapshot、ACL、XAttrs 等，还可以将当前文件/目录正在构建、快照等状态放到这个 Feature 数组中；

2. INodeFile 的 header 由存储策略 ID（4 bit） + 副本数（12 bit） + 数据块大小（48 bit）组成， blocks 是该文件有序的 Block 数组，指明了当前目录/文件由哪些 Block 构成；
3. INodeDirectory 的成员变量 children 是默认大小为 2 的 ArrayList，按照子节点名字有序存储，虽然在插入时会损失一部分写性能，但是可以方便后续快速二分查找提高读性能，对一般存储系统，读操作比写操作占比要高。

#### FSImage & Editlog

* FSImage 元数据镜像空间中记录的是目录树中所有 Inode 的序列化信息，每条记录中包括了该文件/目录的元数据如名称、大小、用户、创建时间等；
* Editlog 元数据编辑日志记录的是所有 Inode 的更新日志，随着 NN 的运行实时更新并定期合并到 FSImage 中。

为了避免 Editlog 过大，HDFS 通过 checkpoint 机制定期合并 FSImage 和 Editlog 产生最新 FSImage，checkpoint 时机有两个 1. 从上一次 checkpoint 后经过足够的时长； 2. 从上一次 checkpoint 后发生足量的事务。NN HA 机制下，由 SNN 来 Checkpoint。

NN 重启时需要加载最近 FSImage 中的元数据并在内存中构造出目录树，然后重放生成该 FSImage 之后的 Editlog 对目录树进行修改。

### Block Mgr

Block Mgr 维护从 Block 到 DN Replica 列表的映射关系。常驻内存，不会持久化。主要数据结构如下：

![img](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605977.png)

BlockInfo 是 Block 的继承类，其中 bc 字段指明了该 Block 属于哪一个 HDFS 文件。HDFS 中每个文件持有一个 BlockInfo 数组，每个 BlockInfo 大小为 3 倍副本数，该 Block 的第 i 个副本的 triplets[i] 记录该 Block 所在的 DN，triplets[i + 1] 和 triplets[i + 2] 记录在该 DN 上前后一个 Block 的位置。

由于 DN 的每一个存储单元 DatanodeStorageInfo 上的所有 Blocks 集合会形成一个双向链表，所以引入一个自实现的 BlocksMap（一种 HDFS 自实现的 GSet HashMap）来快速根据 Block 定位到 BlockInfo（双向链表 + 哈希表的组织方式）



额外的，在 Block Mgr 中还有以下若干种保存不同状态数据块副本的数据结构：

1. excessReplicateMap 用来放置需要删除的多余副本；
2. neededReplications 用来放置需要添加到预期副本数的副本；
3. invalidateBlocks 用来放置被用户删除的副本等。

![image-20220901163739717](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605947.png)

#### Block & Replica State

在 HDFS 中将 NN 中的数据块信息叫作数据块 Block，但是将 Datanode 中保存的数据块称之为副本 Replica。

Block 副本状态

1. FINALIZED。DN 上的副本已经完成写操作，不再修改；
2. RBW。刚刚被创建或者追加写的副本，处于写操作的数据流管道中，正在被写入，且已写入副本的内容还是可读的；
3. RWR。RWR 状态的副本不会出现在数据流管道中，结果就是等着进行租约恢复操作，如果一个 Datanode 挂掉并且重启之后，所有 RBW 状态的副本都将转换为 RWR 状态；
4. RUR。Lease 过期之后发生的 Lease 恢复和数据块恢复时副本所处的状态；
5. TEMPORARY。DN 之间传输副本时，正在传输的副本所处的状态，不可读，如果 DN 重启，会直接删除处于该状态的副本。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605812.png" alt="image-20220901142732516" style="zoom:50%;" /><img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605605.png" alt="image-20220901142751794" style="zoom:60%;" />

Replica 数据块状态

1. UNDER_CONSTRUCTION。文件被创建或者进行追加写操作时正在被写入的数据块状态。处于该状态的数据块 length 和 generationstamp 都是可变的，但是处于该状态的数据块对于读取操作来说是可见的。 
2. UNDER_RECOVERY。正在进行 Lease recovery 和 Block recovery 的数据块状态。如果一个文件的最后一个数据块处于 UNDER_CONSTRUCTION 状态时客户端异常退出并且该文件的 Lease 超过 softLimit 过期，该数据块就需要进行 Lease recovery 和 Block recovery 流程释放 Lease 并关闭文件；
3. COMMITTED。客户端在写文件每次请求新的数据块 addBlock RPC 或者关闭文件时，都会顺带对上一个数据块进行提交 commit 操作将上一个数据块从 UNDER_CONSTRUCTION 状态转换成 COMMITTED 状态。COMMITTED 状态的数据块表明客户端已经把该数据块的所有数据都发送到了 DN 组成的数据流管道中，并且已经收到了下游的 ACK 响应，但是 NN 还没有收到任何一个 DN 汇报有 FINALIZED 副本；
4. COMPLETE。数据块的 length 和 generationstamp 不再发生变化，并且 NN 已经收到至少一个 DN 报告有 FINALIZED 状态的 Replica。 一个 COMPLETE 状态的数据块会在 NN 的内存中保存所有 FINALIZED 副本的位置，只有当 HDFS 文件的所有数据块都处于 COMPLETE 状态时，该 HDFS 文件才能被关闭。

### Datanode Mgr

与 Datanode 相关的，包括添加和删除 Datanode、与 Datanode 启动过程的交互、处理 Datanode 发送的心 跳。



### Lease Mgr

HDFS 支持一次写入多次读取的访问方式，对多个 HDFS Client 写文件的互斥实现依赖于 Lease 机制。Lease 是一个分布式时间约束锁，同一时刻只有一个 HDFS Client 可以持有 Lease 对某个文件进行写操作。涉及到的数据结构有：

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121605894.png" alt="LeaseManager的内存数据结构" style="zoom:67%;" />

* sortedLeases 是一个按照时间顺序有序组织 Lease 的集合，便于检查 Lease 是否超时；
* leases 是一个哈希表，维护 HDFS Client 到 Lease 的映射关系，便于根据 HDFS Client 快速检索 Lease；
* sortedLeasesByPath 是一个哈希表，维护文件路径到 Lease 的映射关系，便于根据文件路径快速检索 Lease；

```java
class Lease {
    // 租约持有者
    private final String holder;
    // 最近更新时间
    private long lastUpdate;
    // 当前租约持有者打开的文件
    private final HashSet<Long> files = new HashSet<>();
    ...
}
```

正常情况下，HDFS Client 向集群写文件前需要向 NN 的 Lease Mgr 申请 Lease，写文件的过程中以 softLimit 的频率定时更新 Lease 以防 Lease 过期，写完数据后申请释放 Lease。但这个过程中可能发生以下两种情况：

1. 写过程中 HDFS Client 没有及时更新 Lease

    当没有及时更新 Lease 超过 softLimit（2s）时间后，有另一客户端尝试对同一文件进行写操作时将会触发 Lease 软超时强制回收。

    > 只要Lease存在，Lease就是有效的，只是超过SoftLimit，是允许其他client抢占。

2. 写完成后 HDFS Client 没有成功释放 Lease

    当没有成功释放 Lease，会由 Lease Mgr 的后台线程 LeaseManager.Monitor 检查是否硬超时（60min）后统一触发超时强制回收。

    > 租约检查可能放生的两个时机：1. 为新写的block选择目标存储节点时 2. 进行数据同步到磁盘时 3. 建立pipeline过程中出现异常，申请额外的 DataNode 用于构建数据写入的 pipeline 时 4. 完成对指定文件的写操作，关闭文件时

超时强制回收由 FSNamesystem.internalReleaseLease() 执行，先对 Lease 过期前最后一次写入的 Block 进行检查和修复，之后释放超时持有的 Lease，内存中与该 Lease 相关的内容都要回收，保证后面其他客户端的写入能够正常申请到该文件的 Lease。

补充说明

1. Lease 续签是以 Lease，也就是以 client 为单位的，不涉及对其中某个文件的续租，而是整体的续租。具体续租了哪些文件，这完全由NameNode(LeaseManager)中该 Lease 对应的文件信息来决定。如果通过其他调用修改了 NN 上该 Lease 对应的文件信息，例如移除了其中的某个文件 file1。那么即使该 Client 端还在不断地续租，也没有对 file1 的写权限。

2. client 要正常拥有对文件的写权限，就需要每隔 2s，完成一次 lease 续约 (renew lease)。如果在 2s 之后，还没有进行续约，则其他 client 可以进行抢占，但如果没有其他 client 抢占，该 client 还是拥有这份 lease，同样还是对文件具有写权限。

3. 在关闭文件的时候不会马上移除租约，而是依赖于 LeaseManager 的租约过期移除操作。文件在关闭的过程中，会把自身从相应的 DFSClient 客户端对象中移除，然后使得这个 DFSClient 从 LeaseRenewer 对象中移除，最后让它的租约不再更新。

4. 服务器修改元数据时，需要阻塞所有的读请求，此时服务器不能颁发新 Lease，以防止新发出的 Lease 保证的数据与服务器刚才修改的数据不一致。

    解决方法：读请求到来时直接返回数据，不颁发 Lease。

5. 服务器需要等待直至所有的 Client 的 Lease 都过期后，才颁发新“修改”后的 Lease。因此，此时服务器上的数据修改了，生成了一个新的 Lease 版本，需要等到 Client 上所有的老 Lease 过期后，该新 Lease 版本才能颁布给 Client。

    解决方法：服务器主动通知持久 Lease 的 Client 放弃当前的 Lease，并请求新 Lease。

[双主问题](https://zhuanlan.zhihu.com/p/140260158)

### Cache Mgr

HDFS最初实现时，并没有区分本地读和远程读，二者的实现方式完全一样，都是先由DataNode读取数据，然后通过DFSClient与DataNode之间的Socket管道进行数据交互。这样的实现方式很显然由于经过DataNode中转对数据读性能有一定的影响。

集中式缓存方案的主要优势：

1. 用户可以指定常用数据或者高优先级任务对应的数据常驻内存，避免被淘汰到磁盘。例如在数据仓库应用中事实表会频繁与其他表JOIN，如果将这些事实表常驻内存，当DataNode内存使用紧张的时候也不会把这些数据淘汰出去，可以很好的实现了对于关键生产任务的SLA保障；
2. 由NameNode统一进行缓存的集中管理，DFSClient根据Block被Cache分布情况调度任务，尽可能实现本地内存读，减少资源浪费；
3. 明显提升读性能。当DFSClient要读取的数据被Cache在同一DataNode时，可以通过零拷贝直接从内存读，略过磁盘IO和checksum校验等环节，从而提升读性能；
4. 由于NameNode统一管理并作为元数据的一部分进行持久化处理，即使DataNode节点出现宕机，Block移动，集群重启，Cache不会受到影响。

指定 Cache 流程

1. 用户通过调用客户端接口向NameNode发起对 CacheDirective（Directory/File）缓存请求；
2. NN 接收到缓存请求后，将 CacheDirective 转换成需要缓存的 Block 集合，并根据一定的策略并将其加入到缓存队列中；
3. NN 将下发 cacheBlock/uncacheBlock 指令到 DN；
4. DN 接收到 NN 发下的缓存指令后根据实际情况触发 Native JNI 进行 mmap/mlock 系统调用，对 Block 数据进行实际缓存；
5. 此后 DN 定期（默认10s）向 NN 进行缓存汇报，更新当前节点的缓存状态；
6. 当客户端进行发起读请求时，上层调度尽可能将任务调度到数据所在的 DN，客户端向 DataNode发出REQUEST_SHORT_CIRCUIT_FDS 请求到达 DN 后，DN 会首先判断数据是否被缓存，如果数据已经缓存，将缓存信息返回到客户端后续直接进行内存数据的零拷贝；如果数据没有缓存采用传统方式进行短路读。

虽然优势明显，但是HDFS集中式缓存目前还有一些不足：

1. 内存置换策略（LRU/LFU/etc.）尚不支持，如果内存资源不足，缓存会失败；
2. 集中式缓存与 Balancer 之间还不能很好的兼容，大量的 Block 迁移会造成频繁内存数据交换；
3. 缓存能力仅对读有效，对写来说其实存在副作用，尤其是 Append；
4. 与 Federation 的兼容非常不友好；

### Start & Stop

NN 启动流程

1. 进入 SafeMode 阶段。此时不会发生 Block 的复制；
2. 加载镜像文件：从 fsimage file 中读取最新的元数据快照即最近生成的 fsimage_xx；
3. 编辑日志回放：读取 fsimage_xx 之后的所有事务的 edit logs，将 edit logs 中的操作重新执行一遍，此时 NameNode 就恢复到上次停止时的状态了；
4. checkpoint：将当前状态写入新的 checkpoint 中，即产生一个新的 fsimage_xx 文件；
5. 接受 DN 的 BlockReport：等待各个 DN 的 BlockReport 和 HeartBeat，构造 blockMap，然后退出安全模式。此时 NameNode 启动结束，等待接受用户的操作请求，并把用户操作写入新的 edit log 中，定期进行 checkpoint，对元数据执行快照。

当元数据规模达到5亿（Namespace中INode数超过2亿，Block数接近3亿），FsImage文件大小将接近到20GB，加载FsImage数据就需要 14min，Checkpoint 需要 6min，再加上其他阶段整个重启过程将持续 50min，极端情况甚至超过60min。

在启动时，NN 进入一个称为 Safemode 的特殊状态。当 NN 处于 Safemode 状态时，不会发生数据块的复制。NN 接收来自 DN 的Heartbeat 和 Blockreport 消息，Blockreport 包含 DN 所托管的数据块列表。每个块都有指定的最小副本数，当该数据块的最小副本数记录到 NN 时，该块就是安全副本。在 NN 记录了一定百分比的安全副本数据块后(再加上30秒)，NN 将退出 Safemode 状态。然后它确定数据块列表(如果有的话)，这些数据块的副本数量少于指定的数量，NN 会将这些块复制到其他 DN。

之后 DN 会进行全量上报，增量上报，当前的 NN 也会定期通知另一个 active NameNode，生成 Editlog。这些 Editlog 也会并行保存在JournalNode 中，当前 NN 会定期去 JournalNode 上拉取 EditLog，更新自己的内存数据。

可以从三个方面做 NN 启动的性能提升：

- 加载元数据镜像文件时，尤其是加载 INODE，INODE_DIR 可以并行加载；
- 在校验 FSImage 时，原来是单点串行，可以做成并行处理；
- 在 NN 解析完元数据之后有一个更新内存的动作，原来也是串行处理，现在完全可以做成并行处理。

## DataNode

### 整体结构

DN 对内以 Block 的形式存储 HDFS 文件，周期性地向 NN 分别汇报心跳、数据块、缓存数据块的信息（周期时间各不相同），对外响应 HDFS Client 的读写请求，NN 跟随心跳下发的命令。

DataNode 记录每个文件中各个块所在的数据节点信息，但并不持久化，因为这些信息会在系统启动时根据数据节点信息重建。

DataNode 会主动周期性的运行一个 block 扫描器（scanner）通过比对 checksum 来检查 block 是否损坏。文件数据存储的可靠性依赖多副本保障，对于每一个 DataNode 节点而言，它只需保证自己存储的 block 是完整且无损坏的即可。

每个 DN 上都有一个名为 DataXceiverServer 的 TCP Serer 守护进程，通过 socket.accept() 接收客户端的读写请求。当一个读写请求到来时，DataXceiverServer 为每一个连接创建一个新线程和一个用以处理请求和数据交换的 DataXceiver 对象。（1.X 版本了，之后应该有更高性能的处理方法）

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606546.png" alt="image-20220901222355490" style="zoom:80%;" />

### DataStorage

负责管理与组织 Datanode 的磁盘存储空间，同时也负责管理存储空间的生命周期（包括升级、回滚、提交等操作）。在 HDFS Federation 架构中一个 Datanode 可以保存多个块池的数据块

### FsDataset

抽象了 Datanode 管理数据块的所有操作，例如创建数据块文件、维护数据块文件和校验和文件的对应关系等。

我们知道每个 Datanode 都可以配置若干个不同类型的存储目录来保存数据块（通过配置项 dfs.datanode.data.dir 配置），所以 HDFS 定义了 FSVolumeImpl 类来管理 Datanode 上单个存储目录保存的所有数据块，同时定义了 FSVolumeList 类来维护 Datanode 上所有 FSVolumeImpl 对象的引用。FsDataset 会通过 FSVolumeList 类 提供的功能管理 Datanode 上所有存储目录保存的数据块

### BlockPool Mgr

管理所有块池的接口类。我们知道 HDFS2.X 版本提供了 Federation 机制，也就是每个 HDFS 集群都可以配置多个命名空间，每个命名空 间在 Datanode 上都有一个与之对应的块池。一个 BlockPoolManager 对象会持有 多个 BPOfferService 对象的实例，每个 BPOfferService 对象都管理这个 Datanode 的一个块池。HDFS2.X 中除了引入 Federation 机制外，还引入了 HA 机制，每 个命名空间都可以定义两个 Namenode，一个作为 Active Namenode，一个作为 Standby Namenode 。所以每个 BPOfferService 对象又都会持有两个 BPServiceActor 对象，一个 BPServiceActor 对象对应于命名空间中的一个 Namenode，该对象负责向这个 Namenode 发送心跳、块汇报、缓存汇报以及增 量块汇报，并执行 Namenode 返回的名字节点指令（DatanodeCommand）

### Blcok Scanner

DataBlockScanner：一个独立的线程，周期性地扫描每个数据块并检查数据块的 校验和是否正常

DirectoryScanner：一个独立的线程，定时发起对磁盘数据块的扫描，对比内存 中元数据与实际磁盘存储数据块的差异，然后更新内存中的元数据，使之与磁 盘保存的数据块信息一致。

### 流式接口

HDFS 除了定义 RPC 调用接口外，还定义了流式接口，流式接口是 HDFS 中基于 TCP 或 者 HTTP 实现的接口。在 HDFS 中，流式接口包括了基于 TCP 的 DataTransferProtocol 接口， 以及 HA 架构中 Active Namenode 和 Standby Namenode 之间的 HTTP 接口。

HDFS 没有采用 Hadoop RPC 来实现 HDFS 文件的读写功能， 是因为 Hadoop RPC 框架的效率目前还不足以支撑超大文件的读写，而使用基于 TCP 的流式接口有利于批量处理数据，同时提高了数据的吞吐量。

#### DataTransferProtocol

DataTransferProtocol 接口定义了用于发送 DataTransferProtocol 请求的 Sender 类，以及用于响应 DataTransferProtocol 请求的 Receiver 类，Sender 类和 Receiver 类都实现了 DataTransferProtocol 接口。图 1-8 给出了 DataTransferProtocol 接口的一个调用示例。我们假设 DFSClient 发起了一个 DataTransferProtocol.readBlock()操作，那么 DFSClient 会调用 Sender 将这个请求序列化，并传 输给远端的 Receiver。远端的 Receiver 接收到这个请求后，会反序列化请求，然后调用代码 执行读取操作。

![image-20220831154213802](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606426.png)

#### AN 和 SN 间的 HTTP 接口

Namenode 会定期将文件系统的命名空间（文件目录树、文件/目录元信息）保存到一个 名叫 fsimage 的文件中，以防止 Namenode 掉电或者进程崩溃。但如果 Namenode 实时地将内存中的命名空间同步到 fsimage 文件中，将会非常地消耗资源且造成 Namenode 运行缓慢。所以 Namenode 会先将命名空间的修改操作保存在 editlog 文件中，然后定期合并 fsimage 和 editlog 文件。

2.X 中有了 SN 后，SN 会不断地将从 JournalNodes 读入的 edit log 文件与自己当前命名空间合并，所以 SN 只需要定期将自己的命名空间写入一个新的 fsimage 文件，并通过 HTTP 协议将这个 fsimage 传回给 AN 即可。

具体操作是 Standby Namenode 成功地将自己的命名空间写入新的 fsimage 文件后，就会向 Active Namenode 的 ImageServlet 发送 HTTP GET 请求/getimage?putimage=1。这个请求的 URL 中包括了新的 fsimage 文件的事务 ID，以及 Standby Namenode 用于下载的端口和 IP 地址。Active Namenode 接收到这个请求后，会根据 Standby Namenode 提供的信息向 Standby Namenode 的 ImageServlet 发起 HTTP GET 请求以下载 fsimage 文件。

> 为什么实时地写入 fsimage 不耗费资源而实时写入 edit log 性能更好？因为每次都要重新创建一个 fsimage 吗？不可以追加写吗？
>
> 如果 AN 与 SN 之间的网路异常呢？AN 拿不到最新的 fsimage 会有什么后果？AN 重启需要自己从 JournalNodes 中读取 edit log 并合并到内存命名空间，重启时间变长。

### Start & Stop

一个完整的 Datanode 启动操作会与 Namenode 进行 4 次交互。

1. 首先调用 versionRequest()与 Namenode 进行握手操作。主要是比对 DN 上的 HDFS 版本号、DN 存储上的块池 ID、文件系统 ID 和 集群 ID 与 NN 返回的是否一致；
2. 然后调用 registerDatanode()向 Namenode 注册当前的 Datanode。传递的信息有 DatanodeID、Datanode 的存储系统的布局版本号（layoutversion）、当前命名空间的 ID （namespaceId）、集群 ID（clusterId）、文件系统的创建时间（ctime）以及 Datanode 当前的软 件版本号（softwareVersion），名字节点会判断 Datanode 的软件版本号与 Namenode 的软件版 本号是否兼容，如果兼容则进行注册操作，并返回一个 DatanodeRegistration 对象供 Datanode 后续处理逻辑使用。
3. 接着调用 blockReport()汇报 Datanode 上存储的所有数据块，默认 6 小时执行一次，NN 据此建立数据块与数据节点之间的对应关系；
4. 最后调用 cacheReport()汇报 Datanode 缓存的所有数据块。默认 6 小时执行一次；

### Block Report

#### 第一次全量汇报

DN 启动与 NN 握手、注册后，processFirstBlockReport()

为了提高 HDFS 的启动速度，第一次的全量块汇报，Namenode 不会计算哪些元数据需要删除，不会计算无效副本，将这些处理都推迟到下一次块汇报时处理。

NN 将第一次全量块汇报的所有副本都加入 NN 内存中

#### 周期性全量汇报

每 6 个小时定期发送，processReport()

既然已经有了启动时的全量块汇报以定期发送的增量块汇报，为什么还需要周期性地发 送全量块汇报呢？这是为了在 Datanode 的增量块汇报发生异常，或者 Namenode 下发的复制 或者删除副本指令丢失等情况发生时，Namenode 能够获取 Datanode 保存的所有副本的信息 并执行相应的补救操作。

对于 Datanode 周期性的块汇报，processReport()方法会调用私有的 processReport()方法处 理。这个方法会调用 reportDiff()方法，将块汇报中的副本与当前 Namenode 内存中记录的副本状态做比对，然后产生 5 个操作队列

1. toAdd。上报副本与 Namenode 内存中记录的数据块有相同的时间戳以及长度，那么将上报副本添加到 toAdd 队列中。对于 toAdd 队列中的元素，调用 addStoredBlock() 方法将副本添加到 Namenode 内存中。对 addStoredBlock()方法的分析请参考添加副本小节。

2. toRemove。副本在 Namenode 内存中的 DatanodeStorageInfo 对象上存在，但是块汇报时并没有上报该副本，那么将副本添加到 toRemove 队列中。对于 toRemove 队 列中的元素，调用 removeStoredBlock()方法将数据块从 Namenode 内存中删除。对 removeStoredBlock()方法的分析请参考删除副本小节。

3.  toInvalidate。BlockManager 的 blocksMap 字段中没有保存上报副本的信息，那么 将上报副本添加到 toInvalidate 队列中。对于 toInvalidate 队列中的元素，调用 addToInvalidates()方法将该副本加入 BlockManager.invalidateBlocks 队列中，然后触发 Datanode 节点删除该副本。 

4. toCorrupt。上报副本的时间戳或者文件长度不正常，那么将上报副本添加到 corruptReplicas队列中。对于corruptReplicas队列中的元素，调用markBlockAsCorrupt() 方法处理。对 markBlockAsCorrupt()方法的分析请参考损坏副本的删除小节。 

    ![image-20220901170350920](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606814.png)

5. toUC。如果上报副本对应的数据块处于构建状态，则调用 addStoredBlockUnder Construction()方法构造一个 ReplicateUnderConstruction 对象，然后将该对象添加到 数据块对应的 BlockInfoUnderConstruction 对象的 replicas 队列中。

    ![image-20220901170406331](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606914.png)

#### 周期性增量汇报

心跳间隔的 100 倍，默认是 300s

如果是新添加的数据块（RECEIVED_BLOCK），则调用 addBlock() 方法处理添加请求；如果是删除的数据块（DELETED_BLOCK），则调用 removeStorageBlock() 修改数据块与存储这个数据块的数据节点存储的对应关系。

对于增量汇报中新添加的副本，可能是客户端通过输入流管道写入了一个副本，也有可能是 NN 发起的复制操作。这里由 addBlock() 方法处理增量块汇报中新添加的数据块， addBlock() 方法会更改 DatanodeDescriptor 上的 blockScheduled 计数，然后从 pendingReplications 中移除这个数据节点上该数据块的复制请求，最后调用 processAndHandleReportedBlock() 处理副本为提交状态（FINALIZED）的数据块副本。

在 HDFS HA 架构中，DN 的心跳信息、全量块汇报以及增量块汇报会同时发送到 SN 以及 AN。SN 处理全量块汇报时，可能出现命名空间还未与 AN 同步的情况，这时就需要将待处理副本暂时缓存起来，等待 SN 完全加载 editlog 并更新命名空间后再处理。

### HDFS Federation

## HDFS HA

NN HA 切换

在 Hadoop 2.X 之前，Namenode 是 HDFS 集群中可能发生单点故障的节点，即每个 HDFS 集群中只有一个 Namenode，一旦这个节点不可用，整个 HDFS 集群就将处于不可用状态。

HDFS 的高可用（High Availability，HA）方案就是为了解决上述问题而产生的。图 1-12 给出了一个 HA HDFS 集群，在 HA HDFS 集群中会同时运行两个 Namenode，一个作为 Active Namenode，一个作为 Standby Namenode。Active Namenode 的命名空间与 Standby Namenode 是实时同步的，所以当 Active Namenode 发生故障而停止服务时， Standby Namenode 可以立即切换为活动状态，而不影响 HDFS 集群的服务。 

为了使 Standby 节点与 Active 节点的状态能够同步一致，就要求两个 Namenode 的命名 空间一致并且数据块与数据节点之间的对应关系一致。对于命名空间的一致性，两个节点都 需要与一组独立运行的节点（JournalNodes，JNS）通信，当 Active Namenode 执行了修改命 名空间的操作时，它会定期将执行的操作记录在 editlog 中，并写入 JNS 的多数节点中。而 Standby Namenode 会一直监听 JNS 上 editlog 的变化，如果发现 editlog 有改动，Standby Namenode 就会读取 editlog 并与当前的命名空间合并。当发生了错误切换时，Standby 节点会 先保证已经从 JNS 上读取了所有的 editlog 并与命名空间合并，然后才会从 Standby 状态切换 为 Active 状态。通过这种机制，保证了 Active Namenode 与 Standby Namenode 之间命名空间 状态的一致性。而对于数据块与数据节点对应关系的一致性，则要求 HDFS 集群中的所有 Datanode 同时向这两个 Namenode 发送心跳以及块汇报信息，这样 Active Namenode 和 Standby Namenode 的数据块与数据节点之间的对应关系也就完全同步了。一旦发生故障，就可以马上切换，也就是热备。

HDFS 提供了 HA 管理命令（DFSHAAdmin）使得管理员可以手动执行主备切换，同时 还提供了自动 Failover 机制，该机制依赖于两个新增的网元：一个是 ZooKeeper 集群；一个 是 ZKFailoverController（org.apache.hadoop.ha.ZKFailoverController）。ZKFailoverController 会实时监控 Namenode 的 HA 状态，如果 Active Namenode 处于不可服务状态，那么 ZKFailoverController 会自动触发主备切换操作，无须管理员执行任何命令。

通过配置奇数个 JournalNode 来实现 NN 热备 HA 策略

在一个 HA HDFS 集群中，AN 和 SN 都会和 JournalNodes 守护进程通信。当 AN 执行任何修改命名空间的操作时，需要保证产生的 edit log 在一半以上的 JournalNode 上持久化。SN 异步地从 JournalNodes 中读取 edit log 文件然后合并更新到自己的命名空间中。一旦 AN 挂掉，SN 会在消化 edit log 之后切换成 Active 状态。

checkpoint 时机：1. 超过了配置的检查点操作时长 2. 从上一次 checkpoint 后发生的事务操作了配置

当满足 checkpoint 时机后，SN 会把自己的命名空间写入一个新的 fsimage，并通过 HTTP 将这个 fsimage 文件传回给 AN。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606629.png" alt="image-20220831171353053" style="zoom:80%;" />

## HDFS Failover

如果客户端在写文件时，数据流管道中的数据节点出现故障，则输出流会进行如下操作来进行故障恢复。 

1. 输出流中缓存的没有确认的数据包会重新加入发送队列，这种机制确保了数据节点 出现故障时不会丢失任何数据，所有的数据都是经过确认的。但是输出流会通过调 用 ClientProtocol.updateBlockForPipeline()方法为数据块申请一个新的时间戳，然后使 用这个新的时间戳重新建立数据流管道。这种机制保证了故障 Datanode 上的数据块 的时间戳会过期，然后在故障恢复之后，由于数据块的时间戳与 Namenode 元数据 中的不匹配而被删除，保证了集群中所有数据块的正确性。
2. 故障数据节点会从输入流管道中删除，然后输出流会通过调用 ClientProtocol.get AdditionalDatanode()方法通知 Namenode 分配新的数据节点到数据流管道中。接下来 输出流会将新分配的 Datanode 添加到数据流管道中，并使用新的时间戳重新建立数 据流管道。由于新添加的数据节点上并没有存储这个新的数据块，这时 HDFS 客户 端会通过 DataTransferProtocol 通知数据流管道中的一个 Datanode 复制这个数据块到新的 Datanode 上。
3. 数据流管道重新建立之后，输出流会调用 ClientProtocol.updatePipeline()更新 Namenode 中的元数据。至此，一个完整的故障恢复流程就完成了，客户端可以正常完成后续的写操作了。

HDFS 写入异常按不同角色出错可以分为三种：

### NN 故障

注意这里的先决条件是 NameNode 已经开始写入了，所以 NameNode 已经完成了对 DataNode 的分配，若写之前 NameNode 就挂了，整个 HDFS 是不可用的所以也无法开始写入。

流水线写入过程中，当一个 block 写完后需向 NameNode 报告其状态，这时 NameNode 挂了，状态报告失败，但不影响 DataNode 的流线工作，数据先被保存下来，但最后一步 Client 写完向 NameNode 请求关闭文件时会出错，由于 NameNode 的单点特性（如果有HA则没事），所以无法自动恢复，需人工介入恢复。

### Client 故障

当 Client 在写入过程中，自己挂了。由于 Client 在写文件之前需要向 NameNode 申请该文件的租约（lease），只有持有租约才允许写入，而且租约需要定期续约。所以当 Client 挂了后租约会超时，HDFS 在超时后会释放该文件的租约并关闭该文件，避免文件一直被这个挂掉的 Client 独占导致其他人不能写入。这个过程称为 lease recovery。

#### lease recovery

下面是 lease recovery 的处理流程：

1. NN 查找 Lease 信息，对于 lease 中的每个文件，找到他的最后一个 Block，从这个 Block 的 DN 列表中选一个作为 primary DN；
2. primary DN 从 NN 获取最新的时间戳，从每个 DN 中获取 Block 信息，用其中最小的 Block 长度和最新的时间戳来更新具有有效时间戳的 DN，然后通知 NN 更新结果；
3. NN 更新 BlockInfo 并从 lease 中删除这个文件，当该 lease 中的所有文件都删除之后删除这个 lease，最后提交修改的 EditLog。

#### block recovery

试想一个场景，当一个file正在被写入，突然，Client挂了，那么这个文件的最后的block 会没有完全写完，这个block在各个节点写入的 packet 的大小也不会一致，那么这种异常的场景就需要进行恢复。在租约恢复导致文件关闭之前，必须确保最后一个块的所有副本具有相同的长度，这个过程称为block recovery，这个过程只能在 lease recovery 过程中发起。

 lease recovery 的目的是当 Client 在写入过程中挂了后，经过一定的超时时间后，收回租约并关闭文件。但在收回租约关闭文件前，需要确保文件 block 的多个副本数据一致（分布式环境下很多异常情况都可能导致存储该block的多个数据节点副本不一致），若不一致就会进入 block recovery 过程进行恢复。

下面是block recovery 处理流程：

1. 获取包含文件最后一个 block 的所有 DataNodes。
2. 指定其中一个 DataNode 作为主导恢复的节点。
3. 主导节点向其他DataNode节点请求获得它们上面存储的该Block的 replica 信息。
4. 主导节点收集了包含该Block信息的所有DataNode节点上的 replica 信息后，就可以计算出那个节点上面的 replica 的长度是最小的。
5. 主导节点向其他节点发起更新，将各自 replica 更新为最小长度值（一般是pipeline流中的最后一个节点的replica的长度是最小的），保持各节点 replica 长度一致。
6. 所有 DataNode 都同步后，主导节点向 NameNode 报告更新一致后的最终结果。
7. NameNode 更新文件 block 元数据信息，收回该文件租约，并关闭文件。

> 为什么在多个副本中选择最小长度作为最终更新一致的标准？
>
> 写入流水线过程，如果 Client 挂掉导致写入中断后，对于流水线上的多个 DataNode 收到的数据在正常情况下应该是一致的。但在异常情况下，排在首位的收到的数据理论上最多，末位的最少，由于数据接收的确认是从末位按反方向传递到首位再到 Client 端。所以排在末位的 DataNode 上存储的数据都是实际已被确认的数据，而它上面的数据实际在不一致的情况也是最少的，所以算法里选择多个节点上最小的数据长度为标准来同步到一致状态。

### DN 故障

HDFS的写入操作不会失败，HDFS将尝试自行恢复，将挂了的 DataNode 从流水线中剔除，并尝试将数据写入其它DataNode，这个过程称为 pipeline recovery，当然如果选择的其它的DataNode写入也失败了，那么本次写入就会失败。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202209121606312.jpg" alt="img" style="zoom:50%;" />

如上图所示，pipeline 写入包括三个阶段：

1. pipeline setup：Client 发送一个写请求沿着 pipeline 传递下去，最后一个 DataNode 收到后发回一个确认消息。Client 收到确认后，pipeline 设置准备完毕，可以往里面发送数据了。
2. data streaming：Client 将一个 block 拆分为多个 packet 来发送（默认一个 block 64MB，太大所以需要拆分）。Client 持续往 pipeline 发送 packet，在收到 packet ack 之前允许发送 n 个 packet，n 就是 Client 的发送窗口大小（类似 TCP 滑动窗口）。
3. close：Client 在所有发出的 packet 都收到确认后发送一个 Close 请求， pipeline 上的 DataNode 收到 Close 后将相应 replica 修改为 FINALIZED 状态，并向 NameNode 发送 block 报告。NameNode 将根据报告的 FINALIZED 状态的 replica 数量是否达到最小副本要求来改变相应 block 状态为 COMPLETE。

任意一个阶段都有可能会发生异常，分别对应三种故障恢复：

#### pipeline setup recovery

1. 新写文件：Client 重新请求 NameNode 分配 block 和 DataNodes，重新设置 pipeline。
2. 追加文件：Client 从 pipeline 中移除出错的 DataNode，然后继续。

#### data streaming recovery

1. 当 pipeline 中的某个 DataNode 检测到写入磁盘出错（可能是磁盘故障或者网络故障等等），它自动退出 pipeline，关闭相关的 TCP 连接。
2. 当 Client 检测到 pipeline 有 DataNode 出错，先停止发送数据，并基于剩下正常的 DataNode 重新构建 pipeline 再继续发送数据。
3. Client 恢复发送数据后从没有收到确认的 packet 开始重发，其中有些 packet 前面的 DataNode 可能已经收过了，则忽略存储过程直接传递到下游节点。

#### close recovery

到了 close 阶段才出错，实际数据已经全部写入了 DataNodes 中，所以影响很小了。Client 依然根据剩下正常的 DataNode 重建 pipeline，让剩下的 DataNode 继续完成 close 阶段需要做的工作。

## 常见问题

1. 为什么 HDFS 中的块大小是 64/128 MB？

    HDFS 块的大小设置主要取决于磁盘传输速率。太小的话会增加寻址时间，程序一直在找块的开始位置；太大的话从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间，导致程序在处理这块数据时非常慢。

2. HDFS 中都有了全局读写锁、为什么还需要 Lease？

    1. 客户端每次执行写入时都需要向 NN 申请全局写锁，这样会增加不必要的网络通信开销；
    2. 假如某个客户端拿到互斥锁之后失去了和 NN 的联系，则可能会出现此客户端持有的互斥锁永不释放造成死锁的现象，从而导致其他客户端的操作被终止。

    客户端执行写操作时，

3. NameNode 内存瓶颈

    在 NN 整个内存对象里，占用空间最大的两个结构是 Namespace 和 BlocksMap。受 JVM 可管理内存上限等物理因素，180G 内存下，NameNode 服务上限的元数据量约 700M。

    当集群和数据均达到一定规模时，仅通过垂直扩展NameNode已不能很好的支持业务发展，可以考虑HDFS Federation方案实现对NameNode的水平扩展，在解决NameNode的内存问题的同时通过Federation可以达到良好的隔离性，不会因为单一应用压垮整集群。

