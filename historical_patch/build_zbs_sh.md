ZBS 编译脚本，运行单测脚本

```
#!/bin/bash
set -x

ws=`pwd`
test=${TEST:-Write}
repeat=${REPEAT:-1}
type=${TYPE:-1}
gdb=${GDB:-0}
ninja_jobs="-j 10"
asan=${ASAN:-0}
extra_build=""
target=$1
build_dir="build$type"
GCC=7

[ $asan -eq 1 ] && extra_build="-DUSE_SANITIZE=address"
[ $# -ne 1 ] && echo "need a target argument!" && exit 1
date
cmd="cd /sys/fs/cgroup; mkdir cpuset/zbs-test; echo 1 >cpuset/zbs-test/cpuset.cpus;cd -"
cmd="$cmd; cp /etc/zookeeper/zoo_zbs.cfg /etc/zookeeper/zoo.cfg;zkServer.sh start"
cmd="$cmd; export GTEST_FILTER='*$test*';export GLOG_v=0;export GTEST_REPEAT=$repeat"
#export ZK_HOSTS='192.168.64.215:2181'
if [ $type -eq 0 ];then
    cmd="$cmd; cmake3 -DBUILD_GOLD_LINKER=ON -DCMAKE_BUILD_TYPE=RELWITHDEBINFO -DBUILD_UTILS=1 $extra_build -GNinja $ws;ninja $ninja_jobs $target"
elif [ $type -eq 3 ];then
    #-DBUILD_UTILS=1
    cmd="$cmd; cmake3 -DBUILD_GOLD_LINKER=OFF -DCMAKE_BUILD_TYPE=Debug -DUSE_SANITIZE=address $extra_build -GNinja $ws;ninja $ninja_jobs $target"
elif [ $type -eq 4 ];then
    cmd="cmake3 -DCMAKE_BUILD_TYPE=RELWITHDEBINFO -DBUILD_BENCHMARKS=ON -GNinja $ws;ninja $ninja_jobs zbs-bench;src/zbs-bench -logtostderr=0 -stderrthreshold=100 --benchmark_min_time=5 --benchmark_out_format=console --benchmark_counters_tabular=true --benchmark_filter='$test'"
    #./zbs-bench -logtostderr=0 -stderrthreshold=100 --benchmark_filter=LSMBenchmark/Async.*/4096* --benchmark_min_time=5 -chunk_use_lsm2=true -test_dir=/tmp/pmem -use_pmem=true -use_ioat=true -zbs_lsm_uses_dmalloc=true -zbs_dma_uses_hugepage=true
fi

#5a988c80c9fb \
docker run -it --rm --privileged -w `pwd` -v /dev/hugepages:/dev/hugepages -v /dev/shm:/dev/shm -v /dev/infiniband/:/dev/infiniband/ \
    -v /root/:/root/ -v /var/crash:/var/crash -v /var/log/zbs:/var/log/zbs \
    docker.smartx.com:5000/zbs-devtime:el7 \
    bash -c \
    "export ASAN_OPTIONS=disable_coredump=0:unmap_shadow_on_exit=1:fast_unwind_on_malloc=0;.  /opt/rh/devtoolset-${GCC}/enable;export PATH=/usr/lib64/ccache/:\$PATH;which gcc;gcc -v;/sbin/rpcbind -w;cd $ws;mkdir $build_dir >/dev/null; cd $build_dir; $cmd; cp compile_commands.json $ws"
```

