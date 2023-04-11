### 任务要求

> Zbs-14721：taskd: wait for the task done before exiting

Taskd 在退出时，如果没有清理本地的任务，则尚在进行中的任务在异步 IO 结束时可能会访问已经释放的内存对象发生错误。但是等待所有的任务都完成，与现实中出现的 Taskd 可能会崩溃的状态不一定匹配。也无法满足单测中验证任务异常中断后重新调度的场景。因此可能的调整方案为：

1. Taskd 退出时对所有运行中的任务赋予一个可被重新调度的错误，让 Dispatcher 主动感知这个状态；
2. Taskd 退出时仅终止本地的任务；

### 从中所得

* 利用 unordered_map （deque 无法保证多线程情况下 push、pop 一一对应）来收集子线程的 ctx ，通过它的 size == 0 来保证子线程执行结束才在主线程中释放相关资源
* defer 的使用，体现了 RAII 思想

### 残余问题

* 没找到容灾备份的命令行指令，没法在自己的集群上验证，仅过了单测
* 对 google rpc 其实还不熟悉，通过 showtask 来跟踪一下吧
* Taskd 停止服务时，（dispatcher 通过 watch dog 路径的方式来感知 runner 的上线） dispatcher 是如何感知到并将 task 重新分配的？找一下相关的代码在哪
* quorumcluster 是所有 server （metaserver、ntpserver、taskserver）共享一个还是分别拥有一个

### 需求分析

这个需求的理解如下：

当一个 runner 挂掉，在它上面开启的异步 IO 线程还会继续运行，当它完成 IO 事件之后，有可能会访问它所在 runner 上已经释放的对象（虽然目前我没找到，但是存在这种可能），为了避免这种情况的出现，要在对 runner 终止之前让其上面开启的子线程（不论是异步 IO 线程还是什么）先终止，即在 runner 析构之前，要让它开启的异步 IO 线程先析构。在 task-center 中，目前只有 CopyVolume 是异步线程，因为要保证在 TaskCopyVolumeRunner 析构之前，在它上面的所有线程、协程都停止了。这就是这个需求要做的事。

FunctionalTest.CopyVolumeTaskWithRunnerFailed 这个单测做的事是模拟某个 runner 挂掉，验证 task 能否正常转移到其他的 runner 并继续执行下去。在原代码的语境下，如果一个 runner 挂掉，Dispatcher 会得到 ZOO_CHILD_EVENT 事件通知，它会主动去把这个奔溃 runner 上面的 task 重新调度（对应代码 TaskDispatcher::TaskWatcher 中对 task.task set_last_failed_ms() 之后，schedule(task) ），所以单测能过。

fengli 的做法，是在 CopyExtent 中等待每一个 CopyBlock 完成，但是由于 CopyExtent 的返回值是 OK，所以造成 DoCopy 中的循环一直继续下去，最终的结果是整个 volume 中的 block 都 copy 完了才让 TaskCopyVolumeRunner 析构，由于 task  被执行完了，不存在被重新调度的情况，所以单测过不了。

