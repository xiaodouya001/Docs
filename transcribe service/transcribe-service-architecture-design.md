# Transcribe Service — Architecture Design Document

> **Document Type**: Consolidated Architecture Design  
> **Scope**: Real-time Speech-to-Text Gateway (GCP ↔ AWS)  
> **Version**: 1.0  
> **Review**: Architecture Perspective

---

## Document Index

| Part | Section | Content |
|------|---------|---------|
| I | Executive Summary | Business context, system boundary, core objectives |
| II | Physical & Network Architecture | Multi-cloud topology, VPC isolation, zero-trust |
| III | Core Application Logic | Tech stack, connection lifecycle, sequence guard, 2PC |
| IV | Storage & Protocol | Kafka, Redis, message schemas |
| V | High Availability & Resilience | Rolling TTL, backpressure, circuit breaker, graceful shutdown |
| VI | Infrastructure & Capacity | Fargate, ALB, auto-scaling, MSK |
| VII | API Contract & Error Codes | WebSocket protocol, request/response, error mapping |
| VIII | Observability & Operations | Logging, monitoring, FMEA |
| IX | Security & Compliance | IAM, encryption, audit |
| X | Performance Baseline & Extensibility | Targets, IAM, evolution roadmap |
| XI | FAQ & Technical Justification | Async vs threading, blocking risks |
| XII | Implementation Reference | Lua scripts, state manager, development requirements |
| Appendix | API Contract Full Specification | Complete request/response schemas |

---

# Part I — Executive Summary & System Boundary

## 1.1 Business Background

In the context of Call Center intelligent upgrade, the bank has introduced a third-party ASR (Automatic Speech Recognition) engine from FanoLab. The engine is **hosted and deployed in the bank's GCP environment**, converting real-time agent-customer calls into text transcripts.

**Transcribe Service (Service B)** is deployed in the bank's AWS environment and serves as the **core multi-cloud real-time data gateway** connecting GCP and AWS internal data ecosystem.

## 1.2 Business Objectives

| Objective | Target |
|-----------|--------|
| **Concurrent capacity** | 700–1,000 concurrent calls at peak (design target) |
| **End-to-end latency** | < 50 ms (GCP receive → AWS Kafka) |
| **Data integrity** | Strict ordering, zero loss |

## 1.3 Responsibility Boundary

| Type | Scope |
|------|-------|
| **In-Scope** | Multi-cloud long-connection management; sessionId/sequenceNumber-based ordering; reliable delivery to Kafka |
| **Out-of-Scope** | No audio stream processing; no intent recognition, sentiment analysis, or downstream business logic; downstream consumers subscribe to Kafka independently |

---

# Part II — Physical & Network Architecture

## 2.1 Multi-Cloud Backbone Topology

The system uses **internal multi-cloud backbone / dedicated links** (AWS Direct Connect + GCP Cloud Interconnect). All WebSocket handshakes and data flows stay within the bank's closed internal network; no public internet is involved.

**Network trace:**

1. **GCP egress**: GKE Service A initiates connection via **Cloud Interconnect**
2. **Dedicated link**: 10 Gbps dedicated link
3. **AWS ingress**: **Direct Connect Gateway**
4. **Internal gateway**: ALB on 443, internal SSL cert, SSL termination
5. **Target**: ALB routes to Fargate tasks on port 8080

## 2.2 VPC Isolation & Security Groups

| Layer | Component | Strategy |
|-------|-----------|----------|
| **Access** | Internal ALB | Allow only GKE cluster CIDR |
| **Compute** | ECS Fargate | Allow only ALB inbound |
| **State & Storage** | Redis, Kafka | Allow only Fargate SG on 6379, 9092 |

## 2.3 Defense in Depth & Encryption

| Layer | Measure |
|-------|---------|
| **Transport** | ALB TLS 1.2+, internal CA certificate |
| **Application** | WebSocket handshake validates `Authorization: Bearer <Token>` |
| **At rest** | MSK and Redis use KMS encryption |

---

# Part III — Core Application Logic

## 3.1 Technology Stack & Concurrency Model

| Component | Choice |
|-----------|--------|
| **Framework** | FastAPI (ASGI) + Uvicorn |
| **Async stack** | redis.asyncio, aiokafka |
| **Concurrency** | Single-thread asyncio, one Worker per vCPU |
| **WebSocket** | websockets, `max_size=1MB` |

**Rationale**: I/O-bound workload; asyncio avoids GIL and context-switch overhead; one process per vCPU for multi-core parallelism.

## 3.2 Connection Lifecycle & Keep-Alive

