---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十五 - 部署kubelet-自签名证书方式"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
description: "自签名其实就是手动颁发证书的自动化。自签名颁发证书也是由 kubelet 的 ca 机构颁发的。 在 /etc/kubernetes/config/kubelet.yaml 配置文件中，不需要指定以下参数： kubelet 启动后会自动生成服务端证书存放在参数 --cert-dir 指定的文件夹位置。"
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kubelet`  

> 自签名 `server` 证书方式

***

# 部署 `kubelet`

## 自签名 `server` 证书方式

自签名其实就是手动颁发证书的自动化。自签名颁发证书也是由 `kubelet` 的 `ca` 机构颁发的。
  

在 `/etc/kubernetes/config/kubelet.yaml` 配置文件中，不需要指定以下参数：
  

*将其注释掉即可*

```yaml
#tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet-server.pem"
#tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet-server-key.pem"
```

`kubelet` 启动后会自动生成服务端证书存放在参数 `--cert-dir` 指定的文件夹位置。
  

  

同样的，由于服务端证书是自动生成的，`kube-apiserver` 也不需要指定以下参数来校验：
  

*将其删除即可*

```shell
--kubelet-certificate-authority=/etc/kubernetes/pki/kubelet/kubelet-ca.pem
```

## 重启

修改以上配置后重启 `kube-apiserver` 以及 `kubelet` 服务。
