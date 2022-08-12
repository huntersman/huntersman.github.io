---
title: "Failed to download metadata for repo ‘AppStream’ [CentOS]"
categories:
  - CentOS
tags:
  - CentOS
---

本文转载自[techglimpse](https://techglimpse.com/failed-metadata-repo-appstream-centos-8)

# 问题介绍
今天在使用CentOS 8时，执行`yum update`命令之后，得到如下报错信息
```bash
Failed to download metadata for repo ‘AppStream’
```
# 问题原因
在2021年12月31日的时候CentOS 8到达了End Of Life。意味着CentOS 8将不会收到官方的CentOS project。
# 问题解决
解决方法需要把镜像改为`vault.centos.org`，或者把CentOS升级为CentOS Stream。
```bash
cd /etc/yum.repos.d/
```
```bash
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
```
```bash
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```
```bash
yum update -y
```

