---
layout: post
title: java inner class and more basic concept
categories: [java]
keywords: java, inner class
---

**todo:**

1. 补充内部类的字节码

成员内部类、局部内部类、静态嵌套类、匿名内部类

### 成员内部类

> 第一：成员内部类中不能存在任何static的变量和方法；

> 第二：成员内部类是依附于外围类的，所以只有先创建了外围类才能够创建内部类

```java
class Outter {
    private int age = 12;
      
    class Inner {
        private int age = 13;
        public void print() {
            int age = 14;
            System.out.println("局部变量：" + age);
            System.out.println("内部类变量：" + this.age);
            System.out.println("外部类变量：" + Outter.this.age);
        }
    }
}
  
public class test1 {
    public static void main(String[] args) {
        Outter out = new Outter();
        Outter.Inner in = out.new Inner(); // 注意创建的方式
        in.print();
    }
}
```

从本例可以看出：成员内部类，就是作为外部类的成员，可以直接使用外部类的所有成员和方法，即使是private的
虽然成员内部类可以无条件地访问外部类的成员，而外部类想访问成员内部类的成员却不是这么随心所欲了
在外部类中如果要访问成员内部类的成员，必须先创建一个成员内部类的对象，再通过指向这个对象的引用来访问:

内部类可以拥有private访问权限、protected访问权限、public访问权限及包访问权限。

比如上面的例子，如果成员内部类Inner用private修饰，则只能在外部类的内部访问，如果用public修饰，则任何地方都能访问；如果用protected修饰，则只能在同一个包下或者继承外部类的情况下访问；如果是默认访问权限，则只能在同一个包下访问。

我个人是这么理解的，由于成员内部类看起来像是外部类的一个成员，所以可以像类的成员一样拥有多种权限修饰。

要注意的是，成员内部类不能含有static的变量和方法。因为成员内部类需要先创建了外部类，才能创建它自己的

### 局部内部类

局部内部类是定义在一个方法或者一个作用域里面的类，它和成员内部类的区别在于局部内部类的访问仅限于方法内或者该作用域内。

```java
public class Parcel5 {
    public Destionation destionation(String str){
        class PDestionation implements Destionation{
            private String label;
            private PDestionation(String whereTo){
                label = whereTo;
            }
            public String readLabel(){
                return label;
            }
        }
        return new PDestionation(str);
    }
    
    public static void main(String[] args) {
        Parcel5 parcel5 = new Parcel5();
        Destionation d = parcel5.destionation("chenssy");
    }
}
```

换句话说，在方法中定义的内部类只能访问方法中final类型的局部变量，这是因为在方法中定义的局部变量相当于一个常量，
它的生命周期超出方法运行的生命周期，由于局部变量被设置为final，所以不能再内部类中改变局部变量的值。
（这里看到网上有不同的解释，还没有彻底搞清楚==）

但这个例子里, 没有 final 不也正常么

### 静态嵌套类

1. 它的创建是不需要依赖于外围类的。

2. 它不能使用任何外围类的非static成员变量和方法。

又叫静态局部类、嵌套内部类，就是修饰为static的内部类。声明为static的内部类，不需要内部类对象和外部类对象之间的联系，就是说我们可以直接引用outer.inner，即不需要创建外部类，也不需要创建内部类。

```java
class Outter {
    private static int age = 12;
    
    static class Inner {
        public void print() {
            System.out.println(age);
        }
    }
}
  
public class test1 {
    public static void main(String[] args) {
        Outter.Inner in = new Outter.Inner();
        in.print();
    }
}
```

### 匿名内部类

1. 匿名内部类是没有访问修饰符的。

2. new 匿名内部类，这个类首先是要存在的。如果我们将那个InnerClass接口注释掉，就会出现编译出错。

3. 注意getInnerClass()方法的形参，第一个形参是用final修饰的，而第二个却没有。同时我们也发现第二个形参在匿名内部类中没有使用过，所以当所在方法的形参需要被匿名内部类使用，那么这个形参就必须为final。

4. 匿名内部类是没有构造方法的。因为它连名字都没有何来构造方法。

