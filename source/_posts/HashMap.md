---
title: HashMap
date: 2016-7-11 10:10:05
tags: [Collection, HashMap]
---

## 基本情况

``` java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

## Map 接口

* map 中不能有相同的key,每一个key最多对应一个value 

	An object that maps keys to values. A map cannot contain duplicate keys; each key can map to at most one value.

* map 接口提供了三个Collection视图，分别是： keys, values, entrys

	The Map interface provides three collection views, which allow a map's contents to be viewed as a set of keys, collection of values, or set of key-value mappings. 

* **当可变对象(mutable object)做key时需要注意**
	
	great care must be exercised if mutable objects are used as map keys. The behavior of a map is not specified if the value of an object is changed in a manner that affects equals comparisons while the object is a key in the map. 

## 成员变量

* EMPTY_TABLE: An empty table instance to share when the table is not inflated.

	``` java
	static final Entry<?,?>[] EMPTY_TABLE = {};
	```

* table: The table, resized as necessary. Length MUST Always be a power of two.

	``` java
	transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
	```
* size: The number of key-value mappings contained in this map.

	``` java
	transient int size;
	```

* threshold: 阈值 The next size value at which to resize (capacity * load factor).

	``` java
	int threshold;
	```

* loadFactor: 负载因子 The load factor for the hash table.

	``` java
	final float loadFactor;
	```

* capacity: 容量 The length of the table
	
	就是 table 数组的长度。这个 table 数组并不会完全存储完，而是达到 threshold 之后，重新分配。

## HashMap 的构造函数

* HashMap(int initialCapacity, float loadFactor)

``` java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        this.loadFactor = loadFactor;
        threshold = initialCapacity;
        init();
    }
```

其中 MAXIMUM_CAPACITY 是 1 >> 30, 也就是 2 的 30 次方。

无参数的构造函数，使用默认的 loadFactor 和 initialCapacity，分别是 DEFAULT_INITIAL_CAPACITY： 16，DEFAULT_LOAD_FACTOR：0.75f

* public HashMap(Map<? extends K, ? extends V> m) 

``` java
    public HashMap(Map<? extends K, ? extends V> m) {
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                      DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
        inflateTable(threshold);

        putAllForCreate(m);
    }
```

使用已有的Map来初始化当前的对象，其中的 inflateTable 方法，用来分配当前table的空间。

``` java
    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        // 将 threshold 调整到 capacity
        int capacity = roundUpToPowerOf2(toSize);

		// 重设阈值
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        
        // 创建 capacity 大小的 Entry 数组。
        table = new Entry[capacity];
        // 初始化 hash 掩码（mark）值: hashSeed
        initHashSeedAsNeeded(capacity);
    }
```

所以当使用默认的参数初始化HashMap后，HashMap，内部的状态是：

loadFactor: 0.75

threashold: 16 * 0.75 = 12

table = new Entry[16]

capacity = table.length = 16

size = 0

## HashMap 的 put 元素的过程中使用的关键方法

### public V put(K key, V value) 方法

``` java
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        
        // 查找 key 是否在 table[i] 这个 entry 链
        // 上存在，如果存在，则直接进行替换。
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

		// 代码执行到这里说明，key 在 table[i]中
		// 并不存在，所以，进行添加操作
        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

上面put过程中比较关键的几个方法：

### hash

根据 key 值，计算 key 的 hashCode.

### indexFor

``` java
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}
```

注意： h & (length-1) <= (length-1) 

### addEntry

``` java
void addEntry(int hash, K key, V value, int bucketIndex) {
	// 对 table 进行 resize 的条件：size 超过 阈值 并且 table 中的 bucketIndex entry 已经被使用过
    if ((size >= threshold) && (null != table[bucketIndex])) {
    
    	// 对 table 进行 resize
        resize(2 * table.length);
        
        // 由于进行了 resize, 所以要对 hash和index
        // 重新进行计算
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}
```

### resize

