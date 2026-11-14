---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十八 -最后总结"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:19:00Z
tags:
  - Kubernetes
description: "这一通二进制部署撸下来，其实可以看出，ssl 证书并不复杂。 ssl 的基本概念就是 ca 签发机构和单向认证/双向认证。 在 k8s 中，难的不是 ssl 证书。难的是了解整个 k8s 的运行机制。 k8s 除了需要 ssl 证书认证之外，还创建了一套鉴权机制。常用的 RBAC 鉴权通过在 ssl 证..."
---

> [!TIP]
> 二进制部署 `k8s` - 最后总结

***

# 最后总结

> [!NOTE]
> 除了聚合层 `Aggregator` 没有部署外，一套高可用的，开启 `node` 鉴权跟 `RBAC` 鉴权的 `k8s` 集群就搭建起来了。

## 关于 `ssl` 证书

这一通二进制部署撸下来，其实可以看出，`ssl` 证书并不复杂。
  

`ssl` 的基本概念就是 `ca` 签发机构和单向认证/双向认证。

> - 只要是作为一个独立的服务对外提供访问，就可以自己拥有一个 `ca` 签发机构。
> - 只要有自身要开启 `ssl` 认证就用自己的 `ca` 机构为自己签发 `server` 证书。
> - 只需要有客户端需要访问自身的服务并且要认证身份，就用自己的 `ca` 机构为其颁发 `client` 证书，并且指定 `CN` 用户名和 `O`
    > 用户组/组织。
> - 只要自身既要作为客户端又要作为服务端互相访问，就用自己的 `ca` 机构签发 `peer` 对等证书。`k8s` 只有 `etcd` 需要用到该证书。

在 `k8s` 中，难的不是 `ssl` 证书。难的是了解整个 `k8s` 的运行机制。
  

`k8s` 除了需要 `ssl` 证书认证之外，还创建了一套鉴权机制。常用的 `RBAC` 鉴权通过在 `ssl` 证书中指定 `CN` 用户名以及 `O`
用户组关联到 `RBAC` 的用户。
  

  

通过二进制部署就可以很好的学习 `k8s` 的运行机制。

## 深入理解 `kubelet` 证书

详细的深入理解总结不写了。通过前面的部署就可以明白证书是怎么一回事了。写文档也是很费劲。头发要多掉好几根。
  

需要明白的一个重要的地方就是，`kubelet` 有三种认证方式：
> - 手动部署证书
> - 自签名部署证书
> - `TLS Bootstrap` 自动引导签发证书。包括 `kubelet` 的 `client` 以及 `server` 证书
> - `kube-apiserver` 还可以对 `kubelet` 开启自动审核以及自动轮转过期证书

## 关于 `kube-apiserver` 以及 `HA`

`k8s` 对于 `kube-apiserver` 没有提供高可用的方式，可以通过 `nginx` + `kube-apiserver` 部署高可用。
  

  

需要注意的是：在签发 `kube-apiserver` 的 `server` 证书的时候，需要把 `HA` 服务器的 `ip` 地址以及 `keepalived` 的 `vip`
地址写进去。
  

否则 `HA` 入口以及 `vip` 入口都没法使用。提示未授权。

## 关于 `RBAC` 鉴权

常用的 `RBAC` 鉴权是需要跟 `ssl` 证书关联的。当客户端访问 `kube-apiserver` 的时候，会使用 `ssl` 中的 `CN` 参数以及 `O` 参数。
  

`CN` 参数用来当作 `RBAC` 的用户名，`O` 参数用来当作 `RBAC` 的用户组。
  

  

`k8s` 有默认的一些重要的集群角色，常用的比如：
> - `kube-controller-manager` 使用的 `system:kube-controller-manager` 集群角色
> - `kubelet` 加入集群使用的 `system:node:<nodeName>` 角色以及 `system:nodes` 角色组
> - `kube-proxy` 使用的 `kube-proxier` 集群角色

在创建 `ssl` 的时候指定了 `CN` 用户名以及 `O` 用户组后，还需要根据实际情况创建集群色或者普通角色，再将角色绑定到 `CN`
指定的用户名以及 `O` 指定的用户组。
  

这样一个客户端用户才有权限操作到 `k8s` 中的资源。

## 关于 `kubeconfig`

`kubeconfig` 是 `k8s` 创建的一种客户端访问 `kube-apiserver` 的方式。
  

在 `kubeconfig` 需要指定集群信息，客户端 `ssl` 认证信息，客户端用户名称信息。
  

还可用通过 `kubeconfig` 切换上下文使用不同的客户端用户访问 `kube-apiserver`。

## 关于使用哪种 `kubelet` 证书颁发方式

直接使用 `TLS Bootstrap` 自动引导证书即可，包括开启 `kubelet` 的服务端 `server` 证书自动引导颁发。
  

并且开启自动审核以及自动轮换。

## 最后关于 `kube-controller-manager` 与 `TLS Bootstrap`

在前面到 `ssl` 证书简介提到的，一堆的关于 `sign` 的参数，其实就是跟 `TLS Bootstrap` 相关的。
  

具体的就是跟 `TLS Bootstrap` 自动颁发 `kubelet` 的 `client` 以及 `server` 证书相关。
  

  

目前不折腾验证对应的参数以及证书是如何颁发与校验的，真的非常的折腾的。
  

主要验证是否可以由同一个 `ca` 机构颁发的。后面有空再折腾了。

***
