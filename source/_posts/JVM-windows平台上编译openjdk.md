---
title: JVM-windows平台上编译openjdk
date: 2017-8-17 13:31:28
---

编译环境： win7 + cygwin

参考 openjdk/README-builds.html

## 准备编译环境

0. jdk-7u80-windows-i586.exe 32位
1. win7  32位或者64位都可以
2. cygwin 2.8.2 32位
3. visual studio 2012 express x86 32位

### 安装工具链

gawk, file ,zip, unzip 已经安装了，不需要安装

zip, unzip 虽然已经安装但是要重新安装新版。一定需要重新安装新版。

free: procps-ng: System
libs: libfreetype-devel

安装 Mercurial ， 用来下载 openjdk 源码

``` bash
hg clone http://hg.openjdk.java.net/jdk8u/jdk8u jdk8u
cd jdk8u
chmod +x get_source.sh
bash ./get_source.sh
```

注意下载过程中如果因为网络原因出现异常终止，则可以不断地调用 get_source.sh , 直到整个项目安全下载完毕。

### 准备 CYGWIN 环境

如果安装是 CYGWIN 非 1.7 版本。则需要修改。两个文件。

openjdk8/common/autoconf 中的 basics_windows.m4 和 generated-configure.sh 将其中对 CYGWIN 的版本检查改为 uname -a 中输出的版本。

### vs2010 

如果安装的是 vs2010 express, 则将目录构建平台改为 32 位。

执行下面的命令进行配置。

``` bash
./configure --with-debug-level=slowdebug --with-target-bits=32 --with-boot-jdk='/cygdrive/c/Program Files (x86)/java/jdk1.7.0_80'
```

### 安装 freetype

wget "http://gnuwin32.sourceforge.net/downlinks/freetype.php" -O /tmp/freetype-setup.exe

chmod +x /tmp/freetype-setup.exe

/tmp/freetype-setup.exe

Follow GUI prompts, and install to default directory "C:\Program Files (x86)\GnuWin32".
After installation, locate lib/libfreetype.dll.a and make a copy with the name freetype.dll.

### 编译

注意设置 classpath为 .;

### tmp 路径

在 cygwin 中调用 TMP 环境变量为 

``` bash
## 注意 C:\tmp 必须存在
TMP=C:\tmp
```

## vs2010 问题 >LINK : fatal error LNK1123: 转换到 COFF 期间失败: 文件无效或损坏

如果是 win7 环境，且后来手动安装了 .net framework 4.5 , 则其中的 cvtres.exe 不能共存。

此时需要将 C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\bin 下面的 cvtres.exe 重命名，这样就不会冲突了。

如果没有安装过  .net framework 4.5， 则不存在这个问题，

.net framework 的安装路径在 C:\Windows\Microsoft.NET\Framework

## 编译

``` bash
./configure --with-debug-level=slowdebug --with-target-bits=32
make
```

耐心等待，大约需要 20 分钟

编译成功之后在 build 目录下面。

## 使用 visual studio 和 eclipse 调试

### 生成 hotspot 工程

#### 导入环境变量

在 C:\cygwin 下创建一个 mintty.bat 的文件，写入下面的内容

``` bash
@echo off

C:
chdir C:\cygwin\bin
call "%VS100COMNTOOLS%vsvars32.bat" > nul

C:\cygwin\bin\mintty.exe -i /Cygwin-Terminal.ico -
```

执行这个脚本启动 cygwin 环境。

#### 生成工程

生成 hotspot 在 visual studio 配置文件。

``` bash
cd %OPENJDK_HOME%/hotspot/make/windows
chmod +x create.bat
## 下面的命令需要传递一个参数
## 这个参数就是上面编译好的 JDK 的路径在
## %OPENJDK_HOME%/build/windows-x86-normal-server-slowdebug/jdk
create <generated-jdk-home>
```

最终生成的工程在 %OPENJDK_HOME%/hotspot/build/vs-i486

此时，就可以使用 visual studio 的导入工程，将 JVM 工程导入。

导入后，耐心等待，构建完毕之后，就可以在 share\vm\runtime\thread.cpp
的 Threads::create_vm 方法处打断点进行调试了。

#### eclipse 和 visual studio 联调

在导入 visual studio 在 jvm 工程中进行如下配置。

JVM 右键 Properties ，打开 jvm Property Pages

依次选择 Configuration Properties -> Debugging

在 Command Arguments 后添加如下配置：

``` bash
## 添加 9999 调试端口
## 在 eclipse 中可以 为 my.World 添加一个远程调试
 -agentlib:jdwp=transport=dt_socket,address=9999,server=y,suspend=y -cp C:\Users\Administrator\workspace1\World\bin my.World
```

## $.参考
1. [openjdk8 source](http://download.java.net/openjdk/jdk8/)
2. [cygwin](https://www.cygwin.com/)
3. [visual_studio_2010_express](ed2k://|file|en_visual_studio_2010_express_x86_dvd_510419.iso|727351296|1D2EE178370FBD55E925045F3A435DCC|/)
4. [vs2010 问题 >LINK : fatal error LNK1123: 转换到 COFF 期间失败: 文件无效或损坏](http://www.cnblogs.com/newpanderking/articles/3372969.html)
5. [用Eclipse和Visual Studio全程调试上层的Java到HotSpot的C++代码](http://hllvm.group.iteye.com/group/topic/41290)
6. [高级编程语言的发展历程](http://kb.cnblogs.com/page/130672/)