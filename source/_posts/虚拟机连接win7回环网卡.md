---
title: 虚拟机连接win7回环网卡
date: 2016-9-24 18:15:04
---

## 场景

在 VirtualBox 中安装的虚拟机 centos7，设置网卡，模式是桥连。可以共享物理机的网络，但是当 host 主机 win7 在，没有连接任何网络的时候。 虚拟机 centos7 是不会分配ip地址的。所以 此时 host win7 没法和 虚拟机 centos7 进行 ssh 连接。

此时就可以使用一个虚拟网卡。


计算机 --> 管理 --> 系统工具 --> 设备管理器 --> 网络适配器

此时选择， 操作 --> 添加过时硬件 --> 下一步 --> 安装我手动从列表中选择的硬件。 --> 网络适配器 --> 选择 Microsoft ： Microsoft Lookback Adapter（倒数第三个） 安装即可。

安装完成之后，选择，控制面板--> 网络和共享中心 --> 更忙适配器设置  到网络连接界面， 其中会出现一个 本地连接2 。

然后在  本地连接2 中右键 属性，设置成静态ip，例如： 192.168.0.100. 确定即可。

然后到虚拟机中，进行配置。将虚拟机桥接的网卡，设置成静态的ip

和上面的ip，设置到同一个网段，例如： 192.168.0.101, 关闭，虚拟机。在虚拟机的管理工具中，将桥接网卡设置成前面配置的虚拟网卡Microsoft Lookback Adapter。

此时，重新启动虚拟机，192.168.0.100 和 192.168.0.101, 就可以互相 ping 通了。

## 命令行启动虚拟机

``` bash
VBoxManage startvm ubuntu --type gui
```

## Virutal Box 配置多个网卡

1. 在 设置 -> 网络 界面启用 网卡1 和 网卡2
2. 网卡1 连接方式 选择 网络地址转换(NAT) 用来连通外网
3. 网卡2 连接方式 选择 桥接网卡 用来物理宿主机 连接此虚拟机
4. 启动虚拟机 执行 ip addr 命令，获得网卡名称
5. 在 /etc/sysconfig/network-scripts/ 在配置 ifcfg-ethx
6. 将 HWADDR 设置成 网卡对应的 MAC 地址
7. ONBOOT 设置为 yes

## centos 安装编译环境

* yum install git
* yum groupinstall 'Development tools'
