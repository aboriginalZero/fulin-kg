### Docker 概念

#### Docker 架构

Docker 其实指代的是用于开发、部署、运行应用的一个平台。平常中说的 Docker 准确来说是 Docker Engine。它是一个 C/S 架构的应用。其中主要的组件有：

- Docker Server：长时间运行在后台的程序，就是熟悉的 daemon 进程.
- Docker Client: 命令行接口的客户端。
- REST API: 用于和 daemon 进程的交互。

<img src="https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211020129.png" alt="image-20220821102002087" style="zoom:50%;" />

通过给 Docker Client 下发各种指令，然后 Client 通过 Docker daemon 提供的 REST API 接口进行交互，来让 daemon 处理编译，运行，部署容器的繁重工作。 大多数情况下， Docker Client 和 Docker Daemon 运行在同一个系统下，但有时也可以使用 Docker Client 来连接远程的 Docker Daemon 进程，也就是远程的 Server 端。

#### Docker 特点

* 更高的资源利用：容器的本质是一个独立的进程，不需要进行硬件虚拟以及运行完整操作系统等额外开销，容器对系统资源的利用率更高。无论是应用执行速度、内存损耗或者文件存储速度，都要比传统虚拟机技术更高效。因此，在单机环境下与KVM之类的虚拟化方案相比，能够运行更多数量的实例，而且多个容器互不影响，彼此独立。

* 更快的启动部署：传统的虚拟化技术启动系统和应用需要分钟级，容器应用共享宿主内核，无需启动完整的操作系统，因此可以做到秒级、甚至毫秒级的启动时间，大大节约了开发、测试、部署的时间。

* 更一致的运行：开发过程中一个常见的问题是环境一致性问题，常常表现为动态库、JDK、依赖等版本的差异。容器以标准的方式，使用镜像提供了除内核外完整的运行时环境，确保了应用运行环境一致性。容器可以跨平台运行，无论是物理机、虚拟机，其运行结果是一致。

* 更快的弹性伸缩：在业务高峰时刻，容器可以根据CPU、内存、甚至业务指标进行快速扩容，提升服务能力。在业务低谷期，可以稳步降低副本数量，节约资源。

### Docker 操作

#### 使用 Docker 镜像

```shell
docker pull img1:latest				# 相当于 registry.hub.docker.com/img1:lastest 
docker images --no-trunc			# 镜像列表，展示截断部分
docker tag img1:v1 user/img2:v1		# 添加标签
docker push user/img2:v1			# 上传镜像，需要提前 tag 成带 user 前缀的镜像名
docker image prune					# 清理系统中的临时镜像
docker rmi mysql tomcat				# 删除镜像
docker rmi -f $(docker images -q)	# 强制删除所有镜像（包含运行中的）
docker load -i /dir/tar_path		# 从压缩包中加载镜像
docker save -o /dir/tar_path img1	# 保存镜像为一个压缩包
docker commit -m "commit msg" cntr1 img1:v1.0	# 将运行的容器做成镜像
```

#### 操作 Docker 容器

```shell
docker run --name "cntr_name" -d img1	# 创建并运行容器
docker run --name "cntr_name" -it --privileged -v /a/b:/tmp:ro -p 80:80 img1
docker ps -a 							# 查看所有状态下的容器
docker rm -f cntr_id					# 删除一个运行中的容器
docker rm -f $(docker ps -aq)			# 强制删除包含运行中的所有容器
docker exec -it cntr_id bash			# 通过启动新进程的方式进入容器，不同用户相互独立	
docker attach cntr_id					# 进入容器启动命令的终端，不会启动新的进程
docker inspect cntr_id					# 查看容器详细信息
docker logs -t cntr_id					# 查看容器中的输出结果
docker top cntr_id						# 查看容器中正在运行进程
docker cp test.txt cntr_id:/tmp			# 从宿主机中拷贝文件到容器
docker system prune --volumes			# 清理处于停止状态的容器、挂载卷、临时镜像
Ctrl-p Ctrl-q 							# 临时退出而非终止一个正在交互的容器终端
```

docker create/run 的参数包括：

