### 任务要求

> zbs-11039：次级网络无法优雅退出

目前 Task Center 在次级网络断开时会主动的 Crash，生成大量的 coredump 文件，需要调整为仅优雅退出或终止服务即可。不需要产生无谓的 Crash 文件。而且，日志中出现了thread join and kill 自己，需要避免。调整的方式可以为：

- 和 meta 一样仅终止服务，不退出程序（尽量选择这个方式）；
- 退出程序，但是不产生 Crash，需要考虑退出频率，避免频繁重启导致 systemd 放弃继续拉起服务；

### 从中所得

* taskd 模块[流程](https://www.processon.com/outline/60a324390791291571182d72)

* 如果主线程想要在子线程某个状态变更之后执行指定的操作，可以通过异步回调的方式实现。传入一个类方法作为回调函数，通过持有该类对象的 this 指针，当子线程某个状态变更时，回调在主线程中的类方法。

* C++ 的写法是变量用时申请，代码组织上，声明与使用要尽量紧凑。一般情况下的函数传参用 const && reference 是最好的，避免修改、拷贝。如果能够十分明确对象的生命周期，手动 new、delete 就好，不需要用智能指针。包括 mutexGuard 也是如此，非常明确的情况下可以手动 lock、unlock。

* 执行 ::exit(1)，表明异常退出所在进程，而不是线程，并留下 coredump。

  exit 和 return 的区别

  1. return是函数的退出（返回），返回函数值，而 exit 是进程的退出，返回 exit code；
  2. return是 C 语言提供的，exit 是操作系统提供的（或者函数库中给出的）；
  3. Main 函数中的 return 实际上调用了 exit 函数，exit 函数运行时会先执行由 atexit() 登记的函数，然后释放当前进程申请的内存空间、同时刷新所有输出流、关闭所有打开的流、关闭通过标准 I/O 函数 tmpfile() 创建的临时文件等。当需要强制终止进程时可以使用 exit(1)，非 0 为非正常退出。

* 在线程中使用协程来执行需要异步的操作，不会造成线程阻塞。

  * 若要线程仅执行一次异步任务可以通过往ThreadContext 中 Sched 一个协程来实现，如`thctx_->Sched(Coroutine::Create([=] (void*) {func();}))`
  * 若要执行异步定时任务可以通过定义一个 timerhandle 成员变量实现，如`vip_th_(loop, [=]() {func();}, interval_ms, true)`

* 变量的严格定义

  * 会被跨线程访问的变量用`std::atomic<type> `声明，用以保证线程安全；
  * 会被信号处理函数访问的变量用`volatile`修饰，`volatile`关键字声明的变量，每次访问时都必须从内存中取出值，否则可能由于编译器的优化，仅从 CPU 寄存器中取值。

* 由于 task-server 或继承或聚合太多其他的类对象，很难手动把资源释放干净，比如对 zk 的关闭与重启、对 quorum_cluster_ 的 `loop_.Loop()/Unloop()`以及一个恰当的析构顺序。所幸提供了一个 Stop 方法，不需要进程退出也能够做到 task-server 重启。顺便提一嘴，如果需要统计进程退出次数，可以通过 mmap 开共享内存（参考 ntpd 中的实现），用以记录退出次数。

* 对信号的响应，起一个协程来 Stop。因为信号是作用于进程的，可能被其中的任意一个线程收到，用 coroutine 塞进 EventLoop （EventLoop 是属于当前进程而不是任意一个线程）可以保证任意一个线程执行的退出操作都可以作用到这个进程上。但是呢，信号处理函数越简单越好，比如不要打日志，因为可能会调用不可重入的 glibc 或 syscall 函数，跟当前线程的信号中断点死锁；不要析构资源，因为可能主线程中正在析构该资源到一半，半析构状态下的重新释放会出现无法预知的错误。

* zbs 中的 Sync 对象用以把异步操作变同步

  ```c++
  Sync sync;
  thctx_->Sched(Coroutine::Create([=, &sync](void*) {
  	...			// 具体的异步操作  
    sync.Run();
  }));
  
  sync.WaitAndReset();	// 等待直到异步操作完成才执行之后的语句
  ```

  其具体实现

  ```c++
  class Sync : public Closure {
    public:
      explicit Sync() {}
      void WaitAndReset()  {
          std::unique_lock<std::mutex> lk(m_);
          cv_.wait(lk, [this]{ return done_; });
          done_ = false;
      }
      void Run() { Done(); }
      void Done() {
          std::unique_lock<std::mutex> lk(m_);
          done_ = true;
          // Sync may be destroyed if lk is unlocked. The notify on a destoyed cv
          // would block forever. So we call notify_one before unlock.
          cv_.notify_one();
          lk.unlock();
      }
      bool IsDone() {
          std::unique_lock<std::mutex> lk(m_);
          return done_;
      }
      Callable GenerateCallback() { return [this]() { Done(); }; }
  
    private:
      bool done_ = false;
      std::mutex m_;
      std::condition_variable cv_;
  };
  ```

### 残余问题

* 分布式系统中要考虑：

  * 容错：有节点挂掉，任务需要调度出去
  * 扩容：有新节点加入，需要尽快被利用起来
  * 负载均衡：按需调度任务到不同节点
  * 暂时下线：节点要临时下线修复，不接受新任务，等旧任务执行完下线

  对应到 task-center，这些对应的代码写法，zbs 中是借助 zk 来感知节点的上下线，每个 runner 持有 zk 成员变量，构造时在 zk 特定目录下 set，析构时 delete 所在目录。而在 dispatcher 中对 zk 的节点注册目录 setWatcher，这样就能够接受节点的上下线通知，及时调度与之相关的任务。

  因为上层业务严重依赖于 zk，因此需要考虑与 zk 相关的各种异常，比如节点如果失去 zk session，这时候怎么处理？

* 在文档「ZBS Task Center设计」中提到，当 Runner 宕机或失去 zk session 使得服务中断，可以考虑增加一个 interrupt 的 task 状态，目前对失去 zk session 的情况好像还没处理？

* zk 中保存所有的任务数据库，即任务队列，完成任务和进行中任务都存放到同一个队列中，由 dispatcher 通过扫描数据库发现未完成的任务队列。更新 zk 任务队列时也会更新 taskdb 中对应的 task 吗？会的。

  taskdb 是用于给 client 通过 taskid 查询，但是文档中有一处说明，在失败产生时，通过从 TaskDB 中获取对应的 Task 对象进行重新调度，这时用 zk 的任务队列不就好了？

  是可以用 zk，但是没必要占用 zk ，因为 zk 是被所有组件依赖，所以通过引入 taskdb 来减轻它的负担。

  文档还有一句话表明，dispatcher 在启动后，会从 zk 任务队列中加载所有的任务状态，从而得到所有未完成的任务并开始 watch。刚启动的 dispatcher，taskdb 中应该没有任何东西，只能从 zk 任务队列中加载。

  

  dispatcher 更新 task 时在 zk 任务队列和 taskdb 中都更新，但是在 dispatcher/ runner

  Taskdb 的 PutTask 包含两个操作，

  1. 调用 zk_db_ 的 PutTask，在 zk 中根据 task_id 注册临时节点，存放的是序列化的 task
  2. 更新本地 leveldb 缓存 

  还没有搞清楚 zk 这个任务队列和 taskdb、leveldb 分别是干啥的？

  Task 的生命周期较短，在完成之后会被定期清理回收，无需持久化存储，数据总量也较低，在 ZK 的负担范围之内，并且任务状态的变化监测可以利用 ZK 的监听机制很好的实现。因此，Task DB 使用 ZK 作为数据承载单元，Level DB 仅作为本地查询缓存。

  所有的 Task 在 ZK 中均在特定目录（/zbs/taskd/db/ ) 下记录，Key 为 Task ID。存储内容为序列化之后的 Task 对象。每条记录可能还存在孩子节点，记录当前任务的 runner。

  Runner 因为仅接受 Dispatcher 的命令，不需要本地 Cache 用于对外提供查询接口，仅加载 ZK Task DB。Dispatcher 加载完整的 Task DB，其中 Level DB Cache 部分每次加载均从 ZK 中重建即可，无需额外的同步机制。

  dispatcher 在 ExecuteTask 的时候，变更 task 的 runner_addr、schedule_time 之后，需要更新 task_db 并对 zk 中的这个 task set Watch，然后才去调用 task client 中对应的方法，比如 SetRunTime（这里为啥没有 CopyVolume，顺着这个看代码）


