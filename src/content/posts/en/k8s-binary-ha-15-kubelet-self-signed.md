---
title: "Kubernetes Binary HA Cluster Deployment Series (15/18) — kubelet Self-Signed Certificates"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
description: "Switching kubelet from manual serving certificates to automatic self-signed serving certificates by removing tlsCertFile/tlsPrivateKeyFile and --kubelet-certificate-authority."
---

> [!TIP]
> Kubernetes Binary Deployment — `kubelet` Self-Signed Server Certificates

***

# `kubelet` Self-Signed Server Certificate Mode

Self-signed mode automates `kubelet`'s serving certificate generation instead of manually signing a `server` certificate for every node.

In `/etc/kubernetes/config/kubelet.yaml`, comment out `tlsCertFile` and `tlsPrivateKeyFile`:

```yaml
#tlsCertFile: "/etc/kubernetes/pki/kubelet/kubelet-server.pem"
#tlsPrivateKeyFile: "/etc/kubernetes/pki/kubelet/kubelet-server-key.pem"
```

When omitted, `kubelet` automatically generates a self-signed serving certificate and private key on startup and stores them in the directory specified by `--cert-dir`.

Correspondingly, **remove** `--kubelet-certificate-authority` from `/etc/kubernetes/config/apiserver.conf` on all three `master` hosts so `kube-apiserver` does not reject `kubelet`'s self-signed serving cert:

```shell
--kubelet-certificate-authority=/etc/kubernetes/pki/kubelet/kubelet-ca.pem
```

## Restart Services

Restart `kube-apiserver` on all masters and `kubelet` on the worker node for the changes to take effect.
