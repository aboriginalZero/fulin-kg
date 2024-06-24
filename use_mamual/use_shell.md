# shell

shell 执行流程

1. 读取并解析命令行的输入，拿到待执行程序名称和参数；
2. 终端进程调用 fork 创建出一个子进程，子进程运行时调用 exec 来执行目标程序 cmd；
3. 正常情况下父进程阻塞，通过 wait 等待子进程结束，才能执行之后的指令；也可以通过设置子进程后台运行的方式来达到异步效果，即设为守护进程，通过 nohup cmd & 命令将该进程 cmd 挂在 systemd 进程名下，做 systemd 的子进程。

## 文本/文件

### less 查看文件

* gg 到开头，G 到末尾；
* 按下 / 键后回车，n / N 来向下/上搜索；
* 输入行号 n 并回车，以当前屏幕中的第一行为起点，跳转到第 n 行（跟用 vi 打开的第 n 行不同）；
* F：实时跟踪文件变化，类似于 tail -f 命令；
* =：显示当前文件的统计信息，包括行数、字节数等；
* &pattern：仅显示匹配某个模式的行，例如 &error 将只显示包含 "error" 的行；

### grep 查找特定文本的文件

grep 用来查找包含特定文本模式的文件，一般范式是 grep pattern file。

* -i 忽略 pattern 大小写；-w 精准匹配；-r 递归搜索；-e 支持正则表达式，可使用多个 pattern；
* -l  只打印匹配的文件名；-n 打印行号；-v 打印不匹配行；-C 3 打印匹配结果前后各 3 行；
* 用 '' 就可以搜索带 "" 的关键字。

常见方式

* 使用 zgrep 会搜索包括压缩文件在内的所有文件，通过正则表达式搜索同时包含 key1 和 key2 的行

    ```shell
    zgrep 'key1.*key2\|key2.*key1' /var/log/zbs/zbs-chunkd.log.2023*
    ```

* ^ 匹配行首，$ 匹配行尾，搜索只有字符 ctx 的行

    ```shell
    grep “^ctx$” file 
    ```

* 搜索指定字符并打印后三行，忽略匹配串大小写并带行号

    ```shell
    grep -n -i 'LIBMETA ADDR UPDATE' zbs_test.INFO -A 3
    ```

* --include 指定或 --exclude 排除某些文件

    ```shell
    grep "main()" . -r --include *.{h, cc}
    ```

* 编写脚本时，可以用 -q 设置静默，返回值 0 表示匹配成功，其他表示失败

    ```shell
    grep -q "$pat" $file		# 等价于 grep "$pat" $file &> /dev/null
    if [ $? -eq 0 ]; then
    	echo "match success"
    fi
    ```

### find 查找特定属性的文件/目录

find 用来查找特定文件属性如文件类型、文件名、文件大小的文件/目录。一般范式是 find /path pattern。

* -type f 指定文件类型，f 文件、d 目录、l 硬链接；
* -iname Who 忽略模式大小写匹配；
* -maxdepth 2 限制搜索范围在给定目录与其子目录；
* -path "*/yw" 限制匹配路径中包含 yw；
* -size +1G 指定文件大于 1G；
* -delete 删除匹配的文件/目录；
* -path ./dir -prune 跳过 ./dir 查找；
* -print0 用 0(NULL) 来分割查找到的元素，避免某些文件名中出现空格解析出错；

常见方式

* 对查找结果执行命令，

    ```shell
    find . -type f -iname *.txt -exec cat {} \;
    find . -type f -iname *.txt | xargs cat	# 效果一样但效率更低
    ```

    其中花括号{}代表匹配到的文件名，以及必须对分号进行转义，否则 shell 会将其视为 find 命令的结束，而非 cat 命令的结束

* 查找大于 1G 的文件并删除

    ```shell
    find . -type f -size +1G -delete
    ```

* 跳过 .git 和 .vscode 目录再查找

    ```shell
    find . \( -path "./.vscode*" -o -path "./.git" \) -prune -o -name "*.txt"
    ```

    其中 () 内部左右要留有空格，-o 表示逻辑或，-path路径不能在结尾加 /，第二个用 -o 而不是 -a

### tr 替换字符

tr（translate）对来自 stdin 的内容进行字符替换、字符删除以及重复字符压缩，一般范式是 tr set1 set2。tr 将来自 stdin 的输入字符按照位置从 set1 映射到 set2（set1 中的第 i 个字符映射到 set2 中的第 i 个字符，不足重复补最后一位），然后将结果写入stdout，tr 只能通过 stdin 接受输入而无法通过命令行参数接收。

