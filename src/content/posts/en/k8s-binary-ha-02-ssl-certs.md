---
title: "Kubernetes Binary HA Cluster Deployment Series (2/18) — SSL/TLS Certificates Overview"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:11:00Z
tags:
  - Kubernetes
  - Security
description: "Demystifying X.509 SSL/TLS certificates in Kubernetes and etcd: CA, server, client, and peer certificates, one-way vs. mutual TLS, and cfssl profiles."
---

> [!TIP]
> Kubernetes Binary Deployment — SSL/TLS Certificates

***

# 1. Introduction to SSL/TLS Certificates

> [!NOTE]
> This guide focuses on practical usage rather than cryptographic theory.

## 1.1 What Is an SSL Certificate?

At its core, an SSL/TLS certificate simply enables encrypted, authenticated communication. What usually confuses newcomers isn't SSL itself—it's seeing a dozen Kubernetes components each needing their own certificates.

Before diving into binary deployment, keep two simple rules in mind:

1. **Any service that accepts encrypted incoming connections needs a `server` certificate:**
   - An HTTPS web server needs a `server` certificate.
   - An `etcd` cluster with TLS enabled needs `server` certificates.
2. **Whenever a server must verify a client's identity, that client needs a `client` certificate:**
   - Just like hardware security tokens in online banking, a `client` certificate proves who the caller is.

Each independent service can use its own Certificate Authority (CA)—self-signed via `openssl` or `cfssl` for internal infrastructure, or commercial CAs such as DigiCert for public web services.

## 1.2 Three Core Concepts

- Certificate Authority (`CA`)
- One-way TLS authentication
- Mutual (two-way) TLS authentication

## 1.3 Key Terminology

- **`client` certificate**: Identifies a client connecting to a server.
- **`server` certificate**: Identifies a server accepting connections.
- **`peer` certificate**: Dual-purpose certificate for cluster members that act as both server and client to each other.
- **`CA` root certificate**: The issuing authority used to sign and verify `server`, `client`, and `peer` certificates.

## 1.4 Certificate Authority (`CA`)

A **CA root certificate** signs `server`, `client`, and `peer` certificates. Any certificate signed by a CA is verified against that same CA root certificate. Independent services (such as `etcd` vs. `kube-apiserver`) can—and often should—have separate CAs.

## 1.5 One-Way TLS Authentication

Consider a public HTTPS website: the server presents its `server` certificate so browsers can encrypt traffic and verify the site's identity, while clients do not present certificates. That is **one-way authentication**.

## 1.6 Mutual (Two-Way) TLS Authentication

If a service restricts access only to authorized clients, the server issues a `client` certificate to each authorized caller and verifies it on every connection. Both sides authenticate each other—this is **mutual TLS (mTLS)**.

## 1.7 Peer Authentication

In a distributed cluster like `etcd`, every node simultaneously acts as a server to other nodes and as a client connecting to its peers. Each node therefore uses both a `server` certificate (for client requests) and a `peer` certificate (carrying both `server auth` and `client auth` usages) for node-to-node communication.

# 2. `cfssl` Configuration

`cfssl` generates X.509 certificates (just like `openssl`). Both `etcd` and Kubernetes use X.509 certificates.

> [!NOTE]
> `cfssl` uses a JSON CA config file:
> - `expiry`: Certificate validity duration
> - `profiles`: Named signing profiles (`server`, `client`, `peer`)
> - `server auth`: Usage for server-side TLS
> - `client auth`: Usage for client-side TLS
> - Both `server auth` and `client auth`: Usage for `peer` certificates

Example `ca-config.json`:

```json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

# 3. Certificates Required by `etcd`

An `etcd` cluster needs:
- An `etcd` **CA root certificate**
- A **`server` certificate** for each `etcd` node
- A **`peer` certificate** for node-to-node communication
- A **`client` certificate** for external callers such as `kube-apiserver`

# 4. Certificates Required by Kubernetes

Kubernetes feels complex certificate-wise only because its control-plane and node components are decoupled services communicating over TLS—much like microservices. Once you map out which components talk to which, the certificate layout becomes completely clear.

## 4.1 Kubernetes Components Using TLS

- `etcd`
- Control plane: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`
- Worker nodes: `kubelet`, `kube-proxy`
- `aggregator-proxy` (optional aggregation layer, not deployed in this series)
- Kubernetes CSR signing (`sign/client`)
- `calico-cni` (Pod-based deployment, no manual certs required here)
- `coredns`

## 4.2 Mapping Communication Paths

