单master+多node安装

本文介绍如何安装kubernetes1.11.3的高可用集群

由于google被墙，国内网络不能正常访问；因此yaml文件中使用的很多镜像地址调整到本人在阿里云上申请的镜像服务，可以直接使用。

环境示例：

```
OS：CentOS 7

master: 
IP: 10.70.13.221 k8s-master01
IP: 10.70.13.222 k8s-master02
IP: 10.70.13.223 k8s-master03

node:
IP: 10.70.13.224 k8s-node01
IP: 10.70.13.225 k8s-node02

虚拟ip:
IP: 10.70.13.220
此IP由三台master虚拟出来，用于高可用。集群外部实际访问的是这个地址
```

一、准备工作（所有服务器）：
1.1 关闭swap,最好是安装CentOS的时候就不安装swap分区

1.2 写hosts文件，绑定所有服务器
```
# vi /etc/hosts

10.70.13.221 k8s-master01
10.70.13.222 k8s-master02
10.70.13.223 k8s-master03
10.70.13.224 k8s-node01
10.70.13.225 k8s-node02
```

1.3 服务器互信(master01上即可)
```
# ssh-keygen -f /root/.ssh/id_rsa -N ''
# for host in k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02; do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; done
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

三、配置haproxy和keepalived（三台master上安装）
3.1 下载haproxy镜像
```
# docker pull haproxy:1.7.8-alpine
```

3.2 设置配置文件
```
# mkdir /etc/haproxy
```

拷贝demo中的haproxy.cfg到/etc/haproxy目录，并调整对应ip信息

3.3 启动haproxy
```
# docker run -d --name my-haproxy \
-v /etc/haproxy:/usr/local/etc/haproxy:ro \
-p 8443:8443 \
-p 1080:1080 \
--restart always \
haproxy:1.7.8-alpine

查看日志
# docker logs my-haproxy
```

3.4 拉取keepalived镜像
```
# docker pull osixia/keepalived:1.4.4
```

3.5 启动keepalived

_注意：enp0s3为网卡标识，请根据具体的服务器网卡进行调整_
```
# docker run --net=host --cap-add=NET_ADMIN \
-e KEEPALIVED_INTERFACE=enp0s3 \
-e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['10.70.13.220']" \
-e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['10.70.13.221','10.70.13.222','10.70.13.223']" \
-e KEEPALIVED_PASSWORD=hello \
--name k8s-keepalived \
--restart always \
-d osixia/keepalived:1.4.4

查看日志,会看到两个成为backup 一个成为master
# docker logs k8s-keepalived
```

提示：失败后清理，使用如下语句
```
# docker rm -f k8s-keepalived
# ip a del 10.70.13.220/32 dev enp0s3
```

四、master安装

4.1 设置配置文件
```
# vi /root/k8s-install/kubernetes-init.config
```
内容参考demo中文件
注意：实际安装时需要调整ip和服务器名称为对应服务器的IP和hostname

4.2 安装
```
# kubeadm config images pull --config kubernetes-init.config
# kubeadm init --config kubernetes-init.config
```

注意：
第一条语句是拉取镜像语句，建议在所有服务器拉取镜像完毕后再进行后续操作
要先在keepalived为master的机器上安装，不用等待出现的安装完毕提示，直接进行4.3和4.4
原因是etcd至少需要2台服务器启动才能完成集群搭建

实在不行就把加密文件留着，所有服务器同时初始化

4.3 拷贝加密文件到其它master
打包ca相关文件上传至其他master节点
```
# cd /etc/kubernetes && tar cvzf k8s-key.tgz admin.conf pki/ca.* pki/sa.* pki/front-proxy-ca.* pki/etcd/ca.*
# scp k8s-key.tgz k8s-master02:~/
# scp k8s-key.tgz k8s-master03:~/
# ssh k8s-master02 'tar xf k8s-key.tgz -C /etc/kubernetes/'
# ssh k8s-master03 'tar xf k8s-key.tgz -C /etc/kubernetes/'
```
4.4 在其它服务器上执行安装
```
# kubeadm init --config kubernetes-init.config
```

4.5 记录类似如下的结果串，用于后续节点添加
```
kubeadm join 10.70.13.220:8443 --token 24cser.h747zmiipvikbgq8 --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
kubeadm join 10.70.13.220:8443 --token llla4w.rmas56lxc8upqlwv --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
kubeadm join 10.70.13.220:8443 --token 59mk2s.6xlwmagjrujewq6i --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
```
注意：如果失败可以使用以下命令复原
```
# kubeadm reset
```

4.6 配置环境变量
```
# export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
```

4.7 flannel安装

使用kube-flannel.yml文件
```
# kubectl apply -f kube-flannel.yml
```

4.8 安装验证
```
# kubectl get pod -n kube-system
# kubectl get node
```

五、node安装
5.1 执行之前记录下的串，例如：
```
# kubeadm join 10.70.13.229:6443 --token 24cser.h747zmiipvikbgq8 --discovery-token-ca-cert-hash sha256:1b0b32be09c2ab5635da85eb9fcc23a67497a44b03a3072ae694ca07a884e7d6
```
5.2 验证
在master上执行
```
# kubectl get node
```

######小技巧#######
遗忘join串的话,可以使用如下方法获取
```
# kubeadm token create --print-join-command
```
