---
layout: post
title: Shell script 1
categories: [shell]
keywords: linux, shell, script
---

## 传递参数

### 通过环境变量传递

```shell
maxPermSize="1024m"
heapSize="24g"

if [[ $PROJECT_PERMSIZE != "" ]];then
    maxPermSize=$PROJECT_PERMSIZE
fi

nohup java  $JAVA_OPTS  -Djdk.logging.allowStackWalkSearch=true -Dconfig.file=$config
    -cp $assembly com.Boot 1>/dev/null 2>$home/logs/err_out.log &
```

### 直接参数传递

```shell
some_program word1 word2 word3

$0 would contain "some_program"
$1 would contain "word1"
$2 would contain "word2"
$3 would contain "word3"
```

## 相对路径

```bash
root
	configs
		apache.conf
	scripts
		start.sh
		stop.sh
        status.sh

## sh start.sh

#!/bin/bash
basedir=`dirname $0/..` ## 跳到 root 目录下

CONF_LOCATION="$basedir/configs/apache.conf"

# 注意 这里的 grep -v grep, 过滤掉 grep
running_status=`ps -ef | grep $TYPE | grep -v grep`

if [[ -n $running_status ]]; then
	echo "$TYPE is running"
	exit 0
elif [[ -z $running_status ]]
```

## default value

`parameter=${parameter:-word}`

If parameter is unset or null, the expansion of word is substituted.

Otherwise, the value of parameter is substituted.

mac 下不可以用, linux 才可用

注意 $ 符号


## regex

## basic

### nohup

nohup is a POSIX command to ignore the HUP (hangup) signal. The HUP signal is, by convention, the way a terminal warns dependent processes of logout.

Output that would normally go to the terminal goes to a file called `nohup.out` if it has not already been redirected.

在 terminal 或者脚本启动一个进程后，新进程在 terminal 关闭后会被 kill 掉。如果想把程序真正放到后台执行，那么久用 nohup 启动进程

```nohup $basedir/bin/logstash -f conf/apache.conf```

### 最大最小匹配

```
## 表示从左到右最大匹配
#  -> 最小匹配

%%  <- 最大匹配
%   <- 最小匹配
```

**Demo1**

```
new_dir="${script_dir%/*}"
dspl_dir="${new_dir%/*}"

假设 script_dir = /home/vdeadmin/dspl/scripts
那么 dspl_dir 就是 /home/vdeadmin/dspl
```

注意, 这里的 script 是一个变量, 而不是一个常量

**Demo2: contains**

不知道能不能用最大最小匹配

```
contains() {
    string="$1"
    substring="$2"
    if test "${string#*$substring}" != "$string"
    then
        return 0    # $substring is in $string
    else
        return 1    # $substring is not in $string
    fi
}
```


### $ and more

```
$$ 
Shell本身的PID（ProcessID） 
$! 
Shell最后运行的后台Process的PID 
$? 
最后运行的命令的结束代码（返回值） 
$- 
使用Set命令设定的Flag一览 
$* 
所有参数列表。如"$*"用「"」括起来的情况、以"$1 $2 … $n"的形式输出所有参数。 
$@ 
所有参数列表。如"$@"用「"」括起来的情况、以"$1" "$2" … "$n" 的形式输出所有参数。 
$# 
添加到Shell的参数个数 
$0 
Shell本身的文件名 
$1～$n 
添加到Shell的各参数值。$1是第1参数、$2是第2参数。 
```

### find

```bash
find /etc -name *.jar ## 加上引号更好

find /home -user someone

// by size
-size [+-]SIZE
find . -size +12k
```

### 数组操作 @todo

缺少对数组的操作

1. 数组定义

```
array[0]="a"
ARRAY=(one two three)

arr=(1 2 3 4 5) # 注意是用空格分开，不是逗号！！
```

2. 遍历数组

```
for var in ${ arr[@] }; do
    echo $var
done

i=0
while [ $i -lt ${ #array[@] } ]
do
    echo ${ array[$i] }
    let i++
done
```

3. 其他操作, 取值

```bash
ARRAY=(one two three)
echo ${ARRAY[*]}
echo $ARRAY[*]
echo ${ARRAY[2]}

ARRAY[3]=four
echo ${ARRAY[*]}

复制 array
cpArr=( ${ARRAY[@]} ) 必须是 @ 不能是 *, 必须加 ()
```

### sed

**sed1:** 选择性打印

```
sed -n '10p' file.txt
sed -n ‘1,5p’ file.txt 打印出第 1 到第 10 行
sed -n '/my/p' datafile
```

**sed2:** 数据搜寻和替换

