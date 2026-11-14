---
title: "【实战加详解】二进制部署k8s高可用集群教程系列二 - ssl 证书简介"
author: "John Yang"
lang: zh
pubDatetime: 2022-10-17T06:11:00Z
tags:
  - Kubernetes
  - Security
description: "其实。。。ssl 证书没啥的，就是加密通讯用的，真正让大家头疼的不是 ssl 证书，而是跟 k8s 放在一块，结合 k8s 产生各种证书绕晕了。 先扫除这个疑虑，不要在有这个心智负担的情况下学习二进制部署k8s。把它认为是一件简单的事。 只要是开启通讯加密并且作为对外提供服务的，让外部访问的，并且开启通..."
---

> [!TIP]
> 二进制部署 `k8s` - `ssl` 证书

***

# 1.`ssl` 证书简介

> [!NOTE]
> 在这里只做应用的简介，原理不做介绍。自行谷歌查阅。  

> 涉及到安全的，也不整理文档了。自行研究。

## 1-1.什么是 `ssl` 证书

其实。。。`ssl` 证书没啥的，就是加密通讯用的，真正让大家头疼的不是 `ssl` 证书，而是跟 `k8s` 放在一块，结合 `k8s` 产生各种证书绕晕了。
  

先扫除这个疑虑，不要在有这个心智负担的情况下学习二进制部署`k8s`。把它认为是一件简单的事。

**只要是开启通讯加密并且作为对外提供服务的，让外部访问的，并且开启通讯加密的，就必须要有 `server` 证书。**

- `web` 网站服务开启了 `https` 请求必须要有 `server` 端的 `ssl` 证书
- 对外提供服务注册与发现的 `etcd` 服务开启通讯加密，也必须要有 `server` 端的 `ssl` 证书

**对于 `server` 服务端要认证 `client` 客户端的，客户端就必须要有 `client` 证书**

- 金融行业就是典型的例子：我们登录金融网站需要使用 `u盾`，其实里面就是包含了 `client` 证书。用来验证客户端身份用的。

**不同服务的证书都需要并且都可以用不同的机构才签发证书**

- 个人 `web` 网站用公用的自签机构。例如工具：`openssl` , `cfssl`
- 商业的，例如各大证书签发机构：`DigiCert`

## 1-2.`ssl` 证书的三个要点

- 签发机构 `ca`
- 单向认证
- 双向认证

## 1-3.关键技术名词

- `client` 客户端证书
- `server` 服务端证书
- `peer` 双向对等证书
- `ca` 根证书(也叫 `ca` 签发机构)
- 单向认证
- 双向认证

## 1-4.`ssl` 签发机构 `ca`

- `ca` 签发机构，也叫 `ca` 根证书，是用来生成上面提到的 `server` `client`
  这两个证书的根证书。也就是要生成这三种证书前，必须先生成 `ca` 根证书。再用这个根证书来生成其他三个证书。
- 有了 `ca` 根证书并且用它来生成了其他三种证书，就必须要用同一个 `ca` 根证书来校验。
  签发机构有商用的和自签的。商业的一般都是可信任的机构。

**不同的、独立的服务可以使用独立的 `ca` 机构来签发证书。**

## 1-5.`ssl` 单向认证

从最简单最常用的场景理解 `ssl` 单向认证：
  

  

有一个 `web` 网站，向世界上所有人提供访问。原来是 `http` 类型的，现在是 `https` 类型的，客户端在跟 `web`
服务端交互的时候，数据已经被加密了。这就是在 `web` 端，也就是 `server` 端使用了 `ssl` 证书。
  

注意：此时的 `web` 网站是面向所有人公开访问的，此时的 `web` 端，也就是 `server` 提供的通讯加密不指定客户端。
  

  

**此时，`web` 端的证书称为 `server` 证书。没有客户端 `client` 证书。这时候就叫 `单向认证`。**

## 1-6.`ssl` 双向认证

接上面的 `web` 应用场景，在上面的 `web` 服务中，是向所有用户提供服务的。
  

  

