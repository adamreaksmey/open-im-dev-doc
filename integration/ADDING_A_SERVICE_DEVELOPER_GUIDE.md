# Adding a New Business Service — Developer Guide

This guide describes how to add a new business (RPC) service to the OpenIM Server codebase, following existing patterns. It covers directory layout, bootstrap, protobuf/gRPC, service-to-service calls, REST gateway, config, middleware, and local run/test. All steps are based on actual code in this repo.

**Prerequisites:** [DEVELOPER.md](../README.md), [ADDING_A_MICROSERVICE.md](ADDING_A_MICROSERVICE.md) (high-level flow). Protocol types and generated gRPC code come from **`github.com/openimsdk/protocol`** (external repo); this repo implements the servers.

---

## Table of Contents

1. [Directory Structure](#1-directory-structure)
2. [Service Bootstrap](#2-service-bootstrap)
3. [Protobuf / gRPC Definition](#3-protobuf--grpc-definition)
4. [Implementing the gRPC Server](#4-implementing-the-grpc-server)
5. [Service-to-Service Communication (gRPC Client)](#5-service-to-service-communication-grpc-client)
6. [HTTP / REST Gateway](#6-http--rest-gateway)
7. [Config](#7-config)
8. [Adding Middleware](#8-adding-middleware)
9. [Running & Testing Locally](#9-running--testing-locally)

---

## 1. Directory Structure

### Where to create the new service

| Purpose | Path | Reference |
|--------|------|-----------|
| **Entrypoint binary** | `cmd/openim-rpc/openim-rpc-<name>/main.go` | e.g. `cmd/openim-rpc/openim-rpc-conversation/main.go`, `cmd/openim-rpc/openim-rpc-auth/main.go` |
| **RPC implementation** | `internal/rpc/<shortname>/` | e.g. `internal/rpc/conversation/`, `internal/rpc/auth/` |
| **RPC client wrappers** | `pkg/rpcli/` | One file per service, e.g. `pkg/rpcli/user.go`, `pkg/rpcli/auth.go` |
| **Config files** | `config/` | One YAML per service: `config/openim-rpc-<name>.yml` |

**Conventions:**

- **cmd:** One directory per runnable under `cmd/openim-rpc/`; directory name = binary name (e.g. `openim-rpc-conversation`).
- **internal/rpc:** Package name is short, lowercase, no hyphens (e.g. `conversation`, `auth`). All server logic, config struct, and `Start` live here.
- **RPC client wrappers:** In `pkg/rpcli/` when you need shared helpers (e.g. `GetUsersInfo`) used by other internal services. The raw gRPC client is from the protocol package; `pkg/rpcli` adds convenience methods.

**Example for a new service `openim-rpc-myservice`:**

```text
open-im-server/
├── cmd/openim-rpc/
│   └── openim-rpc-myservice/
│       └── main.go
├── internal/rpc/
│   └── myservice/
│       ├── server.go      # Config, Start(), server struct
│       └── myservice.go   # RPC method implementations
├── pkg/rpcli/
│   └── myservice.go       # Optional: client wrapper
└── config/
    └── openim-rpc-myservice.yml
```

Create directories:

```bash
mkdir -p cmd/openim-rpc/openim-rpc-myservice
mkdir -p internal/rpc/myservice
```

---

## 2. Service Bootstrap

### Main entrypoint

Each RPC binary has a minimal `main.go` that delegates to a Cobra command in `pkg/common/cmd/`. The command loads config, creates the discovery client, and calls `startrpc.Start` with the service’s `Start` function.

**Reference:** `cmd/openim-rpc/openim-rpc-conversation/main.go`:

```go
package main

import (
	"github.com/openimsdk/open-im-server/v3/pkg/common/cmd"
	"github.com/openimsdk/tools/system/program"
)

func main() {
	if err := cmd.NewConversationRpcCmd().Exec(); err != nil {
		program.ExitWithError(err)
	}
}
```

Pattern: `cmd.New<Service>RpcCmd().Exec()`. No config or discovery logic in `main.go`.

### Command and config map

The real bootstrap lives in `pkg/common/cmd/`. For each RPC service there is a `*RpcCmd` type that:

1. Embeds `*RootCmd` (Cobra root, config path, index).
2. Holds a **config map**: config file name → pointer to the struct that receives that file’s content.
3. In `RunE`, calls `startrpc.Start` with discovery config, RPC config (ports, circuit breaker, rate limiter, prometheus), service index, **discovery name**, and the service’s `Start` function.

**Reference:** `pkg/common/cmd/conversation.go`:

```go
type ConversationRpcCmd struct {
	*RootCmd
	ctx                context.Context
	configMap          map[string]any
	conversationConfig *conversation.Config
}

func NewConversationRpcCmd() *ConversationRpcCmd {
	var conversationConfig conversation.Config
	ret := &ConversationRpcCmd{conversationConfig: &conversationConfig}
	ret.configMap = map[string]any{
		config.OpenIMRPCConversationCfgFileName: &conversationConfig.RpcConfig,
		config.RedisConfigFileName:              &conversationConfig.RedisConfig,
		config.MongodbConfigFileName:            &conversationConfig.MongodbConfig,
		config.ShareFileName:                    &conversationConfig.Share,
		config.NotificationFileName:             &conversationConfig.NotificationConfig,
		config.WebhooksConfigFileName:           &conversationConfig.WebhooksConfig,
		config.LocalCacheConfigFileName:         &conversationConfig.LocalCacheConfig,
		config.DiscoveryConfigFilename:          &conversationConfig.Discovery,
	}
	ret.RootCmd = NewRootCmd(program.GetProcessName(), WithConfigMap(ret.configMap))
	ret.ctx = context.WithValue(context.Background(), "version", version.Version)
	ret.Command.RunE = func(cmd *cobra.Command, args []string) error {
		return ret.runE()
	}
	return ret
}

func (a *ConversationRpcCmd) runE() error {
	return startrpc.Start(a.ctx,
		&a.conversationConfig.Discovery,
		&a.conversationConfig.RpcConfig.CircuitBreaker,
		&a.conversationConfig.RpcConfig.RateLimiter,
		&a.conversationConfig.RpcConfig.Prometheus,
		a.conversationConfig.RpcConfig.RPC.ListenIP,
		a.conversationConfig.RpcConfig.RPC.RegisterIP,
		a.conversationConfig.RpcConfig.RPC.AutoSetPorts,
		a.conversationConfig.RpcConfig.RPC.Ports,
		a.Index(),
		a.conversationConfig.Discovery.RpcService.Conversation,  // discovery name
		&a.conversationConfig.NotificationConfig,
		a.conversationConfig,
		[]string{ /* config file names for watch */ },
		nil,
		conversation.Start)
}
```

For a new service you add a similar file (e.g. `pkg/common/cmd/myservice.go`) and wire your `internal/rpc/myservice.Config` and `myservice.Start`, and the new config file name constant (see [§7](#7-config)).

### Discovery / registry

Discovery is **etcd** (or Kubernetes in k8s mode). The registry is handled inside `startrpc.Start` in `pkg/common/startrpc/start.go`:

- It calls `kdisc.NewDiscoveryRegister(disc, watchServiceNames)` from `pkg/common/discovery/discoveryregister.go`.
- That returns a `discovery.SvcDiscoveryRegistry` (etcd implementation from `github.com/openimsdk/tools/discovery/etcd`, or standalone/kubernetes).
- When the first gRPC service is registered, `startrpc` creates the gRPC server, listens on the configured address, and calls `client.Register(ctx, rpcRegisterName, registerIP, port, grpcOpt)` so the service is discoverable under `rpcRegisterName`.

So you do **not** register the service yourself. You only:

1. Pass the **discovery name** (e.g. `a.conversationConfig.Discovery.RpcService.Conversation`) as `rpcRegisterName` to `startrpc.Start`.
2. Ensure that name is defined in `config.Discovery.RpcService` and in `config/discovery.yml` (see [§7](#7-config)).

### Config loading

Config is loaded in `RootCmd.PersistentPreRunE` → `initializeConfiguration`. That iterates over the command’s `configMap` and calls `config.Load(configDirectory, configFileName, envPrefix, configStruct)` for each entry. So every file name in the map must exist under the config directory (e.g. `config/`), and the corresponding struct field (e.g. `RpcConfig`) receives the parsed YAML. Config directory comes from the `--config_folder_path` / `-c` flag or env (see [DEVELOPER.md](DEVELOPER.md)).

---

## 3. Protobuf / gRPC Definition

Service protos and generated gRPC/Go code live in a separate **protocol repo** that we host (e.g. by cloning `github.com/openimsdk/protocol` and maintaining our own copy or fork). We do not add `.proto` files or generated code inside open-im-server. When we add or change a service or method, we update the protocol repo, generate code there, push the update (and tag a release if needed), then bump the dependency in open-im-server.

### Where .proto files live

- **Protocol repo:** The repo we host (e.g. our clone or fork of `github.com/openimsdk/protocol`). All `.proto` definitions and generated Go live there. The layout and `go_package` options in that repo define the Go import paths (e.g. `github.com/openimsdk/protocol/conversation`, `github.com/openimsdk/protocol/auth`).
- **open-im-server:** Consumes the protocol module via `go.mod`; no `.proto` or generated Go for these services lives in this repo.

### Workflow: add or change a proto in the protocol repo

1. **Clone or open the protocol repo** we host (e.g. our fork or internal clone of the OpenIM protocol repo).

2. **Add or edit .proto files** in the layout used by that repo (e.g. a new file per service or add RPCs to an existing file). Use the same naming and `go_package` conventions as existing services (e.g. `openim.conversation`, `openim.auth`) so generated packages match the import paths we use in open-im-server.

3. **Generate Go (and any other languages)** using that repo’s Makefile or scripts. Do not generate inside open-im-server.

4. **Commit and push** the update to the protocol repo. Tag a release (e.g. `v0.0.xx`) when we want open-im-server to pick up the change.

5. **Bump the dependency in open-im-server:**
   ```bash
   cd /path/to/open-im-server
   go get github.com/openimsdk/protocol@v0.0.xx   # or the module path we use for our hosted protocol
   go mod tidy
   ```

6. **In open-im-server:** Implement or update the generated server interface in `internal/rpc/<service>/` and use the generated client types; import from the protocol module (e.g. `github.com/openimsdk/protocol/conversation`, `github.com/openimsdk/protocol/auth`). No local proto or codegen in this repo.

### Code generation (in the protocol repo)

Run code generation **inside the protocol repo**, following its own docs (Makefile, scripts, or CI). OpenIM also documents a custom Protoc bundle: [docs/contrib/protoc-tools.md](contrib/protoc-tools.md). After we push an update and (if needed) tag a release, open-im-server only needs to update its `go.mod` dependency.

### Where generated code is used in open-im-server

Generated code is **not** in this repo. We import the protocol module, e.g.:

- `github.com/openimsdk/protocol/conversation` — server interface, client, request/response types
- `github.com/openimsdk/protocol/auth`
- `github.com/openimsdk/protocol/user`
- etc.

For a new service added in our hosted protocol repo, we add an import like `github.com/openimsdk/protocol/<servicename>` and use the generated `pb.RegisterXxxServer`, `pb.NewXxxClient`, and message types.

---

## 4. Implementing the gRPC Server

### Server struct and Start

- **Package:** `internal/rpc/<service>` (e.g. `internal/rpc/conversation`).
- **Server struct:** Embeds the generated `pb.Unimplemented<Service>Server` and holds config, discovery, and any dependencies (DB, other RPC clients). Name: `<service>Server` (e.g. `conversationServer`).

**Reference:** `internal/rpc/conversation/conversation.go`:

```go
type conversationServer struct {
	pbconversation.UnimplementedConversationServer
	conversationDatabase controller.ConversationDatabase
	// ...
	config       *Config
	userClient   *rpcli.UserClient
	msgClient    *rpcli.MsgClient
	groupClient  *rpcli.GroupClient
}

func Start(ctx context.Context, config *Config, client discovery.SvcDiscoveryRegistry, server grpc.ServiceRegistrar) error {
	// 1. Build DBs, get connections to other RPCs via client.GetConn(ctx, config.Discovery.RpcService.User), etc.
	// 2. Construct conversationServer.
	// 3. Register: pbconversation.RegisterConversationServer(server, &cs)
	pbconversation.RegisterConversationServer(server, &cs)
	return nil
}
```

Your `Start` is invoked by `startrpc.Start` with the same `config` and `server` registrar used for listening and etcd registration. Do not start your own gRPC server or listener; just register your implementation.

### Implementing RPC methods

Implement the generated interface. Use `github.com/openimsdk/tools/errs` for errors and `github.com/openimsdk/tools/log` for logging; avoid `panic` and the standard `log` package.

**Reference:** `internal/rpc/auth/auth.go` (GetAdminToken), `internal/rpc/conversation/conversation.go` (GetConversation, etc.). Naming: method name matches the proto RPC name (e.g. `GetConversation`).

### Naming conventions

- **Config struct:** `Config` in the same package; holds RPC config, discovery, share, DB configs, etc.
- **Server struct:** `<service>Server` (lowercase first letter), embeds `pb.Unimplemented<Service>Server`.
- **Start function:** `Start(ctx, config, discoveryClient, grpc.ServiceRegistrar) error`.
- **Files:** `server.go` or a single file for config + Start; one or more files for RPC handlers (e.g. `conversation.go`, `sync.go`).

---

## 5. Service-to-Service Communication (gRPC Client)

### How services call each other

Other services obtain a gRPC connection by **service discovery name** via the shared `discovery.SvcDiscoveryRegistry` (or `discovery.Conn`). They do not use static IPs; the discovery client resolves the name (e.g. from etcd) to an address and dials it.

**Reference:** `internal/rpc/conversation/conversation.go`:

```go
userConn, err := client.GetConn(ctx, config.Discovery.RpcService.User)
if err != nil {
	return err
}
groupConn, err := client.GetConn(ctx, config.Discovery.RpcService.Group)
// ...
msgConn, err := client.GetConn(ctx, config.Discovery.RpcService.Msg)
// ...
userClient := rpcli.NewUserClient(userConn)
groupClient := rpcli.NewGroupClient(groupConn)
msgClient := rpcli.NewMsgClient(msgConn)
```

So: same `client` (passed into `Start`), different `config.Discovery.RpcService.<Name>` strings. Discovery is **etcd** (or Kubernetes); see `pkg/common/discovery/discoveryregister.go` (no Zookeeper in this codebase).

### Creating a gRPC client to call another service

1. **From an RPC server:** In your `Start`, use the same `client discovery.SvcDiscoveryRegistry` and the discovery name for the target service (e.g. `config.Discovery.RpcService.User`). Call `conn, err := client.GetConn(ctx, config.Discovery.RpcService.User)` then wrap with the generated client, e.g. `user.NewUserClient(conn)`. Optionally wrap again with `pkg/rpcli` if you have shared helpers (e.g. `rpcli.NewUserClient(conn)`).
2. **From openim-api:** The API gets connections in `internal/api/router.go` via the same discovery client and `cfg.Discovery.RpcService.*`, then constructs the protocol client (and optionally `rpcli` wrapper). See [§6](#6-http--rest-gateway).

### RPC client wrappers (pkg/rpcli)

When multiple callers need the same convenience methods (e.g. “get user info by IDs”), the pattern is a wrapper in `pkg/rpcli/<service>.go` that embeds the generated client and adds methods. **Reference:** `pkg/rpcli/user.go`:

```go
func NewUserClient(cc grpc.ClientConnInterface) *UserClient {
	return &UserClient{user.NewUserClient(cc)}
}

type UserClient struct {
	user.UserClient
}

func (x *UserClient) GetUsersInfo(ctx context.Context, userIDs []string) ([]*sdkws.UserInfo, error) {
	// Build request, call x.UserClient.GetDesignateUsers, extract response.
}
```

So: raw client from protocol; `pkg/rpcli` adds helpers used across internal/rpc and API.

### Example: calling another service from your new service

In your `internal/rpc/myservice/` `Start`:

```go
userConn, err := client.GetConn(ctx, config.Discovery.RpcService.User)
if err != nil {
	return err
}
userClient := rpcli.NewUserClient(userConn)
// Store userClient on your server struct; use in RPC methods.
```

Ensure `config.Discovery` is loaded (e.g. in your cmd config map) and `config.Discovery.RpcService.User` is set (e.g. in `config/discovery.yml`).

---

## 6. HTTP / REST Gateway

The REST API is **openim-api** (Gin). It does not re-expose gRPC via a generic gateway; each route is wired to a handler that calls the appropriate gRPC client.

### How the gateway routes to gRPC

1. **Connections:** In `internal/api/router.go`, `newGinRouter` gets a discovery client and calls `client.GetConn(ctx, cfg.Discovery.RpcService.<Service>)` for each backend (Auth, User, Group, Friend, Conversation, Third, Msg). It then creates the generated gRPC client (e.g. `user.NewUserClient(userConn)`) and optionally a wrapper (e.g. `NewUserApi(..., client, cfg.Discovery.RpcService)`).
2. **Handlers:** Handlers are methods on API structs that hold the gRPC client. They use `github.com/openimsdk/tools/a2r` to map Gin’s JSON body and response to a gRPC method call.
3. **Routes:** Routes are registered on Gin groups (e.g. `r.Group("/user")`) with `POST("/path", api.Method)`.

### Exposing your new service via REST

1. **Get a connection in the router:** In `newGinRouter` in `internal/api/router.go`, add:
   ```go
   myConn, err := client.GetConn(ctx, cfg.Discovery.RpcService.MyService)
   if err != nil {
       return nil, err
   }
   ```
2. **Add discovery name:** Add `MyService` to `config.RpcService` and `config/discovery.yml` (see [§7](#7-config)), and ensure the API config map loads discovery so `cfg.Discovery.RpcService.MyService` is set.
3. **API struct and handlers:** Create e.g. `internal/api/myservice_api.go` with a struct that holds the gRPC client and handlers that call `a2r.Call(c, pb.MyServiceClient.MethodName, o.Client)`.
4. **Register routes:** In `newGinRouter`, create a group and bind handlers:
   ```go
   myApi := NewMyServiceApi(pb.NewMyServiceClient(myConn))
   myGroup := r.Group("/myservice")
   myGroup.POST("/echo", myApi.Echo)
   ```

**Reference:** `internal/api/user.go` (UserApi, `a2r.Call`), `internal/api/router.go` (where connections and route groups are set up). For auth: routes use the same middleware stack as other POST routes (e.g. token parsing). To skip token for specific paths, add them to the `Whitelist` in `internal/api/router.go`.

---

## 7. Config

### Adding new config fields for the service

1. **Config file name constant:** In `pkg/common/config/config.go`, add a constant, e.g. `OpenIMRPCMyserviceCfgFileName = "openim-rpc-myservice.yml"`.
2. **RPC config type (if new):** Add a struct that matches the YAML shape used by other RPCs (e.g. `RPC`, `Prometheus`, `RateLimiter`, `CircuitBreaker`) and a `GetConfigFileName() string` that returns `OpenIMRPCMyserviceCfgFileName`. See existing types: `config.Conversation`, `config.Auth`, etc. in `pkg/common/config/config.go` (around lines 297–350).
3. **Discovery name:** Add a field to `config.RpcService` (e.g. `MyService string`) and include it in `RpcService.GetServiceNames()` so discovery can watch it. **Reference:** `pkg/common/config/config.go` (RpcService struct and GetServiceNames).
4. **Env prefix:** In `pkg/common/config/env.go`, add your config file name to the `fileNames` slice in `init()` so `EnvPrefixMap` gets an env prefix for overrides.
5. **Watch list:** If the all-in-one or any runner uses a central config watcher, add your config file name to the list passed to `startrpc.Start` (watchConfigNames) in your cmd’s `runE`.

### Where config structs are defined

- **RPC-level and shared types:** `pkg/common/config/config.go` (RPC, Prometheus, RateLimiter, CircuitBreaker, Discovery, RpcService, Share, Redis, Mongo, etc.).
- **Per-service config:** Your `internal/rpc/<service>.Config` struct in the service package (e.g. `internal/rpc/conversation.Config`), which embeds or references the types from `pkg/common/config`.

### Accessing config at runtime

- In RPC servers: config is passed into `Start` and stored on the server struct (e.g. `cs.config`). Use it in method handlers.
- In API: config is the `cfg *Config` passed to `newGinRouter`; it is loaded by the API command in `pkg/common/cmd/api.go` and includes `cfg.Discovery`, `cfg.API`, etc.

### Config file placement

- **Service YAML:** `config/openim-rpc-myservice.yml`. Structure should match other RPC configs (see `config/openim-rpc-conversation.yml`): `rpc`, `prometheus`, `ratelimiter`, `circuitBreaker`.
- **Discovery:** `config/discovery.yml` must include the new service name under `rpcService`, e.g. `myservice: myservice-rpc-service`.

---

## 8. Adding Middleware

### Patterns used in this repo

- **HTTP (openim-api):** Middleware is registered in `internal/api/router.go` on the Gin engine or groups. Order: gateway (admin JWT / user HMAC) → then logger, prometheus, recovery, CORS, operation ID, token parsing, admin check. **Reference:** `newGinRouter` (e.g. `r.Use(api.GinLogger(), prommetricsGin(), ..., GinParseToken(...), setGinIsAdmin(...))`). Token parsing is in `GinParseToken`; whitelisted paths skip token. Logging uses `github.com/openimsdk/tools/log`; do not use the standard `log` package.
- **gRPC (RPC servers):** Middleware is applied in `startrpc.Start` when building the gRPC server: metadata context, error conversion, logger, request validation, panic capture, optional circuit breaker and rate limiter, and Prometheus interceptors. You don’t add per-service middleware in your `Start`; it’s global in `pkg/common/startrpc/start.go`. Auth inside RPC methods is done explicitly (e.g. `authverify.CheckAccess(ctx, req.OwnerUserID)`).

### Logging

Use `github.com/openimsdk/tools/log` (e.g. `log.CInfo`, `log.ZError`). Logger is initialized in the root command from `config.Log` (log.yml).

### Auth

- **API:** Token is parsed in `GinParseToken` and user/platform stored in context. Admin check uses `authverify.CtxAdminUserIDsKey`. Some routes are whitelisted (see `Whitelist` in `internal/api/router.go`).
- **RPC:** Call `authverify.CheckAccess(ctx, userID)` (or similar) at the start of RPC methods that require a user context.

### Tracing

Tracing is not explicitly wired in the snippets we inspected; if you add it, follow the same pattern as other middlewares (e.g. in `startrpc` for gRPC, or Gin middleware for HTTP).

---

## 9. Running & Testing Locally

### Build

From repo root:

```bash
mage build
# Or only your binary (binary name = directory name under cmd/)
mage build openim-rpc-myservice
```

Binaries are under `_output/` (exact path depends on gomake/OS/arch). The build system discovers binaries from the `cmd/` tree (see `magefile.go`: `customSrcDir = "cmd"`).

### Config path

Set config directory so the process finds `config/` (including `openim-rpc-myservice.yml` and `discovery.yml`):

- Env: `OPENIMCONFIG=/path/to/config` (or the env name from `config.EnvPrefixMap`).
- Or flag: `--config_folder_path=/path/to/config` (or `-c`).

### Run

1. Start infrastructure (etcd, Redis, MongoDB, Kafka, MinIO as needed). See [docs/INFRASTRUCTURE_AND_DOCKER.md](INFRASTRUCTURE_AND_DOCKER.md).
2. Run your RPC:
   ```bash
   _output/bin/.../openim-rpc-myservice -c /path/to/config
   ```
3. If you want it started with the rest of the stack, add it to **start-config.yml** in the repo root under `serviceBinaries` with instance count, e.g. `openim-rpc-myservice: 1`. Then `mage start` (or `mage start openim-rpc-myservice`) will start it. **Reference:** `start-config.yml`.

### Testing gRPC endpoints

- **From another service or API:** Call the REST route that forwards to your RPC, or use a gRPC client (e.g. grpcurl) against the address where your RPC is listening. With `autoSetPorts: true`, the port is chosen at runtime and registered in etcd; you can read the log line that prints the listen address or use a tool that resolves via discovery.
- **Unit tests:** Add tests in `internal/rpc/myservice/` (or `*_test.go` next to the handler). Use the same config and discovery patterns; in tests you can use a real or mock discovery/client.

### Quick sanity check

1. Start etcd and your RPC (and any dependencies it needs, e.g. Redis/Mongo if your Start uses them).
2. If you exposed REST: start openim-api, then e.g. `curl -X POST http://localhost:10002/myservice/echo -H "Content-Type: application/json" -d '{"message":"hello"}'` (adjust port and path to your API and route).

---

## Summary checklist

- [ ] Directories: `cmd/openim-rpc/openim-rpc-<name>/main.go`, `internal/rpc/<name>/`.
- [ ] Config: `OpenIMRPC<Name>CfgFileName`, RPC config type with `GetConfigFileName()`, `config.RpcService.<Name>` and `GetServiceNames()`, `config/discovery.yml`, `config/openim-rpc-<name>.yml`, and `env.go` file list.
- [ ] Proto: Add service in protocol repo (`github.com/openimsdk/protocol`), generate there, tag release, then bump dep in open-im-server (`go get github.com/openimsdk/protocol@v0.0.xx`).
- [ ] RPC: `Start(ctx, config, discovery, registrar)` that builds deps, gets other clients via `client.GetConn(ctx, config.Discovery.RpcService.*)`, registers `pb.Register<Service>Server(server, impl)`.
- [ ] Cmd: New `*RpcCmd` in `pkg/common/cmd/<name>.go` with config map and `startrpc.Start(..., discoveryName, ..., yourpackage.Start)`; `main.go` calls `cmd.New<Name>RpcCmd().Exec()`.
- [ ] Optional client wrapper: `pkg/rpcli/<name>.go` if other services need shared helpers.
- [ ] Optional REST: In `internal/api/router.go`, `GetConn` for your service, new API struct and handlers with `a2r.Call`, register route group; add discovery name and whitelist if needed.
- [ ] `start-config.yml`: Add `openim-rpc-<name>: 1` (or desired count) if you use `mage start`.
- [ ] Build: `mage build openim-rpc-<name>`; run with `-c` or `OPENIMCONFIG`; test via API or gRPC client.

For more on the codebase and request flow, see [DEVELOPER.md](README.md), [PROCESS_FLOW.md](PROCESS_FLOW.md), [ADDING_A_MICROSERVICE.md](ADDING_A_MICROSERVICE.md), and [TECHNOLOGY.md](TECHNOLOGY.md).
