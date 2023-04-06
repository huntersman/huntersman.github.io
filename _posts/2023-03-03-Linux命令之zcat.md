---
title: "Linux命令之zcat"
categories:
  - Linux
tags:
  - Linux
---

<!--more-->

有时候日志文件是gz的，直接用cat命令查看会报错，这种时候就可以用zcat命令查看
```bash
zcat /var/log/111.log.1.gz
```