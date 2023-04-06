---
title: "Docker logs持久化"
categories:
  - Docker
tags:
  - Docker
---

<!--more-->

可以通过`docker logs`命令查看指定容器的日志，但是当容器被删除后，日志也随之被删除了。

如果想要持久化容器日志，可以`docker logs 容器id >& /var/log/你想要的日志名称`