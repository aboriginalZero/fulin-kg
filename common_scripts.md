### ut 里查看 volume 信息

```
auto print_volume = [&](const Volume& old_volume, const std::string& name = "volume") {
  meta::VolumePath path;
  path.set_volume_id(old_volume.id());

  std::vector<PExtentResp> pextents;
  std::vector<LExtentResp> lextents;
  Volume volume;
  ASSERT_STATUS(GetMeta()->ShowVolume(path, &volume, &pextents, &lextents));

  std::ostringstream oss;

  oss << "yiwu " << name << ", id: " << volume.id() << ", prior: "
      << volume.prioritized() << ", thin: " << volume.thin_provision();

  oss << "\n lextents(lid, perf_pid, cap_pid, is_cow): ";
  for (const auto& lextent : lextents) {
    oss << FormatString(", (%u, %u, %u, %u)", 
                        lextent.id(), lextent.perf_pid(), lextent.cap_pid(),
    lextent.is_cow());
  }

  oss << "\n perf pextents(pid, is_thin, loc): ";
  for (const auto& pextent : pextents) {
    if (pextent.type() == PT_PERF && pextent.pid() != kInvalidPExtentId) {
      oss << FormatString(", (%u, %u, %s)", pextent.pid(), pextent.thin_provision(),
      meta::location_str(pextent.location()));
    }
  }

  oss << "\n cap pextents(pid, is_thin, loc): ";
  for (const auto& pextent : pextents) {
    if (pextent.type() == PT_CAP && pextent.pid() != kInvalidPExtentId) {
      oss << FormatString(", (%u, %u, %s)", pextent.pid(), pextent.thin_provision(),
      meta::location_str(pextent.location()));
    }
  }

  LOG(INFO) << oss.str();
};
```





### 循环输出命令结果

```
[root@zhiwei-node2-128-26 12:13:38 ~]$ cat stats.sh
#!/bin/bash
echo > iops.log
while true;

do
    echo "=================" >> iops.log
    date >> iops.log
    zbs-chunk internal_io get --show_bucket_details --show_perf_details >> iops.log
    sleep 1
done
```



### 随机写入 io

```
# 创建一个 100GiB 的卷，并通过 zbs-chunk volume write <pool-name> <volume-name> <offset> <len> -i input_file 向每个 400 个 pextent 的首 256 KiB 写入数据。

#!/bin/bash

for i in $(seq 0 399)
do
    extent_size=`expr 1024 \* 1024 \* 256`
    len=`expr 1024 \* 256`
    echo $len
    offset=`expr $extent_size \* $i`
    echo $offset
    zbs-chunk volume write pool lp-dealloc-test $offset $len -i /var/log/zbs/zbs-metad.INFO
done
```

zbs-chunk volume write 这个命令只在 volume 上有效，lun 不行。



### 检查某个 chunk 上在 < 90% 负载下为何没有迁移

```
[root@dogfood-idc-elf-103 12:04:23 yiwu]$ cat 2.sh 
#!/bin/bash

pid_list=$(zbs-meta -ftable chunk list_pid 10.255.0.103 10200 | grep "perf thin pid" | grep -o -E '[0-9]+')
for pid in $pid_list; do
    output=$(zbs-meta -fjson pextent show "$pid")
    prefer_local=$(echo "$output" | jq '{"Prefer Local"}')
    ever_exist=$(echo "$output" | jq '{"Ever Exist"}')
    if [[ "$ever_exist" == "True" ]] && [[ "$prefer_local" != *"10"* ]]; then
        echo "pid: $pid"
    fi   
done
```

