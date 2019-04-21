---
title: centos7在vm中扩展容量
date: 2016-9-23 16:51:42
---


参考：

[VirtualBox虚拟机增加Centos根目录容量](c)

按照上面的步骤一步一步进行即可，但是，如果是centos7系统，则最后一步非常重要的刷新逻辑分区的步骤，有变化。在原文章中是：

``` bash
resize2fs /dev/centos/root
```

在 centos7 中使用这个命令将出错。解决办法：

[resize2fs-bad-magic-number-in-super-block-while-trying-to-open](http://stackoverflow.com/questions/26305376/resize2fs-bad-magic-number-in-super-block-while-trying-to-open)

Finally perform an online resize to resize the logical volume, then check the available space.

``` bash
xfs_growfs /dev/centos/root
df -h
```

也就是说 centos7 中使用 xfs_growfs 来刷新容量，而不是resize2fs


1. fdisk /dev/sda

	n --> p --> 3 --> w
	
	重新启动机器
	
2. mkfs.ext4 /dev/sda3

3. vgdisplay 查看 卷名 VG Name

	centos

4. pvcreate /dev/sda3

5. vgextend centos /dev/sda3

6. lvextend /dev/centos/root /dev/sda3

7. xfs_growfs /dev/centos/root

## centos 资源下载

[centos6.8 源码仓库](http://vault.centos.org/6.8/os/Source/)

[centos6.8 源码](http://vault.centos.org/6.8/os/Source/SPackages/)

[CentOS下安装VIM](http://www.centoscn.com/image-text/install/2014/0830/3612.html)

``` bash
yum install vim-enhanced
```

* 查看rpm 包中的内容

	rpm -qip *.src.rpm
	
* rpm -ivh *.src.rpm

* rpmbuild -bs 

* tar tvf SOURCES/glibc-2.12-2-gc4ccff1.tar.bz2


## linux 相关

[int 0x80h](http://ju.outofmemory.cn/entry/1273)

[Linux系统调用详解（实现机制分析）--linux内核剖析（六）](http://blog.csdn.net/gatieme/article/details/50779184)

[Hello from a libc-free world! 不使用libc编写代码](https://blogs.oracle.com/ksplice/entry/hello_from_a_libc_free)

[The Definitive Guide to Linux System Calls](http://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/)

[Linux Systemcall Int0x80方式、Sysenter/Sysexit Difference Comparation](http://www.cnblogs.com/LittleHann/p/4111692.html)


从用户模式到内核模式，windows 平台的实现：

[The Sysenter Instruction and 0x2e Interrupt](http://resources.infosecinstitute.com/the-sysenter-instruction-and-0x2e-interrupt/)

[Windows NT - NATIVE API](http://www.oldlinux.org/Linux.old/docs/interrupts/int-html/rb-4249.htm)

[浅析Windows系统调用——2种切换到内核模式的方法](http://shayi1983.blog.51cto.com/4681835/1710861)

## glibc, man-pages

[glibc的官网](https://www.gnu.org/software/libc/)

[glibc下载](https://ftp.gnu.org/gnu/libc/)

[libc文档](https://www.gnu.org/software/libc/manual/html_mono/libc.html)

[The Linux man-pages project](https://www.kernel.org/doc/man-pages/)

man-pages有关于 linux 的调用的文档。例如关于 open 函数的文档： 

> `man 2 open`
>
> Given  a  pathname  for a file, open() returns a file descriptor, a small, non-negative integer for use in subsequent system calls (read(2), write(2), lseek(2),  fcntl(2),  etc.).   The  file descriptor  returned  by a successful call will be the lowest-numbered file descriptor not cur-rently open for the process.
> 
> By default, the new file descriptor is set to  remain  open  across  an  execve(2)  (i.e.,  the FD_CLOEXEC file descriptor flag described in fcntl(2) is initially disabled; the Linux-specific O_CLOEXEC flag, described below, can be used to change this default).  The file offset  is  set to the beginning of the file (see lseek(2)).


libc 中关于 open 的文档。

> The open function creates and returns a new file descriptor for the file named by filename. Initially, the file position indicator for the file is at the beginning of the file. The argument mode (see Permission Bits) is used only when a file is created, but it doesn’t hurt to supply the argument in any case.

可以看到在 man-pages 中对于 open 函数的描述和 libc 文档中的描述，有所不同。`man 2 open` 中看到的 open 函数，到底是不是 glibc 库提供的 open 函数呢？

从两个方面考虑：

1. 查看 man man-pages

> System calls: Those functions which must be performed by the kernel.
>
> Library calls: Most of the libc functions.
> 
> source    
> 
> The source of the command, function, or system call.
> 
> For  those  few man-pages pages in Sections 1 and 8, probably you just want to
write GNU.
> 
> For system calls, just write Linux.  (An earlier practice  was  to  write  the version  number  of  the  kernel  from  which  the manual page was being writ-ten/checked.  However, this was never done consistently, and so  was  probably worse than including no version number.  Henceforth, avoid including a version
number.)
> 
> For library calls that are part of glibc  or  one  of  the  other  common  GNU libraries, just use GNU C Library, GNU, or an empty string.
> 
> For Section 4 pages, use Linux.
> 
> In cases of doubt, just write Linux, or GNU.

由于上面的摘录的信息可知，当调用

`man 2 open` 查看时，其实查看到的 system call, 即内核提供给用户空间的系统调用。

而 libc 中关于 open 方法的描述有是什么呢？

`man syscalls`

System calls and library wrapper functions

System  calls are generally not invoked directly, but rather via wrapper functions in glibc (or
perhaps some other library).  For details of direct invocation of a system call, see  intro(2).
Often,  but  not always, the name of the wrapper function is the same as the name of the system
call that it invokes.  For example, glibc contains a  function  truncate()  which  invokes  the
underlying "truncate" system call.

Often  the glibc wrapper function is quite thin, doing little work other than copying arguments
to the right registers before invoking the system call, and then  setting  errno  appropriately
after  the  system  call  has  returned.   (These  are  the  same  steps  that are performed by
syscall(2), which can be used to invoke system calls for which  no  wrapper  function  is  pro-
vided.)
       
Roughly speaking, the code belonging to  the  system  call  with  number  __NR_xxx  defined  in /usr/include/asm/unistd.h  can  be  found  in the kernel source in the routine sys_xxx().  (The dispatch table for i386 can be found in  /usr/src/linux/arch/i386/kernel/entry.S.)


通过上面的描述可以看到，通过 syscall 方法进行（man 2 syscall） 的方式，也是间接的系统调用。其实和被 glibc wrapper 过的系统调用函数是相同的。

也可以通过 _syscall 方法，进行直接的系统调用。这样就没有 glibc 对系统调用进行包装了。

** syscall 和 _syscall 其自身，其实就是 glibc 提供的两个函数，它们自身并不是 系统调用**， 虽然在 man-pages 中它们被归类的 2 这个区域内，但是却只是glibc 提供的普通函数而已。[System-Calls](https://www.gnu.org/software/libc/manual/html_mono/libc.html#System-Calls)

**Starting around kernel 2.6.18, the _syscall macros were removed from header files  supplied  to user  space.   Use  syscall(2)  instead. ** 所以基于内核 kernel 2.6.18 及其以上版本，编译的 glic, 中已经不再提供 _syscall 函数了，所以在代码中也不能使用了。

`man 2 intro` 中也有关于 系统调用的 说明。

[汇编程序的Hello world](http://www.cnblogs.com/orlion/p/5316519.html)
为什么 glibc 需要将系统调用进行包装呢？

> 其内部实现就是把传进来的三个参数分别赋给ebx、ecx、edx寄存器，然后执行movl $4,%eax和int $0x80两指令。这个函数不可能完全用C代码写，因为任何C代码都不会编译生成int指令，所以这个函数有可能完全用汇编写的，也有可能是C用内联汇编写的，

因为，如果想直接调用内核提供的系统调用，则必须，使用 int 指令，进行中断操作。 而对于 c 语言来说，任何c语言代码都无法生成 int 指令的代码，所以当我们使用系统调用的时候，就需要编写 汇编代码 来进行调用。很明显增加了开发的复杂度，同时直接写汇编指令，可能会导致源码级别的不可移植性。

所以，即使我们的程序不需要 glibc 做参数的复制，返回值 errno 的设置，等等通用操作，让 glibc 对 系统调用的汇编代码做一层薄的封装，也是有必要的。这样系统调用就以 c 库函数的形式提供给我们调用，降低了复杂度，增加了可移植性（不用直接写汇编代码）。

那么，到底如何查看，我们所安装的 glibc 的文档呢？

其实在安装 glibc-devel 的时候，这些文件已经被安装了。在路径 /usr/share/info 下面。

``` bash
# 就可以查看到我们所使用到的 glibc 的真正的文档
# glibc 所提供（导出）的所有函数都在这个文档中有描述
info libc

cd /usr/share/info
ll libc.info*
```

### 常用命令

``` bash
# 查看共享库的导出函数
objdump -tT /lib64/libc-2.12.so
```

## glibc 实现的标准

[glibc Release](https://sourceware.org/glibc/wiki/Release/)

[glibc所有历史版本的仓库](https://sourceware.org/git/?p=glibc.git;a=tags)

[POSIX.1-2008 规范](http://pubs.opengroup.org/onlinepubs/9699919799/)

[IEEE Std 1003.1, 2004 Edition](http://pubs.opengroup.org/onlinepubs/009695399/)

[POSIX标准](http://lib.yoekey.com/?p=179)

[C的标准库与Gnu Libc](http://dev.gameres.com/Program/Other/bcxszyforgameres/bcxszy/xisofts.sinaapp.com/@p=100.htm)

[opengroup unix](http://www.opengroup.org/unix)

[UNIX® Standards](http://www.opengroup.org/standards/unix)

[UNIX标准化及实现之UNIX标准化、UNIX系统实现、标准和实现的关系以及ISO C标准头文件](http://www.cnblogs.com/nufangrensheng/p/3496106.html)

[c标准](http://port70.net/~nsz/c/)