# OpenIM Server — Process Flow

This document describes where requests start, how they move through the system, and where they end. Diagrams are in Mermaid (render in GitHub, VS Code, or any Mermaid-capable viewer).

---

## 1. High-level architecture

```mermaid
sequenceDiagram
    autonumber

    participant REST as HTTP Client
    participant WS as WebSocket Client
    participant API as openim-api (REST Gateway)
    participant GW as openim-msggateway (WebSocket)
    participant AUTH as openim-rpc-auth
    participant USER as openim-rpc-user
    participant MSG as openim-rpc-msg
    participant GROUP as openim-rpc-group
    participant FRIEND as openim-rpc-friend
    participant CONV as openim-rpc-conversation
    participant THIRD as openim-rpc-third
    participant KAFKA as Kafka
    participant XFER as openim-msgtransfer
    participant REDIS as Redis
    participant MONGO as MongoDB
    participant PUSH as openim-push
    participant ETCD as etcd

    rect rgb(240,240,240)
    REST->>API: HTTP request (login / send msg / etc)
    API->>AUTH: auth RPC
    API->>USER: user RPC
    API->>MSG: send message RPC
    API->>GROUP: group RPC
    API->>FRIEND: friend RPC
    API->>CONV: conversation RPC
    API->>THIRD: third-party RPC
    end

    rect rgb(240,240,255)
    WS->>GW: WebSocket connect / send
    GW->>AUTH: auth RPC
    GW->>MSG: send message RPC
    end

    rect rgb(240,255,240)
    MSG->>KAFKA: publish message
    KAFKA->>XFER: consume message
    XFER->>REDIS: cache/update state
    XFER->>MONGO: persist message
    XFER->>KAFKA: forward delivery event
    KAFKA->>PUSH: push notification
    end

    rect rgb(255,245,230)
    API-->>ETCD: service discovery
    GW-->>ETCD: service discovery
    MSG-->>ETCD: service discovery
    PUSH-->>ETCD: service discovery
    end
```

- **Request start:** HTTP at `openim-api` or WebSocket at `openim-msggateway`.
- **Request end:** Either a response back to the client (sync) or completion of async pipeline (Kafka → msgtransfer → Redis/Mongo/Push).

---

## 2. Request entry points and destinations

| Entry point | Typical use | Ends at |
|-------------|-------------|---------|
| **openim-api** (REST) | Register, login, send msg, sync, friends, groups, conversation, object upload | openim-api returns HTTP response after calling one or more RPCs |
| **openim-msggateway** (WebSocket) | Send msg, pull msg, get seq, logout, subscribe status | openim-msggateway returns binary frame; for send/pull it calls openim-rpc-msg |
| **Kafka (ToRedisTopic)** | Produced by openim-rpc-msg | Consumed by openim-msgtransfer |
| **Kafka (ToMongoTopic)** | Produced by openim-msgtransfer | Consumed by openim-msgtransfer (persist to MongoDB) |
| **Kafka (ToPushTopic)** | Produced by openim-msgtransfer | Consumed by openim-push → delivery to devices / gateway |

---

## 3. Authentication flow (token)

Every API and gateway request that needs auth goes through token parsing first.

```mermaid
sequenceDiagram
    participant Client
    participant API as openim-api
    participant GW as openim-msggateway
    participant Auth as openim-rpc-auth
    participant ETCD as etcd

    Client->>API: HTTP + Token (header)
    API->>ETCD: resolve auth service
    API->>Auth: ParseToken(token)
    Auth-->>API: userID, platformID, ...
    API->>API: route to handler (user/msg/group/...)

    Note over Client,ETCD: WebSocket path
    Client->>GW: WS connect + token (query/header)
    GW->>ETCD: resolve auth service
    GW->>Auth: ParseToken(token)
    Auth-->>GW: userID, platformID
    GW->>GW: register connection, handle frames
```

- **Starts at:** Client (HTTP request or WebSocket upgrade).
- **Ends at:** openim-api or openim-msggateway has user identity; subsequent logic uses it (e.g. call msg RPC with SendID).

---

## 4. Send message — REST path

From HTTP POST to response and to Kafka.

```mermaid
sequenceDiagram
    participant Client
    participant API as openim-api
    participant Auth as openim-rpc-auth
    participant Msg as openim-rpc-msg
    participant Kafka as Kafka (ToRedisTopic)

    Client->>API: POST /msg/send_msg (body: MsgData, token)
    API->>Auth: ParseToken (middleware)
    Auth-->>API: userID
    API->>API: build SendMsgReq (MsgData)
    API->>Msg: SendMsg(SendMsgReq)
    Note over Msg: verify, webhooks, sessionType
    Msg->>Msg: sendMsgSingleChat / sendMsgGroupChat / sendMsgNotification
    Msg->>Kafka: MsgToMQ(key, MsgData)
    Kafka-->>Msg: ack
    Msg-->>API: SendMsgResp (serverMsgID, clientMsgID, sendTime)
    API-->>Client: 200 + SendMsgResp
```

