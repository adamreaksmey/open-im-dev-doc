# OpenIM Server — Infrastructure Dependencies and Docker

OpenIM Server is **not** a single process. It depends on several **external infrastructure components**. You need these running before (or alongside) the OpenIM binaries. This document explains what they are, why they’re needed, and how to run them—typically with **Docker** and the repo’s **docker-compose**.

**Note:** OpenIM uses **Kafka** for the message pipeline, not RabbitMQ. There is no RabbitMQ in this project.

---

## Table of Contents

1. [Do I need Docker?](#1-do-i-need-docker)
2. [What infrastructure does OpenIM need?](#2-what-infrastructure-does-openim-need)
3. [Quick start: run infrastructure with Docker Compose](#3-quick-start-run-infrastructure-with-docker-compose)
4. [Component details](#4-component-details)
5. [Matching config to Docker](#5-matching-config-to-docker)
6. [Running without Docker](#6-running-without-docker)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Do I need Docker?

- **For local development and full-stack testing:** **Yes, in practice.** The standard way to run Kafka, Redis, MongoDB, etcd, and MinIO locally is via **Docker** and the repo’s **docker-compose.yml**. OpenIM does not bundle or start these for you.
- **For production:** You can run the same components on VMs or managed services (e.g. AWS MSK, ElastiCache, DocumentDB, etc.). Docker is not required there, but the **components** (Kafka, Redis, MongoDB, etcd, MinIO or S3) are still required.
- **For unit tests only:** Some tests may mock or skip external services; you might not need real Kafka/Redis/Mongo for pure unit tests. For integration or E2E, you need the real infrastructure (usually via Docker).

So: **you need something running Kafka, Redis, MongoDB, etcd, and (for uploads) MinIO.** Docker is the documented and supported way to do that for local/dev.

---

## 2. What infrastructure does OpenIM need?

| Component   | Purpose | Used by |
|------------|---------|--------|
| **MongoDB** | Persistent storage: users, groups, friends, conversations, message history, auth data. | All RPCs that persist data; openim-msgtransfer (message persistence). |
| **Redis**   | Cache (user, group, friend, conversation, msg, token, seq), RocksCache metadata, session/token. | openim-rpc-* (auth, user, msg, group, etc.), openim-msgtransfer. |
| **Kafka**   | Message pipeline: openim-rpc-msg produces to **ToRedisTopic**; openim-msgtransfer consumes and produces to **ToMongoTopic** / **ToPushTopic**; openim-push consumes **ToPushTopic**. | openim-rpc-msg, openim-msgtransfer, openim-push. |
| **etcd**    | Service discovery: RPCs register; openim-api and openim-msggateway discover RPC addresses. (Optional: distributed lock for cron.) | All services when not in “standalone” mode. |
| **MinIO**   | Object storage for uploads (images, voice, files). Can be replaced by S3/OSS/COS/Kodo via config. | openim-rpc-third. |

**Not used:** RabbitMQ. OpenIM uses **Kafka** only for the message queue.

---

## 3. Quick start: run infrastructure with Docker Compose

### 3.1 Prerequisites

- **Docker** and **Docker Compose** installed.
- Repo cloned; you’re in the repo root.

### 3.2 Environment variables

The repo’s **docker-compose.yml** reads images and paths from environment variables. A sample **.env** exists in the repo root (see `.env`). You can copy and adjust it:

```bash
cp .env .env.local
# Edit .env.local if you need different images or DATA_DIR
```

Important variables (defaults in `.env`):

- `MONGO_IMAGE`, `REDIS_IMAGE`, `KAFKA_IMAGE`, `MINIO_IMAGE`, `ETCD_IMAGE` — images for each component.
- `DATA_DIR` — where container data is stored (default `./`).
- Optional: `KAFKA_USERNAME`, `KAFKA_PASSWORD` for Kafka SASL (commented out by default).

### 3.3 Start only the infrastructure (no OpenIM binaries yet)

From the repo root:

```bash
docker compose up -d mongodb redis etcd kafka minio
```

This starts:

- **mongodb** — port 37017 (mapped from 27017)
- **redis** — port 16379 (mapped from 6379)
- **etcd** — ports 12379, 12380
- **kafka** — port 19094 (external listener)
- **minio** — ports 10005 (API), 19090 (console)

Check that containers are up:

```bash
docker compose ps
```

### 3.4 Then run OpenIM Server

1. Ensure **config/** has the right connection settings for Redis, MongoDB, Kafka, etcd, MinIO (see [§5](#5-matching-config-to-docker)).
2. Start OpenIM (e.g. `mage start` or run the binaries you need). They will connect to the services started above (using `localhost` and the ports from docker-compose when running on the same machine).

### 3.5 Optional: web front and monitoring

- **OpenIM Web UI:**  
  `docker compose up -d openim-web-front`  
  (port 11001)

- **Monitoring (Prometheus, Grafana, etc.):**  
  These are under the `m` profile:  
  `docker compose --profile m up -d prometheus alertmanager grafana node-exporter`

---

## 4. Component details

### 4.1 MongoDB

- **Port:** 37017 (host) in docker-compose.
- **Default credentials:** root / openIM123; OpenIM user/database created by the entrypoint (see docker-compose).
- **Config:** `config/mongodb.yml` (address, port, username, password, database).

### 4.2 Redis

- **Port:** 16379 (host).
- **Password:** openIM123 (set in docker-compose command).
- **Config:** `config/redis.yml`.

### 4.3 Kafka

- **Port:** 19094 (host) for external clients.
- **No ZooKeeper:** This setup uses KRaft (no ZK). Topics (ToRedisTopic, ToMongoTopic, ToPushTopic, etc.) are created automatically if `KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE` is true.
- **Config:** `config/kafka.yml` (address, topic names, optional SASL).

### 4.4 etcd

- **Ports:** 12379 (client), 12380 (peer).
- **Config:** `config/discovery.yml` — set `enable: etcd` and `etcd.address` (e.g. `localhost:12379`). For standalone mode (no etcd), OpenIM uses in-memory discovery.

### 4.5 MinIO

- **Ports:** 10005 (API), 19090 (console).
- **Credentials:** root / openIM123 in docker-compose.
- **Config:** `config/minio.yml`; openim-rpc-third uses this for upload/sign. You can switch to S3/OSS/COS/Kodo via config.

---

## 5. Matching config to Docker

OpenIM reads connection info from **config/*.yml**. For a local Docker setup, those files must point to the **host** (e.g. `localhost`) and the **host ports** used in docker-compose:

| Config file       | What to set for local Docker |
|-------------------|------------------------------|
| `config/redis.yml`     | Address/host: `localhost` (or host.docker.internal from another stack), port: `16379`, password: `openIM123`. |
| `config/mongodb.yml`   | Address: `localhost`, port: `37017`, username/password/database as in docker-compose (e.g. openIM / openIM123 / openim_v3). |
| `config/kafka.yml`     | Address: `localhost:19094` (or the advertised listener you use). Topic names as in your config (e.g. ToRedisTopic, ToMongoTopic, ToPushTopic). |
| `config/discovery.yml` | `enable: etcd`, `etcd.address: [localhost:12379]`. |
| `config/minio.yml`     | Endpoint for MinIO (e.g. `http://localhost:10005`), credentials root/openIM123, bucket/external URL as needed. |

If you use the **init-config** or **environment** scripts (see [docs/contrib/init-config.md](contrib/init-config.md), [docs/contrib/environment.md](contrib/environment.md)), they may generate or override these from environment variables. Adjust either the generated config or the env vars so they match the ports and credentials from docker-compose.

---

## 6. Running without Docker

You can run each component yourself (e.g. Kafka, Redis, MongoDB, etcd, MinIO installed on the host or in VMs):

- Install and start each service with the same ports and credentials (or adjust OpenIM config to match).
- Ensure Kafka has the expected topics (or auto-create enabled) and that OpenIM’s `config/kafka.yml` points to the correct broker(s).
- For etcd, set `config/discovery.yml` to your etcd endpoint.

The **logic** of what OpenIM needs does not change; only the way you run the infrastructure (Docker vs bare metal vs managed services).

---

## 7. Troubleshooting

- **“Connection refused” to Redis/Mongo/Kafka/etcd:**  
  Confirm the component is running (`docker compose ps`) and that the **config/*.yml** host and port match the **host** view (e.g. `localhost` and 16379 for Redis when using docker-compose ports).

- **Kafka “topic does not exist” or producer/consumer errors:**  
  Check `config/kafka.yml` (broker address, topic names). Ensure Kafka is up and, if needed, that auto-create is enabled or create the topics manually.

- **Service discovery (etcd):**  
  If you use etcd, ensure `config/discovery.yml` has the correct etcd address. If you intend to run without etcd, use standalone mode (e.g. single-binary runner with in-memory discovery) as documented in the repo.

- **MinIO uploads fail:**  
  Check `config/minio.yml` (endpoint, credentials) and that MinIO is reachable at the configured URL.

For more on config files and env vars, see [config/README.md](../config/README.md) and [docs/contrib/environment.md](contrib/environment.md). For request flow and which service talks to which component, see [PROCESS_FLOW.md](PROCESS_FLOW.md) and [TECHNOLOGY.md](TECHNOLOGY.md).
