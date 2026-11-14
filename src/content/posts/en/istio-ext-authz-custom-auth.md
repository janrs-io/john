---
title: "Implementing Custom External Auth in Istio with ext-authz for Kubernetes Microservices"
author: "John Yang"
lang: en
pubDatetime: 2023-06-01T05:39:00Z
featured: true
tags:
  - Istio
  - Kubernetes
  - Security
  - Microservices
description: "How to implement a gRPC external authorization service in Go using Envoy ext-authz v3 and Redis token validation, wired into Istio via AuthorizationPolicy."
---

This approach enables Redis-backed token authentication and RBAC authorization at the Istio ingress/mesh layer.

***

Istio's external authorization is built on top of Envoy's `ext_authz` filter. You can inspect the Envoy protobuf definition and Istio's official demo here:

- Envoy `external_auth.proto`: [GitHub](https://github.com/envoyproxy/envoy/blob/e98e41a8e168af7acae8079fc0cd68155f699aa3/api/envoy/service/auth/v3/external_auth.proto "Envoy external_auth.proto")
- Istio official `extauthz` sample: [GitHub](https://github.com/istio/istio/tree/master/samples/extauthz "Istio extauthz sample")

## Implementation

Istio supports both HTTP and gRPC external authorization providers. In this guide, we implement the **gRPC** provider.

### 1. Configure Istio Mesh `ConfigMap`

Edit the `istio` `ConfigMap` (usually in `istio-system`) and add an `extensionProviders` entry under `data.mesh`:

```yaml
extensionProviders:
    - name: "rgrpc-dev-authz-grpc-provider"
      envoyExtAuthzGrpc:
        service: "auth.rgrpc-dev.svc.cluster.local"
        port: 50051
```

### 2. Build the Istio gRPC Auth Service

Because Istio delegates `ext_authz` to Envoy, you only need to implement Envoy's gRPC `AuthorizationServer` interface (`Check` method).

Reference `external_auth.pb.go` in `go-control-plane`: [GitHub](https://github.com/envoyproxy/go-control-plane/blob/main/envoy/service/auth/v3/external_auth.pb.go "go-control-plane external_auth.pb.go")

Envoy provides both `v2` and `v3` APIs; here we implement the `v3` `Check` method, containerize the service, and deploy it to Kubernetes.

Example implementation:

```go
package serverV1

import (
	"encoding/json"
	authv3 "github.com/envoyproxy/go-control-plane/envoy/service/auth/v3"
	typev3 "github.com/envoyproxy/go-control-plane/envoy/type/v3"
	"github.com/go-kit/log"
	"github.com/go-kit/log/level"
	"github.com/redis/go-redis/v9"
	"google.golang.org/genproto/googleapis/rpc/status"
	"google.golang.org/grpc/codes"

	"authservice/config"
	"golang.org/x/net/context"
)

type Server struct {
	authv3.UnimplementedAuthorizationServer
	conf   *config.Config
	redis  *redis.Client
	repo   *Repository
	logger log.Logger
}

func NewServer(
	conf *config.Config,
	redis *redis.Client,
	repo *Repository,
	logger log.Logger,
) authv3.AuthorizationServer {
	return &Server{
		conf:   conf,
		redis:  redis,
		repo:   repo,
		logger: logger,
	}
}

var (
	UnauthorizedMsg = "Unauthorized"
	ForbiddenMsg    = "Forbidden"
)

// Response represents the JSON body returned on denied requests
type Response struct {
	Code int64    `json:"code"`
	Msg  string   `json:"msg"`
	Data struct{} `json:"data"`
}

// Check implements Istio/Envoy gRPC external authorization
func (s *Server) Check(ctx context.Context, req *authv3.CheckRequest) (*authv3.CheckResponse, error) {
	// Custom auth logic below; replace as needed for your domain
	attrs := req.GetAttributes()
	httpHeaders := attrs.GetRequest().GetHttp().GetHeaders()

	// Extract request path
	path, exists := httpHeaders[":path"]
	if !exists {
		_ = level.Info(s.logger).Log("msg", "missing :path header")
		return s.Unauthorized(), nil
	}

	// Allow whitelisted endpoints without authentication
	if s.repo.IsWhiteListApi(path) {
		return s.Allow(), nil
	}

	// Extract Authorization header
	token, exists := httpHeaders["authorization"]
	duration := 7 * 24 * 60 * 60
	if !exists {
		_ = level.Info(s.logger).Log("msg", "missing authorization header")
		return s.Unauthorized(), nil
	}

	// Strip "Bearer " prefix
	if len(token) <= 7 {
		_ = level.Info(s.logger).Log("msg", "invalid authorization format: missing Bearer prefix")
		return s.Unauthorized(), nil
	}
	token = token[7:]

	// Validate token against Redis
	if err := s.repo.GetAuthentication(ctx, token, int64(duration)); err != nil {
		_ = level.Info(s.logger).Log("msg", "access token not found or expired")
		return s.Unauthorized(), nil
	}
	return s.Allow(), nil
}

// Allow permits the request (HTTP 200 / gRPC OK)
func (s *Server) Allow() *authv3.CheckResponse {
	return &authv3.CheckResponse{
		Status: &status.Status{Code: int32(codes.OK)},
		HttpResponse: &authv3.CheckResponse_OkResponse{
			OkResponse: &authv3.OkHttpResponse{},
		},
	}
}

// Unauthorized rejects the request with HTTP 401
func (s *Server) Unauthorized() *authv3.CheckResponse {
	resp := &Response{
		Code: int64(typev3.StatusCode_Unauthorized),
		Msg:  UnauthorizedMsg,
		Data: struct{}{},
	}
	respJson, err := json.Marshal(resp)
	httpBody := ""
	if err == nil {
		httpBody = string(respJson)
	}
	return &authv3.CheckResponse{
		Status: &status.Status{Code: int32(codes.Unauthenticated)},
		HttpResponse: &authv3.CheckResponse_DeniedResponse{
			DeniedResponse: &authv3.DeniedHttpResponse{
				Status: &typev3.HttpStatus{Code: typev3.StatusCode_Unauthorized},
				Body:   httpBody,
			},
		},
	}
}

// Forbidden rejects the request with HTTP 403
func (s *Server) Forbidden() *authv3.CheckResponse {
	resp := &Response{
		Code: int64(typev3.StatusCode_Forbidden),
		Msg:  ForbiddenMsg,
		Data: struct{}{},
	}
	respJson, err := json.Marshal(resp)
	httpBody := ""
	if err == nil {
		httpBody = string(respJson)
	}

	return &authv3.CheckResponse{
		Status: &status.Status{Code: int32(codes.PermissionDenied)},
		HttpResponse: &authv3.CheckResponse_DeniedResponse{
			DeniedResponse: &authv3.DeniedHttpResponse{
				Status: &typev3.HttpStatus{Code: typev3.StatusCode_Forbidden},
				Body:   httpBody,
			},
		},
	}
}
```

### 3. Create the Istio `AuthorizationPolicy`

Finally, apply an `AuthorizationPolicy` with `action: CUSTOM`. Once applied, matching requests passing through Istio are intercepted and routed to your gRPC auth service before reaching upstream workloads.

> [!NOTE]
> The `spec.provider.name` field must match the `extensionProviders[].name` configured in the Istio `ConfigMap`.

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: rgrpc-ext-authz
  namespace: rgrpc-dev
spec:
  action: CUSTOM
  provider:
    name: rgrpc-dev-authz-grpc-provider
  rules:
    - to:
        - operation:
            hosts:
              - api.your-domain.com:31380
```

***
