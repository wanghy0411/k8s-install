单服务器安装

本文介绍如何在单一的服务器上安装kubernetes1.12.9

由于google被墙，国内网络不能正常访问；因此yaml文件中使用的很多镜像地址调整到本人在阿里云上申请的镜像服务，可以直接使用。

环境示例：

```
OS：CentOS 7
IP: 10.70.13.220
Hostname: k8s-single
```

一、准备工作：
1.1 关闭swap,最好是安装CentOS的时候就不安装swap分区

1.2 写hosts文件,解析本机的机器名
```
# vi /etc/hosts

10.70.13.220 k8s-single
```

1.3 关闭selinux防火墙
```
# setenforce 0
# systemctl stop firewalld && systemctl disable firewalld
```
```
# vi /etc/selinux/config
修改SELINUX=disabled
注意：不是修改SELINUXTYPE
```

1.4 安装并启动docker
```
# yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm

# systemctl enable docker && systemctl restart docker
```
1.5  开启forward
```
# iptables -P FORWARD ACCEPT
```

1.6 配置系统路由参数
```
# echo "
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
" >> /etc/sysctl.conf

# sysctl -p
```

1.7 加载ipvs相关内核模块
```
# modprobe ip_vs
# modprobe ip_vs_rr
# modprobe ip_vs_wrr
# modprobe ip_vs_sh
# modprobe nf_conntrack_ipv4
# lsmod | grep ip_vs
```

二、安装组件

2.1 设置使用阿里镜像安装建立repo文件：
```
# vi /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

```
或者直接下载文件到对应目录也可。

2.2  安装 kubeadm, kubelet 和 kubectl
```
# yum install -y kubelet-1.12.9
# yum install -y kubeadm-1.12.9
# yum install -y kubectl-1.12.9
# yum install -y ipvsadm
```

2.3 配置kubelet使用国内pause镜像，修改cgroup配置
```
# vi /etc/sysconfig/kubelet

添加配置如下：
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

2.4 启动kubelet
```
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet
```

三、kubernetes安装

3.1 拉取镜像
```

# docker pull mirrorgooglecontainers/kube-apiserver:v1.12.9
# docker pull mirrorgooglecontainers/kube-proxy:v1.12.9
# docker pull mirrorgooglecontainers/kube-controller-manager:v1.12.9
# docker pull mirrorgooglecontainers/kube-scheduler:v1.12.9
# docker pull mirrorgooglecontainers/etcd:3.2.24
# docker pull kuberneter/coredns:1.2.2
# docker pull mirrorgooglecontainers/pause:3.1
给镜像打标签
# docker tag mirrorgooglecontainers/kube-apiserver:v1.12.9 k8s.gcr.io/kube-apiserver:v1.12.9
# docker tag mirrorgooglecontainers/kube-controller-manager:v1.12.9 k8s.gcr.io/kube-controller-manager:v1.12.9
# docker tag mirrorgooglecontainers/kube-scheduler:v1.12.9 k8s.gcr.io/kube-scheduler:v1.12.9
# docker tag mirrorgooglecontainers/kube-proxy:v1.12.9 k8s.gcr.io/kube-proxy:v1.12.9
# docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
# docker tag mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
# docker tag kuberneter/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2
```

3.2 安装
```
# kubeadm init --kubernetes-version=v1.12.9 --pod-network-cidr=10.244.0.0/16
```


3.3 配置环境变量
```
# vi ~/.bash_profile
加入一行：export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
```

3.4 flannel安装

使用kube-flannel.yml文件
```
# wget  https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
# kubectl apply -f kube-flannel.yml
```

3.5 安装验证
```
# kubectl get pod -n kube-system
# kubectl get node
```

3.6 设置master可以分配pod

master节点上一般不会再分配pod，以保障master的高效运行，
如果仅有1台服务器，没有node的情况下，需要执行如下语句使master可以分配pod
```
# kubectl taint nodes --all node-role.kubernetes.io/master-
```

##小技巧
可以使用如下语句，进入容器的sh界面
```
# kubectl exec -it pod_name -n kube-system /bin/sh
```
其中pod_name为具体的pod名字，查pod名字的方法
```
# kubectl get pod
```
