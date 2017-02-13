---
layout: post
title: Shell script 2
categories: [shell]
keywords: linux, shell, script
---

## 僵尸进程和孤儿进程

在类UNIX系统中，僵尸进程是指完成执行（通过exit系统调用，或运行时发生致命错误或收到终止信号所致）但在操作系统的进程表中仍然有一个表项（进程控制块PCB），处于"终止状态"的进程。这发生于子进程需要保留表项以允许其父进程读取子进程的exit status：一旦退出态通过wait系统调用读取，僵尸进程条目就从进程表中删除，称之为"回收（reaped）"。正常情况下，进程直接被其父进程wait并由系统回收。进程长时间保持僵尸状态一般是错误的并导致资源泄漏。

与正常进程不同，kill命令对僵尸进程无效。孤儿进程不同于僵尸进程，其父进程已经死掉，但孤儿进程仍能正常执行，但并不会变为僵尸进程，因为被init（进程ID号为1）收养并wait其退出。

子进程死后，系统会发送SIGCHLD 信号给父进程，父进程对其默认处理是忽略。如果想响应这个消息，父进程通常在SIGCHLD 信号事件处理程序中，使用wait系统调用来响应子进程的终止。

收割僵尸进程的方法是通过kill命令手工向其父进程发送SIGCHLD信号。如果其父进程仍然拒绝收割僵尸进程，则终止父进程，使得init进程收养僵尸进程。init进程周期执行wait系统调用收割其收养的所有僵尸进程。


## What is the use of a shebang line

Shebang line at top of each script determines the location of the engine which is to be used in 
order to execute the script.

## What are the four fundamental components of every file system on linux

bootblock, super block, inode block and  datablock

**Super block** contains all the information about the file system like size of file system, 
block size used by it,number of free data blocks and list of free inodes and data blocks.

**boot block** contains a small program called “Master Boot record”(MBR) which loads the kernel  during system boot up.

**inode block** contains the **inode** for every file of the file system along with all the file attributes except its name.

## How will you print the login names of all users on a system ?

/etc/shadow file has all the users listed.

```
awk –F ‘:’ ‘{print $1} /etc/shadow’|uniq -u
```

## Given a file find the count of lines containing word “ABC”

```
grep –c  “ABC” file1
```

## How will you emulate wc –l using awk?

NR 是行数, NF 是分隔符

```
awk ‘END {print NR} fileName’
```

## 网络命令都有哪些

**telnet**
Telnet is an application layer protocol used on the Internet or local area networks to provide a bidirectional 
interactive text-oriented communication facility using a virtual terminal connection

```
nc -l port
nc -z host port
nc -c
```

**nslookup**

query Internet name servers interactively

```bash
➜  ~ nslookup google.com
Server:		64.104.123.144
Address:	64.104.123.144#53

Non-authoritative answer:
Name:	google.com
Address: 74.125.200.113
Name:	google.com
```


**netstat **

show network status

``` bash
netstat -nalt | grep tcp

Active kernel control sockets
Proto Recv-Q Send-Q   unit     id name
```

Q 应该是 queue 的意思, 表示待处理消息

## How will you copy file from one machine to other

We can use utilities like “ftp” ,”scp” or “rsync” to copy file from one machine to other.

**scp**-- secure copy (remote file copy program)



```bash
rsync -t *.c foo:src/

rsync -avz foo:src/bar /data/tmp # a means archive, 
```

## What are zombie processes  @todo

These are the processes which have died but whose exit status is still not picked by the parent process. 
These processes even if not functional still have its process id entry in the process table.

Linux 中的僵尸进程

@todel
## What is the difference between $$ and $!?
   
$$ gives the process id of the currently executing process whereas $! shows the
process id of the process that recently went into background.

## 递归创建文件夹

```
mkdir -p
```

## 从“标准输入”读入n次字符串，每次输入的字符串保存在数组 array 里

