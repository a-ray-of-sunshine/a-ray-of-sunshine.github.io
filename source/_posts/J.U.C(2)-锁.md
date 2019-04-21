---
title: J.U.C(2)-锁
date: 2016-7-18 11:36:10
tags: [J.U.C, lock]
---

## Lock

JDK5 中的锁接口： java.util.concurrent.locks.Lock

文档翻译：

Lock提供了对 synchronized 锁机制的扩展。允许更加灵活的组织代码，

Lock 是多线程访问共享资源的一种控制方法。通常，Lock 提供独占（exclusive）的访问共享资源:在任意时刻只有一个线程可以获得锁，与此同时其它任何想访问共享资源的线程必须获得（acquired）锁才可以。然而，一些 Lock 的实现允许并发访问一个共享资源，例如： read lock of a ReadWriteLock

synchronized 方法和语句提供了一种访问对象关联的隐式锁（implicit monitor lock）,but forces all lock acquisition and release to occur in a block-structured way: when multiple locks are acquired they must be released in the opposite order, and all locks must be released in the same lexical scope in which they were acquired. 但是强制所有锁的请求和访问都在同一个代码块中：当一个代码块加了多个锁时，必须以锁的相反的顺序释放锁，同时加锁和释放锁必须在同一个上下文中。这就是所谓 scope 。

synchronized的 scope 的机制，使得使用 monitor lock编程变得简单，同时避免了涉及锁的普遍错误(common programming errors invloving locks.)

``` java
synchronized(A){
	synchronized(B){
		synchronized(C){
			// access shared resource
		}
	}	
}
```

在上面的代码块中获取锁是依照： A, B, C的顺序的，而释放锁是依照： C, B, A的顺序，这个顺序是严格确定的，没法改变的。

而有时候，我们可能要以：you acquire the lock of node A, then node B, then release A and acquire C, then release B and acquire D and so on， 这种顺序使用锁，显然，对于 synchronized 是无法做到的，而Implementations of the Lock interface enable the use of such techniques by allowing a lock to be acquired and released in different scopes, and allowing multiple locks to be acquired and released in any order.  Lock 接口的实现，却可以做以任意顺序释放锁。

虽然释放锁非常灵活，但是对于Lock来说，必须显式的释放锁，而 synchronized 则会自动释放锁。使用 Lock 通用代码格式如下：

``` java
Lock l = ...;
l.lock();
try {
	// access the resource protected by this lock
} finally {
	l.unlock();
}
```

Lock实现的不同于 synchronized  机制的另几种功能：

* tryLock() 非阻塞方式获取锁 

	Acquires the lock only if it is free at the time of invocation. 
	
	Acquires the lock if it is available and returns immediately with the value true. If the lock is not available then this method will return immediately with the value false. 

	就在这个方法被调用的时机，去检测锁是否free，如果锁是free的，则当前线程成功获取到锁，这个方法立即返回true。如果在这个方法调用的时候锁不可用，则方法立即返回false.
	
	典型的使用方法：
	
	``` java
      Lock lock = ...;
      if (lock.tryLock()) {
          try {
              // manipulate protected state
          } finally {
              lock.unlock();
          }
      } else {
          // perform alternative actions
      }
	```

* lockInterruptibly() 可中断锁

	Acquires the lock unless the current thread is interrupted. 
	
	Acquires the lock if it is available and returns immediately. 
	
	获取锁除非当前线程是interrupt的。如果锁可用，则立即返回。如果锁不可用则当前调用线程进行休眠状态，直到下面两种情况发生：
	
	* 当前线程成功获取了锁（The lock is acquired by the current thread.）
	
	* 其它线程中断了当前线程，则这个方法抛出 InterruptedException异常，
	
	在下面两种情况下会出现InterruptedException异常：
	
	* lockInterruptibly 方法执行的第一个操作就是检测interrupted status ，如果有，则直接抛出异常。
	
	* 获取锁的等待过程中被其它线程中断

* tryLock(long, TimeUnit unit) 可超时获取锁

	Acquires the lock if it is free within the given waiting time and the current thread has not been interrupted. 
	
	如果在指定的等待时间内（unit）锁是free的，并且当前线程没有被 interrupt, 则当前线程成功获取锁。
	
	如果锁是free的，则方法立即返回true, 当前线程成功获得锁。如果锁不可用，则当前线程进入休眠状态，直到下面三种情况发生：
	
	* 当前线程获得了锁
	
	* 其它线程中断了当前线程
	
	* 指定的等待时间过去了
	
	如果锁在等待过程中被获取到，则这方法返回true.
	
	在下面两种情况下当前线程会抛出 InterruptedException：
	
	* 方法调用时，当前线程就处于 interrupted status
	
	* 等待请求锁的过程中被其它线程中断，并且 锁的实现 支持在 线程 在 等待请求锁的过程中 被中断。Lock的一种实现 ReentrantLock ，就是支持 获取锁的过程中被中断。

### lock

和 synchronized 类似的功能就是 lock 方法

如果锁不可用，则当前线程休眠，直到当前线程获取到锁。

### unlock

和上面的获取锁的方法对应，释放锁。
	
Lock的实例，也可以用在 synchronized ，并且跟 它的方法所提供的 lock 机制，没有任何关系，但是**最好不要这样使用**

## AbstractQueuedSynchronizer

java.util.concurrent.locks.AbstractQueuedSynchronizer

实现锁机制的核心抽象

文档翻译

AbstractQueuedSynchronizer 是 基于 FIFO 等待队列，用来实现 blocking locks 和 related synchronizers(semaphores, events, etc) 的一个框架。















## LockSupport

java.util.concurrent.locks.LockSupport

##　参考

[JUC锁框架综述](http://www.cnblogs.com/leesf456/p/5344133.html)

