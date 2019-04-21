---
title: 配置vim
date: 2016-06-10 15:25:19
tags: [vim,vundle]
---

## 安装 vundle
1. 下载

	``` bash
	git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
	```

2. 配置

	参考：[VundleVim](https://github.com/VundleVim/Vundle.vim)

	**在 windows 环境下使用该插件，需要注意，必须将 git 的 cmd 目录添加到 path 环境变量中。
同时 curl.exe 也要调整，才能正常使用 vundle 来安装 vim 插件，否则，vundle将无法正常使用
具体的解决办法参考：[Vundle-for-Windows](https://github.com/VundleVim/Vundle.vim/wiki/Vundle-for-Windows)**

3. markdown语法高亮插件

	插件: [vim-markdown](https://github.com/plasticboy/vim-markdown)
	``` bash
	Plugin 'godlygeek/tabular'
	Plugin 'plasticboy/vim-markdown'
	```
	将上面的配置放到 .vimrc 中，在vim中执行 PluginInstall 即可。

