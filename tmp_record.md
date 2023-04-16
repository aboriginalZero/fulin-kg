

非首次运行单测

```
docker run --rm  -v /code/zbs:/zbs -w /zbs/build registry.smtx.io/zbs/zbs-buildtime:el7-x86_64 ninja zbs_test
cd /code && ./newci-x86_64 -builddir zbs/build/ -p 16 -action "/run 200 FunctionalTest.MarkVolumeAllocEven"
```







```shell
zbs-meta pextent read -o <output_file> <pextent_id> <offset> <length> 
zbs-chunk extent list 
```

显示 2023-04-07 14:58:47,878, ERROR, cmd.py:2967, write() argument must be str, not bytes

zbs-nfs 中难以查看 dir

linux主分区、扩展分区、逻辑分区的区别、磁盘分区、挂载，https://blog.csdn.net/qq_24406903/article/details/118763610



centos7 中将 pip 默认是 8.x，升级到 20.3.4（更高版本不支持 Python 2.7）

```
wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
```

pip 根据 whl 文件安装包

```
pip install xx.whl
```

git_review 插件安装 1.28.0，进 pypi.org 找包的各个历史版本

```
pip install git-review==1.28.0
```

pip 更改镜像源

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



