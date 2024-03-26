## linux

zsh 安装

～/.bashrc 中配置快捷命令

## mac

### 常用工具

* 读代码 vscode，相较于 Jetbrains 产品，更轻量级
* 代码追踪或思维导图，用 processon，其附带了文本转思维导图模式
* 用代码画 UML 图，PlantUML
* 本地在开发机上开发，vscode 的 remote-ssh 插件
* 设置 DNS，networksetup -setdnsservers Wi-Fi 114.114.114.114 或者是谷歌 DNS 4 个 8

### 新机部署

拿到一台新 Mac，装 zsh、iterms2、vscode、微信、Typora、PicGo、Chrome、tree、Typora css

PicGo 配置 https://cloud.tencent.com/developer/article/1755859

#### alias 配置

```shell
# vim ~/.bashrc
alias gd='git diff'
alias gb='git branch'
alias gl='git log'
alias ga='git add .'
alias gc='git checkout'
alias gca='git commit --amend'
alias gr='git review'
alias gs='git status'
alias gsu='git submodule update --init --recursive'
alias gp='git pull'
alias gcp='git cherry-pick'
alias gri='git rebase -i'

zbs_compile_prepare() {
cd /home/code/$1 && rm -rf build/ && mkdir build && cd build && cmake3 -DBUILD_MOLD_LINKER=ON -G Ninja ..
}
alias zcp=zbs_compile_prepare

zbs_compile() {
cd /home/code/$1 && ./script/format.sh && cd build && mold -run ninja zbs_test $2
}
alias zc=zbs_compile

# 第三个参数用来添加 --gtest_repeat=100
zbs_run_test() {
cd /home/code/$1/build/src && ./zbs_test --gtest_filter="*$2*" $3
}
alias zrt=zbs_run_test
```

#### SSH 配置免密

```shell
# 安装 sshpass
cd /Users/abori/Downloads && wget https://sourceforge.net/projects/sshpass/files/sshpass/1.06/sshpass-1.06.tar.gz 
tar -xzf sshpass-1.06.tar.gz && cd sshpass-1.06 && ./configure 
sudo -i
cd /Users/abori/Downloads && make && make install
```

创建 ~/.ssh/smartx_passwd 并输入密码，后续通过 sshpass -f ~/.ssh/smartx_passwd ssh smartx@172.20.134.173 登陆

```shell
function pssh () {
    passwd=('HC!r0cks' 'abc123' 'abcd1234' 'abcd_1234' 'smartx@2022')
    user=('smartx' 'root' 'smtxauto')
    for p in "${passwd[@]}";do
    	for u in "${user[@]}";do
            sshpass -p ${p} /usr/bin/ssh -o "StrictHostKeyChecking no"  -o "UserKnownHostsFile /dev/null" -t ${u}@"$1" "sudo su - root";
            #echo "ret: $?"
            [ $? -eq 0 ] && echo "$p, $u " && return
        done
    done
}

ssh_192_168_node() { pssh 192.168.$1 }
alias s9=ssh_192_168_node

ssh_172_20_node() { pssh 172.20.$1 }
alias s7=ssh_172_20_node
```

#### Iterms2

* 在 [Preferences - Pointer] 勾选 [Focus follows mouse]，方便在窗口间切换
* 在 [Preferences - Profile - Keys] 删除已有的并添加新快捷键 [option + ←] 选择 Action 为 “Send Escape Sequence” 以及 [ESC + b]，向右同理，为 [ESC + f]
* [Shift + cmd + enter] 最大化/最小化 table
* [cmd + [ / ] ] 在 table 间移动

#### Chrome

* [cmd + l]，聚焦到地址栏
* [cmd + r]，刷新网页

#### vscode 

为每个项目的 .vscode/c_cpp_properties.json 配置一下内容，按照 c++17 的标准做语法解析，不会出现不必要的红色波浪线

```json
{
    "configurations": [
        {
            "name": "centos",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "compilerPath": "/usr/local/bin/gcc", // 你的编译器路径
            "cStandard": "c17",
            "cppStandard": "c++17",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}

```



