---
title: "Kubernetes Binary HA Cluster Deployment Series (9/18) — Deploying kube-scheduler"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:16:00Z
tags:
  - Kubernetes
description: "Issuing the kube-scheduler client certificate, creating scheduler.kubeconfig, and deploying kube-scheduler with leader election on all 3 master nodes."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `kube-scheduler`

***

# Deploying `kube-scheduler`

`kube-scheduler` connects to `kube-apiserver` and runs co-located on all three `master` hosts.

## 1. Generate the SSL Certificate

> [!NOTE]
> Both `CN` and `O` must be set to `system:kube-scheduler` to match Kubernetes' built-in RBAC bindings.

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

## 2. Distribute the Certificate

```shell
scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-scheduler-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 3. Create `scheduler.kubeconfig`

### 3.1 On `master-01` (`172.16.222.121`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.121:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

### 3.2 On `master-02` (`172.16.222.122`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

### 3.3 On `master-03` (`172.16.222.123`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-scheduler-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler \
--kubeconfig=/etc/kubernetes/kubeconfig/scheduler.kubeconfig
```

## 4. Configure and Start `kube-scheduler`

### 4.1 Create `/etc/kubernetes/config/scheduler.conf`

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

### 4.2 Create the `systemd` Service Unit

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

## 5. Start, Enable, and Verify

```shell
systemctl daemon-reload && \
systemctl start kube-scheduler && \
systemctl enable kube-scheduler
```

Check component status:

```shell
kubectl get cs
```

```text
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
etcd-1               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
scheduler            Healthy   ok
```

## 6. Useful Operations

```shell
systemctl stop kube-scheduler
systemctl status kube-scheduler
journalctl -l --no-pager -u kube-scheduler
```

***

`kube-scheduler` is now deployed and healthy.
