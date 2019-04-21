---
title: CountDownLatch
date: 2016-7-30 16:23:26
tags: [CountDownLatch]
---

## 计数器锁

同步辅助类，允许一个或多个线程等待，直到一系列操作在其它线程中执行成功。线程调用 await 方法进行 block, 线程调用 countDown 方法减少计数器，直到 state = 0，所有wait的线程都醒来了。这个类实现的计数器是 one-shot phenomenon 的，计数器不能够被重置。

可重置的CyclicBarrier： If you need a version that resets the count, consider using a CyclicBarrier. 

当 count == 1, 时，可以将 CountDownLatch 认为是一个开关.

当 count == N,时，A CountDownLatch initialized to N can be used to make one thread wait until N threads have completed some action, or some action has been completed N times. 

**和其它Lock不同，lock 和 unlock 需要成对调用， await 和 countDown 这两个方法不需要线程同时调用，*

**Memory consistency effects: Until the count reaches zero, actions in a thread prior to calling countDown() happen-before actions following a successful return from a corresponding await() in another thread.**

实现非常简洁

``` java

// count 用来初始化 state
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
    
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

	// 只有 state 字段是 0 的时候，线程才可以获得锁
	// 同时请求锁的时候，并不会对字段 state 字段进行任何操作
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

	// 减少计数器，直到 state == 0, 返回 true
	// 此时所有的block在这个锁上的线程，将会被唤醒
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

### lock

``` java
public void await() throws InterruptedException {
	// 阻塞的线程是以共享模式在 sync queue 中 park 的
    sync.acquireSharedInterruptibly(1);
}
```

### release

``` java
public void countDown() {
	// 当 state = 0, 时 调用 doReleaseShared
	// 将 sync queue 中 block 的线程将都会被唤醒
    sync.releaseShared(1);
}
```