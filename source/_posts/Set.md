---
title: Set
date: 2016-7-15 10:14:35
tags: [Collection, Set]
---

## Set接口

``` java
public interface Set<E> extends Collection<E>
```

A collection that contains no duplicate elements. More formally, sets contain no pair of elements e1 and e2 such that e1.equals(e2), and at most one null element. As implied by its name, this interface models the mathematical set abstraction. 

Set 实现了 所有 Collection 接口中的方法，对其中的方法进行了语义的增强。

## HashSet

``` java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

这个内部实现是基于 HashMap

``` java
private transient HashMap<E,Object> map;
```

使用 HashMap 的key来存储，Set 中元素。其中value都是

``` java
// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();
```

由于HashMap的实现，保证了 key 的惟一性，所以就实现了 Set 中元素的惟一性。

### 如何获取其中的元素

由于没有类似于List那样的 get 方法，所以只能使用下面的方法：

* contains

* iterator

### HashSet 的使用场景

HashSet 内部实现使用了 HashMap，而 HashMap 将 key 散列到 table 中使用了 了 key 的 hashCode 方法和 equals 方法，所以如果一个类要做HashMap中的key，或是 HashSet中元素，则该类，必需 override 其 hashCode 和 equals 方法。

### 关于 hashCode 和 equals 方法

Object类提供的 hashCode 方法默认实现是返回对象的存储地址。而 equals 方法则是比较对象存储地址是否相等。


[Java 中正确使用 hashCode 和 equals 方法](http://www.oschina.net/question/82993_75533)

[深入解析Java对象的hashCode和hashCode在HashMap的底层数据结构的应用](http://kakajw.iteye.com/blog/935226)

## LinkedHashSet

``` java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

这个类直接继承自 HashSet， 使用 HashSet 的初始化，

``` java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

## EnumSet

``` java
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E>
    implements Cloneable, java.io.Serializable
```

注意到 EnumSet 是抽象类，这个类定义的大量方法都是 static 的方法，用来创建 EnumSet 的对象。

查看源代码可知：主要的方法是：noneOf

``` java
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}
```

内部实现类：

### RegularEnumSet

``` java
/**
 * Bit vector representation of this set.  The 2^k bit indicates the
 * presence of universe[k] in this set.
 */
private long elements = 0L;
```

用8个字节，8 * 8 = 64 bit 的向量来表示，universe 中的 枚举值 否是 存在。所以这个实现类，只能是 枚举类的元素小于等于64的类。

#### add 

``` java
public boolean add(E e) {
    typeCheck(e);

    long oldElements = elements;
    elements |= (1L << ((Enum)e).ordinal());
    return elements != oldElements;
}
```

	elements |= (1L << ((Enum)e).ordinal());

将 elements 中的 第 ((Enum)e).ordinal() 位的 bit 置为 1. 表示这个元素被插入了。

#### remove

``` java
public boolean remove(Object e) {
    if (e == null)
        return false;
    Class eClass = e.getClass();
    if (eClass != elementType && eClass.getSuperclass() != elementType)
        return false;

    long oldElements = elements;
    elements &= ~(1L << ((Enum)e).ordinal());
    return elements != oldElements;
}
```

	elements &= ~(1L << ((Enum)e).ordinal());

将 elements 中的 第 ((Enum)e).ordinal() 位的 bit 置为 0. 表示这个元素被删除了。

其它方法的实现，都是靠位运算来实现。实现非常巧妙。


### JumboEnumSet

也是使用bit向量来实现的，和 RegularEnumSet 类似

``` java
JumboEnumSet(Class<E>elementType, Enum[] universe) {
    super(elementType, universe);
    elements = new long[(universe.length + 63) >>> 6];
}
```

	elements = new long[(universe.length + 63) >>> 6];

(universe.length + 63) >>> 6 <==> (universe.length + 63) / 64

用来计算存储 universe.length 个bit 需要的long的个数。

#### add

``` java
public boolean add(E e) {
    typeCheck(e);

    int eOrdinal = e.ordinal();
    // 确定当前元素在哪个 element 中
    // <==> int eWordNum = eOrdinal / 64;
    int eWordNum = eOrdinal >>> 6;

    long oldElements = elements[eWordNum];
    elements[eWordNum] |= (1L << eOrdinal);
    boolean result = (elements[eWordNum] != oldElements);
    if (result)
        size++;
    return result;
}
```
### iterator

EnumSet 的 iterator 都不是 fail-fast 的。

### null元素

add 操作不支持 null 元素，会抛出 NullPointerException。remove 方法则不抛异常，直接返回 false. Attempts to test for the presence of a null element or to remove one will, however, function properly. 

**这个类的所有基本操作都是常量级时间**

### 使用场景


[What does EnumSet really mean?](http://stackoverflow.com/questions/11825009/what-does-enumset-really-mean) 其中有一种答案中给出了 EnumSet的使用场景。

在文档中有如下说明：

The space and time performance of this class should be good enough to allow its use as a high-quality, typesafe alternative to traditional int-based "bit flags." Even bulk operations (such as containsAll and retainAll) should run very quickly if their argument is also an enum set.

由此，EnumSet 应该当作 标志位(bit flag) 来使用。

[EnumSet使用场景](http://blog.csdn.net/zengqiang1/article/details/49841551)

## TreeSet

内部使用TreeMap来实现（有序二叉树），使用 Map 中的 key 来保存元素，value为默认的同一个元素。