如果 `web` 不向所有人提供访问，只让客户 `A` 访问，这时候就需要 `web` 颁发一个客户端证书(`client` 证书)给客户 `A` ，让 `web`
的 `server`
端来校验是否是合法的用户。
  

如果客户端没有 `client` 证书，是没法访问 `server` 端的。
  

> 典型的应用场景就是金融行业的网站需要认证客户。
  

**此时，`web` 端的证书还是叫 `server` 证书，颁发给客户的证书就是 `client` 证书。客户端要访问服务端，就要使用 `client`
证书。这时候就叫`双向认证`。**

## 1-7.`ssl` 对等认证

`peer` 对等认证。`peer` 跟以上的区别就在于，`server` 端不仅只是单纯的 `server` 端，同时也是 `client` 端；`client`
点不仅只是单纯的 `client` 端，同时也是 `server` 端。
  

  

在集群上的每一个节点，都要提供服务给其他节点访问。在上面已经提到了，只要对外提供服务，就要有服务端的 `server`
证书，所以每个节点都要有 `server` 证书。
  

在 `etcd` 集群上，由于每个节点既要作为服务端和客户端，这时候就需要一个叫 `peer`
双向认证的证书，而不是跟上面一样的叫客户端(`client`)证书。
  

  

**此时，集群上每个节点都需要一个 `server` 证书，并且同时要为每个节点颁发 `peer`
双向认证证书。每个节点之间互相访问，就要使用 `peer` 证书。这时候在 `etcd` 集群上，就叫`对等认证`。**

# 2.`cfssl` 简介

`cfssl` 是用来生成 `X509` 格式的工具，除了该工具，用 `openssl` 或者其他工具。
  

无论使用什么工具，只需要能够生成 `X.509` 格式的加密证书就行。在 `etcd` 集群和 `k8s` 中都是使用的 `X.509` 格式的证书。

> [!NOTE]
> 在 `cfssl` 工具中，需要创建一个生成 `ca` 根证书的配置文件。格式为 `json`。
>   

>   

> `signing` 参数表示可以签发不同的证书  

> `expiry` 参数表示过期时间  

> `profile` 用于生成证书的时候指定的配置  

> `server auth` 参数表示用于 `server` 端的证书  

> `client auth` 参数表示用于 `client` 端的证书  

> `server auth` / `client auth` 同时指定表示两端都可以使用

示例如下：

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

# 3.`etcd` 所需证书

`etcd` 集群中的证书就比较简单。在上面提到的，每个 `etcd` 节点对外提供服务，所以需要 `server`
证书；每个节点对于其他节点来说，也是客户端，所以需要 `peer` 对等证书。
  

如果有其他客户端需要访问 `etcd` 服务，例如 `kube-apiserver` ，就需要为客户端颁发 `client` 证书。
  

同时，每个证书都需要一个 `ca` 根证书来签发。
  

  

所以，在 `etcd` 集群服务中，有以下三种证书：

- 自身的 `server` 证书
- 集群每个节点的 `peer` 双向对等证书
- 颁发给客户端的 `client` 证书。例如：`kube-apiserver`

# 4.`k8s`  所需证书

在 `k8s` 中，让人感到费劲的地方不在于 `ssl` 证书的概念不好理解。  

而是在于在 `k8s` 中，所有服务必须是要使用 `ssl` 证书才能正常运行。
  

又因为 `k8s` 拥有足够的灵活性，每个组件都可以单独拆分部署，一旦服务单独部署，复杂度就升上去了。
  

  

这跟在日常的开发中，一个项目从单体转为微服务架构，管理的复杂度就升上去同个道理。
  

  

**所以，不需要有心智上的负担。要搞清楚 `k8s` 中的的 `ssl` 证书就必须要理清楚整个 `k8s`
有哪些服务组件，各个组件是怎么工作的。这跟理解微服务是一样的。**
  

**所以学习 `k8s` ，直接撸二进制部署是最好的学习 `k8s`
，因为每个不同的服务组件谷歌都提供了二进制文件进行部署。**

