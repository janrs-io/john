---
title: "Istio 实现 ext-authz 外部扩展鉴权以及对接基于 k8s 的微服务"
author: "John Yang"
lang: zh
pubDatetime: 2023-06-01T05:39:00Z
featured: true
tags:
  - Istio
  - Kubernetes
  - Security
  - Microservices
description: "可以实现基于 redis 的 token 鉴权以及实现 rbac 鉴权。 Istio 的外部鉴权本质是基于 Envoy 实现的，直接看 Envoy 的代码，链接地址：点击自动跳转 Isio 官方的 Demo 代码，链接：点击自动跳转 Istio 提供了基于 HTTP 方式以及 Grpc 方式的外部鉴权扩..."
---

可以实现基于 `redis` 的 `token` 鉴权以及实现 `rbac` 鉴权。

***

`Istio` 的外部鉴权本质是基于 `Envoy` 实现的，直接看 `Envoy`  的代码，链接地址：[点击自动跳转](https://github.com/envoyproxy/envoy/blob/e98e41a8e168af7acae8079fc0cd68155f699aa3/api/envoy/service/auth/v3/external_auth.proto "点击自动跳转")

`Isio` 官方的 `Demo` 代码，链接：[点击自动跳转](https://github.com/istio/istio/tree/master/samples/extauthz "点击自动跳转")

## 实现

`Istio` 提供了基于 `HTTP` 方式以及 `Grpc` 方式的外部鉴权扩展，这里这实现了 `Grpc`。

### 配置

修改 `Istio` 的 `Configmap` 配置。在 `mesh` 字段下面添加以下代码配置：

```yaml
extensionProviders:
    - name: "rgrpc-dev-authz-grpc-provider"
      envoyExtAuthzGrpc:
        service: "auth.rgrpc-dev.svc.cluster.local"
        port: 50051
```

**截图如下**

### 创建 `Istio` 鉴权 `Grpc` 服务

本质上，`Istio` 的外部鉴权是基于 `Evnoy` 实现，只需要实现了 `Envoy` 的 `Grpc` 方法后 `Istio` 就会自动调用。

需要实现的 `Envoy` 的 `external_auth.pb.go`文件 链接：[点击自动跳转](https://github.com/envoyproxy/go-control-plane/blob/main/envoy/service/auth/v3/external_auth.pb.go "点击自动跳转")

只需要实现里面的 `Check` 方法即可。`Envoy` 官方提供了 `v2` 以及 `v3` 代码的实现，这里我只实现了 `v3` 的接口。

写好代码后将服务做成镜像部署到 `k8s`。

案例代码如下：

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
	UnauthorizedMsg = "没有权限"
	ForbiddenMsg    = "没有权限"
)

// Response 返回 HTTP Body 数据
type Response struct {
	Code int64    `json:"code"`
	Msg  string   `json:"msg"`
	Data struct{} `json:"data"`
}

// Check istio-grpc 外部鉴权方法
func (s *Server) Check(ctx context.Context, req *authv3.CheckRequest) (*authv3.CheckResponse, error) {
	// 以下是我的逻辑代码。可以全部删除然后自行修改
	attrs := req.GetAttributes()
	httpHeaders := attrs.GetRequest().GetHttp().GetHeaders()
	// 获取请求路径
	path, exists := httpHeaders[":path"]
	if !exists {
		_ = level.Info(s.logger).Log("msg", "获取不到 :path 字段")
		return s.Unauthorized(), nil
	}
	// 判断是否是白名单
	if s.repo.IsWhiteListApi(path) {
		return s.Allow(), nil
	}
	// 获取头部 token
	token, exists := httpHeaders["authorization"]
	duration := 7 * 24 * 60 * 60
	if !exists {
		_ = level.Info(s.logger).Log("msg", "未传递头部 authorization 字段")
		return s.Unauthorized(), nil
	}
	// 去除头部 "Bearer "字符串
	if len(token) <= 7 {
		_ = level.Info(s.logger).Log("msg", "authorization 数据格式错误。没有设置 Bearer 前缀")
		return s.Unauthorized(), nil
	}
	// 截取后面的 token 字符串
	token = token[7:]

	// 验证 token
	if err := s.repo.GetAuthentication(ctx, token, int64(duration)); err != nil {
		_ = level.Info(s.logger).Log("msg", "access token 不存在")
		return s.Unauthorized(), nil
	}
	return s.Allow(), nil
}

// Allow 通过鉴权。返回 200
func (s *Server) Allow() *authv3.CheckResponse {
	return &authv3.CheckResponse{
		Status: &status.Status{Code: int32(codes.OK)},
		HttpResponse: &authv3.CheckResponse_OkResponse{
			OkResponse: &authv3.OkHttpResponse{},
		},
	}
}

// Unauthorized Unauthorized 未授权 401
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

// Forbidden Forbidden 没有权限 403
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

### 创建 `Istio` 的 `AuthorizationPolicy`

最后设置 `Istio` 的 `AuthorizationPolicy`。设置后，所有经过 `Istio` 网关的请求都会自行被拦截，然后调用部署好的 `Grpc` 鉴权服务进行鉴权。

> **需要注意的是：`provider` 需要跟上面 `Istio` 的 `Configmap` 中的 `extensionProviders.name` 字段的值对应上才会调用到配置中的 `Grpc` 地址**

使用的是 `CUSTOM` 配置，配置如下：

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
