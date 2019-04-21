---
title: Unsafe类
date: 2016-8-23 21:28:02
---

## Unsafe

unsafe.cpp

``` java
// park
UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time))
  UnsafeWrapper("Unsafe_Park");
  HS_DTRACE_PROBE3(hotspot, thread__park__begin, thread->parker(), (int) isAbsolute, time);
  JavaThreadParkedState jtps(thread, time != 0);
  thread->parker()->park(isAbsolute != 0, time);
  HS_DTRACE_PROBE1(hotspot, thread__park__end, thread->parker());
UNSAFE_END

// unpark
UNSAFE_ENTRY(void, Unsafe_Unpark(JNIEnv *env, jobject unsafe, jobject jthread))
  UnsafeWrapper("Unsafe_Unpark");
  Parker* p = NULL;
  if (jthread != NULL) {
    oop java_thread = JNIHandles::resolve_non_null(jthread);
    if (java_thread != NULL) {
      jlong lp = java_lang_Thread::park_event(java_thread);
      if (lp != 0) {
        // This cast is OK even though the jlong might have been read
        // non-atomically on 32bit systems, since there, one word will
        // always be zero anyway and the value set is always the same
        p = (Parker*)addr_from_java(lp);
      } else {
        // Grab lock if apparently null or using older version of library
        MutexLocker mu(Threads_lock);
        java_thread = JNIHandles::resolve_non_null(jthread);
        if (java_thread != NULL) {
          JavaThread* thr = java_lang_Thread::thread(java_thread);
          if (thr != NULL) {
            p = thr->parker();
            if (p != NULL) { // Bind to Java thread for next time.
              java_lang_Thread::set_park_event(java_thread, addr_to_java(p));
            }
          }
        }
      }
    }
  }
  if (p != NULL) {
    HS_DTRACE_PROBE1(hotspot, thread__unpark, p);
    p->unpark();
  }
UNSAFE_END
```

从上面的实现可知，park 和 unpark 最终都调用当前 JavaThread 对象的 parker() 方法返回一个 Parker 对象，然后调用这个 Parker 对象的 park 和 unpark 方法。

thread.hpp

``` java
class JavaThread: public Thread {
	// ...
	private:
	  Parker*    _parker;
	public:
	  Parker*     parker() { return _parker; }	
	// ...
}
```

JavaThread _parker 对象 在 JavaThread::initialize()  方法中被初始化，initialize 在 JavaThread 的构造函数中被调用。

thread.cpp

``` java
void JavaThread::initialize() {
	// ...
	_parker = Parker::Allocate(this) ;
	// ...
}
```

## Parker

Parker 对象定义在 openjdk\hotspot\src\share\vm\runtime\park.hpp & park.cpp 文件中。其中 park 和 unpark 方法的实现依赖于具体的平台，所以分布在

``` java
class Parker : public os::PlatformParker {
private:
  volatile int _counter ;
  Parker * FreeNext ;
  JavaThread * AssociatedWith ; // Current association

public:
  Parker() : PlatformParker() {
    _counter       = 0 ;
    FreeNext       = NULL ;
    AssociatedWith = NULL ;
  }
protected:
  ~Parker() { ShouldNotReachHere(); }
public:
  // For simplicity of interface with Java, all forms of park (indefinite,
  // relative, and absolute) are multiplexed into one call.
  void park(bool isAbsolute, jlong time);
  void unpark();

  // Lifecycle operators
  static Parker * Allocate (JavaThread * t) ;
  static void Release (Parker * e) ;
private:
  static Parker * volatile FreeList ;
  static volatile int ListLock ;

};
```

Parker 类继承自 os::PlatformParker 其定义在 os_windows.hpp

``` java
class PlatformParker : public CHeapObj {
  protected:
    HANDLE _ParkEvent ;

  public:
    ~PlatformParker () { guarantee (0, "invariant") ; }
    PlatformParker  () {
      _ParkEvent = CreateEvent (NULL, true, false, NULL) ;
      guarantee (_ParkEvent != NULL, "invariant") ;
    }

};
```


hotspot\src\os\windows\vm\os_windows.cpp 文件中。