* 替换字符

    ```shell
    echo 12345 | tr '0-9' '9876543210'
    ```

* 删除字符，支持字符类如 [:digit:] [:space:] [:lower:]

    ```shell
    echo "Hello 123 world 456" | tr -d '[:digit:]' 		# 输出 Hello world
    ```

* 删除不在补集中的所有字符

    ```shell
    echo hello 1 yw  | tr -d -c '[:lower:] \n'	# 输出 hello  yw
    ```

### sed 替换文本

sed（stream editor）支持正则表达式文本替换，一般范式是 sed 's/pattern/replace/' file，sed 将 s 之后的字符视为命令分隔符，对 file 文件中第一个 pattern 用 replace 替换。

* 从第 2 个匹配开始全局替换

    ```shell
    sed 's/pattern/replace/2g' file
    ```

* 指定命令分隔符为冒号、用 ; 分隔多个模式、用 -i 让修改后的数据替换原始文件

    ```shell
    sed 's:pattern:replace:;s:pattern2:replace2:g' -i file
    ```

* 删除空行，没有前缀 s

    ```shell
    sed '/^$/d' file
    ```

* 用&指代模式所匹配到的字符串

    ```shell
    echo this is an example | sed 's/\w\+/[&]/g' # 输出 [this] [is] [an] [example]
    ```

### cut  按列切分文本

cut 支持按列切分文本，可用于处理使用固定宽度字段的文件、CSV 文件或是由空格分隔的文件如标准日志文件。

* 查看每行的[n,m]个字符

    ```shell
    cut -c n-m file
    ```

* 显示第 2 3 列

    ```shell
    cut -f 2,3 file
    ```

* 以 ; 为分隔符显示除第 2 3 列外的其他列

    ```shell
    cut -f 2,3 -d ";" --complement file
    ```

### awk 按行处理文本

awk（三位创始人的名字首字母）是一个能够解释并执行程序的解释器，可以处理数据流，支持字典、递归函数、条件语句等功能。一般范式是 awk 'BEGIN {print "start"} pattern {commands} END {print 'end' }' file，awk 以逐行的形式处理文件，先执行 BEGIN 内容，然后对匹配 pattern 的行执行 commands，最后执行 END 内容。

awk 内置变量 

* NR：当前行号；
* NF：列数，即字段数量，默认字段分隔符是空格；
* $0：当前整行内容
* $i：第 i 个列
* $(NF-1)：表示倒二个列

awk 内置字符串处理函数

* length(str)
* index(str, search_str)，返回第一次出现的下标
* split(str, arr, delimeter)，将生成的字符串放入数组 array

pattern 处除了用正则表达式还可以用其他条件筛选行

* 筛选数据，以 : 为分隔符打印 /etc/passwd 中前 5 行的登陆 ID 和对应的用户名

    ```shell
    awk 'BEGIN {FS=":"} NR <= 5 {nam[$1]=$5} END {for (i in nam) {print i, nam[i]}}' /etc/passwd 
    ```

* 多次分割，先使用空格分割，然后对分割结果再使用 "," 分割，输出分割结果中的第 1、3 项

    ```shell
    awk -F '[ ,]' '{print $1, $3}' file
    ```

* 多条件筛选，筛选出第一列大于 10 并且第二列字符串等于 "true" 的每行内容

    ```shell
    awk '$1 > 10 && $2=="true" {print $0}'
    ```

### sort 排序与 uniq 去重

sort 用法

```shell
sort file1.txt file2.txt > sorted.txt
# -n 表示按照数字顺序（区别于字母表顺序），-r 表示逆序，-k 2表示按第 2 列排序
sort -nrk 2 data.txt
# 如果需要特定范围的一组字符，按照第二列中的的第 4-5 个字符，-b 表示忽略前导空格
sort -bk 2.3,2.4 data.txt
```

 uniq 的使用

```shell
# 找出已排序文件中不重复的行
sort file1.txt file2.txt | uniq
# 统计各行在文件中出现的次数
sort unsorted.txt | uniq -c
# 跳过前 3 个字符，指定后续的 2 个字符的作为唯一性的判断
sort data.txt | uniq -s 3 -w 2
```

### 综合用法

如果有一个包含字母和数字的文件，我们想计算其中的数字之和

```shell
cat 1.txt
fst 1
sec 2
thd 3
```

利用tr的-d选项删除文件中的字母，然后将空格替换成+，并用 $[] 求和

```shell
cat 1.txt | tr -d [a-z] | echo "total: $[$(tr ' ' '+')]"
```

移除多余的空格

```
tr -d ' '
sed 's/[ ]\+/ /g'
```

打印 [M, N] 行之间的文本:

```
M=2;N=4;
awk "NR==M,NR==N" M=$M N=$N filename	# 通过空格分隔将外部变量传递给的 awk
cat filename | head -n $N | tail -n $[$N - $M + 1]
```

忽略空格递归统计某个文件夹下 .py 结尾文件的所有代码行数：`find . -name "*.py" -print0 | xargs -0 cat | grep -v ^$ | wc -l`，print0 的做法可以避免文件名中含有空格时解析出错

指定目录下按文件大小排序显示：`du -sh /dir_path | sort -rn`

综合统计

```
awk '{print $1}' access.log | sort | uniq | wc -l
```

取日志的第 1 列内容，排序，去重（跟数据库中的unique一样，去重前都要排序），统计条数

```
awk '{print $2}' access.log | sort | uniq -c | sort -rn | head -n 3
```

取日志的第 2 列内容，排序，去重，再反向降序排序，并显示前3条记录， `sort -rn`（r 表示逆向排序， n 表示按数值排序）

### 正则表达式

1. 位置标记：^ 指定 pat 必须在首部、$ 指定 pat 必须在尾部；
2. 标识符：普通字符必须匹配、. 匹配任意一个字符、[] 匹配括号中的任意一个字符、[^] 匹配不在括号中的任意一个字符；
3. 数量修饰符：+ 大于等于 1 次、* 大于等于 0 次、? 0 或 1 次、{n, m} 匹配 n 到 m 次之间；
4. 其他：() 将括号内容当作一个整体、｜逻辑或、\ 转义符。

给出几个经典例子

* 匹配  IP 地址，重复 3 次 0-255 的匹配，最后一部分由于没有 . ，所以要额外匹配一次

    ```shell
    ([01]?[0-9][0-9]?|2[0-4][0-9]|25[0-5]) # 匹配 0-255
    ^(([01]?[0-9][0-9]?|2[0-4][0-9]|25[0-5])\.){3}([01]?[0-9][0-9]?|2[0-4][0-9]|25[0-5])$
    ```

* 匹配日期，如 1949-10-1

    ```shell
    ^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$
    ```

## 重定向、管道、xargs

命令结果重定向

* 输出重定向，终端上只能看到标准错误：  `cmd 1>output.log`，1 可以省略
* 错误重定向，终端上只能看到标准输出：`cmd 2>error.log`
* 标准输出和标准错误都重定向到 file，终端上看不到任何信息：`cmd >output.log 2>&1`
* 屏蔽命令任何输出：`cmd >/dev/null 2>&1`
* 借助重定向清空文件内容：`> log`

0 stdin 标准输入，1 stdout 标准输出，2 stderr 标准错误，默认情况下，正常输出(stdout)和错误信息(stderr)都会显示在屏幕上。

```shell
cmd 2>stderr.txt 1>stdout.txt	# stderr和stdout分别重定向到不同的文件
cmd 2>&1 alloutput.txt			# stderr和stdout 重定向到同一个文件，
cmd &> output.txt				# 等价于上条命令
```

默认情况下，重定向操作针对的是标准输出。

```shell
 >  等价于 1>
 >> 等价于 1>>
```

stdout作为单数据流 (single stream)，可以被重定向到文件或是通过管道传入其他程序，但是无法两者兼得。所以要借助 tee，tee命令从stdin中读取，然后将输入数据重定向到stdout以及一个或多个文件中。

在下面的代码中，tee命令接收到来自stdin的数据。它将stdout的一份副本以 -a 追加方式写入文件 out.txt，同时将另一份副本作为后续命令的stdin。命令cat -n为从stdin中接收到的每一行数据前加上行号并将其写入stdout:

```shell
cat a* | tee -a out.txt | cat -n
```

管道符连接的多个命令是并行执行的，只不过有些命令会阻塞等待输入以达到同步，比如`cat a.txt | grep "pattern"`



xargs

通过 xargs 命令可以把输入的内容转为参数提供给不支持管道的命令，如递归搜索当前目录下以 .txt 结尾的文件，并删除：`find . -name “*.txt“ | xargs rm -f`

Unix命令可以从标准输入(stdin)或命令行参数中接收数据，利用管道将一个命令的标准输出传入到另一个命令的标准输入，另一种方法就是使用反引号执行命令，然后将其输出作为命令行参数，如 

```
gcc `find '*.c'`
```

cmd `` 等价于 cmd $()，子 shell 法，实际是开了一个子进程，如果想要保留输出的空格和换行符，必须加双引号；

