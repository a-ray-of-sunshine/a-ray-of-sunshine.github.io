---
title: ClassLoader
date: 2017-8-11 22:41:06
---

## java 程序的启动

``` bash
## 调试模式启动 JAVA
## 将输出一系列调试系统
set _JAVA_LAUNCHER_DEBUG=1
```

### 查找 JRE

#### 查找当前启动的启动器所在的路径

**openjdk\jdk\src\windows\bin\java_md.c**

使用 WIN32 API GetModuleFileName 文件查找当前运行模块的路径。

如果 PATH 环境变量中配置了 %JAVA_HOME%/bin。其中 JAVA_HOME 为 JDK 安装的目录。

截取掉文件名和bin目录，则此时找到的 path 为 %JAVA_HOME%

#### 查找 java.dll

在上一个步骤中，以上面的 path 为基路径，查找 <path>/bin 目录中是否存在 java.dll 文件，如果存在，则 path 即为 JRE_HOME, 否则，查看 <path>/jre/bin 目录中是否存在 java.dll 如果存在，则为 JRE_HOME.

因为上面的 path 是 JAVA_HOME, 其 bin 目录下是 JDK 提供的各种工具程序，并不存在 java.dll，所以最终定位到 %JAVA_HOME%/jre 这个目录下。此时找到的 JRE 就是 %JAVA_HOME%/jre 这个目录。

#### 查找公共的 JRE

如果通过查找 java.dll 并没有找到 JRE，则这个启动器，可能在类似 c:\windows\system32\java.exe 这种形式的。自然定位不到 JRE 。

此时就开始查找公共的 JRE。

在 windows 平台上公共 JRE 的查找是通过注册表来实现的。

查找  HKEY_LOCAL_MACHINE 的   "Software\\JavaSoft\\Java Runtime Environment" 这个子键下面的信息。这个子键下面有 

"CurrentVersion\\JavaHome" 这个子键。这个子键的值就是公共的 JRE 的路径。

至此 JRE 找到了，当然如果没有找到进程直接就退出了。

### 查找 JVM

#### 读取已有的 JVM

读取 %JRE_HOME%/lib/<arch>/jvm.cfg 文件。

这个文件中列出了多种可选的 JVM。最终构造好一个 JVM 列表。

``` bash
## - 后面也可以直接跟 JVM 所在的路径。如果是路径，则一定要以
## // 或者 \ 结尾
-server KNOWN
-client IGNORE
```

#### 检测当前要使用的JVM

``` bash
/*
 * Checks the command line options to find which JVM type was
 * specified.  If no command line option was given for the JVM type,
 * the default type is used.  The environment variable
 * JDK_ALTERNATE_VM and the command line option -XXaltjvm= are also
 * checked as ways of specifying which JVM type to invoke.
 */
```

获得 JVM 类型。

#### 查找 JVM

直接查找 %JRE_HOME%\\<jvmtype>\\jvm.dll 是否存在。

如果 jvmtype 是全路径，则查找  <jvmtype>\\jvm.dll 是否存在。

如果存在则 JVM 就找到了。就是 %JRE_HOME%\\<jvmtype>\\jvm.dll 或者 
<jvmtype>\\jvm.dll

## LoadJavaVM

**参考openjdk\jdk\src\share\bin\java.c**

通过 LoadLibrary 或者  dlopen 方法载入上面查找到的 jvm.dll.

并通过 GetProcAddress 或者 dlsym 方法查找到 jvm.dll 中实现的 "JNI_CreateJavaVM" 和 "JNI_GetDefaultJavaVMInitArgs" 方法。

初始化好一个 InvocationFunctions 结构。

## AddApplicationOptions

添加三个 optionis

* -Denv.class.path

	env.class.path is the user's setting of CLASSPATH
	
	从环境变量中获得 "CLASSPATH" 

* -Dapplication.home

	application.home is the directory where the application is installed.
	
	获得当前执行的 java 所在的路径，通常就是 "JAVA_HOME"

* -Djava.class.path

	java.class.path is the classpath to where our apps' classfiles are.
	
	默认是 { "/lib/tools.jar", "/classes" }， 此时会将 classpath 展开，形成 %JAVA_HOME%/lib/tools.jar 和 %JAVA_HOME%/classes 并将其添加到 java.class.path 属性中。
	
默认的 classpath 的设置。首先获得环境变量 CLASSPATH，如果不存在，则默认为当前上当 ".", 将其设置到 -Djava.class.path 中。

## 初始化 JVM

创建一个新的线程。在新的线程调用 JavaMain 方法。

JavaMain 中调用 InitializeJVM 这个方法将通过 InvocationFunctions 的 JNI_CreateJavaVM 创建 JVM， 将初始化 JNIEnv 变量 env.

JNI_CreateJavaVM 将调用 JVM 实现，初始化好整个 JVM 环境。

通过 env 就可以使用 JNI 接口的方法，调用 JVM 方法。执行 Java 类。

## 调用主类的方法

由于 JVM 已经初始化了，就可以通过 env 调用 JVM 的功能了，可以执行 java 类的方法了。

### 执行 Main 类

``` c
// java.c LoadMainClass 方法
// 调用 JVM_FindClassFromBootLoader 方法
// 从 BootLoader 中获得 "sun/launcher/LauncherHelper" Class 对象
jclass cls = GetLauncherHelperClass(env);

// 获得 sun.launcher.LauncherHelper.checkAndLoadMain 方法 ID
(*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;");

// 调用 checkAndLoadMain 获得 Main 类
mainClass = (*env)->CallStaticObjectMethod(env, cls, mid, USE_STDERR, mode, str);

// 获得 Main 类的 main 方法
mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");

// 调用 main 方法
(*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);
```


## ClassPath 

指定 classpath 时有两种类型：

* class 文件打包在 jar 或者 zip 文件中

	这种类型的 classpath 直接指定文件的全路径，以 zip 或者 jar 文件名称结束。
	
* class 文件直接存放在文件夹（目录）当中

	则 classpath 是，包含 "root" package 的 文件夹，为其 classpath

## 运行时类的加载

### 默认的加载位置

### 如何修改

``` bash
-Xbootclasspath:bootclasspath
-Xbootclasspath/a:path
-Xbootclasspath/p:path
```


## 扩展类加载

java.ext.dirs

## 应用类加载

java.class.path

## 参考
1. [java command doc](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/java.html)
2. [javac command doc](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javac.html)
3. [Setting the class path](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/classpath.html)