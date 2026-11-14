---
title: "Microservices Best Practices with Go, gRPC, Kubernetes, and Istio — Part 2/3"
author: "John Yang"
lang: en
pubDatetime: 2023-03-31T06:33:00Z
tags:
  - Go
  - gRPC
  - Kubernetes
  - Istio
  - Microservices
description: "Part 2 of the Go/gRPC/Kubernetes/Istio series: cloning pongservice into pingservice, wiring cross-service gRPC calls with Wire DI, and testing locally."
---

In Part 1, we set up the repository layout and implemented `pongservice`.

In Part 2, we create `pingservice` and have it call `pongservice` over gRPC. Creating a new service is straightforward: copy `pongservice` and update its module, proto definitions, configuration, and client wiring.

# Project Structure

Final directory layout for `pingservice`:

```bash
pingservice
├── buf.gen.yaml
├── cmd
│   ├── main.go
│   └── server
│       ├── grpc.go
│       ├── http.go
│       ├── run.go
│       ├── wire.go
│       └── wire_gen.go
├── config
│   ├── client.go
│   ├── config.go
│   └── config.yaml
├── genproto
│   └── v1
│       ├── gw
│       │   └── pingservice.pb.gw.go
│       ├── pingservice.pb.go
│       └── pingservice_grpc.pb.go
├── go.mod
├── go.sum
├── proto
│   ├── buf.lock
│   ├── buf.yaml
│   └── v1
│       ├── pingservice.proto
│       └── pingservice.yaml
└── service
    ├── client.go
    └── server.go

9 directories, 21 files
```

# Getting Started

## Copy `pongservice`

Inside `src/`, copy `pongservice` to `pingservice`:

```bash
cp -R pongservice pingservice
```

## Remove `wire_gen.go`

Delete `pingservice/cmd/server/wire_gen.go`.

## Update `go.mod` Module Name

Update the module path in `pingservice/go.mod`:

```go
module github.com/janrs-io/Jgrpc/src/pingservice
```

## Regenerate Protobuf Stubs

Delete `pingservice/proto/v1/pongservice.proto`, `pingservice/proto/v1/pongservice.yaml`, and the entire `pingservice/genproto` directory.

Create `pingservice/proto/v1/pingservice.proto`:

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

Create `pingservice/proto/v1/pingservice.yaml`:

```yaml
type: google.api.Service
config_version: 3

http:
  rules:
    - selector: proto.v1.PingService.Ping
      get: /ping.v1.ping
```

Update `pingservice/buf.gen.yaml`:

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

Run `buf generate` in `pingservice/`:

```bash
buf generate proto/v1
```

## Update Imports and Identifiers

Replace `pongservice` import paths with `pingservice` across all files, and rename `Pong`/`pong` symbols to `Ping`/`ping` until the package compiles cleanly.

## Update `config.yaml`

Update `pingservice/config/config.yaml` to use gRPC port `:50052`, HTTP port `:9002`, and service names `ping-grpc` / `ping-http`:

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

## Regenerate Wire DI

Run `wire ./...` inside `pingservice/`:

```bash
wire ./...
```

## Import `pongservice` in `go.mod`

Because `pingservice` calls `pongservice` over gRPC, add a local `replace` directive and dependency in `pingservice/go.mod`:

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

## Add `pong` Client Address to `config.yaml`

Add the `client` block to `pingservice/config/config.yaml`:

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

## Create `client.go` in `config/`

Create `pingservice/config/client.go`:

```go
package config

// Client Client service config
type Client struct {
	Pong string `json:"pong" yaml:"pong"`
}
```

## Update `config.go`

Add `Client` to `Config` in `pingservice/config/config.go`:

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

## Update `service/client.go`

Add `NewPongClient` to `pingservice/service/client.go`:

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

## Update `service/server.go`

Inject `pongClient` and call `s.pongClient.Pong(...)` inside `Ping`:

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

## Update `wire.go`

Add `service.NewPongClient` to `wire.Build` in `pingservice/cmd/server/wire.go`:

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

Regenerate Wire (run `go mod tidy` first if needed):

```bash
wire ./...
```

## Start Both Services

Start `pongservice` and `pingservice` in separate terminals:

```bash
go run cmd/main.go
```

Request the `pingservice` HTTP endpoint:

```text
127.0.0.1:9002/ping.v1.ping?msg=best practice
```

Expected JSON response:

```json
{
    "msg": "response ping msg:best practice and msg from pong service is: response pong msg:request from ping service"
}
```

# Summary

We now have `pingservice` calling `pongservice` over gRPC with Wire dependency injection and `grpc-gateway` HTTP transcoding. In Part 3, we automate CI/CD deployment onto Kubernetes and Istio.

***