| Mechanism | Configuration |
|-----------|---------------|
| **Business signals** | `SESSION_ONGOING`, `SESSION_COMPLETE` |
| **Protocol keep-alive** | Ping/Pong every 20 s (ALB idle timeout 60 s) |

## 3.3 Sequence Guard (Optimistic Data Lock)

All session state is in Redis; no local session state. Sequence validation is atomic via Lua.

| Condition | Action |
|-----------|--------|
| `seq == expected` | Allow delivery |
| `seq < expected` | Idempotent; return ACK, do not write Kafka |
| `seq > expected` | Reject, return E1006 |

**Redis key**: `ts:sess:{sessionId}`

## 3.4 Two-Phase Commit (2PC)

| Phase | Action |
|-------|--------|
| **Prepare** | Lua pre-check (no INCR) |
| **Persistence** | Write to Kafka, `sessionId` as key, `acks=all` |
| **Commit** | On Kafka Ack, call Redis INCR |
| **Ack** | Send `TRANSCRIPT_ACK` |

If Kafka fails, no Commit; upstream retries with same seq; Redis state unchanged → lossless retry.

---

# Part IV — Storage & Protocol

## 4.1 Kafka

| Item | Configuration |
|------|---------------|
| **Topic** | `transcription.raw.stream` or `cc.transcript.realtime.v1` |
| **Partition key** | `sessionId` |
| **Partition count** | Fixed 50 or 100 (hash base must not change) |
| **Reliability** | `acks=all`, `enable_idempotence=True`, `max_in_flight=1` |
| **Compression** | `zstd` |
| **Retention** | 7 days |
| **MSK** | `min.insync.replicas=2` |

**Message structure (JSON/Protobuf):**

```json
{
  "header": {
    "sessionId": "UUID",
    "version": "1.0",
    "producerId": "Fargate-IP"
  },
  "payload": {
    "seq": 105,
    "text": "Hello, how can I help you?",
    "timestamp": 1710668400
  }
}
```

## 4.2 Redis

| Item | Configuration |
|------|---------------|
| **Key** | `ts:sess:{sessionId}` |
| **Value** | Integer (expected sequence) or Hash (`expected_seq`, `start_time`, `last_active`) |
| **TTL** | 900 s rolling renewal |
| **Memory** | ~100 B/session; 1,000 sessions < 1 MB |

---

# Part V — High Availability & Resilience

## 5.1 Rolling TTL

Each valid push extends the session TTL by 15 minutes. Idle sessions are reclaimed after 15 minutes without traffic.

## 5.2 Backpressure & Circuit Breaker

| Mechanism | Configuration |
|-----------|---------------|
| **Kafka timeout** | 2 s; on timeout return E1009 |
| **Circuit breaker** | Open when failure rate > 15% in 10 s; return E1009 immediately |
| **Half-open** | After 30 s, allow 10% traffic |
| **Severe overload** | WebSocket Close Code `1013` (Try Again Later) |

## 5.3 Graceful Shutdown

| Step | Action |
|------|--------|
| 1 | On SIGTERM, stop accepting new connections |
| 2 | Send Close frames (1001/1012) to existing connections |
| 3 | Flush Kafka producer buffer |
| 4 | Exit after in-flight messages are persisted |

---

# Part VI — Infrastructure & Capacity

## 6.1 ECS Fargate

| Parameter | Value |
|-----------|-------|
| CPU | 1 vCPU |
| Memory | 2 GB |
| Ephemeral storage | 20 GB |
| nofile | 65535 |
| Max connections per task | 500 |

## 6.2 Auto-Scaling

| Event | Condition | Action |
|-------|-----------|--------|
| Scale-out | `ActiveConnectionCount > 400` for 2 min | +1 instance |
| Scale-in | `ActiveConnectionCount < 100` for 10 min | -1 instance |

**Principle**: Use connection count, not CPU/memory.

## 6.3 ALB

| Parameter | Value |
|-----------|-------|
| Protocol | WSS (TLS 1.2+) |
| Idle timeout | 3600 s |
| Sticky sessions | Disabled |

## 6.4 MSK

| Parameter | Value |
|-----------|-------|
| min.insync.replicas | 2 |
| compression.type | zstd |

---

# Part VII — API Contract & Error Codes

## 7.1 WebSocket Endpoint

| Item | Value |
|------|-------|
| **Endpoint** | `/ws/v1/realtime-transcriptions` |
| **Method** | WebSocket Upgrade |
| **Payload** | `application/json` (UTF-8) |
| **Transport** | WSS (TLS required) |

**Query parameter**: `sessionId` (required)

## 7.2 Event Types

