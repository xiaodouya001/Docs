# Transcribe Service API Contract 契约文档

> 基于 Confluence API Contract 整理，涵盖 WebSocket 端点、消息结构、请求/响应契约、业务规则、状态码及错误码定义。

---

## 文档结构

| 章节 | 内容 |
|------|------|
| 一、协议概览 | WebSocket 端点、Header、事件类型与流转 |
| 二、请求契约 | Client → Server 消息结构、字段定义、业务规则 |
| 三、响应契约 | Server → Client 成功/错误响应结构 |
| 四、状态码与错误码 | HTTP 握手码、WebSocket 关闭码、应用错误码映射 |
| 五、错误处理策略 | 可恢复/不可恢复错误的处理方式 |
| 六、完整示例 | 请求与响应对照示例 |

---

# 一、协议概览 (Protocol Overview)

## 1.1 WebSocket 端点 (Endpoint)

| 项目 | 说明 |
|------|------|
| **Endpoint** | `/ws/v1/realtime-transcriptions` |
| **Method** | WebSocket Upgrade |
| **Payload 格式** | `application/json` (UTF-8) |
| **传输协议** | `wss` (TLS/mTLS 必需) |

**URL 参数：**

| 参数 | 必填 | 类型 | 说明 | 示例 |
|------|------|------|------|------|
| `sessionId` | 是 | string | 使用 Genesys Call Id，唯一标识本次转写会话 | `/ws/v1/realtime-transcriptions?sessionId=39449992-32f3-4581-a8a1-99d4109f37d4` |

## 1.2 Header (Placeholder)

> 完整 Header 列表待定，以下为预留占位。

| Header | 必填 | 类型 | 最大长度 | 说明 |
|--------|------|------|----------|------|
| `Authorization` | 是（推荐） | string | 4096 | Bearer token 或其他鉴权凭证 |

## 1.3 事件类型与流转 (Event Types and Flow)

**Client → Server：**

| eventType | 说明 |
|-----------|------|
| `SESSION_ONGOING` | 正常转写事件 |
| `SESSION_COMPLETE` | 最终 EOL 事件（会话结束） |

**Server → Client：**

| eventType | 说明 |
|-----------|------|
| `TRANSCRIPT_ACK` | 逐消息确认 |
| `ERROR` | 校验或处理错误 |

---

# 二、请求契约 (Request Body)

*Client → Server 消息格式*

