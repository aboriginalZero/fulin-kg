## SSH

SSH 是一个协议，OpenSSH 是它的一个开源实现。1999年，OpenBSD 的开发人员决定写一个 SSH 2 协议的开源实现，也就是 OpenSSH 项目。SSH 主要用于远程登陆，还可以直接远程操作、作为中间节点做端口转发。SSH 之所以能够保证安全，原因在于它采用了公钥加密。整个过程是这样的：

1. 远程主机收到用户的登录请求，把自己的公钥发给用户；

2. 用户使用这个公钥，将登录密码加密后，发送回来；

3. 远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

以上过程不可避免的，会遭遇中间人攻击。所以引入 ~/.ssh/known_hosts 存放公钥指纹来减小被攻击的可能。

> 当网络状况良好时，每输入一个字符都会立即显示，原因是每输入一个自负就被打成 TCP 包传到服务器上，服务器也立马回复。

### 登录认证

除了密码登录外， SSH 还提供了公钥登录，即用户将自己的公钥储存在远程主机上。登录的时候，远程主机会向用户发送一段随机字符串，用户用自己的私钥加密后，再发回去。远程主机用事先储存的公钥进行解密，如果成功，就证明用户是可信的，直接允许登录 shell，不再要求密码。

#### 生成公私钥

```shell
ssh-keygen -q -t rsa -N '' -f ~/.ssh/smartx_id_rsa
```

上述命令中，-q 指定 silence mode，-t 指定生成密钥的加密算法，-f 指定生成的公私钥存放位置

#### 添加公钥到对端

```shell
ssh-copy-id user@host
```

上述命令将公钥自动添加到 host 的 ~/.ssh/authority_keys 文件中

### 客户端 ssh 

#### 免密登录

需要将公钥添加到对端的 ~/.ssh/authority_keys，以及在登录时指定私钥位置

```shell
ssh –i privatekey_file username@ip 
```

也可以在 ~/.ssh/config 中以指定账号、指定私钥、指定配置来登录某台机器

```shell
Host zbs
	  Hostname 192.168.26.209
    User root
    Identityfile ~/.ssh/id_rsa
    StrictHostKeyChecking no	# 不需要严格认证
    UserKnownHostsFile /dev/null
  	GSSAPIAuthentication no
  	ServerAliveInterval 10		# 没有数据传输之后每 10s 发送一次 keepalive
  	ServerAliveCountMax 3			# 连续发送 3 次 keepalive 之后服务器还没有响应断开连接
  	ConnectTimeout=10					# 最多重试 10 次
```

#### 查看已认证主机

当 host A 首次登录 host B 时，会将 host B 的公钥写入 ~/.ssh/known_hosts，在之后的登录中，判断拿到的公钥是否与注册的一致，如果不一样，就拒绝登录。如果 host B 的公钥确实变了，那么在主机 A 中删除之前的公钥记录，然后重新登录时再记录到 A 的 ~/.ssh/known_hosts 文件中。

#### ssh 常用参数

* -p 指定端口，不指定为 22 端口
* -vvv 输出 ssh 期间的 debug 信息
* -N 表明只用于端口转发，不能执行远程命令
* -o 用来指定配置命令，如 -o BatchMode=yes|no。当 BatchMode=yes 时，表示将不会显示交互式口令输入，而是直接失败，从而避免批处理的时候卡住。当 BatchMode=no 时，将会提示用户输入密码。
* -o ConnectTimeout=5。表示ssh连接的超时时间，单位是秒。
* -o StrictHostKeyChecking=yes|no|ask。当该值为 no 时，表示在连接远程主机时，会主动把对方的公钥加到 known_hosts 中，而不会提示用户是否要记录这样的信息，且当远程主机的公钥变化了，仍然会连接上，不会出现因为公钥不对连接失败。与之配合的另一个参数是 `UserKnownHostsFile /dev/null`
* -o GSSAPIAuthentication no。通常情况下我们在连接 OpenSSH 服务器的时候假如 UseDNS 选项是打开的话，服务器会先根据客户端的 IP 地址进行 DNS PTR 反向查询出客户端的主机名，然后根据查询出的客户端主机名进行 DNS 正向 A 记录查询，并验证是否与原始 IP 地址一致，通过此种措施来防止客户端欺骗。平时我们都是动态 IP 不会有 PTR 记录，所以打开此选项也没有太多作用。我们可以通过关闭此功能来提高连接 OpenSSH 服务器的速度。与之配合的另一个参数是 `UseDNS no`

