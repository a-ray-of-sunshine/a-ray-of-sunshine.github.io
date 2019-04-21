---
title: 并发容器-BlockingQueue(1)-ArrayBlockingQueue
date: 2016-8-1 10:58:39
tags: [BlockingQueue]
---

## BlockingQueue

### API

BlockingQueue methods come in four forms, with different ways of handling operations that cannot be satisfied immediately, but may be satisfied at some point in the future: 

* one throws an exception, 

* the second returns a special value (either null or false, depending on the operation), 

* the third blocks the current thread indefinitely until the operation can succeed, 

* and the fourth blocks for only a given maximum time limit before giving up. 

These methods are summarized in the following table: 

 | Throws exception | Special value | Blocks | Times out
---|:---:|:---:|:---:|:---:
Insert|add(e)|offer(e)|put(e)|offer(e, time, unit)
Remove|remove()|poll()|take()|poll(time, unit)
Examine|elemnt()|peek()|N/A|N/A

### 对 null 的支持
**A BlockingQueue does not accept null elements.**

### 应用场景
BlockingQueue接口的实现主要的应用场景是 producer-consumer 队列。但是他是支持集合类接口，其中有一个 remove(x)这个方法可以用来删除队列中的任意元素，但是这类集合类操作的实现性能并不是特别高，所以这类操作通常使用在特殊的场景下，例如队列中的message可以被 cancell,这样就需要remove来实现了。

### 线程安全
BlockingQueue的实现类应该是线程安全的。对于 队列方法 的实现是原子操作，也就是线程安全的。而对于批量操作（bulk operation）可以不是线程安全，例如： for addAll(c) to fail (throwing an exception) after adding only some of the elements in c. 

### 内存一致性
As with other concurrent collections, actions in a thread prior to placing an object into a BlockingQueue happen-before actions subsequent to the access or removal of that element from the BlockingQueue in another thread. 

**elements A put in queue happen-before elements A access by another thread.** 

## ArrayBlockingQueue

固定容量的Queue

### 初始化
``` java
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    // 创建容量为 capacity 的数组来存放元素
    this.items = new Object[capacity];
    
    // 初始化锁
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

* lock: Main lock guarding all access
* notEmpty: Condition for waiting takes until the queue is not empty
* notFull: Condition for waiting puts until the queue is not full
* count: Number of elements in the queue
* takeIndex: items index for next take, poll, peek or remove
* putIndex: items index for next put, offer, or add

### put
``` java
public void put(E e) throws InterruptedException {
	// 如果插入的元素是null,
	// 则直接抛NullPointerException
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 如果在请求锁的过程中当前线程被中断
    // 则抛出 InterruptedException
    lock.lockInterruptibly();
    try {
    	// 如果队列满了，则进行 wait 状态
        while (count == items.length)
            notFull.await();
        // 插入元素
        insert(e);
    } finally {
        lock.unlock();
    }
}
```
### insert
``` java
private void insert(E x) {
    items[putIndex] = x;
    // 将 putIndex 移动到下一个等待插入的位置
    putIndex = inc(putIndex);
    ++count;
    // 已经成功添加了一个元素，所以
    // notEmpty 条件已经满足，所以 signal.
    notEmpty.signal();
}
```

### take
``` java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
    	//等待直到queue中有元素。
        while (count == 0)
            notEmpty.await();
        return extract();
    } finally {
        lock.unlock();
    }
}
```

### extract
``` java
private E extract() {
    final Object[] items = this.items;
    // 取出元素
    E x = this.<E>cast(items[takeIndex]);
    // help for GC
    items[takeIndex] = null;
    // takeIndex 向前移动，因为，插入的索引
    // putIndex 也是向前移动的。
    takeIndex = inc(takeIndex);
    --count;
    notFull.signal();
    return x;
}
```

### add
``` java
public boolean add(E e) {
    if (offer(e))
        return true;
    else
    	// 添加失败，抛异常
        throw new IllegalStateException("Queue full");
}
```

### remove
``` java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    	// 遍历的次数由 k 来控制：count次
    	// 遍历的顺序由 i 来控制：从 takeIndex 开始
    	// 遍历 count 次，
    	// 按照元素入队列的顺序进行遍历，直到找到
    	// 第一个和待删除元素o相等的元素，将其删除
        for (int i = takeIndex, k = count; k > 0; i = inc(i), k--) {
            if (o.equals(items[i])) {
                removeAt(i);
                return true;
            }
        }
        return false;
    } finally {
        lock.unlock();
    }
}

