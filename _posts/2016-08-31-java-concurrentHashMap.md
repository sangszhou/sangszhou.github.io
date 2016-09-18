---
layout: post
title:  "Java Concurrent Hash Map 分析"
date:   "2016-08-31 10:50:00"
categories: java
keywords: java, concurrent
---

# ConcurrentHashMap

ConcurrentHashMap 的基本原理是把一个 hashMap 转化为多个片段，每个片段使用一把锁，访问不同的数据片段时不需要等待，避免全局锁提供了并发性能。

![ConcurrentHashMapConcept][1]

数据结构在代码中的体现

```java
ConcurrentHashMap
  Segment<K, V>[] segment

static class Segment<K, V> extends ReentrantLock
  HashEntry<K,V>[] table
  int count
  int threshold
  float loadFactor


static class HashEntry<K, V>
  int Hash
  K key
  V value
  HashEntry<K, V> next
```

## Put 流程

从 ConcurrentHashMap 直到插入 HashEntry 数组

```java
// ConcurrentHashMap
V put(K key, V value)
  int hash = hash(Key)
  int j = (hash >>> segmentShift) & segmentMask
  int ((s = (Segment<K, V> UNSAFE.getObject(segments, (j << SSHIFT) + BASE))))
    s = ensureSegment(j)
  return s.put(key, hash, value, false) // last argument, putIfnotAbsent

// Segment
V put(K key, int hash, V value, boolean onlyIfAbsent)
  // 等待的同时扫描元素，什么时候回来？
  HashEntry node = tryLock ? null : scanAndLockForPut(key, hash, value)
  V oldValue
  try
    HashEntry [] tab = table
    int index = (tab.length -1 ) & hash // 没什么随机性
    // 找到第一个元素
    HashEntry first = entryAt(tab, index)
    for(HashEntry e = first; ;)
      if(e != null)
        if((k = e.key) == key || e.hash == hash && key.equals(key))
          oldValue = e.value
          if(!onlyIfAbsent)
            e.value = value
            ++ modCount
          break
        e = e.next
      else // 遍历完毕, 没有找到。插入链表头
        if(node != null)  // 在 scanAndLockForPut 中找到插入位置
          node.setNext(first)
        else
          node = new HashEntry(hash, key, value, first)
        int c = count + 1
        if(c > threshold && tab.length < MAXIMUM_CAPACITY)
          rehash(node)
        else
          setEntryAt(table, index, node)
        ++ modCount
        count = c
        oldValue = null
        break
    finally
      unlock
    return oldValue    
```

ConcurrentHashMap 每个函数都很长，每行都很重要，没什么可以省略的。这里有几点需要注意:

1. 当 tryLock 失败时，线程并没有死等着，而是尝试读取自己应该在的位置
2. index, 它是 hash 与 tab.length -1 的 & 生成的，因为 tab.length 总是 2 的幂，所以 index 是从 0 ~ tab.length-1 取值
3. 当 key 已经存在，是否替换 value 要根据参数确定
4. 当首次遇到为空的 HashEntry 时，新节点插入的位置是 HashEntry 链表的头结点，这样有点 cache 的意思
5. 当 c > threshold 且 c < Maximum capacity 时进行 rehash

```java
HashEntry scanAndLockForPut(K key, int hash, V value)
  HashEntry first = entryForHash(this, hash)
  HashEntry first = e
  HashEntry node = null //参数组成的 node

  int retries = -1 //negative while locating node
  while(!tryLock)
    HashEntry f
    if(retries < 0)
      if(e == null) // e == null 了还是没找到
        if(node == null)
          node = new HashEntry(hash, key, value, null)
        retries = 0
      else if(key.equals(e.key))
        retries = 0 // 找到 key 失败的了
      else
        e = e.next
    else if ++retries > MAX_SCAN_RETRIES // 从第一个 If 中跳出来了
      lock // 在哪被唤醒？
      break
    else if (retries & 1 ) == 0 && (f = entryForHash(this, hash) != first)
      e = first = f // first has changed, retries again
      retries = -1
    endWhile
    return node
```

scanAndLockForPut 函数也很巧妙，在发现锁不可用时，尝试定位到自己应该放的位置。不过做了很多工作，效果却不明显，只比正常的逻辑少一个初始化阶段而已，这么复杂的函数，应该是为了不让自己被 lock 住，因为 lock 本身需要系统调用，这个函数也有几点需要注意