### 解决思路

如果仅仅通过 VIPServer::CleanVIP()、TaskDispatcher::Close() 终止 dispatcher 服务，不仅没有回收相关资源，更重要的是，没有触发 ZK 的 leader 重新选举，会影响集群对内 task 调度、对外提供 VIP 服务。

追踪代码发现 quorum_server 提供的 QuorumServer::Stop 方法能够令当前节点主动退出集群，引发 leader 转移，且 systemd 会把它主动拉起，重新进入集群。于是在 dispatcher thread 中提供一个判断次级网络是否异常的接口供主线程调用，当发现异常时在主线程中 Stop ，而不是在子线程中粗暴地 exit 引起不必要的 coredump。

但是主线程如果只是忙等子线程判断次级网络会造成阻塞，进而影响主线程后续操作（所以第一版 patch 过不了单测），可以让 taskserver 借助成员变量 quorum_cluster 的 loop 来创建一个 TimerHandle，每一秒调用一次对应接口，当次级网络异常时及时 Stop。参考 ntp_server 的实现，使得 task_server 能够保证除了 SIGINT/ SIGTERM 信号传入，taks_server 会一直运行，避免进程退出再由 system 重新拉起的繁琐过程。

上述在 tasksever 中主动轮询的机制可能实现针对 taskserver 的优雅退出，但是考虑到无法优雅退出问题在所有 zbs 组件如 ntpd、chunkd、metad 中都存在，所以希望能做成一个比较通用的方式。从有效管理 task server 生命角度出发，方案如下：

