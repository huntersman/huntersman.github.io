---
title: "K8s入门笔记"
categories:
  - Kubernetes
tags:
  - Kubernetes
---

# K8s介绍

以下是[官网](https://kubernetes.io/)对K8s的介绍

>Kubernetes 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态，其服务、支持和工具的使用范围相当广泛。
>
>Kubernetes 这个名字源于希腊语，意为“舵手”或“飞行员”。k8s 这个缩写是因为 k 和 s 之间有八个字符的关系。 Google 在 2014 年开源了 Kubernetes 项目。 Kubernetes 建立在Google 大规模运行生产工作负载十几年经验的基础上， 结合了社区中最优秀的想法和实践。

我个人的理解是K8s是一个容器管理平台。当我们需要容器化部署我们的项目时，可以使用docker，但是当服务器集群复杂的时候，如上百台服务器组成集群，版本更新时需要操作上百台服务器的docker，当线上遇到问题，需要进行回滚时，也会非常麻烦。这种情况下docker就有点力不从心了，需要K8s对容器进行管理。

# 为什么需要K8s

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/container_evolution.svg)

从上图可以看到项目的部署可以分为三个时代，分别是传统部署时代、虚拟化部署时代、容器部署时代。
## 传统部署时代
最早期，在服务器上直接部署。这种方式简单快速，但缺点是无法限制单个应用程序所能使用的资源，也无法进行资源分配。可能会出现一个程序占用了大量服务器资源的情况。
## 虚拟化部署时代
虚拟化技术就是在服务器上运行多台虚拟机，使得应用程序之间隔离，但由于每个虚拟机是一个完整的计算机，拥有自己的操作系统，这种方式性能损耗较大。
## 容器部署时代
容器共享主机的操作系统，但每个容器有着自己的文件系统、CPU、内存等等，可以认为是轻量级的虚拟机。

> 容器是打包和运行应用程序的好方式。在生产环境中， 你需要管理运行着应用程序的容器，并确保服务不会下线。 例如，如果一个容器发生故障，则你需要启动另一个容器。 如果此行为交由给系统处理，是不是会更容易一些？
> 
> 这就是 Kubernetes 要来做的事情！ Kubernetes 为你提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移、部署模式等。

K8s提供:
- 服务发现和负载均衡
- 存储编排
- 自动部署和回滚
- 自动完成装箱计算
- 自我修复
- 密钥与配置管理

# K8s架构

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/components-of-kubernetes.svg)

# 使用Kubeadm搭建K8s集群

| ip           | 说明       | 环境               |
| ------------ | ---------- | ------------------ |
| 172.0.48.129 | K8S master | CentOS 7           |
| 172.0.48.131 | K8S node1  | CentOS 7           |
| 172.0.48.133 | K8S node2  | CentOS 7           |

## K8S集群搭建

目前有[一键搭建K8S集群](https://github.com/lework/kainstall)的方法，但本次搭建采用kubeadm手动搭建。

### 安装K8S运行环境（所有节点执行）

```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 检查2379和2380的端口占用情况，确保K8S的etcd能成功启用，如果被占用了，请kill掉
lsof -i:2379
lsof -i:2380

#设置epel镜像
yum install epel-release
#替换成清华镜像源
sed -e 's!^metalink=!#metalink=!g' \
    -e 's!^#baseurl=!baseurl=!g' \
    -e 's!//download\.fedoraproject\.org/pub!//mirrors.tuna.tsinghua.edu.cn!g' \
    -e 's!//download\.example/pub!//mirrors.tuna.tsinghua.edu.cn!g' \
    -e 's!http://mirrors!https://mirrors!g' \
    -i /etc/yum.repos.d/epel*.repo
```

将Docker的Cgroup Driver修改为systemd

```bash
# 查看Cgroup Driver
docker info|grep "Cgroup Driver"
# kubernetes 官方推荐 docker 等使用 systemd 作为 cgroupdriver，否则 kubelet 启动不了
cat <<EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ud6340vz.mirror.aliyuncs.com"]
}
EOF
mv daemon.json /etc/docker/

# 重启生效
systemctl daemon-reload
systemctl restart docker
```

依次修改各个节点的hostname

```bash
hostnamectl set-hostname master
hostnamectl set-hostname node1
hostnamectl set-hostname node2
```

在hosts里面做相应添加

```bash
vim /etc/hosts
172.0.48.129 master
172.0.48.131 node1
172.0.48.133 node2
```

```bash
# 配置K8S的阿里云yum源（如果发现阿里云源下载慢，可以替换成清华源）
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 添加 Docker 安装源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

下载kubelet、kubectl、kubeadm

```bash
yum install -y kubelet-1.22.4 kubectl-1.22.4 kubeadm-1.22.4 --disableexcludes=kubernetes
# 如果上述命令下载失败，请用下面的命令下载
yum install -y --nogpgcheck kubelet-1.22.4 kubectl-1.22.4 kubeadm-1.22.4 --disableexcludes=kubernetes
```

下载完毕之后启动并设置 kubelet 开机启动

```bash
systemctl start kubelet
systemctl enable --now kubelet
```

### 初始化集群（仅master节点执行）

```bash
kubeadm init \
    --apiserver-advertise-address 172.0.48.129 \
    --control-plane-endpoint 172.0.48.129 \
    --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
    --pod-network-cidr 10.244.0.0/16
```

安装成功后，会提示

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kuber4s.api:6443 --token zd42s0.s2tabpwrvakxm2yu \
    --discovery-token-ca-cert-hash sha256:e8951771ecb37cd1bf4813c4dbff064cf30296957be1773311a229fcc3cd05e1 \
    --control-plane --certificate-key 1ff38a74d6ae1993ea392b4312ecb1692b52a5ac7fd0a624fe0cea01e39acead

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kuber4s.api:6443 --token zd42s0.s2tabpwrvakxm2yu \
    --discovery-token-ca-cert-hash sha256:e8951771ecb37cd1bf4813c4dbff064cf30296957be1773311a229fcc3cd05e1
```

执行

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

安装网络组件flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

检查coreDNS状态，当coreDNS为running状态时，说明集群运行正常，可以让work节点加入master。

```bash
kubectl get pods --all-namespaces
```

### worker节点加入master组成集群

在worker节点执行刚刚master节点打印出来的加入语句（如果忘了可以在master节点执行`kubeadm token create --print-join-command`重新获取）

```bash
kubeadm join kuber4s.api:6443 --token zd42s0.s2tabpwrvakxm2yu \
    --discovery-token-ca-cert-hash sha256:e8951771ecb37cd1bf4813c4dbff064cf30296957be1773311a229fcc3cd05e1
```

### 检查集群状态

在master节点执行，全部节点处于ready状态

```bash
kubectl get nodes
```

在master节点执行，所有pods处于running状态

```bash
kubectl get pods --all-namespaces
```

以上两项都检查通过说明K8S集群正常运行。