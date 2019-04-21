---
title: ArrayList
date: 2016-6-20 17:19:01
tags: [ArrayList, Collection]
---

## ArrayList的状态

* 实例状态：

``` java
/**
 * The array buffer into which the elements of the ArrayList are stored.
 * The capacity of the ArrayList is the length of this array buffer. Any
 * empty ArrayList with elementData == EMPTY_ELEMENTDATA will be expanded to
 * DEFAULT_CAPACITY when the first element is added.
  数组中包含的元素。
 */
private transient Object[] elementData;

/**
 * The size of the ArrayList (the number of elements it contains).
 * 这个list的中包含的元素的个数。
 */
private int size;

```

* 类级状态

``` java
/**
 * Default initial capacity.
   内部数组 默认 大小
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * Shared empty array instance used for empty instances.
   空数组
 */
private static final Object[] EMPTY_ELEMENTDATA = {};
```

ArrayList 内部使用 Object[] 来保存添加到其中的元素。
但是，ArrayList 是支持泛型，那么可不可以使用这个泛型参数来创建对象数组。
Java 是不允许创建泛型数组的。所有内部数据都是保存在 Object 数组中。

``` java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
```

## ArrayList 的构造函数

* ArrayList()

``` java
    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        super();
        this.elementData = EMPTY_ELEMENTDATA;
    }
```

* ArrayList(int initialCapacity)

``` java
    /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
    }
```

* ArrayList(Collection<? extends E> c)

``` java
    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        size = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
```

## fail-fast iterators

### Iterator 接口
集合类的根接口 Collection 继承自 Iterable 接口，这个接口方法

``` java
Iterator<E> iterator();
```
这个接口会返回一个 Iterator 类型的对象。可以通过这个接口遍历 Collection 对象，同时这个接口提供了一个 remove 方法，可以在遍历对象的同时删除对象中的元素。

### fail-fast

ArrayList 类的父类 AbstractList 有一个字段： modCount 这个字段用来，跟踪 List 对象的被修改情况，

``` java
protected transient int modCount = 0;
```
文档描述如下：

The number of times this list has been structurally modified. Structural modifications are those that change the size of the list, or otherwise perturb it in such a fashion that iterations in progress may yield incorrect results. 

用来记录 list 的结构性的修改。结构性的修改包括 list 的size 变化，或者是其它扰动（perturb）

This field is used by the iterator and list iterator implementation returned by the iterator and listIterator methods. If the value of this field changes unexpectedly, the iterator (or list iterator) will throw a ConcurrentModificationException in response to the next, remove, previous, set or add operations. This provides fail-fast behavior, rather than non-deterministic（非确定性） behavior in the face of concurrent modification during iteration. 

这个字段的主要用途是用来提供 iterator 的 fail-fast 行为(behavior)， 在 iterator 方法 next, remove 中都会检测 list 对象的  modCount 这个字段是否被修改，如果被修改了，则 直接抛出 ConcurrentModificationException 异常

## fail-fast 与 线程安全

iterator 的 fail-fast 机制 并不能保证线程安全。

java.util.ArrayList.Itr 类的 next 方法的实现，

``` java
        public E next() {
        	// 这里检测 modCount 是否发生变化，
        	// 如果有变，则抛异常，
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            // 而出现并发问题的代码是下面这行，
            // 这里并没有使用任何线程同步机制，
            // 来保证 elementData 不会出现，并发
            // 读写问题，所以此时的list仍处理线程不
            // 安全状态。
            return (E) elementData[lastRet = i];
        }
```

由上面的代码及分析可知，fail-fast 机制，由于没有线程同步的保证，所以它并不是完全精确的行为。

那么，iterator 采用 fail-fast 机制的目地或者作用是什么呢？

