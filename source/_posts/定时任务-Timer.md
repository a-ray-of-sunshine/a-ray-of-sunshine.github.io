---
title: 定时任务-Timer
date: 2016-8-20 20:28:09
tags: [Timer, TimerTask]
---

## 核心构造

### 任务队列
``` java
private final TaskQueue queue = new TaskQueue();
```
TaskQueue类，定时任务队列，一个存储 TimerTask 对象的优先队列。其中的 TimerTask 使用其 nextExecutionTime 来排序。对于每一个 Timer 对象都有一个这种类型的对象。这个对象被 TimerThread 所共享。

``` java
class TaskQueue {
    /**
     * Priority queue represented as a balanced binary heap: the two children
     * of queue[n] are queue[2*n] and queue[2*n+1].  The priority queue is
     * ordered on the nextExecutionTime field: The TimerTask with the lowest
     * nextExecutionTime is in queue[1] (assuming the queue is nonempty).  For
     * each node n in the heap, and each descendant of n, d,
     * n.nextExecutionTime <= d.nextExecutionTime.
     */
    private TimerTask[] queue = new TimerTask[128];
```

TaskQueue 内部维护一个 TimerTask 数组，这个数组是一个平衡二叉堆，对于元素 queue[n] 其有两个子结点 queue[2*n] 和 queue[2*n+1]。这个二叉堆使用 nextExecutionTime 来排序。其中 nextExecutionTime 最小的元素在堆顶：queue[1]。这个元素将成为Timer调度的最近的一个元素。


### 定时器线程
``` java
private final TimerThread thread = new TimerThread(queue);

class TimerThread extends Thread 
```

这个对象持有上面的任务队列，表示定时器线程，用来执行 TaskQueue 中的任务
调度算法：
``` java
private void mainLoop() {
    while (true) {
        try {
            TimerTask task;
            boolean taskFired;
            synchronized(queue) {
                // Wait for queue to become non-empty
				// newTasksMayBeScheduled 表示定时器线程的生命周期
				// 初始状态为 ture, 如果队列为空则在 queue 上wait.
				// 那么这个等待何时被唤醒呢？提交任务的时候，会被唤醒
				// 
                while (queue.isEmpty() && newTasksMayBeScheduled)
                    queue.wait();
				// 表示 newTasksMayBeScheduled = false
				// 定时器被取消的时候（Timer.cancel) newTasksMayBeScheduled 
				// 被置为 false, 此时这个线程也要终止了，所以直接 break.
                if (queue.isEmpty())
                    break; // Queue is empty and will forever remain; die

                // Queue nonempty; look at first evt and do the right thing
                long currentTime, executionTime;
				// 获得第一个被执行的任务。
                task = queue.getMin();
                synchronized(task.lock) {
					// 任务有可能被取消了（TimerTask.cancel()），
					// 移除任务。重新开始任务调度（continue）
                    if (task.state == TimerTask.CANCELLED) {
                        queue.removeMin();
                        continue;  // No action required, poll queue again
                    }

					// 当前时间
                    currentTime = System.currentTimeMillis();
					// 任务开始调度的时间
					executionTime = task.nextExecutionTime;

					// 如果 executionTime<=currentTime 表示，任务可以被
					// 调用了，taskFired = true
                    if (taskFired = (executionTime<=currentTime)) {

						// 非周期性任务，直接将其从调度队列中移除
						// 然后将 task 的状态置成 TimerTask.EXECUTED
                        if (task.period == 0) { // Non-repeating, remove
                            queue.removeMin();
                            task.state = TimerTask.EXECUTED;
                        } else { // Repeating task, reschedule
							// 任务需要被周期性调度。所以调整 task 的下次执行的时间
							// 然后将其在队列中重新调整位置，nextExecutionTime发生
							// 变化的所以 task 需要重新调整位置。
                            queue.rescheduleMin(
                              task.period<0 ? currentTime   - task.period
                                            : executionTime + task.period);
                        }
                    }
                }
				
				// 任务还没到触发的时机，所以调度线程进入 wait 状态。
                if (!taskFired) // Task hasn't yet fired; wait
                    queue.wait(executionTime - currentTime);
            }

			// 任务需要被调度了，直接执行 任务的 run 方法。
            if (taskFired)  // Task fired; run it, holding no locks
                task.run();
        } catch(InterruptedException e) {
        }
    }
}
```