> [fengli 的做法](http://gerrit.smartx.com/c/zbs/+/25368/1/src/task/tasks/task_copy_volume_service.cc#b93)，维持一个变量 inflight_ ，每一个 coroutine 还在执行的时候 +1，co->Yield() 的时候 -1，在 TaskCopyVolumeRunner 析构的时候先让这个 runner 中所有的 zbs_client Stop()，然后在 inflight_ == 0 时，释放 zbs_client 和 meta（这里的meta 是因为 CopyVolume 这个任务特殊吧，会跟meta-cluster 交互）。

### 解决思路

我的第一版 patch 思路：在 fengli 代码的基础上引入 suspend_ ，能够做到遇到 runner crash（调用析构函数）的时候， 能够在当前 CopyBlock 结束之后（仅完成一个 block 的读写），整个任务及时结束，因为可能存在当前 CopyBlock 做完正好是整个 task 任务结束的情况，所以要在 TaskCopyVolumeRunner::DoCopy 函数中加以判断，返回一个适当的 Status，如 ERunnerInterrupt 。这个解决方案其实还是依赖 ZK 来发现挂掉的  runner，只不过能够保证异步 IO 子线程不会在完成 IO 之后去访问已经释放的 runner 的某些对象，能安全一些。

但其实第一版没有考虑原有的框架设计，写法面向过程；

第二版从 ctx->control 的角度出发，通过收集各 docopy 的 ctx，在析构函数中对其设置 cancelreason = ERUNNERINTERRUPT，借助 defer 实现资源释放。但是由于 zbs_client 在 等待 docopy 之后才 stop，还不够高效；

第三版，先将 zbs_client stop，即先断开 IO 通道，避免出现 copyblock 都停了，但是 IO 通道没断开仍在做数据传输，也就是效率、安全性更高。但是由于原先写法，在 copy extent 的状态码 != OK 的时候，直接 return，所以造成在析构函数中设置的 cacelreason 没有放入状态栈，修改方法是在判断是否要 return 之前，先判断是否 Cancel 

第四版，由于 zbs_client 之前的 stop 实现有问题，具体指的是 IO 没有全部断开就会退出，所以会有存留的 inflate IO。我们先 stop 只能保证正在执行 docopy 的 IO 关掉，而之后可能还会有 docopy 开启，  后面的 processing_ctxs_ cancel 和 sleep 只能保证所有的 doCopy 做完，但是在最后 delete 的时候是无法保证所有的 IO 通道都关掉，因此存在 BUG。

第五版，先对 processing_ctxs_ cancel 和 sleep，保证所有的 doCopy 做完，然后用 zbs_client->Stop 来保证所有的 IO 通道都关闭，最后再 delete，整个过程十分清楚。

### 过程疑问

尚在进行中的任务在异步 IO 结束时可能会访问已经释放的内存对象发生错误？什么情况会访问？我看 task_copy_volume_service.cc 中没找到会访问到已经释放的内存对象，所以不知道怎么构造单测。

可被重新调度的错误是不是将 task 设置为 paused，对应的方法为 TaskDispatcher::PauseRunningTask(Task* task)，目前仅 CopyVolume Task 可以被 paused，在 TaskCopyVolumeRunner 的析构函数中加上对正在running 的 task 设置为 paused 的操作。

目前单测 TEST_F(FunctionalTest, CopyVolumeTaskWithRunnerFailed) 无法真实的模拟 runner crash 的情景，因为所有单测是一个进程，我们是用多线程的方式来模拟实际环境中的多进程。 因而无法自己写一个单测构造这种失败场景

```
cd /home/abori/zbs/build
export GTEST_REPEAT=100
./src/zbs_test --gtest_filter="FunctionalTest.CopyVolumeTaskWithRunnerFailed"
```

taskd 在代码中具体指的是什么？单测中的 TaskdServer，其 Start 时，调用 StartTaskServer，其实就是一个个 runner

代码中 taskd 在哪退出的？这个单测运行结束时，调用的析构。

taskd 什么时候退出？崩溃、任务异常中断后重新调度

taskd 怎么查所有运行中的任务？目前也就考虑到 CopyVolumeTask

```cpp
task_client->ShowTask(response.id(), &task)
if(task.state() == task::IN_PROGRESS){
	// 代表这个 task 正在运行中
}
```

taskd 怎么对任务赋予一个可被重新调度的错误？首先该任务需要处于未完成且未分配的状态，将其设置为可被重新调度（具体对应哪个状态呢，paused？），并调用相应的服务调度函数，进行调度。目前单测中只是server->StopTaskServer(previous_runner_index); 没有对 task 显式指定一个可被重新调度的错误，这边应该看一下，如果 runner 被 stop，默认是怎么处理运行在它上面的 task 

taskd 怎么终止本地的任务？这句话有问题吧？只能终止本地的 runner 吧，task 会被 dispacher 调度到别的 runner 吧  

runner 的生命周期是怎样的？什么时候构造？什么时候析构？一个 runner 为什么有多个 zbs-client ？可以用StartTaskService() 起一个 runner，用 server->StopTaskServer(i); 停止一个 runner
