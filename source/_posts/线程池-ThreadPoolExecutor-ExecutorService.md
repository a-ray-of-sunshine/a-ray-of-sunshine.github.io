---
title: 线程池-ThreadPoolExecutor-ExecutorService
date: 2016-8-15 11:51:38
tags: [ExecutorService, ThreadPoolExecutor]
---

## 文档
这个类继承自 Executor 
``` java
public interface ExecutorService extends Executor
```

这个接口提供了三类 API:

* 线程池终止函数
	
	ExecutorService 可以被关闭(shut down)，当其被关闭的时候，就会拒绝（reject）提交新任务。
	
	1. shutdown
		
		这个方法允许在关闭之前，将之前提交的任务执行完毕。
		
	2. shutdownNow
	
		这个方法会直接将已经提交的任务全部中断。然后返回。
		
	对于不使用的 ExecutorService，应该调用上面的两个方法来关闭，使其占用的资源被释放。
		
* 线程状态查询
	
	1. isShutdown
	
		Returns true if this executor has been shut down.

	2. isTerminated

		Returns true if all tasks have completed following shut down. Note that isTerminated is never true unless either shutdown or shutdownNow was called first.

* 跟踪任务提交

	1. Future<?> submit(Runnable task)
	
		提交任务，返回一个代表 task 的 Future 的对象。
		

## Future

Future代表异步任务的计算结果。提供了一种机制用来检测任务是否完成，一直等待直到任务完成，并返回计算结果。

返回的结果可以通过 get 方法来检索，如果任务没有执行完毕，则一直blocking.直到任务执行完毕，线程就恢复运行了。

或者调用 cancel 方法将任务，也会使得所有在该任务上 block 的线程恢复运行。

Future get 和 cancel, 方法的实现：Future实际上对提交的任务进行包装了，同时，Future也实现了 Runnable 接口。所以可以猜想，这个功能的实现。
``` java
class Future implements Runnable{

	// 任务是否被取消
	private bollean isCancel;
	
	// 实际提交的任务
	private Runnable task;
	
	// 在这个任务上等待的所有的线程
	private List<Thread> waiters;

	// 由线程池中的线程执行。
	public void run(){
		
		if(isCancel){
			return;
		}
	
		task.run();
		
		// 通知所有任务。
		for(Thread t: waiters){
			LockSupport.unpark(t);
		}
	}
	
	// 一般由提交任务的线程来调用
	// 当然，由于 submit 会返回 this 对象，
	// 这个 只要线程 拥有这个 Future 对象，
	// 就可以调用这个方法。也就是可以由
	// 多个线程来获取任务执行的结果。
	public V get(){
		// 将当前线程添加到任务等待链上
		waiters.add(Thread.currentThread());
		LockSupport.park(this);
	}
	
	// 这个方法也可以由任何，可以获得当前 Future
	// 对象的线程来调用。
	public boolean cancel(){
		this.isCancel = true;
	}
}
```


Future内部，维护了任务的生命周期（lifecycle）
	
	
	
	
	
	
	
	
	
	
	
	
	
	