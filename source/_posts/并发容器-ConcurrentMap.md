---
title: 并发容器-ConcurrentMap
date: 2016-8-1 10:58:39
tags: [ConcurrentMap, ConcurrentHashMap]
---

## ConcurrentMap

``` java
public interface ConcurrentMap<K, V> extends Map<K, V>
```

这个接口增加了以下操作，这个方法实现都是原子操作，都是线程安全的复合操作：

``` java
// 如果key在容器中不存在则将其放入其中，否则donothing.
// 返回 null,表示确实不存在，并且value被成功放入
// 返回非 null, 表示 key 存在，返回值是key在容器中的当前值 。
V putIfAbsent(K key, V value);

boolean remove(Object key, Object value);

boolean replace(K key, V oldValue, V newValue);

V replace(K key, V value);
```

## ConcurrentHashMap

### 构造函数

``` java
// 默认的初始化值是 16, 0.75, 16
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
                         
    // 参数合法性检验                         
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    
    // 参数调整
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 将 concurrencyLevel 调整为 ssize，ssize 是 2 的幂次 
    // 并且，ssize >= concurrencyLevel
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // sshift <==> segment shift
    // ssize  <==> segment size
    // sshift = 4, ssize = 16
    this.segmentShift = 32 - sshift; // == 28
    this.segmentMask = ssize - 1;    // == 15
    
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize; // 16 / 16 = 1 
    if (c * ssize < initialCapacity)
        ++c; 
    // c = 1
    int cap = MIN_SEGMENT_TABLE_CAPACITY; // cap = 2
    while (cap < c)
        cap <<= 1;

    // create segments and segments[0]
    // cap = 2, loadFactor = 0.75, ssize = 16
    // create segments[0]
    // s0: loadFactor = 0.75,
    // s0: (int)cap * loadFactor = 2 * 0.75 = (int)1.5 = 1
    // s0: new HashEntry[2]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    // create segments
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    
    // ss[0] = s0
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```

## 添加元素的过程

* put
``` java
public V put(K key, V value) {
    Segment<K,V> s;
    // 不支持null的value
    if (value == null)
        throw new NullPointerException();
        
    // 计算key的hash值，和
    int hash = hash(key);
    
    // 依据 hash 值，计算这个Entry所在的 segment
    // 的索引。
    int j = (hash >>> segmentShift) & segmentMask;
    
    // 对全局变量 segments 使用 CAS 保证，读取是原子的，线程安全的。
    // 依据 j ，获取 segment, 使用 UNSAFE 
    // 保证线程安全。
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // 查找 索引为 j 的 segment,
        // 如果不存在，则创建它
        // 存在，则直接返回
        s = ensureSegment(j);
	
	// 调用 segment 的 put 方法，
	// 通过分段，将不同的 entry 分流到，不同的 segment 中，
	// 使得下面的 s.put 的调用可以并发执行，提高了 map 的吞吐量
    return s.put(key, hash, value, false);
}
```

* Segment.put
``` java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {

	// 如果成功获得锁 node = null
	// 否则调用 scanAndLockForPut 来，使用CAS来获取锁。
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    
    // 方法执行到这里是，必然已经获得锁了
    // 所以上面，等价于调用了： lock() 方法。
    // 但是其实现都是依靠，tryLock来获取锁，所以在并发的情况下
    // 并不会导致线程 block,而是自旋尝试来获取锁
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 1. 通过 hash 值，计算当前待插入元素的位置
        int index = (tab.length - 1) & hash;
        // 2. 返回Entry链，的第一个元素。
        HashEntry<K,V> first = entryAt(tab, index);
        // 3. 循环Entry链，如果存在key相同的元素，则用新值将
        //	  其替换，否则创建一个新node，将其插入到当前Entry链中。
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                // 判断元素是否存在，进行替换
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
            // 代表两种情况：(1)第一个元素在当前 segment的 table 的
            // 索引为 index 这个Entry 链上插入时，
            // (2) entry链遍历到最后一个结点了，（同时也说明，key在当前链中并不存在）
            // 创建结点，并将这个Entry链的第一个元素设置成，当前新的node
            // 的 next 元素，
            // 然后，将新创建的元素旋转到 tab 的 index 处
            // setEntryAt(tab, index, node);
            // 最近插入的新元素，都将被插入到Entry链的头部
            // 这和 HashMap 的put操作实现是相同的。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                // 增加元素个数计数
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                	// 为表扩容
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
    	// 释放锁。
        unlock();
    }
    return oldValue;
}
```

