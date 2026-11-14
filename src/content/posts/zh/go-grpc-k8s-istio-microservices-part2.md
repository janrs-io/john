---
title: "基于Go/Grpc/kubernetes/Istio开发微服务的最佳实践尝试 - 2/3"
author: "John Yang"
lang: zh
pubDatetime: 2023-03-31T06:33:00Z
tags:
  - Go
  - gRPC
  - Kubernetes
  - Istio
  - Microservices
description: "在上一部分中，我们创建了微服务的目录结构并实现了微服务pongservice。 这部分我们继续实现一个名为pingservice的微服务，访问上一节已经部署好的pongservice微服务。 创建一个新的微服务非常简单，只需复制之前创建的pongservice微服务，然后做一些小改动。"
---

在上一部分中，我们创建了微服务的目录结构并实现了微服务`pongservice`。

这部分我们继续实现一个名为`pingservice`的微服务，访问上一节已经部署好的`pongservice`微服务。

创建一个新的微服务非常简单，只需复制之前创建的`pongservice`微服务，然后做一些小改动。

# 项目结构

这部分最终的目录结构如下：

```bash
pingservice
├── buf.gen.yaml
├── cmd
│   ├── main.go
│   └── server
│       ├── grpc.go
│       ├── http.go
│       ├── run.go
│       ├── wire.go
│       └── wire_gen.go
├── config
│   ├── client.go
│   ├── config.go
│   └── config.yaml
├── genproto
│   └── v1
│       ├── gw
│       │   └── pingservice.pb.gw.go
│       ├── pingservice.pb.go
│       └── pingservice_grpc.pb.go
├── go.mod
├── go.sum
├── proto
│   ├── buf.lock
│   ├── buf.yaml
│   └── v1
│       ├── pingservice.proto
│       └── pingservice.yaml
└── service
    ├── client.go
    └── server.go

9 directories, 21 files
```

# 开始

## 复制

在 `src` 目录下执行如下复制命令：

```bash
cp -R pongservice pingservice
```

## 删除 wire_gen.go

删除 `pingservice/cmd/server` 目录中的 `wire_gen.go` 文件。

## 修改 go.mod module

修改`go.mod`文件的模块，如下代码所示：

```go
module github.com/janrs-io/Jgrpc/src/pingservice
```

## 生成 proto

删除`pingservice/proto/v1`目录下的`pongservice.proto`和`pongservice.yaml`文件以及整个`genproto`文件夹。

然后重新创建`pingservice.proto`和`pingservice.yaml`文件，代码如下：

`pingservice.proto` code:

```proto
syntax = "proto3";

package proto.v1;

option go_package = "github.com/janrs-io/Jgrpc/src/pingservice/genproto/v1";

service PingService {
  rpc Ping(PingRequest) returns(PingResponse){}
}

message PingRequest {
  string msg = 1 ;
}

message PingResponse {
  string msg = 1;
}
```

`pingservice.yaml` code:

```yaml
type: google.api.Service
config_version: 3

http:
  rules:
    - selector: proto.v1.PingService.Ping
      get: /ping.v1.ping
```

修改` pingservice` 目录下的 `buf.gen.yaml` 文件。 修改后的完整代码如下：

```yaml
version: v1
plugins:
  - plugin: go
    out: genproto/v1
    opt:
      - paths=source_relative
  - plugin: go-grpc
    out: genproto/v1
    opt:
      - paths=source_relative
  - plugin: grpc-gateway
    out: genproto/v1/gw
    opt:
      - paths=source_relative
      - grpc_api_configuration=proto/v1/pingservice.yaml
      - standalone=true
```

在`pingservice`目录下执行以下命令生成 `proto` 文件：

```bash
buf generate proto/v1
```

执行命令后，将重新生成`genproto`目录，并自动创建`*pb.go`文件。

## 修改 import 路径和所有代码

检查**所有文件**的导入，将导入路径的`pongservice`修改为`pingservice`。

将所有代码的`Pong/pong`改为`Ping/ping`，直到没有错误为止。

## 修改 config.yaml

修改 `config` 目录下的`config.yaml`文件。
将 `grpc` 的服务器端口更改为 `50052`，将 `http` 的服务器端口更改为 `9002`。
将 `grpc` 的服务名改为`ping-grpc`，将 `http` 的服务名改为`ping-http`。

