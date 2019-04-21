---
title: OpenStack-安装
date: 2019-1-31 16:57:27
---

``` bash
## 下载 centos6.10
docker pull centos:6.10
## 启动容器
docker run -tid centos:6.10
## 使用 exit 命令退出容器
## 退出容器之后使用下面的命令查询当前正在运行的容器
docker ps
## 可以看到对应的 容器 id
## 使用下面的命令再次进入容器
docker exec -ti <id> /bin/bash

## 如果关闭上面的容器。
docker stop <id>

##  再次启动上面的容器
docker start <id>

## 容器使用完毕完全删除这个容器
docker rm <id>
```

``` bash
## 宿主机和容器之前共享目录
## 在启动容器的时候使用下面的命令
## -v 设置共享目录 : 前面是 宿主机目录 后面是容器目录 
## -name 给容器 命名，之后用到 容器id 的地方都可以使用这个名称代替
docker run -ti -v /c/Users/dell/workspace/docker/share:/root/share  --name hadoop centos:6.10
```

 ``` bash
 ## 创建一个 network
docker network create --subnet=172.18.0.0/16 hadoop-net
docker run -ti --net hadoop-net --ip 172.18.0.2 hadoop-distributed-base

## 提交image
docker commit a6d06a446754  hadoop-distributed-base
```

``` bash
docker run -it --name hadoop-master \
--net hadoop-net \
--ip 172.18.0.2 \
--add-host node1:172.18.0.3 \
--add-host node2:172.18.0.4 \
--add-host node3:172.18.0.5 \
-p 9820:9820 \
-p 9870:9870 \
-p 9868:9868 \
-p 8088:8088 \
-p 19888:19888 \
-v /c/Users/dell/workspace/docker/share:/root/share \
-h master hadoop-distributed-base

docker run -it --name hadoop-node1 \
--net hadoop-net \
--ip 172.18.0.3 \
--add-host master:172.18.0.2 \
-p 9864 \
-v /c/Users/dell/workspace/docker/share:/root/share \
-h node1 hadoop-distributed-base

docker run -it --name hadoop-node2 \
--net hadoop-net \
--ip 172.18.0.4 \
--add-host master:172.18.0.2 \
-p 9864 \
-v /c/Users/dell/workspace/docker/share:/root/share \
-h node2 hadoop-distributed-base

docker run -it --name hadoop-node3 \
--net hadoop-net \
--ip 172.18.0.5 \
--add-host master:172.18.0.2 \
-p 9864 \
-v /c/Users/dell/workspace/docker/share:/root/share \
-h node3 hadoop-distributed-base
```

```
service sshd start
bin/hdfs namenode -format <cluster_name>
$HADOOP_HOME/sbin/start-dfs.sh
```

访问 webui 在windows上面其实是docker是运行在虚拟机环境中的。所以不能直接 通过 http://127.0.0.1:9870 来访问。

需要访问虚拟机IP。虚拟机IP在启动 docker 容器的时候会有显示。

http://192.168.99.100:9870

``` bash
## 创建目录
hdfs dfs -mkdir -p /user/root/input
## 删除目录
hdfs dfs -rm -R output
## 查看目录中的内容
hdfs dfs -cat output/*
```


$. Reference
1. [OpenStack Installation Guide](https://docs.openstack.org/install-guide/)
2. [OpenStack组件生态](https://www.openstack.org/software/)