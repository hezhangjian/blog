---
title: Kubernetes pod内调用API
link: kubernetes-pod-api
date: 2021-04-13 21:17:20
tags:
  - Kubernetes
---

Kubernetes pod内调用API的流程总体分为以下步骤

- 创建role
- 创建serviceaccount
- 绑定role到serviceaccount
- 指定pod使用serviceaccount

我们以查pod为例，演示一下整个流程

## 创建role

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-hzj
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get","list"]
```

```bash
kubectl apply -f role.yaml
```

## 创建serviceaccount

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: serviceaccount-hzj
  namespace: default
```

```bash
kubectl apply -f serviceaccount.yaml
```

## 绑定role

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rolebinding-hzj
  namespace: default
subjects:
  - kind: ServiceAccount
    name: serviceaccount-hzj
    namespace: default
roleRef:
  kind: Role
  name: role-hzj
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding.yaml
```

## 部署pod进行测试

### 部署一个zookeeper进行测试

手上刚好有zookeeper的模板文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zookeeper
  labels:
    app: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: zookeeper
        image: ttbb/zookeeper:stand-alone
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 2G
            cpu: 1000m
          requests:
            memory: 2G
            cpu: 1000m
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
                fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
                fieldPath: metadata.name
        - name: PS1
          value: '[\u@zookeeper@\W]\$ '
```



### 调用API

```bash
# Point to the internal API server hostname
APISERVER=https://kubernetes.default.svc
# Path to ServiceAccount token
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
# Read this Pod's namespace
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)
# Read the ServiceAccount bearer token
TOKEN=$(cat ${SERVICEACCOUNT}/token)
# Reference the internal certificate authority (CA)
CACERT=${SERVICEACCOUNT}/ca.crt
# Explore the API with TOKEN
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api/v1/namespaces/default/pods
```

![kubernetes-pod-api1](Images/kubernetes-pod-api1.png)

发现这里，调用后面的api，403错误。第一个api不报错，是因为该接口不需要鉴权。

### 修改pod对应的serviceaccount

让我们修改部署模板对应的ServiceAccountName，注入权限。在pod的spec下，设置serviceAccountName

![kubernetes-pod-api2](Images/kubernetes-pod-api2.png)

### 修改部署模板重启后调用api正常

再次尝试上述命令，api结果返回正常

![kubernetes-pod-api3](Images/kubernetes-pod-api3.png)
