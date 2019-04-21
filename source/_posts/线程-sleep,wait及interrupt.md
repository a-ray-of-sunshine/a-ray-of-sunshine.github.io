---
title: 线程-sleep,wait及interrupt
date: 2016-8-24 22:30:56
---

## Thread.sleep

###　JVM_Sleep

jvm.cpp

``` c++
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  JVMWrapper("JVM_Sleep");

  if (millis < 0) {
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }

  if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) {
    THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
  }

  // Save current thread state and restore it at the end of this block.
  // And set new thread state to SLEEPING.
  JavaThreadSleepState jtss(thread);

  HS_DTRACE_PROBE1(hotspot, thread__sleep__begin, millis);

  if (millis == 0) {
    // When ConvertSleepToYield is on, this matches the classic VM implementation of
    // JVM_Sleep. Critical for similar threading behaviour (Win32)
    // It appears that in certain GUI contexts, it may be beneficial to do a short sleep
    // for SOLARIS
    if (ConvertSleepToYield) {
      os::yield();
    } else {
      ThreadState old_state = thread->osthread()->get_state();
      thread->osthread()->set_state(SLEEPING);
	  // 线程在 sleep 过程中不响应中断
      os::sleep(thread, MinSleepInterval, false);
      thread->osthread()->set_state(old_state);
    }
  } else {
    ThreadState old_state = thread->osthread()->get_state();
    thread->osthread()->set_state(SLEEPING);
    if (os::sleep(thread, millis, true) == OS_INTRPT) {
	   // 线程被中断了，
      // An asynchronous exception (e.g., ThreadDeathException) could have been thrown on
      // us while we were sleeping. We do not overwrite those.
      if (!HAS_PENDING_EXCEPTION) {
        HS_DTRACE_PROBE1(hotspot, thread__sleep__end,1);
        // TODO-FIXME: THROW_MSG returns which means we will not call set_state()
        // to properly restore the thread state.  That's likely wrong.
		// 抛出线程中断异常 InterruptedException
        THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
      }
    }
    thread->osthread()->set_state(old_state);
  }
  HS_DTRACE_PROBE1(hotspot, thread__sleep__end,0);
JVM_END
```

由上面的代码可知，最终调用 os::sleep 。
os::sleep(thread, millis, true)

### os::sleep

os_windows.cpp

``` c++
int os::sleep(Thread* thread, jlong ms, bool interruptable) {
  jlong limit = (jlong) MAXDWORD;

  while(ms > limit) {
    int res;
    if ((res = sleep(thread, limit, interruptable)) != OS_TIMEOUT)
      return res;
    ms -= limit;
  }

  assert(thread == Thread::current(),  "thread consistency check");
  OSThread* osthread = thread->osthread();
  OSThreadWaitState osts(osthread, false /* not Object.wait() */);
  int result;
  if (interruptable) {
    assert(thread->is_Java_thread(), "must be java thread");
    JavaThread *jt = (JavaThread *) thread;
    ThreadBlockInVM tbivm(jt);

    jt->set_suspend_equivalent();
    // cleared by handle_special_suspend_equivalent_condition() or
    // java_suspend_self() via check_and_wait_while_suspended()

    HANDLE events[1];
    events[0] = osthread->interrupt_event();
    HighResolutionInterval *phri=NULL;
    if(!ForceTimeHighResolution)
      phri = new HighResolutionInterval( ms );
    if (WaitForMultipleObjects(1, events, FALSE, (DWORD)ms) == WAIT_TIMEOUT) {
      result = OS_TIMEOUT;
    } else {
      ResetEvent(osthread->interrupt_event());
      osthread->set_interrupted(false);
      result = OS_INTRPT;
    }
    delete phri; //if it is NULL, harmless

    // were we externally suspended while we were waiting?
    jt->check_and_wait_while_suspended();
  } else {
    assert(!thread->is_Java_thread(), "must not be java thread");
    Sleep((long) ms);
    result = OS_TIMEOUT;
  }
  return result;
}
```

调用 WaitForMultipleObjects(1, events, FALSE, (DWORD)ms) 实现等待。

## Thread.yield

### JVM_Yield

``` c++
JVM_ENTRY(void, JVM_Yield(JNIEnv *env, jclass threadClass))
  JVMWrapper("JVM_Yield");
  if (os::dont_yield()) return;
  HS_DTRACE_PROBE0(hotspot, thread__yield);
  // When ConvertYieldToSleep is off (default), this matches the classic VM use of yield.
  // Critical for similar threading behaviour
  if (ConvertYieldToSleep) {
    os::sleep(thread, MinSleepInterval, false);
  } else {
    os::yield();
  }
JVM_END
```

