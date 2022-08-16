---
title: "CentOS下载RPM包"
categories:
  - CentOS
tags:
  - CentOS
---
有的时候服务器无法访问外网，不能使用yum命令下载RPM软件包，这种时候可以通过在其他服务器下载相关的RPM软件包再传到该服务器使用RPM命令进行安装。

我们需要在服务器上安装Yumdownloader
```bash
yum install yum-utils
```
```bash
yumdownloader --resolve 软件包名称
```
再使用rpm命令安装
```bash
rpm -ivh 软件包名称
```