> [!NOTE]
> Virtually all cluster control-plane communication flows through `kube-apiserver`. Inspecting `kube-apiserver`'s CLI flags makes every required certificate obvious.

Official references:
- [Kubernetes Command-Line Tools Reference](https://kubernetes.io/docs/reference/command-line-tools-reference/)
- [PKI Certificates and Requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)

## 4.3 `kube-apiserver`

### 4.3.1 `kube-apiserver` CA Root Certificate

```text
--client-ca-file string
# If set, any request presenting a client certificate signed by one of the authorities
# in the client-ca-file is authenticated with an identity corresponding to the CommonName.
```

### 4.3.2 `kube-apiserver` Server Certificate

```text
--tls-cert-file string
# File containing the default x509 Certificate for HTTPS (CA cert concatenated after server cert if any).

--tls-private-key-file string
# File containing the default x509 private key matching --tls-cert-file.
```

### 4.3.3 `kube-controller-manager` Client Certificate

`kube-controller-manager` connects to `kube-apiserver` via a `kubeconfig` file referencing the apiserver CA and its own client certificate:

```text
--certificate-authority
--client-certificate
--client-key
```

### 4.3.4 `kube-scheduler` Client Certificate

`kube-scheduler` likewise authenticates to `kube-apiserver` via a `kubeconfig` containing:

```text
--certificate-authority
--client-certificate
--client-key
```

## 5. `kubelet`

`kubelet` runs on every worker node and involves four certificates:
- `kubelet` CA root certificate (or cluster CA)
- `kubelet` server certificate
- `kubelet` client certificate (when calling `kube-apiserver`)
- `kube-apiserver` client certificate (when calling `kubelet`)

Because worker nodes scale dynamically, manually issuing certificates per node quickly becomes unmanageable—which is why Kubernetes 1.4 introduced **TLS Bootstrapping**.

### 5.1 Manual Certificate Provisioning

> **Note:** When manual `tlsCertFile` / `tlsPrivateKeyFile` flags are set, self-signed certificate generation is disabled.

`kubelet` configuration:

```text
tlsCertFile string
# File containing x509 Certificate for HTTPS.

tlsPrivateKeyFile string
# File containing x509 private key matching tlsCertFile.

authentication
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: # Path to CA certificate for verifying client certs

authorization
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: "5m"
    cacheUnauthorizedTTL: "30s"
```

`kube-apiserver` flags:

```text
--kubelet-certificate-authority string
--kubelet-client-certificate string
--kubelet-client-key string
```

### 5.2 Self-Signed `kubelet` Certificates

To let `kubelet` auto-generate a self-signed serving certificate, comment out `tlsCertFile` and `tlsPrivateKeyFile` in `kubelet` config and **remove** `--kubelet-certificate-authority` from `kube-apiserver`.

### 5.3 TLS Bootstrapping

With **TLS Bootstrapping**, `kubelet` starts with a bootstrap token and automatically submits a CertificateSigningRequest (CSR) to `kube-apiserver`, which signs and issues `kubelet`'s client certificate. We cover this step by step later in the series.

## 6. `aggregator-proxy`

> [!NOTE]
> Aggregation-layer certificates are omitted in this series as the aggregation proxy is not deployed here.

## 7. `service-account`

> [!NOTE]
> This series reuses the `kube-apiserver` CA keypair for ServiceAccount token signing. You can also generate a dedicated SA keypair if desired.

## 8. Cluster Certificate Signing (`sign/client`)

> [!NOTE]
> This series also uses the `kube-apiserver` CA for cluster signing (`--cluster-signing-cert-file` / `--cluster-signing-key-file`), though Kubernetes supports granular per-signer flags as well:

```text
--cluster-signing-cert-file string
--cluster-signing-duration duration     # default: 8760h0m0s
--cluster-signing-key-file string
--cluster-signing-kube-apiserver-client-cert-file string
--cluster-signing-kube-apiserver-client-key-file string
--cluster-signing-kubelet-client-cert-file string
--cluster-signing-kubelet-client-key-file string
--cluster-signing-kubelet-serving-cert-file string
--cluster-signing-kubelet-serving-key-file string
--cluster-signing-legacy-unknown-cert-file string
--cluster-signing-legacy-unknown-key-file string
```

## 9. `etcd` Client Cert for `kube-apiserver`

`kube-apiserver` is the only Kubernetes component that talks directly to `etcd`, configured via:

```text
--etcd-cafile
--etcd-certfile
--etcd-keyfile
```

***
