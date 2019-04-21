---
title: 线程-java命令的实现
date: 2016-8-27 22:22:10
---

// 不同平台所共享的入口
java.c hotspot/src/share/tools/launcher

// 不同平台提供相应的实现
java_md.c 

* win32

	hotspot/src/os/windows/launcher

* posix

	hotspot/src/os/posix/launcher

## 解析主类和运行时环境

``` c++
// java.c
/*
 * Entry point.
 */
int main(int argc, char ** argv){

// 核心过程

// 1. 根据输入参数及运行的环境获取 java 执行环境

// CreateExecutionEnvironment 由 java_md.c 实现
// 在不同的平台上实现获得环境变量的方法不同
// 所以交由不同的平台来实现。
// 通过这个方法，初始化 jrepath & jvmpath
CreateExecutionEnvironment(&argc, &argv,
                           jrepath, sizeof(jrepath),
                           jvmpath, sizeof(jvmpath),
                           original_argv);

// 2. 加载虚拟机

// 通过 jvmpath 加载虚拟机的 dll （或者 so 文件）
// 并初始化变量 InvocationFunctions ifn
// ifn 中的 CreateJavaVM 是一个函数指针，用来创建 JVM
LoadJavaVM(jvmpath, &ifn)

// 3. 设置 CLASSPATH

/* Set default CLASSPATH */
if ((s = getenv("CLASSPATH")) == 0) {
	// 如果没有CLASSPATH，则默认为当前目录
    s = ".";
}
SetClassPath(s);

// 4. 创建新的线程来创建 JVM 然后调用 main 方法。
/* Create a new thread to create JVM and invoke main method */
// ContinueInNewThread 由不同的平台实现。所以在
// java_md.c 中
// 注意这里的 JavaMain 函数将成为新线程的入口点
return ContinueInNewThread(JavaMain, threadStackSize, (void*)&args);
}


/*
 * Pointers to the needed JNI invocation API, initialized by LoadJavaVM.
 */
// 函数指针：用来创建 JVM
typedef jint (JNICALL *CreateJavaVM_t)(JavaVM **pvm, void **env, void *args);
// 函数指针：获取 JVM 的初始化参数
typedef jint (JNICALL *GetDefaultJavaVMInitArgs_t)(void *args);

typedef struct {
    CreateJavaVM_t CreateJavaVM;
    GetDefaultJavaVMInitArgs_t GetDefaultJavaVMInitArgs;
} InvocationFunctions;
```

ContinueInNewThread

``` c++
// 1. 创建新的线程。
// 这里的 continuation 就是新的线程的入口 JavaMain
// 这个方法一经调用，新线程创建成功将开始执行 JavaMain 方法了。
HANDLE thread_handle =
  (HANDLE)_beginthreadex(NULL,
                         (unsigned)stack_size,
                         continuation,
                         args,
                         STACK_SIZE_PARAM_IS_A_RESERVATION,
                         &thread_id);


if (thread_handle) {
  // 如果新线程创建成功，则当前线程进入等待状态
  // 直到新线程执行结束
  WaitForSingleObject(thread_handle, INFINITE);
  // 获得 JVM 线程的返回结果
  GetExitCodeThread(thread_handle, &rslt);
  // 关闭线程句柄
  CloseHandle(thread_handle);
} else {
  // 如果新线程创建抵账，则直接由当前线程开始运行 JVM
  rslt = continuation(args);
}
```

JavaMain