```
sed -i -e “/akka.tcp/ s/127.0.0.1/$HOST_IP/“ poc.conf
```

**sed3:** 多重编辑

```
sed -e '1,10d' -e 's/My/Your/g' datafile
```

**sed4:** 选择性删除

```
sed -n ‘1,4d’ fileName.md 删除第 1 到第 4 行
```

p, d, 行数, 替换, 多重编辑, quite

**选择性打印 p(Pick)**

p 应该是 pick 的意思, d 是 delete

sed -n ’10p’ file.txt 打印出第十行，打印不出就报错

-n 是 quite 模式，如果不加此参数，sed 会把原始文件打印到标准输出流上

‘10p’ 是打印出第十行，假如没有第十行返回空，这里

sed -n ‘1,5p’ file.txt 打印出第 1 到第 10 行

**数据搜寻与替换**

`sed -i -e “/akka.tcp/ s/127.0.0.1/$HOST_IP/“ poc.conf`

找到 akka.tcp 所在的哪一行， 把 127.0.0.1 替换为 $HOTST_IP

替换模式为  sed ’s/要被替换的字符串/新的字符串/g"

-i 表示 inplace, -e command

下文提到的，还没用过的知识有：

好像数字之后不需要加 / 可以直接跟 p 或者 d, 但是字符串的前后都要加上 / 符号

```bash
sed -n ‘1,5p’ fileName.md 只查看第

sed -n ‘1,4d’ fileName.md 删除第 1 到第 4 行

sed '/My/,/You/d' datafile
#删除包含"My"的行到包含"You"的行之间的行
sed '/My/,10d' datafile
#删除包含"My"的行到第十行的内容


sed '/my/p' datafile
#默认情况下，sed把所有输入行都打印在标准输出上。如果某行匹配模式my，p命令将把该行另外打印一遍。

sed -n '/my/p' datafile
#选项-n取消sed默认的打印，p命令把匹配模式my的行打印一遍。

sed '$d' datafile
#删除最后一行，其余的都被显示

sed '/my/d' datafile
#删除包含my的行，其余的都被显示

s 命令表示替换
sed 's/^My/You/g' datafile
#命令末端的g表示在行内进行全局替换，也就是说如果某行出现多个My，所有的My都被替换为You。

sed -n '1,20s/My$/You/gp' datafile // verified, 怎么能替换整行呢?
#取消默认输出，处理1到20行里匹配以My结尾的行，把行内所有的My替换为You，并打印到屏幕上。

-e是编辑命令，用于sed执行多个编辑任务的情况下。在下一行开始编辑前，所有的编辑动作将应用到模式缓冲区中的行上。

sed -e '1,10d' -e 's/My/Your/g' datafile
#选项-e用于进行多重编辑。第一重编辑删除第1-3行。第二重编辑将出现的所有My替换为Your。
因为是逐行进行这两项编辑（即这两个命令都在模式空间的当前行上执行），所以编辑命令的顺序会影响结果。
```

上面提到了一个问题 即用 You 替换 My 结尾的一行该怎么办, 我想, 应该是使用

好像预先替换某一个行没什么办法, 下面这个写法语法没问题, 但是报错

```bash
sed -n -e '/df$/p' -e 's/.*/You/g' abc
```

### top

top -H 是把线程打出来

top -o 是按照什么标准排序, 可以使 mem, cpu 等等

top -n 1 表示刷新几次后停止

### sort

sort -nrk9

k9 表示按照第 9 列排序, -n 是把这一列当做数字来排序, -r 表示逆序排列, 大的值放前面

### awk

**awk1:** 打印文件

```
awk '{print NF，NR，$0} END {print FILENAME}' temp
```

**awk2:** 找到 $pid 对应的 CPU, MEM 使用率

```
top -n 1 -b | awk -v "pid=$pid" '$1==pid {print $9}'
```

**awk3:** 显示当前目录名

```
echo $PWD | awk -F/ '{print $NF}'   显示当前目录名
```

**awk4:** tcp connections state

```
netstat -tn | awk '/^tcp/ {++S[$NF]}; END {for(a in S) { print a, S[a]}}'
```

**awk5:** if 的用法

```
awk '{if (NR>0 && $4~/Brown/) print $0}' temp  至少存在一条记录且包含Brown
awk 'NR > 0 {print $2}'
```

问题:

awk + if, awk + 选择性, NF NR FILENAME, BEGIN END, 分隔符, 内置函数

**内置的参数**

