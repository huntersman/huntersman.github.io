---
title: "使用GitHub Actions自动化build docker image并push到docker hub"
categories:
  - GitHub
tags:
  - GitHub
  - CI/CD
---

使用GitHub Actions可以帮助我们进行CI/CD。

以自动化build docker image为例。

首先在.github/workflows目录下面创建.yml文件，文件名随意
```yaml
name: Build docker image and push   # workflow名称，可以在Github项目主页的【Actions】中看到所有的workflow

on: # 配置触发workflow的事件
  push:
    branches: # master分支有push时触发此workflow
      - 'master'
    tags: # tag更新时触发此workflow
      - '*'

jobs: # workflow中的job
  push_to_registry: # job的名字
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest   # job运行的基础环境

    steps: # 一个job由一个或多个step组成
      - name: Check Out Repo
        uses: actions/checkout@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v2.1.0  # 三方的action操作， 执行docker login
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}  # 配置dockerhub的认证，在Github项目主页 【Settings】 -> 【Secrets】 添加对应变量
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4  # 抽取项目信息，主要是镜像的tag
        with:
          images: hunterman1/s3fs-fuse

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v3
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```
github上提供了许多现成的action，可以直接拿来用，并且也给出了使用示例。

我们需要做的就是在项目里的设置里配置一下DockerHub的用户名与Token，之后在项目的根目录下创建Dockerfile就行了。
![alt]({{ site.url }}{{ site.baseurl }}/assets/images/GitHubActions.png)