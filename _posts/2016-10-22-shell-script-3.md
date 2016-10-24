### Facts

1. 对于数组遍历， 必须使用 `${arr[*]}`, 如果不用的话， 不能实现遍历
2. for loop, you can use `seq 2..3` or for ele in {1..2}, 要记得， 第二种写法只能在 for loop 中使用，
   在别的地方不能保证可用性。并且， {} 内部不能含有变量， 所以 seq 更好些
3. while loop 的判断条件， while [ $num -lt 3 ];do, for 的另外一种写法， for (( i=2; i < $max; i++ )); do
4. scp 的用法，我都有些忘记了
### awk if 要写在 {} 内部，如果没有 if 的话， 可以写在外面，比如 NR == 10,

```bash
awk '$1=="J.Lulu" {print match($1，"u")}'
awk '{if (NR>0 && $4~/Brown/) print $0}' temp  至少存在一条记录且包含Brown
awk '/^tcp/ {++S[$NF]}; END {for(a in S) { print a, S[a]}}'
```
^tcp 需要用两个 / 符号保护起来，就和 sed 一样

awk 中的 for 循环，只要一对小括号

```bash
awk '{for(i=1;i<=NF;i++) a[$i]++} END {for(k in a) print k,a[k]}' words.txt | sort -k2 -nr
```

这里的逗号，表现出来是空格

另外一个 for 循环

```bash
for host in bgl-fb{01,02,03,04,05,06,07,08,09,10}
do
    ssh -q vdeadmin@$host '  //这个单引号和 host 之间有空格
        echo "$hostname" // 会发生替换
        echo `java -version`
    '
done
```


进程和线程的区别
