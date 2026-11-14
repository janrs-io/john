---
title: "Kubernetes Binary HA Cluster Deployment Series (8/18) — Deploying kube-controller-manager"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:15:00Z
tags:
  - Kubernetes
description: "Generating client certs, binding RBAC roles, creating controller.kubeconfig, and deploying kube-controller-manager with leader election across 3 masters."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `kube-controller-manager`

***

# Deploying `kube-controller-manager`

`kube-controller-manager` communicates with `kube-apiserver` as a control-plane client and runs co-located on all three `master` hosts.

## 1. Generate the SSL Certificate

> [!NOTE]
> Both `CN` and `O` must be set to `system:kube-controller-manager`, matching Kubernetes' built-in RBAC bindings.

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

Distribute the certificate to all `master` nodes:

```shell
scp /ssl/apiserver-controller-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-controller-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-controller-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 2. Bind `cluster-admin` Role (Optional / Lab Convenience)

> [!NOTE]
> If `kube-controller-manager` needs permissions across namespaces beyond default bindings, you can bind `cluster-admin` here (or scope down permissions for stricter production hardening).

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

## 3. Create `controller.kubeconfig`

### 3.1 On `master-01` (`172.16.222.121`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.121:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

### 3.2 On `master-02` (`172.16.222.122`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

### 3.3 On `master-03` (`172.16.222.123`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-controller-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-controller-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig

kubectl config use-context system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/kubeconfig/controller.kubeconfig
```

## 4. Configure and Start `kube-controller-manager`

### 4.1 Create `/etc/kubernetes/config/controller.conf`

> [!NOTE]
> `--service-cluster-ip-range` must match `kube-apiserver` (`10.68.0.1/16`). Lower `--v=4` to `--v=2` once verified.

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

### 4.2 Create the `systemd` Service Unit

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

### 4.3 Start and Enable the Service

```shell
systemctl daemon-reload && \
systemctl start kube-controller-manager && \
systemctl enable kube-controller-manager
```

## 5. Verify Status

```shell
kubectl get cs
```

> [!NOTE]
> `controller-manager` now reports `Healthy`. `scheduler` will remain `Unhealthy` until deployed in the next step.

```text
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
controller-manager   Healthy     ok
etcd-0               Healthy     {"health":"true","reason":""}
etcd-2               Healthy     {"health":"true","reason":""}
etcd-1               Healthy     {"health":"true","reason":""}
```

## 6. Useful Operations

```shell
systemctl stop kube-controller-manager
systemctl status kube-controller-manager
journalctl -l --no-pager -u kube-controller-manager
```

***

`kube-controller-manager` is now deployed.
