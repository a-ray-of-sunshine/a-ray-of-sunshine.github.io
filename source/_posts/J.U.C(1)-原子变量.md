---
title: J.U.C(1)-原子变量
date: 2016-7-15 17:23:10
tags: [J.U.C]
---

## Atomic操作

A small toolkit of classes that support lock-free thread-safe programming on single variables.

在 java.util.concurrent.atomic 包中有关于原子操作的类，其分类有：

### 原子操作单个值

AtomicInteger 不需要额外的线程同步（例如：synchronized, 或者其它锁），就可以实现：

自增 value.incrementAndGet

自减 value.decrementAndGet

加某个值 AtomicInteger.getAndAdd

这些操作都是线程安全的不需要进行额外的线程同步了。这个类的使用场景是什么？

考虑这样一种情况：

``` java
class NumberHolder{

	private int messageCount;
	
}

```

在一个基于队列的消息处理系统中，有两个线程，一个生产者线程：产生消息，将其放置到 队列 queue 中，一个消费者线程：消费数据，从队列 queue 中获取消息，进行处理。有一个管理线程，进行检测队列queue,当queue中的消息数量达到一定数据的时候，启动另一个消费者线程，从而增强，系统的消息处理能力。此时，就需要一个类似 NumberHolder 的类来维护queue中消息的数量。生产者线程向queue中offer 一个消息时，需要将 NumberHolder 类的 messageCount 进行
加1 操作，相反消息者线程，需要进行 减1 操作。由于是两个操作是在两个线程中发生的，所以就必须对 messageCount 进行同步加锁，否则，这个数值在并发的情况下，会出问题。此时就可以使用 AtomicInteger 来代替 int 类型，这样生产者线程和消费者线程 对 messageCount 不需要任何额外的同步，就可以安全的对 messageCount 进行操作。


### 原子操作数组

### 字段修改器

使用 AtomicIntegerFieldUpdater 可以对类的  public volatile int 的变量进行同步修改

注意，可以修改的变量的声明必须是：

``` java
public volatile int varName;
```

### atomic使用场景

atomic variables off the same memory semantics as volatile variables, but with additional support for atomic updates --- making them ideal for counters, sequence generators, and statistics gathering while offering better scalability than lock-based alternatives.

并发环境下：计数器，序列生成器（web 应用生成数据库的id，多线程惟一），采集统计信息

### 注意事项

atomic 类继承自 Number，它们并没有继承 wrapper 类，例如： AtomicInteger 并没有继承 Integer，因为 Integer是 immutable,而 AtomicInteger 是 mutable.

atomic 类没有对 hashCode 和 equals 方法进行 override.所以它们也不能当作 HashMap 的 key 来使用。

在线程竞争不是特别高的时候，使用 Atomic 比使用 Lock 更加的好。

## 线程安全

当多个线程访问一个类时，如果不用考虑这些线程在运行时环境下的调度和交替运行，并且不需要额外的同步及在调用方代码不必做其他的协调，这个类的行为仍然是正确的，那么这个类就是线程安全的。

显然只有资源竞争时才会导致线程不安全，因此无状态对象永远是线程安全的。

## 原子操作

原子操作的描述是： 多个线程执行一个操作时，其中任何一个线程要么完全执行完此操作，要么没有执行此操作的任何步骤，那么这个操作就是原子的。

## 最终一致性 指令重排

## Happens-before 法则

## volatile 语义

**volatile 只能保证内存可见性，而不能保证原子性，所以它并能保证线程安全**

volatile相当于synchronized的弱实现，也就是说volatile实现了类似synchronized的语义，却又没有锁机制。它确保对volatile字段的更新以可预见的方式告知其他的线程。

volatile包含以下语义：

（1）Java 存储模型不会对valatile指令的操作进行重排序：这个保证对volatile变量的操作时按照指令的出现顺序执行的。

（2）volatile变量不会被缓存在寄存器中（只有拥有线程可见）或者其他对CPU不可见的地方，每次总是从主存中读取volatile变量的结果。也就是说对于volatile变量的修改，其它线程总是可见的，并且不是使用自己线程栈内部的变量。也就是在happens-before法则中，对一个valatile变量的写操作后，其后的任何读操作理解可见此写操作的结果。

尽管volatile变量的特性不错，但是volatile并不能保证线程安全的，也就是说volatile字段的操作不是原子性的，volatile变量只能保证可见性（一个线程修改后其它线程能够理解看到此变化后的结果），要想保证原子性，目前为止只能加锁！

volatile通常在下面的场景：

``` java
volatile boolean done = false;

while(!done){
    dosomething();
}
```
应用 volatile 变量的三个原则：

1. 写入变量不依赖此变量的值，或者只有一个线程修改此变量

2. 变量的状态不需要与其它变量共同参与不变约束

3. 访问变量不需要加锁

## CAS (Compare and Swap)

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

