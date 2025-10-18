---
title: 修改运行中kubernetes集群中etcd的参数
link: kubernetes-etcd-config-modify
date: 2023-01-04 19:32:24
tags:
  - Kubernetes
---
在一些场景下，您的kubernetes集群已经搭建完成了，但是还需要修改一些核心组件的参数，如etcd、kube-apiserver、kube-scheduler、kube-controller-manager等。

通过`kubectl get pod -owide -n kube-system` 可以查看到这些核心容器。

```bash
NAME                               READY   STATUS    RESTARTS       AGE
coredns-78fcd69978-rdmjm           1/1     Running   11 (23s ago)   281d
etcd-$NODE1                        1/1     Running   13 (23s ago)   281d
etcd-$NODE2                        1/1     Running   13 (23s ago)   281d
etcd-$NODE3                        1/1     Running   13 (23s ago)   281d
.....
```

以etcd为例，etcd的参数就在pod中的commands参数里。可以通过`kubectl describe pod etcd-$NODENAME -n kube-system`来查看(省略部分参数)

```bash
Name: etcd-$NODENAME
Namespace: kube-system
Containers:
etcd:
Command:
--client-cert-auth=true
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

然而，如果您尝试编辑pod中的参数，会发现它们是不可修改的。

不过，如果您需要修改参数，还有另一个办法，通过修改`/etc/kubernetes/manifests/`下的yaml文件来修改运行中kubernetes集群中"系统"Pod的参数。原理是，当您把yaml文件修改后，kubelet会自动监听yaml文件的变更，并重新拉起本机器上的pod。

举个例子，如果您希望关闭etcd集群对客户端的认证，那么您可以修改`/etc/kubernetes/mainfiest/etcd.yaml`,将**client-cert-auth**设置为false，把**trusted-ca-file**去掉。注意：三台master机器节点都需要执行此操作
