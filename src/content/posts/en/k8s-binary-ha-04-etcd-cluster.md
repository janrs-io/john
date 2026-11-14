---
title: "Kubernetes Binary HA Cluster Deployment Series (4/18) — Deploying the etcd Cluster"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:12:00Z
tags:
  - Kubernetes
  - etcd
description: "Deploying a 3-node TLS-secured etcd v3.5.5 cluster on AlmaLinux 8.6 using cfssl certificates, systemd units, and endpoint health verification."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying the `etcd` Cluster

***

- **OS**: `AlmaLinux 8.6`
- **etcd**: `3.5.5`

# Deploying the `etcd` Cluster

## 1. System Initialization

> [!NOTE]
> Run these steps on every `etcd` node.

### 1.1 Set Hostnames

> [!NOTE]
> Reboot after setting hostnames if needed for prompt/service consistency.

```shell
hostnamectl set-hostname etcd-01
hostnamectl set-hostname etcd-02
hostnamectl set-hostname etcd-03
```

### 1.2 Configure `/etc/hosts`

```shell
cat >> /etc/hosts <<EOF
172.16.222.111  etcd-01
172.16.222.112  etcd-02
172.16.222.113  etcd-03
EOF
```

### 1.3 Disable SELinux

```shell
setenforce 0 && \
sed -ri '/^SELINUX/s/(SELINUX=)(.*)/\1disabled/' /etc/selinux/config
```

### 1.4 Disable `firewalld` and `iptables`

> [!NOTE]
> Skip `iptables` if it is not installed.

```shell
systemctl stop firewalld && systemctl disable firewalld && \
systemctl stop iptables && systemctl disable iptables
```

### 1.5 Create Directories

```shell
mkdir -p /etc/etcd/ssl/ && \
mkdir -p /ssl/
```

### 1.6 Set the `etcd` API Version

> [!NOTE]
> Modern `etcd` uses API v3. Export `ETCDCTL_API=3` (or profile variable) on every node.

```shell
cat >> /etc/profile <<EOF
export ETCDETC_API=3
EOF
source /etc/profile
```

## 2. Generate SSL Certificates

> [!NOTE]
> We use Cloudflare's `cfssl` toolkit to generate `etcd-ca`, `etcd-server`, and `etcd-peer` certificates. Certificates can be generated on one host and distributed to all `etcd` nodes.

### 2.1 Download `cfssl`

