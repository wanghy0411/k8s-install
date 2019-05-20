本文介绍如何如何部署一个kubernetes1.12.8的高可用集群
说明: 

2LVS+3MASTER+WORKER
两台LVS服务器利用keepalived形成主备
LVS和K8S-MASTER同时需出一个虚拟IP，通过LVS的DR模式进行负载均衡

参考文档:

https://blog.csdn.net/nimasike/article/details/51918761

环境示例：

```
OS：CentOS 7

lvs:
IP: 192.168.234.121 lvs-1 (master)
IP: 192.168.234.122 lvs-2 (slave)

master: 
IP: 192.168.234.111 k8s-master01
IP: 192.168.234.112 k8s-master02
IP: 192.168.234.113 k8s-master03

worker:
IP: 192.168.234.114 k8s-worker01
IP: 192.168.234.115 k8s-worker02

虚拟ip:
IP: 192.168.234.110
此IP为集群虚拟出来的IP，用于高可用。集群外部实际访问的是这个地址
```

一、所有服务器准备

1.1 调整操作系统参数
```
# vi /etc/security/limits.conf

添加以下内容
* soft nofile 65536
* hard nofile 65536
* soft nproc 10240
* hard nproc 10240
```

1.2 重启服务器使配置生效

1.3 检查配置
```
# ulimit -n
# ulimit -u
```

1.4 安装网络客户端工具

```
# yum install -y net-tools
```

1.5 安装ipvsadm

```
# yum install -y ipvsadm
```

1.6 关闭selinux防火墙
```
# setenforce 0
# systemctl stop firewalld && systemctl disable firewalld
# vi /etc/selinux/config  修改SELINUX=disabled

注意：不是修改SELINUXTYPE
```

二、etcd服务器安装(三台master上安装)

1、下载安装etcd
```
# yum install -y etcd
```

2、 vi /etc/etcd/etcd.conf 修改为如下信息
```
#[Member]
ETCD_NAME="infra01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

#[Clustering]
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.234.111:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.234.111:2379"
ETCD_INITIAL_CLUSTER="infra01=http://192.168.234.111:2380,infra02=http://192.168.234.112:2380,infra03=http://192.168.234.113:2380"
```
注意：ETCD_INITIAL_ADVERTISE_PEER_URLS和ETCD_ADVERTISE_CLIENT_URLS内容的IP地址和etcd_name名字要根据不同服务器调整
另，只有2台服务启动后，集群才可能进行连接测试，systemctl status etcd才可能是running

3、启动etcd服务
```
# systemctl enable etcd & systemctl start etcd
```

4、etcd集群验证
```
# etcdctl cluster-health
# etcdctl member list
```

二、所有K8S服务器准备：

2.1 安装并启动docker
```
# yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-18.06.1.ce-3.el7.x86_64.rpm

# systemctl enable docker && systemctl restart docker
```
2.2  开启forward
```
# iptables -P FORWARD ACCEPT
```

2.3 加载ipvs相关内核模块
```
# modprobe ip_vs
# modprobe ip_vs_rr
# modprobe ip_vs_wrr
# modprobe ip_vs_sh
# modprobe nf_conntrack_ipv4
# modprobe br_netfilter
# lsmod | grep ip_vs
```

2.4 配置系统路由参数
```
# vi /etc/sysctl.conf 添加以下内容
net.ipv4.ip_forward = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_local_port_range = 30000 60999
net.netfilter.nf_conntrack_max = 26214400
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 3600
vm.swappiness=0

# sysctl -p
```

2.5 设置使用阿里镜像安装建立repo文件：
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

2.6  安装 kubeadm, kubelet 和 kubectl
```
# yum install -y kubectl-1.12.8
# yum install -y kubelet-1.12.8
# yum install -y kubeadm-1.12.8

```

