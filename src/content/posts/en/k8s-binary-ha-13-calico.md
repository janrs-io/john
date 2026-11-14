---
title: "Kubernetes Binary HA Cluster Deployment Series (13/18) — Deploying Calico CNI"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
  - Networking
description: "Configuring NetworkManager for Calico, deploying Tigera Operator and custom-resources with Pod CIDR 10.100.0.0/16, and bringing worker nodes to Ready."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `calico` CNI

***

# Deploying `calico`

> [!NOTE]
> Configure NetworkManager on the worker `node` before deploying Calico manifests.

## 1. Configure NetworkManager

Prevent NetworkManager from interfering with Calico virtual interfaces (see [Calico Troubleshooting Docs](https://projectcalico.docs.tigera.io/maintenance/troubleshoot/troubleshooting#configure-networkmanager)):

```shell
cat > /etc/NetworkManager/conf.d/calico.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF

systemctl restart NetworkManager
```

## 2. Deploy Calico via `tigera-operator`

> [!NOTE]
> Calico officially recommends deploying via `tigera-operator` (Pods) rather than raw binaries. Reference: [Calico Self-Managed On-Premises Docs](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises).

### 2.1 Deploy the Operator

```shell
cd /etc/kubernetes/init_k8s_config/ && \
kubectl create -f /etc/kubernetes/init_k8s_config/calico-tigera-operator.yaml
```

Verify the operator Pod is running:

```shell
kubectl get pods -A
```

```text
NAMESPACE          NAME                                       READY   STATUS    RESTARTS       AGE
tigera-operator    tigera-operator-6dcd98c8ff-f2rw4           1/1     Running   0              104m
```

### 2.2 Deploy `custom-resources`

Update `ipPools[0].cidr` in `calico-custom-resources.yaml` to match our cluster Pod CIDR (`10.100.0.0/16`, overriding Calico's default `192.168.0.0/16`):

```yaml
...
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.100.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
...
```

Apply the custom resources manifest:

```shell
kubectl create -f /etc/kubernetes/init_k8s_config/calico-custom-resources.yaml
```

Verify all Calico Pods are running:

```shell
kubectl get pods -A
```

```text
NAMESPACE          NAME                                       READY   STATUS    RESTARTS       AGE
calico-apiserver   calico-apiserver-69c54b8687-4khms          1/1     Running   1 (102m ago)   104m
calico-apiserver   calico-apiserver-69c54b8687-7mnln          1/1     Running   1 (102m ago)   104m
calico-system      calico-kube-controllers-688968c9b6-kchvq   1/1     Running   0              104m
calico-system      calico-node-m7zxb                          1/1     Running   0              103m
calico-system      calico-typha-7bd99d8c79-vj4lw              1/1     Running   0              104m
calico-system      csi-node-driver-v4g95                      2/2     Running   0              103m
tigera-operator    tigera-operator-6dcd98c8ff-f2rw4           1/1     Running   0              104m
```

## 3. Verify Node Status

Once Calico Pods are healthy, `k8s-node01` transitions from `NotReady` to `Ready`:

```shell
kubectl get nodes
```

```text
NAME         STATUS   ROLES    AGE    VERSION
k8s-node01   Ready    <none>   108m   v1.23.9
```

***

Calico CNI is now deployed and active.