- **Starts at:** Client HTTP POST to `/msg/send_msg`.
- **Ends at:**  
  - **Sync:** Client gets HTTP 200 + `SendMsgResp`.  
  - **Async:** Message is in Kafka `ToRedisTopic`; msgtransfer will consume it (see below).

---

## 5. Send message — WebSocket path

From WebSocket binary frame to response and to Kafka.

```mermaid
sequenceDiagram
    participant Client
    participant GW as openim-msggateway
    participant Msg as openim-rpc-msg
    participant Kafka as Kafka (ToRedisTopic)

    Client->>GW: Binary frame (ReqIdentifier=WSSendMsg, MsgData)
    GW->>GW: handleMessage: decode, validate SendID
    GW->>Msg: SendMsg(SendMsgReq)
    Msg->>Msg: sendMsgSingleChat / sendMsgGroupChat / sendMsgNotification
    Msg->>Kafka: MsgToMQ(key, MsgData)
    Kafka-->>Msg: ack
    Msg-->>GW: SendMsgResp
    GW->>GW: marshal SendMsgResp
    GW->>Client: Binary frame (ReqIdentifier=WSSendMsg, resp)
```

- **Starts at:** Client sends a WebSocket binary message with `WSSendMsg` and payload.
- **Ends at:**  
  - **Sync:** Client receives a WebSocket frame with the send response.  
  - **Async:** Same as REST: message is in Kafka `ToRedisTopic`.

---

## 6. Message pipeline (Kafka → storage and push)

After openim-rpc-msg writes to Kafka, openim-msgtransfer and openim-push complete the flow.

```mermaid
sequenceDiagram
    participant Msg as openim-rpc-msg
    participant K1 as Kafka ToRedisTopic
    participant XFER as openim-msgtransfer
    participant Redis as Redis
    participant K2 as Kafka ToMongoTopic
    participant K3 as Kafka ToPushTopic
    participant Mongo as MongoDB
    participant PUSH as openim-push
    participant GW as openim-msggateway

    Msg->>K1: MsgToMQ(key, MsgData)
    K1->>XFER: consume (batch by key)
    XFER->>XFER: handleMsg / handleNotification
    XFER->>Redis: BatchInsertChat2Cache (conversationID, msgs)
    XFER->>Redis: SetHasReadSeqs (read seq)
    XFER->>XFER: SetHasReadSeqToDB (async, to Mongo)
    XFER->>K2: MsgToMongoMQ (batch)
    XFER->>K3: MsgToPushMQ (per message)
    K2->>XFER: consume ToMongoTopic
    XFER->>Mongo: BatchInsertChat2DB
    K3->>PUSH: consume ToPushTopic
    PUSH->>PUSH: online? → find gateway conn
    PUSH->>GW: PushMessage (gRPC or in-memory)
    GW->>GW: PushMessage to client WS
```

- **Starts at:** openim-rpc-msg produces to **ToRedisTopic** (end of send flow above).
- **Ends at:**  
  - **Redis:** Conversation cache and has-read seq updated.  
  - **MongoDB:** Message persisted via ToMongoTopic consumer.  
  - **Push:** ToPushTopic → openim-push → online users via gateway (WebSocket); offline via FCM/other.

---

## 7. End-to-end send message (single flow)

Single diagram from client send to storage and push.

```mermaid
sequenceDiagram
    participant Client
    participant GW as openim-msggateway
    participant Msg as openim-rpc-msg
    participant K as Kafka
    participant XFER as openim-msgtransfer
    participant Redis as Redis
    participant Mongo as MongoDB
    participant PUSH as openim-push

    Client->>GW: Send message (WS)
    GW->>Msg: SendMsg
    Msg->>K: ToRedisTopic
    Msg-->>GW: SendMsgResp
    GW-->>Client: WS response

    K->>XFER: ToRedisTopic
    XFER->>Redis: cache + seq
    XFER->>K: ToMongoTopic, ToPushTopic
    K->>XFER: ToMongoTopic
    XFER->>Mongo: persist
    K->>PUSH: ToPushTopic
    PUSH->>Client: push (e.g. WS or FCM)
```

---

## 8. Pull message / get seq (WebSocket)

Client reads history or latest seq via gateway.

