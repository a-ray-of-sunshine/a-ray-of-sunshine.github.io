---
title: 并发容器-BlockingQueue(2)-LinkedBlockingQueue
date: 2016-8-3 16:12:22
tags: [BlockingQueue, LinkedBlockingQueue]
---

BlockingQueue 的基于链表的实现

在其内部，实现中使用，two lock queue, 使用是两个锁，

``` java
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();
```

对于记录queue中元素个数的字段 count 使用 AtomicInteger类型，而不是直接的 int 型。主要原因是这个count字段会在 put 和 take 时都会被修改，所以为了锁的独立使用，直接使用 AtomicInteger 来保证 count 的原子操作。 


**理解下面的代码一定要从多个线程并发的实际情况下来理解**

## put 过程

``` java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        /*
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards.
         */
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        
        // 注意到 c 表示的 count 增加之前的值
        c = count.getAndIncrement();
        // c+1代表当前的count，如果满足下面的条件
        // 则表明当前队列还未满，所以调用 notFull.signal()
		// 同时 notFull 应该在获得 putLock 的时候被调用
		// 所以这里在 释放 putLock 锁之前调用		
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    // 如果 c == 0 ,则 通过上面的put，此时其时 count == 1
    // 所以队列中就应该不是空的，所以可以通知 taker 线程，可以
    // 从队列中取数据了。
    if (c == 0)
        signalNotEmpty();
}
```

## take

``` java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        // 和 put 过程一样，首先获得 count 增加之前的值。
        c = count.getAndDecrement();
        // 所以当前线程 take 完成之后，还有数据
        // 则通知其它 taker 线程，继续取数据。
        
        // 考虑这样一种情况：假设，队列的 capacity = 5，而此时队列中
        // 而一个元素也没有，然后有3个线程在等待 take, 而有个空闲线程向
        // 队列中 put 了 2 个元素，则等待的3个线程会有一个醒来，这个醒来
        // 的线程 take 完成之后，瞅瞅，队列中还有没有元素，如果有的话
        // 通知其它兄弟线程（其余两个等待的take线程），让它们来取数据
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    // 如果 c == capacity,则通过上面的 take 之后 ，
    // count 就变成了 capacity - 1， 所以此时
    // 队列就不是空的了，所以通知 put 线程可以
    // 向 队列中 put 数据了。
    if (c == capacity)
        signalNotFull();
    return x;
}
```

* 对 null 值的支持：不支持 null 类型的值。直接抛异常
* 返回的 iterator 也是 "weakly consistent" 的。
