---
title: "【实战加详解】二进制部署k8s高可用集群教程系列九 -部署 scheduler"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:16:00Z
tags:
  - Kubernetes
description: "kube-scheduler 作为 kube-apiserver 的调度器，需要访问 kube-apiserver 的服务，所以需要 kube-apiserver 的 ca 机构为其签发客户端 client 证书。 这里 kube-controller 部署在 kube-apiserver 的服务器上，..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kube-scheduler`

***

# 部署 `kube-scheduler`

`kube-scheduler` 作为 `kube-apiserver` 的调度器，需要访问 `kube-apiserver` 的服务，所以需要 `kube-apiserver` 的 `ca`
机构为其签发客户端 `client` 证书。
  

  

这里 `kube-controller` 部署在 `kube-apiserver` 的服务器上，不单独部署。
  

`ip` 设置成 `kube-apiserver` 的一样的就行。

## 1.生成 `ssl` 证书

### 1-1.创建 `csr` 请求文件

> [!NOTE]
> `CN` 参数表示用户名，必须设置为 `k8s` 中设定的 `system:kube-scheduler`  

> `O` 参数表示用户组，必须设置为 `k8s` 中设定的 `system:kube-scheduler`  

> `kubernetes` 内置的 `ClusterRoleBindings` 中，`system:kube-scheduler` 赋予 `kube-scheduler`
> 工作所需的权限。  

> `kube-schduler` 同样作为客户端，不需要设置 `hosts` 参数。

```shell
cat > /ssl/apiserver-scheduler-client-csr.json <<EOF
{
    "CN": "system:kube-scheduler",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "system:kube-scheduler",
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
-profile=client apiserver-scheduler-client-csr.json | \
cfssljson -bare apiserver-scheduler-client && \
ls apiserver-scheduler-client* | \
grep apiserver-scheduler-client
```

## 2.分发证书

```shell
scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 3.创建 `kubeconfig`

> [!NOTE]
> `kube-scheduler` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。  

> `kubeconfig` 配置文件中会包含了 `kube-scheduler` 的客户端 `client` 证书信息以及身份信息。  

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
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-scheduler  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

### 3-2.在 `master-02` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-scheduler  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

### 3-3.在 `master-03` 创建

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials system:kube-scheduler  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置上下文参数

```shell
kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

## 4.启动服务

### 4-1.创建启动配置文件

> [!NOTE]
> 每台 `master` 服务器都要创建。每台服务器的启动配置文件一样。
>   

>   

> 注意：以下的配置中，日志等级设置为：`4`。日志产生的速度会非常快。学习部署后可以设置为：`2`。

```shell
cat > /etc/kubernetes/config/scheduler.conf <<EOF
KUBE_SCHEDULER_OPTS="--bind-address=127.0.0.1 \

--secure-port=10259 \

--client-ca-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \

--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig \

--leader-elect=true \

--alsologtostderr=true \

--logtostderr=false \

--log-dir=/var/log/kubernetes/scheduler/ \

--v=4"
EOF
```

### 4-2.创建服务启动项

> [!NOTE]
> 需要在每台 `master` 服务器创建。每台服务器的启动项一样。

```shell
cat > /usr/lib/systemd/system/kube-scheduler.service <<'EOF'
[Unit]
Description=Kubernetes Scheduler Service
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config/scheduler.conf
ExecStart=/usr/local/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## 5.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-scheduler
```

## 6.验证

使用 `kubectl` 查看 `kube-apiserver` 是否可以访问到 `kube-scheduler`

```text
kubectl get cs
```

*显示*
> [!NOTE]
> 可以看到 `kube-controller-manager` 和 `kube-scheduler` 的 `STATUS` 都为 `Healthy`。

```text
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
etcd-1               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
scheduler            Healthy   ok
```

## 7.设置开机启动

正常启动且没有任何错误，设置开机自动启动服务。

```shell
systemctl enable kube-scheduler
```

## 8.其他操作

停止服务

```shell
systemctl stop kube-scheduler
```

查看状态

```shell
systemctl status kube-scheduler
```

查看服务运行状态

```shell
journalctl -l --no-pager  -u kube-scheduler
```

***

## 至此。`kube-scheduler` 部署成功。
