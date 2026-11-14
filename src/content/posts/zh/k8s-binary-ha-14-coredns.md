---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十四 -部署coredns"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
  - Networking
description: "直接下载对应硬件架构的 coredns。 如果 wget 下载慢的手动下载再上传 检查是否有错误。直接打印日志文件查看 没有显示任何错误表示启动成功 这里给 node 打标签部署 查看 busybox 的 pod 检测同命名空间下，是否能发现服务。测试集群默认的 kubernetes 服务。"
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `coredns` 插件

***

# 部署 `coredns` 插件

> [!NOTE]
> 在 `node` 节点部署。

## 安装

直接下载对应硬件架构的 `coredns`。
  

*如果 `wget` 下载慢的手动下载再上传*

```shell
cd /home/ && \
wget https://github.com/coredns/coredns/releases/download/v1.10.0/coredns_1.10.0_linux_amd64.tgz && \
tar -zxvf coredns_1.10.0_linux_amd64.tgz && \
mv coredns /usr/local/bin/
```

## 1.创建目录

```shell
# 创建日志存放目录
mkdir -p /var/log/coredns/ && \

# 创建配置文件存放目录
mkdir -p /etc/coredns/
```

## 2.创建 `ssl` 证书

> [!NOTE]
> `coredns` 需要访问 `kube-apiserver` 获取数据。
>   

> 需要 `kube-apiserver` 的 `ca` 机构为其颁发客户端 `client` 证书。

## 3.创建 `csr` 请求文件

```shell
cat > /ssl/apiserver-coredns-client-csr.json <<EOF
{
    "CN": "coredns",
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
cfssl gencert \
-ca=apiserver-ca.pem \
-ca-key=apiserver-ca-key.pem \
-config=ca-config.json \
-profile=client apiserver-coredns-client-csr.json | \
cfssljson -bare apiserver-coredns-client && \
ls apiserver-coredns-client* | \
grep apiserver-coredns-client
```

分发证书
  

*分发到 `node` 节点*

```shell
scp /ssl/apiserver-coredns-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/ && \

scp /ssl/apiserver-ca.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 4.创建配置文件

> [!NOTE]
> 需要指定 `tls` 证书。也就是场面创建的 `client` 证书以及 `ca` 证书。
>   

> 官方文档地址：(https://coredns.io/plugins/kubernetes/)
>   

>   

> 在 `node` 节点操作。

```shell
cat > /etc/coredns/Corefile <<EOF
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        endpoint https://172.16.222.110:8443
        tls /etc/kubernetes/pki/apiserver/apiserver-coredns-client.pem /etc/kubernetes/pki/apiserver/apiserver-coredns-client-key.pem /etc/kubernetes/pki/apiserver/apiserver-ca.pem
    }
    health
    errors
    log
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    reload 10s
    loadbalance
    loop
}
EOF
```

## 5.创建集群角色并绑定

> [!NOTE]
> 除了需要创建客户端证书外，还需要创建 `RBAC` 授权。因为前面部署的 `kube-apiserver` 开启了 `RBAC` 鉴权。
>   

> 这里的集群用户跟上面创建客户端证书指定的用户一致：`coredns`。
>   

>   

> 在 `master` 节点操作。

```shell
cat > /etc/kubernetes/init_k8s_config/create-coredns-rbac-permission.yaml <<EOF
---
# 创建集群角色
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: coredns
rules:
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get","watch","list"]

