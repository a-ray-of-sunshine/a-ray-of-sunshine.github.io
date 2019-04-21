---
title: 线程池-ThreadPoolExecutor
date: 2016-8-11 17:56:52
tags: [ThreadPoolExecutor]
---

## 文档简介

ThreadPoolExecutor 是 ExecutorService的一个实现，其功能是：使用几个被池化的线程来执行提交的任务，这些在池中的线程由 ThreadFactory 来创建。

线程池解决了两个问题：

* 提高了执行大量异步任务的效率。
	
	使用缓存的线程来，执行任务，节省的线程创建的时间。

* 提供了一种限制和管理资源的方法。

此外，ThreadPoolExecutor 还维护了一些基本统计信息，例如已经完成的任务（task）的个数。

为了使这个线程池可以在不同的场景下使用，这个类提供了许多可以配置的参数和一些回调函数。然而，比较推荐的是使用 Executors 类来创建 ThreadPoolExecutor 对象，这个类提供了一些默认的参数，来实现不同功能的线程池。如果 Executors 类提供的线程池，不能够满足需求，可以按照下面的 ThreadPoolExecutor 的参数配置说明，来配置和微调线程池。

* Core and maximum pool size

	ThreadPoolExecutor 将使用参数 corePoolSize 和maximumPoolSize 来自动地调整 pool size (在池中的线程数量)。当使用 execute 方法提交一个新的任务的时候: (1) 如果当前运行的线程数量少于 corePoolSize ，则ThreadPoolExecutor将创建一个新线程来运行这个任务，尽管此时可能有其它线程处于空闲状态（idle）.(2) 如果当前运行的线程数量是介于 corePoolSize 和 maximumPoolSize 之间，当且仅当任务队列是满的时候（也就是这个任务无法入队列），才会创建一个新线程去运行这个任务。(3) 当前运行的线程数量已经是 maximumPoolSize，且队列也是不满，则入队列。(4) 当前运行的线程数量已经是 maximumPoolSize，且队列也是已经满了，则需要reject这个任务。
	
	当将 corePoolSize 和 maximumPoolSize 设置相同的时候，此时就是一个 fixed-size（固定大小） 的线程池。
	
	当将 maximumPoolSize 设置成 Integer.MAX_VALUE的时候，表示这个线程池可以接受任意数量的并发的任务。
	
	通常情况，使用 ThreadPoolExecutor 的构造函数配置这两个参数。当然也可以使用 setCorePoolSize 和 setMaximumPoolSize 方法动态的改变这两个参数。
	
* On-demand construction

	默认情况下池中所有线程都是任务被提交之后按需要创建的。但是，我们可以使用 prestartCoreThread() 和 prestartAllCoreThreads().方法来预先启动 coreThread。因为有一种情况，可能需要这样做。就是，ThreadPoolExecutor在创建的时候需要指定任务的存储队列，如果这个队列中已经有一些任务存在了，就可以使用上面的方法，通过预先启动线程，使得 BlockingQueue 中的任务可以立即被执行，而不是一直等待到有任务被提交时，才有线程可以执行这些任务。

* Creating new threads

	使用 ThreadFactory 来创建新的线程。如果没有指定这个参数，则默认使用 Executors.DefaultThreadFactory 这个实现类。这个类将所有的线程放置到同一个线程组中，其线程的命名规则是：pool-1-thread-1,pool-1-thread-2,pool-1-thread-3,etc. 其中第一个数字是线程池实例的编号，第二个数字是当前线程池中线程的编号。同时将线程的优先级设置在 NORM_PRIORITY 和 non-daemon 状态。

	通过，实现 ThreadFactory 接口，可以对线程池中线程的 name, thread group, priority, daemon status 进行设置。
	
	如果这个实现类的 newThread 方法返回null, 则当前的 executor 将继续执行，但是有可能不会执行任何任务。
	
* Keep-alive times

	如果线程池中当前线程的个数超过了 corePoolSize，并且这些超出的线程处于空闲状态的时间已经超过了 keepAliveTime, 则这些线程将被终止。当池变得不够活跃的时候，通过回收这些线程，节省了线程的资源的占用。
	
	这个参数也可以通过 setKeepAliveTime 方法来动态改变。
	
	默认情况下，超时策略仅适用于，超过 corePoolSize 的那些线程。但是如果使用 allowCoreThreadTimeOut(true) 调用，则可以设置，让 core threads 也可以在超时的时候被回收。
	