## 2.1 消息结构 (JSON Schema)

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "agentId": "3210001",
    "staffId": "45163407",
    "customerId": "12345678",
    "callStartTimeStamp": "2025-03-21T10:30:02.327Z",
    "callEndTimeStamp": null,
    "eventType": "SESSION_ONGOING"
  },
  "payload": {
    "sequenceNumber": 0,
    "speaker": "Agent",
    "transcript": "thank you",
    "engineProvider": "Fanolab",
    "dialect": "yue-x-auto",
    "isFinal": true,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

## 2.2 字段定义 (Field Contract)

### metaData

| 字段 | 必填 | 类型 | 最大长度 | 取值/格式 | 说明 |
|------|------|------|----------|-----------|------|
| `sessionId` | 是 | string | 64 | 每通电话唯一 ID | 会话标识 (Genesys call id from fano assist) |
| `agentId` | 是 | string | 32 | Agent's Staff ID | 坐席在 Genesys 中的标识 |
| `staffId` | 是 | string | 32 | Genesys 下发的 staff ID | 员工标识 |
| `customerId` | 是 | string | 64 | 客户号码 | 客户标识 |
| `callStartTimeStamp` | 是 | string | 32 | ISO-8601 UTC | 通话开始时间 |
| `callEndTimeStamp` | 条件 | string | 32 | ISO-8601 UTC，仅通话结束时提供 | 通话结束时间 |
| `eventType` | 是 | string | 32 | `SESSION_ONGOING` \| `SESSION_COMPLETE` | 事件类型 |

### payload

| 字段 | 必填 | 类型 | 最大长度 | 取值/格式 | 说明 |
|------|------|------|----------|-----------|------|
| `sequenceNumber` | 是 | integer | — | ≥ 0，同一 sessionId 内单调递增 | 转写序列号 |
| `speaker` | 是 | string | 16 | `Agent` \| `Customer` | 说话人角色 |
| `transcript` | 是 | string | 8000 | 转写文本 | 转写内容 |
| `engineProvider` | 是 | string | 64 | 如 `Fanolab` | STT 引擎提供商 |
| `dialect` | 否 | string | 32 | BCP-47，如 `yue-x-auto` | 支持的语言/方言 |
| `isFinal` | 是 | boolean | — | `true` | 是否为最终假设，必须为 true |
| `createdAtTimeStamp` | 是 | string | 32 | ISO-8601 UTC | 客户端转写创建时间 |

## 2.3 业务规则 (Business Rules)

1. **序列号**：同一 `sessionId` 下 `sequenceNumber` 必须严格单调递增。
2. **SESSION_ONGOING**：`callEndTimeStamp` 必须为 `null`。
3. **SESSION_COMPLETE**：`callEndTimeStamp` 必须提供。
4. **结束事件**：建议以 `SESSION_COMPLETE` 作为最终 EOL 事件。
5. **幂等性**：
   - `(sessionId, sequenceNumber)` 组合应视为幂等。
   - 服务端对相同组合会再次返回 ACK。

---

# 三、响应契约 (Response Body)

*Server → Client 消息格式*

## 3.1 成功响应 (TRANSCRIPT_ACK)

**结构示例：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "TRANSCRIPT_ACK"
  },
  "payload": {
    "sequenceNumber": 0,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

**字段说明：**

| 字段 | 必填 | 类型 | 最大长度 | 取值/格式 | 说明 |
|------|------|------|----------|-----------|------|
| `metaData.sessionId` | 是 | string | 64 | 会话 ID | 回显请求中的 sessionId |
| `metaData.eventType` | 是 | string | 32 | `TRANSCRIPT_ACK` | 事件类型 |
| `payload.sequenceNumber` | 是 | integer | — | ≥ 0 | 回显请求中的 sequenceNumber |
| `payload.createdAtTimeStamp` | 是 | string | 32 | ISO-8601 UTC | 服务端 ACK 时间戳 |

> **说明**：部分示例中 `payload` 写作 `message`，契约以 `payload` 为准，具体以 Confluence 最新定义为准。

## 3.2 错误响应 (ERROR)

**结构示例：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "ERROR"
  },
  "error": {
    "code": "E1002",
    "message": "Invalid eventType",
    "details": "metaData.eventType must be SESSION_ONGOING or SESSION_COMPLETE",
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

**字段说明：**

| 字段 | 必填 | 类型 | 最大长度 | 取值/格式 | 说明 |
|------|------|------|----------|-----------|------|
| `metaData.sessionId` | 是 | string | 64 | 会话 ID | 会话标识 |
| `metaData.eventType` | 是 | string | 32 | `ERROR` | 事件类型 |
| `error.code` | 是 | string | 16 | 见「四、状态码与错误码」章节 | 应用错误码 |
| `error.message` | 是 | string | 256 | 任意 | 简短错误描述 |
| `error.details` | 否 | string | 2048 | 任意 | 校验/处理详情 |
| `error.createdAtTimeStamp` | 是 | string | 32 | ISO-8601 UTC | 服务端时间戳 |

---

# 四、状态码与错误码

## 4.1 HTTP 握手阶段 (Upgrade)

| 场景 | 状态码 | 含义 |
|------|--------|------|
| WebSocket 升级成功 | 101 | Switching Protocols |
| 无效请求/参数/Header | 400 | Bad Request |
| 未授权 | 401 | Invalid/expired credential |
| 禁止访问 | 403 | Authenticated but not allowed |
| 负载过大 | 413 | Request Entity Too Large |
| 限流 | 429 | Too Many Requests |
| 握手内部错误 | 500 | Internal Server Error |
| 服务不可用 | 503 | Temporary unavailable |

## 4.2 WebSocket 关闭码 (Close Codes)

| 场景 | Close Code | 含义 |
|------|------------|------|
| 正常关闭 | 1000 | Normal closure |
| 服务端关闭/离开 | 1001 | Going away |
| 协议违规 | 1002 | Malformed frame / flow violation |
| 不支持的数据类型 | 1003 | 要求 JSON 时收到非文本/非 JSON |
| 负载格式无效 | 1007 | JSON 解析/类型/格式错误 |
| 策略违规 | 1008 | 业务规则、鉴权或策略违规 |
| 消息过大 | 1009 | Payload 超限 |
| 服务端内部错误 | 1011 | Server-side processing exception |
| 临时过载 | 1013 | Try again later |

> 对于 `1000` 和 `1001`，若不发送错误帧，可省略 `eventType`。

## 4.3 应用错误码映射表

| 错误码 | eventType | HTTP (握手) | WS Close | 典型场景 |
|--------|-----------|-------------|----------|----------|
| E1001 | ERROR | 400 | 1007 | JSON 解析失败 / 格式无效 |
| E1002 | ERROR | 400 | 1008 | 枚举值无效 / 字段校验错误 |
| E1003 | ERROR | 400 | 1008 | 缺少必填字段 |
| E1004 | ERROR | 400 | 1008 | 字段类型不匹配 |
| E1005 | ERROR | 400 | 1008 | 时间戳格式无效 |
| E1006 | ERROR | 400 | 1008 | 序列号无效或非单调递增 |
| E1007 | ERROR | 413 | 1009 | Payload 超限 |
| E1008 | ERROR | 500 | 1011 | 内部服务错误 |
| E1009 | ERROR | 503/429 | 1013 | 下游不可用 / 限流 |
| E1010 | ERROR | 403/400 | 1008 | 业务/策略冲突、权限问题 |
| E1011 | ERROR | 401 | 1008 | 鉴权/授权失败 |
| E1012 | ERROR | 404 | 1008 | 资源/会话不存在 |
| E1013 | ERROR | 504 | 1013 | 上游/下游超时 |
| E2001 | ERROR | 400 | 1002 | 不支持的协议/子协议/版本 |
| E2002 | ERROR | 400 | 1008 | 不支持的 payload schema 版本 |
| E2003 | ERROR | 426 | 1008 | 版本已弃用，需升级 |

---

# 五、错误处理策略

## 5.1 处理原则

| 错误类型 | 处理方式 |
|----------|----------|
| **可恢复的逐消息错误** | 优先发送 `ERROR` 帧，**不关闭**连接 |
| **不可恢复的协议错误** | 先发送 `ERROR` 帧，再按对应 WebSocket Close Code 关闭连接 |

## 5.2 关闭前错误帧示例

在关闭连接前，可先发送如下错误帧：

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "ERROR"
  },
  "error": {
    "code": "E1010",
    "message": "Session state conflict",
    "details": "callEndTimeStamp must be provided when eventType=SESSION_COMPLETE",
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

---

# 六、完整示例

## 6.1 进行中会话 (SESSION_ONGOING)

**请求：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "agentId": "3210001",
    "staffId": "45163407",
    "customerId": "12345678",
    "callStartTimeStamp": "2025-03-21T10:30:02.327Z",
    "callEndTimeStamp": null,
    "eventType": "SESSION_ONGOING"
  },
  "payload": {
    "sequenceNumber": 0,
    "speaker": "Customer",
    "transcript": "Hello",
    "engineProvider": "Fanolab",
    "dialect": "yue-x-auto",
    "isFinal": true,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

**响应：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "TRANSCRIPT_ACK"
  },
  "payload": {
    "sequenceNumber": 0,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

## 6.2 结束会话 (SESSION_COMPLETE)

**请求：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "agentId": "3210001",
    "staffId": "45163407",
    "customerId": "12345678",
    "callStartTimeStamp": "2025-03-21T10:30:02.327Z",
    "callEndTimeStamp": "2026-02-05T08:49:01.048Z",
    "eventType": "SESSION_COMPLETE"
  },
  "payload": {
    "sequenceNumber": 42,
    "speaker": "Agent",
    "transcript": "Good bye",
    "engineProvider": "Fanolab",
    "dialect": "yue-x-auto",
    "isFinal": true,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

**响应：**

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "TRANSCRIPT_ACK"
  },
  "payload": {
    "sequenceNumber": 42,
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

## 6.3 错误响应 (E1010 示例)

```json
{
  "metaData": {
    "sessionId": "39449992-32f3-4581-a8a1-99d4109f37d4",
    "eventType": "ERROR"
  },
  "error": {
    "code": "E1010",
    "message": "Session state conflict",
    "details": "callEndTimeStamp must be provided when eventType=SESSION_COMPLETE",
    "createdAtTimeStamp": "2025-03-21T10:32:20.000Z"
  }
}
```

---

*— 文档结束 —*
