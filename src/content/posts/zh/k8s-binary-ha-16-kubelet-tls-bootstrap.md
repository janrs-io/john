---
title: "【实战加详解】二进制部署k8s高可用集群教程系列十六 - 部署kubelet-TLS Bootstrap证书方式"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:19:00Z
tags:
  - Kubernetes
  - Security
description: "在前面的手动颁发证书教程中提到的，kubelet 是通过 kubeconfig 文件访问 kube-apiserver 服务的。 需要为每个节点创建 system:node:<nodeName> 的节点用户，并且需要手动设置 kubeconfig。 如果节点有几百个或者几千个，并且经常变动，那么管理难度..."
---

> [!TIP]
> 二进制部署 `k8s` - 部署 `kubelet`   

> `TLS Bootstrap` 引导证书方式

***

# 部署 `kubelet`

## 1.`TLS Bootstrap` 引导证书方式

> [!NOTE]
> 需要搞明白的是：`TLS Bootstrap` 是用来自动管理 `kube-apiserver` 为 `kubelet` 颁发的客户端 `client` 证书的。
>   

> 前面两个手动颁发证书方式以及自动颁发证书方式是用来颁发 `kubelet` 的 `server` 证书的。
>   

>   

> 这里的区别也是绝大部分新手被绕晕的地方之一。

官方文档地址：
  

(https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/bootstrap-tokens/)
  