修改后的完整代码如下：

```yaml
# grpc config
grpc:
  host: ""
  port: ":50052"
  name: "ping-grpc"

# http config
http:
  host: ""
  port: ":9002"
  name: "ping-http"
```

## 生成 generate inject

在 `pingservice` 目录中执行以下 `wire` 命令以重新生成依赖注入文件：

```bash
wire ./...
```

## 引入 pongservier 服务的 go.mod

我们要在 `pingservice` 这个微服务中访问 `pongservice` 的 `grpc` 服务，所以需要导入 `pongservice` 的 `go.mod`。

修改`pingservice`目录下的`go.mod`文件，添加导入`pongservice`的代码，如下：

```go
module github.com/janrs-io/Jgrpc/src/pingservice

go 1.19

replace (
	pongservice => ../pongservice
)

require (
	pongservice v0.0.0
	github.com/google/wire v0.5.0
	github.com/grpc-ecosystem/grpc-gateway/v2 v2.15.2
	github.com/spf13/viper v1.15.0
	google.golang.org/grpc v1.54.0
	google.golang.org/protobuf v1.30.0
)

// the other required package...
```

## 修改 config.yaml

修改`pingservice/config`目录下的`config.yaml`文件，添加如下代码：

```yaml
# service client
client:
  pong: ":50051"
```

修改后的完整代码如下：

```yaml
# grpc config
grpc:
  host: ""
  port: ":50052"
  name: "ping-grpc"

# http config
http:
  host: ""
  port: ":9002"
  name: "ping-http"

# service client
client:
  pong: ":50051"
```

## 生成 client.go

在`pingservice/config`目录下生成`client.go`文件，添加如下代码：

```go
package config

// Client Client service config
type Client struct {
	Pong string `json:"pong" yaml:"pong"`
}
```

## 修改 config.go

在 `Config` 结构体中添加一个 `Client` 字段，代码如下：

```go
Client Client `json:"client" yaml:"client"`
```

修改后的完整代码如下：

```go
package config

import (
	"net/http"

	"github.com/spf13/viper"
	"google.golang.org/grpc"
)

// Config Service config
type Config struct {
	Grpc   Grpc   `json:"grpc" yaml:"grpc"`
	Http   Http   `json:"http" yaml:"http"`
	Client Client `json:"client" yaml:"client"`
}

// NewConfig Initial service's config
func NewConfig(cfg string) *Config {

	if cfg == "" {
		panic("load config file failed.config file can not be empty.")
	}

	viper.SetConfigFile(cfg)

	// Read config file
	if err := viper.ReadInConfig(); err != nil {
		panic("read config failed.[ERROR]=>" + err.Error())
	}
	conf := &Config{}
	// Assign the overloaded configuration to the global
	if err := viper.Unmarshal(conf); err != nil {
		panic("assign config failed.[ERROR]=>" + err.Error())
	}

	return conf

}

// Grpc Grpc server config
type Grpc struct {
	Host   string `json:"host" yaml:"host"`
	Port   string `json:"port" yaml:"port"`
	Name   string `json:"name" yaml:"name"`
	Server *grpc.Server
}

// Http Http server config
type Http struct {
	Host   string `json:"host" yaml:"host"`
	Port   string `json:"port" yaml:"port"`
	Name   string `json:"name" yaml:"name"`
	Server *http.Server
}
```

## 修改 client.go

修改`pingservice/service`目录下的`client.go`文件，添加如下代码：

```go
// NewPongClient New pong service client
func NewPongClient(conf *config.Config) (pongclientv1.PongServiceClient, error) {

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	conn, err := grpc.DialContext(ctx, conf.Client.Pong, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		fmt.Println("dial auth server failed.[ERROR]=>" + err.Error())
		return nil, err
	}
	client := pongclientv1.NewPongServiceClient(conn)
	return client, nil

}
```

修改后的完整代码如下：

