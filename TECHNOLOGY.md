# OpenIM Server — Technology Reference

This document lists the main technologies used in OpenIM Server, what they do, how they communicate, and notable algorithms or patterns used to solve problems.

---

## Table of Contents

1. [Runtime & language](#1-runtime--language)
2. [API & HTTP layer](#2-api--http-layer)
3. [RPC & serialization](#3-rpc--serialization)
4. [Real-time & WebSocket](#4-real-time--websocket)
5. [Authentication & tokens](#5-authentication--tokens)
6. [Configuration](#6-configuration)
7. [Service discovery](#7-service-discovery)
8. [Message queue](#8-message-queue)
9. [Caching](#9-caching)
10. [Database](#10-database)
11. [Object storage](#11-object-storage)
12. [Scheduling](#12-scheduling)
13. [Monitoring & observability](#13-monitoring--observability)
14. [Push notifications](#14-push-notifications)
15. [Build & tooling](#15-build--tooling)
16. [Algorithms & patterns summary](#16-algorithms--patterns-summary)

---

## 1. Runtime & language

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Go (Golang) 1.22+** | Primary language for all server components. Used for API, RPC services, message gateway, transfer, push, and cron. | N/A | Concurrency via goroutines/channels; standard library and modules for I/O, HTTP, gRPC, etc. |

**Where used:** Entire codebase ([cmd/](https://github.com/openimsdk/open-im-server/tree/main/cmd), [internal/](https://github.com/openimsdk/open-im-server/tree/main/internal), [pkg/](https://github.com/openimsdk/open-im-server/tree/main/pkg)).

---

## 2. API & HTTP layer

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Gin** (`github.com/gin-gonic/gin`) | HTTP web framework. Handles REST routes, middleware (CORS, recovery, logging, token parsing), and JSON request/response. | **HTTP/1.1**, **JSON** over POST/GET. Clients call REST endpoints (e.g. `/user/user_register`, `/msg/send_msg`). | Router in [internal/api/router.go](https://github.com/openimsdk/open-im-server/blob/main/internal/api/router.go); handlers in [internal/api/](https://github.com/openimsdk/open-im-server/tree/main/internal/api). |
| **gzip** (`github.com/gin-contrib/gzip`) | Response compression middleware. | Same as HTTP; compresses response body (e.g. gzip). | Configurable level: Default/Best/BestSpeed/NoCompression in API config. |
| **go-playground/validator** | Struct validation for request bodies (e.g. `SendMsg`, user fields). | N/A | Used with Gin binding; custom rules (e.g. `required_if`) in router. |

**Protocols:** HTTP/1.1, TLS when fronted by HTTPS. Request/response: JSON.

---

## 3. RPC & serialization

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **gRPC** (`google.golang.org/grpc`) | RPC framework for service-to-service calls. API and gateway call auth, user, msg, group, friend, conversation, third via gRPC. | **HTTP/2**, **Protocol Buffers**. | Unary and streaming; ports per service (e.g. 10110 user, 10130 msg). |
| **Protocol Buffers** (`google.golang.org/protobuf`) | Binary serialization for gRPC messages and Kafka payloads. | Used over gRPC and inside Kafka message bodies. | Types from `github.com/openimsdk/protocol`; `proto.Marshal`/`Unmarshal` for Kafka. |
| **OpenIM Protocol** (`github.com/openimsdk/protocol`) | Shared .proto definitions and generated Go types for auth, user, msg, group, friend, conversation, push, third, etc. | Same as gRPC/Protobuf. | Defines request/response and SDK message types (e.g. `sdkws.MsgData`). |

**Protocols:** gRPC over HTTP/2; Protobuf binary encoding.

---

## 4. Real-time & WebSocket

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Gorilla WebSocket** (`github.com/gorilla/websocket`) | WebSocket server for long-lived client connections. Handles upgrade, binary/text frames, ping/pong, read/write timeouts. | **WebSocket** (RFC 6455), typically over HTTP/1.1 upgrade. Binary frames for IM protocol (send msg, pull msg, get seq, etc.). | [internal/msggateway/](https://github.com/openimsdk/open-im-server/tree/main/internal/msggateway); one connection per user/platform; max message size and pong wait configurable. |

**Protocols:** WebSocket over HTTP upgrade; custom binary protocol (ReqIdentifier + payload) for IM ops.

---

## 5. Authentication & tokens

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **JWT** (`github.com/golang-jwt/jwt/v4`) | Token format for auth (via `openimsdk/tools/tokenverify`). Issued and parsed by openim-rpc-auth. | Token string in HTTP header or WebSocket query/header; API and gateway call auth RPC to **ParseToken**. | Expiry and policy in config; admin vs user tokens. |
| **openimsdk/tools tokenverify** | Token creation and verification used by auth service. | N/A | Wraps JWT signing/parsing; used in [internal/rpc/auth](https://github.com/openimsdk/open-im-server/tree/main/internal/rpc/auth). |

**Protocols:** Tokens carried over HTTP headers or WebSocket; validation via gRPC to openim-rpc-auth.

---

## 6. Configuration

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Viper** (`github.com/spf13/viper`) | Config file loading (YAML), env binding, key/value access. | Reads from filesystem and environment variables. | Used in [pkg/common/config/load_config.go](https://github.com/openimsdk/open-im-server/blob/main/pkg/common/config/load_config.go); `OPENIMCONFIG` or `--config_folder_path` for config dir. |
| **YAML** (`gopkg.in/yaml.v3`) | Format for all service and shared configs (e.g. `config/*.yml`). | File-based; no network protocol. | One file per concern (redis, kafka, openim-api, openim-rpc-user, etc.). |

**Protocols:** File I/O and env vars; no network protocol for config itself.

---

## 7. Service discovery

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **etcd** (`go.etcd.io/etcd/client/v3`) | Distributed key-value store used for service discovery and (optionally) distributed locking. RPCs register themselves; API and gateway discover RPC endpoints. | **gRPC** to etcd cluster (v3 API). Keys used for service registration and lookup. | When `Discovery.Enable == ETCD`; standalone mode uses in-memory discovery. |
| **openimsdk/tools discovery** | Abstraction over etcd (and standalone). Provides `GetConn(ctx, serviceName)` for gRPC client connections. | Same as etcd when in use. | [pkg/common/discovery/discoveryregister.go](https://github.com/openimsdk/open-im-server/blob/main/pkg/common/discovery/discoveryregister.go); dial timeout and auth configurable. |

**Protocols:** etcd v3 API over gRPC; services register under configurable prefixes.

---

## 8. Message queue

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Apache Kafka** | Distributed log/queue. Used for message pipeline: openim-rpc-msg produces to **ToRedisTopic**; openim-msgtransfer consumes it and produces to **ToMongoTopic** and **ToPushTopic**; openim-push consumes **ToPushTopic**. | **Kafka protocol** (TCP); producer/consumer groups. | Topics: ToRedisTopic, ToMongoTopic, ToPushTopic, ToOfflinePushTopic (config-driven names). |
| **IBM Sarama** (`github.com/IBM/sarama`) | Go client for Kafka (producers and consumers). | Kafka protocol. | Used via [pkg/mqbuild](https://github.com/openimsdk/open-im-server/tree/main/pkg/mqbuild) and storage controllers; sync producer for msg, consumer groups for transfer and push. |

**Protocols:** Kafka wire protocol over TCP; message key (e.g. conversation key) for partitioning and ordering.

**Algorithms / patterns:**
- **Partitioning by key:** Messages produced with a key (e.g. conversation ID) so that all messages for the same conversation go to the same partition and are consumed in order.
- **Consumer groups:** Multiple transfer/push instances share consumption via consumer groups for scalability.

---

## 9. Caching

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Redis** (`github.com/redis/go-redis/v9`) | In-memory store for cache and transient data: user/group/friend/conversation/msg cache, token cache, seq, RocksCache metadata. | **Redis protocol** (RESP over TCP). | Single or cluster; used by RPCs and msgtransfer. |
| **RocksCache** (`github.com/dtm-labs/rockscache`) | Cache-aside layer on top of Redis with strong consistency: prevents stale reads after DB updates by invalidating/locking cache keys. | Uses Redis for storage and locks; app talks to Redis. | Options: `StrongConsistency`, `LockExpire`, `WaitReplicasTimeout`, `RandomExpireAdjustment`; used for user, group, friend, conversation, msg, etc. |
| **Local in-memory cache** (OpenIM [pkg/localcache](https://github.com/openimsdk/open-im-server/tree/main/pkg/localcache)) | Process-local cache in front of Redis/DB to reduce latency and load. Uses LRU eviction and optional TTL. | In-process only; no network. | Slot-based sharding (hash) and optional “link” invalidation; see [Algorithms](#16-algorithms--patterns-summary). |

**Protocols:** Redis RESP over TCP; localcache is in-process only.

**Algorithms / patterns:**
- **RocksCache:** Cache-aside with locking and optional wait-for-replicas to avoid returning stale data after DB writes.
- **LRU eviction:** See [§16](#16-algorithms--patterns-summary).

---

## 10. Database

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **MongoDB** (`go.mongodb.org/mongo-driver`) | Primary persistent store for users, groups, friends, conversations, messages (history), auth-related data, and other documents. | **MongoDB wire protocol** (TCP). | Used by all RPCs that need persistence; msgtransfer persists messages from Kafka to MongoDB. |

**Protocols:** MongoDB wire protocol; driver handles BSON encoding.

---

## 11. Object storage

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **MinIO** | S3-compatible object storage for file uploads (images, voice, etc.). Default option. | **HTTP** (S3 API) or **HTTPS**. | Configured in [config/minio.yml](https://github.com/openimsdk/open-im-server/blob/main/config/minio.yml); openim-rpc-third uses it for upload/sign/complete. |
| **AWS S3** | Optional object storage backend. | S3 REST API over HTTP/HTTPS. | Via `openimsdk/tools` storage abstraction; used when configured. |
| **Tencent COS, Aliyun OSS, Qiniu Kodo** | Optional object storage backends for different clouds. | Vendor REST APIs over HTTP/HTTPS. | Same abstraction; config in openim-rpc-third and env docs. |

**Protocols:** REST over HTTP/HTTPS (vendor-specific APIs).

---

## 12. Scheduling

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **robfig/cron** (`github.com/robfig/cron/v3`) | Cron-style job scheduler. Runs periodic tasks (e.g. clear chat records, delete messages, clear user msg, S3 cleanup). | In-process; no network protocol. | [internal/tools/cron/cron_task.go](https://github.com/openimsdk/open-im-server/blob/main/internal/tools/cron/cron_task.go); expressions from config (e.g. `CronExecuteTime`). |
| **etcd (optional)** | Distributed lock so only one cron instance runs a task in a multi-instance deployment. | etcd gRPC. | When discovery is etcd; `ExecuteWithLock` wraps task execution. |

**Algorithms / patterns:**
- **Cron expressions:** Standard cron syntax (e.g. `0 0 2 * * *` for daily at 2 AM) to trigger jobs.
- **Distributed locking:** Single active executor per task across instances using etcd.

---

## 13. Monitoring & observability

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Prometheus** (`github.com/prometheus/client_golang`) | Metrics collection and exposition. Counters/gauges for RPC, API, and business events (e.g. message success/failure). | **HTTP** scrape (Prometheus pulls from `/metrics`). | Optional per service; ports in config (e.g. 20110 for user RPC). |
| **grpc-prometheus** (`github.com/grpc-ecosystem/go-grpc-prometheus`) | gRPC server/client interceptors that record latency and call counts. | Same as gRPC; metrics exposed via Prometheus HTTP. | Wired in RPC servers and clients. |
| **OpenIM tools log** (`github.com/openimsdk/tools/log`) | Structured logging (levels, context, key-value). | Logs to stdout/files; no network protocol. | Replaces standard `log`; used across the codebase. |

**Protocols:** Prometheus scrape over HTTP; logs are local/output.

---

## 14. Push notifications

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Firebase Cloud Messaging (FCM)** (`firebase.google.com/go/v4`) | Push to Android/iOS via FCM. | **HTTP/2** to FCM APIs. | Used when push is enabled; config (e.g. service account) in openim-push. |
| **Getui, JPush** | Third-party push providers (optional). | Vendor HTTP APIs. | Tokens and task IDs cached in Redis; openim-push sends to provider APIs. |
| **In-app / WebSocket** | For online users, push service finds gateway connection and pushes via gRPC or in-memory to msggateway, which sends WebSocket frames. | gRPC or in-process to gateway; then **WebSocket** to client. | Avoids external push when user is connected. |

**Protocols:** HTTP/2 (FCM); vendor REST (Getui/JPush); WebSocket for online delivery.

---

## 15. Build & tooling

| Technology | Description | Communication | Notes |
|------------|-------------|----------------|-------|
| **Mage** (`github.com/magefile/mage`) | Build and run automation (replaces Make in this repo). Targets: build, start, check, stop. | Invoked locally; no network. | `magefile.go`; uses `openimsdk/gomake` for actual build/start logic. |
| **Golangci-lint** | Static analysis and linting for Go. | N/A | Config in `.golangci.yml`; run in CI and locally. |
| **goimports / gofmt** | Code formatting and import organization. | N/A | Part of code-style checks. |

---

## 16. Algorithms & patterns summary

These are the main algorithms and design patterns used to solve specific problems in the codebase.

### 16.1 Message batching (Batcher)

- **Where:** [pkg/tools/batcher](https://github.com/openimsdk/open-im-server/tree/main/pkg/tools/batcher); used by openim-msgtransfer when consuming Kafka ToRedisTopic.
- **Problem:** Process high-throughput Kafka messages efficiently without handling every message individually.
- **Approach:**
  - **Time- and size-based batching:** A scheduler collects items until either (1) a configured **count** is reached or (2) a **time interval** elapses (ticker), then flushes.
  - **Sharding by key:** A **Sharding(key)** function (e.g. `hash(key) % workerCount`) assigns each key to a worker channel so that all messages for the same key (e.g. conversation) are processed by the same worker, preserving order and reducing lock contention.
  - **Sync wait (optional):** After distributing a batch to workers, the scheduler can wait for all workers to finish before marking the Kafka offset (sync wait), improving at-least-once semantics.
- **Config:** Size, interval, worker count, buffer sizes (see [internal/msgtransfer/online_history_msg_handler.go](https://github.com/openimsdk/open-im-server/blob/main/internal/msgtransfer/online_history_msg_handler.go) and batcher options).

### 16.2 LRU and local cache

- **Where:** [pkg/localcache](https://github.com/openimsdk/open-im-server/tree/main/pkg/localcache) and [pkg/localcache/lru](https://github.com/openimsdk/open-im-server/tree/main/pkg/localcache/lru).
- **Problem:** Reduce latency and load on Redis/DB with a process-local cache while controlling memory and staleness.
- **Approaches:**
  - **LRU eviction:** When the cache is full, least-recently-used entries are evicted (via `hashicorp/golang-lru`: `simplelru` or `expirable`).
  - **Lazy vs expiration-based TTL:**  
    - **LazyLRU:** Expiry is checked on access; expired entries are evicted when read.  
    - **ExpirationLRU:** A background process or expirable LRU evicts when TTL is reached.
  - **Slot-based sharding (SlotLRU):** Keys are hashed to a slot index (`hash(key) % slotNum`). Each slot has its own LRU. This reduces lock contention when many goroutines access the cache and keeps ordering per key when combined with hash consistency.
  - **Link invalidation:** Optional “link” structure records key relationships; when a key is evicted or deleted, related keys can be removed as well (e.g. “when user cache is invalidated, also invalidate friend list cache”).
- **Hash:** For string keys, a hash function (e.g. from `openimsdk/tools/utils/stringutil` or local `LRUStringHash`) is used for slot selection.

### 16.3 RocksCache (strong cache consistency)

- **Where:** [pkg/common/storage/cache/redis](https://github.com/openimsdk/open-im-server/tree/main/pkg/common/storage/cache/redis); used by user, group, friend, conversation, msg, and other caches.
- **Problem:** Avoid returning stale data from cache after the database has been updated (cache-aside consistency).
- **Approach:**
  - **RocksCache** (dtm-labs/rockscache): For each key, a **lock** is used around “fetch from DB and fill cache.” Other requesters **wait** on the same key’s lock or read replicas (if configured) so they see the updated value.
  - **Options:** `StrongConsistency`, `LockExpire`, `WaitReplicasTimeout`, `RandomExpireAdjustment` to balance consistency and latency.
  - **Batch deleter:** Invalidations can be published to Redis channels so other instances drop their local or Redis cache entries (topic-based invalidation).

### 16.4 BBR rate limiting

- **Where:** [internal/api/ratelimit.go](https://github.com/openimsdk/open-im-server/blob/main/internal/api/ratelimit.go); uses `openimsdk/tools/stability/ratelimit/bbr`.
- **Problem:** Protect the API from overload and improve throughput under load.
- **Approach:** **BBR (Bandwidth-Delay)**-inspired limiter: uses observed RTT and throughput to adapt a max-in-flight (or similar) limit. When the limit is exceeded, requests get HTTP 429 and a `Retry-After` header. Headers expose BBR stats (CPU, MinRT, MaxPass, InFlight) for debugging.
- **Config:** Window, bucket count, CPU threshold in API YAML.

### 16.5 Hash-based sharding and IDs

- **Where:** Batcher sharding, SlotLRU, conversation/user key generation, and various utilities.
- **Usage:**
  - **Conversation key:** Deterministic key from (sendID, recvID) or groupID so that all messages for a conversation go to the same Kafka partition and same batcher shard.
  - **ID hashing:** [pkg/util/hashutil/id.go](https://github.com/openimsdk/open-im-server/blob/main/pkg/util/hashutil/id.go) uses **MD5** over JSON-marshaled IDs and takes the first 8 bytes (BigEndian) as uint64 for a compact hash (e.g. for dedup or sharding).
  - **String hash for sharding:** From `openimsdk/tools/utils/stringutil.GetHashCode` (or similar) to map keys to batcher workers or LRU slots.

### 16.6 Cron and distributed lock

- **Where:** [internal/tools/cron](https://github.com/openimsdk/open-im-server/tree/main/internal/tools/cron); cron tasks (clear messages, S3, etc.).
- **Approach:** **Cron expressions** define schedule; **etcd lock** ensures only one instance executes a given task when multiple openim-crontask instances run (leader election style).

### 16.7 Read-receipt and seq aggregation

- **Where:** [internal/msgtransfer/online_history_msg_handler.go](https://github.com/openimsdk/open-im-server/blob/main/internal/msgtransfer/online_history_msg_handler.go) (`doSetReadSeq`).
- **Problem:** Persist “has read up to seq” per user per conversation without writing on every single message.
- **Approach:** In the transfer handler, messages in a batch are scanned for read-receipt notifications; **per (conversationID, userID)** the **maximum** hasReadSeq is computed, then written once to DB (and cache) for the batch, reducing write volume.

---

## Quick reference: communication protocols

| Layer | Protocol(s) |
|-------|-------------|
| Client ↔ API | HTTP/1.1, JSON; optional TLS |
| Client ↔ Gateway | WebSocket (binary IM protocol) |
| API/Gateway ↔ RPC | gRPC (HTTP/2, Protobuf) |
| RPC ↔ etcd | gRPC (etcd v3 API) |
| RPC/Transfer ↔ Redis | Redis RESP (TCP) |
| RPC/Transfer ↔ MongoDB | MongoDB wire (TCP) |
| RPC/Transfer/Push ↔ Kafka | Kafka protocol (TCP) |
| RPC ↔ Object storage | S3-compatible REST (HTTP/HTTPS) |
| Push ↔ FCM/Getui/JPush | HTTP/2 or REST |
| Prometheus ↔ Services | HTTP scrape |

---

For request flow and sequence diagrams, see [PROCESS_FLOW.md](PROCESS_FLOW.md). For development setup and code layout, see [DEVELOPER.md](README.md).