(https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
  

(https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#static-token-file)
  

  

在前面的手动颁发证书教程中提到的，`kubelet` 是通过 `kubeconfig` 文件访问 `kube-apiserver` 服务的。
  

需要为每个节点创建 `system:node:<nodeName>` 的节点用户，并且需要手动设置 `kubeconfig`。
  

如果节点有几百个或者几千个，并且经常变动，那么管理难度可想而知。`k8s` 就提供了一个全自动的方式来管理 `kubelet` 的 `client`
证书。
  

这个方式就是 `TLS Bootstrap` 引导证书方式。具体的工作流程直接查看上面的官方说明。

### 1-1.停止所有服务

停止 `master` 上的三大组件的服务以及 `node` 节点上的 `kubelet` 以及 `kube-proxy` 服务
  

  

停止 `master` 三大组件的服务

```shell
systemctl stop kube-apiserver && \
systemctl stop kube-controller-manager && \
systemctl stop kube-scheduler
```

停止 `node` 节点上的 `kubelet` 以及 `kube-proxy` 服务

```shell
systemctl stop kubelet && \
systemctl stop kube-proxy
```

### 1-2.删除 `kubelet.kubeconfig`

删除 `kubelet` 节点上的 `kubelet.kubeconfig`。使用 `TLS Bootstrap` 机制会创建新的。并且跟原来的不一样。
  

如果不删除，检测到存在，会直接使用而不创建。从而导致鉴权失败，访问不了 `kube-apiserver`。

```shell
rm -rvf /etc/kubernetes/kubeconfig/kubelet.kubeconfig
```

### 1-3.创建 `auth-token.csv`

> [!NOTE]
> 初次连接 `kube-apiserver` 需要提供简单的身份认证。
>   

> 身份认证方式有很多种，这里使用静态令牌认证文件。即：`auth-token`。
>   

> 格式为：`token,user,uid,"group1,group2,group3"`。多个 `group` 要加双引号，单个不能加。
>   

>   

> 该文件在哪里创建都可以，只要随机码至少是 `128` 位，文件格式为 `csv`，
>   

> 并且保证 `master` 跟 `node` 的 `token` 一致就行。

大多数 `linux` 都能提供生成 `token` 的方式

```shell
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

*显示*

```text
dafe33bc8fcdae7f9f16df53a95199fa
```

创建 `auth-token.csv`
  

*复制上面生成的随机码*

```shell
cat > /etc/kubernetes/config/auth-token.csv <<EOF
dafe33bc8fcdae7f9f16df53a95199fa,kubelet-bootstrap,10001,system:bootstrappers
EOF
```

分发到 `master` 节点
  

*在哪里创建都可以。只要保证两边的认证数据都一样就行。*

```shell
scp /etc/kubernetes/config/auth-token.csv root@172.16.222.121:/etc/kubernetes/config/ && \

scp /etc/kubernetes/config/auth-token.csv root@172.16.222.122:/etc/kubernetes/config/ && \

scp /etc/kubernetes/config/auth-token.csv root@172.16.222.123:/etc/kubernetes/config/
```

### 1-4.添加 `kube-apiserver` 启动参数

在 `kube-apiserver` 启动参数中加入指定 `auth-token.csv` 文件地址的参数

```shell
--token-auth-file=/etc/kubernetes/config/auth-token.csv
```

### 1-5.添加 `kube-controller-manager` 启动参数

> [!NOTE]
> 证书颁发是由 `kube-controller-manager` 来完成的。

在 `kube-controller-manager` 启动参数中加入颁发客户端证书的相关参数

```shell
# 这里指定的证书需要跟 kube-apiserver 中 --client-ca-file 参数指定的证书一致
# 在部署 kube-apiserver 的时候已经设置了该参数
--cluster-signing-cert-file=/etc/kubernetes/pki/apiserver/apiserver-ca.pem

--cluster-signing-key-file=/etc/kubernetes/pki/apiserver/apiserver-ca-key.pem

# 前面部署 kube-controller-manager 的时候已经添加
--controllers=*,bootstrapsigner,tokencleaner

# 所签名的证书有效期，默认1年，如果修改不要修改过长，过长有可能导致不生效
--cluster-signing-duration=8760h0m0s
```

### 1-6.授权 `kubelet` 创建 `csr`

当通过了简单的 `token` 认证后，`kubelet` 会向 `master` 请求创建 `csr` 文件以及取回。 需要创建权限允许此操作。

```shell
cat > /etc/kubernetes/init_k8s_config/allow-kubelet-create-client-csr.yaml <<EOF
# 允许启动引导节点创建 CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-kubelet-create-client-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-kubelet-create-client-csr.yaml
```

### 1-7.创建 `bootstrap.kubeconfig`

`master` 使用 `bootstrap` 机制来颁发 `kubelet` 的客户端 `client` 证书。同样的 `bootstrap` 也需要使用 `kubeconfig`
来跟 `kube-apiserver` 通信。

> **在 `node` 节点创建。**

设置 `bootstrap` 参数

```shell
kubectl config set-cluster bootstrap \
--server=https://172.16.222.110:8443 \
--certificate-authority=/etc/kubernetes/pki/apiserver/apiserver-ca.pem \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
```

设置 `kubelet-bootstrap` 认证参数
  

**此处的 `token` 指定的值就是上面生成的 `128` 位随机码。两者必须一致。**

```shell
kubectl config set-credentials kubelet-bootstrap \
--token=dafe33bc8fcdae7f9f16df53a95199fa \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
```

设置用户参数

```shell
kubectl config \
set-context bootstrap --user=kubelet-bootstrap \
--cluster=bootstrap \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
```

设置当前上下文参数

```shell
kubectl config use-context bootstrap \
--kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig
```

### 1-8.添加 `kubelet` 启动参数

启动参数配置文件地址：`/etc/kubernetes/config/kubelet.conf`

```shell
# 引导认证文件
--bootstrap-kubeconfig=/etc/kubernetes/kubeconfig/bootstrap.kubeconfig

# 需要指定真实的认证文件。只需要指定一个位置就行
# 通过认证后生成证书后会自动生成
# 在文章开头已经删除了旧的 kubelet.kubeconfig
--kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig

# 自动颁发的 kubelet 的 client 和 server 存放在这
--cert-dir=/etc/kubernetes/pki/kubelet/
```

### 1-9.启动所有服务

启动 `master` 三大组件的服务

```shell
systemctl start kube-apiserver && \
systemctl start kube-controller-manager && \
systemctl start kube-scheduler
```

> 确保 `master` 服务已经启动后，再启动以下 `node` 节点的服务

启动 `node` 节点的 `kubelet` 以及 `kube-proxy` 服务

```shell
systemctl start kubelet && \
systemctl start kube-proxy
```

当 `kubelet` 启动后，查看服务状态，会显示以下一条信息
  

*该显示表示正在等待通过授权*

```text
"Waiting for client certificate to be issued"
```

### 1-10.通过授权

在 `master` 节点查看 `kubelet` 发起创建 `csr` 的请求

```shell
kubectl get csr
```

显示如下
  

*此时状态为 `Pending` ，正在等待批复授权*

```text
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-RGFX9jfBr_SP-0-yH5Dxl3deBooS3qaqmsH2vN9TGE0   3m1s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

通过授权的操作

```shell
kubectl certificate approve node-csr-RGFX9jfBr_SP-0-yH5Dxl3deBooS3qaqmsH2vN9TGE0
```

拒绝授权的操作

```shell
kubectl certificate deny node-csr-RGFX9jfBr_SP-0-yH5Dxl3deBooS3qaqmsH2vN9TGE0
```

### 1-11.检测

当授权通过后，`kubeliet.kubeconfig` 已经生成。
  

文件位置为：`/etc/kubernetes/kubeconfig/kublet.kubeconfig`，打开文件可以看到已经配置了客户端证书。
  

证书位于：`--cert-dir` 参数指定的 `/etc/kubernetens/pki/kubelet/` 文件夹中。

```text
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/apiserver/apiserver-ca.pem
    server: https://172.16.222.110:8443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-auth
  user:
    client-certificate: /etc/kubernetes/pki/kubelet/kubelet-client-current.pem
    client-key: /etc/kubernetes/pki/kubelet/kubelet-client-current.pem
```

当授权通过后，前面已经部署的所有 `pod` 将会继续调度到该 `node` 节点。
  

等待 `3-5` 分钟，`pod` 会慢慢的全部部署。部署后查看节点是否变成 `Ready`。

```shell
kubectl get nodes
```

*显示状态已经变成 `Ready`*

```text
NAME         STATUS   ROLES    AGE   VERSION
k8s-node01   Ready    <none>   34h   v1.23.9
```

再查看所有 `pod`

```shell
kubectl get pods -A
```

*显示所有 `pod` 正常运行*

```text
NAMESPACE          NAME                                       READY   STATUS    RESTARTS       AGE
calico-apiserver   calico-apiserver-69c54b8687-lm9qv          1/1     Running   0              105m
calico-apiserver   calico-apiserver-69c54b8687-ssqs4          1/1     Running   0              105m
calico-system      calico-kube-controllers-688968c9b6-w7bnw   1/1     Running   0              105m
calico-system      calico-node-m7zxb                          1/1     Running   0              34h
calico-system      calico-typha-7bd99d8c79-vj4lw              1/1     Running   0              34h
calico-system      csi-node-driver-v4g95                      2/2     Running   0              34h
dev                busybox-5b75bcbcb8-rt4lz                   1/1     Running   0              105m
dev                nginx-6b8cfcb774-h8ttq                     1/1     Running   0              105m
ingress-nginx      ingress-nginx-controller-jsz9j             1/1     Running   0              27h
ingress-nginx      nginx-errors-6475d75d86-75g8x              1/1     Running   0              105m
tigera-operator    tigera-operator-6dcd98c8ff-f2rw4           1/1     Running   8 (158m ago)   34h
```

### 1-12.自动授权以及自动轮换

> [!NOTE]
> 此处教程只针对 `kubelet` 的客户端 `client` 证书。

上面的方式是通过手动通过授权的。也可以配置使用自动授权以及证书到期后自动更新时间。

#### 1-12-1.自动授权

自动授权需要绑定集群角色 `system:certificates.k8s.io:certificatesigningrequests:nodeclient` 给 `bootstrap`
使用的 `system:bootstrappers` 组

```shell
cat > /etc/kubernetes/init_k8s_config/allow-auto-approve-kubelet-client-csr.yaml <<EOF
# 批复 "system:bootstrappers" 组的所有 CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: allow-auto-approve-kubelet-client-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-auto-approve-kubelet-client-csr.yaml
```

#### 1-12-2.自动续约

> [!NOTE]
> 此处的自动轮换(自动续约)，只针对 `kube-apiserver` 颁发给 `kubelet` 的客户端 `client` 证书。
>   

> `kubelet` 的服务端 `server` 证书也可以自动生成以及自动轮换。下面会讲解。

自动续约需要绑定集群角色 `system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`
给正常工作节点所在的用户组 `system:nodes`。

```shell
# 批复 "system:nodes" 组的 CSR 续约请求
cat > /etc/kubernetes/init_k8s_config/allow-auto-renewal-kubelet-client-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-kubelet-client-csr
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-auto-renewal-kubelet-client-csr.yaml
```

自动续约除了以上需要绑定集群角色外，还需要开启相关参数。

> `kube-controller-manager` 参数

`kube-controller-manager` 需要添加一个用来控制颁发给 `kubelet` 客户端 `client` 证书的过期时间。

```shell
# 1.19 版本以及之后的参数。默认为 1 年的过期时间，可以改成 10 年
# 在前面部署 kube-controller-manager 的时候已经设置
-cluster-signing-duration=8760h0m0s

# 1.19 版本以前的参数。默认为 1 年的过期时间，可以改成 10 年
--experimental-cluster-signing-duration=8760h0m0s
```

> `kubelet` 参数

`kubelet` 需要添加一个参数标志开启客户端证书自动轮换。
  

*添加该参数的配置文件为：`/etc/kubernetes/config/kubelet.yaml`*

```yaml
# 开启 kubelet 客户端 client 证书自动轮换
rotateCertificates: true
```

#### 1-12-3.自动轮换 `kubelet` 服务端 `server` 证书

> [!NOTE]
> 注意区别上面的 `kubelet` 客户端 `client` 证书。
>   

> 以及注意：`kubelet` 的 `server` 证书是不会自动颁发的，只有自动轮换的特性。
>   

>   

> 此处的区别官网文档也是没说的很明白，没有明确的表达出：`kube-apiserver` 或者 `kube-controller` 并不会自动颁发 `kubelet`
> 的 `server` 证书。
>   

>   

> 不管是国内，还是国外。很多人被这一点绕晕了。具体问题描述查看两个博文地址：
>   

> (https://discuss.kubernetes.io/t/how-to-enable-auto-csr-approval-for-kubelet-server-certificates/4495/2)
>   

> (https://github.com/kubernetes/kubernetes/issues/73356)

引导签名方式中， `kubelet` 的 `server` 证书是由 `client/sign` 的 `ca` 机构颁发的。区别于自签名中，是由 `kubelet`
的 `ca` 机构颁发的。
  

  

自动颁发 `kubelet` 的 `server` 证书需要开启以下参数标志

> `kube-controller-manager` 参数

```shell
# 开启颁发 kubelet 服务端 server 证书的特性门控
--feature-gates=RotateKubeletServerCertificate=true
```

> `kubele` 参数

*添加该参数的配置文件为：`/etc/kubernetes/config/kubelet.yaml`*

```shell
# 开启颁发 kubelet 服务端证书的标志
# 开启此标志，必须同时开启下面的特性门控。否则无效。
serverTLSBootstrap: true

# kubelet 同样要开启 Beta/Alpha 特性门控标志
# 该特性是用来自动轮换的，并不是自动颁发的
featureGates:
  RotateKubeletServerCertificate: true
```

> 创建授权自动轮换 `kubelet` 服务端 `server` 证书的角色以及绑定

在上面已经提到了。`kube-apiserver` 的 `TLS Bootstrap` 机制主要是用来颁发 `kubelet`
的客户端证书的，并且自带的两个集群角色用来创建 `client` 证书以及自动轮换

- `system:certificates.k8s.io:certificatesigningrequests:nodeclient`
- `system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`

但是对于自动轮换 `kubelet` 的服务端 `server` 证书，并没有自带的集群角色，需要自己再创建

```shell
cat > /etc/kubernetes/init_k8s_config/allow-create-kubelet-server-csr.yaml <<EOF
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests"]
  verbs: ["create","list","get","watch"]
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/allow-create-kubelet-server-csr.yaml
```

并且赋予到前面 `auth-token.csv` 文件中配置的用户组：`system:bootstrappers`

```shell
cat > /etc/kubernetes/init_k8s_config/bind-create-kubelet-server-csr.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-create-kubelet-server-csr
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /etc/kubernetes/init_k8s_config/bind-create-kubelet-server-csr.yaml
```

***

## 2.总结

`kubelet` 证书也是一个容易绕晕的地方，有几个地方要注意：

> - 初始引导的身份认证

初始引导的身份认证常用的有两种，一种是 `auth token file` 的方式，也就是本文使用的方式。 另外一种是 `bootstrap token` 方式。
  

两者区别的地方有两个。一个是在配置 `bootstrap.kubeconfig` 的时候指定的 `token` 值的区别；一个是使用 `bootstrap token`
方式需要创建 `secret`。

> - 授权 `kubelet` 创建 `csr`

在通过了初次引导认证后，还需要创建一个集群角色绑定到请求创建 `csr` 文件的用户组，本文设置的是 `system:bootstrappers`。
  

如果没有创建该集群角色的绑定，`kubelet`
服务会启动后又失败。没有注意排查，会以为初始引导认证没通过，实际上已经通过初始认证了。但是由于没有创建 `csr` 的权限，又关闭了服务。

> - 三者的区别

在前面已经提到了。手动颁发证书方式跟自签名证书方式，这两个主要是针对 `kubelet` 服务端的 `server` 证书。
  

自签名证书是不是由 `kube-apiserver` 的 `ca` 机构颁发的，是由 `kubelet` 的 `ca` 机构颁发的。
  

  

而 `TLS Bootstrap自动引导证书` 是针对 `kube-apiserver` 颁发给 `kubelet` 客户端的 `client` 证书。
  

  

也就是说，`手动颁发证书/自签名证书` 跟 `TLS Bootstrap自动引导证书` 是无关的。很多人在这里绕晕了。

> 开启 `kubelet` 的服务端 `server` 证书的自动轮换

`kubelet` 的服务端 `server` 证书是可以通过配置参数标志来开启 `kube-apiserver` 为其自动轮换服务端 `server` 证书的。
  

  

**注意：截止目前位置，并不会自动颁发。这个很多人，不仅国内，国外很多也蒙圈的。**
  

  

但是 `k8s` 并没有自带的集群角色来授权自动轮换的操作，只提供了创建 `kubelet` 客户端 `client` 证书的角色。
  

  

需要自己创建一个拥有可以启动轮换 `kubelet` 的 `server` 证书的角色并绑定到 `auth-token.csv` 中指定的用户组。

***
