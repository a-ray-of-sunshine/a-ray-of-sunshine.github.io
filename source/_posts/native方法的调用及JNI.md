---
title: native方法的调用及JNI
date: 2016-8-25 22:44:15
---

## native方法

参考 JVM规范，6.5 Instructions 其中的指令集：

* invokestatic
* invokevirtual

其中就有关于方法是 native 方法的调用机制的描述。 其中提到了符号链接的 Binging。

## JNI

JNI 有关 注册 native 方法，应该就和 符号链接的绑定 有关。

``` java
private static native void registerNatives();
static {
    registerNatives();
}
```

native 方法的实现

``` c++
// Thread.c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
};

JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
	// 调用 JNIEnv 的 RegisterNatives 将 native 注册到 JNI 环境中
	// native 方法被调用的时候，就可以通过注册的关联关系找到 native 实现
	// start0 <===> &JVM_StartThread
	// 建立起上面的对应关系，当 java 代码中调用 thread.start0() 时
	// JNI 就会找到 JVM_StartThread 调用其提供的实现
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```


同时从 jdk/src/share/native/java/lang/Thread.c 的实现来看
实现JNI接口有两种方法， 

* 直接实现

	registerNatives 接口自身就是一个 native 方法，它是直接在 Thread.c 中实现

* 通过 RegisterNatives 注册

	其它的方法，例如 start0 由于定义在其它地方，所以使用 RegisterNatives 通过注册的形式实现。

## JVM

关于 native 方法的调用 和 JNI 方法调用的 实现，其实都和 JVM 的实现有关，所以 需要系统学习 JVM 的实现。