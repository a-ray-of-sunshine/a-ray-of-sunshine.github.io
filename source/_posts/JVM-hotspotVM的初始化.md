---
title: JVM-hotspotVM的初始化
date: 2017-8-12 22:47:20
---

使用 java.exe 启动一个程序的时候会加载 jvm.dll , 调用其 JNI 接口。

``` c
// JavaVM 和 JNIEnv
// 都是一个包含 JNI 接口的对象。这两个参数都是最终被
// JVM 初始化成功之后，需要初始化的参数。

InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn){
	r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
}
```

### JNI_CreateJavaVM

**openjdk\hotspot\src\share\vm\prims\jni.cpp**

创建一个 JVM