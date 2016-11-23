
## Memory model in golang

1: Within a single goroutine, the happens-before order is the order expressed by the program.

2: If a package p imports package q, the completion of q's init functions happens before the start 
of any of p's.

3: The go statement that starts a new goroutine happens before the goroutine's execution begins.

4: A single call of f() from once.Do(f) happens (returns) before any call of once.Do(f) returns.

解决同步问题的方案就一种，use explicit synchronization


## More About Golang

### Mutex in go is not reentrant

which means you cannot do this

```go

```

### channel

non-buffer chan could block you program's execution

```go
func main() {
    c := make(chan int)
    c <- 1
    fmt.Println(<-c)

}
```

If the channel is unbuffered, the sender blocks until the receiver has received the value. 
If the channel has a buffer, the sender blocks only until the value has been copied to 
the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

`c <- 1` blocks because the channel is unbuffered. As there's no other goroutine to 
receive the value, the situation can't resolve, this is a deadlock.

You can make it not blocking by changing the channel creation to `c := make(chan int, 1)`
so that there's room for one item in the channel before it blocks

**select**

select wait until one of his command can be executed

```go
func fibonacci(c, quit chan int) {
    x, y := 0, 1
    for {
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
            time.Sleep(time.Second * 2)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```

### closure

```go
for idx := range nums {
     go func(index int) {
     }(idx)
}
```

don't use idx inside go func, it could lead to unexpected dirty data

### package name should inaccord with directory name

今天遇到一个错误，就是 package name 是大写的 Chord, 而 import chord, 一个大小写的问题，
就导致找不到 file，报错的信息是 unresolved files

intellij 报错，但是在命令行跑却是正常的，真是奇怪

### testing

### fmt println

```go
%v   -> {25 xinszhou}
%+v -> {Age:25 Name:xinszhou}
%#v  main.Person{Age:25, Name:"xinszhou"}
```

### defer

磁盘 IO 适合使用 defer

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

**1. A deferred function's arguments are evaluated when the defer statement is evaluated.**

```go
func a() {
    i := 0
    defer fmt.Println(i) // print 0
    i++
    return
}
```

**2. Deferred function calls are executed in Last In First Out order after the surrounding function returns.**

```go
func c() (i int) {
    defer func() { i++ }()
    return 1
}
```

返回 2, 为什么呢???

A defer statement defers the execution of a function until the surrounding function returns.

The deferred call's arguments are evaluated immediately, but the function call is not 
executed until the surrounding function returns.

```go
func main() {
    defer fmt.Println("world")

    fmt.Println("hello")
} // hello world
```

Deferred function calls are pushed onto a stack. When a function returns, its deferred calls are executed in last-in-first-out order.

```go
func main() {
    fmt.Println("counting")
    for i := 0; i < 10; i++ {
        defer fmt.Println(i)
    } // 10, 9, 8, 7...
    fmt.Println("done")
}
```

### inheritance

```go
type Car struct {
    wheelCount int
}

// define a behavior for Car
func (car Car) numberOfWheels() int {
    return car.wheelCount
}

type Ferrari struct {
    Car //anonymous field Car
}

func main() {
    f := Ferrari{Car{4}}
    fmt.Println("A Ferrari has this many wheels: ", f.numberOfWheels()) //no method defined for Ferrari, but we have the same behavior as Car.
}
```

所以只要把继承的类型作为第一个成员变量并是匿名的即可

此外, golang 不支持 overload

### 反射

反射的常用方法有 reflect.typeOf(interface{}), reflect.valueOf(), Field, Function, 对于 Function 来讲,
还有 参数,返回值


```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type()) // float64
fmt.Println("kind is float64:", v.Kind() == reflect.Float64) // true
fmt.Println("value:", v.Float()) // 3.4
```

```go
    type T struct {
        A int
        B string
    }

    t := T{23, "skidoo"}

    s := reflect.ValueOf(&t).Elem()

    typeOfT := s.Type()

    for i := 0; i < s.NumField(); i++ {
        f := s.Field(i)
        fmt.Printf("%d: %s %s = %v\n", i, typeOfT.Field(i).Name, f.Type(), f.Interface())
    }
```

动态修改值

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

T 的字段名要大写（可导出），因为只有可导出的字段是可设置的

由于 s 包含可设置的反射对象，所以可以修改结构体的字段