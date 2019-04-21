---
title: 使用 github 和 hexo 搭建个人博客
date: 2016-06-04 16:32:43
tags: [hexo,nodejs,博客]
---
#使用 github 和 hexo 搭建个人博客

## 搭建过程
### 安装基本依赖
1. git
2. nodejs

### 安装过程
#### 安装 hexo

    npm install hexo-cli -g
	hexo init blog
	cd blog
	npm install
	hexo server

#### 创建 github pages 仓库   
仓库名称必须是： username.github.io

#### 配置 hexo
   在生成的 blog 文件夹中有一个文件：_config.yml，编辑如下
在最下面的：deploy 配置项中增加下面的配置：

    type: git
    repository: git@github.com:XXX/XXX.github.io.git
    branch: master
其中 repository 地址就是步骤2 中创建的地址。

#### 部署 hexo 到 github
需要安装插件：
	
	npm install hexo-deployer-git --save
插件安装完成之后，执行以下命令：
	
	hexo deploy
这个命令会创建好的博客发布到 github，命令执行完成之后，使用下面的地址访问：

	username.github.io
就可以访问发布的博客了。

## 写博客
### 写博客的流程
使用以下命令：

	 hexo new [layout] <title>
创建一个新的博客文章
创建好之后，会生成对应的 title.md 文件，在里面编写即可。
### 新博文的发布
使用以下命令

	hexo generate #生成
	hexo deploy   #部署
使用 generate 的过程其实就是，将 md 文件编译成 静态的 html 文件的过程

使用 deploy 其过程，完全重新生成仓库，然后提交

## 使用新的主题