在主线程中创建 task server 并初始化，由于 task server 自带 eventloop，所以通过开启一个Thread 而不是 ThreadContext 的方式将 task server run 起来。至于对 task server 的 stop，既可以由 task server 自己停下，也可以由主线程在 signal stop set true 时调用 stop。stop 也就是退出操作，之后需要对 task server 做析构，在析构之前，通过 join 确保 task server 已经完整退出。整个过程下来，task server 只需要暴露 new、initialize、run、stop、delete 方法给主线程，主线程无法干涉它具体的业务处理。一个细节是，run 和 stop 可能在不同的线程中调用，多线程访问同一变量需要加锁保护

### 需求分析

Chongyu Zhang 的描述如下 ：TaskServer 运行在 main thread，dispatcher 运行在 dispatcher thread。当 dispatcher 发现 data channel 不可用时，直接在 dispatcher thread 中使得 TaskServer 退出，触发 TaskServer 的 default destructor。然后触发了 dispatcher thread 的 ThreadContext 的 destructor，去  join dispatcher thread，然后 kill dispatcher thread。理想情况，上述退出流程应该发生在main thread中的，由main thread 去 join and kill dispatcher thread。现在，直接在dispatcher thread中，触发了上述退出流程，所以导致dispatcher thread join and kill itself。 

可以通过在 leader 节点上 systemctl restart zbs-chunkd 复现 data channel 不可用的场景。退出日志如下：

```c++
I0118 20:17:43.026106  4351 quorum_cluster.cc:698] Ger priority config: 
I0118 20:17:43.028012  4351 quorum_cluster.cc:518] [ROLE CHANGE] I am the leader now.  seq is 879
I0118 20:17:43.028084  4351 quorum_server.cc:59] Get quorum cluster event: ROLE_CHANGED
W0118 20:17:43.028105  4351 quorum_server.cc:67] Role changed.
I0118 20:17:43.082680  4351 quorum_server.cc:59] Get quorum cluster event: Members_Changed
I0118 20:17:43.084945  4351 quorum_server.cc:78] Members changed.
I0118 20:17:43.085014  5096 thread.cc:114] Set thread name to: zbs-task-dispat
I0118 20:17:43.085063  5096 task_server.cc:235] Dispatcher starts at 10.0.0.97:10600
I0118 20:17:46.222466  5096 zookeeper.cc:151] Initialize zookeeper: zoo hosts: 10.0.0.97:2181,10.0.0.98:2181,10.0.0.99:2181 kSessiontimeout: 6000
I0118 20:17:46.293695  5096 zookeeper.cc:180] Initialize zookeeper done: client_id_: 0000000000000000 session_timeout: 6000 local session timeout_: 3600
W0118 20:17:46.318356  5096 zookeeper.cc:226] zookeeper session event:  old state: CONNECTING_STATE new state: CONNECTED_STATE
W0118 20:17:46.318444  5096 zookeeper.cc:205] Stop the session timeout timer.
I0118 20:17:46.400532  5096 zookeeper.cc:136] Destructing zookeeper.
I0118 20:17:46.405977  5096 zookeeper.cc:727] Start closing the zookeeper session.
I0118 20:17:46.412595  5096 zookeeper.cc:734] Finish closing the zookeeper session.
I0118 20:17:46.412636  5096 zookeeper.cc:737] zookeeper closed.
W0118 20:17:46.412655  5096 zookeeper.cc:205] Stop the session timeout timer.
I0118 20:17:46.412673  5096 zookeeper.cc:147] Done destructing zookeeper.
I0118 20:17:46.416054  5096 task_dispatcher.cc:185] Local secondary data channel was invalid, exit
E0118 20:17:46.417596  5096 thread.cc:301] Error occurs when call join thread:  timeout: 0 Ret_val = 35 : Resource deadlock avoided
W0118 20:17:46.422399  5096 thread.cc:199] To kill a thread, thread_id: 139811618502400 tid: 5096
```

追踪代码运行流程，可以发现在 CheckVIP 中，除了 ConfigVIP、SyncVIP ，还会 CheckSecondaryDataChannel，其作用机制为：