``` java
void Parker::park(bool isAbsolute, jlong time) {
  guarantee (_ParkEvent != NULL, "invariant") ;
  // First, demultiplex/decode time arguments
  if (time < 0) { // don't wait
    return;
  }
  else if (time == 0 && !isAbsolute) {
    time = INFINITE;
  }
  else if  (isAbsolute) {
    time -= os::javaTimeMillis(); // convert to relative time
    if (time <= 0) // already elapsed
      return;
  }
  else { // relative
    time /= 1000000; // Must coarsen from nanos to millis
    if (time == 0)   // Wait for the minimal time unit if zero
      time = 1;
  }

  JavaThread* thread = (JavaThread*)(Thread::current());
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;

  // Don't wait if interrupted or already triggered
  if (Thread::is_interrupted(thread, false) ||
    WaitForSingleObject(_ParkEvent, 0) == WAIT_OBJECT_0) {
    ResetEvent(_ParkEvent);
    return;
  }
  else {
    ThreadBlockInVM tbivm(jt);
    OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
    jt->set_suspend_equivalent();

    WaitForSingleObject(_ParkEvent,  time);
    ResetEvent(_ParkEvent);

    // If externally suspended while waiting, re-suspend
    if (jt->handle_special_suspend_equivalent_condition()) {
      jt->java_suspend_self();
    }
  }
}

void Parker::unpark() {
  guarantee (_ParkEvent != NULL, "invariant") ;
  SetEvent(_ParkEvent);
}
```

park 和 unpark 在 windows 上的实现，采用 WaitForSingleObject 在 Events 对象 _ParkEvent 等待来实现。其核心代码如下：

``` java

_ParkEvent = CreateEvent (NULL, true, false, NULL) ;

void Parker::park(bool isAbsolute, jlong time) {
	// 当前线程将在 _ParkEvent 事件对象上 等待
	WaitForSingleObject(_ParkEvent,  time);
	// 上面的函数返回之后，调用 ResetEvent，
	// 使得 _ParkEvent 事件对象可以再次被使用。
	// 线程可以多次调用 park.
    ResetEvent(_ParkEvent);
}

void Parker::unpark() {
	// 激发 _ParkEvent，使得在其 wait 的线程从 wait 状态返回。
	SetEvent(_ParkEvent);
}

```

CreateEvent win32 平台上创建事件对象API.第二个参数为 true 表示创建一个手动重置事件，即当 event 对象被 signal 后，需要手动调用 ResetEvent，使其状态恢复到 nonsignaled. 第三个参数为 false,表示创建的这个 event 对象，初始状态是 nonsignaled 的。

当线程在事件对象 _ParkEvent 上调用 WaitForSingleObject 由于 _ParkEvent初始状态是 nonsignaled 所以当前线程将进入 blocking 状态，直到另一个线程在 _ParkEvent 上调用 SetEvent(_ParkEvent)，使得其 signed. 然后 wait 线程就会醒来，继续执行。调用 ResetEvent 恢复 _ParkEvent 的 nonsignaled 状态。这样 _ParkEvent 对象就可以重新被 park 方法使用了。


## Unsafe的 park & unpark 方法在 win32 平台上的实现

对于每一个 java 线程，在 JVM 内部有一个 JavaThread 的对象与之相对应，这个对象内部有一个 Parker 对象，这个对象有 park 和 unpark 方法，这两个方法的实现依赖于具体的平台。在 windows 平台上使用 WaitForSingleObject 使得线程进入等待状态。这个方法最终在一个事件对象上等待。unpark 将激发这个事件对象，使得线程被唤醒。

当一个线程调用 Unsafe.park 方法时， Unsafe_Park 方法将使用当前线程的 JavaThread 对象中的 Parker 对象的 park 方法使得该线程处于 block 状态。

``` java
// 摘自 Unsafe_Park （unsafe.cpp）
thread->parker()->park(isAbsolute != 0, time);
```

当一个线程调用 Unsafe.unpark 方法时，Unsafe_Unpark 将使用 unpark 的参数 jthread. 获得该 jthread 的 JavaThread 对象，然后调用该对象的 parker 的 unpark.使得 jthread 对应的线程 unblocking.

``` java
// 摘自 Unsafe_Unpark （unsafe.cpp）
java_thread = JNIHandles::resolve_non_null(jthread);
JavaThread* thr = java_lang_Thread::thread(java_thread);
p = thr->parker();
p->unpark();
```

## CAS

Unsafe提供的 CAS 机制，用来实现 Atomic 类操作。

### objectFieldOffset

这个方法来获取任意 java 类的某个字段 field 在内存中的相对于 这个 Java 类的所有字段的 偏移量:

``` java
// 摘自 AtomicInteger 类
valueOffset = unsafe.objectFieldOffset
	(AtomicInteger.class.getDeclaredField("value"));
```