* Queuing

	executor 使用 BlockingQueue 来存储提交的任务。通常有三种策略：(1) Direct handoffs(直接切换): SynchronousQueue. (2) Unbounded queues(无界队列): LinkedBlockingQueue. (3) Bounded queues(有界队列): ArrayBlockingQueue.
	
* Rejected tasks

	当 executor 被 shutdown 或者 maximum threads 和 queue capacity 都是有界的，而且，都已经满了的时候。executor 的 execute 方法 将 调用 RejectedExecutionHandler.rejectedExecution 方法，来 reject 一个任务。
	
	预定义的 handler 有：(1) ThreadPoolExecutor.AbortPolicy: 直接抛出 RejectedExecutionException 异常。(2) ThreadPoolExecutor.CallerRunsPolicy: 被 reject 的任务将直接在 调用者 线程中被执行。它提供了一种简单的反馈控制机制，从而减缓了新任务提交的速率。其实就是新提交的任务被当前提交任务的线程执行，使用提交过程被减缓。(3) ThreadPoolExecutor.DiscardPolicy: 将这个不能提交的任务直接丢弃（不执行，不抛异常，do nothing）。(4) ThreadPoolExecutor.DiscardOldestPolicy: 将当前队列中等待最久的线程移除，然后再次提交这个任务。
	
	It is possible to define and use other kinds of RejectedExecutionHandler classes. Doing so requires some care especially when policies are designed to work only under particular capacity or queuing policies. 

* Hook methods	

	提供了两个方法 beforeExecute 和 afterExecute 在每一个任务被执行的时候，这两个方法都会被调用。这些方法实现的通常目地可以是：调整任务的执行环境，例如：重新初始化 ThreadLocal, 收集统计信息, 添加任务执行日志等等。
	
	此外，可以 override terminated方法来执行当 executor 被终止的时候，执行一些特殊的资源清理操作等。
	
* Queue maintenance 

	getQueue 允许其它线程出于高度或者监控目地。如果将这个方法使用于其它目地是强烈不推荐的。
	
	如果需要调用 queue， 则应该使用 executor 提供的 remove 和 purge 方法，而不是直接操作 queue.
	
* Finalization

	A pool that is no longer referenced in a program AND has no remaining threads will be shutdown automatically.
	
    当池对象不再被任何线程引用并且池中没有剩余的线程，则这个线程将会自动被关闭。

## 状态

### ctl: 线程池的生命周期控制字段 
``` java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

这个变量分为两个部分：

* workerCount: 后 29 个字节。表示当前线程池中有效的线程数。
* runState:        前 3   个字节。表示当前线程池的运行状态。

由初始化的参数可知，当线程池初始化之后的，默认状态是：RUNNING 表示当前线程池可以接受新的任务，并且可以处理queue中的任务。同时0表示线程池中当前的线程个数为0.

关于 ctl 的操作：
``` java
 // Packing and unpacking ctl
// 从 ctl 中取出 runState。
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 从 ctl 中取出 workerCount。
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 将 runState 和 wc 打包成 ctl 变量。
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### workQueue: 任务队列
``` java
private final BlockingQueue<Runnable> workQueue;
```
这个队列中用来持有（存储）任务和分发任务给 工作者线程（worker threads）。这个队列中使用过程中，并不会以 workQueue.poll 方法返回 null ，而认为队列是空的。而是使用 workQueue.isEmpty 方法来判断当前队列是否为空。

### workers: 工作者线程
``` java
private final HashSet<Worker> workers = new HashSet<Worker>();
```
存储池中的所有线程。Set containing all worker threads in pool.

## Worker 内部类
``` java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
```
表示executor中内部线程池中的线程。