```shell
cd /home && \
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 && \
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 && \
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 && \
mv cfssl_linux-amd64 /usr/local/bin/cfssl && \
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

### 2.2 Generate Certificates

#### 2.2.1 Create `ca-config.json`

```shell
cat > /ssl/ca-config.json <<EOF
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
EOF
```

#### 2.2.2 Generate the `etcd` CA Root Certificate

> [!NOTE]
> Uses RSA 4096-bit keys and outputs `etcd-ca-key.pem`, `etcd-ca.csr`, and `etcd-ca.pem`. Keep `etcd-ca-key.pem` secure.

```shell
cat > /ssl/etcd-ca-csr.json <<EOF
{
    "key": {
        "algo": "rsa",
        "size": 4096
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert -initca etcd-ca-csr.json | \
cfssljson -bare etcd-ca - && \
ls etcd-ca* | grep etcd-ca
```

#### 2.2.3 Generate the `etcd` `server` Certificate

> [!NOTE]
> Include all `etcd` node IP addresses in `hosts`.

```shell
cat > /ssl/etcd-server-csr.json <<EOF
{
    "CN": "etcd-server",
    "hosts": [
        "127.0.0.1",
        "172.16.222.111",
        "172.16.222.112",
        "172.16.222.113"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "etcd-cluster",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert \
-ca=etcd-ca.pem \
-ca-key=etcd-ca-key.pem \
-config=ca-config.json \
-profile=server etcd-server-csr.json | \
cfssljson -bare etcd-server && \
ls etcd-server* | grep etcd-server
```

#### 2.2.4 Generate the `etcd` `peer` Certificate

> [!NOTE]
> Each `etcd` member acts as both server and client to other members, so it requires a `peer` certificate listing all node IPs.

```shell
cat > /ssl/etcd-peer-csr.json <<EOF
{
    "CN": "etcd-peer",
    "hosts": [
        "127.0.0.1",
        "172.16.222.111",
        "172.16.222.112",
        "172.16.222.113"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "O": "etcd-cluster",
            "OU": "System",
            "ST": "Beijing"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert \
-ca=etcd-ca.pem -ca-key=etcd-ca-key.pem \
-config=ca-config.json \
-profile=peer etcd-peer-csr.json | \
cfssljson -bare etcd-peer && \
ls etcd-peer* | grep etcd-peer
```

### 2.3 Distribute Certificates

Distribute the CA certificate:

```shell
scp /ssl/etcd-ca.pem root@172.16.222.111:/etc/etcd/ssl/ && \
scp /ssl/etcd-ca.pem root@172.16.222.112:/etc/etcd/ssl/ && \
scp /ssl/etcd-ca.pem root@172.16.222.113:/etc/etcd/ssl/
```

Distribute the `server` certificate:

```shell
scp /ssl/etcd-server*.pem root@172.16.222.111:/etc/etcd/ssl/ && \
scp /ssl/etcd-server*.pem root@172.16.222.112:/etc/etcd/ssl/ && \
scp /ssl/etcd-server*.pem root@172.16.222.113:/etc/etcd/ssl/
```

Distribute the `peer` certificate:

```shell
scp /ssl/etcd-peer*.pem root@172.16.222.111:/etc/etcd/ssl/ && \
scp /ssl/etcd-peer*.pem root@172.16.222.112:/etc/etcd/ssl/ && \
scp /ssl/etcd-peer*.pem root@172.16.222.113:/etc/etcd/ssl/
```

## 3. Deploy `etcd`

### 3.1 Create Data & Working Directories

> [!NOTE]
> Run on every `etcd` server. Working dir: `/var/lib/etcd`, data dir: `/data/etcd`, config: `/etc/etcd/etcd.conf` (`static` cluster mode).

```shell
mkdir /var/lib/etcd/ && \
mkdir -p /data/etcd/ && \
chmod -R 777 /data/etcd/
```

### 3.2 Download and Distribute `etcd` Binaries

```shell
cd /home && \
wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz
tar -zxvf etcd-v3.5.5-linux-amd64.tar.gz
cd /home/etcd-v3.5.5-linux-amd64/ && \
cp etcd etcdctl etcdutl /usr/local/bin/
```

Distribute binaries to all three nodes:

```shell
scp etcd etcdctl etcdutl root@172.16.222.111:/usr/local/bin/ && \
scp etcd etcdctl etcdutl root@172.16.222.112:/usr/local/bin/ && \
scp etcd etcdctl etcdutl root@172.16.222.113:/usr/local/bin/
```

### 3.3 Create `/etc/etcd/etcd.conf`

> [!NOTE]
> Update `ETCD_NAME` and node IP addresses for each host.

#### On `etcd-01`

```shell
cat > /etc/etcd/etcd.conf <<EOF
ETCD_NAME=etcd-01
ETCD_DATA_DIR="/data/etcd/"

ETCD_LISTEN_CLIENT_URLS="https://172.16.222.111:2379,https://127.0.0.1:2379"
ETCD_LISTEN_PEER_URLS="https://172.16.222.111:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.222.111:2380"
ETCD_INITIAL_CLUSTER="etcd-01=https://172.16.222.111:2380,etcd-02=https://172.16.222.112:2380,etcd-03=https://172.16.222.113:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.222.111:2379"

ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-server-key.pem"

ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-peer-key.pem"
EOF
```

#### On `etcd-02`

```shell
cat > /etc/etcd/etcd.conf <<EOF
ETCD_NAME=etcd-02
ETCD_DATA_DIR="/data/etcd/"

ETCD_LISTEN_CLIENT_URLS="https://172.16.222.112:2379,https://127.0.0.1:2379"
ETCD_LISTEN_PEER_URLS="https://172.16.222.112:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.222.112:2380"
ETCD_INITIAL_CLUSTER="etcd-01=https://172.16.222.111:2380,etcd-02=https://172.16.222.112:2380,etcd-03=https://172.16.222.113:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.222.112:2379"

ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-server-key.pem"

ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-peer-key.pem"
EOF
```

#### On `etcd-03`

```shell
cat > /etc/etcd/etcd.conf <<EOF
ETCD_NAME=etcd-03
ETCD_DATA_DIR="/data/etcd/"

ETCD_LISTEN_CLIENT_URLS="https://172.16.222.113:2379,https://127.0.0.1:2379"
ETCD_LISTEN_PEER_URLS="https://172.16.222.113:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.222.113:2380"
ETCD_INITIAL_CLUSTER="etcd-01=https://172.16.222.111:2380,etcd-02=https://172.16.222.112:2380,etcd-03=https://172.16.222.113:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.222.113:2379"

ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/etcd-server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/etcd-server-key.pem"

ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/etcd-ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd-peer.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd-peer-key.pem"
EOF
```

### 3.4 Create the `systemd` Unit File

```shell
cat > /usr/lib/systemd/system/etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
Type=notify

[Install]
WantedBy=multi-user.target
EOF
```

### 3.5 Start and Enable `etcd`

> [!NOTE]
> Start `etcd` on all three nodes concurrently (across 3 SSH sessions) so members can discover each other during initial cluster bootstrap.

```shell
systemctl start etcd && \
systemctl enable etcd
```

### 3.6 Verify Cluster Health

Check endpoint health:

```shell
etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem \
--cert=/etc/etcd/ssl/etcd-peer.pem \
--key=/etc/etcd/ssl/etcd-peer-key.pem \
--endpoints="https://172.16.222.111:2379,https://172.16.222.112:2379,https://172.16.222.113:2379" \
endpoint health --write-out="table"
```

Expected output (`HEALTH` is `true` for all 3 nodes):

```text
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
| https://172.16.222.111:2379 |   true | 20.329616ms |       |
| https://172.16.222.112:2379 |   true | 24.033545ms |       |
| https://172.16.222.113:2379 |   true | 23.918639ms |       |
+-----------------------------+--------+-------------+-------+
```

List cluster members:

```shell
etcdctl --cacert=/etc/etcd/ssl/etcd-ca.pem \
--cert=/etc/etcd/ssl/etcd-peer.pem \
--key=/etc/etcd/ssl/etcd-peer-key.pem \
--endpoints="https://172.16.222.111:2379,https://172.16.222.112:2379,https://172.16.222.113:2379" \
member list --write-out="table"
```

```text
+------------------+---------+---------+-----------------------------+-----------------------------+------------+
|        ID        | STATUS  |  NAME   |         PEER ADDRS          |        CLIENT ADDRS         | IS LEARNER |
+------------------+---------+---------+-----------------------------+-----------------------------+------------+
| 1cdccba2db807b88 | started | etcd-01 | https://172.16.222.111:2380 | https://172.16.222.111:2379 |      false |
| 58badf9cfae9256f | started | etcd-03 | https://172.16.222.113:2380 | https://172.16.222.113:2379 |      false |
| 7cf1d6781c958345 | started | etcd-02 | https://172.16.222.112:2380 | https://172.16.222.112:2379 |      false |
+------------------+---------+---------+-----------------------------+-----------------------------+------------+
```

### 3.7 Cleanup / Reset Commands

```shell
# Stop and disable etcd
systemctl stop etcd && systemctl disable etcd && systemctl daemon-reload

# Remove old data if re-bootstrapping
rm -rvf /data/etcd/* && rm -rvf /var/lib/etcd/*

# Remove SSL certs and config
rm -rvf /etc/etcd/ssl/*
rm -rvf /etc/etcd/etcd.conf
```

***

The `etcd` cluster is now up and healthy.
