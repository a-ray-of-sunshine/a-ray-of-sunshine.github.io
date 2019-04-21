---
title: JVM-类型系统的实现
date: 2017-8-13 9:40:46
tag: [JVM]
---

## hotspot 的历史

### Smalltalk

Smalltalk is an object-oriented, dynamically typed, reflective programming language.

### Self 

Self was designed mostly by David Ungar and Randall Smith in 1986 


They moved to Stanford University and continued work on the language, building the first working Self compiler in 1987
 
The first public release was in 1990, and the next year the team moved to Sun Microsystems where they continued work on the language. 

Self 团队的人去了 Sun Microsystems

### Strongtalk

Dave Griswold wanted to use Smalltalk more extensively。

His efforts resulted in the 1993 paper he co-authored with Gilad Bracha. This version was based on adding type-checking to the ParcPlace Systems implementation of Smalltalk; an implementation that started from scratch would gain a better typing system.

1994年 他和 Urs Hölzle, who had finished the second-generation Self compiler (and his Stanford thesis)，还有其它几个人一起创建了一家小公司叫 LongView Technologies。这个公司重新实现了 Smalltalk 的运行时环境，并称为 Strongtalk。他们的工作于 1996 年完成。 Sun Microsystems 在 1997 年买下了这有公司（LongView Technologies）。

此后 Self 团队和 LongView Technologies 团队的人都在 Sun 公司进行 Java 的开发

LongView Technologies 团队对 Self 的技术进行了整合形成了最终的 Strongtalk，而后来 Sun Microsystems 就是基于 Strongtalk 进行了 Hotspot 的开发。

## 创建 JVM

### JNI_CreateJavaVM

**jni.cpp**

``` c
_JNI_IMPORT_OR_EXPORT_ jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
  bool can_try_again = true;

  // 创建 VM
  result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
  if (result == JNI_OK) {
    JavaThread *thread = JavaThread::current();
    /* thread is thread_in_vm here */
    *vm = (JavaVM *)(&main_vm);
    *(JNIEnv**)penv = thread->jni_environment();
	}
}
```

#### create_vm

**thread.cpp**

下面开始分析 create_vm 各个关键过程

##### Arguments::process_sun_java_launcher_properties(args);

**arguments.cpp**

1. _created_by_gamma_launcher

	如果 args 中的 -Dsun.java.launcher= 属性值为 "gamma", 则将 
	Arguments::_created_by_gamma_launcher 设置为 true
	
2. _sun_java_launcher_pid

	将 Arguments::_sun_java_launcher_pid 设置成 args k中的sun.java.launcher.pid 

##### os::init

**os_windows.cpp** 调用系统相关的 API 的获得系统属性，初始化 os 模块的全局变量

``` c
// this is called _before_ the global arguments have been parsed
void os::init(void) {
  // os 模块全局变量
  // _getpid returns the process ID obtained from the system
  _initial_pid = _getpid();

  // os::_rand_seed = 1234567
  init_random(1234567);

  // 初始化操作平台相关的属性
  // 调用 WIN32 API 获得系统属性
  // GetSystemInfo GlobalMemoryStatusEx GetVersionEx
  win32::initialize_system_info();
  win32::setmode_streams();
  // 在上面的 initialize_system_info 已经设置好了
  // vm_page_size, 
  init_page_sizes((size_t) win32::vm_page_size());

  // For better scalability on MP systems (must be called after initialize_system_info)
#ifndef PRODUCT
  if (is_MP()) {
    NoYieldsInMicrolock = true;
  }
#endif
  // This may be overridden later when argument processing is done.
  FLAG_SET_ERGO(bool, UseLargePagesIndividualAllocation,
    os::win32::is_windows_2003());

  // Initialize main_process and main_thread
  // GetCurrentProcess: 
  // 	Retrieves a pseudo handle for the current process.
  // GetCurrentThreadId:
  // 	Retrieves the thread identifier of the calling thread.
  main_process = GetCurrentProcess();  // Remember main_process is a pseudo handle
 if (!DuplicateHandle(main_process, GetCurrentThread(), main_process,
                       &main_thread, THREAD_ALL_ACCESS, false, 0)) {
    fatal("DuplicateHandle failed\n");
  }
  main_thread_id = (int) GetCurrentThreadId();
}
```

##### Arguments::init_system_properties();

初始化系统属性 sysclasspath, java_home, dll_dir

java_home = %JAVA_HOME%
dll_dir =  %JAVA_HOME%/bin

 Arguments::set_meta_index_path 设置 %JAVA_HOME%/lib/meta-index
 
Arguments::set_sysclasspath 设置成

 ``` c
 static const char classpath_format[] =
        "%/lib/resources.jar:"
        "%/lib/rt.jar:"
        "%/lib/sunrsasign.jar:"
        "%/lib/jsse.jar:"
        "%/lib/jce.jar:"
        "%/lib/charsets.jar:"
        "%/classes";
```

##### Arguments::parse

解析 args 中的参数

##### os::init_2

os::init 已经调用过了， os 模块的全局变量已经初始化。

初始化内存相关的属性，

##### init_globals

**init.cpp** 进行全局初始化

###### classLoader_init

加载 verify.dll java.dll zip.dll, 初始化好 bootstrap classpath.

1. 加载 %JRE_HOME%/bin/verify.dll
2. 加载 %JRE_HOME%/bin/java.dll
	
	将 java.dll 的句柄保存到 os.cpp 的全局变量 _native_java_library
	查找 java.dll 中定义的 Canonicalize 函数，将其保存到 classLoader.cpp 的全局变量 CanonicalizeEntry 中。	
	
	Canonicalize函数由 JDK 实现，最终编译进 java.dll 中，其实现在 jdk 的 jni_util.c 中。其实功能进行平台相关的路径转换。
	
	ClassLoader 在进行类加载的时候会调用这个 API 进行路径转换。

3. 加载 %JRE_HOME%/bin/zip.dll

###### ClassLoader::setup_bootstrap_search_path

**classLoader.cpp**

设置 bootstrap classpath

1. 从 Arguments::get_sysclasspath() 获得 sysclasspath
2. 拆分 classpath, 使用平台相关的分隔符，windows 上是 ";", linux 上是 ":"
3. 如果拆分后的路径存在，则创建一个 ClassPathEntry 对象。
4. 将上面的 ClassPathEntry 添加到 ClassLoader::_first_entry 和 ClassLoader::_last_entry 构成的链表。

###### ClassLoader::setup_meta_index

解析 %JAVA_HOME%/lib/meta-index 文件，这个文件中存储着 sysclass jar包的索引。

初始化上面创建了的 ClassPathEntry 的链表。

## $ 参考
1. [Self Language](http://www.selflanguage.org/)
2. [Self Source Code](https://github.com/russellallen/self)
3. [The Self Handbook](http://handbook.selflanguage.org/2017.1/index.html)
4. [Strongtalk](http://www.strongtalk.org/)
5. [Strongtalk Source Code](https://github.com/talksmall/Strongtalk)
6. [HotSpot wiki](https://en.wikipedia.org/wiki/HotSpot)
7. [Strongtalk wiki](https://en.wikipedia.org/wiki/Strongtalk)
8. [Self wiki](https://en.wikipedia.org/wiki/Self_(programming_language))