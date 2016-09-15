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

从本例可以看出：成员内部类，就是作为外部类的成员，可以直接使用外部类的所有成员和方法，即使是private的。
虽然成员内部类可以无条件地访问外部类的成员，而外部类想访问成员内部类的成员却不是这么随心所欲了。
在外部类中如果要访问成员内部类的成员，必须先创建一个成员内部类的对象，再通过指向这个对象的引用来访问：

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





