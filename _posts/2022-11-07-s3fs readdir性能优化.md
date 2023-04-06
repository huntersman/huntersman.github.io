---
title: "S3FS的readdir性能优化记录"
categories:
  - S3FS
tags:
  - S3FS
---

<!--more-->

最近公司想把对象存储挂载到操作系统上，当做本地目录来使用，技术上选择了S3FS。但是S3FS本身的性能存在一定问题，故对其做了性能优化。性能优化方面主要考虑的是ls性能，顺序读写性能为120MB/s，勉强能用，但是ls性能在单目录为10万对象数时，时间为30秒。
# ls原理
既然要优化ls性能，就不得不先了解在Linux中，输入ls命令，到底发生了什么。这里我在网上找了一张图，非常容易理解ls的过程。

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/FUSE_structure.png)

首先当我们在bash里面执行ls命令时，会进行系统调用，命令会进入内核由VFS处理。VFS需要判断文件系统类型，而这里由于S3FS是FUSE文件系统，因此相关调用发送给了FUSE。FUSE内部维护着一个请求队列，依据先进先出的原则将请求发送给了用户态的libfuse，libfuse找到相关的具体实现方法，比如ls就对应了readdir。

以上就是一个简化后的ls流程，想要了解具体流程就需要看代码了，这里就不深入展开了。
# 优化
知道了ls过程后，开始逐步分析如何优化S3FS性能。我们使用的是1.91版本S3FS。首先可以看到这个方法里主要是通过list_bucket获取`head`、readdir_multi_head获取`buf`的，而这两个方法底层都是通过curl给S3服务端发送请求，由于经过了网络，因此性能受限于网络。优化的思路点为readdir不经过网络，本地维护一个目录缓存，当readdir时，直接调用filler方法生成目录。当然这么做的缺点是无法再使用S3协议往桶里写对象了，对象必须经过S3FS往桶里写。
```c++
static int s3fs_readdir(const char* _path, void* buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info* fi)
{
    WTF8_ENCODE(path)
    S3ObjList head;
    int result;

    S3FS_PRN_INFO("[path=%s]", path);

    if(0 != (result = check_object_access(path, R_OK, NULL))){
        return result;
    }

    // get a list of all the objects
    if((result = list_bucket(path, head, "/")) != 0){
        S3FS_PRN_ERR("list_bucket returns error(%d).", result);
        return result;
    }

    // force to add "." and ".." name.
    filler(buf, ".", 0, 0);
    filler(buf, "..", 0, 0);
    if(head.IsEmpty()){
        return 0;
    }

    // Send multi head request for stats caching.
    std::string strpath = path;
    if(strcmp(path, "/") != 0){
        strpath += "/";
    }
    if(0 != (result = readdir_multi_head(strpath.c_str(), head, buf, filler))){
        S3FS_PRN_ERR("readdir_multi_head returns error(%d).", result);
    }
    S3FS_MALLOCTRIM(0);

    return result;
}
```

因此我们维护一个全局缓存， 且缓存需要在目录下的对象发生变化时变化，即写、删、重命名、移动等。