2.7 配置kubelet使用国内pause镜像，修改cgroup配置
```
# vi /etc/sysconfig/kubelet

添加配置如下：
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

2.8 启动kubelet
```
# systemctl daemon-reload
# systemctl enable kubelet && systemctl start kubelet
```

2.9 拉取镜像

```
# docker pull mirrorgooglecontainers/kube-apiserver:v1.12.8
# docker pull mirrorgooglecontainers/kube-proxy:v1.12.8
# docker pull mirrorgooglecontainers/kube-controller-manager:v1.12.8
# docker pull mirrorgooglecontainers/kube-scheduler:v1.12.8
# docker pull kuberneter/coredns:1.2.2
# docker pull mirrorgooglecontainers/pause:3.1
给镜像打标签
# docker tag mirrorgooglecontainers/kube-apiserver:v1.12.8 k8s.gcr.io/kube-apiserver:v1.12.8
# docker tag mirrorgooglecontainers/kube-controller-manager:v1.12.8 k8s.gcr.io/kube-controller-manager:v1.12.8
# docker tag mirrorgooglecontainers/kube-scheduler:v1.12.8 k8s.gcr.io/kube-scheduler:v1.12.8
# docker tag mirrorgooglecontainers/kube-proxy:v1.12.8 k8s.gcr.io/kube-proxy:v1.12.8
# docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
# docker tag kuberneter/coredns:1.2.2 k8s.gcr.io/coredns:1.2.2
删除临时镜像
# docker rmi mirrorgooglecontainers/kube-apiserver:v1.12.8
# docker rmi mirrorgooglecontainers/kube-proxy:v1.12.8
# docker rmi mirrorgooglecontainers/kube-controller-manager:v1.12.8
# docker rmi mirrorgooglecontainers/kube-scheduler:v1.12.8
# docker rmi kuberneter/coredns:1.2.2
# docker rmi mirrorgooglecontainers/pause:3.1
```

三、首台master安装

3.1 写hosts文件,解析本机的机器名

```
# vi /etc/hosts

192.168.234.111 k8s-master01
192.168.234.112 k8s-master02
192.168.234.113 k8s-master03
```

3.2 LVS的real_server安装

3.2.1 配置VIP地址绑定在lo网卡上

```
# mkdir /opt/scripts
# vi /opt/scripts/lvs_real.sh

#!/bin/bash
#description: Config realserver

VIP=192.168.234.110

. /etc/rc.d/init.d/functions

case "$1" in

start)
       ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
       /sbin/route add -host $VIP dev lo:0
       echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
       sysctl -p >/dev/null 2>&1
       echo "RealServer Start OK"
       ;;

stop)
       ifconfig lo:0 down
       route del $VIP >/dev/null 2>&1
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
       echo "RealServer Stoped"
       ;;

status)
        #Status of LVS-DR real server.
        islothere=`/sbin/ifconfig lo:0 | grep $VIP`
        isrothere=`netstat -rn | grep "lo:0" | grep $VIP`
        if [ ! "$islothere" -o ! "isrothere" ];then
            # Either the route or the lo:0 device
            # not found.
            echo "LVS-DR real server Stopped."
        else
            echo "LVS-DR Running."
        fi
        ;;

*)
        #Invalid entry.
        echo "$0: Usage: $0 {start|status|stop}"
        exit 1
        ;;
esac
exit 0

# chmod +x /opt/scripts/lvs_real.sh
# /opt/scripts/lvs_real.sh start

```

3.2.2 查看lo网口绑定VIP状态

```
# ip a
```

3.2.3 配置lvs_real.sh脚本开机自动执行

```
# vi /etc/rc.d/rc.local

#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.
 
#touch /var/lock/subsys/local
 
bash /opt/scripts/lvs_real.sh start
```

添加执行权限

```
# chmod +x /etc/rc.d/rc.local
```

3.2.4 编辑rc-local.service在末尾添加[Install]部分

```
# vi /usr/lib/systemd/system/rc-local.service

#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
 
# This unit gets pulled automatically into multi-user.target by
# systemd-rc-local-generator if /etc/rc.d/rc.local is executable.
[Unit]
Description=/etc/rc.d/rc.local Compatibility
ConditionFileIsExecutable=/etc/rc.d/rc.local
After=network.target
 
[Service]
Type=forking
ExecStart=/etc/rc.d/rc.local start
TimeoutSec=0
RemainAfterExit=yes
 
[Install]
WantedBy=multi-user.target

```

3.2.5 设置开机启动

```
systemctl daemon-reload
systemctl enable rc-local.service
systemctl start rc-local.service
```


3.3 k8s-master01安装

3.3.1 建立模板配置文件

```
# mkdir /root/k8s-install
# cd /root/k8s-install
# vi kubeadm-config.yaml
```

模板配置文件内容如下，注意管理端口设为虚拟IP

```
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: 1.12.8
apiServerCertSANs:
- "192.168.234.110"
controlPlaneEndpoint: "192.168.234.110:6443"
etcd:
  external:
    endpoints:
    - http://127.0.0.1:2379