关于 [fail-fast wiki](https://en.wikipedia.org/wiki/Fail-fast) 其中提到 fail-fast 机制的使用场景，其实也表达的它的使用目地。 归结起来就是：错误检测和提高容错能力。

[What is fail safe and fail fast Iterator in Java?](http://www.java67.com/2015/06/what-is-fail-safe-and-fail-fast-iterator-in-java.html)

[fail-fast机制](http://cmsblogs.com/?p=1220)

[ArrayList javadocs](https://docs.oracle.com/javase/7/docs/api/java/util/ArrayList.html) 其中有关于 ArrayList 线程同步问题和fail-fast 的相关介绍。

### structurally modified

A structural modification is any operation that adds or deletes one or more elements, or explicitly resizes the backing array; merely setting the value of an element is not a structural modification.

增加，删除元素，或者显示修改底层的数据的大小，都是structural modification

### java.util.ArrayList.SubList

ArrayList 方法 subList 

``` java
public List<E> subList(int fromIndex, int toIndex)
```

返回一个 List 的视图。其操作的还是当前的 List 对象。这个方法的主要用途就是：

This method eliminates the need for explicit range operations (of the sort that commonly exist for arrays). Any operation that expects a list can be used as a range operation by passing a subList view instead of a whole list. For example, the following idiom removes a range of elements from a list:
 list.subList(from, to).clear();

如果需要操作一个list的某个范围的数据，可以使用上面的方法。

**注意**
当使用 subList 方法返回 list的一个视图的时候，在使用 sublist 时，就不能再操作 list, 否则会报 java.util.ConcurrentModificationException

## remove 操作

如果待删除的元素是null,下面的操作也能成功，会找 list 中 null的元素。

* 删除指定索引的元素

	public E remove(int index)

* 删除指定对象

	public boolean remove(Object o)

* 删除指定collections中的元素

	 public boolean removeAll(Collection<?> c)

* 删除collection 以外的元素

	public boolean retainAll(Collection<?> c)

## ArrayList 对 null 的支持

ArrayList实现中，不会对参数为 null ，也就是空对象，进行检测，包括 add ,set , remove, contains 等方法。也就是它们基本不会因为操作的元素的null，而抛 NullPointerException。其实这也反映了 ArrayList 的本质作用，作为容器，就应该可以容纳各种对象，而 null 对象，也是对象的一种。

关于集合类支持 null 值的 thread:

[Why Null value is allowed in ArrayList,Vector,Set or HashMap?](http://www.coderanch.com/t/603686/java/java/Null-allowed-ArrayList-Vector-Set)

ArrayList 文档描述：

Resizable-array implementation of the List interface. Implements all optional list operations, and permits all elements, **including null.**

## 扩容策略

* 公共接口

``` java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != EMPTY_ELEMENTDATA)
        // any size if real element table
        ? 0
        // larger than default for empty table. It's already supposed to be
        // at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```

如果 ArrayList 内部的存储数据的数组： elementData 不为空（elementData != EMPTY_ELEMENTDATA），则 这个方法调用可以传递 > 0 的任何数值，作为扩容的参数。否则，传递的参数，必须大于ArrayList类的初始默认的容量：DEFAULT_CAPACITY = 10，就是说如果此时传递参数为 5, 则这个方法什么都不会做。

* 内部的扩容方法

``` java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

如果elementData是null,则扩容取 DEFAULT_CAPACITY 和 minCapacity 中较大的值。如果它minCapacity 小于 DEFAULT_CAPACITY 这个方法会按 DEFAULT_CAPACITY 去扩容，而上面公共方法中则什么都不会做。这是这两个方法的区别。

* ensureExplicitCapacity

``` java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

调用 grow 方法

``` java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

a. newCapacity 是 当前容量的 1.5 倍

	newCapacity = oldCapacity + (oldCapacity >> 1)

b. newCapacity 小于 minCapacity，则新容量
	
	newCapacity = minCapacity

c. newCapacity 超过数组的最大值 MAX_ARRAY_SIZE

	newCapacity = (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE

注意 minCapacity 如果 < 0, 则会抛出 OutOfMemoryError 的错误。其实这种情况应该是溢出了， 例如： minCapacity = Integer.MAX_VALUE + 20 此时的 minCapacity 其实是 -2147483629 所以会出现 < 0 的状况。

## clone

ArrayList重写的 clone 方法，实现底层数组的浅clone.返回一个新的ArrayList。注意这个clone后的对象的 modCount 会被重置成 0.

##  System.arraycopy

这是一个 native 的方法：

``` java
	public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos, int length);
``` 

其作用是：
Copies an array from the specified source array, beginning at the specified position, to the specified position of the destination array. A subsequence of array components are copied from the source array referenced by src to the destination array referenced by dest. The number of components copied is equal to the length argument. The components at positions srcPos through srcPos+length-1 in the source array are copied into positions destPos through destPos+length-1, respectively, of the destination array.

将数组 src 从位置 srcPos 开始的元素 copy 到 数组 dest的 destPos 中去。其中 copy 元素的个数是 length.

## ArrayList vs Vector

### 线程安全

Vector 是线程安全的， ArrayList未作任何同步处理

### 扩容

当存储元素的数组容量不够时，ArrayList 和 Vector 的扩容数量不同：

* ArrayList

	``` java
	int oldCapacity = elementData.length;
	int newCapacity = oldCapacity + (oldCapacity >> 1);
	```

	newCapacity = oldCapacity * 1.5
	
* Vector

	``` java
	int oldCapacity = elementData.length;
	int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                        capacityIncrement : oldCapacity);
	```

	newCapacity = oldCapacity * 2

### fail-fast iterator

Vector 有一个返回 Enumeration 的方法 elements，返回的 Enumeration 不是 fail-fast, 但是 iterator 和 listIterator 返回的 iterator 是 fail-fast的

ArrayList 没有 elements 方法，它的 iterator 也都是 fail-fast 的。

## Stack

JDK 中提供的 java.util.Stack类，基于 Vector 实现的，也是同步的，但是，其父类已经都不推荐使用了，所以：

A more complete and consistent set of LIFO stack operations is provided by the Deque interface and its implementations, which should be used in preference to this class. For example: 

	Deque<Integer> stack = new ArrayDeque<Integer>();

## 参考资料

[集合类相关博文](http://www.cnblogs.com/chenssy/category/525010.html)

[有关JVM处理Java数组方法的思考](http://developer.51cto.com/art/201001/176671.htm)

[Difference between ArrayList and Vector In java](http://beginnersbook.com/2013/12/difference-between-arraylist-and-vector-in-java/)