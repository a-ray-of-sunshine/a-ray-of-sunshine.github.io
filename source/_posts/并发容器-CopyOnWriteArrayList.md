---
title: CopyOnWriteArrayList
date: 2016-8-8 17:35:23
tags: [CopyOnWriteArrayList]
---

## 文档介绍
This is ordinarily too costly, but may be more efficient than alternatives when traversal operations vastly outnumber mutations, and is useful when you cannot or don't want to synchronize traversals, yet need to preclude interference among concurrent threads.

## 成员变量
``` java
/** The lock protecting all mutators */
transient final ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private volatile transient Object[] array;
```

* array 变量是 volatile 的所以 array 的任何其它线程对 array的修改都会被其它线程立即看到。

* lock 所有的修改操作都会使用 lock 来进行同步。


## 构造函数
``` java
/**
 * Creates an empty list.
 */
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

创建一个0长数组。

## 其它功能
### 结构性修改操作
所有的这类操作，例如： add, set, remove 等待,都会重新创建一个 array, 并在这个 array 上，进行操作，最终，使用 set 方法将其设置为新的底层数据，这类的方法的代码结构：
``` java
public void mutatorOperate(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    	// 1. 获得当前的 array.
        Object[] elements = getArray();
        int len = elements.length;
        
        // 2. 创建新的 array.
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        
        // 3. 对新的 array 进行操作
        newElements[len] = e;
        
        // 4. 将新的 array 调整为，当前的底层数组。
        setArray(newElements);
        
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 非修改性操作

非修改性操作，通常就是遍历查找操作，这些操作，都是方法在调用时刻，当前线程获得的 array 的快照上进行的，由于在并发的情况下如果 array 被修改了，则array会指向新的地址，所以 当前线程获得的 快照是永远不会再被修改的，则这些操作自然不需要加锁。

所以 CopyOnWriteArrayList 是写操作加锁，读操作不加锁的，所以它的通常的使用场景就是 适合用在“读多，写少”的“并发”应用中。写少：使用锁同步进行。读多：多个读线程可以并发进行。所以其并发效率较高。


所有的非修改性操作都是不加锁的，因为不会涉及到底层数组的修改，所以不会加锁，例如，下面是 containsAll 方法的实现。
``` java
public boolean containsAll(Collection<?> c) {
	// 返回当前的一个快照
    Object[] elements = getArray();
    int len = elements.length;
    for (Object e : c) {
        if (indexOf(e, elements, 0, len) < 0)
            return false;
    }
    return true;
}
```
实际上这些不加锁的操作，在进行处理的时候，是针对，当前的 array 的一个快照。
例如，一个线程在进行上面的 containsAll 操作，正在对 elements 进行遍历,而另一个线程执行了 add 操作。由于 containsAll 方法并没有加锁，所以，add 操作是有可能在 containsAll 方法执行过程中 执行成功的，也就是说，此时 底层的array 实际已经发生变化了。但是对于 containsAll 方法却感知不到，因为，它是在 array的一个快照是进行操作的。

### iterator
CopyOnWriteArrayList.COWIterator 这个类的实现，也是在当前 array 的快照上实现的，而 ListIterator 接口定义的对 array 的修改操作全部是不支持的。直接抛出 UnsupportedOperationException 异常。


### subList
CopyOnWriteArrayList.COWSubList 这个类实现了 subList 方法功能。这个反映的当前array的一个视图。这个视图是支持对 原始的 array 进行修改操作的，所以它使用了 lock 对操作进行了加锁。同时当在视图上进行操作的时候，如果有其它线程对当前的 array 做了修改操作，则这个视图的方法就会抛出 ConcurrentModificationException 异常。所以对视图的操作时，其它任何线程都不能对当前array进行修改操作。

这和 ArrayList 实现的 subList 是一样的，其返回的视图也是不能被并发修改的，如果出现并发修改，就会抛出异常。

注意，除了并发修改会导致 ConcurrentModificationException 异常，即使在同一个线程中也有可能出现，上述异常：例如：
``` java

CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("123");
list.add("456");
list.add("789");

List<String> subList = list.subList(0, list.size());
subList.add("999");
System.out.println(list);
// 当进行视图操作是时候，就不能使用原来的list,否则，会导致
// ConcurrentModificationException
list.add("111");
subList.add("000");
System.out.println(list);
```

## 参考

[Java中的CopyOnWrite容器](http://coolshell.cn/articles/11175.html)

[Java中CopyOnWriteArrayList的实现分析](http://blog.lday.me/?p=217)