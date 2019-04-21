---
title: 线程池-ThreadPoolExecutor-ScheduledExecutorService
date: 2016-8-15 11:51:38
tags: [ScheduledExecutorService, ThreadPoolExecutor]
---

## 文档

``` java
public interface ScheduledExecutorService extends ExecutorService
```

这个接口继承自 ExecutorService, 主要作用是提供 延时调度 和 周期性调度。

* schedule

	提供延时调度功能，并且返回一个 ScheduledFuture 对象，这个对象提供取消或检查任务的执行。
	
* scheduleAtFixedRate

	调度机制：
	1. 第一次延时 initialDelay 调用。
	2. 之后以 initialDelay + (n - 1) * period 的周期来循环调用。n 表示当前执行的次数。例如第二次时调用的间隔是： initialDelay + (2 - 1) * period，第三次调用的间隔是： initialDelay + (3 - 1) * period

* scheduleWithFixedDelay
	
	调度机制：
	1. 第一次延时 initialDelay 调用。
	2. 之后以 period 的周期来循环调用。
	
注意，上面的 schedule 方法，延时参数，可以是 0 或者是负值，表示没有延时，立即执行。但是对于，	调用周期 period 这个参数，则不能 <= 0, 如果 <= 0, 则IllegalArgumentException。

**注意：这类方法接受参数 delay 和 period 都是相对时间，而不是直接使用绝对的日期或时间。**
	
也可以使用该类的 execute 和 submit 方法，这些方法调用，默认使用的延时为 0，也就是立即执行。

``` java
public void execute(Runnable command) {
    schedule(command, 0, TimeUnit.NANOSECONDS);
}
```

## ScheduledThreadPoolExecutor	
``` java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService
        

// 构造函数
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
          new DelayedWorkQueue());
}
```

构造函数中使用的 BlockingQueue 是 DelayedWorkQueue 这个类是 ScheduledThreadPoolExecutor 的内部类。

### schedule 任务提交的过程。

``` java
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
        
    // decorateTask 方法直接返回
    //  new ScheduledFutureTask<Void>(command, 
    //      null,triggerTime(delay, unit)
    // 这个任务。和 sumbit 方法类似，schedule对提交
    // 的任务（command）进行了包装，返回一个
    // RunnableFuture 对象。
    RunnableScheduledFuture<?> t = decorateTask(command,
        new ScheduledFutureTask<Void>(command, null,
                                      triggerTime(delay, unit)));
                                      
	// 
    delayedExecute(t);
    return t;
}

// 1. 处理延迟时间
// triggerTime 将时间转换为 纳秒（ns）
// 然后调用内部的 triggerTime， 将这个相对时间
// 转换为绝对时间： now() + delay. 这样，这个值
// 将表示 当前这个任务触发（被调度）的时间。
private long triggerTime(long delay, TimeUnit unit) {
    return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
}
long triggerTime(long delay) {
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}
```

// 2. 将任务添加到任务队列中。
``` java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
    	// 1. 任务直接入队列。
        super.getQueue().add(task);
        // 处理线程池关闭的情况。
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
        // 这个调用非常重要。由于 task 直接入队列了
        // 所以 schedul 必须保证，能够启动 worker
        // 所以这里直接使用下面的方法来启动 worker.
        // 这些 worker 将作为消费者，来处理 
        // BlockingQueue 中的任务。
            ensurePrestart();
    }
}
```

#### ScheduledFutureTask

这个类对 task 进行包装。这上方法的核心是 run 方法，其实现了，周期性调度的功能。
``` java
public void run() {
	
	// 判断任务是否是周期性的。
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    // 如果任务不是周期性的，则直接调用，
    // 然后直接返回
    else if (!periodic)
        ScheduledFutureTask.super.run();
    // 如果任务是周期性的则，调用 runAndReset 方法
    // 这个方法执行完 task 之后会将当前 Future 的
    // state 恢复成 NEW 状态。使得这个 task 可以
    // 被再次调用。
    else if (ScheduledFutureTask.super.runAndReset()) {
    	// 如果上面调用成功：
    	// 1. 计算任务下次被调用的时间。
        setNextRunTime();
        // 2. 因为当前任务已经被调用过一次了
        //    所以需要将任务，重新添加到队列中。
        reExecutePeriodic(outerTask);
    }
}
```

## 任务的获取

