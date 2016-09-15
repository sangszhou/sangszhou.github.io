## Future

Java8 已经具备了很多异步和流的理念

```java
ExecutorService executor = Executors.newWorkStealingPool();

List<Callable<String>> callables = Arrays.asList(
        () -> "task1",
        () -> "task2",
        () -> "task3");

executor.invokeAll(callables)
    .stream()
    .map(future -> {
        try {
            return future.get();
        }
        catch (Exception e) {
            throw new IllegalStateException(e);
        }
    })
    .forEach(System.out::println);
    
// <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> arg)   
```

```scala
implicit val dispatcher = ...
implicit val timeout = ...

List(Future("task1"), Future("task2"), Future("task3")
    .map(_.get)
    .foreach(println)
```

分析上面两段代码, 首先 scala Future 里的是 string, 而 java arrays 放的是 Funtion, 这有所区别,
但是 java arrays 放 function 的原因是 invokeAll 只接受 function, 对于 Future, 它也接受 function (代码块),
但同时接受一个变量, 其内部实现是直接返回, 不用到 threadpool 里面再跑一遍了

其次, scala future 默认使用的是 forkJoinPool

最后, scala future 很少使用 .get 方法, 因为没有使用的必要。而 java 里却很需要, 那是因为 java 的函数很少会以
future 类型为参数, 如果你要被别的函数使用, 就必须用 future 容器内的元素, 所以必须得调用 .get. 即便如此, future
还是有用的, 当 "task1", "task2", "task3" 都比较耗时时, 使用上述 java 代码的时间只是三个任务的最大值, 而不是和

从上面看的出, scala 对 future 的支持是原生的, 而 java 则要考虑与已有数据结构, 函数相适应的问题

关于 scala future 的实现, 已在另一篇 blog 中写了


## ForkJoinPool 的用法