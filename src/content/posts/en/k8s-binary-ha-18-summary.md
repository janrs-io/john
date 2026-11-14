---
title: "Kubernetes Binary HA Cluster Deployment Series (18/18) — Final Summary"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:19:00Z
tags:
  - Kubernetes
description: "Wrapping up the 18-part Kubernetes binary HA cluster series: key takeaways on SSL/TLS CAs, kubelet certificate modes, HA VIPs, RBAC, and kubeconfig."
---

> [!TIP]
> Kubernetes Binary Deployment — Final Summary

***

# Final Summary

> [!NOTE]
> Aside from the optional `Aggregator` proxy layer, we now have a complete high-availability Kubernetes cluster with Node authorization and RBAC enabled, deployed from official binaries.

## 1. SSL/TLS Certificates

Working through a binary deployment makes it clear that SSL/TLS in Kubernetes isn't mysterious—it boils down to CAs, one-way TLS, and mutual TLS:

- Any independent service accepting TLS connections can maintain its own **CA**.
- A service uses its own CA to issue its **`server` certificate**.
- When clients must authenticate to a service, that service's CA issues **`client` certificates** carrying `CN` (username) and `O` (group/organization).
- When cluster peers act as both server and client to each other (only **`etcd`** in Kubernetes), they use **`peer` certificates**.

The real challenge in Kubernetes is understanding how its components interact—once that architecture clicks, the certificates and RBAC bindings fall right into place.

## 2. `kubelet` Certificate Modes

`kubelet` supports three certificate provisioning approaches:
- **Manual certificates** (explicit `kubeconfig` + `server` cert per node)
- **Self-signed serving certificates** (automatic `server` cert generation)
- **TLS Bootstrapping** (automatic `client` and `server` CSR issuance, auto-approval, and certificate rotation)

In production, **TLS Bootstrapping** with automatic CSR approval and rotation is the cleanest and most scalable choice.

## 3. `kube-apiserver` High Availability

Kubernetes does not ship built-in HA for `kube-apiserver`; front it with **Nginx + Keepalived** (or a cloud load balancer). Always remember to include every HA node IP and VIP in `kube-apiserver`'s `server` certificate SAN (`hosts`) list—otherwise TLS verification will fail.

## 4. RBAC Authorization

Kubernetes maps client X.509 certificate fields directly into RBAC identities:
- `CN` -> **User**
- `O` -> **Group**

Built-in cluster roles/identities include `system:kube-controller-manager`, `system:node:<nodeName>` (`system:nodes`), and `system:kube-proxier`. Binding the right `ClusterRole` or `Role` to a client's `CN`/`O` grants it exact API permissions.

## 5. `kubeconfig`

A `kubeconfig` file bundles cluster endpoint + CA trust, client credentials/certs, and contexts—making it easy to switch between clusters and identities cleanly.

***
