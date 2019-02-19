一、1，写hosts文件,解析本机的机器名
# vi /etc/hosts
10.70.13.220 k8s-single
hostname k8s-single
2，关闭selinux防火墙
# setenforce 0
# systemctl stop firewalld && systemctl disable firewalld
# vi /etc/selinux/config
修改SELINUX=disabled
注意：不是修改SELINUXTYPE
4，安装并启动docker
yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm
# systemctl enable docker && systemctl restart docker
5，开启forward
# iptables -P FORWARD ACCEPT
6， 配置系统路由参数
# echo "
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
" >> /etc/sysctl.conf

# sysctl -p
7，加载ipvs相关内核模块
# modprobe ip_vs
# modprobe ip_vs_rr
# modprobe ip_vs_wrr
# modprobe ip_vs_sh
# modprobe nf_conntrack_ipv4
# lsmod | grep ip_vs
8，设置使用阿里镜像安装建立repo文件：
# vi /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpghttps://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
9，指定版本安装 kubeadm, kubelet 和 kubectl，并且安装ipvsadm
yum install -y kubelet-1.12.4 kubeadm-1.12.4 kubectl-1.12.4 ipvsadm
10，配置kubelet使用国内pause镜像，修改cgroup配置
# vi /etc/sysconfig/kubelet

添加配置如下：
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
11，启动kubelet
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet

12,kubeadm init --kubernetes-version=v1.12.4 --pod-network-cidr=10.244.0.0/16

13，拉取镜像
docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.12.4
docker pull mirrorgooglecontainers/kube-proxy:v1.12.4
docker pull mirrorgooglecontainers/kube-controller-manager:v1.12.4
docker pull mirrorgooglecontainers/kube-scheduler:v1.12.4
docker pull mirrorgooglecontainers/etcd:3.2.24
docker pull kuberneter/coredns:1.2.2
docker pull mirrorgooglecontainers/pause:3.1
给镜像打标签
docker tag mirrorgooglecontainers/kube-apiserver-amd64:v1.12.4 k8s.gcr.io/kube-apiserver:v1.12.4
docker tag mirrorgooglecontainers/kube-controller-manager:v1.12.4 k8s.gcr.io/kube-controller-manager:v1.12.4
docker tag mirrorgooglecontainers/kube-scheduler:v1.12.4 k8s.gcr.io/kube-scheduler:v1.12.4
docker tag mirrorgooglecontainers/kube-proxy:v1.12.4 k8s.gcr.io/kube-proxy:v1.12.4
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24
docker tag kuberneter/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2

docker images    检查镜像

14，配置环境变量

# export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
15，flannel安装

使用kube-flannel.yml文件

wget  https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml ----12.4版本

kubectl apply -f kube-flannel.yml

16，安装验证

# kubectl get pod -n kube-system
# kubectl get node

17，设置master可以分配pod

master节点上一般不会再分配pod，以保障master的高效运行， 如果仅有1台服务器，没有node的情况下，需要执行如下语句使master可以分配pod

# kubectl taint nodes --all node-role.kubernetes.io/master-
##小技巧 可以使用如下语句，进入容器的sh界面

# kubectl exec -it pod_name -n kube-system /bin/sh
其中pod_name为具体的pod名字，查pod名字的方法

# kubectl get pod






















