---
title: "Kubernetes Binary HA Cluster Deployment Series (1/18) — Preface"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T03:01:00Z
tags:
  - Kubernetes
  - DevOps
description: "Preface to the 18-part hands-on guide for deploying a high-availability Kubernetes cluster from binaries with etcd, Nginx+Keepalived, Calico, CoreDNS, and TLS Bootstrap."
---

> [!TIP]
> Deploying a High-Availability Kubernetes Cluster from Binaries

***

This is a step-by-step, hands-on tutorial series for deploying a high-availability Kubernetes cluster directly from official binaries, using **Calico** as the CNI network plugin.

The cluster architecture includes:

- **3-node `etcd` cluster**
- **3-node control plane (`master` cluster)** fronted by **Nginx + Keepalived** for HA
- **`kubelet` and `kube-proxy`** on every worker node
- **Calico** as the CNI network plugin (deployed via Pods, as officially recommended)
- **CoreDNS** for cluster DNS resolution

Beyond binary installation itself, this series dives deep into **X.509 certificates** and the **TLS Bootstrap** mechanism.

Unlike tutorials that reuse a single `apiserver` CA for everything or skip directly to TLS Bootstrapping without explanation, this series walks through CA hierarchies, aggregation/client/signing parameters, and compares manual `kubeconfig` provisioning with TLS Bootstrapping step by step.

All configuration files and commands are ready to use—just swap in your own IP addresses and run them.