``` c++
int JNICALL JavaMain(void * _args){

// 1. 初始化 JVM
/* Initialize the virtual machine */
// 初始化下面两个变量
JavaVM *vm = 0;
JNIEnv *env = 0;
// 初始化 JVM 及相关的所有子系统，加载相关的 JAVA 类
// 初始化 env ，提供 JNI 接口
InitializeJVM(&vm, &env, &ifn)

// 2. Get the application's main class.
mainClassName = GetMainClassName(env, jarfile);

// 3. Get the application's main method
mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                   "([Ljava/lang/String;)V");
}

// 4. Build argument array
mainArgs = NewPlatformStringArray(env, argv, argc);

// 5. 调用主类的 main 方法 Invoke main method.
// 整个调用过程由 JNI 变量 env 完成。
(*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

// 6. 销毁 JVM
/*
 * Wait for all non-daemon threads to end, then destroy the VM.
 * This will actually create a trivial new Java waiter thread
 * named "DestroyJavaVM", but this will be seen as a different
 * thread from the one that executed main, even though they are
 * the same C thread.  This allows mainThread.join() and
 * mainThread.isAlive() to work as expected.
 */
// DestroyJavaVM 其实现在 jni.cpp 中 jni_DestroyJavaVM
(*vm)->DestroyJavaVM(vm);
```

### 加载虚拟机 LoadJavaVM

LoadJavaVM 在 java.c 的 main 函数中被调用
其定义在 java_md.c 中

``` c++
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn){
// 1. 加载 jvm.dll
HINSTANCE handle;
handle = LoadLibrary(jvmpath)

// 2. 获得核心入口函数
/* Now get the function addresses */
ifn->CreateJavaVM =
    (void *)GetProcAddress(handle, "JNI_CreateJavaVM");
ifn->GetDefaultJavaVMInitArgs =
    (void *)GetProcAddress(handle, "JNI_GetDefaultJavaVMInitArgs");
}

```

### 初始化虚拟机 InitializeJVM

InitializeJVM 在 java.c 的 JavaMain 函数中被调用
其定义在 java.c 中。

``` c++
static jboolean InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn){
    JavaVMInitArgs args;
    jint r;

    memset(&args, 0, sizeof(args));
    args.version  = JNI_VERSION_1_2;
    args.nOptions = numOptions;
	// options 全局变量，
    args.options  = options;
    args.ignoreUnrecognized = JNI_FALSE;

	// ifn 在 LoadJavaVM 被初始化
	// CreateJavaVM 对应 JNI_CreateJavaVM
	// 这个函数定义在 jni.cpp 中
	r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
	
	return r == JNI_OK;
}
```

## JNI_CreateJavaVM
// jni.cpp
``` c++
// 这个方法的作用是 创建 JVM
// 初始化参数 vm & penv
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
	
	// 1. 创建 VM
	result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);

	// 2. 初始化参数
	JavaThread *thread = JavaThread::current();
	// main_vm 是一个全局变量，提供了 5 个接口指针
    *vm = (JavaVM *)(&main_vm);
	// JNIEnv 包括由函数指针构成的结构体
	// 相当于 JVM 提供的 一系列接口，加载类，执行类的方法都依靠这个指针
	// 这里的 penv 指向的 变量 是 线程对应  thread 在创建的时候
	// 初始化好的，new JavaThread 中 调用 initialize，
	// initialize 调用 set_jni_functions(jni_functions());
	// jni 的功能都实现在 jni.cpp 中。
    *(JNIEnv**)penv = thread->jni_environment();

	return result;
}

struct JavaVM_ {
    const struct JNIInvokeInterface_ *functions;
#ifdef __cplusplus

    jint DestroyJavaVM() {
        return functions->DestroyJavaVM(this);
    }
    jint AttachCurrentThread(void **penv, void *args) {
        return functions->AttachCurrentThread(this, penv, args);
    }
    jint DetachCurrentThread() {
        return functions->DetachCurrentThread(this);
    }

    jint GetEnv(void **penv, jint version) {
        return functions->GetEnv(this, penv, version);
    }
    jint AttachCurrentThreadAsDaemon(void **penv, void *args) {
        return functions->AttachCurrentThreadAsDaemon(this, penv, args);
    }
#endif
};
struct JavaVM_ main_vm = {&jni_InvokeInterface};
const struct JNIInvokeInterface_ jni_InvokeInterface = {
    NULL,
    NULL,
    NULL,

    jni_DestroyJavaVM,
    jni_AttachCurrentThread,
    jni_DetachCurrentThread,
    jni_GetEnv,
    jni_AttachCurrentThreadAsDaemon
};


// Thread.cpp
// 这个方法的主要作用就是创建 main_thread 对象，代表 JVM 主线程。
// 初始化 java 运行时环境，载入 java 运行时系统
// 这个方法是创建 JVM 的核心
jint Threads::create_vm(JavaVMInitArgs* args, bool* canTryAgain) {

// 1. Initialize TLS
// 为 main_thread 分配 TLS 索引
ThreadLocalStorage::init();

// 2. 创建代表当前主线程的 JavaThread 对应 main_thread.
JavaThread* main_thread = new JavaThread();

// 3. 当 main_thread 放置成 TLS 中
// Attach the main thread to this os thread
// main_thread->initialize_thread_local_storage 实现：
// ThreadLocalStorage::set_thread(this);
main_thread->initialize_thread_local_storage();

}


```

