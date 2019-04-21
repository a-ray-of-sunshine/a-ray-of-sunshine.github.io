---
title: redis-调试环境搭建
date: 2019-4-20 8:46:39
---

## 创建redis源码容器

``` bash
docker run -it --name redis-source \
-p 10022:22 \
-h redis-source \
--privileged \
--security-opt seccomp=unconfined \
dev-base:v1
```

容器有一个安全规则在执行容器中执行 gdb 的时候会有问题。所以使用上面的参数 

``` bash
--privileged \
--security-opt seccomp=unconfined \
```

## 搭建编译环境

``` bash
yum install 
```


## $. 参考
1. [debugging-docker-containers-from-visual-studio](https://blogorama.nerdworks.in/debugging-docker-containers-from-visual-studio/)
2. [如何在Docker容器内部使用gdb进行debug](https://blog.csdn.net/snipercai/article/details/80408569)
3. [为什么在Docker里使用gdb调试器会报错](https://yq.aliyun.com/articles/674757)