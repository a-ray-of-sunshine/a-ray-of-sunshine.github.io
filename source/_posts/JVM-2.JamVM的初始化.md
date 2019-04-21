---
title: JVM-2.JamVM的初始化
date: 2017-2-24 14:37:51
---

## JamVM

程序的入口点在 jam.c 的 main 函数。

``` c
int main(int argc, char *argv[]) {
    Class *array_class, *main_class;
    Object *system_loader, *array;
    MethodBlock *mb;
    InitArgs args;
    int class_arg;
    char *cpntr;
    int status;
    int i;

	// 初始化 InitArgs 变量，这个变量中存储这 JamVM 后续
	// 各个模块初始化过程中需要的配置参数。这些参数的来源由：
	// 1. 给定默认值 例如 args->java_stack = DEFAULT_STACK;
	// 2. 通过 sysconf 查询 JamVM 所运行的系统的配置进行设置
	//	  例如，查询系统的内存，从而设置 args->max_heap 和 args->min_heap
	// 3. 通过命令行参数来获取，例如下面的 parseCommandLine 调用
    setDefaultInitArgs(&args);
    class_arg = parseCommandLine(argc, argv, &args);

    args.main_stack_base = &array_class;

    if(!initVM(&args)) {
        printf("Could not initialise VM.  Aborting.\n");
        exit(1);
    }

   if((system_loader = getSystemClassLoader()) == NULL)
        goto error;

    mainThreadSetContextClassLoader(system_loader);

    for(cpntr = argv[class_arg]; *cpntr; cpntr++)
        if(*cpntr == '.')
            *cpntr = '/';

    main_class = findClassFromClassLoader(argv[class_arg], system_loader);
    if(main_class != NULL)
        initClass(main_class);

    if(exceptionOccurred())
        goto error;

    mb = lookupMethod(main_class, SYMBOL(main),
                                  SYMBOL(_array_java_lang_String__V));

    if(mb == NULL || !(mb->access_flags & ACC_STATIC)) {
        signalException(java_lang_NoSuchMethodError, "main");
        goto error;
    }

    /* Create the String array holding the command line args */

    i = class_arg + 1;
    if((array_class = findArrayClass(SYMBOL(array_java_lang_String))) &&
           (array = allocArray(array_class, argc - i, sizeof(Object*))))  {
        Object **args = ARRAY_DATA(array, Object*) - i;

        for(; i < argc; i++)
            if(!(args[i] = Cstr2String(argv[i])))
                break;

        /* Call the main method */
        if(i == argc)
            executeStaticMethod(main_class, mb, array);
    }

error:
    /* ExceptionOccurred returns the exception or NULL, which is OK
       for normal conditionals, but not here... */
    if((status = exceptionOccurred() ? 1 : 0))
        uncaughtException();

    /* Wait for all but daemon threads to die */
    mainThreadWaitToExitVM();
    exitVM(status);

   /* Keep the compiler happy */
    return 0;
}
```

## JavaVM 各个模块的初始化

在 main 中调用 initVM 进行各个模块的初始化

``` c
// init.c initVM
int initVM(InitArgs *args) {
	// 初始化的结果
    int status;

    /* Perform platform dependent initialisation */
    // 进行平台（os_cpu）相关的初始化
    initialisePlatform();

    /* Initialise the VM modules -- ordering is important! */

    status = initialiseHooks(args) &&
             initialiseProperties(args) &&
             initialiseAlloc(args) &&
             initialiseThreadStage1(args) &&
             initialiseUtf8() &&
             initialiseSymbol() &&
             initialiseClassStage1(args) &&
             initialiseDll(args) &&
             initialiseMonitor() &&
             initialiseString() &&
             initialiseException() &&
             initialiseNatives() &&
             initialiseAccess() &&
             initialiseFrame() &&
             initialiseJNI() &&
             initialiseInterpreter(args) &&
             initialiseClassStage2() &&
             initialiseThreadStage2(args) &&
             initialiseGC(args);

	// init.c 中的全局变量，用来标识 VM 的初始化状态
	// 默认为 TRUE, 当 VM 初始化完成之后
	// 设置成 FALSE， 表示初始化工作已经结束。
    VM_initing = FALSE;
    return status;
}
```