注意到：判断元素是否存在的条件和HashMap 有所不同：
``` java
// ConcurrentHashMap
// hash值相等成了充分条件，
// 只要 e.key == key , 就可以认定元素相同，
(k = e.key) == key || (e.hash == hash && key.equals(k))

// HashMap
// hash值相等是必要条件
e.hash == hash && ((k = e.key) == key || key.equals(k))
```

* Segment.scanAndLockForPut
``` java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    // 查找当前 segment(this),中 Hash值是 hash 的 HashEntry
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    
    // 循环直到当前 segment 获取 lock 成功，
    // 考虑，如果当前 tryLock 失败，则必然有其它线程正在操作当前的 segment
    // 所以这里通过 循环 and retry 来，查找 key.equals(e.key) 
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            // f = entryForHash(this, hash)) != first
            // 说明 entry 结构发生了变化，所以重新 从 first 开始
            // 遍历 entry 链。
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

Scans for a node containing given key while trying to acquire lock, creating and returning one if not found. Upon return, guarantees that lock is held.

这个方法的实现等价于：不断尝试获取锁，并在获取锁的过程中，遍历 entry 链，如果

``` java
if (e == null) {
    if (node == null) // speculatively create node
        node = new HashEntry<K,V>(hash, key, value, null);
    retries = 0;
}
else if (key.equals(e.key))
    retries = 0;
else
    e = e.next;
    
