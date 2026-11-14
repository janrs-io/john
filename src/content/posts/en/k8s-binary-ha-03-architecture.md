---
title: "Kubernetes Binary HA Cluster Deployment Series (3/18) — Architecture & Deployment Plan"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:13:00Z
tags:
  - Kubernetes
  - Architecture
description: "Cluster topology, software versions (Kubernetes 1.23.9, CoreDNS 1.9.3, Calico 3.24, AlmaLinux 8.6), Pod/Service CIDRs, and host inventory."
---

> [!TIP]
> Kubernetes Binary Deployment — Cluster Architecture & Plan

***

# Deployment Plan

## Software Versions

- **Kubernetes**: `1.23.9`
- **CoreDNS**: `1.9.3`
- **Calico**: `3.24`
- **OS**: `AlmaLinux 8.6`, Kernel `4.18`

## Network CIDRs

- **Pod / Endpoint / Calico CIDR**: `10.100.0.0/16`
- **Service ClusterIP CIDR**: `10.68.0.1/16`

## Server Topology

- 3 servers for the `etcd` cluster
- 3 servers for the Kubernetes `master` (control-plane) cluster
- 2 servers for `nginx + keepalived` HA load balancing
- 1 server for the worker `node`

*All hosts are `4 vCPU / 4 GB RAM`:*

| Hostname | OS | Spec | IP Address | Components |
| --- | --- | --- | --- | --- |
| `etcd-01` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.111` | `etcd` |
| `etcd-02` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.112` | `etcd` |
| `etcd-03` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.113` | `etcd` |
| `k8s-master-01` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.121` | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` |
| `k8s-master-02` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.122` | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` |
| `k8s-master-03` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.123` | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` |
| `k8s-ha-master` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.201`, VIP `172.16.222.110` | `nginx`, `keepalived` |
| `k8s-ha-backup` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.202` | `nginx`, `keepalived` |
| `node` | `AlmaLinux 8.6` | `4c4G` | `172.16.222.231` | `kubelet`, `kube-proxy`, `coredns`, `calico` |