| Direction | eventType | Description |
|-----------|-----------|-------------|
| Client → Server | `SESSION_ONGOING` | Normal transcript event |
| Client → Server | `SESSION_COMPLETE` | End-of-life event |
| Server → Client | `TRANSCRIPT_ACK` | Per-message ACK |
| Server → Client | `ERROR` | Validation or processing error |

## 7.3 Request Schema (Client → Server)

```json
{
  "metaData": {
    "sessionId": "UUID",
    "agentId": "string",
    "staffId": "string",
    "customerId": "string",
    "callStartTimeStamp": "ISO-8601",
    "callEndTimeStamp": "ISO-8601 | null",
    "eventType": "SESSION_ONGOING | SESSION_COMPLETE"
  },
  "payload": {
    "sequenceNumber": 0,
    "speaker": "Agent | Customer",
    "transcript": "string",
    "engineProvider": "string",
    "dialect": "string",
    "isFinal": true,
    "createdAtTimeStamp": "ISO-8601"
  }
}
```

## 7.4 Response Schema (Success)

```json
{
  "metaData": {
    "sessionId": "UUID",
    "eventType": "TRANSCRIPT_ACK"
  },
  "payload": {
    "sequenceNumber": 0,
    "createdAtTimeStamp": "ISO-8601"
  }
}
```

## 7.5 Application Error Codes

| Code | HTTP | WS Close | Scenario |
|------|------|----------|----------|
| E1006 | 400 | 1008 | Invalid sequence / non-monotonic |
| E1008 | 500 | 1011 | Internal error (Redis, app) |
| E1009 | 503/429 | 1013 | Downstream unavailable / throttling |
| E1013 | 504 | 1013 | Upstream/downstream timeout |

**Additional codes** (validation, auth, etc.): See Appendix.

---

# Part VIII — Observability & Operations

## 8.1 Structured Logging

- Format: JSON
- Required fields: `sessionId`, `sequenceNumber`, `eventType`, `latency_ms`
- Example: `[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms`

## 8.2 Golden Signals

| Metric | Alert |
|--------|-------|
| Connection success rate | — |
| Active connections | — |
| Kafka P99 latency | > 80 ms |
| E1006 / E1009 rate | — |
| 5xx error rate | > 1% |
| Kafka disk usage | > 80% |

## 8.3 FMEA

| Component | Failure | Detection | Mitigation |
|-----------|---------|------------|------------|
| GCP engine | Stops sending | Ping/Pong timeout | Redis TTL reclaim after 15 min |
| GCP link | Connection drop | — | Client exponential backoff; Redis state for resume |
| Fargate | OOM/crash | ALB health check | ALB removes node; new instance started |
| Redis | Failover | ReadOnlyError | Catch exception, return E1008 |
| Kafka | Blocked/overload | KafkaTimeoutException | Circuit breaker, E1009 backpressure |

---

# Part IX — Security & Compliance

## 9.1 IAM Least Privilege

**Fargate role**:

- `kms:Decrypt` (certificate decryption)
- `kafka:Publish` (specific topic only)
- `elasticache:Connect` (Redis port only)

## 9.2 Audit Logging

Logs must include **Trace ID** for cross-cloud troubleshooting.

---

# Part X — Performance Baseline & Extensibility

## 10.1 Performance Targets

| Metric | Target |
|--------|--------|
| Max concurrent connections | 2,000 |
| Avg processing time | < 15 ms |
| P99 delivery latency | < 80 ms |
| End-to-end latency | < 50 ms |
| Connections per pod | 500 (4 pods for HA) |

## 10.2 Evolution

| Direction | Description |
|-----------|-------------|
| **Downstream decoupling** | New consumers join as Kafka Consumer Groups; no changes to gateway |
| **Multi-cloud flexibility** | Any engine following the API contract can integrate |

---

# Part XI — FAQ & Technical Justification

## Q1: Why Python async instead of multi-threading?

| Aspect | Multi-threading | Asyncio |
|--------|-----------------|---------|
| Execution | Pseudo-parallel (GIL) | Cooperative multitasking |
| Context switch | Kernel, high cost | User-space, low cost |
| Concurrency limit | Hundreds of threads | C10K+ connections |
| Memory (1,000 conn) | ~8 GB (stacks) | Few MB (coroutines) |
| Data safety | Locks, deadlock risk | Naturally thread-safe |

**Multi-core**: One Worker per vCPU; multiple processes avoid GIL and use all cores.

## Q2: Can blocking in one coroutine slow all connections?

Yes. Mitigations:

1. **Light logic**: Only sequence check and forward; no heavy computation
2. **Async-only I/O**: Use aiokafka, redis.asyncio; never block event loop
3. **Multi-process**: Other Workers continue if one is busy

---