表示 AtomicInteger 的 value 字段的内存中相对偏移量。

其实现过程如下：

``` c++
// src/share/vm/prims/Unsafe.cpp
UNSAFE_ENTRY(jlong, Unsafe_ObjectFieldOffset(JNIEnv *env, jobject unsafe, jobject field))
  UnsafeWrapper("Unsafe_ObjectFieldOffset");
  // field 参数是：
  // AtomicInteger.class.getDeclaredField("value")
  // 表示 value 字段的 Field 对象的地址
  // THREAD 参数是： 当前线程对象 JavaThread
  return find_field_offset(field, 0, THREAD);
UNSAFE_END


jint find_field_offset(jobject field, int must_be_static, TRAPS) {
  if (field == NULL) {
    THROW_0(vmSymbols::java_lang_NullPointerException());
  }
  
  // JNIHandles::resolve_non_null(field) 方法直接将 转换成 oop 对象
  // 也间接证明了 field 其实就是一个 指向 Field oop 对象的指针
  // 同时也说明 Field f = AtomicInteger.class.getDeclaredField("value")
  // 这个调用返回的所谓 f 对象，其实就是一个指向 Filed 底层 oop 对象的指针。
  oop reflected   = JNIHandles::resolve_non_null(field);
  oop mirror      = java_lang_reflect_Field::clazz(reflected);
  klassOop k      = java_lang_Class::as_klassOop(mirror);
  int slot        = java_lang_reflect_Field::slot(reflected);
  int modifiers   = java_lang_reflect_Field::modifiers(reflected);

  if (must_be_static >= 0) {
    int really_is_static = ((modifiers & JVM_ACC_STATIC) != 0);
    if (must_be_static != really_is_static) {
      THROW_0(vmSymbols::java_lang_IllegalArgumentException());
    }
  }
  
  // 到这里，其实可以看到，这个方法的实现，和 Java类及对象在vm层的实现有关
  // 也就是 oop-Klass 模型。依据这些底层结构之间的关联（指针）。自然可以
  // 字段所对应的 klass 找到 字段 在 类 内的偏移量。

  // 在这里调用了字段所在类的 Klass 的 offset_from_fields 方法获取
  // 字段所在的偏移量。至于 offset_from_fields 这个方法的实现，就和
  // oop-klass 模型如何表示 Java 对象 及 类 相关了。
  // oop-klass 模型 约定了 内存布局及规范，offset_from_fields
  // 方法实现起来亦不难
  int offset = instanceKlass::cast(k)->offset_from_fields(slot);

  // field_offset_from_byte_offset 这个方法调用是只是将 
  // 参数 offset 直接返回了。所以上面的 offset 就是最终的结果。
  return field_offset_from_byte_offset(offset);
}
```

### compareAndSwapInt

``` c++
// 这个一个 java 调用 
unsafe.compareAndSwapInt(this, valueOffset, expect, update);
// 对应的JVM实现如下：

// 其中 obj <==> this, offset <==> valueOffset, 
//		e <==> expect, x <==> update
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  // 获取 this 对象的底层 oop 对象
  oop p = JNIHandles::resolve(obj);
  // 获取 this 对象的 offset 处的字段的地址。
  // index_oop_from_field_offset_long 方法实现的核心是：
  // p + offset; 就是 当前对象的地址 + 字段所处的偏移量
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);

  // 最后真正完成功能就是下面的调用
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```

Atomic::cmpxchg 功能的实现则是利用具体系统和CPU硬件指令的支持来完成的
在 windows x86 架构下在其实现如下：

``` c++
// hotspot\src\os_cpu\windows_x86\vm\atomic_windows_x86.inline.hpp

// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

#ifdef AMD64
// 如果是 AMD64 架构的CPU则使用下面的实现：
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  return (*os::atomic_cmpxchg_func)(exchange_value, dest, compare_value);
}

#else // !AMD64
// 如果不是 AMD64 架构的CPU则使用下面的实现：
// 其中使用到 cmpxchg 指令，这是 CPU 提供的完成 CAS 功能的指令。
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}

#endif // AMD64
```

从上面的实现可知，其实 Unsafe 提供的 CAS 机制，最终是由 CPU 和 系统协作完成的，如果 CPU 提供了 CAS 指令，则可以直接利用 CPU 的 CAS 指令。如果没有，则可以交由操作系统来实现。