```shell
ARGC  命令行参数个数
NF    浏览记录的域个数
AGRV  命令行参数排列
NR    已读的记录数   
ENVIRON  支持队列中系统环境变量的使用
OFS   输出域分隔符
FILENAME  awk浏览的文件名  
ORS  输出记录分隔符
FNR  浏览文件的记录数  
RS  控制记录分隔符
FS  设置输入域分隔符，同- F选项
NF  浏览记录的域个数
```

```shell
awk '{print NF，NR，$0} END {print FILENAME}' temp

awk '{if (NR>0 && $4~/Brown/) print $0}' temp  至少存在一条记录且包含Brown

NF的另一用法:  echo $PWD | awk -F/ '{print $NF}'   显示当前目录名
```

$NF 返回的是这一列的数据

**内置的函数**

```bash
index(s，t)          返回s中字符串t的第一位置

awk 'BEGIN {print index("Sunny"，"ny")}' temp     返回4

length(s)           返回s的长度

match(s，r)          测试s是否包含匹配r的字符串

awk '$1=="J.Lulu" {print match($1，"u")}' temp    返回4

split(s，a，fs)       在fs上将s分成序列a

awk 'BEGIN {print split("12#345#6789"，myarray，"#")"'

返回3，同时myarray[1]="12"， myarray[2]="345"， myarray[3]="6789"

sprint(fmt，exp)     返回经fmt格式化后的exp

sub(r，s)   从$0中最左边最长的子串中用s代替r(只更换第一遇到的匹配字符串)

substr(s，p)         返回字符串s中从p开始的后缀部分

substr(s，p，n)       返回字符串s中从p开始长度为n的后缀部分
```

注意事项

awk 后的代码要使用单引号, 不能使用双引号, 如果需要在内部使用变量, 那么需要使用 -v option

```
top -n 1 -b | awk '$1==30406 {print $9}' // correct
top -n 1 -b | awk "$1==$pid {print $9}" // error

top -n 1 -b | awk -v "pid=$pid" '$1==pid {print $9}'
```

### ps

### 在 man 中找到想要的 Option

比如 grep -A 30 不知道 -A 的意思, 这时候就可以到 man 搜索

man grep 然后

```bash
/^ *-A
```

这可以理解成正则表达式, 匹配任意个空格, 知道找到 -A

## usage

### tcpConnectStatus.sh

注意没有 $ 符号

```bash
netstat | awk '/^tcp/ {++S[$NF]}; END {for(a in S) { print a, S[a]}}'

or
netstat -tn | awk '/^tcp/ {++S[$NF]}; END {for(a in S) { print a, S[a]}}'
```

第二个效率高一些, 因为他帮忙滤掉了很多的非 tcp 请求

如果需要排序的话, 可以按照 第二列排序 sort -nk2

### show busy java threads

```bash
ps -Leo pid,lwp,user,comm,pcpu --no-headers | {
    [ -z "${pid}" ] &&
    awk '$4=="java"{print $0}' ||
    awk -v "pid=${pid}" '$1==pid,$4=="java"{print $0}'
} | sort -k5 -r -n | head --lines "${count}" | printStackOfThreads

```

PID 可以作为参数指定, 如果没给出的话, 就全部检索

-L 表示显示线程, 也就是 lwp, comm 表示程序, pcpu 是 cpu 使用率

count 可是参数, 默认情况下是 1, 用户可以指定

```bash
printStackOfThreads() {
    local line
    local count=1
    while IFS=" " read -a line ; do # -a 表示把输入作为函数
        local pid=${line[0]}
        local threadId=${line[1]}
        local threadId0x="0x`printf %x ${threadId}`"
        local user=${line[2]}
        local pcpu=${line[4]}

        local jstackFile=/tmp/${uuid}_${pid}

        [ ! -f "${jstackFile}" ] && {
            {
                if [ "${user}" == "${USER}" ]; then
                    jstack ${pid} > ${jstackFile}
                else
                    if [ $UID == 0 ]; then
                        sudo -u ${user} jstack ${pid} > ${jstackFile}
                    else
                        redEcho "[$((count++))] Fail to jstack Busy(${pcpu}%) thread(${threadId}/${threadId0x}) stack of java process(${pid}) under user(${user})."
                        redEcho "User of java process($user) is not current user($USER), need sudo to run again:"
                        yellowEcho "    sudo ${COMMAND_LINE[@]}"
                        echo
                        continue
                    fi
                fi
            } || {
                redEcho "[$((count++))] Fail to jstack Busy(${pcpu}%) thread(${threadId}/${threadId0x}) stack of java process(${pid}) under user(${user})."
                echo
                rm ${jstackFile}
                continue
            }
        }
        blueEcho "[$((count++))] Busy(${pcpu}%) thread(${threadId}/${threadId0x}) stack of java process(${pid}) under user(${user}):"
        sed "/nid=${threadId0x} /,/^$/p" -n ${jstackFile}
    done
}
```

