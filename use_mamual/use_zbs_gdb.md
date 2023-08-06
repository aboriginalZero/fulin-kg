## GDB

GDB 是一款可以用来调试 C/C++/Java/Go 的调试工具，在类 Unix 系统中广泛使用。

* 下载：yum install devtoolset-7-gdb
* 开启 gdb 调试：. /opt/rh/devtoolset-7/enable
* gcc 编译时需要带上 -g 参数，才能保留调试信息

### ZBS 调试

1. 将源码打包发到待调试节点上并解压

    ```shell
    tar zcvf zbs.tar.gz zbs
    scp zbs.tar.gz smartx@172.20.134.135:/tmp
    ```

2. 在集群安装 debuginfo

    ```shell
    # 查看 zbs 版本
    rpm -qa zbs*
    # 进 17.20 用 locate 查看对应路径
    locate 5.5.0-rc7.0.release.git.g915e8af70.el7.SMTX.SERVER_SAN.x86_64
    # scp 到待调试节点
    scp /data/distros/repo/pub/smartxos/el7/smtx-zbs/x86_64/zbs-debuginfo-5.5.0-rc7.0.release.git.g915e8af70.el7.SMTX.SERVER_SAN.x86_64.rpm smartx@172.20.134.135:/tmp
    # -U（大写）选项的含义是：如果该软件没安装过则直接安装；若已经安装则升级至最新版本
    rpm -Uvh zbs-debuginfo-5.5.0-rc7.0.release.git.g915e8af70.el7.SMTX.SERVER_SAN.x86_64.rpm 
    # 主动 dump，若所需内存过大，dump meta leader 可能引发切主，将 core.xxx 都放在 /tmp
    gcore <zbs-chunkd-pid>
    ```

3. 查看 ChunkServer 和 MetaServer 的 PTR

    ```shell
    grep "CHUNK SERVER PTR" /var/log/zbs/zbs-chunkd*
    grep "services of meta" /var/log/zbs/zbs-metad*
    ```

4. 让 gdb 支持解析 stl 容器

    ```shell
    # 在 ~/.gdbinit 添加如下，其中路径为 find / -name "*libstdcxx*" 结果
    python
    import sys
    sys.path.insert(0, '/usr/share/gcc-4.8.2/python/libstdcxx')
    from v6.printers import register_libstdcxx_printers
    register_libstdcxx_printers (None)
    end
    ```

    wget [dbinit_stl_views](http://www.yolinux.com/TUTORIALS/src/dbinit_stl_views-1.03.txt) ，并将其内容追加到 ~/.gdbinit

    ```shell
    wget http://www.yolinux.com/TUTORIALS/src/dbinit_stl_views-1.03.txt
    cat dbinit_stl_views-1.03.txt >> ~/.gdbinit
    ```

    代码注释中有对应命令使用方法，比如 vector

    ```shell
    (gdb) pvector <my_vec> <idx>
    (gdb) pvector  ((zbs::meta::MetaServer)*0x562f6297c000)->context_->recover_manager->topo_distance_ 513
    
    (gdb) pmap <my_map> <key_type> <value_type> <key>
    ```

    但并不支持 std::array、std::unordered_map（问问其他同学）

5. 运行 gdb

    ```shell
    gdb /usr/sbin/zbs-chunkd core.xxx
    ```

    如果忘记加载符号表会有提示 missing 

6. 在 gdb 中执行命令

    ```shell
    # 加载 zbs 源码
    (gdb) source /tmp/zbs/src
    # 加载简易命令
    (gdb) source for-debug.py
    ```

7. 打印 zbs 变量

    ```shell
    (gdb) p ((zbs::chunk::ChunkServer)*0x561281a74000).meta_->lease_
    (gdb) p /r  foo			# /r 表示以不加载 python pretty-printers 的方式打印
    ```

### 启动调试

1. 跑 GTest 单测

    ```shell
    gdb --args ./build/src/zbs_test --gtest_filter=AccessTest.RecoverBase
    ```

2. 跑接受命令行输入的程序，在 run 时送入数据

    ```shell
    gdb main 
    run your_input_str
    ```

3. 跑 coredump，通过 bt 看错误栈

    ```shell
    gdb <file_name> <core_file_name>
    backtrace
    ```

    zbs 中的例子如下：

    * 找到 coredump 并复制到 /tmp：cp /var/crash/manager-thd.core.143575.1624880237.gz /tmp/
    * 在 /tmp 下解压：gunzip manager-thd.core.143575.1624880237.gz
    * 开始调试：gdb /usr/sbin/zbs-taskd /tmp/manager-thd.core.143575.1624880237
    * bt：查看 error stack
    * bt full：查看带有局部变量的 error stack
    * assemble：出错的汇编代码处

4. 跑正在运行的进程

    ```shell
    attach <your_PID>
    ```

    如果报错，切换到 root 用户下将/etc/sysctl.d/10-ptrace.conf中的 kernel.yama.ptrace_scope 修改为 0

    如果已运行程序没有调试信息，本地能够马上编译出带调试信息的 release

    ```shell
    gdb <your_exe_file> --pid <running_PID>
    ```

### 设置断点/观察点

1. 设置断点

    ```shell
    # 进入 gdb 界面，设置断点
    breakpoint <file_name_with_suffix:line_num> if <condition>
    breakpoint <func_name>
    # 需要对所有调用 print 函数都设置断点
    rbreak print*
    # 设置临时断点，只会在首次遇到才会在该处停下
    tbreak <func_name> 
    # 跳过多次设置断点，适用于调试循环
    ignore <bk_ID> cnt
    ```

2. 设置观察点

    ```shell
    # 设置观察点，当变量值被写时停下
    watch <var_name>
    # 当变量值被读时停下
    rwatch <>
    # 当变量值被读/写时停下
    awatch <>
    ```

3. 查看、删除观察点

    ```shell
    info breakpoints
    delete <breakpoint_ID>
    ```

### 单步调试

```shell
next 10		# 继续执行下 10 条语句
step		# 单步跟踪到函数内部
finish		# 从一个函数内跳出
continue	# 继续运行，停在下一个断点处

skip function <func_name>
skip delete <breakpoint_ID>
```

### 查看变量

```shell
# run 的时候停在断点处，且还未进入，通过 list 查看源码
list <left>,<right>
list <file_name_with_suffix:line_num>

# 打印指定变量
print '<file_name>'::<var_name>
# 指针解引用，打印以 buf 为起始地址，长度为 i 的字符串
print *buf@i
# 打印链表
print *node		# node 节点的值
print *$.next	# node->next 节点的值
```

### 多线程调试

```shell
# 查看所有线程 
info threads
### 打印内容
  3 Thread 0x688a700 (LWP 20568)  0xe8e0 in sigprocmask () from /lib64/libc.so.6 
  2 Thread 0x708b700 (LWP 20567)  0x8a3d in nanosleep () from /lib64/libc.so.6
* 1 Thread 0x7fe5720 (LWP 20564)  main (argc=1, argv=0x7fffffffe628) at multithreads.cpp:39
# 切换 2 号线程
thread 2
# 锁定线程，即只有当前的线程能够执行
set scheduler-locking on
# 让所有线程一起运行
set sheduler-locking off
# 执行命令，如让所有的线程打印调用栈信息
thread apply all bt
```

[参考1](https://zhuanlan.zhihu.com/p/74897601)，[参考2](https://cloud.tencent.com/developer/article/1142947)

待补充

https://coolshell.cn/articles/3643.html

https://blog.csdn.net/haoel/category_9197.html

gdb 使用方法，https://wizardforcel.gitbooks.io/100-gdb-tips/content/info-function.html