---
title: "fincore安装及使用"
categories:
  - Linux
tags:
  - Linux
---

fincore是一款Linux命令行工具，可以用来查看指定目录下的内核页缓存。

## 安装
由于CentOS上未找到相关RPM包，故通过源码编译的方式安装fincore
```bash
# 国内网络
git clone https://gitee.com/yejinlei-mirror/linux-ftools
# 国外网络
git clone https://github.com/david415/linux-ftools
cd linux-ftools/
# automake --version查看版本号并修改configure文件的am__api_version='1.10'一行的版本号
./configure
make
sudo make install
```
## 使用
```bash
# 查看帮助
linux-fincore --help
# 显示的帮助
fincore version 1.3.0
fincore [options] files...

  -s --summarize          When comparing multiple files, print a summary report
  -p --pages              Print pages that are cached
  -o --only-cached        Only print stats for files that are actually in cache.
  -g --graph              Print a visual graph of each file's cached page distribution.
  -S --min-size           Require that each files size be larger than N bytes.
  -C --min-cached-size    Require that each files cached size be larger than N bytes.
  -P --min-perc-cached    Require percentage of a file that must be cached.
  -h --help               Print this message.
  -L --vertical           Print the output of this script vertically.
# 查看当前目录下的内核页缓存
linux-fincore --pages=false --summarize --only-cached * 
```

参考资料
- [https://unix.stackexchange.com/questions/87908/how-do-you-empty-the-buffers-and-cache-on-a-linux-system](https://unix.stackexchange.com/questions/87908/how-do-you-empty-the-buffers-and-cache-on-a-linux-system)