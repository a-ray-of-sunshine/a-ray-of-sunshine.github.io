---
title: JVM-hotspot-GC机制
date: 2018-5-26 16:16:07
---

## 0. 如何表示一个 JAVA 对象

Java 对象的物理结构，

## 1. 对象是如何被创建的

JVM堆内存的分区是固定的吗？分别有哪些。对象的分配机制是什么，它们分别在哪些区域上。

垃圾收集器 和 垃圾回收算法 的关系是什么。垃圾回收有哪些：minor GC, major GC, FULL GC.

垃圾收集器 垃圾回收线程有哪些？

对象存活性如果判定的？引用可达，不可达如何判定？

## 2. 对象被创建在哪里

## 3. 对象是如何被回收的

## JVM 内存分区

## 垃圾收集器

## 垃圾回收算法

## 对象创建分配机制

Iterable 模型，为什么要有 iterator, 性能考虑。

## 如何确定 GC roots

## safepoint


## $. 参考
1. [JVM初探——使用堆外内存减少Full GC](http://www.importnew.com/23186.html)
2. [Java虚拟机7：内存分配原则](http://www.cnblogs.com/xrq730/p/4841177.html)
3. [频繁FULL GCC排查过程](https://liuzhengyang.github.io/2017/03/21/jvmtroubleshooting/)
4. [HotSpot Java虚拟机中的“方法区”“持久代”“元数据区”的关系？](https://www.zhihu.com/question/27429881)
5. [JVM虚拟机系列文章-五月的仓颉cnblogs博文](https://www.cnblogs.com/xrq730/category/731395.html)
6. [探究JVM之内存结构](https://blog.stormma.me/2017/11/15/%E4%B8%80%E7%82%B9%E4%B8%80%E6%BB%B4%E6%8E%A2%E7%A9%B6JVM%E4%B9%8B%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84/)
7. [深入浅出Java垃圾回收机制](http://www.importnew.com/1993.html)
8. [importnew-jvm](http://www.importnew.com/tag/jvm)
9. [自己关于VM的帖的目录](http://rednaxelafx.iteye.com/blog/362738)
10. [Java性能优化之JVM GC](https://zhuanlan.zhihu.com/p/25539690)
11. [成为JavaGC专家（1）—深入浅出Java垃圾回收机制](http://www.importnew.com/1993.html)
12. [jvm垃圾回收是什么时候触发的？ 垃圾回收算法？ 都有哪些垃圾回收器](https://blog.csdn.net/sunny243788557/article/details/52797088)
13. [聊聊JVM（六）理解JVM的safepoint](https://blog.csdn.net/iter_zc/article/details/41847887)
14. [How to get Java stacks when JVM can't reach a safepoint](https://stackoverflow.com/questions/20134769/how-to-get-java-stacks-when-jvm-cant-reach-a-safepoint)
15. [GC中Stop the world案例实战](https://blog.csdn.net/sinat_25306771/article/details/52374498)
16. [聊聊JVM-关于 safepoint 的机制，非常详细](https://blog.csdn.net/column/details/talk-about-jvm.html)
17. [java8 Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/toc.html)
18. [java9 Garbage Collection Tuning Guide](https://docs.oracle.com/javase/9/gctuning/toc.htm)
19. [Getting Started with the G1 Garbage Collector](http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html)
20. [g1 gc](http://www.oracle.com/technetwork/server-storage/ts-5419-159484.pdf)
21. [Garbage-First garbage collection原始论文](https://www.researchgate.net/publication/221032945_Garbage-First_garbage_collection)