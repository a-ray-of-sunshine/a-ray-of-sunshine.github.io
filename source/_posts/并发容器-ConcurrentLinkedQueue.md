---
title: ConcurrentLinkedQueue
date: 2016-8-8 17:35:23
tags: [ConcurrentLinkedQueue]
---

## 内部类 Node

``` java
private static class Node<E> {
	// 存储队列中的元素
    volatile E item;
    
    // next 存储单向链表，指向下一个结点
    volatile Node<E> next;
}
```

## 初始化

``` java
// 队列头结点
private transient volatile Node<E> head;

// 队列尾结点
private transient volatile Node<E> tail;

// 构造函数
public ConcurrentLinkedQueue() {
	// 头尾均指向一个空的结点
    head = tail = new Node<E>(null);
}
```

## put过程

可以参考下面的链接。

## 参考

[JDK1.8源码分析之ConcurrentLinkedQueue（五）](http://www.cnblogs.com/leesf456/p/5539142.html)

[gcc Unsafe](https://github.com/gcc-mirror/gcc/blob/gcc_5_3_0_release/libjava/sun/misc/Unsafe.java)

[openjdk Unsafe](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/sun/misc/Unsafe.java)
