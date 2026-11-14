---
title: "Kubernetes Binary HA Cluster Deployment Series (6/18) — Deploying kube-apiserver"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:14:00Z
tags:
  - Kubernetes
description: "Generating apiserver CA, server, and etcd client certificates, distributing Kubernetes server binaries, and running kube-apiserver across 3 master nodes."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `kube-apiserver`

***

# Deploying `kube-apiserver`

## 1. Create Directories

> [!NOTE]
> Run on every `master` server.

```shell
# Certificate directories
mkdir -p /etc/kubernetes/pki/{apiserver/,kubelet/,aggregator/,service-account/,sign/,etcd/}

# Config, kubeconfig, and cluster init directories
mkdir -p /etc/kubernetes/{config/,kubeconfig/,init_k8s_config/}

# Default kubectl config directory
mkdir -p /root/.kube/

# Log directories
mkdir -p /var/log/kubernetes/{apiserver/,controller/,scheduler/}
```

## 2. Generate SSL Certificates

### 2.1 Create the `apiserver` CA Root Certificate

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

### 2.2 Create the `apiserver` Server Certificate

> [!NOTE]
> Include all `master` IPs, HA load-balancer IPs, the VIP, and the Kubernetes Service ClusterIP (`10.68.0.1`) in `hosts`.

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

### 2.3 Create the `etcd` Client Certificate for `kube-apiserver`

> [!NOTE]
> Signed by `etcd-ca` because `kube-apiserver` connects as a client to `etcd`.

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

## 3. Distribute Certificates to All `master` Nodes

```shell
# etcd CA
scp /ssl/etcd-ca.pem root@172.16.222.121:/etc/kubernetes/pki/etcd/ && \
scp /ssl/etcd-ca.pem root@172.16.222.122:/etc/kubernetes/pki/etcd/ && \
scp /ssl/etcd-ca.pem root@172.16.222.123:/etc/kubernetes/pki/etcd/

# apiserver CA
scp /ssl/apiserver-ca*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-ca*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-ca*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/

# etcd client cert for kube-apiserver
scp /ssl/etcd-apiserver-client*.pem root@172.16.222.121:/etc/kubernetes/pki/etcd/ && \
scp /ssl/etcd-apiserver-client*.pem root@172.16.222.122:/etc/kubernetes/pki/etcd/ && \
scp /ssl/etcd-apiserver-client*.pem root@172.16.222.123:/etc/kubernetes/pki/etcd/

# apiserver server cert
scp /ssl/apiserver-server*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-server*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-server*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 4. Download and Install Kubernetes Server Binaries

```shell
cd /tmp && \
wget https://dl.k8s.io/v1.23.9/kubernetes-server-linux-amd64.tar.gz

scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.121:/home/ && \
scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.122:/home/ && \
scp /tmp/kubernetes-server-linux-amd64.tar.gz root@172.16.222.123:/home/
```

Extract and copy binaries on each `master` node:

```shell
cd /home/ && \
tar -zxvf kubernetes-server-linux-amd64.tar.gz
cd /home/kubernetes/server/bin/ && \
cp kube-apiserver kube-controller-manager kubectl kube-scheduler /usr/local/bin/
```

## 5. Configure and Start `kube-apiserver`

### 5.1 Create `/etc/kubernetes/config/apiserver.conf`

> [!NOTE]
> Log verbosity `--v=6` produces verbose logs for initial debugging; lower it to `--v=2` once the cluster is stable.

#### On `master-01` (`172.16.222.121`)

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

#### On `master-02` (`172.16.222.122`)

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

#### On `master-03` (`172.16.222.123`)

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

### 5.2 Create the `systemd` Service Unit

> [!NOTE]
> Quote `'EOF'` so `$KUBE_APISERVER_OPTS` is preserved literally in the unit file. Identical on all 3 master nodes.

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

### 5.3 Start and Enable `kube-apiserver`

```shell
systemctl daemon-reload && \
systemctl start kube-apiserver && \
systemctl enable kube-apiserver
```

### 5.4 Useful Operations

```shell
systemctl stop kube-apiserver
systemctl status kube-apiserver
journalctl -l --no-pager -u kube-apiserver
rm -rvf /var/log/journal/*
```

***

`kube-apiserver` is now deployed across all three master nodes.