任务获取是从 DelayedWorkQueue 这个队列中。使用 take 方法。take 方法的核心部分。
``` java
for (;;) {
    RunnableScheduledFuture first = queue[0];
    if (first == null)
        available.await();
    else {
        long delay = first.getDelay(TimeUnit.NANOSECONDS);
        if (delay <= 0)
            return finishPoll(first);
        else if (leader != null)
            available.await();
        else {
            Thread thisThread = Thread.currentThread();
            leader = thisThread;
            try {
                available.awaitNanos(delay);
            } finally {
                if (leader == thisThread)
                    leader = null;
            }
        }
    }
}
```

## DelayedWorkQueue

这是一个基于二叉堆的队列结构。同时这也是个最小堆，堆顶元素是当前队列中 delay time 最小的元素。正因为如此，队列的 take 操作将在堆顶进行，也就是 queue[0] 元素上进行，这个任务将是第一个需要出队列的任务。

### 入队列操作

入队列是无阻塞的操作。

```
public boolean offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture e = (RunnableScheduledFuture)x;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0);
        } else {
			// 入队列，并将最开始执行的任务，调整到队列头部。
            siftUp(i, e);
        }
		// queue[0] == e 表示 e 是队列中当前惟一一个元素
		// 在此之前，有可能 worker 线程已经在 进行 take 操作了，
		// 而由于队列中没有元素，所以 worker 线程会进入等待状态
		// available.await();
		// 所以，当队列中的第一个元素入队列成功之后，调用
		// available.signal() 来唤醒 worker 线程。
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

### 出队列操作

这个一个阻塞操作，并且也是线程安全的。线程池中的多个 worker 线程，可能同时调用 take 方法来获取任务。此时由于锁 lock 的保护，当且仅当 只有一个线程会获得锁。

``` java
public RunnableScheduledFuture take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
			// 队头元素 queue[0], 就是要执行的任务
            RunnableScheduledFuture first = queue[0];
			// 任务为null, 说明此时还没有任务被提交，所以 wait.
            if (first == null)
                available.await();
            else {
				// 任务不为空，则获取当前时间 到 任务开始执行的时间（getDelay）
				// 的时间间隔 delay
                long delay = first.getDelay(TimeUnit.NANOSECONDS);
				// 如果 delay <= 0 表示，任务已经可以运行了，所以调用
				// finishPoll(first) 方法将 first 出队列，获取到 first
				// 的 worker 线程将开始执行任务。
                if (delay <= 0)
                    return finishPoll(first);
                else if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
						// 开始进入等待状态，等待的时间就是 delay
						// 从逻辑上看，这个等待返回之后，delay 将
						// <= 0，所以 first 任务可以被调度了
						// 所以这个循环将在上面的 return finishPoll(first);
						// 中退出。
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

#### take 方法中 leader 的作用

在上面的 take 方法的实现过程中还维护了一个 leader 字段，

```
private Thread leader = null;
```

这个字段的作用应该
是，协调在 first 任务上等待的多个 worker 线程。
ScheduledThreadPoolExecutor.DelayedWorkQueue.leader
中有关于这个字段的注释。

Thread designated to wait for the task at the head of the queue.  This variant of the Leader-Follower pattern serves to minimize unnecessary timed waiting.

作用：最小化 worker 线程的等待时间。

take 过程：假设，有 T1, T2, T3 三个 worker 线程，在竞争获取 queue[0] 任务。

* 当 T1 成功执行到 take 的循环中的最后一个 else 中的  available.awaitNanos(delay); 线程时， leader 就会被设置成 T1。
* 此时处于 wait 过程的线程 T1, 将放弃锁 lock, 所以 T2, T3 又开始竞争锁
假设 T2 获得锁，进行循环执行到 else if (leader != null) ，显然 leader 已经是 T1 了，所以这个判断成立，所以 T2 会进入  available.await();
* 此时 T1 因为 available.awaitNanos(delay) delay 时间已过，所以被唤醒，重新获取锁之后，将 leader 置为 null, 在 T1 返回之前，执行最后的 finally 语句，将通知 像 T2 这样的线程（处于 available.await()），使得 T2 可以立即参与到下一次获取 queue[0] 的过程中。
* 此时，T1 获取了任务，正在执行，T2 被 T1 唤醒，T2 和 T3 又开始了第一个步骤中的过程一样进行任务获取了。

通过这三个 worker 线程获取任务的过程可知：leader 的使用，使得 worker 线程的 wait 最小化，尽量使得 worker 能够参与到任务的获取和执行中来。

试想如果不使用 leader， 则 T1, T2, T3 线程最终都进入  available.awaitNanos(delay); 的过程，而对于一个任务来说，多个线程在其上等待是没有意义的，因为最终只需要一个线程来执行任务，而不是所有在其上 wait 的线程，所以 leader 其实也可以认为是当前queue[0]最终可以被获成功的那个线程。显然应该只有一个，所以其它线程应该是直接available.await(); 而不是  available.awaitNanos(delay);

**注意：上面的描述分析完全要结合ScheduledThreadPoolExecutor.DelayedWorkQueue.take 的源代码来看**

## 周期性调度方法的实现

* scheduleAtFixedRate

	表示固定的调用周期，其对应的 task 是： 

		new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period))

