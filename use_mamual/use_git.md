## Git

### 相关概念

Git 作为一款分布式版本控制系统，常被用来管理项目的版本更迭。

* 工作区：在你的计算机上能看到的目录。

* 暂存区：在 Git 的版本库中，需要提交的文件修改通通放到暂存区，然后一次性提交暂存区的所有修改，`git add . `之后存放的位置。

* 版本库：即本地仓库，是新建版本库时在工作区下生成的一个隐藏目录`.git`，通过 `ls -a` 命令可以看到。`git commit `之后存放的位置。
* 远程库：关联的远程仓库（github/gerrit）。

![image-20220821102045019](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202208211020071.png)

### 快速使用

#### 安装并初始化

安装 git，配置基本信息，以及创建版本库

```shell
sudo apt-get install git
git config --global user.name "Your Name"
git config --global user.email "email@example.com" 
git config --global core.editor "vim"	# 这样写 commit msg 可以看到列号
git init 
```

可以看到在当前目录下生成 .git 文件，以及名字邮箱配置成功

```shell
ls -a
git config user.name
git config user.email
```

添加修改文件到暂存区，并提交到版本库

```shell
git commit -am "<JIRA-num> module: short description"	# 生成新节点
git commit --amend										# 复用原有节点
```

在 title 和 change-ID 之间添加详细描述的 commit msg，如果需要描述的内容过多，在 JIRA 中说明

#### 创建与合并分支

```shell
# 在 master 上 checkout
git checkout -b patch1		
# 在 v5.0.x 上开发完成后，回到主分支合并当前分支
git checkout master
git merge patch1
git branch -d patch1
# 对于不需要合并的分支可以通过 git branch -D patch2 删除
```

始终保持 master 与远程 master 同步，每个 patch 在单独的分支上开发

#### 查看提交历史

```shell
git log
# 查看最新提交的 log commit ID
git rev-parse --short HEAD
# 查看最新的一条 log 以及 diff
git log -1 -p
```

#### 关联远程仓库

在 GitHub 上添加开发机上的公钥，这样可以免密 ssh push/pull/clone 代码

<img src="https://img-blog.csdnimg.cn/img_convert/49dd7a37277e59face3bc0b1f0e4e2a4.png" alt="img" style="zoom:50%;" />

本地没有仓库，从远程 clone

```shell
# 推荐配置公钥使用
git clone git@github.com:your_github_name/your_hub.git	

# 设置 http 许可，并通过 http 下载
git config --global http.sslVerify false
# 如果遇到 RPC failed, curl 18 transfer closed with outstanding read data remaining
# 增加 git 可允许传输的项目大小
git config --global http.postBuffer 1024288000

git clone http://github.com/your_github_name/your_hub.git
# 通过 https 下载，速度较慢
git clone https://github.com/your_github_name/your_hub.git 		
```

本地已有仓库，关联到远程仓库

```shell
git remote add <init_remote_name> git@github.com:your_github_name/your_hub.git
# 如果本地仓库非空
git pull <init_remote_name> master --allow-unrelated-histories 
# 第一次推送 master 分支到远程需要加 -u 参数
git push -u <init_remote_name> master
```

通过以下命令查看配置

```shell
git remote -v
git remote remove <delete_remote_name>
```

#### 比对版本

* 查看工作区和暂存区的代码版本区别：`git diff`
* 查看工作区和版本库的代码版本区别：`git diff HEAD`
* 查看工作区和远程库的代码版本区别：`git diff origin/master`
* 查看版本库和远程库的代码版本区别：`git diff origin/master HEAD`

#### 回退版本

* 放弃工作区的修改：`git checkout .`
* 放弃从工作区已经提交到暂存区的修改：`git reset HEAD <filename>`
* 放弃从工作区已经提交到版本库的修改：`git reset --hard HEAD^`
* 放弃从工作区已经提交到远程库的修改：`git reset --hard <commitID>`、`git push origin master --force`
*  git reset 的三种 mode 说明
    * --mixed：仅 reset 暂存区，工作区还会有一份修改保留，这也是默认模式
    * --hard：reset 暂存区和工作区，之前的修改不会有任何保留
    * --soft：仅 reset 版本库，暂存区和工作区还会有一份修改保留
