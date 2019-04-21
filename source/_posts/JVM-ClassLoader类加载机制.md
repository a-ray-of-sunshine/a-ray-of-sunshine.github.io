---
title: JVM-ClassLoader类加载机制
date: 2017-8-5 22:41:04
---

## JVM 的类加载机制

### ExtClassLoader 和 AppClassLoader

openjdk\jdk\src\share\classes\sun\misc\Launcher.java

中 包含 ExtClassLoader 和 AppClassLoader 的实现。还有 jdk 中的 ClassLoader 的源码分析

### eclipse 中的 classpath 问题

eclipse 的 .claspath 文件的作用，以其
classpathentry

在 eclipse 中启动一个 java 程序时，发生什么，如何设置 classpath.

### classpath

### jar 文件格式

### jar 包执行

## JVM  的调试机制的实现

## jvm 的启动

### eclipse 中启动 java 类

如何添加 classpath

### 通过 maven 启动类

如何添加 classpath

### web 环境启动类

tomcat 的 classpath. tomcat 如何实现 ClassLoader

## JVM 系统属性的设置

java.class.path 两个系统属性的设置 java.ext.dirs

## 参考
1. [JDB]()