* scheduleWithFixedDelay

	表示固定的调用频率，其对应的 task 是：
		
		 new ScheduledFutureTask<Void>(command,
                                       null,
                                       triggerTime(initialDelay, unit),
                                       unit.toNanos(-delay))

注意到通过这两个方法提交的任务惟一不同的就是第四个参数 period. 
scheduleWithFixedDelay 调用中将 period 设置成 -delay.

ScheduledThreadPoolExecutor.ScheduledFutureTask.period 字段的定义：

``` java
/**
 * Period in nanoseconds for repeating tasks.  A positive
 * value indicates fixed-rate execution.  A negative value
 * indicates fixed-delay execution.  A value of 0 indicates a
 * non-repeating task.
 */
private final long period;
```
由此：
* period < 0 <==> fixed-delay execution 固定延时任务
* period = 0 <==> non-repeating task    非周期性任务
* period > 0 <==> fixed-rate execution  固定频率任务

对于周期性任务，在前面的分析可知，当该任务成功执行完毕之后，将会重新添加到 queue 中，此时就要重新设置 time ,也就是下次执行的时间，使用方法 setNextRunTime：
``` java
private void setNextRunTime() {
    long p = period;
    if (p > 0) // 固定频率，所以直接在上次执行的time上加 period.
        time += p; // time = time + period;
    else  // 固定延时，表示以后任务开始执行的时间，都是当前时间 加 period
        time = triggerTime(-p); // time = now() + period
}
```


## 定时调度的设计实现

ScheduledThreadPoolExecutor 线程池实现分析。

所谓定时调度就是：延迟调度和周期性循环调度。其实现过程分析如下：

不管是延迟调度，还是周期性调度，所有这类 任务(task) 最终都有其所对应的开始执行时间(enbaled time)，这个开始时间。ScheduledFutureTask 是这类任务的一个具体实现。其中有一个 time 字段，这个字段就表示，任务可以被调度的开始时间。

``` java
private class ScheduledFutureTask<V>
        extends FutureTask<V> implements RunnableScheduledFuture<V> {

    /** Sequence number to break ties FIFO */
    private final long sequenceNumber;

    /** The time the task is enabled to execute in nanoTime units */
    private long time;
```

所以对于每一个 任务(ScheduledFutureTask) 都有对应的开始时间。并且任务只有在这个时间点（time） 才可以运行。所以线程池的任务的执行，自然地就有了顺序，就是按照这个开始时间。所以 ScheduledThreadPoolExecutor 就应该将所有提交的任务进行排队（按开始执行时间排序），然后 worker 线程从队列中取出，时间最近的任务开始执行。这样就需要任务是可以排序的，所以 ScheduledFutureTask 实现了  java.lang.Comparable 接口。用任务的开始执行时间来确定任务的优先级。


compareTo 方法实现的基本思路就是：使用 开始执行时间 time 和 任务提交时的序号 sequenceNumber（体现任务的提交时间先后顺序），开确定任务的大小。
``` java
public int compareTo(Delayed other) {
    if (other == this) // compare zero ONLY if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long d = (getDelay(TimeUnit.NANOSECONDS) -
              other.getDelay(TimeUnit.NANOSECONDS));
    return (d == 0) ? 0 : ((d < 0) ? -1 : 1);
}
```

这个方法将被用在 任务（ScheduledFutureTask） 入队列的时候，也就是 DelayedWorkQueue 这个队列的 入队列操作（offer） 中。offer方法将以
此来确定堆顶元素。从而保证堆顶的元素是当前所有任务要最近执行的任务。