* 如果误放弃了，通过 git reflog 找回那次 commit_id，通过`git checkout commit_id .`恢复

#### Gerrit 使用

把 Gerrit 看成一个带有更方便 review/comment 的 GitHub 

```shell
# 远程仓库名必须是 gerrit
git remote add gerrit ssh://yiwu.cai@gerrit.smartx.com:29518/zbs
# 拉取远程分支版本
git checkout -b local-v4.0.x gerrit/v4.0.x
# 注意是在项目的主目录下提交
git commit --am "xxx"
# 确保 Change ID 前后一致在多次修改提交中都是一致的
git commit --amend
# 如果不指定，默认 push 到远程的 master 分支
git review v4.0.x
```

git review 实际上会 rebase 创建一个临时分支，当 git review 时跟远程有冲突

```shell
git review -t 5.x.x
# 这时提示有冲突，手动修改之后
git add .
git rebase --continue
git checkout -f dev_path
git commit --amend
git review
```

git review 变基一半时显示冲突，origin master 被更新了，可以先去本地 master 拉取最近的更新，然后 checkout 一个新分支，并将旧分支 pick 到这

```shell
git rebase --abort
git status 
git show HEAD (gsh)
git status
# 此时恢复到都放在版本库的状态
git checkout master
git pull
git submodule update
git checkout -b zbs-25204-2
git cherry-pick zbs-25204
git diff
git add src/proto	# 因为对 src/proto 做改进了
git status
git cherry-pick --continue
git show HEAD			# 此时应该符合所有自己已修改的
```

git pull 之后想撤销，先 git reflog 看一下想回退到之前的哪一个状态，然后用 git reset --hard log_id 回退



在分支上需要重新 pull，并把本地的这个放到最前面

1. git pull --rebase origin master
2. git submodule update --init --recursive



pick 到非 master 分支如 v5.4.x 时

1. 在 master 分支上 git pull && git submodule update --init --recursive
2. 拉取远端最新代码 git checkout remotes/origin/v5.4.x -B v5.4.x
3. 此时在 v5.4.x 分支 git submodule update 
4. 把 gerrit 上的 patch 拉过来 git review -x 49627，这是网址 http://gerrit.smartx.com/c/zbs/+/49627
5. git status 发现 proto 有冲突
6. 放弃此时分支上的 proto 版本，其实就是远程分支， git checkout HEAD src/proto 
7. git status 发现干净了
8. git cherry-pick --continue
9. 此时进入 cd src/proto && git review -d 44444，这是 proto 的提交 patch
10. 然后 cd .. && cd .. 回到 zbs 目录 git add -u
11. 然后 git commit --amend && git review v5.4.x -R（-R 表示不变基，在这有无都行）



远端 master 合并了新 patch，导致我的已有通过 CI 的 patch 显示 merge conflict，如何解决？

1. git checkout master
2. git branch -D zbs-21246
3. git pull origin master
4. git checkout -b zbs-21246
5. git fetch "ssh://yiwu.cai@gerrit.smartx.com:29518/zbs" refs/changes/71/51871/3 && git cherry-pick FETCH_HEAD
6. 此时会显示哪些文件冲突了，手动解决下冲突，用 git status 查看都是冲突的文件是红色的
7. git add .
8. git cherry-pick --continue
9. git review 此时就把没有冲突的 patch 版本更新上去了



 commit 链提交与修改（指的是有前后依赖关系的多个 commit）

1. 第一个 patch 的 commit 跟之前的做法一样，直到执行完 gca；
2. 紧接着写第二个 patch，写完要执行到 git commit -a -m 这一步，这样 git tree 中才有 2 个节点，再 gca 补充这个 commit 的 detail msg；
3. gr 时会提醒把 2 个 commit 都 push 到 gerrit 上。

被 review 后如果需要修改

1. git rebase -i HEAD^^，有几个 commit 就打几个 ^，
2. 然后把要改的 commit 设置为 edit，就可以直接在 vs code 上改被设成 edit 的 n 个 patch；
3. 每个 patch 改完都需要 ga gca 再 git rebase --continue；
4. 全部改完后，再 git review，会把所有 commit 都更新到 gerrit，需要手动去每个 patch 执行 /runtest。



放弃所有子模块的修改，git submodule foreach --recursive git restore .

