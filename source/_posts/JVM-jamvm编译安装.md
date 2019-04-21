---
title: JVM-jamvm编译安装
date: 2017-1-22 15:09:47
---

## 编译环境

操作系统 centos6.8

0.  调用 auotgen 脚本

1. autoconf2.1
2. libtool

1. 安装 zlib

zlib 是 jamvm 的惟一的依赖。下载地址见下面的链接。

在这里使用 zlib-1.2.3

``` bash
tar axvf zlib-1.2.3.tar.gz
cd zlib-1.2.3
./configure --shared --prefix=/usr
make install

## 或者直接安装 
yum install zlib-devel
```

2. 编译安装 jamvm

下载到 jamvm-2.0.0.tar.gz 

``` bash
tar axvf jamvm-2.0.0.tar.gz
cd jamvm-2.0.0
./configure --with-java-runtime-library=openjdk7
## 在cygwin下编译
./configure --host=i686-pc-linux  CFLAGS="-ggdb3 -O0" CXXFLAGS="-ggdb3 -O0"
make install
```

最后安装完成之后，其目录在 /usr/local/jamvm 中。

3. 安装 openjdk

``` bash
yum install java-1.7.0-openjdk
```

安装路径 `/usr/lib/jvm/jre-1.7.0-openjdk.x86_64/lib/amd64/server`

貌似 openjdk 不支持。

4. 使用

在 oracle-jdk 1.7 中可以正常使用。方法就是直接替换 $(JDK_HOME)/jre/lib/amd64/server/libjvm.so 文件，就可以了。

``` bash
## 执行以下命令
java -version
	java version "1.7.0_67"
	Java(TM) SE Runtime Environment (build 1.7.0_67-b01)
	JamVM (build 2.0.0, inline-threaded interpreter)
```

可以看到 jvm 已经从 hotspot 替换为 JavmVM 了。

5. 编译 GNU classpath

``` bash
 yum install GConf2-devel
 yum install gtk2
 yum install gtk2-devel
 yum install libgcj-devel
 yum install java-1.5.0-gcj
 yum install java-1.5.0-gcj-devel
 yum install antlr
 
 ./configure CFLAGS=-fno-strict-aliasing
```

安装成功之后的class 位于 `/usr/local/classpath/share/classpath ` 路径下分别是 glibj.zip 和 tools.zip 


## 源码学习

### 使用 gdb 调试 jamvm

``` bash
## 结构体格式化显示
set print pretty on

# main_class 是 Class 类型
# main_class+1 则存储的是 ClassBlock
print *(ClassBlock*)(main_class+1)

## 把地址按照某个类型进行解释
## Type 表示类型，addr表示地址。
print *(<Type>*)(<addr>)

## 打印字符串。
print (char*)0x751770
```

### 比较

hotspot 定义 Class 是在 openjdk\hotspot\src\share\vm\classfile 这个路径下的所有文件就是用来加载解析，其中 classFileParser.cpp 这个文件中的是核心用来解析 .class 文件。ClassFileParser::parseClassFile 方法向外部提供接口。

## 编译openjdk

编译环境 centos6.8

编译 hotspot `/usr/local/src/openjdk/hotspot/make`

``` bash
yum groupinstall "Development tools"
wget http://www.java.net/download/openjdk/jdk7/promoted/b147/openjdk-7-fcs-src-b147-27_jun_2011.zip

 ## 安装 jdk
 yum install java-1.6.0-openjdk-devel
 ## 设置 bootjdk
 ALT_BOOTDIR=/etc/alternatives/java_sdk_1.6.0
 export  ALT_BOOTDIR
 
 ## 编译时会产生 32 位的库
 ## 依赖 32 位的 glibc 和 libstdc++
 yum install glibc-devel.i686
 yum install libstdc++-devel.i686

```

http://download.java.net/openjdk/jdk7

## 安装 cygwin

**注意安装32位**

### 安装 [cygwin setup-x86](https://cygwin.com/setup-x86.exe)

### 安装 cyg-apt

通过 setup.exe 安装 lynx

``` bash
lynx -source https://raw.githubusercontent.com/transcode-open/apt-cyg/master/apt-cyg > apt-cyg
install apt-cyg /bin
ln -s /bin/apt-cyg /bin/yum
yum install wget
```

此后，就可以使用 yum 进行安装了。

## 在 cygwin 中编译安装 gnuclasspath

``` bash
## 0. 搭建编译环境
yum install gcc-java
yum install autoconf
yum install libtool
yum install automake
yum install gettext-devel
## win7 + cygwin 2.8.2 32位
## 0.1 安装libiconv
cd /usr/src
wget https://ftp.gnu.org/pub/gnu/libiconv/libiconv-1.14.tar.gz
tar axvf libiconv-1.14.tar.gz
cd libiconv-1.14
./configure
make && make install
## 0.2 编译 ecj1.exe
wget http://ftp.mirrorservice.org/sites/sourceware.org/pub/java/ecj-4.5.jar
gcj -oecj1.exe --main=org.eclipse.jdt.internal.compiler.batch.GCCMain ecj-4.5.jar
cd /usr/lib/gcc/i686-pc-cygwin/5.4.0
mv ecj1.exe ecj1.exe.origin
cp ~/ecj1.exe .

## 1. 检出代码
cd /usr/local/src
git clone git@github.com:a-ray-of-sunshine/classpath.git
cd classpath
git checkout dev

## 2. 编译
./autogen.sh
./configure --disable-gtk-peer --disable-gconf-peer --disable-gjdoc --with-antlr-jar=/usr/src/classpath-code/antlr-2.7.5.jar
make && make install
```

