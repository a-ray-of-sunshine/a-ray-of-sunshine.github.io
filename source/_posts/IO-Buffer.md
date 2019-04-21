---
title: Buffer
date: 2016-10-17 15:31:25
---

## Buffer

```
						  capacity
                              ↓
 _______________________________________________________________
↓                                                               ↓
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
|   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
        ↑                   ↑                       ↑
      mark               position                 limit
```

一个 buffer 有一个四个状态，并且这四个状态遵循下面的不变式：

```
0 <= mark <= position <= limit <= capacity 
```

* capacity

	> A buffer's capacity is the number of elements it contains. The capacity of a buffer is never negative and never changes. 
	
* limit

	> A buffer's limit is the index of the first element that should not be read or written. A buffer's limit is never negative and is never greater than its capacity.
	
* position

	> A buffer's position is the index of the next element to be read or written. A buffer's position is never negative and is never greater than its limit. 
	
* mark

	> A buffer's mark is the index to which its position will be reset when the reset method is invoked. The mark is not always defined, but when it is defined it is never negative and is never greater than the position.
	>
	> **If the mark is defined then it is discarded when the position or the limit is adjusted to a value smaller than the mark.**

### Buffer 状态查询及设置

1. capacity

	创建的时候，就固定了不会再改变。由 `capacity()` 方法可以对 Buffer 的 capacity 进行查询。
	
2. limit

	使用 `limit()` 进行查询。使用 `limit(int newLimit)` 进行设置。这个设置会确保，不变式成立：
	
	``` java
    public final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }
	```
	
3. position

	使用 `position()` 进行查询。使用 `position(int newPosition)` 进行设置，同时也会确保不变式成立：
	
	``` java
	public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }
	```
	
4. mark

	``` java
	// Sets this buffer's mark at its position.
	// 将 mark 标记为当前的读写位置。
	public final Buffer mark() {
        mark = position;
        return this;
    }

	// Resets this buffer's position to the previously-marked position.
	// 将 position 设置成最近一次 mark 的位置。
	public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
	```

### Buffer 中的剩余数据

hasRemaining & remaining

``` java
// 是否存在剩余
public final boolean hasRemaining() {
    return position < limit;
}

// 剩余数量
public final int remaining() {
    return limit - position;
}
```

### Clearing, flipping, and rewinding 

* clear

	> makes a buffer ready for a new sequence of channel-read or relative put operations: It sets the limit to the capacity and the position to zero. 

	``` java
	public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
	```
	
	对 Buffer 执行 clear 方法，从而重置 Buffer 的状态。此时 channel 就可以执行 read 方法，将数据缓存到 Buffer 中。
	
	``` java
	// 通过 clear 重置状态位
	buf.clear();     // Prepare buffer for reading
	// channel 会写入数据到 buf 中
 	in.read(buf);    // Read data
	```
	
* flip

	> makes a buffer ready for a new sequence of channel-write or relative get operations: It sets the limit to the current position and then sets the position to zero. 
	
	``` java
	public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
	```
	
	对 Buffer 执行 flip 方法后，channel 就可以将 Buffer 中的数据进行 write 操作了。
	
	``` java
	// 准备数据（向 Buffer 中写入数据）
	buf.put(magic);    // Prepend header
	in.read(buf);      // Read data into rest of buffer
	
	// 执行 flip 操作，准备开始读取
	buf.flip();        // Flip buffer
	// 将 buf 中的数据写入到 out（channel） 中
	out.write(buf);    // Write header + data to channel
	```
	
* rewind

	> makes a buffer ready for re-reading the data that it already contains: It leaves the limit unchanged and sets the position to zero. 
	
	``` java
	public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
	```
	
	如果需要再次读取已经在 Buffer 中存在的数据，则应该调用 rewind 方法。
	
	``` java
	// 将 buf 中的数据写入到 out（channel） 中
	out.write(buf);    // Write remaining data
	// 对 buf 进行 rewind (倒回) 操作，其使用可以
	// 再次被读取
	buf.rewind();      // Rewind buffer
	buf.get(array);    // Copy data into array
	```

## Buffer的子类

Buffer 的底层使用数组来存储数据。所以对应的有7种原生数据类型的 Buffer，内部持有对应的数据类型的数组，用来存储数据。

* Buffer
	* ByteBuffer
	* CharBuffer
	* ShortBuffer
	* IntBuffer
	* LongBuffer
	* FloatBuffer
	* DoubleBuffer

### Buffer的创建

通常 Buffer 会提供下面几种方法来创建一个Buffer的实例：

* allocate

	静态方法，提供一个capacity参数，从java heap分配内存空间，创建数组。

* allocateDirect

	静态方法，提供一个capacity参数，从由 Unsafe类的 `unsafe.allocateMemory(size)` 直接分配内存，以数组的形式访问。
	
* wrap

	静态方法，将一个已经存在的数组包装成 Buffer 类型。

* asReadOnlyBuffer

	实例方法。创建一个新的只读的Buffer。这个Buffer来原来的Buffer共享底层的数组。不过这个新创建的Buffer是**只读**的。任何向 Buffer 进行的 put 操作将，直接抛出 ReadOnlyBufferException 异常。