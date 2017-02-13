---
layout: post
title: Concurrent queue and more
categories: [distributed]
description: cas
keywords: cas
---

### Non-intrusive MPSC node-based queue

Advantages:

+ Wait free and fast producers. One XCHG is maximum what one can get with multi-producer non-distributed queue.

+ Extremely fast consumer. On fast-path it's atomic-free, XCHG executed per node batch, in order to grab 'last item'.

+ No need for node order reversion. So pop operation is always O(1).

+ ABA-free.

+ No need for PDR. That is, one can use this algorithm out-of-the-box. No need for thread registration/deregistration, periodic activity, deferred garbage etc.

Disadvantages:

- Push function is blocking wrt consumer. I.e. if producer blocked in (*), then consumer is blocked too. Fortunately 'window of inconsistency' is extremely small - producer must be blocked exactly in (*). Actually it's disadvantage only as compared with totally lockfree algorithm. It's still much better lock-based algorithm.

- The algorithm is not linearizable.

```c

tail -> N1 -> N2 ----> head

每次追加元素，总是在 head 后，tail pop 元素

struct mpscq_node_t
{
    mpscq_node_t* volatile  next;
    void*                   state;

};

struct mpscq_t
{
    mpscq_node_t* volatile  head;
    mpscq_node_t*           tail;
};

void mpscq_create(mpscq_t* self, mpscq_node_t* stub)
{
    stub->next = 0;
    self->head = stub;
    self->tail = stub;
}

void mpscq_push(mpscq_t* self, mpscq_node_t* n)
{
    n->next = 0;
    mpscq_node_t* prev = XCHG(&self->head, n); // serialization-point wrt producers, acquire-release
    prev->next = n; // serialization-point wrt consumer, release
}

mpscq_node_t* mpscq_pop(mpscq_t* self)
{
    mpscq_node_t* tail = self->tail;
    mpscq_node_t* next = tail->next; // serialization-point wrt producers, acquire
    if (next)
    {
        self->tail = next;
        tail->state = next->state;
        return tail;
    }
    return 0;
}
```

再看 scala 的 Message queue 实现，发现其并没有使用上述的 CAS queue 而是直接使用了 Java 的 Queue, 有 ConcurrentLinkedQueue 等等

## Disruptor 的设计方案

避免垃圾回收，使用数组而非链表

数组长度为 2^n 通过位运算，快速定位到元素

无锁

设计实现由一个生产者和多个生产者的区别

## ConcurrentLinkedQueue