// 上面这段代码等价于：
for(e.next != null; e = e.next){
	if (key.equals(e.key))
		node = null;
}
// 遍历上面的entry, 直到最后一个e,还是没有找到，则创建一个。
if(e.next == null){
	node = new HashEntry<K,V>(hash, key, value, null);
}
```

完成上述操作后，再 tryLock 尝试获取锁，如果没有成功，则说明，有可能还有其它线程在操作当前这个segment,所以需要重新遍历，下面这段代码的作用就是，调整状态，重新进行遍历。

``` java
else if ((retries & 1) == 0 &&
                     (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
```

scanAndLockForPut方法在等价于下面的一个循环：

``` java
HashEntry<K,V> first = entryForHash(this, hash);
for(HashEntry<K,V> e = first; e.next != null; e = e.next){
	if(tryLock()){ // 如果锁获取成功，直接返回
		break;
	}
   if(e == null){// e如果是null,则entry链遍历完成了，表明没有找到
   // 则直接创建 
	   node =  new HashEntry<K,V>(hash, key, value, null);
   }else if(key.equals(e.key)){// 找到了返回 null
	   node = null;
   }

	// 下一次循环开始之前，判断entry 链是否发生变化了
	// 如果变化了，则调整 循环变量e ，重新开始遍历
   if(f = entryForHash(this, hash)) != first) {
       e = first = f; // re-traverse if entry changed
   }
}
```
由这个方法的实现可知，这个方法也是可以并发执行的。

* entryForHash
``` java
static final <K,V> HashEntry<K,V> entryForHash(Segment<K,V> seg, int h) {
    HashEntry<K,V>[] tab;
    // 如果是第一次put，则 tab = seg.table 一定是 null,所以返回 null
    // 如果 key 被 put 过，则，使用 (tab.length - 1) & h) 定位到 
    // Entry 在 table 中的位置。
    return (seg == null || (tab = seg.table) == null) ? null :
        (HashEntry<K,V>) UNSAFE.getObjectVolatile
        (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
}
```

## 获得元素的过程
``` java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

可见整个 get 过程并没有对 segment 进行加锁操作，关于在不加锁还可以保证并发安全的原因参考：[探索 ConcurrentHashMap 高并发性的实现机制](https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/)

## 其它方法

* replace

相当于update,如果key不存在，则不会将其添加到table中

* remove

调用对每一个segment调用其 clear 方法
``` java
final void clear() {
    lock();
    try {
    // 将Entry链和table断开，但是Entry链是完整的
        HashEntry<K,V>[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            setEntryAt(tab, i, null);
        ++modCount;
        count = 0;
    } finally {
        unlock();
    }
}
```

## Segment

``` java
static final class Segment<K,V> extends ReentrantLock implements Serializable 
```

这个类继承自 ReentrantLock ，所以其自身就是一个Lock。



## HashEntry

这个类的作用和 HashMap中Entry是一样的，用来存储 key-value 对。其中有惟一一个方法：

``` java
final void setNext(HashEntry<K,V> n) {
    UNSAFE.putOrderedObject(this, nextOffset, n);
}
```

## ConcurrentHashMap.HashIterator
``` java
HashIterator() {
	// 从 segment 的 segments.length - 1 开始向前遍历，
    nextSegmentIndex = segments.length - 1;
    nextTableIndex = -1;
    advance();
}

/**
 * Set nextEntry to first node of next non-empty table
 * (in backwards order, to simplify checks).
 */
final void advance() {
    for (;;) {
        if (nextTableIndex >= 0) {// 遍历 table
            if ((nextEntry = entryAt(currentTable,
                                     nextTableIndex--)) != null)
                break;
        }
        else if (nextSegmentIndex >= 0) {// 遍历 segment
            Segment<K,V> seg = segmentAt(segments, nextSegmentIndex--);
            if (seg != null && (currentTable = seg.table) != null)
                nextTableIndex = currentTable.length - 1;
        }
        else
            break;
    }
}

final HashEntry<K,V> nextEntry() {
    HashEntry<K,V> e = nextEntry;
    if (e == null)
        throw new NoSuchElementException();
    lastReturned = e; // cannot assign until after null check
    if ((nextEntry = e.next) == null)// 遍历 Entry 链
    	// Entry遍历结束之后，步进到下一个Entry链上。
        advance();
    return e;
}
```

iterator, values的实现都依赖于这个类。

iterator 表示的是其被创建时或被创建后 hash table的状态。

Similarly, Iterators and Enumerations return elements reflecting the state of the hash table at some point at or since the creation of the iterator/enumeration.

这个类的方法并不会在遍历过程中检查hash table是否发生结构性修改，所以不会抛出 ConcurrentModificationException 异常。

They do not throw ConcurrentModificationException. **However, iterators are designed to be used by only one thread at a time.** 

所以这个iterator 不能作为共享变量在多个线程被使用，在任意时刻，他只能是由一个线程使用。

### ConcurrentHashMap的弱一致性和迭代器的弱一致性问题

ConcurrentHashMap返回的迭代器具有弱一致性（weakly consistent），而并非“及时失败”--强一致性。弱一致性的迭代器可以容忍并发的修改。

ConcurrentHashMap的弱一致性主要是为了提升效率，是一致性与效率之间的一种权衡。要成为强一致性，就得到处使用锁，甚至是全局锁，这就与Hashtable和同步的HashMap一样了。

ConcurrentHashMap中的弱一致性方法：get, clear在实现过程中都是没有加锁的，所以它们都是弱一致性方法。当然还有iterator,在遍历过程中没有对修改进行跟踪判断，所以也是弱一致性的。

[深究HashMap以及ConcurrentHashMap的一致性问题](https://zhouhualei.github.io/2015/06/08/deep-into-consitency-of-hashmap-and-concurrenthashmap/)

[为什么ConcurrentHashMap是弱一致的](http://ifeve.com/concurrenthashmap-weakly-consistent/)