这种方法在很多情况下都管用，但是如果要处理的文件过多，你会看到错误信息: Argument list too long。xargs命令可以解决这个问题。xargs命令从stdin处读取一系列参数，然后使用这些参数来执行指定命令。它能将单行或 多行输入文本转换成其他格式，例如单行变多行或是多行变单行。

xargs命令应该紧跟在管道操作符之后，接受来自stdin的输入，将数据解析成单个元素，然后调用指定命令并将这些元素作为该命令的参数。xargs默认使用空白字符分割输入并执行/bin/echo。如果文件或目录名中包含空格(甚至是换行)的话，可以指定分隔符

```shell
echo "splitXsplitXsplitXsplit" | xargs -d X -n 2
split split
split split
```

限制传入参数个数

```shell 
# 由xargs添加的参数只能被放置在指定命令的尾部
cat args.txt | xargs -n 2 ./cecho.sh	
# xargs有一个选项-I，可以用于指定替换字符串，这个字符串会在xargs解析输入时被参 数替换掉。如果将-I与xargs结合使用，对于每一个参数，指定命令只会执行一次。使用-I的时候，命令以循环的方式执行。如果有3个参数，那么命令就会连同{}一起被执行3次，此时作为中间参数
cat args.txt | xargs -I {} ./cecho.sh -fst_arg {} -last_arg
```

## 基础内容

### 特殊符号

