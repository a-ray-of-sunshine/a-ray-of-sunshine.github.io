---
title: Collections
date: 2016-7-15 16:30:52
tags: [Collections, Collection]
---

## 概述

Collections类是一个工具类，提供了大量关于 Collection 的操作。算法，包装类等等。

## synchronized系方法

这种类型的方法，返回不同类型的Collection的包装类，内部使用 synchronized 进行完成的同步。

**同步容器返回的iterator，并没有做同步处理，所以需要显示同步处理**

``` java
// SynchronizedCollection 的几个方法：
public int size() {
    synchronized (mutex) {return c.size();}
}
public boolean isEmpty() {
    synchronized (mutex) {return c.isEmpty();}
}
public boolean contains(Object o) {
    synchronized (mutex) {return c.contains(o);}
}

// 上面的几个方法都进行了同步
// 而iterator，则是直接返回的底层的未经同步的iterator
// 实事上，这里做同步也没有任何意义
public Iterator<E> iterator() {
    return c.iterator(); // Must be manually synched by user!
}

// 例如 如果按照上面的方法做同步，则就是
public Iterator<E> iterator() {
    synchronized (mutex) {return c.iterator();} 
}
// 而这样做却没有任何意义！！！
// 来看一个具体的 ArrayList的 iterator 方法的实现：
public Iterator<E> iterator() {
	// 这里仅是 new 了一个 Itr 的变量然后返回，
	// 而且 java.util.ArrayList.Itr 的实现中，其使用的默认的构造函数
	// 所以 new Itr() 并没有对 ArrayList 发生任何访问和修改，也就是说 
	// 共享资源（ArrayList中的全局变量）并没有在 new Itr 中被访问
	// 自然 ！！！不需要同步了！！！
	// 所以 上面 SynchronizedCollection 实现的 iterator 方法，是
	// 不需要同步，所以没有同步
    return new Itr();
}
```

SynchronizedCollection 实现的 iterator 方法，是不需要同步的，所以没有同步，真正需要进行同步的是这个方法返回之后的 iterator对象，客户端代码使用这个iterator对象调用其 next 和 remove 方法时，这时候需要访问到 ArrayList 的 modCount 字段，如果有其它线程此时对 ArrayList 进行了结构性修改，modCount字段就会发生变化，而这里（next, remove 方法的调用）会读取 modCount, **读写并发于modCount字段**，所以这里才是需要同步的地方。

由此，可以对于 SynchronizedCollection 的 iterator 方法返回的 iterator 对象的同步的责任就落到的客户端代码处，同步代码使用如下：
``` java


  Collection c = Collections.synchronizedCollection(myCollection);
  // 上面的构造函数默认使用this同步，所以这里必须使用 c 进行同步
  synchronized (c) {
      Iterator i = c.iterator(); // Must be in the synchronized block
      while (i.hasNext())
         foo(i.next());
  }

// 或者：
  Object lock = new Object();
  Collection c = Collections.synchronizedCollection(myCollection, lock);
  // 上面的构造函数默认使用 lock 同步，所以这里必须使用 lock 进行同步
  synchronized (lock) {
      Iterator i = c.iterator(); // Must be in the synchronized block
      while (i.hasNext())
         foo(i.next());
  }
```

**同步时，要使用和创建同步集合时同样的锁对象，才能正确的同步。**

## unmodifiable系方法

对 Collection 进行包装，将所有对 Collection 有结构上修改的方法全部实现成

	 throw new UnsupportedOperationException();

返回不可修改的集合

## empty系方法

对 Collection 进行包装，返回空的 Collection，一般都是不可修改的。

使用场景：

在某些情况下，我们经常需要发挥一个空的集合对象，比如说在数据查询时，并不需要发挥一个NULL或是异常，那么就可以返回一个空的集合对象。

关于这种类型的方法使用参考：

[Collections.emptyList() vs. new instance](http://stackoverflow.com/questions/5552258/collections-emptylist-vs-new-instance)

[Collections.EMPTY_LIST和Collections.emptyList()简单使用体会](http://inter12.iteye.com/blog/1023761)

## checked系方法

对 Collection进行包装，返回 checked 的集合。

## Arrays类

和 Collections 类似定义了大量关于数组的操作。