networking:
  podSubnet: "10.244.0.0/16"
kubeProxy:
  config:
    mode: ipvs
```

3.3.2 安装k8s-master01

```
# kubeadm init --config kubeadm-config.yaml
```

3.3.3 配置环境变量

```
# vi ~/.bash_profile
加入一行：export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
```

3.3.4 flannel安装

```
# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```


3.3.5 拷贝权限文件到其它二台master服务器上

先在另外两台服务器上建立目录

```
# mkdir /etc/kubernetes/pki
```
在master1上操作

```
# scp /etc/kubernetes/pki/* k8s-master02:/etc/kubernetes/pki/
# scp /etc/kubernetes/admin.conf k8s-master02:/etc/kubernetes/admin.conf

# scp /etc/kubernetes/pki/* k8s-master03:/etc/kubernetes/pki/
# scp /etc/kubernetes/admin.conf k8s-master03:/etc/kubernetes/admin.conf
```

四、配置LVS服务器

只有配置LVS负载均衡服务后，其它master才可以继续按照

4.1 Linux 内核参数配置

```
# vi /etc/sysctl.conf

net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1

# sysctl -p
```

4.2 安装keepalived

```
# yum -y install keepalived
```

4.3 配置故障检测文件

```
# vi /etc/keepalived/chk_keepalived.sh

#!/bin/bash
keepalived_counter=$(ps -C keepalived --no-heading|wc -l)

if [ "${keepalived_counter}" = "0" ]; then
  /usr/sbin/keepalived
fi

# chmod +x /etc/keepalived/chk_keepalived.sh

```

4.4 配置keepalived

lvs-1的配置文件

```
# vi /etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckKeepalived {
    script "/etc/keepalived/chk_keepalived.sh"
    interval 3
    weight -10
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface enp0s3 #此处为网卡名
    virtual_router_id 66
    priority 100
    advert_int 1
    vrrp_garp_master_repeat 5
    vrrp_garp_master_refresh 10
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.110 dev enp0s3 label enp0s3:vip #此处为网卡名
    }
    track_script {
        CheckKeepalived
    }
}

virtual_server 192.168.234.110 6443 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
#    persistence_timeout 0
    protocol TCP

    real_server 192.168.234.111 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10
        }
    }
#    real_server 192.168.234.112 6443 {
#        weight 10
#        TCP_CHECK {
#            connect_timeout 10
#        }
#    }
#    real_server 192.168.234.113 6443 {
#        weight 10
#        TCP_CHECK {
#            connect_timeout 10
#        }
#    }
}
```

lvs-2的配置文件

```
# vi /etc/keepalived/keepalived.conf

! Configuration File for keepalived
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckKeepalived {
    script "/etc/keepalived/chk_keepalived.sh"
    interval 3
    weight -10
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface enp0s3 #此处为网卡名
    virtual_router_id 66
    priority 95
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.110 dev enp0s3 label enp0s3:vip #此处为网卡名
    }
    track_script {
        CheckKeepalived
    }
}

virtual_server 192.168.234.110 6443 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 0
    protocol TCP

    real_server 192.168.234.111 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10
        }
    }
#    real_server 192.168.234.112 6443 {
#        weight 10
#        TCP_CHECK {
#            connect_timeout 10
#        }
#    }
#    real_server 192.168.234.113 6443 {
#        weight 10
#        TCP_CHECK {
#            connect_timeout 10
#        }
#    }
}
```

注意

```
* 根据服务器网卡名称调整配置
* 192.168.234.112和192.168.234.113两台服务器暂时未搭好, 因此此处应是注释掉的
* 主备两台lvs的keepalived.conf有区别，缺省一个为MASTER一个为SLAVE
```

4.5 启动keepalived服务

```
# systemctl enable keepalived & systemctl start keepalived
```

五、剩余2台master安装

5.1 添加到k8s集群

```
kubeadm join 192.168.234.110:6443 --token yo2skv.twravx0k0x85g67d --discovery-token-ca-cert-hash sha256:415b5d84051dc6cace21410f5159ae305e2cbe44f9f4a3ad393e4809e964a743  --experimental-control-plane
```

5.2 配置环境变量

```
# vi ~/.bash_profile
加入一行：export KUBECONFIG=/etc/kubernetes/admin.conf
# source ~/.bash_profile
```

