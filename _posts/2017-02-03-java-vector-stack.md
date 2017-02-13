---
layout: post
title: Java Vector Stack
categories: [java]
keywords: java
---

JDK 文档对于 Vector 的描述如下：

The vector class implements a growable array of objects. Like an array, it contains components that can be accessed using an integer index. However, the size of a vector can grow or `shrink` as needed to accommodate adding and removing items after vector has been created.

Each vector tries to optimize storage management by maintaining a `capacity` and a `capacityIncrement`. The capacity is always at least as large as the vector size. It is usually larger because as components added to vector, the vector's storage increases in chunks the size of capacityIncrement.
An application can increase the capacity of a vector before inserting a large number of components, this reduce the amount of incremental reallocation.

The iterator returned by this class's iterator and listIterator are `fast-fail`: if the vector is structurally modified at any time the iterator's own remove or add methods, the iterator will throw a ConcurrentModificationException. Thus, in the face of concurrent modification, the iterator fails quickly and clearly, rather than risking arbitray, non-deterministic behavior at an undermined time in the future. `Enumerations` returned by the elements method are `not` fail-fast.

Note that the fail-fast behavior of an iterator cannot be guaranteed as it is, generally speaking, impossible concurrent modification. Fail-fast iterators throw ConcurrentModificationException on a best-effor basis. Therefor, it would be wrong to write a program that depended on this exception for its correctness: the fail-fast behavior of iterators should be only to detect bugs.

Note: capacity is the length of elementData object array.

```java
public class Vector extends AbstractList implements List, RandomAccess, Cloneable {
        Object[] elementData;
        int elementCount;
        int capacityIncrement;
        final long serialVersionUID = -2767605614048989439L;
    
        
    
        private void grow(int minCapacity) {
            // overflow-conscious code
            int oldCapacity = elementData.length;
            int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                             capacityIncrement : oldCapacity);
            if (newCapacity - minCapacity < 0)
                newCapacity = minCapacity;
            if (newCapacity - MAX_ARRAY_SIZE > 0)
                newCapacity = hugeCapacity(minCapacity);
            elementData = Arrays.copyOf(elementData, newCapacity);
        }
}
```

从类的声明来看，vector 的实现和 arrayList 几乎是一致的，只不过 vector 的关键方法都被 synchronize 方法修饰，是线程安全的。Stack 是 vector 的子类，对 stack 的 pop, peek, push 都是在 vector 的基础上完成的。

官方已经不推荐使用 Vector 和 Stack。要使用线程安全的 List，可以用 Collections.synchronizedList 来 包裹 ArrayList。至于栈，则推荐使用 Deque 接口的实现，如 LinkedList 和 ArrayDeque。

