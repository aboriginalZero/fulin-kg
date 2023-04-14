```shell
zbs-meta pextent read -o <output_file> <pextent_id> <offset> <length> 
zbs-chunk extent list 
```

显示 2023-04-07 14:58:47,878, ERROR, cmd.py:2967, write() argument must be str, not bytes

zbs-nfs 中难以查看 dir

https://blog.csdn.net/qq_24406903/article/details/118763610



metaDB 中的 Vtable 表存储 Volume -> Extent 的关联关系

对着 log 梳理一下 zbs 架构，角色在什么位置，哪些要持久化到 metaDB，关键数据结构的类型

CreateSession 这个过程 SessionMaster 做什么了

ZBS 中的一个 extent，读的同时不允许写？



目前 ZBS 仅支持以多副本方式存储数据，适用于对读写延迟比较敏感的业务场景。EC 相比副本占用更少的存储空间，同时提供与副本同等甚至更高的容错能力，其代价是更新或者故障数据恢复的性能略差（注：写不一定比副本差，虽然需要多一次读，但数据量变少了。看最终实现的效果）。EC 非常适合归档、备份等数据量较大且更新很少的业务，也适用于对延迟不敏感而对带宽敏感的业务。需注意，EC 的目标是为了保证完整性而非正确性，只可以用于恢复丢失的数据，而无法修复位翻转等数据正确性问题。需要有其他机制对数据篡改进行检测，例如 CheckSum 检测，被篡改的数据可以视为丢失，再通过 EC 做数据恢复。

ZBS 副本机制采用 Lease Owner + Generation 方式实现数据一致性。1. 通过 Lease Owner，所有的 IO 都被顺序化，客户端的多读者多写者模型被转化为单读者单写者模型；2. 通过 Generation 判断多个副本是否一致，当一个副本成功写入一个 IO 后，Generation 自增，每个 IO 携带当前的 Generation，Chunk 只有在 Generation 匹配时才允许 IO 处理。





FunctionalTest::SetUp()  --> new MiniCluster(kNumChunks); gtest 如何开启 VLOG DLOG 部分的日志



gtest系列之事件机制
“事件” 本质是框架给你提供了一个机会, 让你能在这样的几个机会来执行你自己定制的代码, 来给测试用例准备/清理数据。gtest提供了多种事件机制，总结一下gtest的事件一共有三种：
1、TestSuite事件
需要写一个类，继承testing::Test，然后实现两个静态方法：SetUpTestCase方法在第一个TestCase之前执行；TearDownTestCase方法在最后一个TestCase之后执行。
2、TestCase事件
是挂在每个案例执行前后的，需要实现的是SetUp方法和TearDown方法。SetUp方法在每个TestCase之前执行；TearDown方法在每个TestCase之后执行。
3、全局事件
要实现全局事件，必须写一个类，继承testing::Environment类，实现里面的SetUp和TearDown方法。SetUp方法在所有案例执行前执行；TearDown方法在所有案例执行后执行。
例如全局事件可以按照下列方式来使用：
除了要继承testing::Environment类，还要定义一个该全局环境的一个对象并将该对象添加到全局环境测试中去
原文链接：https://blog.csdn.net/ONEDAY_789/article/details/76718463