1. entryForHash 和 put 函数的定位 first 有什么区别？
2. retries 为负时，表示正在定位链表状态，为 0 时表示已经定位到，当为 2 的倍数时，判断一下链表是否已经发生了变化

当 Segment 中的某个 HashEntry 长度超过一定值时，开始 rehash 模式，其实就是把 Segment 中 HashEntry 数组的长度扩充一倍

```java
// rehash 后，HashEntry[] 某个 index 下的 HashEntry 节点要么仍在此 index 上，要么移动 2^N 个位置，到 HashEntry[index+2^N] 链上
void rehash(HashEntry node)
  HashEntry[] oldTable = table
  int oldCapacity = oldTable.length
  int newCapacity = oldCapacity << 1
  threshold = newCapacity * loadFactor
  HashEntry[] newTable = new HashEntry[newCapacity]
  int sizeMark = newCapacity - 1 // new mask

  for(int i = 0; i < oldCapacity; i ++)
    HashEntry e = oldTable[i]
    if(e != null) // 首个元素即为空就不再处理了
      HashEntry next = e.next
      int idx = e.hash & sizeMark // new index
      if(next == null) newTable[idx] = e //single node
      else
        HashEntry lastRun = e
        int lastIdx = idx
        // 找到哨兵位置， 这个哨兵有什么用呢？(尽量重用 list, 没有用到什么特性)
        for(HashEntry last = next; last != null; last = last.next)
          int k = last.hash & sizeMask
          if k != lastIdx
            lastIdx = k
            lastRun = last

        newTable[lastIdx] = lastRun; //设置头节点

        for(HashEntry p = e; p != lastRun; p = p.next)
          v = p.value
          int h = p.hash
          k = h & sizeMask
          HashEntry n = new HashEntry[k]
          newTable[k] = new HashEntry(h, p.key, v, n)
  endfor
    // 插入参数节点
    int nodeIndex = node.hash & sizeMask
    node.setNext(newTable[nodeIndex]) // 为空也没关系
    newTable[nodeIndex] = node
    table = newTable
```

参数节点还是放到 HashEntry 的第一个位置上，但是原始节点的顺序就都乱掉了。
没看明白中间那段找哨兵(lastRun) 的意义在哪，每个节点都像插入 node 那样又有什么问题呢

哨兵的身后的节点，hash 值不变，这意味着这些节点都将存在于同一个 new index 下，这样能够减少一定数量的数据拷贝。
其实，这种优化不一定有效，只是做个尝试，为了这些优化多循环了一遍链表，不知道是不是合适。

## Get 流程

```java
V get(Object key)
  Segment s
  HashEntry[] tab
  int h = hash(key)
  long u =  ((h >>> segmentShift) & segmentMask) << SSHIFT + SBASE
  if((s = UNSAFE.getObjectVolatile(segments, u)) != null && s.table != null)
    for(e = UNSAFE.getObjectVolatile(table, Mask & h); e != null; e = e.next)
      if(k.key == key || (e.hash == h && key equals e.key))
        return e.value
  return null
```

get 的流程比较直观，没有陷入到 Segment 中查找元素。containsKey 方法与 get 几乎完全相同，除了它返回的是 boolean.

containsValue 函数从 Map 中查找元素，如果第一遍查不到的会再查找，如果在查找的过程中 Map size 发生了变化，就加锁再查

```java
boolean containsValue(Object value)
  Segments[] segments = this.segments
  int last = 0
  int retries = -1
  boolean found = false

  try
    outer: for(;;)
      if(++ retries == RETRIES_BEFORE_LOCK)
        for(int i = 0; i < segments.length; i ++)
          Segments[i].ensureSegment().lock
      int sum = 0
      for(int i = 0; i < segments.length; i ++)
        HashEntry [] tab
        Segment seg = SegmentAt(segments, i)
        if(seg != null && (tab = seg.table) != null)
          for(int j = 0; j < tab.length; j ++)
            HashEntry e
            for( e = entryAt(table, j), e != null; e = e.next)
              V v = e.value
            if( v != null && value.equals(v))
              found = true
              break outer // 没直接 return 是因为当前可能加锁了
          sum += seg.modCount

      if(retries > 0 && last == sum) (1)
        break //break outer
      last = sum
  finally
    if(retries > RETRIES_BEFORE_LOCK)
      for(int i = 0; i < segments.length; i ++)
        SegmentAt(segments, i).unlock
```