```go
package service

import (
	"context"
	"fmt"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	"github.com/janrs-io/Jgrpc/src/pingservice/config"
	v1 "github.com/janrs-io/Jgrpc/src/pingservice/genproto/v1"
	pongclientv1 "github.com/janrs-io/Jgrpc/src/pongservice/genproto/v1"
)

// NewClient New service's client
func NewClient(conf *config.Config) (v1.PingServiceClient, error) {

	serverAddress := conf.Grpc.Host + conf.Grpc.Port
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	conn, err := grpc.DialContext(ctx, serverAddress, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return nil, err
	}
	client := v1.NewPingServiceClient(conn)
	return client, nil

}

// NewPongClient New pong service client
func NewPongClient(conf *config.Config) (pongclientv1.PongServiceClient, error) {

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	conn, err := grpc.DialContext(ctx, conf.Client.Pong, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		fmt.Println("dial auth server failed.[ERROR]=>" + err.Error())
		return nil, err
	}
	client := pongclientv1.NewPongServiceClient(conn)
	return client, nil

}
```

## 修改 server.go

修改`pingservice/service`目录下的`server.go`文件，修改后的完整代码如下：

```go
package service

import (
	"context"
	"google.golang.org/grpc/grpclog"

	"github.com/janrs-io/Jgrpc/src/pingservice/config"
	v1 "github.com/janrs-io/Jgrpc/src/pingservice/genproto/v1"
	pongclientv1 "github.com/janrs-io/Jgrpc/src/pongservice/genproto/v1"
)

// Server Server struct
type Server struct {
	v1.UnimplementedPingServiceServer
	pingClient v1.PingServiceClient
	pongClient pongclientv1.PongServiceClient
	conf       *config.Config
}

// NewServer New service grpc server
func NewServer(
	conf *config.Config,
	pingClient v1.PingServiceClient,
	pongClient pongclientv1.PongServiceClient,
) v1.PingServiceServer {
	return &Server{
		pingClient: pingClient,
		pongClient: pongClient,
		conf:       conf,
	}
}

func (s *Server) Ping(ctx context.Context, req *v1.PingRequest) (*v1.PingResponse, error) {
	pongReq := &pongclientv1.PongRequest{Msg: "request from ping service"}
	pongResp, err := s.pongClient.Pong(ctx, pongReq)
	if err != nil {
		grpclog.Error("connect pong failed.[ERROR]=>" + err.Error())
		return nil, err
	}

	return &v1.PingResponse{
		Msg: "response ping msg:" + req.Msg + " and msg from pong service is: " + pongResp.Msg,
	}, nil
}
```

## 修改 wire.go

修改`pingservice/cmd/server`的`wire.go`文件，添加`service.NewPongClient`依赖注入。 代码如下：

```go
service.NewPongClient
```

修改后的完整代码如下：

```go
//go:build wireinject
// +build wireinject

package server

import (
	"github.com/google/wire"

	"github.com/janrs-io/Jgrpc/src/pingservice/config"
	v1 "github.com/janrs-io/Jgrpc/src/pingservice/genproto/v1"
	"github.com/janrs-io/Jgrpc/src/pingservice/service"
)

// InitServer Inject service's component
func InitServer(conf *config.Config) (v1.PingServiceServer, error) {

	wire.Build(
		service.NewPongClient,
		service.NewClient,
		service.NewServer,
	)

	return &service.Server{}, nil

}
```

在 `pingservice` 目录下执行以下 `wire` 命令重新生成依赖注入文件：

> *如果出现 `go.mod` 引入错误，只需在 `pingservice` 目录中再次运行 `go mod tidy`。*

```bash
wire ./...
```

## 启动 service

分别在`pongservice`目录和`pingservice`目录下执行`go run`命令。

```bash
go run cmd/main.go
```

在浏览器中输入以下请求地址：

```text
127.0.01:9002/ping.v1.ping?msg=best practice
```

一切正确的情况下返回以下 json 数据：

```json
{
    "msg": "response ping msg:best practice and msg from pong service is: response pong msg:request from ping service"
}
```

# 总结

这部分我们新建一个 `pingservice` 微服务，实现访问 `pongservice` 的 `grpc` 服务。

相信通过这两次创建微服务的简单尝试，你一定觉得基于`Go`和`Grpc`开发微服务并不难。

在下一部分中，我们将利用 `Jenkins/Gitlab/Harbor` 和 `Kubernets/Istio` 进行 `devops` 的 `CICD` 部署。

***
