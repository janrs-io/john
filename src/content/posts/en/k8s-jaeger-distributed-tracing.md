---
title: "Distributed Tracing Infrastructure with Jaeger on Kubernetes"
author: "John Yang"
lang: en
pubDatetime: 2023-03-02T14:49:00Z
tags:
  - Kubernetes
  - Observability
  - Jaeger
description: "Deploying Jaeger on Kubernetes for distributed tracing across gRPC microservices: Go instrumentation, Sidecar vs. DaemonSet agents, Collector, Query, and Prometheus monitoring."
---

As an integral part of any distributed system, observability infrastructure cannot be overstated. Monitoring goes beyond binary up/down checks—it gives visibility into complex system behavior, latency, service health, and long-term performance trends.

This post focuses on one pillar of observability infrastructure: **distributed tracing**.

# Observability in Microservices

Kubernetes has become the de facto orchestrator for microservice workloads. Its rich ecosystem pairs cleanly with an observability stack built around Prometheus, Elasticsearch, Grafana, Envoy/Consul, and Jaeger/Zipkin to cover metrics, logs, dashboards, service discovery, and distributed tracing.

# Distributed Tracing

Distributed tracing captures requests end-to-end and reconstructs the full call graph across hundreds of services. It measures per-request latency, tracks the lifecycle of HTTP/RPC network calls, and pinpoints performance bottlenecks across service boundaries.

***

The sections below walk through setting up distributed tracing for gRPC services using **Jaeger** on **Kubernetes**, breaking down each Jaeger component and its Kubernetes deployment manifest.

# Jaeger Components

Jaeger is an open-source distributed tracing system implementing the OpenTracing specification. It includes components for trace collection, storage, filtering, and visualization.

## Jaeger Client

Application instrumentation starts with the Jaeger client. Below is an example using the Jaeger Go client library initialized from environment variables with Prometheus metrics enabled:

```go
package tracer

import (
    "io"

    "github.com/uber/jaeger-client-go/config"
    jprom "github.com/uber/jaeger-lib/metrics/prometheus"
)

func NewTracer() (opentracing.Tracer, io.Closer, error) {
    // load config from environment variables
    cfg, _ := jaegercfg.FromEnv()

    // create tracer from config
    return cfg.NewTracer(
        config.Metrics(jprom.New()),
    )
}
```

