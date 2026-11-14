---
title: "【实战加详解】二进制部署k8s高可用集群教程系列一 - 前言"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T03:01:00Z
tags:
  - Kubernetes
  - DevOps
description: "超详细的二进制部署 k8s 教程文章。使用灵活强大的 calico 作为网络插件。 - 3 master 集群。使用 nignx + keepalived 作为高可用 - 每个 node 节点的 kubelet 以及 kube-proxy - 使用 calico 作为网络插件。calico 官方推荐使用..."
---

> [!TIP]
> 二进制部署 `k8s` 高可用集群

***

超详细的二进制部署 `k8s` 教程文章。使用灵活强大的 `calico` 作为网络插件。
  

  

部署的组件包含如下：

- 3 `etcd` 集群
- 3 `master` 集群。使用 `nignx` + `keepalived` 作为高可用
- 每个 `node` 节点的 `kubelet` 以及 `kube-proxy`
- 使用 `calico` 作为网络插件。`calico` 官方推荐使用 `pod` 部署
- `coredns` 域名解析插件。也是二进制部署

除了以上二进制部署之外，还深入详细的介绍了证书的使用以及 `TLS Bootstrap` 机制。
  

该教程区别于网上其他文章简单的使用的 `apiserver` 的 `ca` 证书一把梭哈的教程以及直接使用 `TLS Bootstrap` 的教程。
  

在该教程中，除了聚合层以及 `sign/client` 的参数外，详细的介绍了 `ca` 机构的使用，这也是让新手学习 `k8s` 及其头疼的地方。以及通过循环渐进和对比的方式介绍 `TLS Bootstrap` 机制的使用。
  

  

整个教程中，示例代码已经全部写好。只需要修改一下 `ip` 地址然后复制黏贴就可以部署出一套集群高可用的二进制 `k8s` 集群。四个字：稳如老狗。
  

  

总之，在该教程中，部署方式简单，只要复制黏贴微修改 `ip` 地址就能学习到高难度的证书以及二进制部署。文章设计到的一些额外的知识点，相信能够帮助新人通过简单的方式快速的学习 `k8s`。
  

  

该教程已经在微信群内部推广过，小伙伴反响都不错。一个字，撸它。撸就完事了。
  

  

有啥问题。环境来博客评论区提问