void removeAt(int i) {
    final Object[] items = this.items;
    // if removing front item, just advance
    if (i == takeIndex) {
    	// 如果删除的是队列头部的元素
    	// 直接将 takeIndex 步进即可。
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
    } else {
        // slide over all others up through putIndex.
        // 从位置 i 开始将后面的元素依次向前移动
        // 最后调整好 putIndex 的值。
        for (;;) {
            int nexti = inc(i);
            if (nexti != putIndex) {
                items[i] = items[nexti];
                i = nexti;
            } else {
                items[i] = null;
                putIndex = i;
                break;
            }
        }
    }
    --count;
    notFull.signal();
}
```

### clear
清除队列中的所有元素，将内部的状态变量恢复到初始状态。

### drainTo
这个方法的作用是将当前queue中的元素按照FIFO的顺序插入到参数Collection中。
``` java
public int drainTo(Collection<? super E> c) {
    checkNotNull(c);
    if (c == this)
        throw new IllegalArgumentException();
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = takeIndex;
        int n = 0;
        int max = count;
        while (n < max) {
        	// 按照 FIFO 顺序
            c.add(this.<E>cast(items[i]));
            items[i] = null;
            i = inc(i);
            ++n;
        }
        if (n > 0) {
        // 只有一种情况会出现 n<=0
        // 就是 count = 0, 也就是
        // 当前 queue 中并没有元素，
        // 所以也就不需要，进行状态初始化操作了
            count = 0;
            putIndex = 0;
            takeIndex = 0;
         // 因为queue中没有元素了，所以
         // 通知所有在 notFull 条件上 wait 的线程
            notFull.signalAll();
        }
        return n;
    } finally {
        lock.unlock();
    }
```

### iterator

iterator是其创建时刻时的queue的一个快照，它所持有的关于queue的状态信息，只来自于创建的时刻，至于之后queue是否发生变化iterator并不关心。

这个类提供的 iterator 是具有弱一致性，同时它也仅仅代表iterator 被创建的时刻的 queue 的状态，

``` java
// 构造方法
Itr() {
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        lastRet = -1;
        // 在iterator 被创建的时刻的状态
        // remaining = count
        // nextItem = itemAt(nextIndex = takeIndex)
        // 有可能在这个iterator被创建之后，当前
        // queue中元素又增加了，count变大了
        // 而这里的 remaining 维持的还是原来的count
        // 在iterator被创建之后新增加的元素，将不会被
        // next方法返回。
        if ((remaining = count) > 0)
            nextItem = itemAt(nextIndex = takeIndex);
    } finally {
        lock.unlock();
    }
}

// next 方法
public E next() {
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        if (remaining <= 0)
            throw new NoSuchElementException();
        lastRet = nextIndex;
        E x = itemAt(nextIndex);  // check for fresher value
        if (x == null) {
        	// 即使当前值已经被修改
        	// next 方法依旧返回快照元素
        	// 而不是 null
            x = nextItem;         // we are forced to report old value
            lastItem = null;      // but ensure remove fails
        }
        else
            lastItem = x;
            
        // 跳过所有Null元素，注意 remaining 也会
        // 相应减少，所以 next 能够执行的次数一定是
        // <= iterator 创建时刻的queue的count的。
        while (--remaining > 0 && // skip over nulls
               (nextItem = itemAt(nextIndex = inc(nextIndex))) == null)
            ;
        return x;
    } finally {
        lock.unlock();
    }
}
```

由 next 方法实现可以确定，这个iterator返回的是queue的快照元素，因为在并发的情况下，nextItem 记录的元素很有可能已经被消费，而 next 方法却依旧会返回它。

这也说 iterator 是弱一致性的，iterator在循环过程中可以容忍并发地对 queue 进行修改，而不会抛出ConcurrentModificationException。

### bulk operator
ArrayBlockingQueue类没有重写 addAll, containsAll, retainAll and removeAll 这四个批量操作方法，所有虽然其中的 add, contains 方法是原子操作，但是这些批量操作方法却是通过循环来完成，所以它们并不是原子操作。