---
title: "【实战加详解】二进制部署k8s高可用集群教程系列三 - 部署方案"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:13:00Z
tags:
  - Kubernetes
  - Architecture
description: "k8s 版本：1.23.9 coredns 版本：1.9.3 calico 版本：3.24 系统：AlamLinux8.6 ，内核版本：4.18 - pod/endpoint/calico 网段：10.100.0.0/16 - service 网段：10.68.0.1/16 - 三台服务器部署 etcd 集群"
---

> [!TIP]
> 二进制部署 `k8s` - 方案

***

# 方案

## 版本

`k8s` 版本：`1.23.9`
  

`coredns` 版本：`1.9.3`
  

`calico` 版本：`3.24`
  

系统：`AlamLinux8.6` ，内核版本：`4.18`

## 网络规划

- `pod/endpoint/calico` 网段：`10.100.0.0/16`
- `service` 网段：`10.68.0.1/16`

## 服务器方案

- 三台服务器部署 `etcd` 集群
- 三台服务器部署 `master` 集群
- 两台服务器部署 `nginx + keepalived` 做 `HA`
- 一台服务器部署 `node` 节点

配置如下：
  

*全部为 `4c4G`*

| 主机名称 | 操作系统 | 配置 | IP | 部署服务 |
| -- | ---- | --- | --- | ------- |
| `etcd-01` | `Alma-8.6` | `4c4G` | `172.16.222.111` | `etcd` |
| `etcd-02` | `Alma-8.6` | `4c4G` | `172.16.222.112` | `etcd` |
| `etcd-03` | `Alma-8.6` | `4c4G` | `172.16.222.113` | `etcd` |
| `k8s-master-01` | `Alma-8.6` | `4c4G` | `172.16.222.121` | `kube-apiserver` , `kuber-controller-manager` , `kube-scheduler` |
| `k8-master-02` | `Alma-8.6` | `4c4G` | `172.16.222.122` | `kube-apiserver` , `kuber-controller-manager` , `kube-scheduler` |
| `k8-master-03` | `Alma-8.6` | `4c4G` | `172.16.222.123` | `kube-apiserver` , `kuber-controller-manager` , `kube-scheduler` |
| `k8s-ha-master` | `Alma-8.6` | `4c4G` | `172.16.222.201`,`vip172.16.222.110` | `nginx`,`keepalived` |
| `k8s-ha-backup` | `Alma-8.6` | `4c4G` | `172.16.222.202` | `nginx`,`keepalived` |
| `node` | `Alma-8.6` | `4c4G` | `172.16.222.231` | `kubelet` , `kube-proxy`,`coredns`,`calico` |