1. 分号; ，分隔多条命令；
1. 转义符 \，把特殊字符变成普通字符；
1. 美元符 $，变量符，`${var}` 表示变量值，`${#var}`表示变量值的长度；
1. 倒引号 `，引号内的字符串当做 shell 命令行解释执行，等价于$(cmd)；
6. 双引号 ""，对美元符 $ 、转义符 \、倒引号 `保留特殊功能的字符串引用形式，如果字符串中有引用变量得用双引号；
6. 单引号''，被引起的字符全部做普通字符，Shell 会默认扩展没有引号或是出现在双引号(")中的通配符，单引号能够阻止shell扩展*.txt，对于单引号的字符串，想要传递变量需要加上单引号。

### 变量

在 Shell 中定义的变量默认就是全局变量，即使是在 shell 函数中定义的变量（如果想要声明局部比纳凉，可以用 local 定义作用域仅限于函数内部的变量）。全局变量在当前的整个 Shell 进程中都有效，对其它 Shell 进程和子进程都无效。

* 声明变量：变量名=值（等号前后不留空格，前后留空的等号是等量关系测试）

* 引用变量：${变量名}

* 显示变量：echo ${变量名}

* 清除变量：unset 变量名

#### 特殊变量

1. $$ ：本进程的 PID
2. $0 ：shell 程序的名称
3. $? ：shell 程序返回值，退出状态
4. $! ：上一个进程（命令）的 PID
5. $n ：第 n 个参数
6. $# ：传送给 shell 程序的位置参数数量
7. $@ ：“参数1”，“参数2”…形式保存的参数
8. $* ：调用 shell 程序时所传送的全部参数的单字符串， “参数1”，“参数2” … 形式保存的参数

#### 环境变量

环境变量均为大写，通过 export 导出的环境变量是临时的，只对当前 Shell 进程以及它的子进程有效

* 设置环境变量：环境变量名=值、export 环境变量名

* 显示环境变量：env、echo ${环境变量名}

* 清除环境变量：unset 环境变量名

持久化的环境变量需要写在配置文件中，所在路径有三个：

* /etc/profile 和 /etc/profile.d

    存放一些全局变量，每个用户登录时都会读取 /etc/profile 和 /etc/profile.d 目录下的所有 .sh 文件

    通常用以设置一些 Shell 变量 PATH、USER、 HOSTNAME 和 HISTSIZE 等。更改后，需要退出终端再重新登录才能生效。

* ~/.bash_profile 或 ~/.profile

    各用户专属存放 shell 信息的文件，当用户登录时，该文件仅执行一次，通常只是读取 ~/.bashrc的内容。更改后，source ~/.profile 就有效，

* ~/.bashrc

    各用户专属文件，当用户登录以及每次打开新的 shell 时该文件被读取。通常设置一些提示文本、字体、命令别名，更改后，source ~/.bashrc 或打开新终端就能生效，对 Shell 的修改基本放在这就行。

### 语法

#### 浮点数

借助 bc 执行浮点数运算并使用一些高级函数

```shell
echo "4 * 0.56" | bc
echo "sqrt(100)" | bc
```

#### 数组

```shell
arr=(a1 a2 a3);
i = 2;
echo ${arr[$i]};	# 打印指定值 a3
echo ${arr[*]};		# 打印所有值 a1 a2 a3
echo ${#arr[*]};	# 数组长度 3
```

#### 字典

```shell
declare -A dict;				# 创建字典
dict=([k1]='v1' [k2]='v2');	
echo "${dict[k1]}";				# 打印指定值
echo ${!dict[*]};				# 打印所有键
echo ${dict[*]};				# 打印所有值
```

#### 循环

```shell
for i in {1..6}; # 比 for i in `seq 1 6` 执行速度快
do 
	echo $i
done

for ((i=1;i<=6;i++)) {
	echo $i;
}

i=0;sum=0;
until [ $i -ge 5 ];	# 不满足条件时才会继续执行，与 C++ 不同
do 
	let sum=sum+i;	# 等价于 sum=$[$sum+$i];	
	let i++;		# let 用以执行基本的整数算术操作，此时变量名之前不需要再添加 $
done;
echo $sum
```

重复执行命令直到成功为止

```shell
repeat() {
	while true
	do
		$@ && return	# 短路判断，$@ 执行成功才会执行 return
	done
}
repeat() { while true; do $@ && return; done }
# true 是作为 /bin 中的一个二进制文件来实现的，每循环一次 shell 就得生成一个进程
# 所以可以使用 shell 内建命令:，它的退出状态总为 0，这样执行速度更快
repeat() { while :; do $@ && return; done }
```

#### 判断

在 Shell 中判断为 true 返回 0 ，否则返回 1。以下 3 种判断形式等价：

```shell
test 1 -lt 4	# test 是一个外部程序，需要开对应的子进程
[ 1 -lt 4 ]		# [] 是 Bash 的一个内部函数，执行效果更高
[[ 1 -lt 4 ]]	# 双中括号的形式支持正则判断
```

* 整数：-lt 小于、-le 小于等于、-gt 大于、-ne 不等于、-eq 等于；
* 字符串：test s 判断字符串 s 非空、test s1=s2 判断字符串 s1 等于 s2、test s1!=s2 判断字符串 s1 不等于 s2、test -z s 判断字符串为空串、test -n 判断字符串长大于 0
* 文件：-f 存在且是普通文件、-d 存在且是目录、-r 存在且可读、-w 存在且可写、-x 存在且可执行、-s 存在且字节数大于 0
* 逻辑：-a 或 && 表示逻辑与、-o 或 || 表示逻辑或、! 表示逻辑非

```shell
if [ $# -eq 0 ]
then
	echo “输入了 0 个参数”
elif [ $# -lt 1 ]
then echo “输入了多个参数”
else echo “输入了 1 个参数”
fi
```

#### 函数

```shell
# 定义函数
function f1() {};
f2() {
    if [ $# -ne 2 ];
    then 
    	echo "need 2 params";
    	exit -1
    fi
    echo $2;
};

# 使用函数 
f1;
f2 arg1 arg2;
```

#### 从控制台输入

从控制台输入 2 个单词

```shell
read var1 var2
```

从控制台输入读取 2 个字符并存入 var

```shell
read -n 2 var 
```

#### 输出到控制台

在 printf 中使用格式化字符串来指定字符串的宽度、左右对齐方式等

```shell
#!/bin/bash 
# %-5s 表示左对齐、宽度为 5 的字符串
# %-4.2f 表示左对齐、整数宽度为 4、小数宽度为 2 的浮点数
printf  "%-5s %-10s %-4s\n" No Name  Mark
printf  "%-5s %-10s %-4.2f\n" 1 Sarath 80.3456
printf  "%-5s %-10s %-4.2f\n" 2 James 90.9989
printf  "%-5s %-10s %-4.2f\n" 3 Jeff 77.564

# 可以得到格式化的输出
No    Name       Mark
1     Sarath     80.35
2     James      91.00
3     Jeff       77.56
```

要打印彩色文本，可输入如下命令：

```shell
echo -e "\e[1;31m This is red text \e[0m"
```

其中\e[1;31m是一个转义字符串，可以将颜色设为红色，\e[0m将颜色重新置回。重置=0，黑色=30，红色=31，绿色=32， 黄色=33，蓝色=34，洋红=35，青色=36，白色=37。

设置终端前/背景色

```shell
tput setf 7		# 前景，0-7
tput setb 0		# 背景，0-7 
```

### 程序调试

set -u  脚本在头部加上它，遇到不存在的变量就会报错，并停止执行

set -x 打印调试信息如执行参数和执行命令，与 set +x 配合使用，在配合 ./xxx.sh 2>&1 | tee log 可以将输出调试信息写入指定文件，将环境变量 PS4 设置为 '$LINENO: ' 可以显示每行的行号

set -e 脚本只要发生错误，就终止执行，与 set +e 配合使用。但是它不适用管道命令，用管道连接的一条大命令只要最后一个子命令不失败，管道命令总是会执行成功。所以可以用 set -eo pipefail 来解决这种情况

set -E 一旦设置了-e 参数，会导致函数内的错误无法被 trap 命令捕获。-E 可以纠正这个行为，使得函数也能继承 trap 命令（用于响应系统信号）

常用的 set 组合	set -Eeuxo pipefail

防止错误积累，遇到错误不继续执行之后的代码（默认是会）的三种写法（虽然用 set -e 就能防止错误积累）

```shell
# 写法一
cmd || { echo "cmd failed"; exit 1; }
# 写法二
if ! cmd; then echo "cmd failed"; exit 1; fi
# 写法三
cmd
if [ "$?" -ne 0 ]; then echo "cmd failed"; exit 1; fi
```

如果两个命令有继承关系，只有第一个命令成功了，才能继续执行第二个命令，那么就要采用下面的写法

```
cmd1 && cmd2	
```

Debug 相关

```
bash -xv xxx.sh		# 输出每一行语句运行结果前，会先输出该行语句
echo "This is line $LINENO"		# 变量`LINENO`返回它在脚本里面的行号
```

### 多核 CPU 使用

通过 parallel 库利用多核 CPU 加速单线程 Linux  命令如 grep, wc, awk, sed

```shell
apt-get install parallel

grep pattern bigfile.txt
cat bigfile.txt | parallel --block 10M --pipe grep 'pattern'

cat rands20M.txt | awk '{s+=$1} END {print s}'
cat rands20M.txt | parallel --pipe awk '{s+=$1} END {print s}'

sed s/old/new/g bigfile.txt
cat bigfile.txt | parallel --pipe sed s/old/new/g
```

### 快捷键

`(cmd &)`，后台运行程序

`cd - `，返回刚才呆的目录

`history | grep 'echo'`，查看之前使用的echo相关指令

`yes | your_cmd`，自动化一些软件的安装

特殊变量 `$$`记录当前进程的 PID

* shell 脚本中加上`set -u`可以在遇到不存在的变量直接报错，并停止运行
* 复制文件夹到远程：`scp -P 80 -r /dir root@hostname:/dir`
* 检索历史命令：`ctrl + r`，下一个匹配项`ctrl + r`
* 光标回到行头 / 行尾：`crtl + a / e`
* 中断任务：`ctrl + c`
* 挂起任务：`ctrl + z`，通过`fg`恢复
* 查看某进程占用内存、CPU、子线程数：`top -Hp $(pidof zbs-taskd)`
* 给命令起别名，在`~/.bashrc`中添加`alias ll='ls -lt'`，查看已有的别名列表`alias -p`
* `cmd + w` 在命令行中往前按单词删除

## 系统资源

### 实用

1. 查看进程执行卡住位置：cat /proc/`pidof zbs-metad`/stack
1. 查看进程中各线程使用 CPU：top -H -p `pidof zbs-metad `，按 f 选中 P (Last Used Cpu)
1. 查看最近 20 条日志： journalctl -n 20 -u zbs-metad；
2. 实时显示最新日志： journalctl -f /usr/sbin/crond；
3. 查看内核日志： journalctl -k，等同于 dmesg -T；
4. 查看 CPU 使用：没有 htop 时使用 mpstat -P ALL 1 可以看各 CPU 的负载情况；
5. 查看磁盘读写情况：iostat -xm 1 每秒打印一次磁盘流量，rMB/s 每秒从设备读取的扇区数（带宽），r/s: 每秒完成的合并后读请求次数（iops）；
6. 查看网卡流量大小：sar -n DEV 1 每秒打印一次网卡流量，rxpck/s: 每秒接收的数据包数量，rxkB/s: 每秒接收的数据量；
7. 查看 TCP 流量：sar -n TCP,ETCP 1， active/s 每秒本地发起的TCP连接数，passive/s 每秒远端发起的TCP连接数， retrans/s 为每秒重传的包数；
7. 查看系统上一次启动时间：last reboot。

### 硬件环境

* CPU： `lscpu`，如 Intel(R) Core(TM) i7-7820X CPU @ 3.60GHz

* 内存： `free -h ` ，以 GB 为单位展示数据

* 磁盘：`df -h`，以 GB 为单位展示数据

* GPU：`lspci`，查看 PCI 搭载设备中有显卡信息 

### 软件配置

`uname -a`：查看系统内核版本以及系统名称`Linux VM-0-6-ubuntu 4.15.0-88-generic x86_64 GNU/Linux`

`getconf PAGE_SIZE`：获取页面大小4KB

### 进程

获取进程 PID，pidof 或 pgrep

查看进程的环境变量，一般用于在配置时判断环境变量是否给对

```shell
cat /proc/${PID}/environ | tr '\0' '\n'	# 将其中的 \0 替换成 \n
```

终止进程，发送 SIGTERM 信号，进程有机会执行清理操作，如关闭文件、释放资源等。无法杀死那些不响应 SIGTERM 信号的僵尸进程。

```
kill PID
```

参数`-9`强制杀死进程，发送 SIGKILL 信号，进程会立即终止,无法执行任何清理操作

```
kill -9 $(ps -ef | grep dst_user) 
```

杀死指定用户dst_user的所有进程

* `ps`：进程查看
* `top`：监控进程，监控CPU，内存使用率

Linux 环境下最多能有多少线程

用`ulimit -a`查看

* 最大文件数：1024
* 栈大小：8M
* 最大用户进程数：31378，最大 PID：32768

用`top`查看

* 内存：8G
* 交换空间：2G

在Ubuntu18.04中，使用36位来表示虚拟地址空间，使用48位来表示物理地址空间（通过`cat /proc/cpuinfo`查看），当37-48位为0表示用户空间，为1表示内核空间，内核空间被所有用户进程共享使用，一个进程被分配的虚拟内存有$2^{36}=64G$，分配给线程的调用栈大小是$2^{23}=8M$，所以一个进程可以创建大约$2^{36} /2^{23}=64G/8M=8192=2^{13}$个线程。

如果想要尽可能多提高线程数，先调小栈空间，此时受限于最大用户进程数和最大 PID。再之后就可能受限于 CPU，CPU 被占满导致的系统卡死。

Linux下如何检测内存泄露？

每隔一段时间检查内存状态`ps -aux | grep -E program_name | PID`，或者用top命令查看，哪个进程所占内存越来越大。

### 文件

计算校验和 md5sum

```shell
md5sum file1 file2 ...	# 32 个字符的十六进制串
sha1sum file1 file2 ... # 40 个字符的十六进制串
```

校验和是从文件中计算得来的。对目录计算校验和意味着需要对目录中的所有文件以递归的 方式进行计算。

加密技术主要用于防止数据遭受未经授权的访问。和校验和算法不同，加密算法可以无损地重构原始数据。

```shell
gpg -c filename			# 用 GNU 隐私保护加解密
gpg filename.gpg		
base64 file > output 	# 用 base64 加解密
base64 -d file > output
```

TODO 求两个文本文件的交集与差集、查找并删除重复文件

打包/解压

```shell
tar -cvf 压缩包名 待压缩目录
gzip -9 压缩包名 待压缩目录		# 最大程度压缩
tar -xvf 解压包名 -C 解压路径
unzip 解压包名
# 解压 zbs_meta.log.gz
gzip -d 解压包名 
```

### 网络

1. 查看路由表： route -n ；

2. 查看本机网络接口属性如 IP 地址：ifconfig；

3. 开关网络接口：ifup/ifdown eth0；

4. 添加 DNS：echo NameServer IPAddr >> /etc/resolv.conf；

5. 添加主机名到 IP 的映射：echo IPAddr NameServer >> /etc/hosts；

6. 获取网关地址：ip route | grep default；

7. 添加默认网关： route add default gw 192.168.0.1 eth0；

8. 阻止 1.2.3.4 访问本机：iptables -I INPUT -s 1.2.3.4 -j DROP ；

9. 配置静态 IP：vi /etc/sysconfig/network-scripts/ifcfg-eth0，然后通过 systemctl restart network 重启网络服务

    ```
    BOOTPROTO="static"
    ONBOOT="yes"
    IPADDR=SMTX_ManageIP_address
    NETMASK=SMTX_ManageIP_netmask
    GATEWAY=SMTX_ManageIP_gateway
    ```


**netstat 分析网络流量**

常用方式

1. 查看所有监听端口：netstat -lntp
2. 查看已经建立的 TCP 连接：netstat -antp

参数说明

    -a：将所有的连接、监听、Socket数据都列出来（默认情况下，不会列出监听状态的连接）
    -t：列出tcp连接
    -u：列出udp连接
    -n：将连接的进程服务名称以端口号显示（如Local Address会换成10.0.2.15:22）
    -l：列出处于监听状态的连接
    -p：添加一列，显示网络服务进程的PID（需要root权限）
    -i：显示网络接口列表，可以配合ifconfig一起分析

打印信息中如果 Recv-Q（已接收还未递交给应用）和 Send-Q （接收方未确认的数据）长时间不为 0 可能存在问题。

定位分布式系统中服务之间的依赖关系：

1. 查看用到本地某特定（端口上）服务的地址：netstat -antp | grep zbs-taskd；
2. 在客户端机器上用 netstat -antp | grep :port 来找到是哪个进程与之连接

**arp 分析 ARP 缓存**

* 查看arp 缓存： arp -a $ip（这个 IP 是当前机器要去访问的 IP）
* 每隔 1s 刷新查看一次 arp -a 的内容：watch -n 1 arp -a
* 删除 arp 缓存： arp -d $ip 

**tcpdump 抓包**

* 抓取 eth0 网卡上 9877 号端口的 TCP 数据包：tcpdump -i eth0 tcp port 9877
* 抓取 port-mgt 网卡上的 ip 为192.168.87.101 的 arp 包： tcpdump arp -i port-mgt | grep 192.168.87.101
* 抓取 port-mgt 网卡上的 ip 为192.168.87.101 的 arp 包：tshark arp -i port-mgt -R "ip.addr==192.168.87.101"

### 存储

du（disk usage） 提供文件/目录磁盘使用信息

列出当前目录及其子目录中非 txt 文件的各个文件大小du -ah --exclude ".*txt" --max-depth 2 .

找出指定目录中最大的 10 个文件：find . -type f exec du -h {} \; | sort -nrk 1 | head -10

du 统计的是文件字节数，而非文件占用的磁盘空间（磁盘空间以块为单位分配，1字节的文件也会占据 4 字节的磁盘空间）

df（disk free）提供磁盘可用空间信息，df -h /dir 会输出该目录所在分区及其可用磁盘空间情况

lsblk（list block devices） 提供块设备信息，

使用 dd 生成任意大小的文件

```shell
dd if=/dev/random of=rand_file bs=1M count=2	# 总大小为 2M 的随机文件
dd if=/dev/random of=/dev/null bs=1G count=100 	# 实际在测内存速度
```

如果不指定输入参数(if)，dd会从stdin中读取输入。如果不指定输出参数(of)，则dd 会使用stdout作为输出。

## 应用

### 定时全量备份

在 linux 中做定时全量备份，且每次备份按照时间戳命名，定期删除超过指定时间的历史备份文件。编写 backup.sh 如下：

```shell
#！/bin/sh
# 将数据 copy 到备份目录，先 copy 是因为怕在压缩的过程当中源文件产生变化
cp -r /dir /tmp/back 
# 将 copy 过来的 /home/back/py 文件目录进行压缩，使用日期当做新压缩文件名字存入到 /home/back 中
tar -czvf /home/backup/$(date +%Y%m%d).tar.gz /tmp/back
# 删除copy过来的文件，保留压缩过的文件就好
rm -rf /tmp/back  
# 查找并删除超过30天的.tar.gz文件
cd /home/backup
find ./ -mtime +30 -name "*.tar.gz" -exec rm -rf {} \;  
```

给予权限并通过 crontab 定时执行

```shell
chmod -R 777 backup.sh 
crontab -e  
# 在 crontab 中设定每天 23 点 59 分运行 backup.sh 脚本
59 23 * * * /home/back/backup.sh
systemctl restart crond.service
```

### 快速制作大文件

快速制造一个大文件：`tailf/cat file.txt >> file.txt`，（TODO 奇怪，怎么只会多项式复制，而非阶乘式）

具体流程：

1. 打开`file.txt`，准备在文件尾部追加内容。

2. 将`cat`命令的标准输出指向`file.txt`文件。

3. `cat`命令读取`file.txt`中的一行内容并写入标准输出（追加到`file.txt`文件中）。

4. 由于刚写入了一行数据，`cat`命令发现`file.txt`中还有可以读取的内容，就会重复步骤 3。

### 快速清空大文件

线上服务如果出现死循环写，日志很有可能撑爆磁盘，这时需要快速清空一个大文件，但用 rm 的话又需要该服务释放文件句柄，更合理的做法是：`echo "" >  xxx.log` 或 `cat xxx.log > xxx.log`

具体流程：

1. shell 打开`file.txt`并清空其内容。
2. shell 将`cat`命令的标准输出指向`file.txt`文件。
3. shell 执行`cat`命令，读了一个空文件。
4. `cat`命令将空字符串写入标准输出（`file.txt`文件）。

### 并行 ping 

(cmd)& 意味着开一个子 shell 运行 cmd 并挂在后台，通过 wait 同步等待所有的子进程结束

```shell
for ip in 192.168.0.{1..255};
do
	(
		ping $ip -c2 &> /dev/null;
		if [ $? -eq 0 ];
		then echo $ip is alive;
		fi
	)&
	done
wait
```
