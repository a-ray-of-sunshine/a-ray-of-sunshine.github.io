---
title: Centos-自定义服务并添加到开机启动
date: 2017-5-9 16:29:19
---

## 自定义服务

以禅道为例

创建脚本 /etc/init.d/zentao

``` bash
#!/bin/bash
# zentao service
# chkconfig: 2345 55 25

otart(){
 /opt/zbox/zbox start
}

stop(){
 /opt/zbox/zbox stop
}

restart(){
 /opt/zbox/zbox restart
}


status(){
 /opt/zbox/zbox status
}


case "$1" in
  start)
        start
        ;;

  stop)
        stop
        ;;

  restart)
        restart
        ;;

  status)
        status
        ;;
  *)
        echo $"Usage: $0 {start|stop|restart|status}"
        ;;
esac
```

## 设置脚本权限

``` bash
chmod a+x zentao
```

## 添加服务到 chkconfig

``` bash
chkconfig --add zentao
```

## 设置开机启动

``` bash
chkconfig zentao on
```

## VMware vSphere 自动启动虚拟机

<主机> -> 配置 -> 软件 -> 虚拟机启动/关机

在这个界面上有一个 <属性...> 链接按钮 点击可以设置虚拟机自动启动。

点击 虚拟机启动/关机 可以刷新界面。

## Centos 安装扩展仓库

``` bash
yum install epel-release
```

添加 CentOS-Source.repo 到 /etc/yum.repos.d 目录中，

``` repo
[base-source]
name=CentOS-$releasever - Base Source
#baseurl=http://vault.centos.org/6.8/os/Source/
baseurl=http://vault.centos.org/centos/6.8/os/Source/
enabled=1

[updates-source]
name=CentOS-$releasever - Updates Source
#baseurl=http://vault.centos.org/6.8/updates/Source/
baseurl=http://vault.centos.org/centos/6.8/updates/Source/
enabled=1

[extras-source]
name=CentOS-$releasever - Extras Source
#baseurl=http://vault.centos.org/6.8/extras/Source/
baseurl=http://vault.centos.org/centos/6.8/extras/Source/
enabled=1

[centosplus-source]
name=CentOS-$releasever - Plus Source
#baseurl=http://vault.centos.org/6.8/centosplus/Source/
baseurl=http://vault.centos.org/centos/6.8/centosplus/Source/
enabled=1
```

## 安装 src.rpm

``` bash
cd /tmp
yumdownloader --source kernel-3.10.0-327.el7
rpmbuild --rebuild kernel-3.10.0-327.el7.src.rpm

rpm -i glibc-2.17-105.el7.src.rpm
cd /root/rpmbuild/SPECS
rpmbuild -bp glibc.spec
```

## 使用 Visual Studio 进行 Linux 内核源码调试

需要的工具 VisualKernel + VisualGDB + Visual Studio 注意 Visual Studio Express 版本不支持。

同时还需要安装一个需要调试的 Linux 虚拟机。可以使用 Oracle VM VirtualBox 安装 Centos 7.2 。

虚拟机安装成功之后，设置好网络，要能够和外网连通，同时和本地的机器也能够 ping 能虚拟机。

按照上面的方法配置源码仓库。配置完成之后使用 yumdownloader 命令，进行验证。

按照这个连接 [Building and modifying Linux Kernel with Visual Studio](http://sysprogs.com/VisualKernel/tutorials/kernel/) 给的步骤进行源码编译。

**注意在选择需要导入的源码目录的时候，不要选择全部的源码，源码文件太多，可以无法导入，所以可以先选择 kernel 目录导入即可，后续可以使用 add 方法将需要的目录导入进来**

## 使用 Visual Studio 导入或者创建 Linux 工程

使用 File -> New -> Project... -> VisualGDB -> Liunx Project Wizard

填写工程名称之后，进入下一步。到 New Linux Project 界面

在这个界面，可以选择 Create a new project 创建一个 Linux 工程。**创建工程的时候最好选择 Use GNU Make**

也可以选择 Import a project. 导入一个已经在 虚拟机 中编译好的工程。

## 参考
1. [CentOS 开机启动自定义脚本详解及实现](http://www.jb51.net/article/98471.htm)
2. [chkconfig命令](http://blog.csdn.net/youyu_buzai/article/details/3956845)
3. [VMware vSphere 4/5/5.5 设置虚拟机随机 自动启动](http://www.aikaiyuan.com/7477.html)
4. [Building and modifying Linux Kernel with Visual Studio](http://sysprogs.com/VisualKernel/tutorials/kernel/)