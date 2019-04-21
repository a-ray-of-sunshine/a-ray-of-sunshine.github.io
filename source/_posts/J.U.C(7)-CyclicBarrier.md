---
title: CyclicBarrier
date: 2016-7-30 16:23:26
tags: [CyclicBarrier]
---

CyclicBarrier

同步辅助类，允许一组线程互相等待，直到 reach a common on barrier point. barrier point 是 parties 减少到0的时候。CyclicBarrier 在涉及到固定的 party 线程等待成功之后执行一个动作（barrierCommand）的场景。当等待条件满足之后，这个 Barrier 会被重置，所以这个 Barrier 可以循环使用，所以会有 Cyclic（循环）修饰。

barrierCommand 在  barrier point 到达的时候，被会执行，但是，此时其它的线程并没有被释放。直到 barrierCommand 执行完成，其它 party 线程才会从 await中唤醒，然后返回。A CyclicBarrier supports an optional Runnable command that is run once per barrier point, after the last thread in the party arrives, but before any threads are released. 

barrierCommand 一般用来做什么呢？在其它 party 继续执行之前，更新共享状态。This barrier action is useful for updating shared-state before any of the parties continue. 

* dowait --- CyclicBarrier 实现的核心

``` java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 由于 await 方法会被多个线程并发调用，而这个方法中使用到了这个类的
    // 全局变量: generation, count， 所以为了线程安全，这里执行了加锁
    // 操作，注意，这和CyclicBarrier的核心功能并没有关系。
    // CyclicBarrier 的实现是通过在 Condition 对象 trip 上 wait来
    // 完成的。
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

       int index = --count;
       // index == 0 是 CyclicBarrier  的 barrier point。
	   // 此时:
	   // 0. barrierCommand 将会执行
	   // 1. CyclicBarrier 的状态将被 nextGeneration 重置
	   // 2. nextGeneration 会唤醒 其它 wait 的 party 线程
       if (index == 0) {  // tripped
           boolean ranAction = false;
           try {
               final Runnable command = barrierCommand;
               if (command != null)
                   command.run();
               ranAction = true;
               nextGeneration();
               return 0;
           } finally {
               if (!ranAction)
                   breakBarrier();
           }
       }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                	// 当前线程如果被中断，其它 party 线程
                	// 会被唤醒，由于 breakBarrier 中将 
                	// g.broken 置为 true，所以其它 party 线程
                	// 醒来之后，就会抛出 BrokenBarrierException
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
            // 当前线程如果等待超时，则当前线程会抛出
            // TimeoutException,而其它线程则由于 breakBarrier
            // 原因，被唤醒，并抛出 BrokenBarrierException
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

## 参考

[CyclicBarrier的用法:](http://www.cnblogs.com/liuling/p/2013-8-21-01.html)

CyclicBarrier是一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 CyclicBarrier 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 barrier。

CyclicBarrier类似于CountDownLatch也是个计数器， 不同的是CyclicBarrier数的是调用了CyclicBarrier.await()进入等待的线程数， 当线程数达到了CyclicBarrier初始时规定的数目时，所有进入等待状态的线程被唤醒并继续。 CyclicBarrier就象它名字的意思一样，可看成是个障碍， **所有的线程必须到齐后才能一起通过这个障碍。** CyclicBarrier初始时还可带一个Runnable的参数，此Runnable任务在CyclicBarrier的数目达到后，所有其它线程被唤醒前被执行。

[CyclicBarrier的应用场景:](http://ifeve.com/concurrency-cyclicbarrier/)

CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。


[Java并发编程：CountDownLatch、CyclicBarrier和Semaphore](http://www.cnblogs.com/dolphin0520/p/3920397.html)