# Part XII — Implementation Reference

## 12.1 Core Facts

- **Stack**: Python 3.10+ / FastAPI / redis.asyncio / aiokafka
- **Concurrency**: Single-thread asyncio, one Worker per vCPU
- **State**: Redis Lua two-phase commit
- **Kafka**: sessionId as key, acks=all, enable_idempotence=True
- **Keep-alive**: 20 s Ping/Pong

## 12.2 Redis Lua Scripts

**Pre-check (Check only):**

```lua
-- KEYS[1]: session key, ARGV[1]: incoming_seq, ARGV[2]: ttl
local current = redis.call('GET', KEYS[1])
local incoming = tonumber(ARGV[1])

if not current then
    if incoming == 0 then
        redis.call('SETEX', KEYS[1], ARGV[2], 1)
        return 100  -- SUCCESS_INIT
    else
        return 400  -- ERROR_INVALID_START
    end
end

if incoming == tonumber(current) then
    return 200  -- PRE_CHECK_OK
elseif incoming < tonumber(current) then
    return 300  -- IDEMPOTENT_REPLAY
else
    return 500  -- ERROR_OUT_OF_ORDER
end
```

**Commit (after Kafka Ack):**

```lua
redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
return 1
```

## 12.3 Implementation Checklist

1. **WebSocket route**: `/ws/v1/realtime-transcriptions` (or `/ws/transcribe` per internal convention); handshake validates `Authorization: Bearer`
2. **Message loop**: validate_sequence → if IDEMPOTENT, return ACK without Kafka; else write Kafka (2 s timeout) → commit_sequence → return ACK
3. **Exceptions**: KafkaTimeout / circuit open → E1009; uncaught → E1008
4. **Graceful shutdown**: On SIGTERM, flush Kafka, then close connections
5. **Logging**: JSON with sessionId, seq
6. **Initialization**: Kafka Producer and Lua `script_load` in lifespan/startup only
7. **Client disconnect**: Detect and release Redis connections correctly

---

# Appendix A — API Contract Full Specification

## A.1 Request Field Contract

### metaData

| Field | Required | Type | Max | Format | Description |
|-------|----------|------|-----|--------|-------------|
| sessionId | Yes | string | 64 | Unique per call | Genesys call id |
| agentId | Yes | string | 32 | — | Agent staff ID |
| staffId | Yes | string | 32 | — | Staff ID |
| customerId | Yes | string | 64 | — | Customer ID |
| callStartTimeStamp | Yes | string | 32 | ISO-8601 UTC | Call start |
| callEndTimeStamp | Conditional | string | 32 | ISO-8601 UTC | Call end (SESSION_COMPLETE only) |
| eventType | Yes | string | 32 | SESSION_ONGOING \| SESSION_COMPLETE | Event type |

### payload

| Field | Required | Type | Max | Format | Description |
|-------|----------|------|-----|--------|-------------|
| sequenceNumber | Yes | integer | — | ≥ 0, monotonic | Sequence number |
| speaker | Yes | string | 16 | Agent \| Customer | Speaker role |
| transcript | Yes | string | 8000 | — | Transcript text |
| engineProvider | Yes | string | 64 | e.g. Fanolab | STT provider |
| dialect | No | string | 32 | BCP-47 | Language/dialect |
| isFinal | Yes | boolean | — | true | Final hypothesis |
| createdAtTimeStamp | Yes | string | 32 | ISO-8601 UTC | Client timestamp |

## A.2 Business Rules

1. **Sequence**: `sequenceNumber` must be strictly monotonic per `sessionId`
2. **SESSION_ONGOING**: `callEndTimeStamp` must be null
3. **SESSION_COMPLETE**: `callEndTimeStamp` must be provided
4. **Idempotency**: `(sessionId, sequenceNumber)` is idempotent; server may re-ACK

## A.3 Error Response Schema

```json
{
  "metaData": {
    "sessionId": "UUID",
    "eventType": "ERROR"
  },
  "error": {
    "code": "E1006",
    "message": "Invalid sequence number",
    "details": "optional details",
    "createdAtTimeStamp": "ISO-8601"
  }
}
```

## A.4 HTTP & WebSocket Status Mapping

| Scenario | HTTP | WS Close |
|----------|------|----------|
| Upgrade success | 101 | — |
| Bad request | 400 | 1007/1008 |
| Unauthorized | 401 | 1008 |
| Forbidden | 403 | 1008 |
| Payload too large | 413 | 1009 |
| Rate limit | 429 | 1013 |
| Internal error | 500 | 1011 |
| Unavailable | 503 | 1013 |
| Try again later | — | 1013 |

---

*— End of Document —*