cmpxchg：这个指令是CPU提供的用于支持 CAS 操作。

## 有关内存的操作

### allocateMemory

由 malloc 实现：

`void* malloc (size_t size);`

> Allocates a block of size bytes of memory, returning a pointer to the beginning of the block.
> 
> The content of the newly allocated block of memory is not initialized, remaining with indeterminate values.

``` c
void* os::malloc(size_t size) {
  NOT_PRODUCT(inc_stat_counter(&num_mallocs, 1));
  NOT_PRODUCT(inc_stat_counter(&alloc_bytes, size));

  if (size == 0) {
    // return a valid pointer if size is zero
    // if NULL is returned the calling functions assume out of memory.
    size = 1;
  }

  NOT_PRODUCT(if (MallocVerifyInterval > 0) check_heap());
  u_char* ptr = (u_char*)::malloc(size + space_before + space_after);
#ifdef ASSERT
  if (ptr == NULL) return NULL;
  if (MallocCushion) {
    for (u_char* p = ptr; p < ptr + MallocCushion; p++) *p = (u_char)badResourceValue;
    u_char* end = ptr + space_before + size;
    for (u_char* pq = ptr+MallocCushion; pq < end; pq++) *pq = (u_char)uninitBlockPad;
    for (u_char* q = end; q < end + MallocCushion; q++) *q = (u_char)badResourceValue;
  }
  // put size just before data
  *size_addr_from_base(ptr) = size;
#endif
  u_char* memblock = ptr + space_before;
  if ((intptr_t)memblock == (intptr_t)MallocCatchPtr) {
    tty->print_cr("os::malloc caught, " SIZE_FORMAT " bytes --> " PTR_FORMAT, size, memblock);
    breakpoint();
  }
  debug_only(if (paranoid) verify_block(memblock));
  if (PrintMalloc && tty != NULL) tty->print_cr("os::malloc " SIZE_FORMAT " bytes --> " PTR_FORMAT, size, memblock);
  return memblock;
}
```


### reallocateMemory

`void* realloc (void* ptr, size_t size);`

> Changes the size of the memory block pointed to by ptr.

``` c
void* os::realloc(void *memblock, size_t size) {
#ifndef ASSERT
  NOT_PRODUCT(inc_stat_counter(&num_mallocs, 1));
  NOT_PRODUCT(inc_stat_counter(&alloc_bytes, size));
  return ::realloc(memblock, size);
#else
  if (memblock == NULL) {
    return malloc(size);
  }
  if ((intptr_t)memblock == (intptr_t)MallocCatchPtr) {
    tty->print_cr("os::realloc caught " PTR_FORMAT, memblock);
    breakpoint();
  }
  verify_block(memblock);
  NOT_PRODUCT(if (MallocVerifyInterval > 0) check_heap());
  if (size == 0) return NULL;
  // always move the block
  void* ptr = malloc(size);
  if (PrintMalloc) tty->print_cr("os::remalloc " SIZE_FORMAT " bytes, " PTR_FORMAT " --> " PTR_FORMAT, size, memblock, ptr);
  // Copy to new memory if malloc didn't fail
  if ( ptr != NULL ) {
    memcpy(ptr, memblock, MIN2(size, get_size(memblock)));
    if (paranoid) verify_block(ptr);
    if ((intptr_t)ptr == (intptr_t)MallocCatchPtr) {
      tty->print_cr("os::realloc caught, " SIZE_FORMAT " bytes --> " PTR_FORMAT, size, ptr);
      breakpoint();
    }
    free(memblock);
  }
  return ptr;
#endif
}
```

### setMemory	

`void * memset ( void * ptr, int value, size_t num );`

> Sets the first num bytes of the block of memory pointed by ptr to the specified value

