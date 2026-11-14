---
title: "Google Releases Service Weaver: A Framework for Distributed Applications"
author: "John Yang"
lang: en
pubDatetime: 2023-03-07T01:50:00Z
tags:
  - Go
  - Microservices
  - Architecture
description: "An overview of Google's Service Weaver framework for Go, letting you write applications as a modular monolith locally and deploy them as distributed microservices."
---

> A framework that runs as a modular monolith locally and deploys as a distributed microservices architecture.

GitHub Repository: [https://github.com/ServiceWeaver/weaver](https://github.com/ServiceWeaver/weaver "https://github.com/ServiceWeaver/weaver")

*The debate between monoliths and microservices seems endless.*

Depending on who you ask and what they have built, you will get a different answer every time. In practice, the right choice depends on team size, traffic volume, and product complexity. Both approaches have clear trade-offs—but what if you could get the best of both worlds? That is exactly what Google's open-source framework aims to offer. Let's take a closer look.

## What Is Service Weaver?

Service Weaver is an open-source framework created by Google (currently focused on Go, though the model could extend to other languages).

It is designed for building distributed applications: during local development it runs as a **modular monolith**, and when deployed to production it runs as a **distributed microservices architecture**.

## What Is a Modular Monolith?

A modular monolith is an architecture where the entire application lives in a single codebase and repository, but is divided into independent components with clean, well-defined interfaces between them.

For example, consider an application with three components: Orders, Payments, and Shipping. Each component encapsulates a specific domain, keeps its implementation details private, and communicates with other components strictly through explicit interfaces.

As long as those interfaces remain intact, teams can modify and evolve each component's internals independently—creating clear boundaries and explicit dependencies across teams.

In a traditional modular monolith, however, deployment still ships everything as a single binary/process—for example, one process per EC2 instance.

## How Does Service Weaver Differ from a Traditional Modular Monolith?

During development, a Service Weaver application looks very similar to a modular monolith: you build components in a single repository and define explicit interfaces for inter-component communication.

The difference lies in **deployment**. Instead of deploying everything into one monolithic process on the same machine, Service Weaver can deploy components separately as independent microservices. You get the developer ergonomics of a single repo and local debugging, alongside the operational flexibility of scaling individual components independently by CPU, memory, and replica count.

## How Does Service Weaver Work?

Service Weaver is written in Go. Every component is defined by a Go interface—acting as its public API for other components. For example, a component that reverses strings might look like this:

```go
type Reverser interface {
    Reverse(context.Context, string) (string, error)
}
```

Any component that needs to reverse a string calls this `Reverser` interface, while the underlying implementation remains private inside the `Reverser` component.

Locally, you build and test your code as usual: Service Weaver handles interactions between components as standard local method calls.

When deployed as separate microservices, however, cross-component calls become remote procedure calls (RPCs). Under the hood, Service Weaver uses Protocol Buffers to serialize and deserialize data across the network—completely transparently. Your application code simply calls `Reverse(...)` without caring whether the target component runs in the same process or across the network.

## Structuring Microservices with Service Weaver

One classic pain point of traditional microservices is ending up with overly chatty interfaces between two services as requirements evolve. Fixing that usually forces you either to tolerate added network latency and failure modes, or to spend significant effort merging the two services back together.

Service Weaver solves this cleanly through configuration. Suppose you define four components (`A`, `B`, `C`, and `D`): you can co-locate `A` and `B` inside a single microservice process while running `C` and `D` as their own microservices. If two components become chatty over time, you can colocate them together by updating the configuration—without changing a single line of application code.

## Cloud Deployment Options

Although created by Google and tightly integrated with GCP, Service Weaver is not locked into Google Cloud—it can target other clouds such as AWS or Azure as well. Deployment behavior is configured via clean TOML files.

You can run locally with `go run`, or deploy to the cloud with `weaver gke deploy` (using Kubernetes under the hood to manage component placement and communication).

## Getting Started

If you want to try it out, visit the official documentation at [serviceweaver.dev](https://serviceweaver.dev/docs.html "serviceweaver.dev"), which covers the architecture, installation, and a quick Hello World walkthrough.

Overall, Service Weaver offers a compelling middle ground between monoliths and microservices—solving many of the hardest operational and refactoring headaches of distributed systems.