### os::yield

``` c++
os::YieldResult os::NakedYield() {
  // Use either SwitchToThread() or Sleep(0)
  // Consider passing back the return value from SwitchToThread().
  if (os::Kernel32Dll::SwitchToThreadAvailable()) {
    return SwitchToThread() ? os::YIELD_SWITCHED : os::YIELD_NONEREADY ;
  } else {
    Sleep(0);
  }
  return os::YIELD_UNKNOWN ;
}

void os::yield() {  os::NakedYield(); }
```

调用 win32 API Sleep 来实现。

### Sleep & WaitForMultipleObjects

这两个函数都是 windows 在实现 JVM 的线程 sleep 和 yelid 操作时使用的 win32 API.

对于实现，线程的挂起（sleep），等待（wait）这些都是由操作系统提供的线程调度功能实现。线程调度器如果提供了这样的接口，自然线程就可以 Sleep 或 wait. 如果操作系统的线程调度器没有提供这些功能，自然也就无法实现了。

[WaitForMultipleObjects](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms687025(v=vs.85).aspx) 函数的主要功能是使当前线程等待事件（lpHandles）的触发，然后在指定的时间段内如果没有触发，则该函数就返回。
``` c++
DWORD WINAPI WaitForMultipleObjects(
  _In_       DWORD  nCount,
  _In_ const HANDLE *lpHandles,
  _In_       BOOL   bWaitAll,
  _In_       DWORD  dwMilliseconds
);
```
如果参数 dwMilliseconds 是 0：If dwMilliseconds is zero, the function does not enter a wait state if the specified objects are not signaled; it always returns immediately. 这个函数并不会进入 wait 状态，即使此时 lpHandles 指向的对象还没有被 signaled, 并且函数将立即返回。所以当  
dwMilliseconds = 0 时，函数不会放弃当前的执行时间片，而会立即返回后，正常继续向下执行。