5.3 安装验证

```
# kubectl get pod -n kube-system
# kubectl get node
```

5.4 LVS调整

前面的配置中，LVS负载均衡仅仅指向了K8S-MASTER01，需要调整为指向3台MASTER

5.4.1 在k8s-master02和k8s-master03上重复3.2动作, 建立对应的real_server

5.4.2 调整lvs-1和lvs-2的keepalived.config文件, 打开针对192.168.234.112和192.168.234.113的注释并重启keepalived服务

六、高可用验证

6.1 关闭LVS-1检验效果

6.2 关闭K8S-MASTER01检验效果

七、LVS对traefik的负载均衡

参考如下keepalived.config配置文件：
```
#192.168.234.101(lvs-1)
! Configuration File for keepalived
global_defs {
   router_id LVS_k8s
}
 
vrrp_script CheckKeepalived {
    script "/etc/keepalived/chk_keepalived.sh"
    interval 3
    weight -10
    fall 2
    rise 2
}
 
vrrp_instance VI_1 {
    state MASTER
    interface enp0s3
    virtual_router_id 66
    priority 100
    advert_int 1
    vrrp_garp_master_repeat 5
    vrrp_garp_master_refresh 10
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.110 dev enp0s3 label enp0s3:vip 
    }
    track_script {
        CheckKeepalived
    }
}
 
virtual_server 192.168.234.110 6443 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
#    persistence_timeout 0
    protocol TCP
 
    real_server 192.168.234.111 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
    real_server 192.168.234.112 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
    real_server 192.168.234.113 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
}

vrrp_instance VI_2 {
    state BACKUP
    interface enp0s3
    virtual_router_id 67
    priority 95
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.100 dev enp0s3 label enp0s3:vip 
    }
    track_script {
        CheckKeepalived
    }
}

virtual_server 192.168.234.100 30001 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 0
    protocol TCP

    real_server 192.168.234.114 30001 {
        weight 10
        TCP_CHECK {
            connect_timeout 10
        }
    }
    real_server 192.168.234.115 30001 {
        weight 10
        TCP_CHECK {
            connect_timeout 10
        }
    }
}
```

```
#192.168.234.102(lvs-2)
! Configuration File for keepalived
global_defs {
   router_id LVS_k8s
}
 
vrrp_script CheckKeepalived {
    script "/etc/keepalived/chk_keepalived.sh"
    interval 3
    weight -10
    fall 2
    rise 2
}
 
vrrp_instance VI_1 {
    state BACKUP
    interface enp0s3
    virtual_router_id 66
    priority 95
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.110 dev enp0s3 label enp0s3:vip
    }
    track_script {
        CheckKeepalived
    }
}
 
virtual_server 192.168.234.110 6443 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 0
    protocol TCP
 
    real_server 192.168.234.111 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
    real_server 192.168.234.112 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
    real_server 192.168.234.113 6443 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
}

vrrp_instance VI_2 {
    state MASTER
    interface enp0s3
    virtual_router_id 67
    priority 100
    advert_int 1
    vrrp_garp_master_repeat 5
    vrrp_garp_master_refresh 10
    authentication {
        auth_type PASS
        auth_pass 6666
    }
    virtual_ipaddress {
        192.168.234.100 dev enp0s3 label enp0s3:vip 
    }
    track_script {
        CheckKeepalived
    }
}
 
virtual_server 192.168.234.100 30001 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    protocol TCP
 
    real_server 192.168.234.114 30001 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
    real_server 192.168.234.115 30001 {
        weight 10
        TCP_CHECK {
            connect_timeout 10 
        }
    }
}
```

说明:

a. LVS-1作为k8s-master的主负载均衡节点, LVS-2为BACKUP; LVS-2作为traefik的主负载均衡节点, LVS-1为BACKUP. 这样可以分摊网络流量, 避免全部集中在一台服务器上

b. traefik应使用DaemonSet方式部署, 用nodeSelector方式指定部署traefik的节点(边缘节点edgenode), 用hostPort方式对外提供服务, lvs对这些边缘节点进行负载均衡

请参考如下traefik部署配置文件traefik-daemonset.yaml
```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 30001
        - name: admin
          containerPort: 8080
        securityContext:
          privileged: true
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
      nodeSelector:
        edgenode: "true"
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - nodePort: 30001
      protocol: TCP
      port: 30001
      targetPort: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort
```