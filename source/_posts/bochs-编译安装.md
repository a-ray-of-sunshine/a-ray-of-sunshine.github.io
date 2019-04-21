---
title: bochs-编译安装
date: 2017-9-10 17:22:08
---

## 编译可以使用 gdb 进行调试的 bochs

编译环境 win7 + cygwin-2.8.2(x86, 32位)

``` bash
svn checkout svn://svn.code.sf.net/p/bochs/code/tags/REL_2_6_7_FINAL/bochs bochs
cd bochs
./configure --enable-gdb-stub
make && make install
```

**注意在cygwin-2.8.2这个版本下，目前 bochs 2.6.7 可以编译成功，其它新版本均会在编译时报错**

## 调试

``` bash
## 在 bochsrc 中添加下面的配置
gdbstub: enabled=1, port=1234, text_base=0, data_base=0, bss_base=0
## 然后进行 gdb 远程调试
gdb
(gdb) target remote localhost:1234
```

## bochs 中安装 centos

### 创建磁盘

``` bash
## 创建一个 7T 大小的砸盘
## 默认大小 4.1M , 可以自行增长
bximage -mode=create -hd=7168G -imgmode=growing -q centos.img
```

## bximage

``` bash
bximage.exe -q -mode=create -hd=1024M -imgmode=growing centos.img
```

## 参考
1. [bochs home page](http://bochs.sourceforge.net/)
2. [bochs project page](https://sourceforge.net/p/bochs/code/HEAD/tree/)
3. [Bochs User Manual](http://bochs.sourceforge.net/doc/docbook/user/index.html)
4. [Disk Images](http://bochs.sourceforge.net/diskimages.html)
5. [SeaBIOS](https://www.seabios.org/SeaBIOS)