``` c
// Fill bytes; larger units are filled atomically if everything is aligned.
void Copy::fill_to_memory_atomic(void* to, size_t size, jubyte value) {
  address dst = (address) to;
  uintptr_t bits = (uintptr_t) to | (uintptr_t) size;
  if (bits % sizeof(jlong) == 0) {
    jlong fill = (julong)( (jubyte)value ); // zero-extend
    if (fill != 0) {
      fill += fill << 8;
      fill += fill << 16;
      fill += fill << 32;
    }
    //Copy::fill_to_jlongs_atomic((jlong*) dst, size / sizeof(jlong));
    for (uintptr_t off = 0; off < size; off += sizeof(jlong)) {
      *(jlong*)(dst + off) = fill;
    }
  } else if (bits % sizeof(jint) == 0) {
    jint fill = (juint)( (jubyte)value ); // zero-extend
    if (fill != 0) {
      fill += fill << 8;
      fill += fill << 16;
    }
    //Copy::fill_to_jints_atomic((jint*) dst, size / sizeof(jint));
    for (uintptr_t off = 0; off < size; off += sizeof(jint)) {
      *(jint*)(dst + off) = fill;
    }
  } else if (bits % sizeof(jshort) == 0) {
    jshort fill = (jushort)( (jubyte)value ); // zero-extend
    fill += fill << 8;
    //Copy::fill_to_jshorts_atomic((jshort*) dst, size / sizeof(jshort));
    for (uintptr_t off = 0; off < size; off += sizeof(jshort)) {
      *(jshort*)(dst + off) = fill;
    }
  } else {
    // Not aligned, so no need to be atomic.
    // 内部使用 memset 实现
    Copy::fill_to_bytes(dst, size, value);
  }
}
```


### copyMemory

`void * memmove ( void * destination, const void * source, size_t num );`

> Copies the values of num bytes from the location pointed by source to the memory block pointed by destination. Copying takes place as if an intermediate buffer were used, allowing the destination and source to overlap.

``` c
// Copy bytes; larger units are filled atomically if everything is aligned.
void Copy::conjoint_memory_atomic(void* from, void* to, size_t size) {
  address src = (address) from;
  address dst = (address) to;
  uintptr_t bits = (uintptr_t) src | (uintptr_t) dst | (uintptr_t) size;

  // (Note:  We could improve performance by ignoring the low bits of size,
  // and putting a short cleanup loop after each bulk copy loop.
  // There are plenty of other ways to make this faster also,
  // and it's a slippery slope.  For now, let's keep this code simple
  // since the simplicity helps clarify the atomicity semantics of
  // this operation.  There are also CPU-specific assembly versions
  // which may or may not want to include such optimizations.)

  if (bits % sizeof(jlong) == 0) {
    Copy::conjoint_jlongs_atomic((jlong*) src, (jlong*) dst, size / sizeof(jlong));
  } else if (bits % sizeof(jint) == 0) {
    Copy::conjoint_jints_atomic((jint*) src, (jint*) dst, size / sizeof(jint));
  } else if (bits % sizeof(jshort) == 0) {
    Copy::conjoint_jshorts_atomic((jshort*) src, (jshort*) dst, size / sizeof(jshort));
  } else {
    // Not aligned, so no need to be atomic.
    // 内部使用 memmove 实现
    Copy::conjoint_jbytes((void*) src, (void*) dst, size);
  }
}
```

### freeMemory

`void free (void* ptr);`

> A block of memory previously allocated by a call to malloc, calloc or realloc is deallocated, making it available again for further allocations.

``` c
void  os::free(void *memblock) {
  NOT_PRODUCT(inc_stat_counter(&num_frees, 1));
#ifdef ASSERT
  if (memblock == NULL) return;
  if ((intptr_t)memblock == (intptr_t)MallocCatchPtr) {
    if (tty != NULL) tty->print_cr("os::free caught " PTR_FORMAT, memblock);
    breakpoint();
  }
  verify_block(memblock);
  NOT_PRODUCT(if (MallocVerifyInterval > 0) check_heap());
  // Added by detlefs.
  if (MallocCushion) {
    u_char* ptr = (u_char*)memblock - space_before;
    for (u_char* p = ptr; p < ptr + MallocCushion; p++) {
      guarantee(*p == badResourceValue,
                "Thing freed should be malloc result.");
      *p = (u_char)freeBlockPad;
    }
    size_t size = get_size(memblock);
    inc_stat_counter(&free_bytes, size);
    u_char* end = ptr + space_before + size;
    for (u_char* q = end; q < end + MallocCushion; q++) {
      guarantee(*q == badResourceValue,
                "Thing freed should be malloc result.");
      *q = (u_char)freeBlockPad;
    }
    if (PrintMalloc && tty != NULL)
      fprintf(stderr, "os::free " SIZE_FORMAT " bytes --> " PTR_FORMAT "\n", size, (uintptr_t)memblock);
  } else if (PrintMalloc && tty != NULL) {
    // tty->print_cr("os::free %p", memblock);
    fprintf(stderr, "os::free " PTR_FORMAT "\n", (uintptr_t)memblock);
  }
#endif
  ::free((char*)memblock - space_before);
}
```