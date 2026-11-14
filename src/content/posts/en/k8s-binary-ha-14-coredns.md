---
title: "Kubernetes Binary HA Cluster Deployment Series (14/18) — Deploying CoreDNS"
author: "John Yang"
lang: en
pubDatetime: 2022-10-17T06:18:00Z
tags:
  - Kubernetes
  - Networking
description: "Deploying CoreDNS 1.10.0 as a systemd binary on worker nodes with client TLS certificates, RBAC ClusterRoleBindings, and DNS service discovery verification."
---

> [!TIP]
> Kubernetes Binary Deployment — Deploying `CoreDNS`

***

# Deploying `CoreDNS`

> [!NOTE]
> Deploy the `coredns` binary on the worker `node`.

## 1. Download and Install `CoreDNS`

```shell
cd /home/ && \
wget https://github.com/coredns/coredns/releases/download/v1.10.0/coredns_1.10.0_linux_amd64.tgz && \
tar -zxvf coredns_1.10.0_linux_amd64.tgz && \
mv coredns /usr/local/bin/
```

Create log and config directories:

```shell
mkdir -p /var/log/coredns/ && \
mkdir -p /etc/coredns/
```

## 2. Generate Client SSL Certificate

`coredns` queries `kube-apiserver` for Services and Endpoints/EndpointSlices, so it needs a client certificate signed by `apiserver` CA:

```shell
cat > /ssl/apiserver-coredns-client-csr.json <<EOF
{
    "CN": "coredns",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Beijing",
            "L": "Beijing",
            "O": "k8s",
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
-profile=client apiserver-coredns-client-csr.json | \
cfssljson -bare apiserver-coredns-client && \
ls apiserver-coredns-client* | \
grep apiserver-coredns-client
```

Distribute the certificate and CA to the worker node:

```shell
scp /ssl/apiserver-coredns-client*.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/ && \
scp /ssl/apiserver-ca.pem root@172.16.222.231:/etc/kubernetes/pki/apiserver/
```

## 3. Create `/etc/coredns/Corefile`

Reference: [CoreDNS Kubernetes Plugin Docs](https://coredns.io/plugins/kubernetes/).

```shell
cat > /etc/coredns/Corefile <<EOF
.:53 {
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        endpoint https://172.16.222.110:8443
        tls /etc/kubernetes/pki/apiserver/apiserver-coredns-client.pem /etc/kubernetes/pki/apiserver/apiserver-coredns-client-key.pem /etc/kubernetes/pki/apiserver/apiserver-ca.pem
    }
    health
    errors
    log
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    reload 10s
    loadbalance
    loop
}
EOF
```

## 4. Create CoreDNS RBAC Permissions

Run on a `master` node so the `coredns` user can watch cluster resources:

```shell
cat > /etc/kubernetes/init_k8s_config/create-coredns-rbac-permission.yaml <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: coredns
rules:
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["get","watch","list"]

- apiGroups: [""]
  resources: ["nodes","pods","namespaces","services"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: coredns-access-resources
subjects:
- kind: User
  name: coredns
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: coredns
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/create-coredns-rbac-permission.yaml
```

## 5. Create `systemd` Unit & Start `CoreDNS`

On the worker `node`:

```shell
cat > /usr/lib/systemd/system/coredns.service << EOF
[Unit]
Description=CoreDNS DNS Server
Documentation=https://coredns.io/
After=network.target
StartLimitBurst=1
StartLimitIntervalSec=15s

[Service]
# Using Type=notify causes continuous restarts with CoreDNS
Type=simple
User=root
ExecStart=/usr/local/bin/coredns -conf /etc/coredns/Corefile
StandardOutput=append:/var/log/coredns/coredns.log
StandardError=append:/var/log/coredns/coredns_error.log
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && \
systemctl start coredns && \
systemctl enable coredns
```

Check `/var/log/coredns/coredns.log`:

```text
.:53
[INFO] plugin/reload: Running configuration SHA512 = 75a7e545d2e00af59a3f1984d819f3b35db009002c603603942190c2fe71d13d8a0a86e605e4d42ad2027f8fd9b21ef4a765811b68fde1a0ffc89ce752387c51
CoreDNS-1.10.0
linux/amd64, go1.19.1, 596a9f9
```

## 6. Test DNS Resolution & Service Discovery

> [!NOTE]
> Use `busybox:1.28.3` rather than `1.28.4`, which has a known `nslookup` bug.

On a `master` node:

```shell
kubectl create ns dev
kubectl label node k8s-node01 dev-node=true

cat > /home/busybox.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: dev
spec:
  selector:
    matchLabels:
      app: busybox
  replicas: 1
  template:
    metadata:
      labels:
        app: busybox
    spec:
      restartPolicy: Always
      nodeSelector:
        dev-node: 'true'
      containers:
      - name: busybox
        command:
        - sleep
        - "3600"
        image: busybox:1.28.3
EOF
kubectl apply -f /home/busybox.yaml
```

Verify `/etc/resolv.conf`, external DNS, and internal cluster DNS:

```shell
kubectl get pods -n dev
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- cat /etc/resolv.conf
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup www.baidu.com
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup kubernetes.default
```

Deploy a test `nginx` Deployment + Service in `dev` and verify service discovery:

```shell
cat > /home/nginx.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.cn-shenzhen.aliyuncs.com/yjy_k8s/nginx:v1.23.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: dev
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
EOF
kubectl apply -f /home/nginx.yaml
kubectl get svc -n dev
kubectl exec -it busybox-5b75bcbcb8-pqrqr -n dev -- nslookup nginx-svc
```

```text
Server:    10.68.0.1
Address 1: 10.68.0.1

Name:      nginx-svc
Address 1: 10.68.141.101
```

Both external resolution and in-cluster service discovery work as expected.

***
