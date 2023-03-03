---
title: "Linux命令之iptables"
categories:
  - Linux
tags:
  - Linux
---

> iptables是运行在用户空间的应用软件，通过控制Linux内核netfilter模块，来管理网络数据包的处理和转发。

清空所有规则
```bash
iptables -F
```
查看规则
```bash
iptables -L
# 或
iptables -S
# 以序号标记显示
iptables -L -n --line-numbers
```
删除指定序号规则
```bash
# 删除INPUT里序号为6的
iptables -D INPUT 6
```
在尾部新增规则
```bash
# 允许22端口接收TCP
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```
在头部新增规则
```bash
# 允许2049端口接收UDP
iptables -I INPUT -p udp --dport 2049 -j ACCEPT
```
清理重复规则，[来源](https://blog.csdn.net/tobyliu415/article/details/124781249)
```bash
iptables-save | awk '{if($1=="COMMIT"){delete x}}$1=="-A"?!x[$0]++:1' | iptables-restore
```
