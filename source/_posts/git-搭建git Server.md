---
title: git-搭建git Server
date: 2017-9-15 14:59:22
---

## 安装步骤

``` bash
yum install git
cd /usr/local/src
git clone git://github.com/res0nat0r/gitosis.git
cd gitosis
yum install python-setuptools
python setup.py install

useradd \
      -r \
      -s /bin/sh \
      -c 'git version control' \
      -d /home/git \
      git
mkdir -p /home/git
chown git:git /home/git
```

``` bash
useradd -r -s /sbin/nologin -c 'git version control' -d /home/git -g git
 git
useradd git
su git
cd /usr/local/src
git clone https://github.com/sitaramc/gitolite
mkdir -p $HOME/bin
gitolite/install -to $HOME/bin
## 添加一个管理员用户
## 使用 ssh-keygen 生成
## 添加的时候将 id_rsa.pub 改成 YourName.pub
gitolite setup -pk YourName.pub
ssh-keygen.exe -q

## 创建仓库
cd /home/git/repositories
git init --bare --shared myproject.git

## 配置仓库权限
repo mypro
    RW      =   @all

## 用户管理
git clone git@host:gitolite-admin
## gitolite-admin 中存储着 ssh pub key
## keydir 存储 key, conf 存储权限配置信息
## 如果需要添加一个新的用户，则使用下面的命令生成 key
ssh-keygen.exe -q
cd ~/.ssh
cp id_rsa.pub gitolite-admin/keydir/<username>.pub
## 然后到 conf 目录中编辑 gitolite.conf
repo myproject
    RW      =   tom jack
上面的配置表示 tom 和 jack 用户对仓库 myproject 具有读写权限
```

## 安装 gitweb

``` bash
yum install epel-release
yum install lighttpd gitweb

mvn archetype:generate -DarchetypeCatalog=local -DgroupId=com.example  -DartifactId=example -
Dversion=1.0 -DarchetypeGroupId=com.nebula -DarchetypeArtifactId=nebula-framework-archetype -Di
nteractivMode=false
```

## git 子模块

``` bash
## 添加子模块
git submodule add https://github.com/chaconinc/DbConnector
git commit -am 'added DbConnector module'

## 检出带有子模块的项目
## 1. 
git clone --recursive git@host:project
## 2. 或者使用下面的命令，首先 cd 子模块的目录中，然后执行下面的命令
git submodule init
git submodule update

## 更新子模块
## 1. 全局方式
git submodule update --remote
## 1. 首先 cd 子模块的目录中，然后执行下面的命令
git fetch
git merge origin/master

## 完成之后，子模块路径会发生一个 new commit 的提示。
## 从远程更新子模块并合并到本地
git submodule update --remote --merge

## 删除子模块

## 检出指定分支
## 1. 修改配置
##    修改 .gitmodules 文件
## 设置要检出的分支
[submodule "mypro/src/main/webapp/pages"]
        path = mypro/src/main/webapp/pages
        url = git@host:mypro-webpages
        branch = dev

git submodule update --checkout --remote  htcem-web/src/main/webapp/pages
```

多模块项目协作模式。模块添加之后，切换到指定的工作分支。然后，使用上面检出指定的子模块分支的方法，检出分支。注意这个检出操作只是在当前的工作分支上。

最终功能开发完成之后，可以在，主项目的 master 分支上，进行 子模块 的 update.

### 添加子仓库忽略

一旦子仓库开始进行更新了， 此时使用 `git status` 的时候就会表现出 子仓库的路径会发生一个 modify，需要被提交，

而一般的我们使用子仓库的时候这个 path 的修改不需要提交。所以使用下面的方法，这个子仓库路径的变化添加到忽略中。

在子仓库的配置中添加下面的配置。`ignore = all`

```
[submodule "htcem-web/src/main/webapp/pages"]
        path = htcem-web/src/main/webapp/pages
        url = git@192.168.0.10:htcem-webpages
        ignore = all
```

## git 导出代码

``` bash
# 导出当前分支上最新的代码到 latest.zip 中。
git archive -o latest.zip HEAD
```

## gitolite 使用异常问题

在时候在 clone 仓库的时候，会出现下面的问题

``` bash
FATAL: split conf set, gl-conf not present for 'myproject'
fatal: 无法读取远程仓库。
```

应该是 gitolite 的配置缓存出现问题了。 解决办法

``` bash
su git
cd 
cd .gitolite
mv gl-conf.cache gl-conf.cache.bak
## 然后修改 gitolite-admin 仓库的权限配置
## 对这个仓库进行一次 push
## 上面的 gl-conf.cache 将从新生成。
## 之后上面的问题就解决了
```

## 查看日志

`--graph` 表示图形化展示日志。

``` bash
git log --pretty=format:"%h %s" --graph
```

git 回退本地提交（commit）

git reset --soft <commit-id>

## 参考
1. [gitosis](https://github.com/res0nat0r/gitosis)
2. [gitolite](https://github.com/sitaramc/gitolite)
3. [Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
4. [How do I remove a submodule?](https://stackoverflow.com/questions/1260748/how-do-i-remove-a-submodule)
5. [git 导出源码](https://git-scm.com/docs/git-archive)
6. [How to ignore changes in git submodules](http://www.nils-haldenwang.de/frameworks-and-tools/git/how-to-ignore-changes-in-git-submodules)