- apiGroups: [""]
  resources: ["nodes",pods","namespaces","services"]
  verbs: ["get", "watch", "list"]
---
# 绑定用户
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: coredns-access-resources
subjects:
- kind: User
  name: coredns
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: coredns
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/create-coredns-rbac-permission.yaml
```

## 6.创建启动项

> [!NOTE]
> 在 `node` 节点操作。

```shell
cat > /usr/lib/systemd/system/coredns.service << EOF
[Unit]
Description=CoreDNS DNS Server
Documentation=https://coredns.io/
After=network.target
StartLimitBurst=1
StartLimitIntervalSec=15s

[Service]
# Type设置为notify时，服务会不断重启
Type=simple
User=root
# 指定运行端口和读取的配置文件
ExecStart=/usr/local/bin/coredns -conf /etc/coredns/Corefile
# append类型可以在原有文件末尾继续追加内容，而file类型则是重新打开一个新文件
# 两者的区别类似于 echo >> 和 echo >
StandardOutput=append:/var/log/coredns/coredns.log
StandardError=append:/var/log/coredns/coredns_error.log
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

## 7.启动服务

> [!NOTE]
> 在 `node` 节点操作。

启动服务

```shell
systemctl daemon-reload && \
systemctl start coredns
```

查看运行状态

```shell
systemctl status coredns --no-pager -l
```

检查是否有错误。直接打印日志文件查看
  

*没有显示任何错误表示启动成功*

```shell
cat /var/log/coredns/coredns.log
```

*显示如下，没有任何错误*

```text
.:53
[INFO] plugin/reload: Running configuration SHA512 = 75a7e545d2e00af59a3f1984d819f3b35db009002c603603942190c2fe71d13d8a0a86e605e4d42ad2027f8fd9b21ef4a765811b68fde1a0ffc89ce752387c51
CoreDNS-1.10.0
linux/amd64, go1.19.1, 596a9f9
```

设置开机启动

```shell
systemctl enable coredns
```

## 8.测试 `coredns` 是否正常工作

> [!NOTE]
> 在 `master` 节点操作。

### 8-1.创建 `busybox` 进行测试

> [!NOTE]
> `busybox` 不要用 `1.28.4` 的版本，有 `bug`。就算是 `coredns` 正常运行，也无法正常测试。
>   

> 用 `1.28.3` 版本的进行测试。并且创建一个 `dev` 命名空间进行测试。

#### 8-1-1.创建 `dev` 命名空间

```shell
kubectl create ns dev
```

#### 8-1-2.打标签

这里给 `node` 打标签部署

```shell
kubectl label node k8s-node01 dev-node=true
```

#### 8-1-3.部署 `busybox`

```shell
cat > /home/busybox.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: dev
spec:
  selector:
    matchLabels:
      app: busybox
  replicas: 1
  template:
    metadata:
      labels:
        app: busybox
    spec:
      restartPolicy: Always
      nodeSelector:
        dev-node: 'true'
      containers:
      - name: busybox
        command:
        - sleep
        - "3600"
        image: busybox:1.28.3
EOF
kubectl apply -f /home/busybox.yaml
```

### 9.测试

查看 `busybox` 的 `pod`

```shell
kubectl get pods -n dev
```

*显示*

```text
NAME                       READY   STATUS    RESTARTS   AGE
busybox-5b75bcbcb8-pqrqr   1/1     Running   0          118s
```

#### 9-1.查看 `/etc/resolv.conf`

```shell
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev cat /etc/resolv.conf
```

*显示*

```text
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
nameserver 10.68.0.1
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

#### 9-2.检测是否能访问外网

访问百度测试

```shell
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup www.baidu.com
```

*显示如下表示有正常解析*

```text
Server:    10.68.0.1
Address 1: 10.68.0.1

Name:      www.baidu.com
Address 1: 14.215.177.38
Address 2: 14.215.177.39
```

#### 9-3.检测访问其他服务

检测同命名空间下，是否能发现服务。测试集群默认的 `kubernetes` 服务。

```shell
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup kubernetes.default
```

*显示如下表示有正常解析*

```text
Server:    10.68.0.1
Address 1: 10.68.0.1

Name:      kubernetes.default
Address 1: 10.68.0.1
```

#### 9-4.检测服务发现

> [!NOTE]
> 部署 `nginx` 并且创建 `nginx` 服务，再用 `busybox` 检测是否能自动发现服务。
>   

> `nginx` 部署在跟 `busybox` 同个命名空间 `dev` 下。

```shell
cat > /home/nginx.yaml <<EOF
---
# 部署 pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.cn-shenzhen.aliyuncs.com/yjy_k8s/nginx:v1.23.1
        ports:
        - containerPort: 80
---
# 创建 ngixn 的 service
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: dev
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
EOF
kubectl apply -f /home/nginx.yaml
```

查看 `nginx` 的 `svc`

```shell
kubectl get svc -n dev
```

*显示*

```text
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   10.68.141.101   <none>        80/TCP    2m48s
```

测试访问

```shell
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup nginx-svc
```

*显示*

```text
Server:    10.68.0.1
Address 1: 10.68.0.1

Name:      nginx-svc
Address 1: 10.68.141.101
```

### 10.检测通过

以上测试全部都可以访问到服务，能够正常显示对应的 `ip`
  

表示 `coredns` 能够访问外网，并且能够进行服务发现，能够访问其他服务。即正常工作。
***

## 至此。二进制域名解析插件 `coredns` 部署成功。