* Chunk Server 定期通过 Ping 的方式检查集群中与其他 Chunk 节点的链路状态，当与所有其他节点都失去链接时，判定自身次级网络为失效；

* VIP 服务通过 RPC 利用同机部署的 Chunk Server 的接入网络健康状态作为自身的接入网络健康状态。当发现健康状态为异常（`sec_valid == "false"`）时，会主动 CleanVIP 并退出（`::exit(1)`）以让 ZK 重新选举新的节点做 leader ，重新配置 ConfigVIP

### 过程疑问

* 把相关组件的生命周期补充（主要是析构部分），对这个模块有完整的了解。当前 task_server 失去 leader 角色时，stopdispatcher；当 role change 时，stop runner 。在 dispatcher / runner Initialize 失败的话，会触发 task_server 的 stop。NtpServer、TaskServer 的 Stop() 都是借助成员变量 QuorumServer 中的 quorum_cluster_ 触发 QuorumClusterEvent::STOPPED，于是开始释放资源以及 `loop_.Unloop();`，对应的是 QuorumCluster::Run() 中的 Start() 中的 `loop.Loop();`

* 这里不用 exit，只是置 bool，这个函数仍可能被再次执行？上面执行了 CleanVIP()，会不会有一些代码依赖 VIP 并且执行到一半，继续执行会有诡异的情况出现？可能会有，但是如果依赖重新启动的时候 CleanVIP ，也会存在短时间内 IP 地址冲突的现象，所以感觉这个地方加不加 CleanVIP 都会有问题

* 用 RAII 的方式使用锁。类中声明 CoMutex 为私有成员变量，在类成员方法中

  ```c++
  {
      CoLockGuard l(&co_mutex_);
      ...
  }
  ```

* 对 leader 中设置 VIP 导致的出错，用 stop 退出而不是 exit(1)？追了一遍 leader 中对 VIP 的操作，所有可能返回的 Error 包括 EBadRequest、ENoutFound、 ENetLink、EARPBroadCastFailed、EInterface、ESocket。其中，EBadRequest 表示用户设定的 VIP 地址不合理；ENotFound 表示没有匹配的子网路由以及没有设置默认路由导致的路由匹配失败；其他是跟网络相关的 Error。我认为 EBadRequest 和 ENotFound 导致的 Error 一般是由于用户设定 VIP 不当造成的，可以用 Stop 的方式退出，不留 coredump，会有对应的 Log。 

* 用 objdump 反汇编查看 task_main.cc 中的 stop 有没有被优化，以决定需不需要加 volatile？能够看到汇编代码中用 volatile 修饰的会从内存读取而不是直接从寄存器读。

* systemd 是怎么将退出的节点重新拉回到集群，对应代码在哪？systemd 作为 Linux 下的第一个进程管理所有的守护进程，如 zbs-taskd，在`/etc/system/`中 zks-taskd 的 START 字段是 on_failure，所以会将退出的节点重新拉回到集群。

* 参考 ntpd 实现，它里面啥时候主动放弃 leader？当 follower 发现自己的时间比 leader 快 5 分钟以上，就通过 NotifyLeader() 更改 / 新建 zk 中的 leader 节点，leader 节点在 watcher() 函数中有对 ZOO_CREATED_EVENT 和 ZOO_CHANGED_EVENT 的事件处理，其中就有主动放弃 leader 的逻辑

* 什么叫次级网络？与集群内的其他节点链接不上就可以判定为次级网络链接失效，所以这个次级网络可以理解存储网络吗？集群内的管理网络？是集群内的管理网络，因为是靠管理网络对外提供 VIP 服务。

* 这个任务可以抽象成：主线程想要在子线程某个状态变更之后执行指定的操作？除了目前的做法，还有什么？通过异步回调实现的话，好像回调函数还是在子线程中执行？对，回调函数还是在子线程中执行，但是如果传入的回调函数是某个类方法，通过持有该类对象的 this 指针，一个线程可以改变在另一个线程中的对象

  ```c++
  void callback() {
      ...
  }
  void AsyncFunc(..., callback) {
      // do async operator
      ...
      callback();
  }
  int main() {
      AsyncFunc(..., callback);
  }
  ```

* ::exit(1) 对应什么语义，结束当前进程吗，为何会在 dispatcher thread 中使得 TaskServer 退出？tid 5096 发现的 data channel不可用，调用 ::exit(1)，应该在当前线程 5096 结束才是，为啥会导致在 tid 4351 中创建的 taskserver 退出？因为 exit 结束当前进程，所以要去回收所有该进程中所有线程中的资源，造成线程死锁

* Meta 在代码哪里终止服务而不退出程序，看哪个文件的哪个函数？搜 stopmetaserver 函数

* 如何复现 data channel 不可用的场景，复现错误案例？目前的做法是在 leader 节点上 systemctl restart zbs-chunkd