---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十一 - 部署kubelet手动证书方式"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:17:00Z
tags:
  - Kubernetes
description: "在 node 节点上，kubelet 是需要对外提供服务的。在 k8s 中，调用 kubelet 服务的也只有 kube-apiserver。 所以，kubelet 可以拥有自身的 ca 签发机构。 kubelet 自身作为服务，需要生成自身的 server 证书。 并且 kube-apiserver..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kubelet`   

> 手动颁发证书方式

***

# 部署 `kubelet`

> [!NOTE]
> 部署 `kubelet` 之前需要初始化系统环境。

在 `node` 节点上，`kubelet` 是需要对外提供服务的。在 `k8s` 中，调用 `kubelet` 服务的也只有 `kube-apiserver`。
  

所以，`kubelet` 可以拥有自身的 `ca` 签发机构。
  

  

`kubelet` 自身作为服务，需要生成自身的 `server` 证书。
  

  

并且 `kube-apiserver` 需要调用 `kubelet` 以调度 `pod` ，所以 `kubelet` 需要为 `kube-apiserber` 签发客户端 `client` 证书。
  

  

`node` 节点上的 `kubelet` 在使用 `kubeconfig` 跟 `kube-apiserver` 建立连接的时候，也需要 `kube-apiserver`
为其颁发客户端 `client` 证书。

## 1.创建目录

```shell
# 创建证书存放目录
mkdir -p /etc/kubernetes/pki/{kubelet/,apiserver/}

# 创建日志存放目录
mkdir -p /var/log/kubernetes/{kubelet/,kube-proxy/}

# 创建存放 kubeconfig 的目录以及配置目录
mkdir -p /etc/kubernetes/{kubeconfig/,config/}

# 创建 kubelet 工作目录以及 kube-proxy 工作目录
mkdir -p /var/lib/{kubelet/,kube-proxy/}

```

## 2.安装 `docker`

添加源