查看版本库中最新提交修改的文件列表 git show --stat 

#### 管理指定文件

在项目根目录下添加 .gitnore 文件，下面是一些.gitignore文件忽略的匹配规则：
```
*.a       # 忽略所有 .a 结尾的文件
!lib.a    # 但 lib.a 除外
/TODO     # 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
build/    # 忽略 build/ 目录下的所有文件
doc/*.txt # 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
```
下面给出一个 demo ：
```
.ieda
.xml
out
gen
```
.gitignore 只能忽略那些原来没有被track的文件，如果某些文件已经被纳入了版本管理中，那么解决方法就是先把本地缓存删除（改变成未track状态），然后再提交： 
```shell
git rm -r –-cached .   #把所有暂存区里的文件删了
git add . 
git commit -m “refactor: update .gitignore”
```

#### 小技巧

* 列出该文件每行的修改记录：`git blame filename`
* 更新子模块：`git submodule update --init --recursive`
* 提交所有被删除、被替换、被修改和新增的文件到数据暂存区：`git add -A`
* 临时要开几个新分支补 patch，怎么暂存当前分支的代码？建立多个文件夹 zbs1-5 + git stash 使用
* git 仓库中的一个 submodule 对应的版本号存储在哪？.git 中，可以用 git prune 减小 .git 的大小

#### 常见场景

* code reiew 期间开发其他 feature 时，先 checkout 到 master 并 git pull 远程最近的改动，在master 的基础上 checkout 出一个新分支。如果直接在上一个 dev branch 上 checkout 就会产生不必要的依赖关系，尽可能保证每一个 Change 的完整性以及独立性，且越小越好。

* git stash 用以在当前 branch 修改了部分代码，没到 commit 的地步，这时要去看其他 branch 的代码情况，可以先 git stash 再 git checkout master，要回到 dev 时，git checkout dev、git stash pop。git stash 还有一个常见用法。如果发现在 1st dev branch 上修改的代码应该放在另一个 branch 上时，可以先 git stash，然后切换到 2nd dev branch 上 git stash pop，这时可能存在冲突，需要手动修改（有冲突的时候，stash里面还会暂存着改动）。并回到 1st dev branch 中通过 git checkout . 把还未 git add . 的代码删掉

* push 代码到远程，发现相同部分代码已被他人更新，此时陷入合并中间状态，应对方法：

    ```shell
    git merge --abort	# 暂停 merge
    git reset --merge
    git pull 			# 解决冲突
    git add .
    git commit --amend
    git push origin dev	# 从当前分支 push 到远程的 dev 分支
    ```

* git 合并远程多个 commit

    ```shell
    # 合并这个commit之后的所有commit (不包括这个)
    git rebase -i <log-id>
    # 在窗口中修改，将除了第一个的 pick 保留，其他都改成 s，保存退出
    # 这时 git log 发现本地的多个 commit 已经合并
    # 强制 push 到远程，这样就能把远程的多个 commit 合并，这种情况适用于这个远程仓库就你一个人在维护
    git push -f
    ```
    
* 误提交子模块的老版本数据，需要还原

    ```shell
    git checkout HEAD~1 libiscsi	# 放弃 libiscsi 最近一次的修改
    cd libiscsi
    git clean -xdf	# 删除编译带来的临时文件
    git status 		# 查看应该是没有了
    git add -u		# 提交删除文件到版本库
    git log --stat	# 查看只有目标文件被修改
    git submodule update --init --recursive
    git commit --amend
    git review
    ```
    

#### CentOS 7 自编译 Git

CentOS 7  yum 中自带的 Git 版本过低，需要自行下载高版本 Git 源代码编译 Git

  1. 安装依赖、卸载旧版本

     ```shell
     yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel asciidoc
     yum install gcc perl-ExtUtils-MakeMaker
     yum remove git
     ```

  2. 安装新版 git

     ```shell
     cd /usr/local/src/
     wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.23.0.tar.xz
     tar -xvf git-2.23.0.tar.xz
     rm -y git-2.23.0.tar.xz && cd git-2.23.0/
     make prefix=/usr/local/git all
     make prefix=/usr/local/git install
     echo "export PATH=$PATH:/usr/local/git/bin" >> /etc/profile
     source /etc/profile
     ```