## 如何获得当前线程

``` c++
inline JavaThread* JavaThread::current() {
  Thread* thread = ThreadLocalStorage::thread();
  assert(thread != NULL && thread->is_Java_thread(), "just checking");
  return (JavaThread*)thread;
}


// ThreadLocalStorage 这个类是由不同平台实现的
// ThreadLocalStorage 类定义在 runtime/threadLocalStorage.hpp
// Interface for thread local storage
// Fast variant of ThreadLocalStorage::get_thread_slow
extern "C" Thread*   get_thread();

// Get raw thread id: e.g., %g7 on sparc, fs or gs on x86
extern "C" uintptr_t _raw_thread_id();

class ThreadLocalStorage : AllStatic {
 public:
  static void    set_thread(Thread* thread);
  static Thread* get_thread_slow();
  static void    invalidate_all() { pd_invalidate_all(); }

// Machine dependent stuff
// win32 实现定义在: 
// hotspot\src\os_cpu\windows_x86\vm\threadLS_windows_x86.hpp
// Processor dependent parts of ThreadLocalStorage
// ---- win32 ----
protected:

  static int _thread_ptr_offset;

public:

  // Java Thread
  static inline Thread* thread() {
    return (Thread*)TlsGetValue(thread_index());
  }

  static inline Thread* get_thread() {
    return (Thread*)TlsGetValue(thread_index());
  }

  static inline void set_thread_ptr_offset( int offset ) { _thread_ptr_offset = offset; }

  static inline int get_thread_ptr_offset() { return _thread_ptr_offset; }

// ---- win32 ----

 public:
  // Accessor
  static inline int  thread_index()              { return _thread_index; }
  static inline void set_thread_index(int index) { _thread_index = index; }

  // Initialization
  // Called explicitly from VMThread::activate_system instead of init_globals.
  static void init();
  static bool is_initialized();

 private:
  static int     _thread_index;

  static void    generate_code_for_get_thread();

  // Processor dependent parts of set_thread and initialization
  static void pd_set_thread(Thread* thread);
  static void pd_init();
  // Invalidate any thread cacheing or optimization schemes.
  static void pd_invalidate_all();

};

// ThreadLocalStorage 类的 实现部分定义在
// hotspot\src\share\vm\runtime\threadLocalStorage.cpp

// 在 win32 平台上还包括下面的文件
#include "os_windows.inline.hpp"
#include "thread_windows.inline.hpp"

// ----share实现部分---
// static member initialization
int ThreadLocalStorage::_thread_index = -1;

Thread* ThreadLocalStorage::get_thread_slow() {
  return (Thread*) os::thread_local_storage_at(ThreadLocalStorage::thread_index());
}

void ThreadLocalStorage::set_thread(Thread* thread) {
  // 调用具体平台的实现
  pd_set_thread(thread);

  // The following ensure that any optimization tricks we have tried
  // did not backfire on us:
  guarantee(get_thread()      == thread, "must be the same thread, quickly");
  guarantee(get_thread_slow() == thread, "must be the same thread, slowly");
}

void ThreadLocalStorage::init() {
  assert(!is_initialized(),
         "More than one attempt to initialize threadLocalStorage");
  // 在 win32 平台上面 pd_init 是空实现
  pd_init();
  // 初始化 thread_index 为一个 TLS index.
  set_thread_index(os::allocate_thread_local_storage());
  generate_code_for_get_thread();
}

bool ThreadLocalStorage::is_initialized() {
    return (thread_index() != -1);
}
// ----share实现部分---

// -----win32 实现部分，-----
// hotspot\src\os_cpu\windows_x86\vm\threadLS_windows_x86.cpp
int ThreadLocalStorage::_thread_ptr_offset = 0;

static void call_wrapper_dummy() {}

// We need to call the os_exception_wrapper once so that it sets
// up the offset from FS of the thread pointer.
void ThreadLocalStorage::generate_code_for_get_thread() {
      os::os_exception_wrapper( (java_call_t)call_wrapper_dummy,
                                NULL, NULL, NULL, NULL);
}

void ThreadLocalStorage::pd_init() { }

void ThreadLocalStorage::pd_set_thread(Thread* thread)  {
  // 将 thread 对应放置到 TLS
  os::thread_local_storage_at_put(ThreadLocalStorage::thread_index(), thread);
}
// ------win32------


// os_windows.cpp
// Allocates a thread local storage (TLS) index.
// 这个函数返回一个 TLS 索引，
// 分配
int os::allocate_thread_local_storage() {
  return TlsAlloc();
}

// 释放
void os::free_thread_local_storage(int index) {
  TlsFree(index);
}

// 设置
void os::thread_local_storage_at_put(int index, void* value) {
  TlsSetValue(index, value);
  assert(thread_local_storage_at(index) == value, "Just checking");
}

// 获取
void* os::thread_local_storage_at(int index) {
  return TlsGetValue(index);
}
```