```shell
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

安装

```shell
dnf install docker-ce -y
```

创建 `docder` 的 配置文件 `daemon-reload`
> [!NOTE]
> `Docker` 在默认情况下使用的 `Cgroup Driver` 为 `cgroupfs` ，而 `kubernetes` 推荐使用 `systemd` 来代替 `cgroupfs`

```shell
mkdir /etc/docker && \
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
    "http://hub-mirror.c.163.com",
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "log-opts": {
    "max-size": "100m"
  }
}
EOF
```

启动

```shell
systemctl daemon-reload && \
systemctl enable docker && \
systemctl enable containerd && \
systemctl start docker
```

检查启动状态

```shell
systemctl status docker && \
systemctl status containerd
```

## 3.下载镜像

```shell
docker pull registry.aliyuncs.com/google_containers/pause:3.6
```

## 4.上传组件

上传 `kubectl` , `kubelet` , `kube-proxy` 组件
  

*前面已经下载到 `k8s-master01` 服务器，直接上传即可*

```shell
cd /home/kubernetes/server/bin/ && \
scp kubectl kubelet kube-proxy root@172.16.222.231:/usr/local/bin/
```

## 5.创建 `ssl` 证书

### 5-1.创建 `kubelet` `ca` 根证书

> [!NOTE]
> `ca` 根证书采用 `4096` 位加密。

```shell
cat > /ssl/kubelet-ca-scr.json <<EOF
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
cfssl gencert -initca kubelet-ca-scr.json | \
cfssljson -bare kubelet-ca - && \
ls kubelet-ca* | \
grep kubelet-ca
```

### 5-2.创建 `kubelet` `server` 证书

> [!NOTE]
> 这里部署的 `node` 节点的 `ip` 地址为：`172.16.222.231`。  

> 生成的 `server` 证书只针对该服务器生成。

```shell
cat > /ssl/kubelet-server-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "172.16.222.231"
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
cfssl gencert -ca=kubelet-ca.pem \
-ca-key=kubelet-ca-key.pem \
-config=ca-config.json \
-profile=server kubelet-server-csr.json | \
cfssljson -bare kubelet-server && \
ls kubelet-server* | \
grep kubelet-server
```

### 5-3.创建 `kubelet` 提供给 `kube-apiserver` 访问的 `client` 证书

该证书由 `kubelet` 的 `ca` 签发机构创建。
  

客户端 `client` 证书不需要设置 `hosts` 参数。

```shell
cat > /ssl/kubelet-apiserver-client-csr.json <<EOF
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
cfssl gencert -ca=kubelet-ca.pem \
-ca-key=kubelet-ca-key.pem \
-config=ca-config.json \
-profile=client kubelet-apiserver-client-csr.json | \
cfssljson -bare kubelet-apiserver-client && \
ls kubelet-apiserver-client* | \
grep kubelet-apiserver-client
```

### 5-4.创建 `kube-apiserver` 提供给 `kubelet` 建立连接的 `client` 证书

> [!NOTE]
> `kubelet` 是使用 `system:node:<nodeName>` 的用户名以及 `system:nodes` 的用户组访问 `kube-apiserver` 的。  

> 只有指定了 `<nodeName>` 的 `kubelet` 才会被授权访问 `kube-apiserver`。

关于证书要求以及 `Node` 鉴权查看官方文档说明：
  

  

证书要求文档：(https://kubernetes.io/zh-cn/docs/setup/best-practices/certificates/)
  

`Node` 鉴权文档：(https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/node/)
  

  

该证书由 `kube-apiserver` 的 `ca` 签发机构创建。

```shell
cat > /ssl/apiserver-kubelet-client-csr.json <<EOF
{
    "CN": "system:node:k8s-node01",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "system:nodes",
            "OU": "system"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert -ca=apiserver-ca.pem \
-ca-key=apiserver-ca-key.pem \
-config=ca-config.json \
-profile=client apiserver-kubelet-client-csr.json | \
cfssljson -bare apiserver-kubelet-client && \
ls apiserver-kubelet-client* | \
grep apiserver-kubelet-client
```

## 6.分发证书

### 6-1.分发 `kubelet` 的 `ca` 证书

分发到 `master` 节点以及 `kubelet` 节点

```shell
scp /ssl/kubelet-ca.pem root@172.16.222.121:/etc/kubernetes/pki/kubelet/ && \

scp /ssl/kubelet-ca.pem root@172.16.222.122:/etc/kubernetes/pki/kubelet/ && \

scp /ssl/kubelet-ca.pem root@172.16.222.123:/etc/kubernetes/pki/kubelet/ && \

scp /ssl/kubelet-ca*.pem root@172.16.222.231:/etc/kubernetes/pki/kubelet/
```

### 6-2.分发 `kubelet` 的 `server` 证书

分发到 `kubelet` 节点

```shell
scp /ssl/kubelet-server*.pem root@172.16.222.231:/etc/kubernetes/pki/kubelet/
```

### 6-3.分发 `kubelet` 颁发给 `kube-apiserver` 的 `client` 证书

分发到 `master` 节点

```shell
scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.121:/etc/kubernetes/pki/kubelet/ && \

scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.122:/etc/kubernetes/pki/kubelet/ && \

scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.123:/etc/kubernetes/pki/kubelet/
```

### 6-4.分发 `kube-apiserver` 颁发给 `kubelet` 的 `client` 证书

分发到 `kubelet` 节点

```shell
scp /ssl/apiserver-kubelet-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

### 6-5.分发 `kube-apiserver` 的 `ca` 公钥给 `kubelet`

分发到 `node` 节点

```scp
scp /ssl/apiserver-ca.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 7.创建 `kubeconfig`

> [!NOTE]
> `kubelet` 是使用 `kubeconfig` 跟 `kube-apiserver` 建立连接的。
>   

> 在 `kubeconfig` 配置文件中会包含 `kubelet`  的客户端 `client` 证书信息以及身份信息。  

>   

> 由于已经部署了 `master` 高可用，所以设置集群参数的时候指定的参数：`--server` 需要指向 `vip` 地址。
>   

> 也就是前面创建的 `172.16.222.110`，并且端口为 `8443`。

设置集群参数

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.110:8443 \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials kubelet  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

设置上下文

```shell
kubectl config set-context kubelet \
--cluster=kubernetes \
--user=kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

## 8.启动服务

### 8-1.设置 `kube-apiserver` 的客户端证书参数

> [!NOTE]
> `kube-apiserver` 作为访问 `kubelet` 服务的客户端，需要提供 `kubelet` 的 `ca` 机构为其颁发的客户端证书。  

> 需要在每台 `master` 服务器上修改 `kube-apiserver` 的启动参数。

### 8-2.修改 `kube-apiserver` 参数

> [!NOTE]
> 添加 `kubelet` 为 `kube-apiserver` 颁发的客户端 `client` 证书。

```shell
--kubelet-certificate-authority=/etc/kubernetes/pki/kubelet/kubelet-ca.pem

--kubelet-client-certificate=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client.pem

--kubelet-client-key=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client-key.pem
```

### 8-3.创建 `kubelet` 配置文件

```shell
cat > /etc/kubernetes/config/kubelet.yaml <<EOF
apiVersion: "kubelet.config.k8s.io/v1beta1"
kind: "KubeletConfiguration"
enableServer: true
address: "172.16.222.231"
tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet-server.pem"
tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet-server-key.pem"
authentication:
  anonymous:
    enabled: false
  webhook:
     enabled: false
     cacheTTL: "2m"
  x509:
    clientCAFile: "/etc/kubernetes/pki/kubelet/kubelet-ca.pem"
authorization:
   mode: "Webhook"
   webhook:
     cacheAuthorizedTTL: "5m"
     cacheUnauthorizedTTL: "30s"
cgroupDriver: "systemd"
clusterDomain: "cluster.local"
clusterDNS:
- "10.68.0.1"
EOF
```

### 8-4.创建启动配置文件

```shell
cat > /etc/kubernetes/config/kubelet.conf <<EOF
KUBELET_OPTS="--network-plugin=cni \

--config=/etc/kubernetes/config/kubelet.yaml \

--alsologtostderr=true \

--logtostderr=false \

--log-dir=/var/log/kubernetes/kubelet/ \

--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig \

--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6 \

--v=2"
EOF
```

### 8-5.创建启动文件

```shell
cat > /usr/lib/systemd/system/kubelet.service <<'EOF'
[Unit]
Description=Kubernetes Kubelet Service
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet/
EnvironmentFile=-/etc/kubernetes/config/kubelet.conf
ExecStart=/usr/local/bin/kubelet $KUBELET_OPTS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

## 9.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kubelet
```

检查没有错误后，设置开机启动

```shell
systemctl enable kubelet
```

## 10.其他操作

停止服务

```shell
systemctl stop kubelet
```

查看服务

```shell
systemctl status kubelet --no-pager -l
```

查看进程错误

```shell
journalctl -l --no-pager  -u kubelet
```

删除进程日志

```shell
rm -rvf /var/log/journal/*
```

删除 `kubelet` 日志

```shell
rm -rvf /var/log/kubernetes/kubelet/*
```

重启或者重新部署报错，有可能需要删除已经生成的工作数据

```shell
rm -rvf /var/lib/kubelet/*
```

## 11.检测

到 `master` 服务器查看节点是否加入

```shell
kubectl get nodes
```

*显示*

```text
NAME         STATUS     ROLES    AGE     VERSION
k8s-node01   NotReady   <none>   4h38m   v1.23.9
```

可以看到 `node` 节点已经加入进来了，但是显示的 `NotReady` 是因为 `node` 节点还没部署网络插件。后面再部署网络插件。

***

## 至此。`node` 节点的 `kubelet` 部署成功。
