---
title: "CentOS升级Linux内核"
last_modified_at: 2022-09-22T16:20:02-05:00
categories:
  - CentOS
tags:
  - CentOS
---

<!--more-->

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
   内核升级完毕后，目前内核还是默认的版本，如果此时直接执行reboot命令，重启后使用的内核版本还是默认的3.10，不会使用新的4.12.4，首先，我们可以通过命令`awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg`查看默认启动顺序。
```bash
vim /etc/default/grub
```
```text
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
7. 卸载旧内核（可选）
```bash
rpm -qa | grep kernel
yum remove 内核名
```
转自[博客](https://blog.51cto.com/zlyang/4903964)