### initialiseHooks

初始化定义在 hooks.c 中的全局变量 vfprintf_hook 和 exit_hook，导出两个函数 `jam_fprintf` 和 `jamvm_exit`

其中模块就可以使用上面的两个函数，进行控制台打印输出。

``` c
// hooks.c
int initialiseHooks(InitArgs *args) {
	
	// args->vfprintf 的值是 vfprintf
	// args->exit 的值是 exit
	// 这两个都是 glibc 中提供的标准函数
    vfprintf_hook = args->vfprintf;
    exit_hook = args->exit;
    return TRUE;
}
```

### initialiseProperties

初始化三个变量

``` c
// 存储 JamVM 启动时的命令行参数
// 也就是通过 java 或者 jamvm 传递过来的参数
static Property *commandline_props;
// 上面参数数组的个数
static int commandline_props_count;
// 保存 java_home
// 其实就是 JamVM 的安装路径，默认是
// java_home = "/usr/local/jamvm"
static char *java_home;
```

### initialiseAlloc

初始化堆区（heap）

其中使用到了三个参数：

* args->min_heap

	这个参数可以通过 -Xms<size> 进行设置
	
	set the initial size of the heap (default = MAX(physical memory/64, 16M))
	
	其含义是设置堆的初始化分配的内存。

* args->max_heap

	这个参数可以通过 -Xmx<size> 进行设置       
	
	set the maximum size of the heap (default = MIN(physical memory/4, 256M))
	
	表示 JamVM 堆的最大内存

* args->verbosegc

	Set verbose option from initialisation arguments
	
	当这个选项是 TRUE 的时候，gc 线程在进行 gc 的时候会输出详细信息。

``` c
// 关于对齐和整除
// 1. 对齐
// 对于给定的地址 n ，将其对齐到字长为 m 的地址处
// 内存地址对齐的含义：要把n按m对齐，就是找到大于等于n且整除m的最小的那个数。
// 计算方法如下
(n+m-1)&~(m-1)

// 2. 整除
// 对于给定的内存块，已知在其上分配的对象需要按照 m 对齐， 则对于 n 个字节的内存块，按 m 对齐之后，内存块大小 rn 必须能够被 m 整除，且 rn 是 小于 n 且能够被 m 整除的最大的那个数。
// 计算方法如下
rn = (n)&~(m-1) 
// 例如，设置的 jvm 堆的最小字节个数为 260, jvm 要求
// 对齐到 8 字节，则 上面的 jvm 堆的最小字节个数将被
// 调整到 260 &~(8-1) = 256 个字节

```