RETRIES_BEFORE_LOCK 默认值是 2，retries 初始值是 -1. 第一次查找未成功时，(1) 处判断不通过，再查找一次，如果第二次未通过且 sum 未变，返回 false, sum 变了进入第三次循环，第三次循环会加锁，这次是最后一次

size 函数和 containsValue 的写法类似，但是 (1) 处没有对 retries > 0 的判断，我觉得这个判断也是多余的，第一次 last 肯定不等于 sum, 除非 map 是空的

## Remove 流程

因为 remove 是 mutate 操作，所以需要加锁且会更改 modCount (mutate 操作数)

```java
V remove(Object key, int hash, Object value)
  if(!tryLock) scanAndLock(key, hash)
  val oldValue = null
  try
    HashEntry[] tab = table
    int index = hash & (tab.length - 1)
    HashEntry e = entryAt(tab, index)
    HashEntry pred = null

    while(e != null)
      HashEntry next = e.next
      if(k == e.key || (e.hash == hash && key.equals k))
        v = e.value
        // 这个是为了让 remove 操作能够 Handle 的场景更多
        if(value == null || value == v || value.equals(v))
          if(pred == null) setEntryAt(tab, index, next)
          else pred.setNext(next)
          ++ modCount
          --count
          oldValue = v
      pred = e
      e = next
  finally
    unlock
```

remove 的 value 参数是为了增加 remove 的功能，假如要求只有当 (key, value) 中的 value 没变时才删除 value, 第三个参数的作用就体现出来登了
剩下的就是简单的链表操作，从单链表中删除一个元素

scanAndLock 和 scanAndLockForPut 相比，操作更加简单，应该也是为了拖延时间防止进入 lock 状态

# HashMap
1.8 版的 HashMap 变化比较大, 先从 1.7 看起来

![hashMap](/images/posts/java/collection/hashmap.JPG)

图的架构和 ConcurrentHashMap 有点像, 这里的链表叫做 Entry, ConcurrentHashMap 叫做 Node

先看 put 函数

```java
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null) return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

没有任何 CAS 操作, 看起来很清晰。inflateTable 是用来 new Entry list 的, 因为没有并行的概念, 所以就没有 Segment 
 这一层。indexFor 还是求 hash 与 size-1 并, size 也是 2 的幂。 只有当 hash 相同 并且 key 的 == 和 equals 运算同时
 成了的情况下, 才认为相当。
 
真正的添加元素运算分为两种, 第一种是 putForNullKey, 第二种是 addEntry

```java
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
            }
        }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

null 只会出现在 table[0] 中, 我想 find null key 肯定也是来这找

create Entry 依然是把新建的节点放到链表的头部, 这边没有直接使用 loadFactor 而是直接用 threshold

在 concurrentHashMap 中, rehash 操作还是很麻烦的, 需要考虑到并发和优化, 而 hashMap 的处理就很暴力了, 没有考虑任何并发和优化

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table)
        while(null != e) 
            Entry<K,V> next = e.next;
            if (rehash) e.hash = null == e.key ? 0 : hash(e.key);
            
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;        
```

## Get 操作

```java
public V get(Object key) {
    if (key == null) return getForNullKey();
    Entry<K,V> entry = getEntry(key);
    return null == entry ? null : entry.getValue();

final Entry<K,V> getEntry(Object key) {
        if (size == 0) return null;
        
        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)]; e != null; e = e.next)
            Object k;
            if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        return null;
```

Get 操作依然很暴力, 没有考虑一次没找到, 再找一次的情况。

## Remove 操作

```java
public V remove(Object key) {
    Entry<K,V> e = removeEntryForKey(key);
    return (e == null ? null : e.value);
}

final Entry<K,V> removeEntryForKey(Object key) {
    if (size == 0) return null;
        
    int hash = (key == null) ? 0 : hash(key);
    int i = indexFor(hash, table.length);
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            if (prev == e) table[i] = next;
            else
                prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }
        return e;
    }
```

# HashTable

HashTable 是 hashMap 的多线程版本, 通过使用 synchronized 得到线程安全的效果
 
## put 操作
 
```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) throw new NullPointerException();

    // Makes sure the key is not already in the hashtable.
    Entry tab[] = table;
    int hash = hash(key); // hashSeed ^ k.hashCode
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            V old = e.value;
            e.value = value;
            return old;
    modCount++;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();
    
        tab = table;
        hash = hash(key);
        index = (hash & 0x7FFFFFFF) % tab.length;            
