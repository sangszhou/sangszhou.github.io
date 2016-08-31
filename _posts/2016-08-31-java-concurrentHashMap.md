---
layout: post
title:  "Java ConcurrentHashMap 分析"
date:   "2016-08-30 10:50:00"
categories: Java
keywords: Java, Concurrent, map
---

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
      else // 插入链表头
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
        // 找到哨兵位置， 这个哨兵有什么用呢？
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

[1]: /images/posts/javaconcurrent/ConcurrentHashMap.png