匿名内部类应该是平时我们编写代码时用得最多的，在编写事件监听的代码时使用匿名内部类不但方便，而且使代
码更加容易维护。下面这段代码是一段Android事件监听代码：

```java
scan_bt.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                // TODO Auto-generated method stub
            }
        });
          
        history_bt.setOnClickListener(new OnClickListener() {      
            @Override
            public void onClick(View v) {
                // TODO Auto-generated method stub
            }
        });
```

这种写法虽然能达到一样的效果，但是既冗长又难以维护，所以一般使用匿名内部类的方法来编写事件监听代码。
同样的，匿名内部类也是不能有访问修饰符和static修饰符的。

匿名内部类是唯一一种没有构造器的类。正因为其没有构造器，所以匿名内部类的使用范围非常有限，
大部分匿名内部类用于接口回调。匿名内部类在编译的时候由系统自动起名为Outter$1.class。一般来说，
匿名内部类用于继承其他类或是实现接口，并不需要增加额外的方法，只是对继承方法的实现或是重写。


## Exception

![](/images/posts/java/exception_arch.png)

图 1 描述了异常的结构，其实我们都知道异常分检测异常和非检测异常，但是在实际中又混淆了这两种异常的应用。由于非检测异常使用方便，很多开发人员就认为检测异常没什么用处。其实异常的应用情景可以概括为以下：

1. 调用代码不能继续执行，需要立即终止。出现这种情况的可能性太多太多，例如服务器连接不上、参数不正确等。这些时候都适用非检测异常，不需要调用代码的显式捕捉和处理，而且代码简洁明了。

2. 调用代码需要进一步处理和恢复。假如将 SQLException 定义为非检测异常，这样操作数据时开发人员理所当然的
认为 SQLException 不需要调用代码的显式捕捉和处理，进而会导致严重的 Connection 不关闭、Transaction 不回
滚、DB 中出现脏数据等情况，正因为 SQLException 定义为检测异常，才会驱使开发人员去显式捕捉，并且在代
码产生异常后清理资源。当然清理资源后，可以继续抛出非检测异常，阻止程序的执行。根据观察和理解，检测异常
大多可以应用于工具类中

@todo ?都知道哪些异常 (在 Effective Java 中有提到)

NullPointerException, RuntimeException, InterruptedException, OOM Exception, ArtimeException

ConcurrentModificationException

**Spring 专属的异常**



### 误区五、将异常包含在循环语句块中

我们都知道异常处理占用系统资源。一看，大家都认为不会犯这样的错误。换个角度，类 A 中执行了一段循环，循环中调用了 B 类的
方法，B 类中被调用的方法却又包含 try-catch 这样的语句块。

## IO

Java从1.4版本引入NIO，提升了I/O性能。Java的I/O操作类大概分为如下四组：

* 基于字节操作的I/O接口：InputStream和OutputStream
* 基于字符操作的I/O接口：Writer和Reader
* 基于磁盘操作的I/O接口：File
* 基于网络操作的I/O接口：Socket

前两组是传输数据的数据格式，后两组主要是传输数据的方式

### 基于字节的I/O操作接口

基于字节流的I/O操作接口输入和输出分别是InputStream和OutputStream，两者都被划分成若干子类，每个子类处理不同的操作类型。

1、操作数据的方式是可以组合使用的

```java
OutputStream out = new BufferedOutputStream(new ObjectOutputStream("fileName"));
```

2、流最终写到的地址必须要指定，要么写到磁盘，要么写到网络

基于字符的I/O操作接口

```java
write(char chuff[], int off, int len);
int read(char chuff[], int off, int len);
```

### 字节与字符的转化接口

InputStreamReader类是字节到字符的转化桥梁，转化过程中需要指定编码字符集，否则采用默认字符集。OutputStreamWriter类似

```java
try{
  StringBuffer str = new StringBuffer();
  char[] buf = new char[1024];
  FileReader f = new FileReader("file");
  while(f.read(buf)>0){
    str.append(buf);
  }
  str.toString();
}
//注意FileReader继承了InputStreamReader类
```

### 磁盘I/O工作机制

**标准访问文件方式**

read() -> 内核高速缓存 -> 没有则读取磁盘，然后缓存在系统中

