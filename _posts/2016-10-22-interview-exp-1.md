---
layout: post
title: interview exp summary 1
categories: [exp]
description: exp
keywords: exp summary
---

### 反射的实现原理

Class.forName(classname)方法,实际上是调用了Class类中的 Class.forName(classname, true, currentLoader)方法。
参数：name - 所需类的完全限定名；initialize - 是否必须初始化类；loader - 用于加载类的类加载器。currentLoader
则是通过调用ClassLoader.getCallerClassLoader()获取当前类加载器的。类要想使用，必须用类加载器加载，所以需要加载器。
反射机制，不是每次都去重新反射，而是提供了cache，每次都会需要类加载器去自己的cache中查找，如果可以查到，则直接返回该类。

![](/images/posts/javavm/class_loader.png)

这幅图简单的说明了类加载器的类加载过程。先检查自己是否已经加载过该类，如果加载过，则直接返回该类，
若没有则调用父类的loadClass方法，如果父类中没有，则执行findClass方法去尝试加载此类，也就是我们通常所理解的片面的"反射"了。
这个过程主要通过ClassLoader.defineClass方法来完成。defineClass 方法将一个字节数组转换为 Class 类的实例（任何类
的对象都是Class类的对象）。这种新定义的类的实例需要使用 Class.newInstance 来创建，而不能使用new来实例化。

其实说的简单通俗一点，就是在运行期间，如果我们要产生某个类的对象，Java虚拟机（JVM）会检查该类型的
Class对象是否已被加载。如果没有被加载，JVM会根据类的名称找到.class文件并加载它。一旦某个类型的Class对象
已被加载到内存，就可以用它来产生该类型的所有对象。


反射实现 IOC

```java
interface fruit{
    public abstract void eat();
}

class Apple implements fruit{
    public void eat(){
        System.out.println("Apple");
    }
}

class Orange implements fruit{
    public void eat(){
        System.out.println("Orange");
    }
}
//操作属性文件类
class init {
    public static Properties getPro() throws FileNotFoundException, IOException{
        Properties pro=new Properties();
        File f=new File("fruit.properties");
        if(f.exists()){
            pro.load(new FileInputStream(f));
        }else{
            pro.setProperty("apple", "Reflect.Apple");
            pro.setProperty("orange", "Reflect.Orange");
            pro.store(new FileOutputStream(f), "FRUIT CLASS");
        }
        return pro;
    }
}

class Factory{
    public static fruit getInstance(String ClassName){
        fruit f=null;
        try{
            f=(fruit)Class.forName(ClassName).newInstance();
        }catch (Exception e) {
            e.printStackTrace();
        }
        return f;
    }
}

class hello{
    public static void main(String[] a) throws FileNotFoundException, IOException{
        Properties pro= init.getPro();
        fruit f= Factory.getInstance(pro.getProperty("apple"));
        if(f! =null){
            f.eat();
        }
    }
}
//【运行结果】: Apple
```

### 能不能画出 hadoop, kafka, spark 的执行图, 还有 spring 的

### Search Insert Position

```java
    public int searchInsert(int[] nums, int target) {
        int start = 0, end = nums.length - 1, mid = 0;
        while(start <= end){//the key is  "=" !!
            mid = start + (end - start) / 2;
            if(nums[mid] == target) return mid;
            if(target < nums[mid]) end = mid - 1;
            if(target > nums[mid]) start = mid + 1;
        }
        return start;
    }
```

这样就不需要特殊处理了, 原因在于 start, end 的判断条件, 当 start == end 时, 究竟该怎么办

不花时间看了, 返回的是 start, 不是 high

### 类的初始化顺序

[link](http://t240178168.iteye.com/blog/1637647)

```java
public class InitialOrderTest {   
  
    // 静态变量   
    public static String staticField = "静态变量";   
    // 变量   
    public String field = "变量";   
  
    // 静态初始化块   
    static {   
        System.out.println(staticField);   
        System.out.println("静态初始化块");   
    }   
  
    // 初始化块   
    {   
        System.out.println(field);   
        System.out.println("初始化块");   
    }   
  
    // 构造器   
    public InitialOrderTest() {   
        System.out.println("构造器");   
    }   
  
    public static void main(String[] args) {   
        new InitialOrderTest();   
    }   
}  

// result
静态变量 
静态初始化块 
变量 
初始化块 
构造器 
```

如果是包含继承关系的

```
父类--静态变量 
父类--静态初始化块 
子类--静态变量 
子类--静态初始化块 
父类--变量 
父类--初始化块 
父类--构造器 
子类--变量 
子类--初始化块 
子类--构造器 
```

现在，结果已经不言自明了。大家可能会注意到一点，那就是，并不是父类完全初始化完毕后才进行子类的初始化，实际上子类的静态变量和静态初始化块的初始化是在父类的变量、初始化块和构造器初始化之前就完成了。 

那么对于静态变量和静态初始化块之间、变量和初始化块之间的先后顺序又是怎样呢？是否静态变量总是先于静态初始化块，变量总是先于初始化块就被初始化了呢？实际上这取决于它们在类中出现的先后顺序。我们以静态变量和静态初始化块为例来进行说明。 

### 用一个数组实现两个堆, 目标是占满整个堆空间

可以把隔板设置在数组的两边, 也可以设置在中间, 设置在中间时就是麻烦些

**一个数组实现三个栈**

第一个栈使用空间 1, 4, 7, 第二个栈使用 2, 5, 8, 这种方式空间不能得到有效的利用

第二种办法是, 第一个栈头在左边, 第二个栈头在右边, 第三个栈头在中间。在中间的那个栈分别往左右插入元素。

### 全排列

交换位置的解法

```java
    public static void permutate(char[] data, int begin){
        int length = data.length;
        if(begin == length)
            System.out.println(data);
        for(int i = begin ; i < length; i++)
        {
            if(isUnique(data, begin, i)){
                swap(data, begin, i);
                permutate(data, begin + 1);
                swap(data, begin, i);
            }                
        }
    }
```

DFS 可能根本不需要考虑吧, 即使这么久没写了

### 二叉树问题

前序 中序 后序

```java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode(int x) { val = x; }
}
```

