---
title: "Kubernetes Binary HA Cluster Deployment Series (7/18) — Configuring kubectl"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:15:00Z
tags:
  - Kubernetes
description: "Issuing an admin client certificate for kubectl, generating admin.kubeconfig across all 3 masters, verifying cluster access, and granting kubelet API permissions."
---

> [!TIP]
> Kubernetes Binary Deployment — Configuring `kubectl`

***

# Configuring `kubectl`

`kubectl` connects to `kube-apiserver` as an administrative client, so it needs a client certificate signed by the `apiserver` CA.

## 1. Generate the Client SSL Certificate

### 1.1 Create the CSR and Sign the Certificate

> [!NOTE]
> - `CN` (`admin`) sets the Kubernetes username (avoiding over-privileged `cluster-admin` naming if not needed).
> - `O` (`system:masters`) sets the built-in Kubernetes superuser group.
> - Client certificates do not require a `hosts` list.

```shell
cat > /ssl/apiserver-admin-client-csr.json <<EOF
{
    "CN": "admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "system:masters",
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
-profile=client apiserver-admin-client-csr.json | \
cfssljson -bare apiserver-admin-client && \
ls apiserver-admin-client* | \
grep apiserver-admin-client
```

### 1.2 Distribute the Certificate to All `master` Nodes

```shell
scp /ssl/apiserver-admin-client*.pem root@172.16.222.121:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-admin-client*.pem root@172.16.222.122:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-admin-client*.pem root@172.16.222.123:/etc/kubernetes/pki/apiserver/
```

## 2. Create `admin.kubeconfig`

> [!NOTE]
> Create `admin.kubeconfig` under `/etc/kubernetes/kubeconfig/` on each `master` host pointing to its local `kube-apiserver`, then copy it to `/root/.kube/config`.

### 2.1 On `master-01` (`172.16.222.121`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.121:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

### 2.2 On `master-02` (`172.16.222.122`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.122:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

### 2.3 On `master-03` (`172.16.222.123`)

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.123:6443 \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-credentials admin \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-admin-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-admin-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kubeconfig/admin.kubeconfig

cp /etc/kubernetes/kubeconfig/admin.kubeconfig /root/.kube/config
```

## 3. Verify Cluster Access

Run on each `master` host:

```shell
kubectl cluster-info
```

```text
Kubernetes control plane is running at https://172.16.222.122:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```shell
kubectl get all --all-namespaces
```

```text
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.68.0.1    <none>        443/TCP   120m
```

## 4. Grant `kube-apiserver` Access to `kubelet` APIs

> [!NOTE]
> When running `kubectl logs` or `kubectl exec`, `kube-apiserver` proxies requests to `kubelet`. Bind the built-in `system:kubelet-api-admin` `ClusterRole` to the `kubernetes` user.

```shell
cat > /etc/kubernetes/init_k8s_config/allow-kubectl-access-kube-apiserver-kubelet-apis.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-kubectl-access-kube-apiserver-kubelet-apis
subjects:
- kind: User
  name: kubernetes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:kubelet-api-admin
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-kubectl-access-kube-apiserver-kubelet-apis.yaml
```

***
