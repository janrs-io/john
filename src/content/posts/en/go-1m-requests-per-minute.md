---
title: "Architecture Notes: Handling 1M Requests/Min with Go"
author: "John Yang"
lang: en
pubDatetime: 2023-04-11T01:26:00Z
featured: true
tags:
  - Go
  - Concurrency
  - Architecture
description: "Concise architecture notes on handling 1 million requests per minute in Go using bounded worker pools and buffered channels, inspired by Marcio Castilho."
---

> [!NOTE] Reference & Attribution
> These are concise architecture notes based on **Marcio Castilho**'s classic 2015 engineering post: [*Handling 1 Million Requests per Minute with Golang*](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/) (`marcio.io`). Rather than reproducing the original article verbatim, this post distills the core architectural lessons for high-throughput Go services.

## The Problem: Ingestion at Scale

When building an analytics ingestion pipeline that receives millions of HTTP `POST` requests carrying JSON payloads—each needing to be persisted to object storage (such as Amazon S3) for downstream batch processing—traditional stacks often split into two tiers:

- A **web frontend cluster** handling HTTP requests
- A separate **background worker cluster** backed by an external queue (Sidekiq, Resque, RabbitMQ, SQS, etc.)

While reliable, external queue hops add operational overhead, serialization cost, and infrastructure footprint when the workload is primarily I/O-bound fan-out.

## Why Naive `go handle(payload)` Fails Under Load

Spawning an unbounded goroutine per incoming HTTP request works well at moderate traffic, but collapses during traffic spikes or slow downstream I/O (e.g., S3 latency):

- Goroutines are cheap (~2 KB initial stack), not free.
- Millions of concurrent requests hitting slow downstream endpoints create unbounded goroutine growth, rising memory pressure, GC pauses, and file-descriptor/socket exhaustion.
- Backpressure disappears: the server keeps accepting work faster than downstream storage can drain it.

## Core Pattern: Bounded Worker Pool + Buffered Job Channel

Instead of spawning a goroutine per request or maintaining an external message broker, the architecture introduces an in-process **bounded worker pool** decoupled by a buffered channel:

1. **HTTP Handler (Producer)**: Decodes/validates the incoming payload, pushes a `Job` struct onto a global or injected buffered `JobQueue`, and returns `200 OK` immediately without blocking on S3 upload.
2. **Dispatcher**: Reads jobs from `JobQueue` and routes each job to an available worker channel pulled from a `WorkerPool` (`chan chan Job`).
3. **Fixed Worker Pool (Consumers)**: A fixed number (`MaxWorker`) of long-lived goroutines register their personal job channel back into `WorkerPool`, wait for a `Job`, upload to S3, and loop.

```
HTTP Request -> Handler -> [ Buffered JobQueue ] -> Dispatcher -> [ WorkerPool ] -> Worker (S3 Upload)
```

## Key Architectural Takeaways

- **Predictable Resource Ceiling**: Capping both `MaxWorkers` and `MaxQueue` bounds memory usage and active outbound connections regardless of burst traffic.
- **Explicit Backpressure**: When `JobQueue` fills up, the service can shed load cleanly (e.g., returning `503 Service Unavailable`) or block with a timeout instead of OOM-crashing.
- **Eliminate Unnecessary Network Hops**: For stateless, fire-and-forget ingestion tasks where brief in-memory buffering is acceptable, Go's CSP primitives (`goroutines` + `channels`) replace an entire external queue tier.
- **Keep Per-Request Allocation Minimal**: Reusing buffers, avoiding unnecessary JSON re-marshaling, and bounding concurrency keep GC overhead low at high RPS.

## When *Not* to Use an In-Memory Pool Alone

An in-memory worker pool trades durability for throughput and simplicity:

- If a node crashes or is terminated mid-flight, jobs sitting in `JobQueue` are lost unless drained on `SIGTERM` (graceful shutdown) or backed by a durable log/queue.
- For strict at-least-once delivery guarantees across restarts, pair this pattern with graceful shutdown draining or an external durable broker.

For the full original walkthrough, benchmarks, and code evolution, read Marcio Castilho's post: [Handling 1 Million Requests per Minute with Golang](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/).