```bash
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

获取数组长度, 记住外面的是 {} 而不是 ()

```bash
echo ${#a[@]}

echo ${array[index]} // get by index

#copy
ary=${str[*]}

str="ABC"
echo ${#str[@]} // length
echo ${str[*]} // all elements
```

**遍历**

```bash
for data in ${array[@]}  
do  
    echo ${data}  
done  
```

```bash
chars='abcdefghijklmnopqrstuvwxyz'

for (( i=0; i<26; i++ )) ; do // 记住这个 for 循环的写法
    array[$i]=${chars:$i:1}
    echo ${array[$i]}
done
```

这里有趣的地方是 ${chars:$i:1}，表示从chars字符串的 $i 位置开始，获取 1 个字符。如果将 1 改为 3 ，就获取 3 个字符

## AWK

查询本机的 ipaddress

```
HOST_IP=$(ifconfig  | grep 'inet addr' | grep -v 127.0.0.1 | awk '{print $2}' | awk -F: '{print $2}')
```

如果只是显示/etc/passwd的账户

```bash
cat /etc/passwd | awk  -F ':'  '{print $1}'  

cat /etc/passwd |awk  -F ':'  'BEGIN {print "name,shell"}  {print $1","$7} END {print "blue,/bin/nosh"}'
```

awk 也就是三步而已, 开始 BEGIN, 然后是中间的操作, 然后是 END, 如果开始符号没有的话, 就直接 {}, 就像第一行代码展示的一样

file name 要写在外面

```
awk 'END {print NR}' filename.txt 效果等同于 wc -l filename.txt
```

## 父子 shell 传递变量

```
export JAVA_OPTS="-XX:MaxPermSize=1024m -Xmx24g" sbt -Dconfig.file=config/unitest.conf 
    -XX:MaxPermSize=2048m -Xmx24g -Djdk.logging.allowStackWalkSearch=true clean assembly
```

## 打印到文件同时显示到界面上

```bash
ls | tee file

ls | tee file1 file2 file3
```

## netstat 测试

grep port 的时候不需要指定 LocalAddress 或 RemoteAddress 么

```bash
meterapi.sh

while [ true ]; do

        HANG_CLIENT=$(netstat -alnp 2>/dev/null | grep 8444 | awk '$2 != 0' | wc -l)
        TOTAL_CLIENT=$(netstat -alnp 2>/dev/null | grep 8444  | wc -l)
        HANG_BY_ICCS=$(netstat -alnp 2>/dev/null | grep 16790 | awk '$2 != 0' | wc -l)
        HANG_BY_ICAMAPI=$(netstat -alnp 2>/dev/null | grep 16790 | awk '$3 != 0' | wc -l)

        echo "total clients: $TOTAL_CLIENT, $HANG_CLIENT hang, $HANG_BY_ICAMAPI to iccs hang due to icamapi, 
            $HANG_BY_ICCS to iccs hang due to iccs"
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

```bash
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

标准输出转化成参数

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

```shell
#/bin/bash
echo $0
echo $*
while getopts ":a:bc" opt
do
        case $opt in
                a )
                        echo $OPTARG                    
                        echo $OPTIND;;
                b )
                        echo "b $OPTIND";;
                c )
                        echo "c $OPTIND";;
                ? )
                        echo "error"                    
                        exit 1;;
                esac
done
echo $OPTIND
echo $*
shift $(($OPTIND - 1))
echo $*
echo $0


运行sh getopt.sh  -a 12 -b -c 34 -m
输出：
getopt.sh
-a 12 -b -c 34
12
3
b 4
c 5
5
-a 12 -b -c 34
34
getopt.sh
```

选项之间可以通过冒号:进行分隔，也可以直接相连接，：表示选项后面必须带有参数，如果没有可以不加实际值进行传递

例如：getopts ahfvc: option表明选项a、h、f、v可以不加实际值进行传递，而选项c必须取值。使用选项取值时，必须使用变量OPTARG保存该值。

注意这个脚本里的:afghc:  的冒号  前面的冒号表示屏蔽脚本的系统提示错误，转而使用自己提供的错误提示方式。后面的冒号表示c这个选项为必选项，并且需要指定具体的参数值，同时需要获取到参数值后将参数值存放到变量OPTARG中，供以后读取用。


1. OPTARG 存储相应选项的参数 OPTIND指向的是下一个参数的index
2. shift 会改变参数的顺序，通过左移去掉某些参数
3. getopts 检测到非法参数就会停止，比如上例中遇到34就会终止
4. unset OPTIND  可以解决shell脚本的函数中使用getopts

使用 shift 而不是 getopts 这种可能更灵活吧

```shell
#!/bin/bash  
usage()  
{  
  echo "usage:`basename $0` -[l|u] file [files]" >&2  
  exit 1   
}     
if [ $# -eq 1 ]; then  
    usage  
fi  
opt=""  
while [ $# -ne 1 ]   
do  
   opt=$1;  
   case $opt in   
      -l|-L) echo "-l or -L options is specified"  
             shift  
             ;;  
      -u|-U) echo "-u or -U options is specified"  
              shift  
             ;;  
      *)  usage  
             ;;  
   esac  
done  
```

## strip

already described before

```
# left strip
% right strip

$ x=xyz.2.3.4.fc15.i686
$ y=${x#*fc} // 15.i686
$ z=${y%.*} // 15
$ echo $z
15

awk -F. '{print substr ($5,3,2)}' <<< x=xyz.2.3.4.fc15.i686
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

上面的读法有问题, 有时候最后一行会读不到

```bash
while read line || [[ -n $line ]]
do
    do something
done < file.txt
```


## seq

```
max=10

for i in `seq 2 $max`
do
    echo "$i"
done
```

### 要求分析apache访问日志，找出访问页面数量在前100位的IP数。日志大小在78M左右。以下是apache的访问日志节选
         
```
202.101.129.218 - - [26/Mar/2006:23:59:55 +0800] "GET /online/stat_inst.php?pid=d065 HTTP/1.1" 302 20-"-" "-" "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1)"
```
    
```bash
    # awk '{print $1}' log      |sort |uniq -c|sort -r |head -n10
          5 221.224.78.15
          3 221.233.19.137
          1 58.63.148.135
          1 222.90.66.142
          1 222.218.90.239
          1 222.182.95.155
          1 221.7.249.206
          1 221.237.232.191
          1 221.235.61.109
          1 219.129.183.122
```

### 求2个数之和
    
```
#/bin/bash
typeset first second
read -p "Input the first number:" first
read -p "Input the second number:" second
result=$[$first+$second] // 不是 {}, 是 []
echo "result is : $result"
exit 0
```

```bash
${{ 1 + 2 }}
```



