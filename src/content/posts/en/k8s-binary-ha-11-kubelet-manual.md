---
title: "Kubernetes Binary HA Cluster Deployment Series (11/18) — Deploying kubelet (Manual Certificates)"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:17:00Z
tags:
  - Kubernetes
description: "Installing Docker on worker nodes, issuing manual kubelet CA/server/client certificates, creating kubelet.kubeconfig, and joining the cluster."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `kubelet` (Manual Certificate Mode)

***

# Deploying `kubelet`

> [!NOTE]
> Complete [Server Initialization](../k8s-binary-ha-05-server-init/) on the worker node before proceeding.

Because `kubelet` exposes its own HTTPS API (called by `kube-apiserver`) and connects back to `kube-apiserver` as a client, manual certificate provisioning involves:
- A `kubelet` CA root certificate
- A `kubelet` server certificate
- A client certificate issued by `kubelet`'s CA for `kube-apiserver`
- A client certificate issued by `apiserver`'s CA for `kubelet` (`system:node:<nodeName>`)

## 1. Create Directories

```shell
mkdir -p /etc/kubernetes/pki/{kubelet/,apiserver/}
mkdir -p /var/log/kubernetes/{kubelet/,kube-proxy/}
mkdir -p /etc/kubernetes/{kubeconfig/,config/}
mkdir -p /var/lib/{kubelet/,kube-proxy/}
```

## 2. Install Docker & Pull `pause` Image

```shell
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf install docker-ce -y
```

Configure `systemd` as the cgroup driver:

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

systemctl daemon-reload && \
systemctl enable docker && \
systemctl enable containerd && \
systemctl start docker

docker pull registry.aliyuncs.com/google_containers/pause:3.6
```

## 3. Copy Node Binaries from `master-01`

```shell
cd /home/kubernetes/server/bin/ && \
scp kubectl kubelet kube-proxy root@172.16.222.231:/usr/local/bin/
```

## 4. Generate SSL Certificates

### 4.1 `kubelet` CA Root Certificate

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
ls kubelet-ca* | grep kubelet-ca
```

### 4.2 `kubelet` Server Certificate (`172.16.222.231`)

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
ls kubelet-server* | grep kubelet-server
```

### 4.3 Client Certificate for `kube-apiserver` -> `kubelet`

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
ls kubelet-apiserver-client* | grep kubelet-apiserver-client
```

### 4.4 Client Certificate for `kubelet` -> `kube-apiserver`

> [!NOTE]
> `kubelet` authenticates with `CN=system:node:<nodeName>` and `O=system:nodes` for Node authorization (see [Certificates](https://kubernetes.io/docs/setup/best-practices/certificates/) and [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/)).

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
ls apiserver-kubelet-client* | grep apiserver-kubelet-client
```

## 5. Distribute Certificates

```shell
# kubelet CA to all masters and node
scp /ssl/kubelet-ca.pem root@172.16.222.121:/etc/kubernetes/pki/kubelet/ && \
scp /ssl/kubelet-ca.pem root@172.16.222.122:/etc/kubernetes/pki/kubelet/ && \
scp /ssl/kubelet-ca.pem root@172.16.222.123:/etc/kubernetes/pki/kubelet/ && \
scp /ssl/kubelet-ca*.pem root@172.16.222.231:/etc/kubernetes/pki/kubelet/

# kubelet server cert to node
scp /ssl/kubelet-server*.pem root@172.16.222.231:/etc/kubernetes/pki/kubelet/

# kubelet client cert for kube-apiserver to masters
scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.121:/etc/kubernetes/pki/kubelet/ && \
scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.122:/etc/kubernetes/pki/kubelet/ && \
scp /ssl/kubelet-apiserver-client*.pem root@172.16.222.123:/etc/kubernetes/pki/kubelet/

# apiserver client cert and CA to node
scp /ssl/apiserver-kubelet-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
scp /ssl/apiserver-ca.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 6. Create `kubelet.kubeconfig` (Pointing to HA VIP)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.110:8443 \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig

kubectl config set-credentials kubelet \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kubelet-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig

kubectl config set-context kubelet \
--cluster=kubernetes \
--user=kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig

kubectl config use-context kubelet \
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

## 7. Update `kube-apiserver` Flags & Start `kubelet`

Add these flags to `/etc/kubernetes/config/apiserver.conf` on all three `master` hosts and restart `kube-apiserver`:

```shell
--kubelet-certificate-authority=/etc/kubernetes/pki/kubelet/kubelet-ca.pem
--kubelet-client-certificate=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client.pem
--kubelet-client-key=/etc/kubernetes/pki/kubelet/kubelet-apiserver-client-key.pem
```

Create `/etc/kubernetes/config/kubelet.yaml` on `node`:

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

Create `/etc/kubernetes/config/kubelet.conf` and `/usr/lib/systemd/system/kubelet.service`:

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

systemctl daemon-reload && \
systemctl start kubelet && \
systemctl enable kubelet
```

## 8. Verify Node Registration

On a `master` node:

```shell
kubectl get nodes
```

```text
NAME         STATUS     ROLES    AGE     VERSION
k8s-node01   NotReady   <none>   4h38m   v1.23.9
```

`NotReady` is expected until we deploy the CNI network plugin (Calico) in the upcoming steps.

***
