---
title: Queue
date: 2016-7-13 16:29:04
tags: [Collection, Queue]
---

## 接口介绍

### Queue

``` java
public interface Queue<E> extends Collection<E>
```

Queue是一种FIFO的数据结构。Queues typically, but do not necessarily, order elements in a FIFO (first-in-first-out) manner. 

Collection中提供的Queue操作有两类：Each of these methods exists in two forms: one throws an exception if the operation fails, the other returns a special value (either null or false, depending on the operation). 

如下表所示：

 | Throws exception | Returns special value
---|:---:|:---:
insert|add(e)|offer(e)
remove|remove()|poll()
examine|element()|peek()

其中add 方法是继承自Collection接口，其它方法都是 Queue 接口新定义的。

**关于null:**

虽然Collection中提供的某些Queue的实现（例如：LinkedList），可以允许插入null元素，但 Queue 的使用过程中，一般不要插入 null 元素，因为，当 null 作为 poll 操作返回时，具有特殊的语义。

**关于BlockQueue:**

Queue 接口中并没有定义Block和Timeout的方法，这类方法通常的并发的情况下使用。定义的接口是：java.util.concurrent.BlockingQueue

### Deque

``` java
public interface Deque<E> extends Queue<E>
```

双端队列：A linear collection that supports element insertion and removal at both ends. 允许在队列两端执行插入和删除操作。

关于这个接口的定义的方法，可以参考文档，其中描述的非常详细。

Deque 可以也可以当作 LIFO 结构的 stack 来使用。This interface should be used in preference to the legacy Stack class. 在 Deque 的文档中明确提出，使用这个接口的实现中当作 stack 来使用，而不要使用 java.util.Stack。 例如：

``` java
Deque<Integer> stack = new ArrayDeque<Integer>();
```

## LinkedList

* 基于链表的实现

``` java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

可以看到 LinkedList 实现了接口 Deque。所以 LinkedList 可以当作 Queue 的一种实现来使用。 

## ArrayQeque

* 基于数组的实现

没有容量限制，非线程安全，不支持null，This class is likely to be faster than Stack when used as a stack, and faster than LinkedList when used as a queue. 

当栈使用时，比 Stack 性能好

当队列使用时，比 LinkedList 性能好

``` java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
```

### 初始化过程中的容量设置

无参数的构造函数默认分配16个元素的数组

可以指定数组的大小的构造函数，数组大小最小是8，同时数组的大小会被调整成 power of 2.

### 扩容策略

addFirst 和 addLast 方法中会调用 doubleCapacity 方法来进行扩容。主要逻辑是：

``` java
int n = elements.length;
int newCapacity = n << 1;
Object[] a = new Object[newCapacity];
System.arraycopy(elements, p, a, 0, r);
System.arraycopy(elements, 0, a, r, p);
elements = (E[])a;
```

1. 新的容量是原来容量的2倍
2. 将原来数组中的元素copy到新的数组
3. 将新的数组赋给 elements 字段

### addFirst

``` java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    if (head == tail)
        doubleCapacity();
}
```
**head = (head - 1) & (elements.length - 1)**

* head <= elements.length - 1
* 当初始 head = 0, elements.length = 16, 上面的表达式的值为 (0 - 1) & (16 - 1) = -1 & 15 = 15, 所以第一个addFirst 的元素插入到数组 elements 的最后一个位置
* head始终指向当前头部的元素
* 上面的表达式，会随着不断的 addFirst， 数值依次是： 15, 14, 13, ..., 2,0,1


### addLast

``` java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```

**tail = (tail + 1) & (elements.length - 1)**

* tail <= (elements.length - 1)
* 当初始 tail = 0, elements.length = 16, 上面的表达式的值为 1 & 15 = 1
* tail 始终表示最后一个元素的下一个待插入的位置，这和 head 的含义不同！！！
* 表达式的值依次是： 1, 2, 3, ..., 15

由此可知：

* 0 至 (tail - 1)，存放的是 Last 的元素

* head 至 (elements.length - 1), 存放的是 First 的元素

当 head = tail 的时候，调用 doubleCapacity 进行扩容

## clone

``` java
result.elements = Arrays.copyOf(elements, elements.length);
```

使用 Arrays 的 copyOf 方法将底层数组copy一份。

### iterator 和 descendingIterator

* java.util.ArrayDeque.DeqIterator

java.util.ArrayDeque.iterator() 方法返回 DeqIterator， 正序迭代：从 head 到 tail


* java.util.ArrayDeque.DescendingIterator

 java.util.ArrayDeque.descendingIterator() 方法返回 DescendingIterator， 逆序迭代：从 tail 到 head
 
这两个 iterator 都是 fail-fast 的。

## PriorityQueue

``` java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
```

实现队列元素的排序，使用两种方法：

在构造对象时，传递一个参数：Comparator， 或者 队列中元素实现了接口 java.lang.Comparable， 当 元素被添加到队列中时，会使用上面的两种方式，对元素进行优先级排序。

### 扩容策略

初始容量11，如果容量小于64，则：newCapacity = 2 * oldCapacity。 否则，newCapacity = 1.5 * oldCapacity

### 实现

内部使用数组开存储元素。实现维护的数据结构是 最小堆 。它是一种基于优先级堆的极大优先级队列。优先级队列是不同于先进先出队列的另一种队列。每次从队列中取出的是具有最高优先权的元素。

在 offer 和 poll 方法中都分别使用了 siftUp 和 siftDown 方法来平衡这个最小堆。

维护的最小堆，其堆顶是整个Queue中，优先级最大的元素。

### iterator 

Returns an iterator over the elements in this queue. The iterator does not return the elements in any particular order.

iterator按数组的存储返回元素，没有任何优先顺序。