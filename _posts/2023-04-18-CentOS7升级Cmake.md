---
title: "CentOS7升级Cmake"
categories:
  - CentOS
tags:
  - CentOS
---

`CentOS`自带的`Cmake`版本太低，不符合一些使用场景，需要升级。

<!--more-->
1. 卸载Cmake
    ```bash
   yum remove cmake 
   ```
2. 从[官网](https://cmake.org/download/)下载相应的源码
    ```bash
    wget https://github.com/Kitware/CMake/releases/download/v3.26.3/cmake-3.26.3.tar.gz
   ```
3. 解压
    ```bash
    tar -zxvf cmake-3.26.3.tar.gz
    ```
4. 安装
    ```bash
    cd cmake-3.26.3
    ./bootstrap --prefix=/usr/local
    make
    make install
    ```
5. 在shell中添加路径
    ```bash
    vi ~/.bash_profile
    PATH=/usr/local/bin:$PATH:$HOME/bin
    ```
6. 重新打开ssh，检查版本，判断是否安装成功
    ```bash
    cmake --version
    ```





