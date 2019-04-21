---
title: windows命令行增强工具cmder
date: 2016-06-05 11:02:46
tags: [windows,命令行,cmder]
---
## cmder
官网：[cmder](http://cmder.net/)

下载地址：[cmder github](https://github.com/cmderdev/cmder/releases)

## chocolatey
windows下的软件下载工具

官网：[chocolatey](https://chocolatey.org/)

安装：
``` bash
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))" && SET PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin
```