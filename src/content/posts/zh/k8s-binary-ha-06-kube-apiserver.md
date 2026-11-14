---
title: "【实战加详解】二进制部署k8s高可用集群系列教程六 -部署 kube-apiserver"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:14:00Z
tags:
  - Kubernetes
description: "分发 etcd-ca.pem 密钥给 kube-apiserver 分发 kube-apiserver 的 ca 根证书 分发 etcd 颁发给 kube-apiserver 的 client 证书 分发所有 kube-apiserver 的 server 证书 分发到每台 master 服务器"
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kube-apiserver`

***

# 部署 `kube-apiserver`

## 1.创建目录

> [!NOTE]
> 每台 `master` 服务器都要创建。

```shell
# 创建证书目录
mkdir -p /etc/kubernetes/pki/{apiserver/,kubelet/,aggregator/,service-account/,sign/,etcd/}

# 创建配置文件存放目录以及 kubeconfig 存放目录和初始化集群所需配置文件目录
mkdir -p /etc/kubernetes/{config/,kubeconfig/,init_k8s_config/}

# 创建 kubectl 使用 config 的默认目录
mkdir -p /root/.kube/

# 创建日志存放目录
mkdir -p /var/log/kubernetes/{apiserver/,controller/,scheduler/}
```

## 2.创建 `ssl` 证书

### 2-1.创建 `ca` 根证书

> [!NOTE]
> `ca` 根证书采用 `4096` 位加密。  

> 创建证书可以在其他地方生成后再上传，设置好对应的 `ip` 就行。

```shell
cat > /ssl/apiserver-ca-csr.json <<EOF
{
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert -initca apiserver-ca-csr.json | \
cfssljson -bare apiserver-ca - && \
ls apiserver-ca* | \
grep apiserver-ca
```

### 2-2.创建 `server` 证书

> [!NOTE]
> `hosts` 参数的 `ip` 需要把 `master` 节点，`HA` 节点以及 `vip` 地址都写进去。

```shell
cat > /ssl/apiserver-server-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "172.16.222.121",
        "172.16.222.122",
        "172.16.222.123",
        "172.16.222.201",
        "172.16.222.202",
        "172.16.222.110",
        "10.68.0.1",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "k8s",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert -ca=apiserver-ca.pem \
-ca-key=apiserver-ca-key.pem \
-config=ca-config.json \
-profile=server apiserver-server-csr.json | \
cfssljson -bare apiserver-server && \
ls apiserver-server* | \
grep apiserver-server
```

### 2-3.创建 `etcd` 提供给 `kube-apiserver` 访问的 `client` 证书

> [!NOTE]
> 由于是访问 `etcd` 的服务，所以要使用 `etcd` 的 `ca` 机构签发证书。

```shell
cat > /ssl/etcd-apiserver-client-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "k8s",
            "OU": "system"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert -ca=etcd-ca.pem \
-ca-key=etcd-ca-key.pem \
-config=ca-config.json \
-profile=client etcd-apiserver-client-csr.json | \
cfssljson -bare etcd-apiserver-client && \
ls etcd-apiserver-client* | \
grep etcd-apiserver-client
```

## 3.分发证书

> [!NOTE]
> 以下证书全部分发到 `kube-apiserver` 的服务器。

分发 `etcd-ca.pem` 密钥给 `kube-apiserver`

```shell
scp /ssl/etcd-ca.pem root@172.16.222.121:/etc/kubernetes/pki/etcd/ && \

scp /ssl/etcd-ca.pem root@172.16.222.122:/etc/kubernetes/pki/etcd/ && \

scp /ssl/etcd-ca.pem root@172.16.222.123:/etc/kubernetes/pki/etcd/
```

分发 `kube-apiserver` 的 `ca` 根证书

```shell
scp /ssl/apiserver-ca*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-ca*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-ca*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

分发 `etcd` 颁发给 `kube-apiserver` 的 `client` 证书

```shell
scp /ssl/etcd-apiserver-client*.pem root@172.16.222.121:/etc/kubernetes/pki/etcd/ && \

scp /ssl/etcd-apiserver-client*.pem root@172.16.222.122:/etc/kubernetes/pki/etcd/ && \

scp /ssl/etcd-apiserver-client*.pem root@172.16.222.123:/etc/kubernetes/pki/etcd/
```

分发所有 `kube-apiserver` 的 `server` 证书

```shell
scp /ssl/apiserver-server*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-server*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-server*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 4.下载 `k8s` `server` 端的二进制文件

> [!NOTE]
> 每台 `master` 服务器都要下载。

下载

```shell
cd /tmp && \
wget https://dl.k8s.io/v1.23.9/kubernetes-server-linux-amd64.tar.gz
```

分发到每台 `master` 服务器

```shell
scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.121:/home/ && \

scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.122:/home/ && \

scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.123:/home/
```

解压
  

*到每台 `master` 服务器解压*

```shell
cd /home/ && \
tar -zxvf kubernetes-server-linux-amd64.tar.gz
```

复制二进制执行文件到 `/usr/local/bin/`
  

*到每台 `master` 服务器复制*

```shell
cd /home/kubernetes/server/bin/ && \
cp kube-apiserver kube-controller-manager kubectl kube-scheduler /usr/local/bin/
```

## 5.部署 `kube-apiserver`

### 5-1.创建服务启动配置文件

> [!NOTE]
> 以下操作需要登录到对应的 `master` 服务器操作。  

> 只需要修改配置参数中的 `ip` 地址即可。
>   

>   

> 注意：以下配置中，日志等级设置为：`6` 。产生的日志的速度会非常快。学习部署后可以修改为：`2`。

#### 5-1-1.创建 `master-01` 服务器的启动配置文件

```shell
cat > /etc/kubernetes/config/apiserver.conf <<EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota

--anonymous-auth=false

--bind-address=172.16.222.121

--secure-port=6443

--advertise-address=172.16.222.121

--insecure-port=0

--authorization-mode=Node,RBAC

--runtime-config=api/all=true

--service-cluster-ip-range=10.68.0.1/16

--service-node-port-range=30000-39999

--service-account-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-server.pem

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-server-key.pem

--client-ca-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--service-account-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem

--service-account-issuer=https://kubernetes.default.svc.cluster.local

--api-audiences=https://kubernetes.default.svc

--etcd-cafile=/etc/kubernetes/pki/etcd/etcd-ca.pem

--etcd-certfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client.pem

--etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client-key.pem

--etcd-servers=https://172.16.222.111:2379,https://172.16.222.112:2379,https://172.16.222.113:2379

--feature-gates=RemoveSelfLink=false

--enable-swagger-ui=true

--allow-privileged=true

--apiserver-count=3

--enable-aggregator-routing=true

--audit-log-maxage=30

--audit-log-maxbackup=3

--audit-log-maxsize=100

--audit-log-path=/var/log/kubernetes/apiserver/apiserver-audit.log

--event-ttl=1h

--alsologtostderr=true

--logtostderr=false

--log-dir=/var/log/kubernetes/apiserver/

--v=6"
EOF
```

#### 5-1-2.创建 `master-02` 服务器的启动配置文件

```shell
cat > /etc/kubernetes/config/apiserver.conf <<EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota

--anonymous-auth=false

--bind-address=172.16.222.122

--secure-port=6443

--advertise-address=172.16.222.122

--insecure-port=0

--authorization-mode=Node,RBAC

--runtime-config=api/all=true

--service-cluster-ip-range=10.68.0.1/16

--service-node-port-range=30000-39999

--service-account-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-server.pem

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-server-key.pem

--client-ca-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--service-account-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem

--service-account-issuer=https://kubernetes.default.svc.cluster.local

--api-audiences=https://kubernetes.default.svc

--etcd-cafile=/etc/kubernetes/pki/etcd/etcd-ca.pem

--etcd-certfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client.pem

--etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client-key.pem

--etcd-servers=https://172.16.222.111:2379,https://172.16.222.112:2379,https://172.16.222.113:2379

--feature-gates=RemoveSelfLink=false

--enable-swagger-ui=true

--allow-privileged=true

--apiserver-count=3

--enable-aggregator-routing=true

--audit-log-maxage=30

--audit-log-maxbackup=3

--audit-log-maxsize=100

--audit-log-path=/var/log/kubernetes/apiserver/apiserver-audit.log

--event-ttl=1h

--alsologtostderr=true

--logtostderr=false

--log-dir=/var/log/kubernetes/apiserver/

--v=6"
EOF
```

#### 5-1-3.创建 `master-03` 服务器的启动配置文件

```shell
cat > /etc/kubernetes/config/apiserver.conf <<EOF
KUBE_APISERVER_OPTS="--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota

--anonymous-auth=false

--bind-address=172.16.222.123

--secure-port=6443

--advertise-address=172.16.222.123

--insecure-port=0

--authorization-mode=Node,RBAC

--runtime-config=api/all=true

--service-cluster-ip-range=10.68.0.1/16

--service-node-port-range=30000-39999

--service-account-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--tls-cert-file=/etc/kubernetes/pki/apiserver/apiserver-server.pem

--tls-private-key-file=/etc/kubernetes/pki/apiserver/apiserver-server-key.pem

--client-ca-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--service-account-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem

--service-account-issuer=https://kubernetes.default.svc.cluster.local

--api-audiences=https://kubernetes.default.svc

--etcd-cafile=/etc/kubernetes/pki/etcd/etcd-ca.pem

--etcd-certfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client.pem

--etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-apiserver-client-key.pem

--etcd-servers=https://172.16.222.111:2379,https://172.16.222.112:2379,https://172.16.222.113:2379

--feature-gates=RemoveSelfLink=false

--enable-swagger-ui=true

--allow-privileged=true

--apiserver-count=3

--enable-aggregator-routing=true

--audit-log-maxage=30

--audit-log-maxbackup=3

--audit-log-maxsize=100

--audit-log-path=/var/log/kubernetes/apiserver/apiserver-audit.log

--event-ttl=1h

--alsologtostderr=true

--logtostderr=false

--log-dir=/var/log/kubernetes/apiserver/

--v=6"
EOF
```

### 5-2.创建服务启动文件

> [!NOTE]
> 使用 `cat` 命令创建文件时，环境变量参数会丢失。需要在开头的 `EOF` 加上单引号即可。  

> 在每台 `master` 服务器都要创建。每个启动文件都一样。

```shell
cat > /usr/lib/systemd/system/kube-apiserver.service <<'EOF'
[Unit]
Description=Kubernetes API Server Service
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config/apiserver.conf
ExecStart=/usr/local/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 5-3.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-apiserver
```

### 5-4.设置开机启动

没有错误后，设置开启启动

```shell
systemctl enable kube-apiserver
```

### 5-5.其他操作

停止服务

```shell
systemctl stop kube-apiserver
```

查看状态

```shell
systemctl status kube-apiserver
```

查看错误

```shell
journalctl -l --no-pager  -u kube-apiserver
```

删除进程日志

```shell
rm -rvf /var/log/journal/*
```

***

## 至此。`kube-apiserver` 组件部署成功。