### 服务端 sshd 

配置文件在 /etc/ssh/sshd_config，一般不需要怎么配置

日志文件在 /var/log/auth.log 或者 secure

#### 仅指定公钥登录

在服务器上关闭密码登陆，仅允许 authority_keys 中的机器登陆，做法是在服务器的 /etc/ssh/sshd_config 上将 PasswordAuthentication 设为 no，并重启 ssd。

#### 指定展示信息

Banner 字段指定用户登录后，sshd 向其展示的信息文件（ Banner /usr/local/etc/warning.txt），默认不展示任何内容。

### 端口转发

SSH 除了登录服务器，还有一大用途，就是作为加密通信的中介，充当两台服务器之间的通信加密跳板，使得原本不加密的通信变成加密通信。这个功能称为端口转发（port forwarding），又称 SSH 隧道（tunnel）。

#### 动态转发

本机与 SSH 服务器之间创建了一个加密连接，本机内部针对某个端口的通信，都通过这个加密连接转发。它的一个使用场景就是，访问所有外部网站，都通过 SSH 转发。

```shell
ssh -D 2121 tunnel-host -N
```

上面命令中，-D 表示动态转发，2121 是本地端口，tunnel-host 是 SSH 服务器，-N 表示这个 SSH 连接只进行端口转发，不登录远程 Shell，不能执行远程命令，只能充当隧道。这种转发采用了 SOCKS5 协议。访问外部网站时，需要把 HTTP 请求转成 SOCKS5 协议，才能把本地端口的请求转发出去。

```shell
curl -x socks5://localhost:2121 http://www.baidu.com
```

上面命令中，curl 的 -x 参数指定代理服务器，即通过 SOCKS5 协议的本地 2121 端口，访问百度。

动态转发对应的持久配置，可以将设置写入 SSH 客户端的用户个人配置文件（~/.ssh/config）

```shell
DynamicForward tunnel-host:2121
```

#### 本地转发

指定一个本地端口，所有发向那个端口的请求，都会转发到 SSH 跳板机，然后 SSH 跳板机作为中介，将收到的请求发到目标服务器的目标端口。举个例子：

```shell
ssh -L 2121 www.baidu.com:80 tunnel-host -N
```

上面命令中， -L 表示本地转发，访问本地的 2121 端口，就是访问百度的 80 端口。注意：

* 本地端口转发采用 HTTP 协议，不用转成 SOCKS5 协议
* 作为跳板机，tunnel-host 必须运行 sshd
* 上述通信中，只有本机到 tunnel-host 这一段是加密的，tunnel-host 到百度这一段并不加密

一个简易的 VPN 如下，访问本地的 2080 2443 端口，通过持有公网 IP 的 SSH 跳板机转发到的内网的 10.1.73.103 的 80 443 端口

```shell
ssh -L 2080:10.1.73.103:80 -L 2443:10.1.73.103:443 tunnel-host -N
```

对应的持久配置

```shell
Host www.baidu.com
LocalForward localhost:2121 www.baidu.com:80
```

#### 远程转发

主要针对内网情况，概念不好解释，直接提供两个例子：