## 提交任务的过程
``` java
// 提交任务
public void execute(Runnable command) {
    // 任务为 null, 直接抛异常
    if (command == null)
        throw new NullPointerException();
            
    // 开始尝试提交
    int c = ctl.get();
    // 1. 如果当前线程池中的线程数量 < corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 开始将任务 command 直接进行添加
        // 添加成功，直接返回
        if (addWorker(command, true))
            return;
        // 添加失败，有可能是存在其它线程也在使用当前的线程池
        // 所以，重新获取 ctl。
        c = ctl.get();
    }
    
    // 代码执行到这里，表明此时线程池中线程的数量 >= corePoolSize 了。
    // 也就是当前线程数量已经到了 corePoolSize了，
    // 所以后面的 addWorker 中的第二个参数 都是 false,
    //  表明使用 maximumPoolSize 作为线程池数量的边界值。
    
    // 2. 如果当前线程池是 RUNNING 状态(isRunning(c))，
    // 则将当前任务 command 入队列(workQueue.offer(command))
    if (isRunning(c) && workQueue.offer(command)) {
        
        // 需要 recheck 的原因是，当前线程池可能被多个线程并发使用，
        // 所以必须重新获取 ctl.
        int recheck = ctl.get();
        
        // 3. 如果当前线程池已经不在运行状态了，
        // 并且将command从队列中移除成功。
        // 则 reject 当前任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 4. (1) 线程池已经不处于 RUNNING 状态了， 但是 任务仍处于 队列中
        // 并且，当前线程池中的线程中没有任何线程了，则添加一个新的工作者线程
        // 这个工作者线程会执行会将队列中的任务取出来执行。
        // 因为之前已经将 command 入队列了，所以下面的
        // addWorker 中第一个参数是 null
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 5. 线程不处于运行状态，且这个任务还没有入队列，
    //     将这个任务添加到一个工作者线程中去执行
    //     线程没有入队列，所以将 command 传入
    //     addWorker 参数，用新创建的线程来执行这个任务。
    else if (!addWorker(command, false))
         // 6. 上面的添加失败，则 reject 这个任务。
        reject(command);
}
```

## addWorker
``` java
private boolean addWorker(Runnable firstTask, boolean core) {

   // 检验和调整线程池的状态
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 检测 线程池的运行状态
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        // 到这里表明当前线程池可以接受任务
        for (;;) {
            int wc = workerCountOf(c);
            // 判断线程池线程的数量是否超过配置的数量，如果超过了，则直接返回 false.
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
                
            // 代码执行到此处，表明可以增加新的 worker, 
            // 所以尝试增加 workerCount 计数，
            // 由于可以并发执行，所以使用 CAS 循环来保证线程安全。
            if (compareAndIncrementWorkerCount(c))
                break retry;
                
            // 在并发循环的过程中，有可能线程池的状态发生变化了，
            // 所以，重新获取 ctl 的值，如果 运行状态 确实发生的变化
            // 就重置整个循环，再次尝试。
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 代码执行到这里，表明可以创建新的worker(线程)了，
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock;
        // 1. 创建新的 work
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 使用 mainLock 来保护 变量 workers, 
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();
                int rs = runStateOf(c);
                
                // 当线程获得锁的时候，有可能，线程池
                // 状态已经发生变化了，所以重新检测线程池的状态。
                // 线程池处于 RUNNING 状态，
                // 或者虽然线程池已经 SHUTDOWN，但是
                // firstTask == null，也可以启动线程。
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    
                    // 线程是否已经被启动，如果被启动，
                    // 则直接抛异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                        
                    // 将当前线程添加到 workers
                    workers.add(w);
                    
                    // 使用 largestPoolSize 来跟踪当前线程池，
                    // 所能够达到的最大的线程个数。这个字段，应该是线程池的运行
                    // 状态统计信息。
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                        
                    // 表明线程添加成功
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            
            // 线程添加成功，所以启动这个线程。
            if (workerAdded) {
                t.start();
                // 表明线程启动成功
                workerStarted = true;
            }
        }
    } finally {
        // 如果线程启动失败，则调用 addWorkerFailed 方法来调整
        // 线程池的状态：（1）从 workers 移除 w, (2) 减少 workCount 字段的计数
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## runWorker
上面的 addWorker 如果添加 worker 成功，则启动 worker 线程。
``` java
// worker 的 run 方法
/** Delegates main run loop to outer runWorker  */
public void run() {
    runWorker(this);
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
    
        // 工作者线程的主循环，不断的运行任务，
        // 任务的来源有： 
        //       (1) 如果这个线程是第一次运行，则一般会有一个任务和其关联，就是 w.firstTask, 
        //       (2) 当 firstTask 运行完毕之后，就从 BlockingQueue 中获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
                
            // 获取 task 之后，开始执行 task.
            try {
                // 每一个任务在开始执行前的回调接口
                // 子类可以 override 这个方法，来调整 task 的执行环境，进行统计，日志分析
                // 调试等工作
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 运行任务.
                    task.run();
                    
                    // 如果任务运行过程中出现异常,
                    // 将导致,worker线程运行 throw 异常,而退出.
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 每一个任务在执行结束之后的回调接口
                    // 子类可以 override 这个方法，来调整 task 的执行环境，进行统计，日志分析
                    // 调试等工作,通过第二个参数 thrown == null,来判断任务是否执行成功,或是
                    // 执行过程中抛出异常了.
                    afterExecute(task, thrown);
                }
            } finally {
                // 一个任务执行完毕之后 
                // 设置 task 为 null,
                task = null;
                // 工作者线程的执行 task 的个数统计.
                w.completedTasks++;
                w.unlock();
            }
        }
        
        // 如果线程没有在执行 task 的过程中抛出异常
        // 也就是说线程是正常从上面的 main loop 退出的,
        // 则 设置 completedAbruptly 为 false.
        completedAbruptly = false;
    } finally {
        // worker 退出了, 调用processWorkerExit调整线程池的状态.
        processWorkerExit(w, completedAbruptly);
    }
}
```

在线程退出之后会调用 processWorkerExit 方法，这个方法的第二个字段 completedAbruptly， 表示
线路退出的原因，如果是正常退出为 false, 如果线程异常退出为 ture, 在 processWorkerExit 方法
中将会将 workerCount - 1.


## 线程的生命周期

* RUNNING

    可以接受新的任务,同时可以处理队列中的任务.
    
* SHUTDOWN

    不接受新的任务,但是仍然会处理队列中的任务.
    
* STOP

    不接受新的任务,不处理队列中的任务了,将正在处理任务的线程中断.

* TIDYING

    所有的任务被 terminated, 并且 workerCount == 0,线程状态就会变成这个状态, terminated 回调函数被调用.    

* TERMINATED

    terminated 回调函数被调用完成之后, 线程池就处于这个状态.
    
生命周期的转换:
* RUNNING -> SHUTDOWN

    调用 shutdown 函数的时候, 同时 finalize 函数内部也调用了该函数.

* (RUNNING or SHUTDOWN) -> STOP

    调用 shutdownNow 函数的时候
    
* SHUTDOWN -> TIDYING

    当队列和线程池都为空时.

* STOP -> TIDYING

    当线程池为空时.    

* TIDYING -> TERMINATED

    When the terminated() hook method has completed

### 线程池的终止
``` java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||    // 当前线程池处于RUNNING状态
            runStateAtLeast(c, TIDYING) ||     // 当前线程池已经是 TIDYING 或 TERMINATED 状态了 
            // 线程处于 SHUTDOWN, 但是队列中仍然有任务存在
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
            
        // 执行到这里表示,线程可能处于 STOP 状态,
        // 或者是 SHUTDOWN 状态 且 workQueue 中 没有线程了.
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        // 调整线程池的状态: ctl ==> TIDYING ==> TERMINATED
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

