# zookeeper安装
本文介绍如何在kubernetes集群中部署单节点zookeeper

1、创建目录/home/deployment-k8s/zookeeper目录，并下载zookeeper.yaml和persistent-volume.yaml文件到该目录中

2、在/var/lib中创建zookeeper目录
```
cd /var/lib
mkdir zookeeper
```

3、创建zookeeper使用的用户组和用户，并将目录/var/lib/zookeeper的权限赋给新的用户
```
groupadd USER
useradd -g USER zookeeper
chown -R zookeeper:USER /var/lib/zookeeper
```

4、执行yaml文件
```
cd /home/deployment-k8s/zookeeper
kubectl apply -f .
```