* 建立内网 IP+port 与公网 IP+port 的映射（NAT 的作用）

    ```shell
    ssh -R 8080:localhost:80 -N 54.223.138.35
    ```

    上面命令在内网的机器 10.1.73.103 上执行，建立从 10.1.73.103 到 54.223.138.35 的 SSH 隧道。之后访问公网 IP 54.223.138.35:8080 就会映射到内网的 10.1.73.103:80 

* 内网跳板机协助公网 IP+port 访问内网 IP+port

    ```shell
    ssh -R 8080:10.1.73.103:80 -N 54.223.138.35
    ```

    上面命令在内网的 SSH 跳板机上执行，建立跳板机到 54.223.138.35 的隧道（要求 54.223.138.35 上运行 sshd），并且这条隧道的出口映射到 10.1.73.103:80。之后访问公网IP 54.223.138.35:8080 就会映射到内网的 10.1.73.103:80 

对应的持久配置

```
Host your_define_name
	HostName 54.223.138.35
	RemoteForward 8080 10.1.73.103:80
```

### SCP 命令

openssh-8.4p1 scp 执行流程

```
+-----------+   remote command: scp -t file2    +------+
| ssh hostB |---------------------------------->| sshd |
+-----------+                                   +---+--+
		 ^                                         			|
		 |                                         			|
		 |fork()                             		fork()	|
		 |                                         			|
+----+-----------------+                +-----------V--+
| scp file hostB:file2 |                | scp -t file2 |
+----------------------+                +--------------+
```

1. scp 进程解析输入参数并读取系统配置
2. scp 进程 Fork 出一个运行 ssh 进程
3. Ssh 进程发起跟对端 sshd 进程的连接，其中包括身份认证环节
4. 对端的 sshd 守护进程 fork 出一个 scp 进程（-t 代表目的地址，-f 代表源地址）
5. 如果是从本地传到远程，那么本地 scp read 待传输文件，对端 scp 进程 write 

在传输数据时，会先发送一个消息头，其中包括加密协议类型、文件的权限位、长度以及名称，如 C0644 299 a.txt ，双方通过 tcp socket 通信以 poll 的形式多路复用传递，在拿到消息头之后就知道要发送/接受多大的文件大小。在源码中，文件大小用 unsigned long long 来定义，所以理论上传输的最大文件大小是 2^64 bytes（2^40 bytes 是1T），所以可以认为 scp 协议本身不会对文件大小有限制。

### Tmux

当 ssh 登陆远程计算机，打开一个远程窗口执行命令。这时，网络断线，再次登录的时候，是找不回上一次执行的命令，因为上一次 ssh 会话已经终止了，shell 作为它的子进程也随之消失了。为了解决这个问题，可以使用 tmux 将会话与窗口“解绑”，通过 ssh 到远端之后运行 tmux 会开启一个 tmux server 的进程，之后的 bash shell 是 tmux server 的子进程，所以当 ssh 关掉之后 shell 不会死。

![在这里插入图片描述](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202112101647555.png)

![在这里插入图片描述](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202112101647103.png)

常用命令如下：

```shell
# 安装 tmux
sudo apt-get/yum/brew install tmux
# 新建会话
tmux new -s <session-name>
# 查看会话
tmux ls
# 分离会话
tmux detach
# 接入会话
tmux attach -t <session-name>
# 杀死会话
tmux kill-session -t <session-name>
# 切换会话
tmux switch -t <session-name>
# 向上滚屏（mac）
shift + fn + up
# 切换会话
Ctrl + b 按完，然后按 s，esc 退出
```

tmux 什么情况下会被销毁进程？

### 常见错误

但凡有问题，客户端用 ssh -vvv login，服务器端到 /var/log/auth.log 或者 secure 看 ssh 日志

1. FIPS mode initialized

    ESXi主机防火墙的 SSH 服务没开启

2. 能 ping 通但是 ssh 连接不上

    实验室机器用户表由 LDAP Server 统一管理，可能是访问 LDAP Server 失败导致 ssh 无法做用户登录