## getTask 
在 runWorker 的 main leep 中获取任务
``` java
// 这个函数有三个出口， 查看  runWorker 函数中使用 getTask ,
// 当 getTask 这个函数返回 null 时，worker 主线程会退出主循环
// worker 线程就会退出。
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 响应线程池的生命周期
        // Check if queue empty only if necessary.
        // 下面的判断条件相当于：
        // if( (rs >= SHUTDOWN && workQueue.isEmpty()) || rs >= STOP )
        // 就是 线程池状态处于 STOP 状态，
        // 或者 线程池状态处于 SHUTDOWN 状态 并且 workQueue 中 没有任务了。
        // 在这两种状态下 worker 线程将会退出。
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
			// 当前线程要退出了，所以减少 workerCount 的计数。
            decrementWorkerCount();
            return null;
        }

        // 是否允许当前线程在当前线程池的状态下可以超时。
        boolean timed;      // Are workers subject to culling?

        for (;;) {
            // 线程是否超时判断。从这里可以看出，其实一个线程是否是
			// core 线程，还是非 core 线程，是没有固定的，也就是
			// 在线程池中的所有的worker线程，是依据当前线程池的状态
			// 来判断是超时，如果不可以超时，则认为它是 core 线程
			// 到第二次获取任务的时候，有可能变成 可以超时的，所以
			// 它变成了 非core 线程了。 
            int wc = workerCountOf(c);
            timed = allowCoreThreadTimeOut || wc > corePoolSize;
            
            if (wc <= maximumPoolSize && ! (timedOut && timed))
                break;
            // 1. 当前线程数量大于 maximumPoolSize
            // 2. 如果当前线程上次超时了，并且这次也允许超时
            // 这两种情况下，当前线程就应该从线程池中退出了。
			// 当前线程要退出了，所以减少 workerCount 的计数。
            if (compareAndDecrementWorkerCount(c))
                return null;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }

        try {
            // 如果允许线程超时，使用 poll 来获取任务，否则使用 take 来阻塞获取任务。
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
                
            // 这个设计非常好，这里并不会直接返回r，因为有可能，workQueue有可能超时
            // 返回 null, 所以如果直接返回 r 的时候，会使得当前线程直接退出，这会有问题，
            // 因为在上面的 poll 过程的 block 过程中有可能，线程池中的其它线程有可能已经
            // 退出，如果直接使得当前线程返回 null, 就有可能，导致线程池中的线程数量 小于
            // corePoolSize , 显然这不符合线程池对 corePoolSize 的语义。
            // 所以，这里使用 timedOut 字段标识，线程上次超了，然后，继续循环，
            // 在上面的代码中，会使用timedOut 状态判。然后作对应的处理。
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## Executors 提供的几种线程池

### newFixedThreadPool
``` java
public static ExecutorService newFixedThreadPool(int nThreads) {
	return new ThreadPoolExecutor(nThreads, nThreads,
                      0L, TimeUnit.MILLISECONDS,
                      new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

这类线程池将创建线程数量最多为 nThreads 个线程，由于个这个原因， getTask 方法从队列中获取任务的方法将总是：

	workQueue.take();

所以，对于超时的配置，其实是用不到的，所以这里 keepAliveTime 参数的值是 0, 其实根本用不到。

这 nThreads 个线程将一直运行来执行任务。

#### 线程创建执行机制：
* 前 nThreads 个任务的提交，将使当前线程中创建好 nThreads 个线程
* 后续提交的任务，将依时间顺序入队列，
* 然后由这 nThreads 个线程不继的循环消费。

#### 线程超时
**这个 nThreads 线程永远不会超时。**

### newSingleThreadExecutor
``` java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

这个线程池内只有一个线程，当向这个线程池提交第一个任务时，创建这个线程。
提交第二个任务的时候，会直接将该任务，添加到 workerQueue 队列中。线程中的惟一那个
线路，开始从 workerQueue 中按添加顺序不断的执行任务。

同样，由于 corePoolSize == maximumPoolSize == 1, getTask 获取任务的方法是：
	
	workQueue.take();
	
所以其超时参数没有使用到。

#### 线程创建执行机制：
* 第一个任务的提交会创建这个线程池中的惟一个线程，然后任务被执行，
* 后续提交的任务，将依时间顺序直接入队列
* 线程池中的惟一线程将从队列中不断循环地取出任务，然后执行

#### 线程超时
**这一个线程永远不会超时。**

### newCachedThreadPool
``` java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

由于 corePoolSize = 0， maximumPoolSize = Integer.MAX_VALUE， 所以，
这个线程中的行为是，添加一个任务就会，创建一个对应的线程，当这个线程就执行
完毕，就等待 60 秒，如果这 60 秒内有其它任务被提交，则这个提交的任务将被这个线程
执行，否则，线程在 60 秒之后被回收。直接新的 任务被提交，然后重复上面的过程。

	workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)

由传入的参数，可知，在这个线程池中提交的任务执行完毕之后将等待 60 秒，然后退出。

#### 线程创建执行机制：
* **如果第二个任务提交的时候，前一个任务还在执行，则这个任务会入队列失败，然后创建一个新的线程来执行**
* **如果第二个任务提交的时候，前一个任务已经处于 poll 状态了，则这个任务将交由执行前一个任务的那个线程来执行，而不是创建新的线程**

#### 线程超时
**这个线路池中的每一个线程都是 60s 的超时时间，过了 60s 这个线程就会被回收。**

## 关于 workerCount 状态的维护
### addWorker
在每次调用 addWorker 的时候，如果一个worker(线程)可以被创建，则其在创建之后必然会调用 compareAndIncrementWorkerCount 方法来，增加 workCount ，表示线程池中增加了一个新的线程。

### getTask
线程通过轮询的方式来不断的执行任务。每次获取任务之前会判断是否可以线程执行。如果不行，则调用 compareAndDecrementWorkerCount， 表示当前线程将从线程池中被移除。

但是，存在一个问题，在 runWorker 的 main loop 中，用户代码有可能抛出异常，例如：run, beforeExecute, afterExecute 都有可能抛出异常。一旦出现异常，当前线程就会被 JVM 终止。而此时由于没有来得及调用循环中的 getTask 方法，所以，这个线程虽然已经异常终止了，但是在 workerCount 的计数中还占一位，所以必须，在处于 worker 退出的地方，进行判断，如果是异常终止，则调用 decrementWorkerCount 来清除这个异常终止的线程所占用的workerCount位。这也就是下面代码片断的原因：
``` java
// 当线程是异常终止的，completedAbruptly 就是 true.
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
```

## 参考
[Java线程池使用说明](http://www.oschina.net/question/565065_86540)