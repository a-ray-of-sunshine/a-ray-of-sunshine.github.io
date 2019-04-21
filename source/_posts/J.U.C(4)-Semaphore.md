---
title: Semaphore
date: 2016-7-25 14:46:45
tags: [Semaphore]
---

## 锁状态的初始化

``` java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

NonfairSync(int permits) {
    super(permits);
}

Sync(int permits) {
    setState(permits);
}
```

将传递进来的参数，permit, 作为， state 字段的初始值。表示**可以有几个线程获取锁**，或者说是：state的值表示，还剩几个锁可以被直接获取到。当 state == 0 的时候，表示Semaphore持有的锁已经全部被使用了，再没有可以使用的锁的，此时如果一个线程请求获取锁，则这个线程将进行入 snyc queue，从而进入 block 状态。

## acquire

acquire 方法最终调用： java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly 方法来获取锁。


和 ReentrantLock 不同的是，这里的 state 的语义发生的变化。

### release

``` java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
    	// 释放成功，调用 doReleaseShared 方法
        doReleaseShared();
        return true;
    }
    return false;
}

// tryReleaseShared将 current + releases 设置成
// state.
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

## doReleaseShared

释放锁是实现信号量(Semaphore)的关键。对于互斥锁来说，release 就是释放 head 处的 node 就可以了。

但是对于，共享锁来说，一次 tryRelease(permits) 调用有可以释放多个锁，例如 r 个锁，所以至少可以有 r 个线程来进行调度了。所以共享锁（Semaphore）的释放是比较复杂的。


其基本思路就是不断的释放 head.直到 head 不变为止。head 不发生变化有两种情况：

* head 已经是最后一个结点了

* 锁使用完了，state == 0 了。

doReleaseShared 在线程 A 上调用。

线程 B, C, D 在 doAcquireSharedInterruptibly方法的
parkAndCheckInterrupt 处自旋等待。

``` java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 假设线程 A 执行成功到这里，使得
                // 线程 B 被唤醒。线程B 被唤醒之后
                // 执行 setHeadAndPropagate
                // 使得 head 发生改变。
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        
        // 由于线程 B 唤醒，head 有可能被改变
        // 不同的线程，无法确定执行的顺序所以
        // 只能说，head 有可能被改变。如果线程B
        // 没来得及改变 head, 下面的判断将成功
        // 循环将中断（跳出），！！！这样很可能造成，
        // 空闲的锁还有，却没法唤醒线程去竞争锁
        // 造成性能问题。！！！
        if (h == head)                   // loop if head changed
            break;
    }
}
```

至少有一个线程会因为上面的 doReleaseShared 调用而被唤醒，所以上面出现的锁空闲的问题就交给这个被唤醒的线程来继续完成。

``` java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 在队列里面的线程是一个个按顺序被唤醒的
    // 所以 setHead 方法在改变 head 的时候并不需要
    // 进行同步或者是使用 compareAndSetHead 方法
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    // head 结点唤醒之后，如果处于 Propagation 模式
    // 并且，node 的后继是以 shared mode 在 wait
    // 则 唤醒这个结点。
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

这样 释放锁的线程的 doReleaseShared 方法调用 和获得锁的线程的 setHeadAndPropagate 方法调用，会相互激发调用，使得 信号量 所拥有的空闲锁尽快被等待的线程所持有，从而提高的并发的性能。

## 文档

semaphore 将 state 字段的语义定义为：可以并发访问共享资源的线程个数。

acquire 方法调用会减少 permit 的可用个数。

release 方法可以增加 permit 的可用个数。

Semaphore（信号量）对同步语义的实现和 ReentrantLock (可重入锁) 完全不同。Semaphore 实现的同步控制是对共享资源的并发访问线程个数的同步。所以 Semaphore 所获取的permit 并没有 ReentrantLock 的 lock 功能。所以对于资源的并发修改进行同步还需要互斥锁进行同步。不会维护数据一致性。

Note that no synchronization lock is held when acquire() is called as that would prevent an item from being returned to the pool. The semaphore encapsulates the synchronization needed to restrict access to the pool, separately from any synchronization needed to maintain the consistency of the pool itself. 

如果一个信号量的 permit 是 1，则这个信号量也被称为：binary semaphore. 以这种方式使用 semaphor 时，相当于一个互斥锁，因为任意时刻，只有一个线程可以获取锁。

信号量并没有所有权的概念。

内存一致性效应：Actions in a thread prior to calling a "release" method such as release() happen-before actions following a successful "acquire" method such as acquire() in another thread.



