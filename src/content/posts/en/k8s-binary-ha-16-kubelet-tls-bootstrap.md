---
title: "Kubernetes Binary HA Cluster Deployment Series (16/18) — kubelet TLS Bootstrapping"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:19:00Z
tags:
  - Kubernetes
  - Security
description: "Configuring kubelet TLS Bootstrapping with auth-token.csv, bootstrap.kubeconfig, CSR approval, auto-approval, and automatic client/server certificate rotation."
---

> [!TIP]
> Kubernetes Binary Deployment — `kubelet` TLS Bootstrapping

***

# 1. `kubelet` TLS Bootstrapping

> [!NOTE]
> **Key distinction:** **TLS Bootstrapping** automates issuing and rotating `kubelet`'s **client** certificates (signed by `kube-apiserver`'s CA), whereas manual and self-signed modes in Parts 11 and 15 govern `kubelet`'s **server** certificate.

Official references:
- [Bootstrap Tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [TLS Bootstrapping](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
- [Static Token File](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file)

Manually creating `system:node:<nodeName>` client certs and `kubeconfig` files for hundreds of nodes does not scale. TLS Bootstrapping lets `kubelet` present a bootstrap token on first startup and automatically obtain a signed client certificate from the control plane.

### 1.1 Stop Control-Plane and Node Services

On all `master` nodes:

```shell
systemctl stop kube-apiserver && \
systemctl stop kube-controller-manager && \
systemctl stop kube-scheduler
```

On the worker `node`:

```shell
systemctl stop kubelet && \
systemctl stop kube-proxy
```

### 1.2 Remove the Old `kubelet.kubeconfig`

If `/etc/kubernetes/kubeconfig/kubelet.kubeconfig` already exists, `kubelet` will reuse it instead of bootstrapping a new one:

```shell
rm -rvf /etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

### 1.3 Create `auth-token.csv`

Generate a 128-bit random token and create `/etc/kubernetes/config/auth-token.csv` (format: `token,user,uid,"group1,group2"`):

```shell
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
# Example output: dafe33bc8fcdae7f9f16df53a95199fa

cat > /etc/kubernetes/config/auth-token.csv <<EOF
dafe33bc8fcdae7f9f16df53a95199fa,kubelet-bootstrap,10001,system:bootstrappers
EOF
```

Distribute `auth-token.csv` to all `master` nodes:

```shell
scp /etc/kubernetes/config/auth-token.csv root@172.16.222.121:/etc/kubernetes/config/ && \
scp /etc/kubernetes/config/auth-token.csv root@172.16.222.122:/etc/kubernetes/config/ && \
scp /etc/kubernetes/config/auth-token.csv root@172.16.222.123:/etc/kubernetes/config/
```

### 1.4 Enable Static Token Auth on `kube-apiserver`

Add to `/etc/kubernetes/config/apiserver.conf` on all masters:

```shell
--token-auth-file=/etc/kubernetes/config/auth-token.csv
```

### 1.5 Verify Signing Flags on `kube-controller-manager`

`kube-controller-manager` signs the CSRs using the flags we configured in Part 8:

```shell
--cluster-signing-cert-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem
--cluster-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem
--controllers=*,bootstrapsigner,tokencleaner
--cluster-signing-duration=8760h0m0s
```

### 1.6 Allow `system:bootstrappers` to Create CSRs

```shell
cat > /etc/kubernetes/init_k8s_config/allow-kubelet-create-client-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-kubelet-create-client-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-kubelet-create-client-csr.yaml
```

### 1.7 Create `bootstrap.kubeconfig` on the Worker Node

Use the exact same token from `auth-token.csv`:

```shell
kubectl config set-cluster bootstrap \
--server=https://172.16.222.110:8443 \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig

kubectl config set-credentials kubelet-bootstrap \
--token=dafe33bc8fcdae7f9f16df53a95199fa \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig

kubectl config set-context bootstrap --user=kubelet-bootstrap \
--cluster=bootstrap \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig

kubectl config use-context bootstrap \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
```

### 1.8 Update `kubelet.conf` Flags

Add to `/etc/kubernetes/config/kubelet.conf`:

```shell
--bootstrap-kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig
--cert-dir=/etc/kubernetes/pki/kubelet/
```

### 1.9 Start Services & Approve the CSR

Start control-plane services first, then `kubelet` and `kube-proxy` on the worker node:

```shell
# On masters
systemctl start kube-apiserver && \
systemctl start kube-controller-manager && \
systemctl start kube-scheduler

# On worker node
systemctl start kubelet && \
systemctl start kube-proxy
```

`kubelet` logs `"Waiting for client certificate to be issued"`. Inspect and approve the pending CSR on a `master`:

```shell
kubectl get csr
```

```text
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-RGFX9jfBr_SP-0-yH5Dxl3deBooS3qaqmsH2vN9TGE0   3m1s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

```shell
kubectl certificate approve node-csr-RGFX9jfBr_SP-0-yH5Dxl3deBooS3qaqmsH2vN9TGE0
```

Once approved, `/etc/kubernetes/kubeconfig/kubelet.kubeconfig` is auto-generated pointing to `/etc/kubernetes/pki/kubelet/kubelet-client-current.pem`, and `kubectl get nodes` shows `Ready` with Pods scheduled normally.

### 1.10 Auto-Approval & Automatic Certificate Rotation

#### Auto-Approve Initial Client CSRs

```shell
cat > /etc/kubernetes/init_k8s_config/allow-auto-approve-kubelet-client-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-auto-approve-kubelet-client-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-auto-approve-kubelet-client-csr.yaml
```

#### Auto-Approve Client Certificate Renewals

```shell
cat > /etc/kubernetes/init_k8s_config/allow-auto-renewal-kubelet-client-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-kubelet-client-csr
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-auto-renewal-kubelet-client-csr.yaml
```

Enable client cert rotation in `/etc/kubernetes/config/kubelet.yaml`:

```yaml
rotateCertificates: true
```

#### Auto-Rotate `kubelet` Server Certificates

> [!NOTE]
> Unlike `client` certs, `kubelet` **server** CSRs are not auto-approved out of the box—only rotation is supported via `RotateKubeletServerCertificate` (see [Kubernetes Discuss](https://discuss.kubernetes.io/t/how-to-enable-auto-csr-approval-for-kubelet-server-certificates/4495/2) and [kubernetes/kubernetes#73356](https://github.com/kubernetes/kubernetes/issues/73356)).

Enable the feature gate on `kube-controller-manager` and `kubelet`:

```yaml
# kube-controller-manager flag:
# --feature-gates=RotateKubeletServerCertificate=true

# /etc/kubernetes/config/kubelet.yaml:
serverTLSBootstrap: true
featureGates:
  RotateKubeletServerCertificate: true
```

Create and bind the `ClusterRole` allowing `system:bootstrappers` to submit `selfnodeserver` CSRs:

```shell
cat > /etc/kubernetes/init_k8s_config/allow-create-kubelet-server-csr.yaml <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests"]
  verbs: ["create","list","get","watch"]
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-create-kubelet-server-csr.yaml

cat > /etc/kubernetes/init_k8s_config/bind-create-kubelet-server-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-create-kubelet-server-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/bind-create-kubelet-server-csr.yaml
```

***

# 2. Summary

- **Bootstrap Auth**: `auth-token.csv` (or Bootstrap Tokens) authenticates the initial connection.
- **CSR Permission**: Without binding `system:node-bootstrapper` to `system:bootstrappers`, `kubelet` passes token auth but fails immediately when submitting its CSR.
- **Manual / Self-Signed vs. TLS Bootstrap**: Manual and self-signed modes control `kubelet`'s `server` certificate; TLS Bootstrapping manages `kubelet`'s `client` certificate (and optionally rotates `server` certificates via `serverTLSBootstrap: true`).

***
