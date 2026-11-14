---
title: "【实战加详解】二进制部署k8s高可用集群系列教程八 - 部署 kube-controller"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:15:00Z
tags:
  - Kubernetes
description: "kube-controller-mangae 作为 kube-apiserver 的控制器，需要访问 kube-apiserver 的服务，所以需要 kube-apiserver 的 ca 机构为其签发客户端 client 证书。 这里 kube-controller 部署在 kube-apiserve..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kube-controller-manager`

***

# 部署 `kube-controll-manager`

`kube-controller-mangae` 作为 `kube-apiserver` 的控制器，需要访问 `kube-apiserver` 的服务，所以需要 `kube-apiserver`
的 `ca` 机构为其签发客户端 `client` 证书。
  

  

这里 `kube-controller` 部署在 `kube-apiserver` 的服务器上，不单独部署。
  

`ip` 设置成 `kube-apiserver` 的一样的就行。

## 1.创建 `ssl` 证书

### 1-1.创建 `csr` 请求文件

> [!NOTE]
> `CN` 参数表示用户名，必须设置为 `k8s` 中设定的 `system:kube-controller-manager`  

> `O` 参数表示用户组，必须设置为 `k8s` 中设定的 `system:kube-controller-manager`  

> `kubernetes` 内置的 `ClusterRoleBindings` 中，`system:kube-controller-manager` 赋予 `kube-controller-manager`
> 工作所需的权限。  

> `kube-controller-manager` 同样作为客户端，不需要设置 `hosts` 参数。

```shell
cat > /ssl/apiserver-controller-client-csr.json <<EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "system:kube-controller-manager",
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
-profile=client apiserver-controller-client-csr.json | \
cfssljson -bare apiserver-controller-client && \
ls apiserver-controller-client* | \
grep apiserver-controller-client
```

### 1-2.分发证书

```shell
scp /ssl/apiserver-controller-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-controller-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-controller-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 2.绑定 `cluster-admin` 集群角色

> [!NOTE]
> 二进制安装的 `kube-controller-manager` 没有操作其他命名空间的资源，需要赋予一些权限。
>   

> 这里我直接赋予了最高权限的集群角色：`cluster-admin`，省的再研究需要啥权限了。
>   

> 如果角色权限太大，自行裁剪权限。

```shell
cat > /etc/kubernetes/init_k8s_config/controller-bind-cluster-admin-clusterrole.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: controller-bind-cluster-admin
subjects:
- kind: User
  name: system:kube-controller-manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/controller-bind-cluster-admin-clusterrole.yaml
```

## 3.创建 `kubeconfig`

> [!NOTE]
> `kube-controller-manager` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。  

> 在 `kubeconfig` 配置文件中会包含了 `kube-controller-manager` 的客户端 `client` 证书信息以及身份信息。  

>   

> 需要在每台服务器都创建该请求文件。  

> 以下操作在每台 `master` 服务器创建，`ip` 地址设置为本地的 `kube-apiserver` 的服务地址 `ip`。

### 3-1.在 `master-01` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.121:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-controller-manager  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

### 3-2.在 `master-02` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-controller-manager  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

### 3-3.在 `master-03` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-controller-manager  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

## 4.启动服务

### 4-1.创建启动配置文件

> [!NOTE]
> 每台 `master` 服务器都要创建。每台服务器的启动配置文件一样。
>   

> 需要注意的是：参数 `--service-cluster-ip-range` 设置的值必须要跟 `kube-apiserver` 设置的一致。
>   

>   

> 注意：以下配置中，日志等级设置为：`6` 。产生的日志的速度会非常快。学习部署后可以修改为：`2`。

```shell
cat > /etc/kubernetes/config/controller.conf <<EOF
KUBE_CONTROLLER_MANAGER_OPTS="--port=0 \

--secure-port=10257 \

--bind-address=127.0.0.1 \

--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig \

--service-cluster-ip-range=10.68.0.1/16 \

--cluster-name=kubernetes \

--cluster-signing-cert-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \

--cluster-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem \

--root-ca-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \

--allocate-node-cidrs=true \

--cluster-cidr=10.100.0.0/16 \

--cluster-signing-duration=87600h \

--leader-elect=true \

--controllers=*,bootstrapsigner,tokencleaner \

--horizontal-pod-autoscaler-sync-period=10s \

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \

--alsologtostderr=true \

--logtostderr=false \

--log-dir=/var/log/kubernetes/controller/ \

--v=4"
EOF
```

### 4-2.创建服务启动项

> [!NOTE]
> 需要在每台 `master` 服务器创建。每台服务器的启动项一样。

```shell
cat > /usr/lib/systemd/system/kube-controller-manager.service <<'EOF'
[Unit]
Description=Kubernetes Controller Manager Service
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config/controller.conf
ExecStart=/usr/local/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 4-3.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-controller-manager
```

## 5.验证

使用 `kubectl` 查看 `kube-apiserver` 是否可以访问到 `kube-controller-manager`

```shell
kubectl get cs
```

*显示*
> [!NOTE]
> 可以看到 `kube-controller-manager` 的 `STATUS` 为 `Healthy`。  

> `kube-scheduler` 的状态为 `Unhealthy` 是因为还没部署。接下来开始部署。

```text
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
controller-manager   Healthy     ok
etcd-0               Healthy     {"health":"true","reason":""}
etcd-2               Healthy     {"health":"true","reason":""}
etcd-1               Healthy     {"health":"true","reason":""}
```

## 6.设置开机启动

正常启动且没有任何错误，设置开机自动启动服务。

```shell
systemctl enable kube-controller-manager
```

## 7.其他操作

停止服务

```shell
systemctl stop kube-controller-manager
```

查看状态

```shell
systemctl status kube-controller-manager
```

查看服务运行日志

```shell
journalctl -l --no-pager  -u kube-controller-manager
```

***

## 至此。`kube-controller-manager` 部署成功。
