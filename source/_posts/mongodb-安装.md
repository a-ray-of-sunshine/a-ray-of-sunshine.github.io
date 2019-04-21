---
title: mongodb-安装
date: 2018-3-26 10:39:45
---

## 常用操作

1. 启动 MongoDB
``` bash
## 启动 mongo
mongod --dbpath /var/lib/mongodb/ --bind_ip_all &
mongod --dbpath /var/lib/mongodb/ --bind_ip 192.168.1.123

## 连接 mongo
mongo --host 192.168.1.123:27017
mongo mongodb://xxxxxxxxx
```

### 集合操作

``` bash
## 创建集合：
db.createCollection('bak_collection')

## 删除
db.userrank.remove({age: 132});

## 查找 
db.userrank.find({rank: {$gt: 10}});
db.userrank.remove({rank: {$gt: 9},{"periodId" : "5ac34d7b8da8f115431fb167"}});

## 更新
db.userrank.update({"rank": {$gt: 75}}, {$set: {"periodId": "5ac47a52442ab4eff15485cd"}}, false, true)

## 备份数据：
db.userrank.find().forEach(function(x){db.userrank_copy.insert(x)});

## 导出数据
mongoexport --host <ip:port> --authenticationDatabase admin -u root -p <password> -d <database> -c <collection> -q '{rank:{$gt:8}, rankTime:1523462400, userId:{$lt:376}}' -o userrank.1523462400.json

## 导入数据
mongoimport --host <host:port> --authenticationDatabase admin -u root -p <port> -d app -c userrank_test_only --file userrank.1523462400.json
```

## $. 参考
1. [mongodb download](https://www.mongodb.com/download-center)
2. [mongodb centos download page](https://www.mongodb.org/dl/linux/x86_64-rhel62)
3. [安装文档](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-linux/)
4. [MongoDb的“not master and slaveok=false”错误及解决方法](http://www.cnblogs.com/anny0404/p/5276169.html)