```shell
-i								# 开启标准输入
-t								# 分配一个伪终端
-v abs_host_path:cntr_path:ro	# 宿主机中的绝对路径映射到容器中的路径
-w cntr_path					# 容器内的默认工作目录
--link img1:rename_img1			# 链接到容器 img1，这样就可以通过 rename_img1 访问容器 img1
--entrypoint=""					# 镜像存在入口命令时，覆盖为新的命令
-p host_port:cntr_port			# 端口映射
--privileged					# 给容器 root 权限
```

* 容器互联相当于在两个互联的容器之间创建一个虚拟通道，而不需要映射它们的端口到宿主主机上。
* 绑定数据卷时，本地路径必须是宿主机中目录而非文件的绝对路径，容器内路径可以是相对默认工作目录的相对路径。使用时应该直接挂载文件所在目录到容器内。
* 当用 docker run 来创建并启动容器时，Docker Daemon 的操作包括：1. 若本地存在指定镜像直接使用，否则从共有仓库下载；2. 根据镜像创建并启动容器；3. 分配一个文件系统给容器，并在只读的镜像层上挂在一层可读写层；4. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中；5. 从网桥的地址池配置一个 IP 地址给容器；6. 执行用户指定的应用程序，执行完成后容器被自动终止。

#### 基于 DockerFile 创建镜像

基本语法

* FROM 指明基础镜像，一定要有一个基础镜像
* EXPOSE 声明镜像内服务监听的端口，并不会自动完成端口映射
* ENV 指定环境变量
* WORKDIR 使用绝对路径指定默认工作目录
* COPY 复制文件/目录到镜像，与 ADD 类似
* RUN 运行指定命令
* ENTRYPOINT 指定镜像的默认入口命令，建议使用 exec 表示法
* CMD 指定启动容器时默认执行命令，ENTRYPOINT 和 CMD 同时存在时会拼接在一起后被执行

注意点

1. 每条 RUN/ADD/COPY 指令在当前镜像的基础上执行指定命令后提交为新的镜像层，所以要尽可能用 \ 换行来合并到同一条语句
2. ENTRYPOINT 与 CMD 可以组成使用且可以在运行容器的时候分别被覆盖，如 docker run --entrypoint hostname demo python 1.py

创建一个文件夹，用以存放创建镜像用到的所有文件

```shell
mkdir -p /tmp/nginx && cd /tmp/nginx
touch Dockfile
```

编写 Dockfile

```dockerfile
ARG VERSION=9.3
FROM debian:${VERSION}
LABEL description="This is example desc"
EXPOSE 80 8443
ENV APP_HOME=/usr/local/app
ENV PATH=$PATH:/usr/local/bin
WORKDIR /tmp/code
COPY *.c /bin							# 在 /tmp/code/ 下的所有 .c 文件
RUN apt-get update \
	&& apt-get install -y libsnappy-dev zlib1g-dev libbz2-dev \
	&& rm -rf /var/cache/apt \
	&& rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/bin/bash", "-c", "entrypoint.sh"]
CMD ["/bin/bash", "-c", "echo finish"]
```

entrypoint.sh 内容

```bash
#! /bin/bash
/usr/sbin/sshd &
```

根据 Dockerfile 构建镜像

```shell
docker build --force-rm -t abori/ngnix .
```

该命令将读取 . 目录及其子目录下的 Dockerfile，并将该路径下所有数据作为上下文发送给 Docker 服务器生成镜像，--force-rm 指的是无论是否构建成功，都删除中间产生的容器。

如果所给上下文过大，会导致发送大量数据给服务端而延缓创建过程，通过 .dockerignore 文件来忽略指定文件

```shell
*/temp*		# * 表示任意多个字符
file_?		# ? 表示单个字符
!README.md	# ! 表示不匹配
```

#### 基于 Compose 编排容器

使用 docker-compose 的命令时，默认会在当前目录下找 docker-compose.yml 文件

```shell
docker-compose up -d
docker-compose down
docker-compose ps
docker-compose logs -f
```

常见的 docker-compose.yml 模板

```yaml
# 构建 mysql 容器
services:
  cmysql:  # 服务的名称
    restart: always  # 代表只要docker自动，容器随之启动
    build:  # 构建自定义镜像
      context: ./  # 指定Dockerfile文件所在相对于docker-compose.yml路径
      # 指定 Dockerfile 文件名称，本例中，仅 from mysql:lastest
      dockerfile: Dockerfile 
    image: cmysql:1.0.1  # 指定镜像名称
    container_name: mysql  # 指定容器名称
    environment:
      MYSQL_ROOT_PASSWORD: root  # 指定MySQL的ROOT用户登录密码
      TZ: Asia/Shanghai  # 指定时区
    ports:
      - "3306:3306"
```

