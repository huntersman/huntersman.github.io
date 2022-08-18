---
title: "CentOS升级Linux内核"
categories:
  - CentOS
tags:
  - CentOS
---

有时候我们需要升级CentOS的内核，操作步骤如下  

1. 查看系统当前内核
```bash
uname -r 
```
2. 使用内核使用的yum仓库
```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
3. 查看哪些内核可供安装
```bash
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```
4. 安装
```bash
yum -y --enablerepo=elrepo-kernel install kernel-ml
```
5. 设置Grub启动项
```bash
vim /etc/default/grub

GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved  #这里把saved改为0即可
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="rhgb quiet"
```
6. 重启系统
```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot 
```

转自[博客](https://blog.51cto.com/zlyang/4903964)