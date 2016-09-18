---
layout: post
title: Shell script 2
categories: [shell]
keywords: linux, shell, script
---

## What is the use of a shebang line

Shebang line at top of each script determines the location of the engine which is to be used in order to execute the script.

## What are the four fundamental components of every file system on linux

bootblock, super block, inode block and  datablock

**Super block** contains all the information about the file system like size of file system, 
block size used by it,number of free data blocks and list of free inodes and data blocks.

**boot block** contains a small program called “Master Boot record”(MBR) which loads the kernel  during system boot up.

**innode block** contains the **inode** for every file of the file system along with all the file attributes except its name.

## How will you print the login names of all users on a system?

/etc/shadow file has all the users listed.

```
awk –F ‘:’ ‘{print $1} /etc/shadow’|uniq -u
```

## Given a file find the count of lines containing word “ABC”

```
grep –c  “ABC” file1
```

## How will you emulate wc –l using awk?

```
awk ‘END {print NR} fileName’
```

## 网络命令都有哪些

telnet

nc -l port; nc -z host port; nc -c

nslookup

netstat 

## How will you copy file from one machine to other

We can use utilities like “ftp” ,”scp” or “rsync” to copy file from one machine to other.

## What are zombie processes

These are the processes which have died but whose exit status is still not picked by the parent process. These processes even if not functional still have its process id entry in the process table.

Linux 中的僵尸进程

## What is the difference between $$ and $!?
   
$$ gives the process id of the currently executing process whereas $! shows the
process id of the process that recently went into background.

## 递归创建文件夹

```
mkdir -p
```

## 从“标准输入”读入n次字符串，每次输入的字符串保存在数组 array 里
```
   i=0
   n=5
   while [ "$i" -lt $n ] ; do
     echo "Please input strings ... `expr $i + 1`"
     read array[$i]
     b=${array[$i]}
     echo "$b"
     i=`expr $i + 1`
   done
```

## 将字符串里的字母逐个放入数组，并输出到“标准输出”

```
chars='abcdefghijklmnopqrstuvwxyz'
for (( i=0; i<26; i++ )) ; do
    array[$i]=${chars:$i:1}
    echo ${array[$i]}
done
```

这里有趣的地方是 ${chars:$i:1}，表示从chars字符串的 $i 位置开始，获取 1 个字符。如果将 1 改为 3 ，就获取 3 个字符啦～ 结果是：

## AWK

查询本机的 ipaddress

```
HOST_IP=$(ifconfig  | grep 'inet addr' | grep -v 127.0.0.1 | awk '{print $2}' | awk -F: '{print $2}')
```

如果只是显示/etc/passwd的账户

```
cat /etc/passwd | awk  -F ':'  '{print $1}'  

cat /etc/passwd |awk  -F ':'  'BEGIN {print "name,shell"}  {print $1","$7} END {print "blue,/bin/nosh"}'
```

```
awk 'END {print NR}' filename.txt 效果等同于 wc -l filename.txt
```

## 父子 shell 传递变量

```
export JAVA_OPTS="-XX:MaxPermSize=1024m -Xmx24g" sbt -Dconfig.file=config/unitest.conf -XX:MaxPermSize=2048m -Xmx24g -Djdk.logging.allowStackWalkSearch=true clean assembly
```

## 打印到文件同时显示到界面上

```
ls | tee file

ls | tee file1 file2 file3
```

## netstat 测试

```
meterapi.sh

while [ true ]; do

        HANG_CLIENT=$(netstat -alnp 2>/dev/null | grep 8444 | awk '$2 != 0' | wc -l)
        TOTAL_CLIENT=$(netstat -alnp 2>/dev/null | grep 8444  | wc -l)
        HANG_BY_ICCS=$(netstat -alnp 2>/dev/null | grep 16790 | awk '$2 != 0' | wc -l)
        HANG_BY_ICAMAPI=$(netstat -alnp 2>/dev/null | grep 16790 | awk '$3 != 0' | wc -l)

        echo "total clients: $TOTAL_CLIENT, $HANG_CLIENT hang, $HANG_BY_ICAMAPI to iccs hang due to icamapi, $HANG_BY_ICCS to iccs hang due to iccs"
        sleep 5
done
```

netstat -anlp 输出各列的意义

Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name

Proto 一般有三种，分别是 tcp, udp, unix 其中 Unix 应该表示 unix 里的 socket

## curl

```
curl -X POST --data @json_out.txt http://localhost:8080/
```

## 分隔符 IFS

IFS stands for "internal field separator". It is used by the shell to determine how to do word splitting, i. e. how to recognize word boundaries.

```
IFS=": "

mystring="foo:bar baz rab"

for word in $mystring; do
  echo "Word: $word"
done
```

如果不设置 IFS 那么默认的分隔符是空格，设置了 IFS 就可以增加或者修改分隔符

The first character of IFS is special: It is used to delimit words in the output when using the
special `$*` variable (example taken from the Advanced Bash Scripting Guide, where you can also find more 
information on special variables like that one):

```
$ bash -c 'set w x y z; IFS=":-;"; echo "$*"'
w:x:y:z
```

```
$ bash -c 'set w x y z; IFS="-:;"; echo "$*"'
w-x-y-z
```

Note that in both examples, the shell will still treat all of the characters :, - and ; as word boundaries. The only thing that changes is the behaviour of $*.

## xargs

But what about the general case. What you really want is to convert stdout of one command to command line args of another. As others have said, xargs is the canonical helper tool in this case, reading its command line args for a command from its stdin, and constructing commands to run.

```
ls | xargs echo
```

You could also convert this some, using the substitution command $()

```
echo $(ls)
```

Would also do what you want.

## getopt

```
while getopts ":c" opt
do
    case $opt in
        c) clean='true';;
    esac
done
```

## stip

already decribed before

```
# left strip
% right strip

$ x=xyz.2.3.4.fc15.i686
$ y=${x#*fc}
$ z=${y%.*}
$ echo $z
15

[jaypal:~/Temp] awk -F. '{print substr ($5,3,2)}' <<< x=xyz.2.3.4.fc15.i686
15
```

## 读取文件的方式

```
IFS="  "   
for LINE in `cat /etc/passwd`  
do   
        echo $LINE 
done
```
读完后记得把 IFS 重置

```
#! /bin/bash  
  
cat /etc/passwd | while read LINE  
do
        echo $LINE 
done
```

IFS 会影响 for loop, 不要轻易用 IFS 


## seq

```
max=10

for i in `seq 2 $max`
do
    echo "$i"
done
```



