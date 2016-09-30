---
layout: post
title: effective java
categories: [java]
keywords: java
---

### 构建器

builder模式只在有很多参数的时候才能使用，在设计阶段如果能够预想到将来多参数的情况，那么最好在最开始使用这种模式

总之，如果类的构造器或者静态工厂中具有多个参数，设计这种类时，Builder模式就是中不错的选择。

```java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;
    //static inner builder class
    public static class Builder {
        // Required parameters
        private final int servingSize;
        private final int servings;
        // Optional parameters - initialized to default values
        private int calories      = 0;
        private int fat           = 0;
        private int carbohydrate  = 0;
        private int sodium        = 0;
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings    = servings;
        }
        public Builder calories(int val)
            { calories = val;      return this; }
        public Builder fat(int val)
            { fat = val;           return this; }
        public Builder carbohydrate(int val)
            { carbohydrate = val;  return this; }
        public Builder sodium(int val)
            { sodium = val;        return this; }
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }
    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
    //test method
    public static void main(String[] args) {
        NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).
            calories(100).sodium(35).carbohydrate(27).build();
    }
}
```

### 覆盖equals时请遵守通用约定

**不覆盖equals的情况:**

1. 类的每个实例本质上都是唯一的。
2. 不用关心类是否提供了“逻辑相等（logical equality）“的测试功能。
3. 超类已经覆盖了equals，从超类继承过来的行为对于子类也是合适的。
4. 类是私有的或包级私有的，可以确定他的equals方法永远不会被调用。

**应该覆盖equals的情况:**
 
1. 如果类具有自己特有的“逻辑想等”概念，而且超类还没有覆盖equals以实现期望的行为，这时我们就需要覆盖equals方法。这通常属于“值类”的情形。
2. 覆盖equals方法需要遵守的通用约定 
3. 自反性（reflexivity）：对于任何非空引用值 x，x.equals(x) 都应返回 true。
4. 对称性（symmetry）：对于任何非空引用值 x 和 y，当且仅当 y.equals(x) 返回 true 时，x.equals(y) 才应返回 true。
5. 传递性（transitivity）：对于任何非空引用值 x、y 和 z，如果 x.equals(y) 返回 true，并且 y.equals(z) 返回 true，那么 x.equals(z) 应返回 true。
6. 一致性（consistency）：对于任何非空引用值 x 和 y，多次调用 x.equals(y) 始终返回 true 或始终返回 false，前提是对象上 equals 比较中所用的信息没有被修改。
7. 非空性（Non-nullity）对于任何非空引用值 x，x.equals(null) 都应返回 false。


**实现高质量equals方法的诀窍:**

1. 使用==操作符检查“参数是否为这个对象的引用”；
2. 使用instanceof操作符检查“参数是否为正确的类型”；
3. 把参数转换成正确的类型；
4. 对于该类中的每个“关键”域，检查参数中的域是否与该对象中对应的域相匹配（为了获得最佳性能，应该先比较最有可能不一致的域，或者开销最低的域，最理想的情况是两个条件同时满足的域）；

### 覆盖equals时总要覆盖hashCode

在每个覆盖了equals方法的类中，也必须覆盖hashCode方法。如果不那样做的话，就会违反Object.hashCode的通用约定，从而导致该类无法结合所有基于散列的集合一起正常运作，这样的集合包括HashMap、HashSet和Hashtable。

string 的 hashcode

```java
// Lazily initialized, cached hashCode - Page 49
private volatile int hashCode;  // (See Item 71)
@Override public int hashCode() {
     int result = hashCode;
     if (result == 0) {
         result = 17;
         result = 31  -  result + areaCode;
         result = 31  -  result + prefix;
         result = 31  -  result + lineNumber;
         hashCode = result;
     }
     return result;
}
```

### 始终要覆盖toString

提供好的toString实现可以使类用起来更加舒适，当对象呗传递给println、printf、字符串练操作符（+）以及assert或者被调试器大一出来时，toString方法会被自动调用。

toString方法应该返回对象中包含的所有值得关注的信息。

### 只针对异常的情况才使用异常

现代的JVM实现上，基于异常的模式比标准模式要慢得多。因为它不仅模糊了代码的意图，而且降低了它的性能。甚至不保证正常工作。

异常应该只用于异常的情况下：他们永远不应该用于正常的控制流。

设计良好的API不应该强迫它的客户端为了正常的控制流而使用异常。

### 对可恢复的情况使用受检异常，对编程错误使用运行时异常

java 程序设计有三种可抛出的异常, 分别是受检异常(checked exception), 运行时异常 (runtime exception) 和错误。
如果期望调用者能够适当地恢复，对于这种情况就是应该使用受检的异常

用运行时异常来表明编程错误, 比如 越界异常就是前提被违反了。所有非受检异常都是 runtime exception 的子类。

### 使用常用的异常

**IllegalArgumentException** 当调用者传递的参数不合适时, 使用这个异常。比如参数要求正数, 传了个负数进去, 就可以抛出这个异常

**IllegalStateException** 如果某对象还没被初始化好就执行, 可能会抛出这个异常

**NullPointerException**, **IndexOutOfBoundsException**
 
**ConcurrentModificationException** 被并发修改

**UnsupportedOperationException** 用的很少