```

rehash 名字和 concurrentHashMap 的一致了, 但是实现还是和 hashMap 一样, 因为调用 rehash
的函数确保被 synchronized 保护, 所以 rehash 就没有再加上 synchronized 关键字

# 问题

## 为什么String, Interger这样的wrapper类适合作为键
String, Interger这样的wrapper类作为HashMap的键是再适合不过了，而且String最为常用。
因为String是不可变的，也是final的，而且已经重写了equals()和hashCode()方法了。其他的wrapper类也有这个特点。
不可变性是必要的，因为为了要计算hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的hashcode的话，
那么就不能从HashMap中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个field声明成final就
能保证hashCode是不变的，那么请这么做吧。因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的hashcode的话，那么碰撞的几率就会小些，这样就能提高HashMap的性能。

## 我们可以使用自定义的对象作为键吗

这是前一个问题的延伸。当然你可能使用任何对象作为键，只要它遵守了equals()和hashCode()方法的定义规则，
并且当对象插入到Map中之后将不会再改变了。如果这个自定义对象时不可变的，那么它已经满足了作为键的条件，因为当它
创建之后就已经不能改变了。

## 我们可以使用CocurrentHashMap来代替HashTable吗

这是另外一个很热门的面试题，因为ConcurrentHashMap越来越多人用了。我们知道HashTable是synchronized的，但是ConcurrentHashMap同步性能更好，因为它仅仅根据同步级别对map的一部分进行上锁。ConcurrentHashMap当然可以代替HashTable，
但是HashTable提供更强的线程安全性。

## 如果HashMap的大小超过了负载因子(load factor)定义的容量

默认的负载因子大小为0.75，也就是说，当一个map填满了75%的bucket时候，和其它集合类(如ArrayList等)一样，
将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。
这个过程叫作rehashing，因为它调用hash方法找到新的bucket位置。

## 重新调整HashMap大小存在什么问题吗

当重新调整HashMap大小的时候，确实存在条件竞争，因为如果两个线程都发现HashMap需要重新调整大小了，它们会同时试着调整大小。在调整大小的过程中，存储在LinkedList中的元素的次序会反过来，因为移动到新的bucket位置的时候，HashMap并不会将元素放在LinkedList的尾部，
而是放在头部，这是为了避免尾部遍历(tail traversing)。如果条件竞争发生了，那么就死循环了


## 当两个对象的hashcode相同会发生什么
一些面试者会回答因为hashcode相同，所以两个对象是相等的，HashMap将会抛出异常，或者不会存储它们。
然后面试官可能会提醒他们有equals()和hashCode()两个方法，并告诉他们两个对象就算hashcode相同，但是它们可能并不相等。
一些面试者可能就此放弃，而另外一些还能继续挺进，他们回答“因为hashcode相同，所以它们的bucket位置相同，‘碰撞’会发生。
因为HashMap使用LinkedList存储对象，这个Entry(包含有键值对的Map.Entry对象)会存储在LinkedList中

## 如果两个键的hashcode相同，你如何获取值对象

当我们调用get()方法，HashMap会使用键对象的hashcode找到bucket位置，然后获取值对象。面试官提醒他如果有两个值对象
储存在同一个bucket，他给出答案:将会遍历LinkedList直到找到值对象。面试官会问因为你并没有值对象去比较，你是如何确定
确定找到值对象的？除非面试者直到HashMap在LinkedList中存储的是键值对，否则他们不可能回答出这一题。

其中一些记得这个重要知识点的面试者会说，找到bucket位置之后，会调用keys.equals()方法去找到LinkedList中正确的节点，
最终找到要找的值对象。完美的答案！
许多情况下，面试者会在这个环节中出错，因为他们混淆了hashCode()和equals()方法。因为在此之前hashCode()屡屡出现，
而equals()方法仅仅在获取值对象的时候才出现。一些优秀的开发者会指出使用不可变的、声明作final的对象，并且采用合适
的equals()和hashCode()方法的话，将会减少碰撞的发生，提高效率。不可变性使得能够缓存不同键的hashcode，这将提高整个获取对
象的速度，使用String，Interger这样的wrapper类作为键是非常好的选择。



[1]: /images/posts/javaconcurrent/ConcurrentHashMap.png