### Docker 实战

#### 为容器设置固定 IP

Linux 中 Docker 在创建容器后，删除了宿主主机上 /var/run/netns 目录中相当的网络命名空间文件，因此在宿主机上是无法看到或访问容器的网络命名空间，首先查看容器进程信息

```shell
docker inspect --format='{{.State.Pid}}' cntr_id
sudo ln -s /proc/<pid>/ns/net /var/run/netns
sudo ip netns show										# 显示容器的网络命名空间号
sudo ip netns exec <pid> ifconfig eth0 172.17.0.100/16	# 设置成固定 IP
```

#### 编译多版本 gcc/python

如在 mac 中借助 docker 使用 gcc 编译文件

```shell
docker run -v abs_host_path:/tmp -w /tmp rikorose/gcc-cmake gcc 1.cpp -o 1
docker run -v abs_host_path:/tmp -w /tmp rikorose/gcc-cmake gcc ./1
```

#### 搭建本地私有仓库

在私有服务器 10.0.0.2 上启动一个私有仓库服务，端口为 5000

```shell
docker run -d -p 5000:5000 -v /tmp/registry:/var/lib/registry registry:2
```

在开发机 A 上给即将上传的镜像打标签，并上传

```shell
docker tag img1 10.0.0.2:5000/img1
docker push 10.0.0.2:5000/img1
curl http://10.0.0.2:5000/v2/search		# 用以验证是否上传成功
```

在开发机 B 上下载私有镜像，需要先关闭对私有仓库的 TLS/SSL 安全性检查

```shell
# 修改 Docker daemon 的启动参数
DOCKER_OPTS="--insecure-registry 10.0.0.2:5000"	
# 或者修改 ~/.docker/daemon.json | /etc/docker/daemon.json
{ "insecure-registries":["your_registry.example.com:5000"] }

service docker restart							# 重启 docker 服务
docker pull 10.0.0.2:5000/img1
```

#### Docker 安装

Docker 要求 CentOS 系统的内核版本高于 3.10（namespace/cgroups 等的支持）

```shell
# 升级软件包及内核
yum -y update
# 卸载老版本
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
# 安装依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2 
# 设置阿里云镜像
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 
# 安装 docker
yum install docker-ce docker-ce-cli containerd.io
# 启动 docker
systemctl start docker
# 将 docker 服务设为开机启动
systemtctl enable docker
# 检测是否安装成功
docker -v
docker run hello-world
```

#### Web 展示文件

借助 Nginx 实现，目录结构如下

```
➜  nginx_web tree
.
├── dir
│   ├── nginx.conf
│   └── show.txt
└── docker-compose.yml
```

docker-compose.yml 配置

```yml
web:
  image: nginx
  volumes:		# 可以是相对/绝对路径
    - ./dir/nginx.conf:/etc/nginx/nginx.conf:ro
    - ~/:/dir
  command: [nginx-debug, '-g', 'daemon off;']
  ports:
    - "84:80"
```

nginx.conf 配置

```
events {
    use epoll;
}
http {
    autoindex on;
    server {
        listen 80;
        server_name  localhost;
        location / {
            root   /dir;    	# 指定 nginx 映射的根目录
            # index show.txt;   # 不指定的话，通过 autoindex on 配合，映射整个文件夹
        }
    }
}
```

#### 分布式 Web 项目

HAproxy

搭建分布式 Redis 集群，https://juejin.cn/post/6992872034065727525

Python 使用分布式 redis

```shell
pip install redis-py-cluster
```

使用方式

```python
from rediscluster import RedisCluster
startup_nodes = [{"host": "127.0.0.1", "port": "7000"}, 
                 {"host": "127.0.0.1", "port": "7001"}]
rc = RedisCluster(host="127.0.0.1", port=7000, decode_responses=True)
rc.set("foo", "bar")
```

#### 搭建 CI/CD 平台

Jenkins + Gitlab + 自动部署

https://blog.csdn.net/weboof/article/details/104491998