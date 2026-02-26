# OpenIM Server — Developer Guide

This guide helps your team maintain, debug, and extend the OpenIM Server codebase. It covers setup, build, run, test, code standards, and where to look when fixing bugs or adding features.

---

## Quick reference

| Task | Command |
|------|--------|
| Install Mage | `./bootstrap.sh` or `go install github.com/magefile/mage@latest` |
| Build all | `mage build` |
| Build some | `mage build openim-api openim-rpc-user` |
| Start all | `mage start` |
| Check status | `mage check` |
| Stop | `mage stop` |
| Unit tests | `go test ./...` |
| Coverage | `go test -cover ./...` |
| Lint | `golangci-lint run ./...` |
| Config path | `OPENIMCONFIG=/path` or `--config_folder_path=/path` |

---

## Table of Contents

1. [Project overview](#1-project-overview)
2. [Prerequisites](#2-prerequisites)
3. [Repository structure](#3-repository-structure)
4. [Setup and first run](#4-setup-and-first-run)
5. [Build and run](#5-build-and-run)
6. [Configuration](#6-configuration)
7. [Testing](#7-testing)
8. [Code standards](#8-code-standards)
9. [Logging and error handling](#9-logging-and-error-handling)
10. [Git workflow and releases](#10-git-workflow-and-releases)
11. [CI/CD](#11-cicd)
12. [Finding your way (bugs and features)](#12-finding-your-way-bugs-and-features)
13. [Reference: existing docs](#13-reference-existing-docs)

For **request flow** (where a request starts and ends, with Mermaid and sequence diagrams), see **[PROCESS_FLOW.md](PROCESS_FLOW.md)**.

---

## 1. Project overview

**OpenIM Server** is the backend for the OpenIM instant messaging platform. It is not a standalone chat app; it provides services (APIs, RPCs, gateways) that your applications use to implement IM (users, friends, groups, messages, etc.).

- **Language:** Go 1.22+
- **Architecture:** Microservices (API gateway, multiple RPC services, message gateway, transfer, push, cron).
- **Key stacks:** gRPC, REST (Gin), WebSocket, MongoDB, Redis, Kafka, MinIO (or other object storage).
- **Config:** YAML in `config/`, env vars, and `start-config.yml` for service instance counts.

---

## 2. Prerequisites

- **Go:** 1.22.7 or later (see `go.mod`).  
  - [Install Go](https://go.dev/doc/install).
- **Docker & Docker Compose:** Required to run the **infrastructure** OpenIM depends on: **MongoDB**, **Redis**, **Kafka**, **etcd**, and **MinIO**. OpenIM Server does not run these itself; you must start them (e.g. via the repo’s `docker-compose.yml`) before or alongside the OpenIM binaries. There is **no RabbitMQ**—the message queue is **Kafka** only. See **[INFRASTRUCTURE_AND_DOCKER.md](INFRASTRUCTURE_AND_DOCKER.md)** for what each component is for and how to run them with Docker.
- **Mage:** Build and run tasks use [Mage](https://github.com/magefile/mage). Install with:
  ```bash
  go install github.com/magefile/mage@latest
  ```
  Ensure `$GOBIN` or `$GOPATH/bin` is on your `PATH`. The repo’s `bootstrap.sh` can install Mage for you.
- **Optional (macOS):** Scripts assume GNU utils. If you hit issues with `sed`/other, install:
  ```bash
  brew install coreutils findutils gawk gnu-sed gnu-tar grep make
  ```
  And prepend their `gnubin` paths to `PATH` (see [docs/contrib/development.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/development.md)).

---

## 3. Repository structure

| Path | Purpose |
|------|--------|
| **`cmd/`** | Entrypoints for each binary. One directory per binary (e.g. `openim-api`, `openim-rpc/openim-rpc-user`). |
| **`internal/`** | Private application logic: `api/`, `rpc/` (auth, user, msg, group, friend, conversation, third), `msggateway/`, `msgtransfer/`, `push/`. |
| **`pkg/`** | Shared, potentially reusable packages: cache, notification, msgprocessor, rpcli, rpccache, mqbuild, localcache, etc. |
| **`config/`** | YAML config files (per-service and shared). See [Configuration](#6-configuration). |
| **`scripts/`** | Init/config and template scripts. |
| **`tools/`** | Standalone tools: `imctl`, `check-component`, `seq`, `url2im`, `s3`, `ncpu`, stress tests. |
| **`test/`** | E2E, stress, and other tests. |
| **`build/`** | Dockerfiles and build-related config. |
| **`deployments/`** | Kubernetes/deployment manifests. |
| **`docs/`** | Documentation; `docs/contrib/` has detailed guides (see [Reference](#13-reference-existing-docs)). |

**Important:**  
- RPC service definitions and shared types live in **`github.com/openimsdk/protocol`** (external repo). This repo implements the servers.  
- Shared utilities (log, errs, config helpers, etc.) are in **`github.com/openimsdk/tools`**.

---

## 4. Setup and first run

1. **Clone and enter the repo**
   ```bash
   git clone https://github.com/openimsdk/open-im-server.git
   cd open-im-server
   ```

2. **Install Mage (if not already)**
   ```bash
   ./bootstrap.sh
   ```
   Or: `go install github.com/magefile/mage@latest` and ensure it’s on `PATH`.

3. **Download Go modules**
   ```bash
   go mod download
   ```

4. **Config**
   - Copy or generate configs into `config/` (see [Configuration](#6-configuration)).
   - For local dev, often `config/` is already present; you may only need to set `OPENIMCONFIG` or `--config_folder_path` if you use a different path.
   - For full stack (e.g. with Docker), use the project’s init scripts / env (e.g. `scripts/install/environment.sh` and init config) as described in [docs/contrib/init-config.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/init-config.md) and [docs/contrib/environment.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/environment.md).

5. **Start dependencies**
   - E.g. start Redis, MongoDB, Kafka, MinIO (often via `docker-compose up` or your deployment). The repo’s `docker-compose.yml` can be used for a full local stack.

6. **Build and run**
   ```bash
   mage build
   mage start
   ```
   Use `mage check` to see if services are up; `mage stop` to stop them.

---

## 5. Build and run

### Build

- **Build all binaries (default):**
  ```bash
  mage build
  ```
- **Build specific binaries:**
  ```bash
  mage build openim-api openim-rpc-user
  ```
- **Custom paths (root, output, config, tools):**
  ```bash
  mage buildcc   # uses custom RootDir, OutputDir, ConfigDir, ToolsDir from magefile.go
  ```

Binaries are produced under `_output/` (exact path depends on OS/arch and gomake config).

### Run

- **Start all services (as per start-config):**
  ```bash
  mage start
  ```
- **Start specific binaries:**
  ```bash
  mage start openim-api openim-rpc-user
  ```
- **Custom config directory:**
  ```bash
  mage startcc
  ```

### Control

- **Check status:** `mage check`
- **Stop:** `mage stop`

Config path can be set with:

- Env: `OPENIMCONFIG=/path/to/config`
- Or per-binary: `--config_folder_path=/path/to/config`

Ports and other options come from `config/*.yml` (and for API, `-p` can override port).

---

## 6. Configuration

- **Location:** `config/` (or path given by `OPENIMCONFIG` / `--config_folder_path`).
- **Main files:**
  - **External components:** `kafka.yml`, `redis.yml`, `minio.yml`, `mongodb.yml`, `discovery.yml` (e.g. etcd).
  - **OpenIM:** `share.yml`, `log.yml`, `webhooks.yml`, `notification.yml`, plus one YAML per service (e.g. `openim-api.yml`, `openim-rpc-user.yml`, `openim-msggateway.yml`, `openim-rpc-msg.yml`, etc.).
- **How many instances of each binary:** `start-config.yml` in the repo root, e.g.:
  ```yaml
  serviceBinaries:
    openim-api: 1
    openim-rpc-user: 1
    openim-msggateway: 1
    # ...
  ```
- **Init/generation:** Config can be generated from templates and env (e.g. `scripts/install/environment.sh` and init scripts). See [docs/contrib/init-config.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/init-config.md) and [config/README.md](https://github.com/openimsdk/open-im-server/blob/main/config/README.md).

Detailed env vars and per-file descriptions: [docs/contrib/environment.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/environment.md).

---

## 7. Testing

- **Unit tests**
  - Run: `go test ./...` from repo root (or in a specific package).
  - The contrib docs mention `make test` and `make cover`; this repo uses **Mage** for build/start/stop. If your fork adds a Makefile that wraps `go test`, use that; otherwise run `go test` and `go test -cover` directly.
- **API / integration**
  - Script: `scripts/install/test.sh` (if present). Example: `./scripts/install/test.sh openim::test::msg`, `openim::test::user`, etc. See [docs/contrib/test.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/test.md).
- **E2E**
  - Tests live under `test/` (e.g. `test/e2e/`). CI runs full stack (mage build → mage start → mage check) and then API/registration/login flows. See [.github/workflows/go-build-test.yml](https://github.com/openimsdk/open-im-server/blob/main/.github/workflows/go-build-test.yml) and [docs/contrib/test.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/test.md).
- **Coverage**
  - `go test -cover ./...` or your CI coverage job. Keep coverage in line with project expectations when adding features.

---

## 8. Code standards

- **Formatting:** Use `gofmt` and `goimports`. Many targets in contrib docs assume a `make format`; without a Makefile, run:
  ```bash
  gofmt -s -w .
  goimports -w .
  ```
- **Linting:** The repo has [.golangci.yml](https://github.com/openimsdk/open-im-server/blob/main/.golangci.yml). Run:
  ```bash
  golangci-lint run ./...
  ```
- **Style:** Follow [Effective Go](https://go.dev/doc/effective_go.html) and [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments). Project conventions:
  - [docs/contrib/code-conventions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/code-conventions.md) — naming (files, dirs, packages), POSIX shell, testing.
  - [docs/contrib/go-code.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/go-code.md) — line length (~120), file/function length, imports, struct layout, error handling, no `panic`, etc.
- **Commits:** [Conventional Commits](https://www.conventionalcommits.org/): e.g. `fix:`, `feat:`, `docs:`. See [docs/contrib/commit.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/commit.md).
- **Logging:** Do **not** use the standard `log` package. Use `github.com/openimsdk/tools/log` (levels: debug, info, warn, error). Log errors only at the place that first handles them to avoid duplication. See [Logging and error handling](#9-logging-and-error-handling).
- **Errors:** Use `github.com/openimsdk/tools/errs` for wrapping; no `panic`. Prefer returning errors to the caller. See [docs/contrib/logging.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/logging.md) and [CONTRIBUTING.md](https://github.com/openimsdk/open-im-server/blob/main/CONTRIBUTING.md).

---

## 9. Logging and error handling

- **Logger:** `github.com/openimsdk/tools/log` — e.g. `log.CInfo`, `log.ZError`, with context and key-value pairs where appropriate.
- **Errors:** `github.com/openimsdk/tools/errs` — e.g. `errs.Wrap(err, "message")`, `errs.WrapMsg(err, "message", "key", value)`.
- **Error codes:** Documented in [docs/contrib/error-code.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/error-code.md). Use consistent naming (e.g. `InvalidParameter.BindError`) and avoid duplicate semantics.
- **When to wrap:** Wrap errors at the boundary where you add context; don’t re-wrap every layer. See [docs/contrib/logging.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/logging.md).

---

## 10. Git workflow and releases

- **Branching:** GitHub flow: main branch for current development; release branches for supported versions (e.g. `release-3.0.0`). See [docs/contrib/git-workflow.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/git-workflow.md).
- **Forking:** Typical flow: fork → branch from main → change → push → open PR against upstream main. Keep upstream as `upstream` and never push to it directly.
- **Releases/versioning:** [docs/contrib/version.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/version.md) and [docs/contrib/release.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/release.md). Changelog under `CHANGELOG/`.

---

## 11. CI/CD

- **Pipelines:** Under [.github/workflows/](https://github.com/openimsdk/open-im-server/tree/main/.github/workflows). Main one for build/test: **Go Build Test** (e.g. `go-build-test.yml`).
- **Steps (typical):**
  - Checkout, setup Go, install Mage.
  - `go mod tidy`, `go mod download`.
  - Optional: start infra (e.g. `compose-action` with `docker-compose.yml`).
  - `mage build` → `mage start` → `mage check`.
  - Optional: clone OpenIM Chat repo and run its build/start/check.
  - API tests (e.g. register, login, admin token).
- **Running CI locally:** You can run the same commands (build, start, check, tests). For running the workflow file itself, see [docs/contrib/local-actions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/local-actions.md) (e.g. `act`).

---

## 12. Finding your way (bugs and features)

- **REST API / HTTP:** Start from `internal/api/` and the router/handlers; config in `config/openim-api.yml`.
- **gRPC services:** Implementations under `internal/rpc/<service>/`; entrypoints in `cmd/openim-rpc/openim-rpc-*`. Protocol and request/response types: `github.com/openimsdk/protocol`.
- **WebSocket / message gateway:** `internal/msggateway/`, `cmd/openim-msggateway`. Config: `openim-msggateway.yml`.
- **Message pipeline:** `internal/msgtransfer/`, `openim-msgtransfer`; also `pkg/msgprocessor`, `pkg/mqbuild`.
- **Auth / tokens:** `internal/rpc/auth`, `openim-rpc-auth`; config: `openim-rpc-auth.yml` (e.g. token expiry).
- **Users, friends, groups, conversations:** `internal/rpc/user`, `friend`, `group`, `conversation` and corresponding `openim-rpc-*` binaries.
- **Object storage (uploads, etc.):** `internal/rpc/third`, `openim-rpc-third`; config: `openim-rpc-third.yml`, `minio.yml` (or OSS/COS/Kodo/AWS in config).
- **Push notifications:** `internal/push/`, `openim-push`; config: `openim-push.yml`.
- **Scheduled tasks:** `internal/crontask`, `openim-crontask`; config: `openim-crontask.yml`.
- **Caching:** `pkg/rpccache`, `pkg/localcache`; config: `local-cache.yml`.
- **Webhooks / callbacks:** Config: `webhooks.yml`; usage in API/RPC where callbacks are invoked.

When fixing a bug, identify the API or RPC involved (from logs or client), then open the matching `internal/` package and trace from handler to storage/rpc. When adding a feature, follow existing patterns in that service and in [Code standards](#8-code-standards).

---

## 13. Reference: existing docs

| Topic | Doc |
|-------|-----|
| Contributing (PRs, CLA, standards) | [CONTRIBUTING.md](https://github.com/openimsdk/open-im-server/blob/main/CONTRIBUTING.md) |
| Development environment (Go, Docker, Vagrant, macOS) | [docs/contrib/development.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/development.md) |
| Environment variables and config | [docs/contrib/environment.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/environment.md) |
| Init and config generation | [docs/contrib/init-config.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/init-config.md) |
| Config file descriptions | [config/README.md](https://github.com/openimsdk/open-im-server/blob/main/config/README.md) |
| Directory layout (high-level) | [docs/contrib/directory.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/directory.md) |
| Code conventions and naming | [docs/contrib/code-conventions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/code-conventions.md) |
| Go style and format | [docs/contrib/go-code.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/go-code.md) |
| Logging and errors | [docs/contrib/logging.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/logging.md) |
| Error codes | [docs/contrib/error-code.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/error-code.md) |
| Testing (unit, API, E2E) | [docs/contrib/test.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/test.md) |
| Git workflow and branching | [docs/contrib/git-workflow.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/git-workflow.md) |
| Commits and versioning | [docs/contrib/commit.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/commit.md), [docs/contrib/version.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/version.md), [docs/contrib/release.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/release.md) |
| CI/CD and local actions | [docs/contrib/cicd-actions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/cicd-actions.md), [docs/contrib/local-actions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/local-actions.md) |
| Makefile/tools (if you add a Makefile) | [docs/contrib/util-makefile.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/util-makefile.md) |
| Protobuf/API tooling | [docs/contrib/protoc-tools.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/protoc-tools.md), [docs/contrib/api.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/api.md) |
| Full contrib index | [docs/contrib/README.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/README.md) |
| **Request flow (Mermaid & sequence diagrams)** | [PROCESS_FLOW.md](PROCESS_FLOW.md) |
| **Technologies, protocols & algorithms** | [TECHNOLOGY.md](TECHNOLOGY.md) |

---

**Quick commands (no Makefile):**

```bash
# One-time
./bootstrap.sh
go mod download

# Build & run
mage build [binary1 binary2 ...]
mage start [binary1 binary2 ...]
mage check
mage stop

# Test & lint
go test ./...
go test -cover ./...
golangci-lint run ./...
gofmt -s -w . && goimports -w .
```

If you add a Makefile that wraps these (e.g. via gomake), you can align it with [docs/contrib/util-makefile.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/util-makefile.md) and [docs/contrib/cicd-actions.md](https://github.com/openimsdk/open-im-server/blob/main/docs/contrib/cicd-actions.md) so `make test`, `make cover`, `make lint` stay consistent for your team.
