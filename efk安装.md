# efk安装
本文介绍如何在kubernetes集群中部署efk

下载demo/log-tools中的yaml文件

##一. 执行所有的yaml文件

```
# kubectl apply -f .
```

需注意的问题：

a. es-statefulset.yaml文件中，定义的路径为emptyDir: {}。 这意味着es-logging容器重启后会丢失掉所有的日志文件。如果对日志文件有记录要求，请使用外部存储，例如nfs等。

b. fluentd-es-ds.yaml中指定了selector
```
nodeSelector:
        beta.kubernetes.io/fluentd-ds-ready: "true"
```
因此，需要采集日志的服务器，需执行如下语句使fluentd可以创建在该服务器上
```
# kubectl label nodes k8s-node01 beta.kubernetes.io/fluentd-ds-ready=true
# kubectl label nodes k8s-node02 beta.kubernetes.io/fluentd-ds-ready=true
```
注意：如果只需要采集应用服务的日志，请不要在master上建立日志采集服务，除非是一个单机kubernetes

c. kibana-service中，指定了NodePort为30010，请根据需要自行调整。

d. demo中的kibana-deployment.yaml，去掉了原版中以下部分内容。否则无法kibana界面无法正常进入
```
- name: SERVER_BASEPATH
            value: /api/v1/namespaces/kube-system/services/kibana-logging/proxy
```

##二、访问kibina界面

```
http://222.128.2.110:30010
```

如果elastic-logging服务处于未就绪状态，请等待几分钟

如果不能正常启动，则检查各pod的日志信息