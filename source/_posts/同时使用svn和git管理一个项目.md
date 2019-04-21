---
title: 同时使用svn和git管理一个项目
date: 2016-7-6 18:17:13
tags: [git,svn]
---

假设项目所在的目录是： d:\\project-demo\\

它本身是一个git仓库，则其下必然有 .git 目录。

检出 svn 到 上面的目录

svn checkout svnpath d:\\project-demo\\

此时，上面的目录多了一个 .svn 目录。

在 .gitignore 文件中添加 .svn/ 可以使得 svn目录 被 git 忽略

svn 如何 忽略 git

cd 到当前目录
cd d:\\project-demo\\

使用下面的命令
svn propset svn:ignore '.classpath' .

忽略目录 不要带 / 例如： .git/
svn propset svn:ignore '.git' .