write() -> 内核高速缓存 -> 对用户来说写操作已经完成，至于什么时候写到磁盘由操作系统决定，除非显式调用sync

**Java访问磁盘文件**

![](/images/posts/java/read-disk.png)

**Java序列化技术**

Java序列化就是将一个对象转化成一串二进制表示的字节数组，通过保存或转移这些字节数据来达到持久化的目的。需要持久化，对象必须继承java.io.Serializable接口。

* 当父类继承Serializable接口，所有子类都可以被序列化
* 子类实现了Serializable接口，父类没有，父类中的属性不能被序列化（不报错，数据会丢失），但是子类中的属性仍能正确序列化
* 如果序列化的属性是对象，这个对象也必须可序列化，否则会报错
* 在反序列化时，如果对象的属性有修改或删减，修改的部分会丢失，但不会报错
* 如果serialVersionUID被修改，反序列化会失败

### 网络I/O工作机制

![](/images/posts/java/tcp-state-machine.png)

CLOSED：起始点，在超时或者连接关闭时进入此状态

LISTEN：Server端在等待连接时的状态，Server端为此要调用Socket、bind、listen函数，就能进入此状态，等待客户端连接

SYN-SENT：客户端发起连接，发送SYN给服务器端。如果服务器端不能连接，则直接进入CLOSED状态。

SYN-RCVD：与3对应，服务器端接受客户端SYN请求，服务器由LISTEN状态进入SYN-RCVD状态，同时回应一个ACK，发送SYN给客户端。另外一种情况是，客户端在发起SYN的同时接收到服务器端的SYN请求，客户端会由SYN-SENT到SYN-RCVD状态

ESTABLISHED：服务器和客户端在完成三次握手后进入状态，说明已经可以开始传输数据了

FIN-WAIT-1：主动关闭的一方，由状态5进入此状态，具体动作是发送FIN给对方

FIN-WAIT-2：主动关闭的一方，接收到对方的FIN ACK，进入此状态。由此不能再接收到对方的数据，但是能够向对方发送数据

CLOSE-WAIT：接收到FIN以后，被动关闭的一方进入此状态。具体动作是接收到FIN同时发送ACK

LAST-ACK：被动关闭的一方，发起关闭请求，由状态8进入此状态。具体动作是发送FIN给对方，同时在接收到ACK时进入CLOSED状态。

CLOSING：两边同时发起关闭请求，会由FIN-WAIT1进入此状态。具体动作是接收到FIN请求，同时响应一个ACK

TIME-WAIT：有三个状态可以转换为此状态

由FIN-WAIT-2转换到TIME-WAIT，在双方不同时发起FIN情况下，主动关闭一方在接收到FIN后进入的状态

由CLOSING转换到TIME-WAIT，双头同时发起关闭，同时接收FIN并做了ACK，由CLOSING进入TIME-WAIT

由FIN-WAIT-1转换到ITME-WAIT，同时接收到FIN（对方发起）和ACK（本身发起的FIN回应）

```java
public void selector()throws IOException{
  ByteBuffer buffer = ByteBuffer.allocate(1024);
  Selector selector = Selector.open();
  ServerSocketChannel ssc = ServerSocketChannel.open();
  ssc.configureBlocking(false); //设置为非阻塞方式
  ssc.socket().bind(new InetSocketAddress(8080));
  ssc.register(selector, SelectionKey.OP_ACCEPT); //注册监听的事件
  while(true){
    Set selectedKeys = selector.selectedKeys();//取得所有key集合
    Iterator it = selectedKeys.iterator();
    while(it.hasNext()){
      SelectionKey key = (SelectionKey)it.next();
      if((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT){
        ServerSocketChannel ssChannel = (ServerSocketChannel)key.channel();
        SocketChannel sc = ssChannel.accept(); //接收到服务端的请求
        sc.configureBlocking(false);
        sc.register(selector, SelectionKey.OP_READ);
        it.remove();
      }else if((key.readyOps()  & SelectionKey.OP_READ) == SelectionKey.OPREAD){
        SocketChannel sc = (SocketChannel)key.channel();
        while(true){
          buffer.clear();
          int n = sc.read(buffer);//读取数据
          if(n <= 0){
            break;
          }
          buffer.flip();
        }
        it.remove();
      }
    }
}
}
```

