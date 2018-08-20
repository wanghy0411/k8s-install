单master+多node安装

本文介绍如何安装kubernetes1.11.2的简单集群

由于google被墙，国内网络不能正常访问；因此yaml文件中使用的很多镜像地址调整到本人在阿里云上申请的镜像服务，可以直接使用。

环境示例：

```
OS：CentOS 7

master: 
IP: 10.70.13.229 k8s-master01

node:
IP: 10.70.13.224 k8s-node01
IP: 10.70.13.225 k8s-node02
```

一、准备工作（所有服务器）：
1.1 关闭swap,最好是安装CentOS的时候就不安装swap分区

1.2 写hosts文件，绑定所有服务器
```
# vi /etc/hosts

10.70.13.229 k8s-master01
10.70.13.224 k8s-node01
10.70.13.225 k8s-node02
```

1.3 服务器互信(master上即可)
```
# ssh-keygen -f /root/.ssh/id_rsa -N ''
# for host in k8s-master01 k8s-node01 k8s-node02; do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; done
```

1.4 关闭selinux防火墙
```
# setenforce 0
# systemctl stop firewalld && systemctl disable firewalld
```
```
# vi /etc/selinux/config
修改SELINUX=disabled
注意：不是修改SELINUXTYPE
```

1.5 安装并启动docker
```
# yum install -y https://mirrors.aliyun.com/docker-engine/yum/repo/main/centos/7/Packages/docker-engine-17.03.1.ce-1.el7.centos.x86_64.rpm

# systemctl enable docker && systemctl restart docker
```
1.6  开启forward
```
# iptables -P FORWARD ACCEPT
```

1.7 配置系统路由参数
```
# echo "
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
" >> /etc/sysctl.conf

# sysctl -p
```

1.8 加载ipvs相关内核模块
```
# modprobe ip_vs
# modprobe ip_vs_rr
# modprobe ip_vs_wrr
# modprobe ip_vs_sh
# modprobe nf_conntrack_ipv4
# lsmod | grep ip_vs
```

二、安装组件（所有服务器）

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
# yum install -y kubelet kubeadm kubectl ipvsadm
```

2.3 配置kubelet使用国内pause镜像，修改cgroup配置
```
# vi /etc/sysconfig/kubelet

添加配置如下：
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs --pod-infra-container-image=registry.cn-beijing.aliyuncs.com/k8s-install/pause-amd64:3.1"
```

2.4启动kubelet
```
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet
```

三、master安装

3.1 设置配置文件
```
# vi /root/k8s-install/kubernetes-init.config

apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.2
imageRepository: registry.cn-beijing.aliyuncs.com/k8s-install

apiServerCertSANs:
- "k8s-master01"
- "10.70.13.229"

api:
  advertiseAddress: 10.70.13.229
  controlPlaneEndpoint: 10.70.13.229:6443

etcd:
  local:
    extraArgs:
      listen-client-urls: "https://127.0.0.1:2379,https://10.70.13.229:2379"
      advertise-client-urls: "https://10.70.13.229:2379"
      listen-peer-urls: "https://10.70.13.229:2380"
      initial-advertise-peer-urls: "https://10.70.13.229:2380"
      initial-cluster: "k8s-master01=https://10.70.13.229:2380"
    serverCertSANs:
      - k8s-master01
      - 10.70.13.229
    peerCertSANs:
      - k8s-master01
      - 10.70.13.229

controllerManagerExtraArgs:
  node-monitor-grace-period: 10s
  pod-eviction-timeout: 10s

kubeProxy:
  config:
    mode: ipvs

networking:
  podSubnet: 10.244.0.0/16
```
注意：实际安装时需要调整ip和服务器名称为对应服务器的IP和hostname

3.2 安装
```
# kubeadm config images pull --config kubernetes-init.config
# kubeadm init --config kubernetes-init.config
```

记录类似如下的结果串，用于后续节点添加
```
kubeadm join 10.70.13.229:6443 --token 24cser.h747zmiipvikbgq8 --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
```
注意：如果失败可以使用以下命令复原
```
# kubeadm reset
```

3.3 配置环境变量
```
# export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
```

3.4 flannel安装

使用kube-flannel.yml文件
```
# kubectl apply -f kube-flannel.yml
```

3.5 安装验证
```
# kubectl get pod -n kube-system
# kubectl get node
```

四、node安装
4.1 执行之前记录下的串，例如：
```
# kubeadm join 10.70.13.229:6443 --token 24cser.h747zmiipvikbgq8 --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
```
4.2 验证
在master上执行
```
# kubectl get node
```



----小技巧
遗忘join串的话,可以使用如下方法获取
```
# kubeadm token create --print-join-command
```