## 4-1.`k8s` 中的服务以及组件

在 `k8s` 中，涉及到 `ssl` 的服务或者组件主要有以下

- `etcd` 组件
- `master` 节点的 `kube-apiserver` 组件，`kube-controller-manager` 组件，`kube-scheduler` 组件
- `node` 节点的 `kubelet` 组件以及 `kube-proxy` 组件
- `aggregator-proxy` 服务(聚合层的应用这里没有部署)
- `k8s` 的证书自动申请签发服务
- `calico-cni` 网络插件(`calico` 官方跟 `k8s` 推荐的都是 `pod` 部署，可以不涉及证书)
- `coredns` 域名解析插件

## 4-2.`k8s` 中的证书

> [!NOTE]
> 按照 `k8s` 中不同的服务来区别需要哪些 `ca` 签发机构，再按照各个服务为哪些客户端提供服务，就可以搞清楚需要哪些客户端证书。

理解 `k8s` 中的证书前，需要理解有哪些服务以及组件之间通信的过程。因为 `ssl`
就是对通信加密的。只要理清楚了有哪些通信过程，也就搞明白了 `ssl` 证书了。
  

  

所有的通信都跟 `kube-apiserver` 有关，直接看 `kube-apiserver` 参数就能搞明白需要哪些证书了。因为 `kube-apiserver`
的参数都有指定证书文件地址的参数。
  

  

组件参数查看官方文档，地址：(https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/)
  

