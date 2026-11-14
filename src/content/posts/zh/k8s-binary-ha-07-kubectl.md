---
title: "【实战加详解】二进制部署k8s高可用集群系列教程七 - 部署 kubectl"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:15:00Z
tags:
  - Kubernetes
description: "kubectl 作为 kube-apiserver 的客户端工具，需要访问 kube-apiserver 的服务，所以需要 kube-apiserver 的 ca 机构为其签发客户端 client 证书。 复制到 k8s 默认的引用目录位置 一般是 /root/.kube/ 目录 复制到 k8s 默认的..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kubectl`

***

# 部署 `kubectl`

`kubectl` 作为 `kube-apiserver` 的客户端工具，需要访问 `kube-apiserver` 的服务，所以需要 `kube-apiserver` 的 `ca`
机构为其签发客户端 `client` 证书。

## 1.生成 `ssl` 证书

### 1-1.创建 `csr` 请求文件

> [!NOTE]
> `CN` 参数表示用户名，这里设置为 `k8s` 中设定的 `admin`  

> `O` 参数表示用户组，这里设置为 `k8s` 中设定的 `system:masters`  

> `kubectl` 作为客户端，不需要设置 `hosts` 参数。
>   

>   

> `CN` 可以设置其他的，这里设置常用的 `admin` 用户角色就行。不推荐设置成 `cluster-admin` 超级管理员角色。

```shell
cat > /ssl/apiserver-admin-client-csr.json <<EOF
{
    "CN": "admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "system:masters",
            "OU": "system"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert \
-ca=apiserver-ca.pem \
-ca-key=apiserver-ca-key.pem \
-config=ca-config.json \
-profile=client apiserver-admin-client-csr.json | \
cfssljson -bare apiserver-admin-client && \
ls apiserver-admin-client* | \
grep apiserver-admin-client
```

### 1-2.分发证书

```shell
scp /ssl/apiserver-admin-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-admin-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-admin-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 2.创建 `kubeconfig`

> [!NOTE]
> `kubectl` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。  

> 在 `kubeconfig` 配置文件中会包含了 `kubectl` 的客户端 `client` 证书信息以及身份信息。  

> 需要在每台服务器都创建该请求文件。  

> 以下操作在每台 `master` 服务器创建，`ip` 地址设置为本地的 `kube-apiserver` 的服务地址 `ip`。  

> 文件存放目录为：`/etc/kubernetes/kubeconfig/`。

### 2-1.在 `master-01` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.121:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置当前上下文

```shell
kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

复制到 `k8s` 默认的引用目录位置
  

*一般是 `/root/.kube/` 目录*

```shell
cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

### 2-2.在 `master-02` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置当前上下文

```shell
kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

复制到 `k8s` 默认的引用目录位置
  

*一般是 `/root/.kube/` 目录*

```shell
cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

### 2-3.在 `master-03` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

设置当前上下文

```shell
kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig
```

复制到 `k8s` 默认的引用目录位置
  

*一般是 `/root/.kube/` 目录*

```shell
cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

## 3.验证

> [!NOTE]
> 每台 `master` 服务器配置 `admin.kubeconfig` 都验证一遍。

随便执行 `kubectl` 命令查看是否可以正常访问 `kube-apiserver`
  

  

查看集群信息

```shell
kubectl cluster-info
```

*显示*

```text
Kubernetes control plane is running at https://172.16.222.122:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

查看所有服务

```shell
kubectl get all --all-namespaces
```

*显示*

```text
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.68.0.1    <none>        443/TCP   120m
```

## 4.授予访问 `kubelet` 的权限

> [!NOTE]
> `kubectl` 很多时候需要查看 `pod` 的日志等其他信息，
>   

> `kubectl` 是通过 `kube-apiserver` 访问 `kubelet` 的服务的。
>   

> `k8s` 有自带的集群角色：`system:kubelet-api-admin` 拥有此权限，需要绑定到 `kubectl` 的用户上。
>   

> `kubectl` 的用户这里设置的是：`kubernetes`。

```shell
cat > /etc/kubernetes/init_k8s_config/allow-kubectl-access-kube-apiserver-kubelet-apis.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-kubectl-access-kube-apiserver-kubelet-apis
subjects:
- kind: User
  name: kubernetes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:kubelet-api-admin
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-kubectl-access-kube-apiserver-kubelet-apis.yaml
```

***

## 至此。`kubectl` 部署成功。