``` c
int initialiseAlloc(InitArgs *args) {
	// 直接通过内存映射的方法，分配 args->max_heap 大小的内存
	// 这些内存将作为堆，被 JVM 所使用
    char *mem = (char*)mmap(0, args->max_heap, PROT_READ|PROT_WRITE,
                                               MAP_PRIVATE|MAP_ANON, -1, 0);
    if(mem == MAP_FAILED) {
        perror("Couldn't allocate the heap; try reducing the max "
               "heap size (-Xmx)");
        return FALSE;
    }

    /* Align heapbase so that start of heap + HEADER_SIZE is object aligned */
    // 把 mem+HEADER_SIZE 这个地址调整到  OBJECT_GRAIN（8） 对齐
    // 这样 heapbase + HEADER_SIZE 也就是和 OBJECT_GRAIN 是对齐的
    heapbase = (char*)(((uintptr_t)mem+HEADER_SIZE+OBJECT_GRAIN-1)&
               ~(OBJECT_GRAIN-1))-HEADER_SIZE;

    /* Ensure size of heap is multiple of OBJECT_GRAIN */
    // (args->min_heap-(heapbase-mem))&~(OBJECT_GRAIN-1) 的作用
    // 是将 min_heap 调整到能被 OBJECT_GRAIN 整除，且不会超过自身
    heaplimit = heapbase+((args->min_heap-(heapbase-mem))&~(OBJECT_GRAIN-1));
    heapmax = heapbase+((args->max_heap-(heapbase-mem))&~(OBJECT_GRAIN-1));

    /* Set initial free-list to one block covering entire heap */
    // 初始化堆空闲列表
    freelist = (Chunk*)heapbase;
    // header 表示空闲堆的大小
    freelist->header = heapfree = heaplimit-heapbase;
    freelist->next = NULL;

    TRACE_GC("Alloced heap size %p\n", heaplimit-heapbase);
    // 用来标记内存，和 gc 有关
    allocMarkBits();

    /* Initialise GC locks */
    // 初始化锁，和 GC 有关
    initVMLock(heap_lock);
    initVMLock(has_fnlzr_lock);
    initVMLock(registered_refs_lock);
    initVMWaitLock(run_finaliser_lock);
    initVMWaitLock(reference_lock);

    /* Cache system page size -- used for internal GC lists */
    // 系统调用 获取内存页大小 和 GC 有关
    sys_page_size = getpagesize();

    /* Set verbose option from initialisation arguments */
    // verbosegc 设为 TRUE, 则 GC 时会打印一些统计信息
    verbosegc = args->verbosegc;
    return TRUE;
}
```

### initialiseThreadStage1

初始化线程相关的锁。

初始化 当前线程（主线程）的 Thread 对象 main_thread, 使用 pthread_setspecific 放到当前线程的 TLS 中。 

### initialiseUtf8

初始化一个 HashTable 对象。

``` java
// utf8.c
static HashTable hash_table;
```

### initialiseSymbol

初始化 JVM 所使用的符号

### initialiseClassStage1

初始化 bootpath, 从参数中获取，从默认的配置中获取。

将 bootpath 转化成 static BCPEntry *bootclasspath;

### initialiseDll

初始化动态链接库的路径。

### initialiseMonitor

### initialiseString

``` java
// 调用 findSystemClass0 方法
// 初始化 java.lang.String 类
static Class *string_class;
// 缓存 String 类的 char[] value 字段
static int value_offset;
// 
static HashTable hash_table;
```

### initialiseException

初始化异常相关的类

### initialiseNatives


### initialiseAccess

### initialiseFrame

### initialiseJNI

初始化 JNI 相关的锁

### initialiseInterpreter

初始化解释器相关的锁

### initialiseClassStage2

初始化 java_lang_Class 类，

### initialiseThreadStage2

初始化 java_lang_Thread 类，缓存一些重要的字段偏移量。

并创建一个 Signal Handler 线程。

### initialiseGC

* 在堆上预先分配一个 java_lang_OutOfMemoryError 对象
* 创建一个 Finalizer 线程
* 创建一个 Reference Handler 线程

JamVM 的初始化，就是初始化各个模块的全局变量，锁，分配需要的内存，启动相关的线程。

## $. 参考
1. [mmap](http://man7.org/linux/man-pages/man2/mmap.2.html)
2. [地址对齐的位运算代码的解释](http://www.cnblogs.com/waytofall/p/4109514.html)
3. [Data structure alignment](https://en.wikipedia.org/wiki/Data_structure_alignment)
4. [static关键字的用法](https://msdn.microsoft.com/en-us/library/c06ztef7.aspx)
5. [Internal Linkage](https://msdn.microsoft.com/en-us/library/cftw3t5e.aspx)
6. [offsetof Macro](https://msdn.microsoft.com/en-us/library/dz4y9b9a.aspx)