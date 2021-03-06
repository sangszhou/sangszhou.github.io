---
layout: post
title: java collection
categories: [web]
description: java collection
keywords: java collection
---

## CopyOnWriteArrayList

```java
public class CopyOnWriteArrayListExample{
    public static void main(String args[]) {
      
        CopyOnWriteArrayList<String> threadSafeList = new CopyOnWriteArrayList<String>();
    
        threadSafeList.add("Java");
        threadSafeList.add("J2EE");
        threadSafeList.add("Collection");
      
        //add, remove operator is not supported by CopyOnWriteArrayList iterator
        Iterator<String> failSafeIterator = threadSafeList.iterator();
        while(failSafeIterator.hasNext()){
            System.out.printf("Read from CopyOnWriteArrayList : %s %n", failSafeIterator.next());
            failSafeIterator.remove(); //not supported in CopyOnWriteArrayList in Java
        }}}
```

源码分析

add Elements

```java
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }}

// 右边的元素向右移动
    
    public void add(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: " + index + ", Size: "+len);
            Object[] newElements;
            int numMoved = len - index;
            if (numMoved == 0)
                newElements = Arrays.copyOf(elements, len + 1);
            else {
                newElements = new Object[len + 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index, newElements, index + 1, numMoved);
            }
            newElements[index] = element;
            setArray(newElements);
        } finally {
            lock.unlock();
        }}

    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index, numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
}}
```

代码量还有很多, 不分析了

### ConcurrentSkipList

SkipList 的实现, 以前没能做出来的原因是, SkipList 中的一个节点不等同于普通链表, 它的一个节点是一整列

![](/images/posts/ds/SkipList.png)

```java
interface SkippableList<T extends Comparable<? Super T>> {
    int LEVELS = 5;
    boolean delete(T target);
    void print();
    void insert(T data);
    SkipNode<T> search(T data); // 感觉应该返回 T
}

class SkipNode<N extends Comparable<? super N>> {
    N data;
    SkipNode<N> [] next = (SkipNode<N> []) new SkipNode[SkippableList.LEVELS];
    SKipNode(N data) {
        this.data = data;
    }
    
    void setNext(SkipNode<N> next, int level) {
        this.next[level] = next;
    }
    SKipNode<N> getNext(int level) {
        return this.next[level];
    }
    
    SkipNode<N> search(N data, int level} {
        SkipNode<N> result = null;
        SkipNode<N> current = this.getNext(level);
        while(current != null && current.data.compareTo(data) < 1) {
            if(current.data.equals(data)) {
                result = current;
                break;
            }
            current = current.getNext(level);
        }
        return result;
    }
    
    void insert(SkipNode<N> skipNode, int level) {
        SkipNode<N> current = this.getNext(level);
        if(current == null) {
            this.setNext(skipNode, level);
            return;
        }
        
        if(skipNode.data.comapreTo(current.data) < 1) {
            this.setNext(skipNode, level);
            skipNode.setNext(current, level);
            return;
        }
        
        while(current.getNext(level) != null &&
            current.data.compareTo(current.data) > 0 ) {
            current = current.getNext(level);   
         }
        
        SkipNode<N> successor = current.getNext(level);
        current.setNext(SkipNode, level);
        skipNode.setNext(successor, level);
    }
}
```

SkipList 本身就很简单了, 它主要借助 SkipNode 完成操作

```java
class SkipList< T extends Comparable<? super T> implements SkippableList<T> {
    private final SkipNode<T> head = new SkipNode<>(null);
   
    public void insert(T data) {
        SkipNode<T> skipNode = new SkipNode<>(data);
        for(int i = 0; i < LEVELS; i ++) {
            // 确定最高的节点是多少
            if(rand.nextInt((int) Math.pow(2, i)) == 0)
                insert(skipNode, i);
        }
    }
    
    private void insert(SkipNode<T> skipNode, int level) {
        head.insert(skipNode, level);
    }
    
    // 并不是最优解法, 应该存在下降搜索的过程
    SkipNode<T> search(T data) {
        SkipNode<T> result = null;
        for(int i = LEVELS -1; i >= 0; i --) {
            if((result == head.search(data, i) != null)
                break;
        }
        return result;
    }
    
    // delete
    
    public boolean delete(T target) {
        SkipNode<T> victim = search(target)
        if(victim == null) return false;
        victim.data = null;
        
        for(int i = 0; i < LEVELS; i ++)
            head.refreshAfterDelete(i);
        
        return true;
    }
    
    
}

```


### ConcurrentLinkedQueue & ConcurrentLinkedDeque

ConcurrentLinkedQueue 的实现形式就和 CAS 那一篇文章写得差不多

ConcurrentLinkedQueue 在本小节后都称为 CLQ。clq 的源码中提到了 garbage free 的问题，那一点似乎很重要，但是还是看不太明白

ConcurrentLinkedDeque 比 clq 复杂的多，因为 cld 允许两头都可以任意的添加删除元素，所以它是一个双向队列

虽然大部分源码看不懂，但是 Node 的操作应该需要理解

```java
    static final class Node<E> {
        volatile Node<E> prev;
        volatile E item;
        volatile Node<E> next;

        Node() {  // default constructor for NEXT_TERMINATOR, PREV_TERMINATOR
        }

        /**
         * Constructs a new node.  Uses relaxed write because item can
         * only be seen after publication via casNext or casPrev.
         */
        Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        void lazySetNext(Node<E> val) {
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        void lazySetPrev(Node<E> val) {
            UNSAFE.putOrderedObject(this, prevOffset, val);
        }

        boolean casPrev(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, prevOffset, cmp, val);
        }

        // Unsafe mechanics

        private static final sun.misc.Unsafe UNSAFE;
        private static final long prevOffset;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                prevOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("prev"));
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```

此外，需要注意 ConcurrentLinkedQueue 中的 head, tail 都是 volatile 的