打印堆栈信息

jstack $pid | grep $tid -A 30 // A 表示 30

### show process cpu and memory

```shell
#!/bin/bash
readonly cur_date="`date +%Y%m%d`"

readonly total_mem="`free -m | grep 'Mem'`"
readonly total_cpu="`top -n 1 | grep 'Cpu'`"

echo '**********'$cur_date'**********'
echo
echo "total memory: $total_mem total cpu: $total_cpu"
echo

for pid in `ps -ef | awk 'NR > 0 {print $2}'` ; do
    mem=`cat /proc/$pid/status 2> /dev/null | grep VmRSS | awk '{print $2 $3}'`
    cpu=`top -n 1 -b | awk -v "pid=${pid}" '$1==pid {print $9}'`

    echo "pid: $pid, memory: $mem, cpu:$cpu%"
done
```

只有极个别的符号可以在 awk 中用 $ 修饰

### find file in jar

```bash
PROG=`basename $0`

usage() {
    cat <<EOF
Usage: ${PROG} [OPTION]... PATTERN
Find file in the jar files under specified directory(recursive, include subdirectory)
Example: ${PROG} -d libs 'log4j\.properties$'
Options:
    -d, --dir       the directory that find jar files
    -h, --help      display this help and exit
EOF
    exit $1
}

ARGS=`getopt -a -o d:h -l dir:,help -- "$@"`
[ $? -ne 0 ] && usage 1
eval set -- "${ARGS}"

while true; do
    case "$1" in
    -d|--dir)
        dir="$2"
        shift 2
        ;;
    -h|--help)
        usage
        ;;
    --)
        shift
        break
        ;;
    esac
done
[ -z "$1" ] && { echo No find file pattern! ; usage 1; }
dir=${dir:-.}

find ${dir} -iname '*.jar' | while read jarFile; do
    jar tf ${jarFile} | egrep "$1" | while read file; do
        echo "${jarFile}"\!"${file}"
    done
done
```



### for given PID find the most busy thread(CPU 占用率最高的)

```bash
#!/bin/bash

if [ $# -eq 0 ];then
    echo "please enter java pid"
    exit -1
fi

pid=$1
jstack_cmd=""

if [[ $JAVA_HOME != "" ]]; then
    jstack_cmd="$JAVA_HOME/bin/jstack"
else
    r=`which jstack 2>/dev/null`
    if [[ $r != "" ]]; then
        jstack_cmd=$r
    else
        echo "can not find jstack"
        exit -2
    fi
fi

#line=`top -H  -o %CPU -b -n 1  -p $pid | sed '1,/^$/d' | grep -v $pid | awk 'NR==2'`

line=`top -H -b -n 1 -p $pid | sed '1,/^$/d' | sed '1d;/^$/d' | grep -v $pid | sort -nrk9 | head -1`

echo "$line" | awk '{print "tid: "$1," cpu: %"$9}'

tid_0x=`printf "%0x" $(echo "$line" | awk '{print $1}')`

$jstack_cmd $pid | grep $tid_0x -A20 | sed -n '1,/^$/p'
```

这里的 , 应该是或者的意思

### copy files

```
copy_files() {
    src=$1
    dst=$2
    scripts=$3

    for file in $src/*
    do
        name=`basename $file`
        dst_file=$dst/$name
        if [[ $scripts != "" ]] && [[ $name == "install.sh" ]];then
            continue
        fi
        if [[ $scripts != "" ]];then
            cp -f $file $dst_file
            chmod u+x $dst_file
        elif [[ ! -f $dst_file ]];then
            cp -f $file $dst_file
        fi
    done
}
```

cp -f If the destination file cannot be opened, remove it and create a
                 new file, without prompting for confirmation regardless of its per-
                 missions.  (The -f option overrides any previous -n option.)

### chmod

```bash

ls -l

drwxrw-xr-x staff cache

chmod u+x fileName
```

r -> read, w -> write, x -> execute

u 表示 user permission bit, x 表示可执行权限, 所以 u+x 表示给 user 添加可执行权限

644 make a file readable by anyone and writable by the owner only

chmod ug=rx fileName 表示给 user 和 group 添加 read, execute 权限

chmod u-x 去掉 owner 的执行权限

### remote login

```bash
#!/usr/bin/env bash

for host in bgl-fb{01,02,03,04,05,06,07,08,09,10}
do
    ssh -q vdeadmin@$host '  //这个单引号和 host 之间有空格
        echo "$hostname" // 会发生替换
        echo `java -version`
    '
done
```

