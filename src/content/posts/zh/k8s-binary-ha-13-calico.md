---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十三 - 部署calico"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
  - Networking
description: "之前需要配置一下网络。具体查看官网说明。 下载 yaml 并执行部署 修改网段。修改 ippool 的网段为：10.100.0.0/16 官方自带的是 192.168.0.0/16 网段。这里使用的是自定义的。 部署成功，所有 pods 运行正常，且 node 节点状态由 NotReady 变成 Rea..."
---

> [!TIP]
> 二进制部署 `k8s` - `node` 节点部署 `calico`

***

# 部署 `calico`

> [!NOTE]
> 在 `node` 节点部署。

## 1.配置网络

部署 `calico`
之前需要配置一下网络。具体查看官网说明。
  

地址：(https://projectcalico.docs.tigera.io/maintenance/troubleshoot/troubleshooting#configure-networkmanager)

```shell
cat > /etc/NetworkManager/conf.d/calico.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
EOF
```

重启网络

```shell
systemctl restart NetworkManager
```

## 2.部署 `calico`

> [!NOTE]
> 官方不推荐手动二进制部署。推荐使用 `tigera-operator` 部署。
>   

> 我的是自建群，就按照官方推荐的方式部署。
>   

> 地址：(https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)
>   

> `yaml` 已经复制到我的博客，直接下载就行。

### 2-1.部署 `operator`

> [!NOTE]
> 镜像地址已经修改为阿里的。不然又是卡住不动。

下载 `yaml` 并执行部署

```shell
cd /etc/kubernetes/init_k8s_config/ && \
kubectl create -f /etc/kubernetes/init_k8s_config/calico-tigera-operator.yaml
```

查看

```shell
kubectl get pods -A
```

*显示*

```text
NAMESPACE          NAME                                       READY   STATUS    RESTARTS       AGE
tigera-operator    tigera-operator-6dcd98c8ff-f2rw4           1/1     Running   0              104m
```

### 2-2.部署 `custom-resources`

> [!NOTE]
> `custom-resources` 可以自定义。不过按照官方默认的就可以了。

下载 `yaml` 并部署

````shell
cd /etc/kubernetes/init_k8s_config/ && \
````

修改网段。修改 `ippool` 的网段为：`10.100.0.0/16`
  

*官方自带的是 `192.168.0.0/16` 网段。这里使用的是自定义的。*

```text
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

部署

```shell
kubectl create -f /etc/kubernetes/init_k8s_config/calico-custom-resources.yaml
```

查看

```shell
kubectl get pods -A
```

*显示*

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

## 2-3.检测 `node` 状态

部署成功，所有 `pods` 运行正常，且 `node` 节点状态由 `NotReady` 变成 `Ready` 状态。

```shell
kubectl get nodes
```

*显示*

```text
NAME         STATUS   ROLES    AGE    VERSION
k8s-node01   Ready    <none>   108m   v1.23.9
```

***

## 至此。网络插件 `calico` 部署成功。
