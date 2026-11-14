---
title: "Kubernetes Binary HA Cluster Deployment Series (12/18) — Deploying kube-proxy"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:17:00Z
tags:
  - Kubernetes
  - Networking
description: "Generating the kube-proxy client certificate, creating kubeconfig pointing to the HA VIP, and running kube-proxy in IPVS mode on worker nodes."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `kube-proxy`

***

# Deploying `kube-proxy`

`kube-proxy` acts as the cluster's service network proxy and load balancer, watching Kubernetes `Service` and `Endpoints` objects from `kube-apiserver` and programming node networking rules. Because it connects to `kube-apiserver`, it needs a client certificate signed by the `apiserver` CA.

## 1. Generate the SSL Certificate

> [!NOTE]
> Set `CN` to `system:kube-proxy`. As a client certificate, `hosts` is not needed.

```shell
cat > /ssl/apiserver-kube-proxy-client-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "kube-proxy",
            "OU": "system"
        }
    ]
}
EOF
cd /ssl/ && \
cfssl gencert \
-ca=apiserver-ca.pem \
-ca-key=apiserver-ca-key.pem \
-config=ca-config.json \
-profile=client apiserver-kube-proxy-client-csr.json | \
cfssljson -bare apiserver-kube-proxy-client && \
ls apiserver-kube-proxy-client* | \
grep apiserver-kube-proxy-client
```

## 2. Distribute the Certificate

```shell
scp /ssl/apiserver-kube-proxy-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 3. Create `kubeconfig`

> [!NOTE]
> Since the control-plane HA VIP (`172.16.222.110:8443`) is up, point `--server` to `https://172.16.222.110:8443`.

```shell
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--embed-certs=true \
--server=https://172.16.222.110:8443 \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
--client-certificate=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client.pem \
--client-key=/etc/kubernetes/pki/apiserver/apiserver-kube-proxy-client-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig

kubectl config set-context kube-proxy \
--cluster=kubernetes \
--user=kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig

kubectl config use-context kube-proxy \
--kubeconfig=/etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
```

## 4. Create `kube-proxy.yaml`

> [!NOTE]
> Run on the worker node (`172.16.222.231`). Uses `mode: "ipvs"`.

```shell
cat > /etc/kubernetes/config/kube-proxy.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: '172.16.222.231'
healthzBindAddress: '172.16.222.231:10256'
metricsBindAddress: '127.0.0.1:10249'
bindAddressHardFail: true
clientConnection:
  kubeconfig: /etc/kubernetes/kubeconfig/kube-proxy.kubeconfig
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  qps: 5
clusterCIDR: 10.100.0.0/16
enableProfiling: false
mode: "ipvs"
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 30s
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
EOF
```

## 5. Create Startup Flags File

```shell
cat > /etc/kubernetes/config/kube-proxy.conf <<EOF
KUBE_PROXY_OPTS="--alsologtostderr=true \
--logtostderr=false \
--config=/etc/kubernetes/config/kube-proxy.yaml \
--log-dir=/var/log/kubernetes/kube-proxy \
--v=2"
EOF
```

## 6. Create the `systemd` Service Unit

```shell
cat > /usr/lib/systemd/system/kube-proxy.service <<'EOF'
[Unit]
Description=Kubernetes Kube Proxy Service
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy/
EnvironmentFile=-/etc/kubernetes/config/kube-proxy.conf
ExecStart=/usr/local/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

## 7. Start and Enable `kube-proxy`

```shell
systemctl daemon-reload && \
systemctl start kube-proxy && \
systemctl enable kube-proxy
```

## 8. Verify Service Status

```shell
systemctl status kube-proxy --no-pager -l
```

## 9. Useful Operations

```shell
systemctl stop kube-proxy
systemctl status kube-proxy --no-pager -l
journalctl -l --no-pager -u kube-proxy
rm -rvf /var/log/journal/*
```

***

`kube-proxy` is now running on the worker node.
