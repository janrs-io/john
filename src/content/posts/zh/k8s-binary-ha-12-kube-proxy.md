---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十二 - 部署kube-proxy"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:17:00Z
tags:
  - Kubernetes
  - Networking
description: "kube-proxy 主要作为 k8s 的网络代理与负载均衡，只负责定时通过 kube-apiserver 从 etcd 数据库获取 service 的信息来做创建网络代理。 同样的，kube-proxy 是需要访问 kube-apiserver 服务的，这就要求 kube-apiserver 需要为其..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kube-proxy`

***

# 部署 `kube-proxy`

`kube-proxy` 主要作为 `k8s` 的网络代理与负载均衡，只负责定时通过 `kube-apiserver` 从 `etcd` 数据库获取 `service`
的信息来做创建网络代理。
  

  

同样的，`kube-proxy` 是需要访问 `kube-apiserver` 服务的，这就要求 `kube-apiserver` 需要为其颁发客户端 `client` 证书。

## 1.创建 `ssl` 证书

### 1-1.创建 `csr` 请求文件

> [!NOTE]
> `CN` 参数表示用户名，必须设置为 `k8s` 中设定的 `system:kube-proxy`  

> `O` 参数表示用户组，必须设置为 `k8s` 中设定的 `system:kube-proxy`  

> `kube-proxy` 同样作为客户端，不需要设置 `hosts` 参数。

```shell
cat > /ssl/apiserver-kube-proxy-client-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "kube-proxy",
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
-profile=client apiserver-kube-proxy-client-csr.json | \
cfssljson -bare apiserver-kube-proxy-client && \
ls apiserver-kube-proxy-client* | \
grep apiserver-kube-proxy-client
```

## 2.分发证书

分发到 `kube-proxy` 节点

```shell
scp /ssl/apiserver-kube-proxy-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 3.创建 `kubeconfig`

> [!NOTE]
> `kube-proxy` 是使用 `kubeconfig` 跟 `kube-apiserver` 进行通信的。  

> `kubeconfig` 配置文件中会包含了 `kube-scheduler` 的客户端 `client` 证书信息以及身份信息。  

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
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
```

设置客户端认证参数

```shell
kubectl config set-credentials kube-proxy  \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
```

设置上下文

```shell
kubectl config set-context kube-proxy \
--cluster=kubernetes \
--user=kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
```

## 4.创建 `kube-proxy` 配置文件

> [!NOTE]
> 在部署 `kube-proxy` 的 `node` 服务器操作。

```shell
cat > /etc/kubernetes/config/kube-proxy.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: '172.16.222.231'
healthzBindAddress: '172.16.222.231:10256'
metricsBindAddress: '127.0.0.1:10249'
bindAddressHardFail: true
clientConnection:
  kubeconfig: /etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  qps: 5
clusterCIDR: 10.100.0.0/16
enableProfiling: false
mode: "ipvs"
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 30s
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
EOF
```

## 5.创建启动配置文件

```shell
cat > /etc/kubernetes/config/kube-proxy.conf <<EOF
KUBE_PROXY_OPTS="--alsologtostderr=true \

--logtostderr=false \

--config=/etc/kubernetes/config/kube-proxy.yaml \

--log-dir=/var/log/kubernetes/kube-proxy \

--v=2"
EOF
```

## 6.创建启动项

```shell
cat > /usr/lib/systemd/system/kube-proxy.service <<'EOF'
[Unit]
Description=Kubernetes Kube Proxy Service
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy/
EnvironmentFile=-/etc/kubernetes/config/kube-proxy.conf
ExecStart=/usr/local/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

## 7.启动服务

启动服务

```shell
systemctl daemon-reload && \
systemctl start kube-proxy
```

正常启动没有错误，设置开机启动

```shell
systemctl enable kube-proxy
```

## 8.验证服务

> [!NOTE]
> 执行查看进程运行情况。没有 `ERROR` 或者 `FAILED` 等错误正常启动就可以。

```shell
systemctl status kube-proxy --no-pager -l
```

## 9.其他操作

停止服务

```shell
systemctl stop kube-proxy
```

查看状态

```shell
systemctl status kube-proxy --no-pager -l
```

查看进程运行日志

```shell
journalctl -l --no-pager  -u kube-proxy
```

删除进程运行日志

```shell
rm -rvf /var/log/journal/*
```

***

## 至此。`kube-proxy` 组件部署成功。
