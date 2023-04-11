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