[Sleep](https://msdn.microsoft.com/zh-cn/library/windows/desktop/ms686298(v=vs.85).aspx) Suspends the execution of the current thread until the time-out interval elapses. 主要作用就是使当前线程等待这个时间段（dwMilliseconds）过去。然后返回。

``` c++
VOID WINAPI Sleep(
  _In_ DWORD dwMilliseconds
);
```
当参数 dwMilliseconds 是 0：
A value of zero causes the thread to relinquish the remainder of its time slice to any other thread that is ready to run. If there are no other threads ready to run, the function returns immediately, and the thread continues execution.

当导致当前线程放弃当前的执行时间片，而尝试重新竞争执行。所以 java.lang.Thread.yield 方法就使用这个方法（Sleeping）来实现放弃当前执行的时间片。

## Object.wait


### JVM_MonitorWait

``` c++
// Object.wait
JVM_ENTRY(void, JVM_MonitorWait(JNIEnv* env, jobject handle, jlong ms))
  JVMWrapper("JVM_MonitorWait");
  // THREAD 是一个全局变量定义在 runtime\sharedRuntime.cpp 文件中:
  // Thread* THREAD = JavaThread::current();
  // 由此可知 THREAD 表示当前线程。
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  assert(obj->is_instance() || obj->is_array(), "JVM_MonitorWait must apply to an object");
  JavaThreadInObjectWaitState jtiows(thread, ms != 0);
  if (JvmtiExport::should_post_monitor_wait()) {
    JvmtiExport::post_monitor_wait((JavaThread *)THREAD, (oop)obj(), ms);
  }
  // 调用 ObjectSynchronizer 进行 wait.
  ObjectSynchronizer::wait(obj, ms, CHECK);
JVM_END

// Object.notify
JVM_ENTRY(void, JVM_MonitorNotify(JNIEnv* env, jobject handle))
  JVMWrapper("JVM_MonitorNotify");
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  assert(obj->is_instance() || obj->is_array(), "JVM_MonitorNotify must apply to an object");
  ObjectSynchronizer::notify(obj, CHECK);
JVM_END

// Object.notifyAll
JVM_ENTRY(void, JVM_MonitorNotifyAll(JNIEnv* env, jobject handle))
  JVMWrapper("JVM_MonitorNotifyAll");
  Handle obj(THREAD, JNIHandles::resolve_non_null(handle));
  assert(obj->is_instance() || obj->is_array(), "JVM_MonitorNotifyAll must apply to an object");
  ObjectSynchronizer::notifyall(obj, CHECK);
JVM_END
```

### ObjectSynchronizer::wait

openjdk\hotspot\src\share\vm\runtime\synchronizer.cpp

``` c++
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    TEVENT (wait - throw IAX) ;
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  monitor->wait(millis, true, THREAD);

  /* This dummy call is in place to get around dtrace bug 6254741.  Once
     that's fixed we can uncomment the following line and remove the call */
  // DTRACE_MONITOR_PROBE(waited, monitor, obj(), THREAD);
  dtrace_waited_probe(monitor, obj, THREAD);
}
```

其中调用 ObjectMonitor::wait 方法。

### ObjectMonitor::wait

openjdk\hotspot\src\share\vm\runtime\objectMonitor.cpp

这个文件中定义了 ObjectMonitor 实现了 wait ， notify 方法。 这个 ObjectMonitor 类应该就是 Object 进行同步时的 monitor, 每一个 java Object 都有一个 monitor.

ObjectMonitor类定义的一些方法和字段：
``` c++
  // 获取 monitor(成为 monitor 的 owner) 
  bool      try_enter (TRAPS) ;
  void      enter(TRAPS);
  // 释放 monitor
  void      exit(TRAPS);
  // 在锁上等待。
  void      wait(jlong millis, bool interruptable, TRAPS);
  // 激发锁。
  void      notify(TRAPS);
  void      notifyAll(TRAPS);
  
protected:                         // protected for jvmtiRawMonitor
  void *  volatile _owner;          // pointer to owning thread OR BasicLock
  volatile intptr_t  _recursions;   // recursion count, 0 for first entry
```
这个类的实现和 JAVA 中的 java.util.concurrent.locks.AbstractQueuedSynchronizer.AbstractQueuedSynchronizer 类的实现非常相似。这些字段和方法都有相互对应的。

ObjectWaiter（objectMonitor.hpp 中定义） 对应  AbstractQueuedSynchronizer.Node 表示等待的线程。

``` c++
// 请求锁时的阻塞队列
ObjectWaiter * volatile _EntryList ;     // Threads blocked on entry or reentry.

// wait 的阻塞队列
ObjectWaiter * volatile _WaitSet; // LL of threads wait()ing on the monitor
```

这两个队列在 AbstractQueuedSynchronizer 类都有相应的实现。

## 线程中断

### JVM_Interrupt

``` c++
JVM_ENTRY(void, JVM_Interrupt(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_Interrupt");

  // Ensure that the C++ Thread and OSThread structures aren't freed before we operate
  oop java_thread = JNIHandles::resolve_non_null(jthread);
  MutexLockerEx ml(thread->threadObj() == java_thread ? NULL : Threads_lock);
  // We need to re-resolve the java_thread, since a GC might have happened during the
  // acquire of the lock
  JavaThread* thr = java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread));
  if (thr != NULL) {
    Thread::interrupt(thr);
  }
JVM_END
```

### Thread::interrupt

``` c++
void Thread::interrupt(Thread* thread) {
  trace("interrupt", thread);
  debug_only(check_for_dangling_thread_pointer(thread);)
  os::interrupt(thread);
}
```

### os::interrupt

``` c++
void os::interrupt(Thread* thread) {
  assert(!thread->is_Java_thread() || Thread::current() == thread || Threads_lock->owned_by_self(),
         "possibility of dangling Thread pointer");
	
  // 获得 thread 所对应的 win32 底层的线程对象
  OSThread* osthread = thread->osthread();
  // 设置其中断状态。
  osthread->set_interrupted(true);
  // More than one thread can get here with the same value of osthread,
  // resulting in multiple notifications.  We do, however, want the store
  // to interrupted() to be visible to other threads before we post
  // the interrupt event.
  OrderAccess::release();

  // 下面是三种不同的机制使得被中断的线程被唤醒
  // 确保那些被阻塞的线程能够被唤醒。
  // (1) win32 使用 interrupt_event 使用 signaled.
  // Thread.Sleep 方法就是在 osthread->interrupt_event() 上wait的
  // 所以如果。thread 调用了 Thread.sleep ，则此时下面的调用将会使用其
  // 被唤醒。
  SetEvent(osthread->interrupt_event());
  // For JSR166:  unpark after setting status
  // (2) 对于 java 线程，有可能在其 parker 上 block
  // 所以使用 下面的语句唤醒。
  if (thread->is_Java_thread())
    ((JavaThread*)thread)->parker()->unpark();
  // (3) 线程可能由于 synchronized 语句，处于 block
  // 所以 使用 ev->unpark() 来唤醒。
  ParkEvent * ev = thread->_ParkEvent ;
  if (ev != NULL) ev->unpark() ;
}
```

从上面的实现可知 java.lang.Thread.interrupt() 方法调用的功能是：

``` java
// 当前线程 T1
T2.interrupt();
```
如果 线程T1 在 线程 T2 上调用 interrupt 方法。则：

1. T2线程的底层系统线程的 _interrupted 被设置成 ture
	
	``` c++
	osthread->set_interrupted(true);
	```
	
2. T1 线程将调用三种唤醒 T2 线程的方法，确保 可能 处于 block 状态的 T2 线程能够被唤醒。

	``` c++
	SetEvent(osthread->interrupt_event());
	((JavaThread*)thread)->parker()->unpark();
	ev->unpark() ;
	```

这两个件事做完之后，`T2.interrupt()`方法就返回了，T1线程继续向下执行，而此时 T2 就可能从 block 状态唤醒了（有可能 T1 本身就没有处于 block 状态），那么对于由于被T1线程中断的线程 T2 而醒来的线程，会怎样牌这个中断呢？这取决于不同的阻塞操作的实现。Thread.sleep , Object.wait 都是可中断的，线程T2唤醒之后，就会有不同的中断响应:

* sleep

``` c++
THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted")
```

* wait

``` c++
THROW(vmSymbols::java_lang_InterruptedException());
```

两者都是抛出 java.lang.InterruptedException 异常，并且将中断状态位清除掉。

对于 java 线程这边，对于中断处理有两种机制：

1. 抛出 java.lang.InterruptedException 异常
2. isInterrupted 方法返回中断状态为 true.

一般来说响应中断操作，可以使用上面的两种方法来表达，并且，是互斥的。例如 Thread.sleep 选择使用 抛出异常的方法来表达线程被中断了，所以就将线程的中断状态位设置成 false. 此时线程调用 isInterrupted 返回false.

对于可以被中断的方法如何响应中断，可以参考文档：java.lang.Thread.interrupt 的文档，其中有详细的描述。

### os::is_interrupted

判断线程是否被中断
``` java
bool os::is_interrupted(Thread* thread, bool clear_interrupted) {
  assert(!thread->is_Java_thread() || Thread::current() == thread || Threads_lock->owned_by_self(),
         "possibility of dangling Thread pointer");

  OSThread* osthread = thread->osthread();
  // 返回线程中断状态。
  bool interrupted = osthread->interrupted();
  // There is no synchronization between the setting of the interrupt
  // and it being cleared here. It is critical - see 6535709 - that
  // we only clear the interrupt state, and reset the interrupt event,
  // if we are going to report that we were indeed interrupted - else
  // an interrupt can be "lost", leading to spurious wakeups or lost wakeups
  // depending on the timing
  if (interrupted && clear_interrupted) {
    osthread->set_interrupted(false);
    ResetEvent(osthread->interrupt_event());
  } // Otherwise leave the interrupted state alone

  return interrupted;
}
```

中断状态操作：

* 获取 osthread->interrupted();
* 设置 osthread->set_interrupted(true);

``` c++
// osThread.hpp
class OSThread: public CHeapObj {
 private:
  volatile jint _interrupted;     // Thread.isInterrupted state

  volatile bool interrupted() const { return _interrupted != 0; }
  void set_interrupted(bool z) { _interrupted = z ? 1 : 0; }
```

## java 中断的本质

从中断的发生和中断状态的判断的实现来看，中断是 Java 平台 或者说是 JVM 提供的一种线程操作行为，和java线程对应的底层的操作系统线程没有任何关系。所以当一个 java 线程发生中断时，JVM 所做的就是记录这个状态位。系统底层线程的调度运行不会感知到 JVM 的中断行为。抛异常，也是 JVM 的行为。至于如何应对这个中断，就看具体的可中断方法（sleep, wait, join）是如何处理的。

所以 Thread.interrupt **不会直接导致线程被终止运行**，而只是 JVM 对线程的一种操作而已。

这篇文章 [详细分析Java中断机制](http://www.infoq.com/cn/articles/java-interrupt-mechanism) 对 Thread.interrupt 的功能描述：**Java中断机制是一种协作机制，也就是说通过中断并不能直接终止另一个线程，而需要被中断的线程自己处理中断**

[Java 理论与实践: 处理 InterruptedException](http://www.ibm.com/developerworks/cn/java/j-jtp05236.html)