### 设计模式解析之适配器模式

Java I/O 中的是配置模式：InputStreamReader，字符和字节数据转换类，实现了Reader接口，并持有了InputStream的引用，这里是通过StreamDecoder类间接持有的，因为从byte到char要经过编码。

![](/images/posts/java/InputStreamReader.png)

### 装饰器模式结构

![](/images/posts/java/decorator.png)

* Component：抽象接口

* ConcreteComponent：实现接口的所有功能

* Decorator：装饰器角色，持有Component对象引用

* ConcreateDecorator：具体的装饰器实现者，负责实现装饰器角色定义的功能


![](/images/posts/java/FilterInputStream.png)

InputStream：抽象接口

FileInputStream实现了组件的所有接口

FilterInputStream即装饰角色，持有了InputStream对象实例的引用。

BufferedInputStream是具体的装饰器实现者，给InputStream类附加了功能。




## ThreadLocal 用法
 
先观察一段代码, Mysql Connection 问题 

```java
public class DBUtil {
    // 数据库配置
    private static final String driver = "com.mysql.jdbc.Driver";
    private static final String url = "jdbc:mysql://localhost:3306/demo";
    private static final String username = "root";
    private static final String password = "root";

    // 定义一个数据库连接
    private static Connection conn = null;

    // 获取连接
    public static Connection getConnection() {
        try {
            Class.forName(driver);
            conn = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return conn;
    }

    // 关闭连接
    public static void closeConnection() {
        try {
            if (conn != null) {
                conn.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

关闭 connection 是关闭 static Connection conn 的, 而不是把 connection 作为参数, 这是有问题的

在多线程环境下, static connection 成员变量会被共享

### 使用 ThreadLocal 解决问题

```java
public class DBUtil {
    // 数据库配置
    private static final String driver = "com.mysql.jdbc.Driver";
    private static final String url = "jdbc:mysql://localhost:3306/demo";
    private static final String username = "root";
    private static final String password = "root";

    // 定义一个用于放置数据库连接的局部线程变量（使每个线程都拥有自己的连接）
    private static ThreadLocal<Connection> connContainer = new ThreadLocal<Connection>();

    // 获取连接
    public static Connection getConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn == null) {
                Class.forName(driver);
                conn = DriverManager.getConnection(url, username, password);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connContainer.set(conn);
        }
        return conn;
    }

    // 关闭连接
    public static void closeConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn != null) {
                conn.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connContainer.remove();
        }
    }
}
```

把 Connection 放到 ThreadLocal 里, 这样就不会共享变量了

### ThreadLocal 的实现

```java
public class MyThreadLocal<T> {

    private Map<Thread, T> container = Collections.synchronizedMap(new HashMap<Thread, T>());

    public void set(T value) {
        container.put(Thread.currentThread(), value);
    }

    public T get() {
        Thread thread = Thread.currentThread();
        T value = container.get(thread);
        if (value == null && !container.containsKey(thread)) {
            value = initialValue();
            container.put(thread, value);
        }
        return value;
    }

    public void remove() {
        container.remove(Thread.currentThread());
    }

    protected T initialValue() {
        return null;
    }
}

// 使用
public class SequenceC implements Sequence {

    private static MyThreadLocal<Integer> numberContainer = new MyThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    public int getNumber() {
        numberContainer.set(numberContainer.get() + 1);
        return numberContainer.get();
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceC();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}

```

### ThreadLocal 在 Spring 中的应用

```java
public class TopicDao {
    private static ThreadLocal connThreadLocal = new ThreadLocal
    
    public static Connection getConnect
        if(connThread.get == null)
            Connection conn = ConnectionMgr.getConnection
            connThreadLocal.set(conn)
            return conn
        else 
            return connThreadLocal.get
     
     public void addTopic
        Statement stat = getConnection.createStatement
}
```

当然，这个例子本身很粗糙，将Connection的ThreadLocal直接放在DAO只能做到本DAO的多个方法
共享Connection时不发生线程安全问题，但无法和其它DAO共用同一个Connection，要做到
同一事务多DAO共享同一Connection，必须在一个共同的外部类使用ThreadLocal保存Connection。