``` java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

其中： 

* this, valueOffset 确定了内存值V

* expect 确定了预期值 A，

* update 确定了新值 B

现代的CPU提供了特殊的指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。

整体的过程就是这样子的，利用CPU的CAS指令，同时借助JNI来完成Java的非阻塞算法。其它原子操作都是利用类似的特性完成的。

而整个J.U.C都是建立在CAS之上的，因此对于synchronized阻塞算法，J.U.C在性能上有了很大的提升。

CAS 相关参考：

[Java并发编程之CAS](http://ifeve.com/compare-and-swap/)

[深入理解并发之CompareAndSet(CAS)](http://flychao88.iteye.com/blog/2269438)

Implementation of class sun.misc.Unsafe 的c++实现：

openjdk\hotspot\src\share\vm\prims\unsafe.cpp

openjdk\hotspot\src\os_cpu\windows_x86\vm\atomic_windows_x86.inline.hpp

[深入JVM锁机制1-synchronized](http://blog.csdn.net/chen77716/article/details/6618779)

[深入JVM锁机制2-Lock](http://blog.csdn.net/chen77716/article/details/6641477)

[聊聊并发（二）Java SE1.6中的Synchronized](http://ifeve.com/java-synchronized/)

[CPU对原子操作的支持](http://www.cnblogs.com/fanzhidongyzby/p/3654855.html)

## 锁的缺点

* 引起的频繁调度

	多个线程请求同一个锁时，得不到锁的线程会被操作系统 suspend,等锁被释放，线程只是被 resume,而不被会立即调度，所以使用锁是非常重量级的。当锁被频繁竞争时，线程调度的压力非常大。

* 轻量级同步机制 --- volatile 变量

	只能实现，内存可见性，不能实现原子性。
	
* 锁导致其它等待锁的线程 cann't make progress.

	引起 priority inversion 问题。高优先级线程 wait 低优先级持有的 lock.如果一个持有 lock 的线程持续阻塞（permanently blocked）, 那么其它等待该 lock 的线程也将 never make progress.

## 悲观（pessimistic） 和 乐观（optimistic）

* 悲观锁

	线程通过获取锁的方式来确保，在接下来的代码中不会有其它线程对共享变量进行操作
	
* 乐观锁

	总是认为可以在没有其它干涉的情况下完成对共享变量的操作（更新）。方法是：当更新的时候，如果发现当前的共享变量已经被其它线程干涉（更新），那么这次操作就算失败，然后重新尝试。

### CAS --- 乐观锁的实现

CPU对共享变量的并发访问提供了硬件级别的支持，早期的实现有： test-and-set, fetch-and-increment or swap 指令。现代CPU提供了以 原子 形式执行 read-modify-write 操作的 指令（instruction）, 例如： compare-and-swap, load-linked/store-conditional.

大多数处理器使用 compare-and-swap (CAS) instruction.

CAS means “I think V should have the value A; if it does, put B there, otherwise don't change it but tell me I was wrong."

CAS addresses the problem of implementing atomic read-modify-write sequences without locking, because it can detect interference(冲突，干涉) from other threads.

CAS 实现了在没有锁的情况下，原子地执行 read-modify-write 操作。

锁机制，涉及 JVM和OS加锁操作，线程调度，上下切换。而CAS不会涉及 JVM code, 系统调用（system call）,和其它调度活动。

### CAS 的缺点

The primary disadvantage of CAS is that it forces the caller to deal with contention(竞争)（通过 循环，重试或者放弃）。whereas locks deal with contention automatically by blocking until the lock is available.

CAS 迫使调用线程必须处理竞争，而 锁机制 则将当前线程阻塞挂起，直到获得锁。

### JVM 对 CAS 的支持

从 java 5.0 开始，JVM 提供了对CAS的支持。对于支持 CAS 的 platform (CPU) ,Java runtime 会 inline them into the appropriate machine instruction. 如果CPU不支持 CAS， 则 JVM 使用 spin lock 来实现 CAS。

## 非阻塞算法（nonblocking algorithms）

* blocking 阻塞
	
	基于锁的算法，可能导致阻塞。如果一个线程因为 blocking I/O, 页错误（page fault）,或者其它延迟（例如：网络延迟，甚至网络中断的情况），就会导致其它所有在锁在等待的线程 blocking.

* nonblocking 非阻塞

	如果任意一个线程的失败或者挂起，不会导致其它线程的失败或者挂起，则这个算法就是 nonblocking 的。
	
	非阻塞算法，也不会产生死锁和优先级反转

* lock-free 锁自由

	如果在一个算法的每一步中，都有其它的一些线程可以继续执行，则这样的算法称为 lock-free 的

## 参考

[有什么好的并发书籍推荐？](https://www.zhihu.com/question/27072408)

[深入浅出 Java Concurrency](http://www.blogjava.net/xylz/archive/2010/07/08/325587.html)

[JUC线程池框架综述](http://www.cnblogs.com/leesf456/p/5550473.html)

[JAVA并发编程J.U.C学习总结](http://www.cnblogs.com/chenpi/p/5614290.html)

[Java并发编程实战](http://blog.csdn.net/column/details/java-concurrent-prog.html)

[concurrency tutorial](https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html)

[聊聊并发](http://www.infoq.com/cn/author/%E6%96%B9%E8%85%BE%E9%A3%9E)

[sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)

[正确使用 Volatile 变量](http://www.ibm.com/developerworks/cn/java/j-jtp06197.html)