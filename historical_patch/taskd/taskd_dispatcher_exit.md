### 从中所得

* 如果含有 Coroutine 的线程还未析构，且 Coroutine 传入函数是个类成员函数，但这个类对象已经被析构了，就会造成 coroutine 重新获取 CPU 时访问已析构对象的问题，所以需要在类对象析构之前保证当前的 Coroutine 都已经运行结束了。
* 如果 Coroutine 的传入参数是栈变量，那么可能出现上层函数已经执行完，这个栈变量已经被释放，但是 Couritine 重新获取 CPU 时访问已析构的栈对象的问题，所以最好改用在 co enter 前 new 需要的堆对象，并通过 defer 在 coroutine 执行完 delete。

### 问题所在

没有任何修改的情况下，单测 ReschedulePendingTask 失败

```
The failure case is:
[ RUN      ] TaskdTest.ReschedulePendingTask

=================================================================

==111==ERROR: AddressSanitizer: heap-use-after-free on address 0x631000e4c848 at pc 0x5593e237142b bp 0x7fd0fc889260 sp 0x7fd0fc889250

READ of size 8 at 0x631000e4c848 thread T1532 (zbs-task-dispat)

    #0 0x5593e237142a in zbs::task::ZkTaskDb::PutTask(zbs::task::Task const&) ../src/task/task_db.cc:109

    #1 0x5593e23870c5 in zbs::task::TaskDb::PutTask(zbs::task::Task const&, bool) ../src/task/task_db.cc:210

    #2 0x5593e23b9d49 in zbs::task::TaskDispatcher::ExecuteTask(zbs::task::Task*)
```

### 问题追因

诱因：当 TaskServer::StopDispatcher() ，存在这种情况——当前处于 TaskDispatcher::ExecuteTask Coroutine 让出 CPU 的时候，task_dispatcher_ 被 reset 了，但 dispatcher_thctx_ 还未 reset，当 ExecuteTask Coroutine 重新拿到 CPU 时，这时的 task_dispatcher_ 对象已被析构，this 指针不存在，导致访问失败。

对策：通过引入 in_execute_task_cnt_ 来记录当前 ExecuteTask Coroutine 个数，创建 coroutine 之间 ++，coroutine 执行完 --，当 in_execute_task_cnt_ 为 0 时才允许 TaskDispatcher 析构，这样就能保证 TaskDispatcher 之后不会有 couroutine 还来访问 TaskDispatcher 对象。并把 closed_ 的判断放在 CheckTask() 的 for 循环内，提供及时退出的效果。

触发了新的问题

诱因：TaskDispatcher::ExecuteTask 的形参是指针传入，而在 TaskDispatcher::CheckTask 中的 task 是一个局部变量（栈变量），它的生命周期在 CheckTask 执行完也就结束了，但是此时 ExecuteTask Coroutine 还在执行，而在其中的 client->CallMethod() 作为一个 grpc 方法，当 收到 rpc 对端响应后，会重新申请 CPU 并继续执行，此时如果访问 task 对象，就出现了访问一个已经析构对象的错误。

对策：给 ExecuteTask 传入堆变量，并在 Coroutine 结束时做析构

