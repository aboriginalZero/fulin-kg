## GDB

GDB 是一款可以用来调试 C/C++/Java/Go 的调试工具，在类 Unix 系统中广泛使用。

* 下载：yum install devtoolset-10-gdb
* 开启 gdb 调试：. /opt/rh/devtoolset-10/enable
* gcc 编译时需要带上 -g 参数，才能保留调试信息

### ZBS 调试

1. 在测试集群待调试节点安装 debuginfo

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
    grep "Starting the services of meta" /var/log/zbs/zbs-metad*
    ```

4. （选做）支持查看 stl 容器简易命令

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
    
    支持的容器不包括 std::array、std::unordered_map
    
    如果将容器所有内容都打印到日志中再去检索倒是不需要这么做。
    
4. 运行 gdb

    ```shell
    /opt/rh/devtoolset-10/root/bin/gdb /usr/sbin/zbs-chunkd core.xxx | tee /tmp/your_gdb.log
    /opt/rh/devtoolset-10/root/bin/gdb -c /home/core/rpc-server.core.3219.1692671991 /usr/sbin/zbs-metad
    (gdb) set height 0							# 多行输出时会全部输出
    (gdb) set print elements 0			# 多列输出时不会有默认的 200 个元素限制
    ```

    如果忘记加载符号表会有提示 missing，另外将所有 gdb 输出也重定向到 /tmp/your_gdb.log 中方便查询。

    如果只想保存指定部分日志的话

    ```shell
    (gdb) set logging file /tmp/your_gdb.log
    (gdb) set logging on
    (gdb) set height 0
    (gdb) set print elements 0	
    (gdb) do what you want...
    (gdb) set logging off
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

### 典型调试过程

1. 用户提供了一个 coredump 和集群收集日志

2. 起一个 zbs-buildtime 的 docker，从 17.20 上下载、安装对应的 zbs 和对应的 debuginfo 的 rpm

   ```shell
   wget zbs-4.0.14-rc3.0.release.git.gdb25d78fa.el7.SMTX.HCI.x86_64.rpm
   # 解压并找到这个 coredump 
   rpm2cpio xxx.rpm | cpio -div
   # 设置正在调试的可执行文件
   (gdb) file zbs-metad
   
   wget zbs-debuginfo-4.0.14-rc3.0.release.git.gdb25d78fa.el7.SMTX.HCI.x86_64.rpm
   # 安装 debuginfo 包以正确显示符号表
   rpm -Uvh zbs-debuginfo-4.0.14-rc3.0.release.git.gdb25d78fa.el7.SMTX.HCI.x86_64.rpm
   
   # 设置源文件搜索路径
   docker cp /home/code/zbs2 <docker_id>:/path
   (gdb) directory /path
   ```

3. 如果符号还是不能正常显示，需要查看 glibc 版本是否一致（在用户集群收集日志里的 report_node_info_ip 中会显示集群使用的 glibc 版本）

   若要更换低版本的 glibc，需要找到对应的 [common](https://mirrors.bfsu.edu.cn/centos-vault/7.2.1511/os/x86_64/Packages/glibc-2.17-105.el7.x86_64.rpm) 和 [debuginfo](http://ftp.scientificlinux.org/linux/scientific/7rolling/archive/debuginfo/glibc-debuginfo-2.17-105.el7.x86_64.rpm) 等安装包并替换现有的

   ```
   rpm -Uvh glibc-2.17-106.el7_2.8.x86_64.rpm glibc-common-2.17-106.el7_2.8.x86_64.rpm glibc-debuginfo-2.17-106.el7_2.8.x86_64.rpm glibc-headers-2.17-106.el7_2.8.x86_64.rpm glibc-debuginfo-common-2.17-106.el7_2.8.x86_64.rpm glibc-devel-2.17-106.el7_2.8.x86_64.rpm --force
   ```

4. 查看 coredump 中的变量来推测退出时机

   ```
   gdb usr/sbin/zbs-metad UrgentThread.core.2042.1705303636
   (gdb) file zbs-metad
   (gdb) directory /path
   (gdb) thread apply all bt
   (gdb) thread 1
   (gdb) bt full
   (gdb) frame 1
   (gdb) print 在这个线程中的变量
   ```

5. 查看特殊类变量

   1. 查看 co-list，[对应说明](https://docs.google.com/document/d/1kLfRK0X44sq_8O5hOYIYdZQk8XV5rragK1izof8ZqyI/edit#heading=h.b88mlnz9busr)

   2. 查看 btree size 等直接通过 gdb 无法获取的信息，可以借助 gdb.parse_and_eval() 来实现，[参考]( http://gerrit.smartx.com/c/zbs/+/34467)

      ```shell
      # 执行对应的 C++ 代码 ((detail::ThreadLocalData*)cur_tld)->pid
      gdb.parse_and_eval("((detail::ThreadLocalData*)%s)->pid"%cur_tld.str())
      ```



[参考1](https://zhuanlan.zhihu.com/p/74897601)，[参考2](https://cloud.tencent.com/developer/article/1142947)

待补充

https://coolshell.cn/articles/3643.html

https://blog.csdn.net/haoel/category_9197.html

gdb 使用方法，https://wizardforcel.gitbooks.io/100-gdb-tips/content/info-function.html

