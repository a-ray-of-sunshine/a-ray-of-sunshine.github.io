---
title: JVM-0.学习目录
date: 2017-2-9 10:51:26
---

## 学习目录

* class文件的加载和其对应的 Class 对象的创建过程

	类加载机制
	
	对象模型的实现
	
* JVM heap 的创建和管理，gc的实现

	heap 的分配使用情况
	
	gc 的实现
	
	各种引用的实现  soft, weak, phantom 引用的实现

* Thread 模型的实现

	Thread对象在JVM级别的表示，线程的实现。

* 对象同步的实现

	synchronized 代码块如何实现同步
	
* 解释器的实现

	字节码如何被执行
	
	new 关键字的实现
	
* JNI的实现

	其中的各个 invoke 方法的实现。
	
	NewObject 系统方法的实现，掌握对象创建的算法和对象是如何在JVM heap上进行分配的。

* JVM的启动过程

	启动了什么，初始化了什么模块
	