---
title: Windows 上安装 docker
date: 2019-4-21 10:34:47
---

## install docker in windows

1. download docker toolbox
如果在 win7 或者 win8 上使用 docker ，可以首先安装下面的工具。

https://download.docker.com/win/stable/DockerToolbox.exe

2. config docker toolbox to cmder

``` bash
"C:\Program Files\Git\bin\bash.exe" --login -i "C:\Program Files\Docker Toolbox\start.sh"
```

3. 基本配置

``` bash
# 安装 centos 镜像
docker pull centos:6.10
# 启动 centos 
docker run -ti centos:6.10
# 启动 centos 挂载本地目录 ~/workspace/docker 到 /root/data 下面
# /root/data data 目录如果不存在会自动创建。
# 在windows 下面本地目录如果是非C盘，好像会有问题。最好直接挂C盘下的目录。
docker run -it --name hadoop-env -v ~/workspace/docker:/root/data  centos:6.10
```

## 参考
1. [Get started with Docker for Windows](https://docs.docker.com/docker-for-windows/)
2. [Docker Toolbox overview](https://docs.docker.com/toolbox/overview/)
3. [Running Docker Quickstart Terminal Inside ConEmu](https://jayvilalta.com/blog/2016/05/02/running-docker-toolbox-inside-conemu/)