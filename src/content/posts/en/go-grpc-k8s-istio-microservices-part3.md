---
title: "Microservices Best Practices with Go, gRPC, Kubernetes, and Istio — Part 3/3"
author: "John Yang"
lang: en
pubDatetime: 2023-03-31T06:33:00Z
tags:
  - Go
  - gRPC
  - Kubernetes
  - Istio
  - Microservices
description: "Part 3 of the Go/gRPC/Kubernetes/Istio series: setting up Jenkins/GitLab/Harbor CI/CD pipelines, Git branching workflows, and multi-team scaling."
---

In Parts 1 and 2, we built two microservices: `pongservice` and `pingservice`. In this final part, we set up automated CI/CD pipelines for deployment.

This guide assumes you already have **Jenkins**, **GitLab**, **Harbor**, **Kubernetes**, and **Istio** deployed.

## Project Structure

```bash
devops
├── README.md
├── ping
│   └── dev
│       ├── Deployment.yaml
│       ├── Dockerfile
│       ├── Jenkinsfile
│       └── Service.yaml
└── pong
    └── dev
        ├── Deployment.yaml
        ├── Dockerfile
        ├── Jenkinsfile
        └── Service.yaml

4 directories, 9 files
```

## CI/CD Workflow in Practice

- In **Jenkins**, create a folder for each microservice and configure `dev`, `test`, and `prod` pipelines inside it.
- In **GitLab**, protect three environment branches: `dev`, `test`, and `prod` (merge-only, no direct commits).
- When developing a new feature or service, branch off `dev` using a naming convention such as `dev-*` (e.g., `dev-ping`, `dev-pong`).
- Configure webhooks on each environment branch to automatically trigger the corresponding Jenkins pipeline and deploy to Kubernetes.

For local microservice debugging against a remote cluster, tools like [kubefwd](https://github.com/txn2/kubefwd "kubefwd") or Kubernetes' recommended [telepresence](https://kubernetes.io/docs/tasks/debug/debug-cluster/local-debugging/ "telepresence") work well.

## Scaling for Larger Teams

If your organization grows across multiple distributed teams, split `devops`, `istio-manifests`, and `kubernetes-manifests` into dedicated Git repositories.

You can also move each microservice under `src/` into its own independent repository, and publish generated gRPC/OpenAPI documentation online so teams can integrate and test against shared contracts without coupling release cycles.

## Related Projects & References

Thanks to the contributors behind these resources:

- [GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo "GoogleCloudPlatform / microservices-demo")
- [GitOps Decisions: Creating a Mature Pipeline](https://blog.container-solutions.com/gitops-decisions "GitOps Decisions: Creating a Mature Pipeline")
- [buf build](https://buf.build/ "buf build")
- [wire](https://github.com/google/wire "wire")
- [gRPC Ecosystem](https://github.com/grpc-ecosystem "gRPC Ecosystem")

***
