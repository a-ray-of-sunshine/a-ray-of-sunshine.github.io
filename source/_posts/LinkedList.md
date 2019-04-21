---
title: LinkedList
date: 2016-07-10 10:20:14
tags: [Collection,LinkedList]
---

## 类的签名

``` java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

和ArrayList不同的是，LinkedList类继承自 AbstractSequentialList, 并实现了接口 Deque

## AbstractSequentialList

``` java
public abstract class AbstractSequentialList<E> extends AbstractList<E>
```

AbstractSequentialList类继承自AbstractList, 而ArrayList类则直接继承自AbstractList类，那么AbstractSequentialList类的使用是什么？


这个类提供了一个对于顺序访问的集合的基本实现。如果要实现随机访问，则应该直接实现AbstractList类。

AbstractSequentialList 和 AbstractList 不同的地方就是，AbstractSequentialList 使用 listIterator 方法返回的 ListIterator 对象来实现 List 接口中的随机访问集合类的方法：

``` java
get(int index)
set(int index, E element)
add(int index, E element)
remove(int index)) 
```

而 AbstractList 类本身也提供了一个 ListIterator 的实现，但是 ListIterator 的实现则是依靠上面的随机访问方法来实现的。

同时这也带来一个问题，上面依靠 ListIterator 实现的 get,set,add,remove 方法，有可能抛出 ConcurrentModificationException 异常。


例如：
``` java
public E remove(int index) {
    try {
        ListIterator<E> e = listIterator(index);
        // 这个方法调用有可能出现
        // ConcurrentModificationException 异常
        E outCast = e.next();
        e.remove();
        return outCast;
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}
```

**注意**
LinkedList类将上面的随机访问方法全部 override 了。

## java.util.Deque



## LinkedList.Node

表示 LinkedList 在存储的每一个 element. 实现代码如下：
``` java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### LinkedList中关于 Node 的变量

* 头结点

``` java
transient Node<E> first;
```

* 尾结点

``` java
transient Node<E> last;
```

### LinkedList 中关于 Node 的方法

LinkedList 实现的关键就是对 Node 的操作。

### 插入操作
* Links e as first element. 将元素e插入到链表的头部
	
	private void linkFirst(E e) 

* Links e as last element. 将元素e插入到链表的尾部

	void linkLast(E e)
	
* Inserts element e before non-null Node succ. 将元素插入到结点 succ 的前面。

	 void linkBefore(E e, Node<E> succ)

### 删除操作
* Unlinks non-null first node f. 删除头部结点 first

	private E unlinkFirst(Node<E> f)

* Unlinks non-null last node l. 删除尾部结点 last

	private E unlinkLast(Node<E> l)
	
* Unlinks non-null node x. 删除任意一个结点 x

	E unlink(Node<E> x)

对于 unlinkFirst 方法 和 unlinkLast 方法来说，其实其参数对应的就是： first 和 last, 可以不用传递的，但是由于这两个方法被使用的时候**可能**会对 first 和 last 进行 null 检测，所以这里以参数传递的目的应该是预防在并发的情况下出问题。

### 搜索操作

* Returns the (non-null) Node at the specified element index. 搜索链表中指定 index 处的 node
	
	Node<E> node(int index) 

List , Deque 接口的实现就是依靠上面的方法。

## clone

LinkedList override 了 clone 方法，依然是 shallow copy 