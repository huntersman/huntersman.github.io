---
title: "使用AWS CPP SDK简单教程"
categories:
  - AWS
tags:
  - AWS
---

<!--more-->
1. 下载AWS CPP SDK [https://github.com/aws/aws-sdk-cpp](https://github.com/aws/aws-sdk-cpp)。如`wget https://github.com/aws/aws-sdk-cpp/archive/1.0.164.tar.gz`
2. 安装依赖

    ```bash
    yum -y erase cmake

    yum -y install cmake3 gcc-c++ libstdc++-devel libcurl-devel zlib-devel

    cd /usr/bin && ln -s cmake3 cmake
    ```

3. 准备源码

    ```bash
    tar -zxvf 源码包名 -C /root/

    mkdir -p /root/build && cd /root/build

    cmake -DCMAKE_BUILD_TYPE=Release /root/解压后的源码文件夹名
    ```

4. 编译源码

    ```bash
    make -j `nproc` -C aws-cpp-sdk-core
    make -j `nproc` -C aws-cpp-sdk-s3
    ```

5. 安装头文件以及库

    ```bash
    mkdir -p /root/install
    make install DESTDIR=/root/install -C aws-cpp-sdk-core
    make install DESTDIR=/root/install -C aws-cpp-sdk-s3
    ```

6. 使用示例（往桶内上传文件）

    ```c++
    #include <aws/core/Aws.h>
    #include <aws/core/auth/AWSCredentialsProvider.h>
    #include <aws/s3/S3Client.h>
    #include <aws/s3/model/PutObjectRequest.h>
    #include <iostream>
    #include <fstream>
    #include <sys/stat.h>

    using namespace std;

    inline bool file_exists(const string& name)
    {
        struct stat buffer;
        return (stat(name.c_str(), &buffer) == 0);
    }

    bool put_s3_object(const Aws::String& s3_bucket_name, 
        const Aws::String& s3_object_name, 
        const string& file_name, 
        const Aws::String& region = "")
    {
        // 判断文件是否存在
        if (!file_exists(file_name)) {
            cout << "ERROR: 找不到这个文件，这个文件不存在" 
                << endl;
            return false;
        }

        // 如果指定了地区
        Aws::Client::ClientConfiguration cfg;
        if (!region.empty())
            cfg.region = region;

        //s3连接
        cfg.endpointOverride = "192.168.11.201:7480";
        cfg.scheme = Aws::Http::Scheme::HTTP;
        cfg.verifySSL = false;
        // ak,sk
        Aws::Auth::AWSCredentials cred("84MLS4O2U9PVF1F6FFDD", "f5LfdftuEOE9TVtlfHazRAkDXbYSuq85mgnRdfUp"); 
        Aws::S3::S3Client s3_client(cred, cfg, false, false);
        Aws::S3::Model::PutObjectRequest object_request;

        object_request.SetBucket(s3_bucket_name);
        object_request.SetKey(s3_object_name);
        const shared_ptr<Aws::IOStream> input_data = 
            Aws::MakeShared<Aws::FStream>("SampleAllocationTag", 
                                        file_name.c_str(), 
                                        ios_base::in | ios_base::binary);
        object_request.SetBody(input_data);

        // 上传文件
        auto put_object_outcome = s3_client.PutObject(object_request);
        if (!put_object_outcome.IsSuccess()) {
            auto error = put_object_outcome.GetError();
            cout << "ERROR: " << error.GetExceptionName() << ": " 
                << error.GetMessage() << endl;
            return false;
        }
        return true;
    }

    int main(int argc, char** argv)
    {

        if (argc < 2)
        {
            cout << "put_object - 上传一个文件" << endl
                << "\nUsage:" << endl
                << "  put_object <bucket> <objname> <objname>" << endl
                << "\nWhere:" << endl
                << "  bucket - 桶名称" << endl
                << "  objname - 文件名" << endl
                << "\nExample:" << endl
                << "  put_object testbucket testfile testfile\n" << endl << endl;
            exit(1);
        }

        Aws::SDKOptions options;
        Aws::InitAPI(options);
        {
            
            const Aws::String bucket_name = argv[1];
            const string file_name = argv[2];
            const Aws::String object_name = argv[3];
            const Aws::String region = "";     
            
            if (put_s3_object(bucket_name, object_name, file_name, region)) {
                cout << "上传文件 " << file_name 
                    << " 到存储桶 " << bucket_name 
                    << " 存为 " << object_name << endl;
            }
        }
        Aws::ShutdownAPI(options);
    }
    ```

7. 编译代码

    ```bash
    g++ -std=c++11 -I/root/install/usr/local/include -L/root/install/usr/local/lib64 -laws-cpp-sdk-core -laws-cpp-sdk-s3 PutObject.cpp -o PutObject
    ```

    ```bash
    export LD_LIBRARY_PATH=/root/install/usr/local/lib64
    ```

8. 运行

    ```bash
    ./PutObject 你的桶名 你的文件名 桶内文件名
    ```

参考资料：
- [https://blog.csdn.net/weixin_41438870/article/details/106859300](https://blog.csdn.net/weixin_41438870/article/details/106859300)