* 定时任务线程对异常的处理

	可以看到，在 mainloop 中，只对 InterruptedException 进行了捕获，所以一旦task 出现其他类型的异常，这个 定时任务线程 将直接因为异常而退出。
	``` java
    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }
	```
	可以看到，退出时，会将newTasksMayBeScheduled置为 false, 这样 Timer 就不会再接受任务了，同时将 queue 中现有的任务全部清空。

* 定时任务的启动：

	创建定时器的时候，这个定时任务线程就会被启动。
``` java
public Timer(String name) {
    thread.setName(name);
    thread.start();
}
```

## 提交任务

``` java
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");

    // Constrain value of period sufficiently to prevent numeric
    // overflow while still being effectively infinitely large.
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");

		// 初始化任务
        synchronized(task.lock) {
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            task.nextExecutionTime = time;
            task.period = period;
            task.state = TimerTask.SCHEDULED;
        }
		
		// 将任务添加到 queue 中
        queue.add(task);

		// queue.getMin() == task，说明此时提交的任务是
		// queue 中延时最小的任务，则这个 task 的执行优先级，目前是最大的
		// 所以将 mainloop 中的 wait 唤醒，从而来执行这个 task. 否则有可能
		// 造成这个任务不能用时被调度.
        if (queue.getMin() == task)
            queue.notify();
    }
}
```

## Timer的终止
``` java
public void cancel() {
    synchronized(queue) {
		// 1. 将定时调度线程的状态置为 false.
        thread.newTasksMayBeScheduled = false;
		// 2. 将队列中所有的 task 直接移除。        
		queue.clear();
		// 3. 使得 mainloop 从 wait 中被唤醒
		//    由于 queue 为 空，且 newTasksMayBeScheduled 为 false
		//    mainloop 将从此退出。
        queue.notify();  // In case queue was already empty.
    }
}
```
由于 调度线程（thread） 执行结束，所有 timer 对象一但被 cancel， 就不能再次
提交任务了，如果还向一个已经被 cancel 的 timer 对象提交任务，则：
``` java
  if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");
```

## Timer 和 ScheduledThreadPoolExecutor

这两个类都提供了定时调度和周期性调度，Timer的实现无法对提交的任务进行管理，而 ScheduledThreadPoolExecutor 有 Future 。

这两个类提供的功能都是线程安全。

虽然 实现Timer 的基础组件：TaskQueue 和 TimerTask 并不是线程安全的。但是
Timer 实现的任务提交 Timer.sched() 方法在使用 queue 和 task 做了同步处理。

同样，定时调度线程 TimerThread 的 main loop 方法了是线程安全的。所以Timer类是线程
安全的。一个定时器对象可以在多个线程中被使用。

ScheduledThreadPoolExecutor which is a thread pool for repeatedly executing tasks at a given rate or delay. **It is effectively a more versatile replacement for the Timer/TimerTask combination, as it allows multiple service threads, accepts various time units, and doesn't require subclassing TimerTask (just implement Runnable).** Configuring ScheduledThreadPoolExecutor with one thread makes it equivalent to Timer. 

## Timer，ThreadPoolExecutor，ScheduledThreadPoolExecutor 对异常的处理

* Timer
	
	任务一个任务发生异常，导致 调度线程 退出，Timer将无效，无法接受任务了。同时对于提交任务的那个线程，将对其提交的任务发生异常一无所知。

* ThreadPoolExecutor

	任务发生异常同样会导致，worker 线程被终止，但是当再次向这个线程池提交任务的，线程池可以正常接受任务，并创建一个新的 worker 线程。异常信息的记录。可以通过 override 这个类的 afterExecute 方法，这个方法的第二个参数就是导致线程被终止的异常对象。当然这个对象可能为 null, 表示线程是正常终止的。这个方法是被执行任务的 worker 线程所调用的。

* ScheduledThreadPoolExecutor

	这个类的 schedule 方法返回一个 Future<?> 对象，这个对象的 get 方法可以返回任务执行的结果。但是如果任务执行过程中出现异常，则 get 方法将返回 异常对象。提交任务的线程可以使用这个异常对象对任务进行处理。

从应对异常的角度来看，Timer是非常脆弱的，一旦一个任务异常，将导致整个 Timer 对象失效，并且，提交任务的线程也无法通过任务机制，来获知任务执行出现了异常。

而使用 线程池 类，则任务出现异常也会导致 worker 线程终止，但线程孙池仍然可以正常工作，并且会自动创建新的 worker 线程。同时也提供了 submit, schedule 等方法，通过这种方法提交任务，可以使用其返回的 Future 对象跟踪任务的执行情况，出现异常，也能够通过 get 方法获取到。