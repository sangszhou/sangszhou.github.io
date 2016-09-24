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

$? Exit status of last command

$$ Process ID of shell

$_ Name of last command.

$# number of function arguments

$@ 表示所有的参数，包括 command 本身，所以实际的参数应该是

$*

### find

```bash
find /etc -name *.jar ## 加上引号更好

find /home -user someone

// by size
-size [+-]SIZE
find . -size +12k
```

### 数组操作

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

**选择性打印**

sed -n ’10p’ file.txt 打印出第十行，打印不出就报错

-n 是 quite 模式，如果不加此参数，sed 会把原始文件打印到标准输出流上

‘10p’ 是打印出第 十行，假如没有第十行返回空，这里

sed -n ‘1,5p’ file.txt 打印出第 1 到第 10 行

**数据搜寻与替换**

`sed -i -e “/akka.tcp/ s/127.0.0.1/$HOST_IP/“ poc.conf`

找到 akka.tcp 所在的哪一行， 把 127.0.0.1 替换为 $HOTST_IP

替换模式为  sed ’s/要被替换的字符串/新的字符串/g"

-i 表示 inplace, -e command

下文提到的，还没用过的知识有：

sed -n ‘1,5p’ fileName.md

sed -n ‘1,4d’ fileName.md 删除行

## usage

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

cp -f 应该是递归创建文件目录的意思吧

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

### 将一目录下所有的文件的扩展名改为bak

```shell
for i in *.*;
    do 
        mv $i ${i%%.*}.bak;
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