``` c++

JVM_ENTRY(jobject, JVM_CurrentThread(JNIEnv* env, jclass threadClass))
  JVMWrapper("JVM_CurrentThread");
  oop jthread = thread->threadObj();
  assert (thread != NULL, "no current thread!");
  return JNIHandles::make_local(env, jthread);
JVM_END

因为每一个 JavaThread 都有一个 _jni_environment，所以每一个线程调用 native 方法时 其 JNIEnv 是不同的，应该是当前线程的 JNIEnv 变量。

JavaThread ------>  +-----------------+ --}
					|                 |   |
					|   other_vars    |   |
					+-----------------+   |---> in_bytes(jni_environment_offset())                               
					|                 |   |
					|_jni_environment |   |
(intptr_t)env <---- +-----------------+ --}
					|                 |
					|   other_vars    |
					+-----------------+

class JavaThread: public Thread {
	// ...
	JNIEnv        _jni_environment;
	// ...
}

从上图可以观察到：当一个线程调用 Thread.currentThread() 时，

env 变量是已知的，同时这个 env 变量也是当前 JavaThread 对象的一个成员变量，则依据 env 在 JavaThread 中的类结构
的位置，可以计算出 env 所以 JavaThread 的偏移量，此时计算 当前的线程对象：

JavaThread *thread = (JavaThread*)((intptr_t)env - in_bytes(jni_environment_offset()));

所以 Thread.currentThread 是依据 env 所在的内存布局，计算而来，这样计算非常快速。
```

## 新启动的 java 线程

对于第一个新启动的线程，都会将其自身 JavaThread 对象放置到 TLS 中。

``` c++
// The first routine called by a new Java thread
void JavaThread::run() {
  // ...

  // Initialize thread local storage; set before calling MutexLocker
  // 调用 TLS ThreadLocalStorage::set_thread(this);
  this->initialize_thread_local_storage();

  // ...
}
```