```mermaid
sequenceDiagram
    participant Client
    participant GW as openim-msggateway
    participant Msg as openim-rpc-msg
    participant Redis as Redis
    participant Mongo as MongoDB

    Client->>GW: WSPullMsg / WSPullMsgBySeqList / WSGetNewestSeq
    GW->>Msg: GetMaxSeq / PullMessageBySeqList / GetSeq (etc.)
    Msg->>Redis: get from cache (or fallback)
    alt cache miss
        Msg->>Mongo: query
    end
    Msg-->>GW: seq list or messages
    GW-->>Client: binary frame (messages or seq)
```

- **Starts at:** Client sends WebSocket request (e.g. pull by seq list or get newest seq).
- **Ends at:** Client receives messages or seq from openim-msggateway; data comes from openim-rpc-msg (Redis/Mongo).

---

## 9. Pull message / get seq (REST)

Same logical flow via HTTP.

```mermaid
sequenceDiagram
    participant Client
    participant API as openim-api
    participant Msg as openim-rpc-msg
    participant Redis as Redis
    participant Mongo as MongoDB

    Client->>API: POST /msg/pull_msgs_by_seq_list (or get_max_seq, etc.)
    API->>Msg: PullMessageBySeqList / GetMaxSeq / ...
    Msg->>Redis: (and Mongo if needed)
    Msg-->>API: messages or seq
    API-->>Client: 200 + JSON
```

- **Starts at:** Client HTTP POST to `/msg/*` (pull/get seq).
- **Ends at:** Client gets HTTP 200 with messages or seq.

---

## 10. Other API flows (summary)

| Flow | Start | End |
|------|--------|-----|
| **User register / update** | POST /user/* → openim-api | openim-rpc-user → MongoDB/Redis; response to client |
| **Friend add / list** | POST /friend/* → openim-api | openim-rpc-friend → DB; response to client |
| **Group create / invite** | POST /group/* → openim-api | openim-rpc-group (and sometimes user/relation); response to client |
| **Conversation** | POST /conversation/* → openim-api | openim-rpc-conversation; response to client |
| **Object upload / sign** | POST /object/*, /third/* → openim-api | openim-rpc-third (+ MinIO/OSS/COS); response to client |
| **Token / force logout** | POST /auth/* → openim-api | openim-rpc-auth; response to client |

All of these: **start** at client → openim-api; **end** at openim-api returning HTTP response after RPC and optional DB writes.

---

## 11. Kafka topics (summary)

| Topic | Producer | Consumer | Purpose |
|-------|----------|----------|---------|
| **ToRedisTopic** | openim-rpc-msg | openim-msgtransfer | New messages to cache and forward |
| **ToMongoTopic** | openim-msgtransfer | openim-msgtransfer | Persist messages to MongoDB |
| **ToPushTopic** | openim-msgtransfer | openim-push | Deliver to online/offline users |
| **ToOfflinePushTopic** | openim-push (internal) | openim-push | Offline push pipeline (if used) |

---

## 12. Service discovery

- **openim-api** and **openim-msggateway** (and other RPCs) use **etcd** to discover RPC endpoints.
- On startup they get gRPC connections to openim-rpc-auth, openim-rpc-msg, openim-rpc-user, etc., via `discovery.SvcDiscoveryRegistry` / `GetConn(ctx, serviceName)`.
- So every “API → RPC” or “gateway → RPC” arrow in the diagrams goes through discovery (etcd) to get the target address.

---

## 13. Where to look in code

| What you trace | Where to look |
|----------------|----------------|
| REST routes and which RPC is called | `internal/api/router.go`, `internal/api/*.go` (e.g. `msg.go`, `user.go`) |
| WebSocket message types and handler | `internal/msggateway/client.go` (handleMessage switch), `internal/msggateway/message_handler.go` |
| SendMsg and MsgToMQ | `internal/rpc/msg/send.go`, `pkg/common/storage/controller/msg.go` |
| Kafka producer (ToRedisTopic) | `internal/rpc/msg/server.go` (builder.GetTopicProducer ToRedisTopic), `MsgToMQ` in msg.go |
| Consume ToRedisTopic, write Redis/Mongo/Push | `internal/msgtransfer/online_history_msg_handler.go`, `internal/msgtransfer/init.go` |
| Consume ToMongoTopic, persist | `internal/msgtransfer/online_msg_to_mongo_handler.go`, `init.go` |
| Consume ToPushTopic, push to user | `internal/push/push.go`, push handler |
| Token parsing | `internal/api/router.go` (GinParseToken), `internal/msggateway/ws_server.go` (ParseToken), `internal/rpc/auth` |

Use this with [DEVELOPER.md](README.md) to follow a request from entry (API or gateway) to RPC, Kafka, storage, and push.