【cmd + ,】打开设置面板，右上角找到 setting.json

```json
{
    "explorer.confirmDelete": false,
    "editor.fontSize": 16,
    "workbench.colorTheme": "Visual Studio Light",
    "git.path": "/usr/bin/git",
    "files.autoGuessEncoding": true,
    "editor.minimap.maxColumn": 40,
    "editor.fontFamily": "Courier, PT Mono",
    "window.title": "${dirty}${activeEditorLong}${separator}${rootName}${separator}${appName}",
    "files.trimTrailingWhitespace": true,
    "security.workspace.trust.untrustedFiles": "open",
    "explorer.confirmDragAndDrop": false,
    "editor.minimap.enabled": false,

    "files.watcherExclude": {
        "**/.git/objects/**": true,
        "**/.git/subtree-cache/**": true,
        "**/node_modules/*/**": true,
        "**/.hg/store/**": true
    },
    "files.maxMemoryForLargeFilesMB": 8192,
    "C_Cpp.codeAnalysis.updateDelay": 1000,

    // about cpp
    "C_Cpp.default.includePath": [
        "/usr/include",
        "/usr/local/include",
        "${workspaceFolder}/**"
    ],
    "cmake.configureOnOpen": true,

    // about python
    "workbench.startupEditor": "none",
    "workbench.editorAssociations": {
        "*.ipynb": "jupyter-notebook"
    },
    "notebook.cellToolbarLocation": {
        "default": "right",
        "jupyter-notebook": "left"
    },
    // pip install yapf before using
    "python.formatting.provider":"yapf",
    // set maximum len of a single line
    "python.formatting.yapfArgs": ["--style", "{column_limit: 120}"],
    "python.linting.pylintEnabled": false,
    "[python]": {
        "editor.formatOnType": true
    },
    "terminal.integrated.enableMultiLinePasteWarning": false,
    "window.zoomLevel": -1,
    "extensions.ignoreRecommendations": true,
    "editor.quickSuggestionsDelay": 5,
    "editor.quickSuggestions": {
        "strings": "on"
    },
    "editor.suggest.localityBonus": true,
    "editor.suggest.shareSuggestSelections": true,
}
```

Sftp.json

```json
{
    "name": "code-base",
    "protocol": "sftp",
    "host": "192.168.26.209",
    "port": 22,
    "username": "root",
    "password": "660000",
    "uploadOnSave": true,
    "ignore": [
        "\\.vscode",
        "\\.git",
        "\\.DS_Store",
        "\\.history",
    ],
    "remotePath": "/home/abori/zbs"
}
```

在vscode 的命令行中，使用【sftp:download】将远程服务器上的变化拉到本地来；修改完成以后，记得在VSCode的命令中键入【sftp:upload】

扩展

* Bracket pair colorize
* C/C++
* Python
* Chinese Language
* Material Icon Theme
* SFTP
* Remote-ssh

快捷键