The Go client makes configuration via environment variables straightforward—notably `JAEGER_SERVICE_NAME`, `JAEGER_AGENT_HOST`, and `JAEGER_AGENT_PORT`. See the full list of supported variables in [jaeger-client-go](https://github.com/jaegertracing/jaeger-client-go "jaeger-client-go").

To instrument gRPC microservices, we use `grpc-ecosystem/go-grpc-middleware`, which provides OpenTracing interceptors for both gRPC servers and clients.

The `grpc_opentracing` package exposes interceptors that work with any `opentracing.Tracer` implementation. Here we configure a gRPC server with chained unary and stream interceptors so every incoming RPC gets a root `serverSpan` attached to its context:

```go
package grpc_server

import (
	"github.com/opentracing/opentracing-go"
	"github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"
	"github.com/grpc-ecosystem/go-grpc-middleware"
	"google.golang.org/grpc"

  	"github.com/masroorhasan/myapp/tracer"
)

func NewServer() (*grpc.Server, error) {
 	// initialize tracer
	tracer, closer, err := tracer.NewTracer()
	defer closer.Close()
	if err != nil {
		return &grpc.Server{}, err
	}
	opentracing.SetGlobalTracer(tracer)

	// initialize grpc server with chained interceptors
	s := grpc.NewServer(
		grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
			// add opentracing stream interceptor to chain
			grpc_opentracing.StreamServerInterceptor(grpc_opentracing.WithTracer(tracer)),
  		)),
	  	grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
			// add opentracing unary interceptor to chain
			grpc_opentracing.UnaryServerInterceptor(grpc_opentracing.WithTracer(tracer)),
		)),
	)
	return s, nil
}
```

To propagate traces across upstream and downstream gRPC calls, initialize the gRPC client with the corresponding client-side OpenTracing interceptors as well:

```go
package grpc_client

import (
    "github.com/opentracing/opentracing-go"
    "github.com/grpc-ecosystem/go-grpc-middleware/tracing/opentracing"
    "github.com/grpc-ecosystem/go-grpc-middleware"
    "google.golang.org/grpc"

    "github.com/masroorhasan/myapp/tracer"
)

func NewClientConn(address string) (*grpc.ClientConn, error) {
    // initialize tracer
    tracer, closer, err := tracer.NewTracer()
    defer closer.Close()
    if err != nil {
        return &grpc.ClientConn{}, err
    }

    // initialize client with tracing interceptor using grpc client side chaining
    return grpc.Dial(
        address,
        grpc.WithStreamInterceptor(grpc_middleware.ChainStreamClient(
            grpc_opentracing.StreamClientInterceptor(grpc_opentracing.WithTracer(tracer)),
        )),
        grpc.WithUnaryInterceptor(grpc_middleware.ChainUnaryClient(
            grpc_opentracing.UnaryClientInterceptor(grpc_opentracing.WithTracer(tracer)),
        )),
     )
}
```

The parent span created by the gRPC middleware is injected into Go's `context.Context`, enabling child spans, fine-grained span lifetimes, and custom tags downstream.

## Jaeger Agent

The Jaeger Agent is a daemon that listens for spans from Jaeger clients over UDP and forwards them in batches to the Collector—abstracting batching and routing away from application code.

On Kubernetes, the Agent can run either as a **sidecar container** inside each application Pod or as a **node-level DaemonSet**. Below are the trade-offs of each pattern.

### Jaeger Sidecar Agent

A sidecar agent runs alongside your application container in the same Pod. The application (`myapp`) sends spans over `localhost` on UDP port `6831`, configured via `JAEGER_SERVICE_NAME`, `JAEGER_AGENT_HOST`, and `JAEGER_AGENT_PORT`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: masroorhasan/myapp
        ports:
        - containerPort: 80
        env:
        - name: JAEGER_SERVICE_NAME
          value: myapp
        - name: JAEGER_AGENT_HOST
          value: localhost  # default
        - name: JAEGER_AGENT_PORT
          value: "6831"
        resources:
          limits:
            memory: 500M
            cpu: 250m
          requests:
            memory: 500M
            cpu: 250m
      # sidecar agent
      - name: jaeger-agent
        image: jaegertracing/jaeger-agent:1.6.0
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        command:
          - "/go/bin/agent-linux"
          - "--collector.host-port=jaeger-collector.monitoring:14267"
        resources:
          limits:
            memory: 50M
            cpu: 100m
          requests:
            memory: 50M
            cpu: 100m
```

Sidecars allow per-app routing to different collectors/backends, but they tightly couple the agent's lifecycle to the Pod. During Pod termination, the sidecar may exit before the application finishes flushing spans, losing critical shutdown traces. See [this GitHub issue](https://github.com/jaegertracing/jaeger/issues/295 "GitHub issue") for details on handling `SIGTERM` cleanly.

### Jaeger DaemonSet Agent

Alternatively, deploy the agent as a Kubernetes `DaemonSet` so one agent Pod runs on every cluster node.

Applications set `JAEGER_AGENT_HOST` to `status.hostIP` via the Kubernetes Downward API. The DaemonSet uses `hostNetwork: true` with `dnsPolicy: ClusterFirstWithHostNet` and exposes `hostPort: 6831` over UDP:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  namespace: default
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: masroorhasan/myapp
        ports:
        - containerPort: 80
        env:
        - name: JAEGER_SERVICE_NAME
          value: myapp
        - name: JAEGER_AGENT_HOST   # NOTE: Point to the Agent daemon on the Node
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: JAEGER_AGENT_PORT
          value: "6831"
        resources:
          limits:
            memory: 500M
            cpu: 250m
          requests:
            memory: 500M
            cpu: 250m
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: jaeger-agent
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: agent-daemonset
spec:
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: agent-instance
    spec:
      hostNetwork: true     # NOTE: Agent is configured to have same IP as the host/node
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: agent-instance
        image: jaegertracing/jaeger-agent:1.6.0
        command:
          - "/go/bin/agent-linux"
          - "--collector.host-port=jaeger-collector.monitoring:14267"
          - "--processor.jaeger-binary.server-queue-size=2000"
          - "--discovery.conn-check-timeout=500ms"
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 6831
          protocol: UDP
          hostPort: 6831
        - containerPort: 6832
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        resources:
          requests:
            memory: 200M
            cpu: 200m
          limits:
            memory: 200M
            cpu: 200m
```

It can be tempting to put a Kubernetes `Service` in front of the Agent `DaemonSet` to spread spans across nodes. However, under traffic spikes, sending UDP Thrift packets over the cluster network rather than node-local loopback often causes UDP packet size overflows and dropped spans.

Instead, size node resources properly for even Pod scheduling, increase client buffer capacity (`JAEGER_REPORTER_MAX_QUEUE_SIZE`), and tune `--processor.jaeger-binary.server-queue-size` on the agent.

## Jaeger Collector

The Jaeger Collector receives batched spans from agents over TChannel/TCP on port `14267`, runs them through its processing pipeline, and writes them to the storage backend.

Because the Collector is stateless, it scales horizontally behind a standard Kubernetes `ClusterIP` Service:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jaeger-collector
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: collector-deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: collector-pod
    spec:
      containers:
      - image: jaegertracing/jaeger-collector:1.6.0
        name: jaeger-collector
        args: ["--config-file=/conf/collector.yaml"]
        ports:
        - containerPort: 14267
          protocol: TCP
        - containerPort: 14268
          protocol: TCP
        - containerPort: 9411
          protocol: TCP
        readinessProbe:
          httpGet:
            path: "/"
            port: 14269
        volumeMounts:
        - name: jaeger-configuration-volume
          mountPath: /conf
        env:
        - name: SPAN_STORAGE_TYPE
          valueFrom:
            configMapKeyRef:
              name: jaeger-configuration
              key: span-storage-type
      volumes:
        - configMap:
            name: jaeger-configuration
            items:
              - key: collector
                path: collector.yaml
          name: jaeger-configuration-volume
      resources:
        requests:
          memory: 300M
          cpu: 250m
        limits:
          memory: 300M
          cpu: 250m
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: collector-service
spec:
  ports:
  - name: jaeger-collector-tchannel
    port: 14267
    protocol: TCP
    targetPort: 14267
  selector:
    jaeger-infra: collector-pod
  type: ClusterIP
```

## Jaeger Query

Jaeger Query powers the web UI, retrieving and formatting traces from storage with a minimal resource footprint. Expose it internally via a Kubernetes `Ingress`:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jaeger-query
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: query-deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jaeger
        jaeger-infra: query-pod
    spec:
      containers:
      - image: jaegertracing/jaeger-query:1.6.0
        name: jaeger-query
        args: ["--config-file=/conf/query.yaml"]
        ports:
        - containerPort: 16686
          protocol: TCP
        readinessProbe:
          httpGet:
            path: "/"
            port: 16687
        volumeMounts:
        - name: jaeger-configuration-volume
          mountPath: /conf
        env:
        - name: SPAN_STORAGE_TYPE
          valueFrom:
            configMapKeyRef:
              name: jaeger-configuration
              key: span-storage-type
        resources:
          requests:
            memory: 100M
            cpu: 100m
          limits:
            memory: 100M
            cpu: 100m
      volumes:
        - configMap:
            name: jaeger-configuration
            items:
              - key: query
                path: query.yaml
          name: jaeger-configuration-volume
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: query-service
spec:
  ports:
  - name: jaeger-query
    port: 16686
    targetPort: 16686
  selector:
    jaeger-infra: query-pod
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: jaeger-ui
 namespace: monitoring
 annotations:
   kubernetes.io/ingress.class: traefik # or nginx or whatever ingress controller
spec:
 rules:
 - host: jaeger.internal-host # your jaeger internal endpoint
   http:
     paths:
     - backend:
         serviceName: jaeger-query
         servicePort: 16686
```

## Storage Configuration

Jaeger supports both Elasticsearch and Cassandra as storage backends. Using Elasticsearch ties traces and structured logs together in Kibana via shared trace IDs. Set `SPAN_STORAGE_TYPE` to `elasticsearch` and provide the endpoints in a `ConfigMap`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-configuration
  namespace: monitoring
  labels:
    app: jaeger
    jaeger-infra: configuration
data:
  span-storage-type: elasticsearch
  collector: |
    es:
      server-urls: http://elasticsearch:9200
    collector:
      zipkin:
        http-port: 9411
  query: |
    es:
      server-urls: http://elasticsearch:9200
```

***

# Monitoring Jaeger Itself

Your tracing pipeline is part of your observability stack, so its own components should also be monitored.

Jaeger exposes Prometheus metrics on component-specific ports (`5778` for Agent, `14269` / `14628` for Collector, `16687` / `16686` for Query—see [Jaeger Monitoring Docs](https://www.jaegertracing.io/docs/1.42/monitoring/ "Jaeger Monitoring Docs")).

Useful PromQL queries to track:

- **Memory usage per component**:
  `sum(rate(container_memory_usage_bytes{container_name=~"^jaeger-.+"}[1m])) by (pod_name)`
- **CPU usage per component**:
  `sum(rate(container_cpu_usage_seconds_total{container_name=~"^jaeger-.+"}[1m])) by (pod_name)`
- **Agent batch failures**:
  `sum(rate(jaeger_agent_tc_reporter_jaeger_batches_failures[1m])) by (pod)`
- **Spans dropped by Collector**:
  `sum(rate(jaeger_collector_spans_dropped[1m])) by (pod)`
- **Collector queue latency (p95)**:
  `histogram_quantile(0.95, sum(rate(jaeger_collector_in_queue_latency_bucket[1m])) by (le, pod))`

***