``` java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 如果当前的容量已经到上限，
    // 则调整阈值成最大值，后直接返回
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    
    // 重新计算 hashSeed,然后，对当前的 table 的
    // Entry 重新进行 hash 散列，放置到新的 table 中
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    
    // 调整 table
    table = newTable;
    
    // 调整 阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

### transfer

``` java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            // newTable[i] 处的 Entry，放置到新的 Entry的后部，
            e.next = newTable[i];
            // 当前新的 Entry ，放到队头。
            newTable[i] = e;
            e = next;
        }
    }
}
```

整个过程，将原来table中的Entry，重新进行了 散列，分布到 newTable 中。

### createEntry

``` java
void createEntry(int hash, K key, V value, int bucketIndex) {
	// 获得位置为 bucketIndex 处的链表头部的Entry
    Entry<K,V> e = table[bucketIndex];
    
    // 将头部的Entry置为新插入的Entry。
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```
## HashMap 的 get 过程

根据 key 计算出 Entry 在 table 中索引，然后循环，Entry链表，找到 key 所对应的 value.

详见： java.util.HashMap.getEntry(Object key)

## HashMap 的 remove 过程

根据 key 计算出 Entry 在 table 中索引，然后循环，Entry链表，找到 key 所对应的 Entry, 然后将这个Entry的 prev Entry 的 next 值，指向，当前 Entry的next,这样就完成了将 当前 Entry 从list 移除的动作。

## HashMap 的 clear 过程

``` java
public void clear() {
    modCount++;
    Arrays.fill(table, null);
    size = 0;
}
````

详见： java.util.HashMap.removeEntryForKey(Object key)

## hashSeed 和 hash 方法

## java.util.HashMap.Entry<K, V>

``` java
static class Entry<K,V> implements Map.Entry<K,V> 
```

这个类有四个字段：

        final K key;
        V value;
        Entry<K,V> next;
        int hash;

## ^ 和 >>> 运算

* ^ 按位异或操作符

	按位异或操作符，两个操作数的某一位不相同时候结果的该位就为1。

* '>>>'按位右移补零操作符

	按位右移补零操作符。左操作数的值按右操作数指定的位数右移，移动得到的空位以零填充。

## HashMap 的三个 Collection 视图

* values

	这个 values 是一个继承了AbstractCollection的类。这个类不支持 add 操作，如果调用这个视图的 add, addAll 方法，将会抛出 UnsupportedOperationException 异常。但是它支持：	Iterator.remove, Collection.remove, removeAll, retainAll and clear operations

* keySet

	同上，不支持 add, addAll。 但是它 支持：Iterator.remove, Set.remove, removeAll, retainAll and clear operations.

* entrySet
	
	同上，不支持 add, addAll。但是它 支持：Iterator.remove, Set.remove, removeAll, retainAll and clear operations.

这三个视图所对应的 iterator 都是 fail-fast 的。

## HashMap 对 null 的支持

add: HashMap 支持 key 为 null 的操作，它会将 key 为 null 的 放置到 table[0] 的 Entry 链中，其 hash = 0. 

get: 从 table[0]中，获取 key 为 null 的 value.

## HashMap vs Hashtable

### 线程安全

Hashtable 是线程安全的，而 HashMap 是非线程安全的。而 Hashtable 的线程同步实现的非常简洁，对于一般方法直接在方法上使用 synchronized 关键字，来实现同步，其实就是一种粗粒度的同步。在并发状况下使用，性能必然是问题。

而其对应的三个视图的同步，则是通过，Collections 类的同步方法。例如：

``` java
public Collection<V> values() {
    if (values==null)
        values = Collections.synchronizedCollection(new ValueCollection(), this);
    return values;
}
```

ValueCollection 的实现并不是线程安全的。

keySet 和 entrySet 也是如此。

[关于几个Map的线程安全问题](http://www.cnblogs.com/wuchanming/p/4762390.html)

[HashMap, Hashtable, ConcurrentHashMap 比较](http://blog.csdn.net/zhangerqing/article/details/8193118)

### hash散列算法

* Hashtable 使用： 让length为素数，然后用hashCode(key) mod length的方法得到索引

* HashMap 使用：让length为2的指数倍，然后用hashCode(key) & (length-1)的方法得到索引


**参考：**

[哈希函数的设计原理](http://my.oschina.net/u/2400412/blog/506477)

[为什么求模运算要用素数（质数）—— 哈希表设计](http://www.vvbin.com/?p=376)

[good hash table primes](http://planetmath.org/goodhashtableprimes)

[Understanding strange Java hash function](http://stackoverflow.com/questions/9335169/understanding-strange-java-hash-function)

[数据结构可视化](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

### 对 key 或 value 为 null 的支持

Hashtable 不支持 null 的 key 和 value, 在进行 put 操作时，会直接抛出 NullPointerException

HashMap 支持

[HashMap 和 HashTable 比较](http://www.codeceo.com/article/java-hashmap-hashtable-differ.html)

### iterator

HashMap 的 iterator 是 fail-fast 的

Hashtable 的 iterator 的实现类是：java.util.Hashtable.Enumerator<T>， 它除实现接口：java.util.Iterator<T>，还实现也java.util.Enumeration<T>接口。在 Jdk7 中 Enumerator 实现的 iterator 也是 fail-fast 的。貌似在以前的版本中这个 iterator 不是 fail-fast.

## 其它的 Map 实现

### LinkedHashMap

LinkedHashMap 在主要思路在创建 Entry 的时候，将所有的Entry通过，双向链表，将其串连起来，始终维护一个 header 表示这个链表的头部。每一个 Entry 内部，又多了两个状态，before, after, 

那么，这些状态是如何维护的？

#### header 的初始化

``` java
@Override
void init() {
    header = new Entry<>(-1, null, null, null);
    header.before = header.after = header;
}
```

LinkedHashMap override 了 init 方法， 初始化了 header, 并将其 before 和 after 都指向自身。

#### header 链的中元素的的创建

``` java
void createEntry(int hash, K key, V value, int bucketIndex) {
    HashMap.Entry<K,V> old = table[bucketIndex];
    Entry<K,V> e = new Entry<>(hash, key, value, old);
    table[bucketIndex] = e;
    e.addBefore(header);
    size++;
}
```

addBefore 方法将新创建的 Entry ，加入到 header 链中。

#### header 链的中元素的删除

java.util.LinkedHashMap.Entry<K, V> 中的 recordRemoval 方法会在 removeEntryForKey 和 removeMapping 方法中被调用， 所以 recordRemoval 方法会将自身从 header 链中移除。


#### header 链的的中元素的位置的调整

依据参数

``` java
private final boolean accessOrder;
```

这个参数默认初始化为 false, 它是在 java.util.LinkedHashMap.Entry.recordAccess(HashMap<K, V> m) 被用到：

``` java
void recordAccess(HashMap<K,V> m) {
    LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
    if (lm.accessOrder) {
        lm.modCount++;
        remove();
        addBefore(lm.header);
    }
}
```

而 recordAccess 方法将会在 Map中Entry中被访问时被调用，Map中Entry中被访问的时机有三个: get, put, putForNullKey. 在 HashMap的实现中， put 和 putForNullKey 在 Entry存在的情况下将修改 key 所对应的value值，所以 HashMap 将这和情况定义为对Entry的access, 但是它并没有在 get 方法中调用recordAccess，说明，HashMap的实现，并不认为 get 方法算是 access,

在 LinkedHashMap override 了 get 方法，在其中调用了recordAccess，说明LinkedHashMap的设计，认为 get 也算是对 Entry 的一种 access.

recordAccess的作用是：将最近一次访问的Entry，调整到 header 链的头部。

#### header 链的使用

上面的过程完成了 header 链的维护，而 header 链的使用，则是对应于 HashMap的性能优化。 LinkdedHashMap 优化了 两个方法： transfer 和 containsValue， 加速了遍历过程。

#### LinkedHashMap vs HashMap

LinkedHashMap 返回的 iterator 是有序的，其默认顺序元素插入的顺序，也可以是最近访问优先顺序。

HashMap 返回的 iterator 是无序的，这种无序是相对于使用者的操作来说的，例如 iterator 返回的 Entry 并没有和 put get remove 操作发生联系。但是对于 iterator 内部实现来说，其实也是“有序的”，java.util.HashMap.HashIterator 返回 entry 是按照，Entry的逻辑存储结构的顺序来返回的。

### java.util.Properties

这个类是我们常用来解析 properties 文件的类，

``` java
class Properties extends Hashtable<Object,Object>
```

它继承自 Hashtable, 这个类是线程安全的，This class is thread-safe: multiple threads can share a single Properties object without the need for external synchronization

Properties 类也可以使用方法 loadFromXML 解析如下格式的 xml文件

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
	<properties>
		<comment>testconfig</comment>
		<entry key="hello">HELLO1</entry>
		<entry key="world">WORLD1</entry>
	</properties>

### IdentityHashMap

This class implements the Map interface with a hash table, using reference-equality in place of object-equality when comparing keys (and values). In other words, in an IdentityHashMap, two keys k1 and k2 are considered equal if and only if 
	
	(k1==k2)

In normal Map implementations (like HashMap) two keys k1 and k2 are considered equal if and only if 
	
	(k1==null ? k2==null : k1.equals(k2))
	
IdentityHashMap有其特殊用途，比如序列化或者深度复制。或者记录对象代理。

#### 存储结构比较

HashMap 会将相同的 hash 的元素(key-value),存储在hash之后的 table 的 index 位置，以 Entry 链的结构存储起来，而 IdentityHashMap 横向地将，hash相同，而 key 的引用不同的元素（key-value）存储在 hash table 的索引的连续的下一个位置。其计算next index 的方法如下：

	return (i + 2 < len ? i + 2 : 0);

因为 key 和 value 各占一个slot，所以是 i + 2.

### EnumMap

如果 Map 的 key 值是，固定的 枚举（Enum）类型，则可以考虑使用 EnumMap，它对Map的实现进行了优化。所有的基本操作都是常量时间。key 不能为空，只能是声明的 枚举类型。使用方法：

``` java
EnumMap<Book, Integer> eMap = new EnumMap<Book, Integer>(Book.class);
eMap.put(Book.Math, 99);
eMap.put(Book.English, 66);
eMap.put(Book.Music, 88);
eMap.put(Book.Music, 33);
```

其中 Book 为 枚举类型。

``` java
public enum Book {
	Math,
	English,
	Music
}
```

Iterators returned by the collection views are weakly consistent: they will never throw ConcurrentModificationException and they may or may not show the effects of any modifications to the map that occur while the iteration is in progress. 

EnumMap 返回的 Iterator不是 fail-fast 的。

### WeakHashMap

和 key 的引用类型有关，关于它的使用参考：

[Java内存泄露与WeakHashMap](http://www.xiaoyaochong.net/wordpress/index.php/2013/08/05/java%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E4%B8%8Eweakhashmap/)

### TreeMap

内部使用二叉树来存储Entry。 不支持 null 的 key.

TreeMap是有序的

[HashMap、Hashtable、LinkedHashMap、和TreeMap介绍和区别](http://www.cnblogs.com/kathyrani/articles/2520709.html)

TreeMap 平均时间复杂度 O(log n)

HashMap 平均时间复杂度 O(1)