有关 `PKI` 证书以及要求查看官方文档地址：(https://kubernetes.io/zh-cn/docs/setup/best-practices/certificates/)

## 4-3.`kube-apiserver` 服务

`kube-apiserver` 作为一个单独的服务，所以可以拥有自己的 `ca` 签发机构。使用 `kube-apiserver` 这个服务的客户端有哪些，就签发对应的客户端证书就行。

### 4-3-1.自己的 `ca` 根证书

`kube-apiserver` 作为独立的服务，所以可以自建 `ca` 签发机构。
  

  

生成的 `ca` 证书需要使用以下参数指定：

```text
--client-ca-file string
# 如果已设置，则使用与客户端证书的 CommonName 对应的标识对任何出示由 client-ca 文件中的授权机构之一签名的客户端证书的请求进行身份验证。
```

### 4-3-2.自身的 `server` 证书

`kube-apiserver` 作为服务，所以本身需要自身的 `server` 端证书

配置 `server` 证书的参数如下：

```text
--tls-cert-file string
# 包含用于 HTTPS 的默认 x509 证书的文件。（CA 证书（如果有）在服务器证书之后并置）。
# 如果启用了 HTTPS 服务，并且未提供 --tls-cert-file 和 --tls-private-key-file， 为公共地址生成一个自签名证书和密钥，并将其保存到 --cert-dir 指定的目录中。

--tls-private-key-file string
# 包含匹配 --tls-cert-file 的 x509 证书私钥的文件。
```

### 4-3-3.`kube-controller-manager` 的 `client` 证书

`kube-controller-manager` 作为 `kube-apiserver` 的客户端调用 `kube-apiserver` 的服务。所以需要 `kube-apiserver` 的 `ca`
机构为其颁发 `client` 证书。
  

  

`kube-controller-manager` 是使用 `kubeconfig` 的方式访问 `kube-apiserver` 的服务。
  

在设置 `kubeconfig`
的时候需要指定 `kube-apiserver` 的 `ca` 证书的地址以及 `kube-controller-manager` 的 `client` 证书的地址。
  

  

`kubeconfig` 相关参数如下：
  

*设置集群参数指定 `ca` 证书地址*

```text
--certificate-authority
```

*设置客户端认证参数指定 `client` 证书地址*

```text
--client-certificate
--client-key
```

### 4-3-4.`kube-scheduler` 的 `client` 证书

`kube-scheduler` 作为 `kube-apiserver` 的客户端调用 `kube-apiserver` 的服务。所以需要 `kube-apiserver` 的 `ca`
机构为其颁发 `client` 证书。
  

  

`kube-scheduler` 也是使用 `kubeconfig` 的方式访问 `kube-apiserver` 的服务。
  

在设置 `kubeconfig` 的时候同样需要指定 `kube-apiserver` 的 `ca` 证书的地址以及 `kube-scheduler` 的 `client` 的证书地址。
  

  

`kubeconfig` 相关参数如下：
  

*设置集群参数指定 `ca` 证书地址*

```text
--certificate-authority
```

*设置客户端认证指定 `client` 证书地址*

```text
--client-certificate
--client-key
```

## 5.`kubelet` 服务

`kubelet` 同样作为一个单独的服务，所以可以拥有自己的 `ca` 签发机构。对于 `kubelet` ，只有 `kube-apiserver`
这个客户端，只需要为其签发客户端 `client` 证书就行了。
  

那么在 `kubelet` 这个服务中，需要以下四种证书：

- `kubelet` 自建的 `ca` 根证书
- `kubelet` 自身 `server` 端证书
- `kubelet` 访问 `kube-apiserver` 的 `client` 证书
- `kube-apiserver` 访问 `kubelet` 的 `client` 证书

但是在 `kubelet` 中，存在一个证书管理的问题的。
  

该问题就是，`kubelet` 作为 `node` 节点上的服务。`node`
节点会随着服务的增加而增加，或者经常变动和删除节点。
  

如果手动为每个节点颁发 `server` 端证书以及创建 `kubelet` 客户端 `client` 证书，那么当 `node`
的节点越来越多的时候，证书将变得难以管理。
  

  

所以 `k8s` 在 `1.4` 版本中，引入了 `TLS Bootstrap` 自动引导颁发证书的功能。

### 5-1.手动颁发证书

> **注意：配置了手动颁发证书的参数后，自签名证书的参数将失效。**

手动颁发证书涉及到的参数如下：
  

`kubelet` 的配置参数：

```text
tlsCertFile string
# tlsCertFile是包含 HTTPS 所需要的 x509 证书的文件 （如果有 CA 证书，会串接到服务器证书之后）。
# 如果tlsCertFile 和tlsPrivateKeyFile都没有设置，则系统会为节点的公开地址生成自签名的证书和私钥， 并将其保存到 kubelet --cert-dir参数所指定的目录下。

tlsPrivateKeyFile string
# tlsPrivateKeyFile是一个包含与tlsCertFile 证书匹配的 X509 私钥的文件。

authentication #authorization设置发送给 kubelet 服务器的请求是如何进行身份认证的。
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: # kubelet 的 ca 根证书的地址

authorization #authorization设置发送给 kubelet 服务器的请求是如何进行鉴权的。
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: "5m"
    cacheUnauthorizedTTL: "30s"
```

`kube-apiserver` 的配置参数：

```text
--kubelet-certificate-authority string
# 证书颁发机构的证书文件的路径。

--kubelet-client-certificate string
# TLS 的客户端证书文件的路径。

--kubelet-client-key string
# TLS 客户端密钥文件的路径。
```

### 5-2.自签名证书

自签名就是手动颁发证书的自动化。跟手动颁发证书的区别在于，不需要指定 `ca` 机构以及 `kubelet` 的 `tls` 证书参数。
  

  

注释掉 `kubelet` 的配置

```yaml
# tlsCertFile string
# tlsPrivateKeyFile string
```

删除掉 `kube-apiserver` 的配置
  

*是删除，不是注释*

```shell
--kubelet-certificate-authority string
```

### 5-3.`TLS Bootstrap` 自动颁发证书

在 `TLS Bootstrap` 自动引导颁发证书中，`kubelet` 的客户端是由 `kube-apiserver` 颁发的。
  

  

`TLS Bootsrap` 涉及到的地方比较多，后面部署会有详细操作过程。

## 6.`aggregator-proxy` 服务

> [!NOTE]
> 即聚合层的服务，这里没部署该服务，后期有用到再补充。

## 7.`service-account` 服务

> [!NOTE]
> 该文档部署使用的是 `kube-apiserver` 的 `ca` 机构作为 `sa` 服务的 `ca` 机构。
>   

> 要把 `sa` 服务的 `ca` 独立出来，就跟 `kube-apiserver` 一样的自己创建一个 `ca`。然后参数指定证书地址就就行。
>   

> 这里不再折腾研究了，太特么折腾了。后期有空再研究验证后补充。

## 8.`k8s` 的证书申请服务

> [!NOTE]
> 证书申请服务，即 `sign/client` 服务。创建 `ca` 同 `sa` 一样。这里也不再折腾研究了，后期有空再补上来，太特么折腾了。
>   

> 涉及到到参数如下，非常多，折腾一遍脑袋瓜直接炸裂。
>   

>   

> 该文档二进制部署同样使用的是 `kube-apiserver` 的 `ca` 机构。

```text
--cluster-signing-cert-file string
# 包含 PEM 编码格式的 X509 CA 证书的文件名。该证书用来发放集群范围的证书。 如果设置了此标志，则不能指定更具体的--cluster-signing-* 标志。

--cluster-signing-duration duration     默认值：8760h0m0s
# 所签名证书的有效期限。每个 CSR 可以通过设置 spec.expirationSeconds 来请求更短的证书。

--cluster-signing-key-file string
# 包含 PEM 编码的 RSA 或 ECDSA 私钥的文件名。该私钥用来对集群范围证书签名。 若指定了此选项，则不可再设置 --cluster-signing-* 参数。

--cluster-signing-kube-apiserver-client-cert-file string
# 包含 PEM 编码的 X509 CA 证书的文件名， 该证书用于为 kubernetes.io/kube-apiserver-client 签署者颁发证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-kube-apiserver-client-key-file string
# 包含 PEM 编码的 RSA 或 ECDSA 私钥的文件名， 该私钥用于为 kubernetes.io/kube-apiserver-client 签署者签名证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-kubelet-client-cert-file string
# 包含 PEM 编码的 X509 CA 证书的文件名， 该证书用于为 kubernetes.io/kube-apiserver-client-kubelet 签署者颁发证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-kubelet-client-key-file string
# 包含 PEM 编码的 RSA 或 ECDSA 私钥的文件名， 该私钥用于为 kubernetes.io/kube-apiserver-client-kubelet 签署者签名证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-kubelet-serving-cert-file string
# 包含 PEM 编码的 X509 CA 证书的文件名， 该证书用于为 kubernetes.io/kubelet-serving 签署者颁发证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-kubelet-serving-key-file string
# 包含 PEM 编码的 RSA或ECDSA 私钥的文件名， 该私钥用于对 kubernetes.io/kubelet-serving 签署者的证书进行签名。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-legacy-unknown-cert-file string
# 包含 PEM 编码的 X509 CA 证书的文件名， 用于为 kubernetes.io/legacy-unknown 签署者颁发证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。

--cluster-signing-legacy-unknown-key-file string
# 包含 PEM 编码的 RSA 或 ECDSA 私钥的文件名， 用于为 kubernetes.io/legacy-unknown 签署者签名证书。
# 如果指定，则不得设置 --cluster-signing-{cert,key}-file。
```

> **看完之后是不是有一种头皮发麻，原地高潮到赶脚。**

## 9.`etcd` 服务

`etcd` 作为一个单独的服务，所以可以拥有自己的 `ca` 签发机构。
  

使用 `etcd` 这个服务的客户端只有 `kube-apiserver` ，所以除了以上提到的
每个 `etcd` 节点自身需要的 `server` 证书以及 `peer` 对等证书外，
  

还需要生成一个 `client` 证书提供给 `kube-apiserver` 使用。
  

  

配置该证书的 `kube-apiserver` 参数为：

```text
--etcd-cafile
# 用于保护 etcd 通信的 SSL 证书颁发机构文件。

--etcd-certfile
# 用于保护 etcd 通信的 SSL 证书文件。

--etcd-keyfile
# 用于保护 etcd 通信的 SSL 密钥文件。
```

***
