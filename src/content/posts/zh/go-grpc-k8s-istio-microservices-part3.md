---
title: "基于Go/Grpc/kubernetes/Istio开发微服务的最佳实践尝试 - 3/3"
author: "John Yang"
lang: zh
pubDatetime: 2023-03-31T06:33:00Z
tags:
  - Go
  - gRPC
  - Kubernetes
  - Istio
  - Microservices
description: "在前两部分中，我们创建了两个微服务：pingservice 和 pongservice。 在这一部分中，我们将创建用于自动部署的 CICD pipeline。 我们假设您已经部署了Jenkins/Gitlab/Harbor 和Kubenertes/Istio。 在 Jenkins 上，为每个微服务项目创..."
---

在前两部分中，我们创建了两个微服务：`pingservice` 和 `pongservice`。 在这一部分中，我们将创建用于自动部署的 `CICD pipeline`。

我们假设您已经部署了`Jenkins/Gitlab/Harbor` 和`Kubenertes/Istio`。

# 项目结构

```bash
devops
├── README.md
├── ping
│   └── dev
│       ├── Deployment.yaml
│       ├── Dockerfile
│       ├── Jenkinsfile
│       └── Service.yaml
└── pong
    └── dev
        ├── Deployment.yaml
        ├── Dockerfile
        ├── Jenkinsfile
        └── Service.yaml

4 directories, 9 files
```

# 实践

在 `Jenkins` 上，为每个微服务项目创建一个目录，然后在该目录下创建`dev/test/prod`流水线。

在 `Gitlab` 上，设置三个分支保护分支：`dev/test/prod`。 这三个分支用于`dev/test/production`环境。这三个分支只能合并不能提交。

如果有新的微服务要开发，在 `dev` 分支的基础上新建一个分支，名称格式为：`dev-*`。 例如：`dev-ping、dev-pong`。
然后为每个分支设置 `webhook` ，自动触发 `Jenkins pipeline` 自动部署到 `kubernetes` 集群。

微服务本地开发需要调试. 可以使用 [kubefwd](https://github.com/txn2/kubefwd "kubefwd") 工具或者`kubernetes` 官方推荐的 [telepresence](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/local-debugging/ "telepresence") 。

# 大型开发实践

如果你的公司发展到集团化规模，需要异地协同开发，可以将`devops`、`istio-manifests`、`kubernetes-manifests`分开，创建一个独立的 `git-repo` 进行管理。

并且还可以在`src/`目录下将不同的微服务分开，创建不同的 `git-repos` 进行管理。

不同团队需要将开发好的 `grpc` 接口文档化并发布到网上，所有人员根据网上的接口文档进行开发调试。

# 相关项目和资料

感谢以下资源的贡献者：

- [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo "GoogleCloudPlatform / microservices-demo")

- [GitOps Decisions: Creating a Mature Pipeline](https://blog.container-solutions.com/gitops-decisions "GitOps Decisions: Creating a Mature Pipeline")

- [buf build](https://buf.build/ "buf build")

- [wire](https://github.com/google/wire "wire")

- [gRPC Ecosystem](https://github.com/grpc-ecosystem "gRPC Ecosystem")

***