``` bash
yum install make
yum install gcc-g++
yum install gcc-java
cd /usr/src/classpath-0.99
wget http://www.antlr.org/download/antlr-4.7-complete.jar
./configure --disable-gtk-peer --disable-gconf-peer --disable-gjdoc --with-antlr-jar=antlr-4.7-complete.jar
mkdir /usr/local/lib/javalib
wget http://central.maven.org/maven2/org/eclipse/jdt/core/compiler/ecj/4.6.1/ecj-4.6.1.jar

wget http://www.antlr3.org/download/antlr-3.5.2-complete.jar

## 编译 libiconv
## 生成 libiconv.dll.a
ln -s  /usr/local/lib/libiconv.dll.a /lib/libiconv.dll.a
## 编译 ecj1.exe
gcj -oecj1.exe --main=org.eclipse.jdt.internal.compiler.batch.GCCMain /usr/share/java/ecj.jar

gcj -oecj4.exe --main=org.eclipse.jdt.internal.compiler.batch.GCCMain ecj-4.5.jar

 ./configure --disable-gtk-peer --disable-gconf-peer --disable-gjdoc  --with-antlr-jar=/usr/src/classpath-0.99/antlr-3.5.2-complete.jar

D:\cygwin\usr\src\classpath-0.99\tools\gnu\classpath\tools\gjdoc\expr>java -cp D
:\cygwin\usr\src\classpath-0.99\antlr-2.7.5.jar antlr.Tool java-expression.g
 ```

### cygwin 安装 gnuclasspath
``` bash
## 需要设置 CLASSPATH，
## 必须将其设置成如下形式 
## CLASSPATH=,;
## 如果使用 gcc-java 进行安装
## 则不要设置 JAVA_HOME

yum install gcc-java

wget http://www.antlr2.org/download/antlr-2.7.5.jar

 ./configure --disable-gtk-peer --disable-gconf-peer --disable-gjdoc --with-antlr-jar=/usr/src/classpath-code/antlr-2.7.5.jar
```

## cygwin 中安装 jamvm

``` bash
cd /usr/local/src
git clone git@github.com:a-ray-of-sunshine/jamvm.git
git checkout dev
./autogen.sh --host=i686-pc-linux  CFLAGS="-ggdb3 -O0" CXXFLAGS="-ggdb3 -O0"
make && make install

cd /usr/local/classpath/lib/classpath
ln -s cygjavaio-0.dll libjavaio.so
ln -s cygjavalang-0.dll libjavalang.so
ln -s cygjavalangmanagement-0.dll libjavalangmanagement.so
ln -s cygjavalangreflect-0.dll libjavalangreflect.so
ln -s cygjavanet-0.dll libjavanet.so
ln -s cygjavanio-0.dll libjavanio.so
ln -s cygjavautil-0.dll libjavautil.so
```

## $. 参考
1. [zlib](https://sourceforge.net/projects/libpng/files/zlib/1.2.3/)
2. [jamvm](https://sourceforge.net/projects/jamvm/files/jamvm/)
3. [How to download and install prebuilt OpenJDK packages](http://openjdk.java.net/install/)
4. [gnu classpath](http://www.gnu.org/software/classpath/)
5. [Java™ Native Interface](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/)
6. [使用jconsole监控JVM虚拟机](https://yflog.com/2015/06/15/%E4%BD%BF%E7%94%A8jconsole%E7%9B%91%E6%8E%A7JVM%E8%99%9A%E6%8B%9F%E6%9C%BA/)
7. [classpath-0.99](ftp://ftp.gnu.org/gnu/classpath/classpath-0.99.tar.gz)
8. [Strict Aliasing，神坑？](http://blog.kongfy.com/2015/09/strict-aliasing%EF%BC%8C%E7%A5%9E%E5%9D%91%EF%BC%9F/)
9. [Understanding C/C++ Strict Aliasing](http://dbp-consulting.com/tutorials/StrictAliasing.html)
10. [Understanding Strict Aliasing](http://cellperformance.beyond3d.com/articles/2006/06/understanding-strict-aliasing.html)
11. [cygwin setup-x86](https://cygwin.com/setup-x86.exe)
12. [gcc](ftp://sourceware.org/pub/gcc/releases/gcc-5.4.0/)
13. [ecj](ftp://sourceware.org/pub/java/)
14. [libiconv](http://www.gnu.org/software/libiconv/)
15. [antlr2](http://www.antlr2.org/download.html)
16. [ecj--](http://ftp.mirrorservice.org/sites/sourceware.org/pub/java/)