* VSCode左下角 “管理 / Manage” -> “键盘快捷方式 / Keyboard Shortcuts” -> 搜索 "cursorWordPartLeft" 设置成 opt + 向左，cursorWordPartLeft，搜索“后退 / Go Back“ 设置成 cmd + 向左，向右同理
* 【opt + shitft + ->】选定单词
* 【cmd + p】输入文件名 + “:” + 行数，跳转到指定文件的指定行数
* 【cmd + shift + f】打开全局搜索，搜索时【opt + cmd + w】全字匹配、【opt + cmd + c】区分大小写匹配
* 【cmd + shift + e】打开资源管理器
* 【opt + up】整行代码向上移动
* 【opt + shift + f】格式化文档，需要安装 yapf 并在 setting.json 中设置
* 【opt + z】查看日志长字符串不换行
* 【ctl + w】 多窗口切换
* 【ctl + `】显示终端

Python 调试代码

1. 选择 Python 解释器

2. 在项目根目录中创建 ./vscode/launch.json 配置文件

3. 打断点调试

    ![click_start_debug_python](https://gitee.com/aboriginalZero/blogimage3/raw/master/img/202111241017953.jpg)

    其中，F10 单步跳过，F11 单步进入函数，Shift+F11 单步跳出函数

#### zsh

下载 zsh 并更改成默认 Shell

```shell
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# 拷贝到根目录下
cp -r ohmyzsh ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
chsh -s /bin/zsh		# 将 shell 设置为 zsh
```

下载需要的插件

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

在`~/.zshrc`中配置插件

```shell
plugins=(其他的插件 zsh-autosuggestions zsh-syntax-highlighting)
```

使配置生效

```shell
source ~/.zshrc
```

如果无法下载任何东西，或者 brew update 无法到最新，尝试以下指令

```shell
git -C $(brew --repo homebrew/core) checkout master
brew doctor
```

避免每次下载东西都要 update

```shell
export HOMEBREW_NO_AUTO_UPDATE=true
```

git 无法下载 https，设置环境变量，7890 是翻墙软件所占用的端口

```shell
export https_proxy=127.0.0.1:7890
```

#### vim

vim 配置文件，创建`/root/.vimrc`

```
"显示行号
set nu
"智能缩进
set smartindent
"语法高亮
syntax on
"显示标尺，就是在右下角显示光标位置
set ruler
set rulerformat=%l,%v
"tab缩进
set tabstop=4
set shiftwidth=4
set expandtab
set smarttab
"高亮查找匹配
set hlsearch
"显示匹配
set showmatch
```

#### 自动备份

创建 cron 任务，执行 shell 脚本

```shell
#! /bin/bash
cd ai-kg
git add -all
git commit -m "by commit_all shell"
git push origin master
echo "============================="
echo "ai-kg has committed && pushed"
echo "============================="
```

通过 crontab -e 添加，其中 commit_all.sh 中的内容也用绝对路径

```
20 11 * * * /bin/bash /Users/abori/Documents/doc/commit_all.sh >> /Users/abori/Documents/doc/commit.log
```

#### Mac 使用

在键盘设置中将【键重复速率】设置成快，【重复前延迟】设置成短，这样键盘响应更快。

快捷键

* [cmd + shift + 4] 截图并保存在桌面
* [cmd + ctrl + q] 锁屏
* [cmd + opt + esc] 强制关闭应用
* [cmd + shift + .] 显示隐藏文件
* [space] 预览文件，esc 退出

macOS 的文件结构

macOS 的文件系统格式是 APFS，而 Windows 的文件系统格式是 NTFS。因为版权和商业上的原因，macOS 名义上只能读取 NTFS 格式的硬盘，不支持写入。但是我们可以使用第三方付费软件（例如 [Paragon NTFS](https://china.paragon-software.com/home-mac/ntfs-for-mac/)、[Tuxera NTFS](https://www.tuxera.com/products/tuxera-ntfs-for-mac-cn/)）来读写 NTFS 格式的硬盘，或者将移动硬盘改为兼容格式（ExFAT）。

macOS 中的硬盘也可以分区，但是不像 Windows 一样有「C 盘、D 盘、E 盘、F 盘」这种「盘符」的概念。每个磁盘只有一个名称，没有字母序号。Macintosh HD  就相当于 Windows 的 C 盘。 Macintosh HD 盘里分为 系统、资源库、应用程序、用户 4 个文件夹。

* 系统：相当于 Windows 的 C 盘中的 WINDOWS 文件夹，存放的是操作系统文件，不要进行修改。
* 资源库：存放这一些系统和软件的配置，不要随意修改。
* 应用程序：相当于 Windows 中的 Program Files 文件夹，应用软件安装在这里边
* 用户：相当于 Windows 中 C:/User/  文件夹。里边为每一个用户创建了一个用户文件夹（文件夹名为用户名称），每个用户文件夹里包含了下载、文档、桌面、图片等文件夹。

macOS 版本的软件安装包后缀名通常是 .dmg（类似于 win 下的 iso）或者是 .pkg（安装器）
