---
title: win提升效率
date: 2016-06-10 09:35:29
tags: [win,效率]
---

## 遇到的问题
1. 使用hexo 写博文，使用 hexo n 命令之后，不会打开写作软件


## 快速搭建终端环境
1. cmder
2. chocolatey
3. vim

## 提升效率软件

## cygwin

### cygwin 安装 tmux

**cygwin 2.8.2 已经集成好了，可以直接下载安装，不需要编译安装**

需要提前安装好 gcc-core 和 g++

分别下载 libevent， ncurses， tmux 到 /usr/src/ 路径下

使用下面的命令安装

``` bash
./configure --prefix=/usr
make && make install
```


## 相关连接
1. [Windows下效率必备软件](http://www.jeffjade.com/2015/10/19/2015-10-18-Efficacious-win-software/)
2. [vim安装markdown插件](http://www.jianshu.com/p/24aefcd4ca93)
3. [libevent](http://libevent.org/)
4. [ncurses](http://ftp.gnu.org/gnu/ncurses/)
5. [tmux](https://github.com/tmux/tmux/releases)