### 单双引号

单双引号决定替换的时机

```
#!/usr/bin/env bash

for host in vdeadmin@icam-dev-app1
do
    ssh -q $host '
        echo `hostname`
    '
done

打印出 icam-dev-app1

#!/usr/bin/env bash

for host in vdeadmin@icam-dev-app1
do
    ssh -q $host "
        echo `hostname`
    "
done

打印出 macbook 的机器名
```

### check user

```bash
echo “You are logged in as `whoami`"
if [`whoami` != linuxtone]; then
  echo “Must be logged on as linuxstone to run this script."
  exit
fi

echo “Running script at `date`"
```

### transpose file

这个解法会 OOM

```bash
declare -a data // 关联数组
columnNum=0
while read line || [[ -n $line ]]
do
    words=( $line ) // 编程数组

    columnNum=$(( ${#words[@]} - 1 )) // 很重要

    for index in `seq 0 $columnNum`
    do
        data[$index]="${data[$index]} ${words[$index]}"
    done

done < file.txt

for index in `seq 0 $columnNum`
do
    echo ${data[$index]}
done
```

### Tenth line

```bash
# Solution One:
# head -n 10 file.txt | tail -n +10

# Solution Two:
# awk 'NR==10' file.txt

# Solution Three:
sed -n 10p file.txt

# In awk, the predefined variable NR stores the current line number
# This allows for an easy throwaway 1 liner solution
cat file.txt | awk '{if (NR==10) print $0}'
```

### Word Frequency

```
the day is sunny the the
the sunny is is

the 4
is 3
sunny 2
day 1
```

```bash
# Solution1:
看来在 awk 里也是可以使用  $i 的, 以前的理解还是不对
awk '{for(i=1;i<=NF;i++) a[$i]++} END {for(k in a) print k,a[k]}' words.txt | sort -k2 -nr

#Solution2:

declare -A freq # 关联数组

fields=($(cat $input | awk -F: '{$1=$1} 1'))

while read field; do
	freq[$field]=$((freq[$field] + 1))
done <<<"$(printf "%s\n" "${fields[@]}")"

for word in "${!freq[@]}"; do
	echo "$word - ${freq["$word"]}";
done | sort -rn -k3
```

### 编写个shell脚本将当前目录下大于10K的文件转移到/tmp目录下

```shell
#/bin/sh

#Programm :
# Using for move currently directory to /tmp
for FileName in `ls -l | awk '$5>10240 {print $9}'`
    do
        mv $FileName /tmp
    done
ls -al /tmp
echo "Done! "
```

### 编写shell脚本获取本机的网络地址

```bash
#!/bin/bash
#This programm will printf ip/network

IP=`ifconfig eth0 |grep 'inet ' |sed 's/^.*addr://g'|sed 's/ Bcast.*$//g'`
NETMASK=`ifconfig eth0 |grep 'inet '|sed 's/^.*Mask://g'`
echo "$IP/$NETMASK"
exit
```

### 将一目录下所有的文件的扩展名改为 bak

```shell
for i in *.*;
    do
        mv $i ${i%%.*}.bak ## 这个 .* 而不是 *.*, 因为它是 strip 的过程
    done
```

### 打印root可以使用可执行文件数  ???

```shell
echo "root's bins: $(find ./ -type f | xargs ls -l | sed '/-..x/p' | wc -l)"
root's bins: 3664
```

### addProperty

```bash
#!/bin/bash
add_property() {
    file=$1
    key=$2
    value=$3
    if [[ ! -e $file ]];then
        echo "file $file not exists"
        return -- -1
    fi
    if grep -Fq "$key=" "$file"
    then
        replace="sed -i 's#^${key}=.*#${key}=\"${value}\"#g' \"$file\""
        eval $replace
    else
        echo "$key=\"$value\"">>"$file"
    fi
}
```

```scala
#!/user/bin/env bash

cd `dirname $0` /..

SCRIPT_NAME=`basename $0`
PID=run/icam-api.pid

status() {
    if [[ -f $PID ]]; then
        echo "PID file exists, pid = `cat $PID`"
        if ps -p `cat $PID` &> /dev/null; then
            return 0
        else
            echo "process for $PID is not exist"
            return 1
        fi
     else
        echo "PID file not exist"
        return 1
     fi

     return 0
}

SCRIPT_NAME=`base $0`

if [[ $SCRIPTS_NAME == 'status.sh' ]]; then
    if status; then
        echo "process is running"
        exit 0
    else
        echo "process is not running"
    fi
fi
```
