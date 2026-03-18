# Transcribe Service — 架构设计文档

> **文档类型**：整合架构设计  
> **范围**：实时语音转写网关（GCP ↔ AWS）  
> **版本**：1.0  
> **视角**：架构评审

---

## 文档索引


| 部分 | 章节             | 内容                                |
| ---- | ---------------- | ----------------------------------- |
| I    | 执行摘要         | 业务背景、系统边界、核心目标        |
| II   | 物理与网络架构   | 多云拓扑、VPC 隔离、零信任          |
| III  | 核心应用逻辑     | 技术栈、连接生命周期、序列守卫、2PC |
| IV   | 存储与协议       | Kafka、Redis、消息 Schema           |
| V    | 高可用与韧性     | 滚动 TTL、背压、熔断、优雅停机      |
| VI   | 基础设施与容量   | Fargate、ALB、弹性伸缩、MSK         |
| VII  | API 契约与错误码 | WebSocket 协议、请求/响应、错误映射 |
| VIII | 可观测性与运维   | 日志、监控、FMEA                    |
| IX   | 安全与合规       | IAM、加密、审计                     |
| X    | 性能基线与发展   | 目标、IAM、演进路线                 |
| XI   | FAQ 与技术辩护   | 异步 vs 多线程、阻塞风险            |
| XII  | 实现参考         | Lua 脚本、状态管理、开发要求        |
| 附录 | API 契约完整规格 | 完整请求/响应 Schema                |


---

# 第一部分 — 执行摘要与系统边界

## 1.1 业务背景

在呼叫中心智能化升级背景下，本行引入了第三方厂商 FanoLab 的 ASR（自动语音识别）引擎。该引擎**由本行代为托管并部署于本行 GCP 环境中**，负责将客服通话语音实时转化为文本。

**Transcribe Service（Service B）** 部署于本行 AWS 环境，作为连接 GCP 与 AWS 内部数据生态的**核心多云实时数据网关**。

## 1.2 业务目标


| 目标           | 指标                                     |
| -------------- | ---------------------------------------- |
| **并发容量**   | 早高峰 700～1,000 路并发通话（设计目标） |
| **端到端延迟** | < 50 ms（GCP 接收 → AWS Kafka）          |
| **数据完整性** | 严格保序、零丢失                         |


## 1.3 职责边界


| 类型                       | 范围                                                                       |
| -------------------------- | -------------------------------------------------------------------------- |
| **In-Scope（系统内）**     | 多云长连接管理；基于 sessionId/sequenceNumber 的保序校验；可靠投递至 Kafka |
| **Out-of-Scope（系统外）** | 不处理音频流；不包含意图识别、情感分析等业务逻辑；下游需自行订阅 Kafka     |


---

# 第二部分 — 物理与网络架构

## 2.1 多云骨干网拓扑

系统采用**内部多云骨干网/专线**（AWS Direct Connect + GCP Cloud Interconnect）。所有 WebSocket 握手与数据流均在本行封闭内网内完成，不经过公网。

**网络轨迹：**

1. **GCP 出口**：GKE Service A 通过 **Cloud Interconnect** 发起建连
2. **专线穿越**：10 Gbps 专用链路
3. **AWS 入口**：**Direct Connect Gateway**
4. **内部网关**：ALB 监听 443，内部 SSL 证书，SSL 终结
5. **目标转发**：ALB 将请求路由至 Fargate 任务 8080 端口

## 2.2 VPC 隔离与安全组


| 层级             | 组件         | 策略                         |
| ---------------- | ------------ | ---------------------------- |
| **接入层**       | Internal ALB | 仅放行 GKE 集群 CIDR         |
| **计算层**       | ECS Fargate  | 仅允许 ALB 入站              |
| **状态与存储层** | Redis、Kafka | 仅对 Fargate 开放 6379、9092 |


## 2.3 纵深防御与加密


| 层级       | 措施                                               |
| ---------- | -------------------------------------------------- |
| **传输层** | ALB TLS 1.2+，本行 CA 证书                         |
| **应用层** | WebSocket 握手校验 `Authorization: Bearer <Token>` |
| **落盘**   | MSK、Redis 启用 KMS 加密                           |


---

# 第三部分 — 核心应用逻辑

## 3.1 技术栈与并发模型


| 组件          | 选型                                |
| ------------- | ----------------------------------- |
| **框架**      | FastAPI (ASGI) + Uvicorn            |
| **异步生态**  | redis.asyncio、aiokafka             |
| **并发模型**  | 单线程 Asyncio，每 vCPU 一个 Worker |
| **WebSocket** | websockets，`max_size=1MB`          |


**选型理由**：I/O 密集型场景；Asyncio 规避 GIL 与上下文切换开销；每 vCPU 一进程实现多核并行。

## 3.2 连接生命周期与保活


| 机制         | 配置                                     |
| ------------ | ---------------------------------------- |
| **业务信号** | `SESSION_ONGOING`、`SESSION_COMPLETE`    |
| **协议保活** | 每 20 秒 Ping/Pong（ALB 空闲超时 60 秒） |


## 3.3 序列守卫（乐观数据锁）

所有会话状态下沉 Redis，无本地会话状态。通过 Lua 脚本实现原子序列校验。


| 条件              | 行为                         |
| ----------------- | ---------------------------- |
| `seq == expected` | 放行投递                     |
| `seq < expected`  | 幂等；直接回 ACK，不写 Kafka |
| `seq > expected`  | 阻断，返回 E1006             |


**Redis Key**：`ts:sess:{sessionId}`

## 3.4 两阶段提交（2PC）


| 阶段                      | 操作                                       |
| ------------------------- | ------------------------------------------ |
| **Prepare（预检）**       | Lua 预检（不自增）                         |
| **Persistence（持久化）** | 写入 Kafka，`sessionId` 为 Key，`acks=all` |
| **Commit（提交）**        | Kafka Ack 后调用 Redis INCR                |
| **Ack**                   | 发送 `TRANSCRIPT_ACK`                      |


若 Kafka 失败则不 Commit；上游重试时 Redis 仍为原值，实现无损重试。

---

# 第四部分 — 存储与协议

## 4.1 Kafka


| 项目               | 配置                                                     |
| ------------------ | -------------------------------------------------------- |
| **Topic**          | `cc.transcript.realtime.v1`                              |
| **Partition Key**  | `sessionId`                                              |
| **Partition 数量** | 定死 50 或 100（Hash 基数不可变）                        |
| **可靠性**         | `acks=all`、`enable_idempotence=True`、`max_in_flight=1` |
| **压缩**           | `zstd`                                                   |
| **保留期**         | 7 天                                                     |
| **MSK**            | `min.insync.replicas=2`                                  |


**消息结构（JSON/Protobuf）：**

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


| 项目      | 配置                                                                     |
| --------- | ------------------------------------------------------------------------ |
| **Key**   | `ts:sess:{sessionId}`                                                    |
| **Value** | 整数（期望序列号）或 Hash（`expected_seq`、`start_time`、`last_active`） |
| **TTL**   | 900 秒滚动续租                                                           |
| **内存**  | 约 100 B/会话；1,000 会话 < 1 MB                                         |


---

# 第五部分 — 高可用与韧性

## 5.1 滚动 TTL

每次合法推送将对应 sessionId 的 TTL 顺延 15 分钟。无流量 15 分钟后自动回收僵尸会话。

## 5.2 背压与熔断


| 机制           | 配置                                           |
| -------------- | ---------------------------------------------- |
| **Kafka 超时** | 2 秒；超时即返回 E1009                         |
| **熔断器**     | 10 秒内失败率 > 15% 时熔断；秒回 E1009         |
| **半开**       | 30 秒后放行 10% 流量                           |
| **严重过载**   | WebSocket Close Code `1013`（Try Again Later） |


## 5.3 优雅停机


| 步骤 | 动作                                 |
| ---- | ------------------------------------ |
| 1    | 收到 SIGTERM 后停止接收新连接        |
| 2    | 向存量连接发送 Close 帧（1001/1012） |
| 3    | Flush Kafka 生产者缓冲区             |
| 4    | 待飞行中消息落盘后退出               |


---

# 第六部分 — 基础设施与容量

## 6.1 ECS Fargate


| 参数               | 值     |
| ------------------ | ------ |
| CPU                | 1 vCPU |
| Memory             | 2 GB   |
| Ephemeral Storage  | 20 GB  |
| nofile             | 65535  |
| 单 Task 最大连接数 | 500    |


## 6.2 弹性伸缩


| 事件 | 条件                                       | 动作    |
| ---- | ------------------------------------------ | ------- |
| 扩容 | `ActiveConnectionCount > 400` 持续 2 分钟  | +1 实例 |
| 缩容 | `ActiveConnectionCount < 100` 持续 10 分钟 | -1 实例 |


**原则**：基于连接数，不用 CPU/内存。

## 6.3 ALB


| 参数            | 值             |
| --------------- | -------------- |
| 协议            | WSS (TLS 1.2+) |
| Idle Timeout    | 3600 秒        |
| Sticky Sessions | 禁用           |


## 6.4 MSK


| 参数                | 值   |
| ------------------- | ---- |
| min.insync.replicas | 2    |
| compression.type    | zstd |


---

# 第七部分 — API 契约与错误码

## 7.1 WebSocket 端点


| 项目         | 值                               |
| ------------ | -------------------------------- |
| **Endpoint** | `/ws/v1/realtime-transcriptions` |
| **Method**   | WebSocket Upgrade                |
| **Payload**  | `application/json` (UTF-8)       |
| **传输协议** | WSS（TLS 必需）                  |


**Query 参数**：`sessionId`（必填）

## 7.2 事件类型


| 方向            | eventType          | 说明           |
| --------------- | ------------------ | -------------- |
| Client → Server | `SESSION_ONGOING`  | 正常转写事件   |
| Client → Server | `SESSION_COMPLETE` | 最终 EOL 事件  |
| Server → Client | `TRANSCRIPT_ACK`   | 逐消息确认     |
| Server → Client | `ERROR`            | 校验或处理错误 |


## 7.3 请求 Schema（Client → Server）

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

## 7.4 响应 Schema（成功）

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

## 7.5 应用错误码


| 错误码 | HTTP    | WS Close | 典型场景                |
| ------ | ------- | -------- | ----------------------- |
| E1006  | 400     | 1008     | 序列号非法/非单调递增   |
| E1008  | 500     | 1011     | 内部错误（Redis、应用） |
| E1009  | 503/429 | 1013     | 下游不可用/限流         |
| E1013  | 504     | 1013     | 上游/下游超时           |


**其他错误码**（校验、鉴权等）：见附录。

---

# 第八部分 — 可观测性与运维

## 8.1 结构化日志

- 格式：JSON
- 必含字段：`sessionId`、`sequenceNumber`、`eventType`、`latency_ms`
- 示例：`[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms`

## 8.2 黄金指标


| 指标               | 告警    |
| ------------------ | ------- |
| 建连成功率         | —       |
| 活跃连接数         | —       |
| Kafka P99 延迟     | > 80 ms |
| E1006 / E1009 频率 | —       |
| 5xx 错误率         | > 1%    |
| Kafka 磁盘使用率   | > 80%   |


## 8.3 FMEA（故障模式影响分析）


| 组件     | 故障      | 探测                  | 应对                               |
| -------- | --------- | --------------------- | ---------------------------------- |
| GCP 引擎 | 停止发包  | Ping/Pong 超时        | 15 分钟后 Redis TTL 回收           |
| GCP 专线 | 连接掉线  | —                     | 客户端指数退避重连；Redis 断点续传 |
| Fargate  | OOM/宕机  | ALB 健康检查          | ALB 摘除节点；自动拉起新实例       |
| Redis    | 主从切换  | ReadOnlyError         | 捕获异常，返回 E1008               |
| Kafka    | 阻塞/过载 | KafkaTimeoutException | 熔断，E1009 背压                   |


---

# 第九部分 — 安全与合规

## 9.1 IAM 最小权限

**Fargate role**：

- `kms:Decrypt`（证书解密）
- `kafka:Publish`（仅限特定 Topic）
- `elasticache:Connect`（仅限 Redis 端口）

## 9.2 审计日志

日志必须包含 **Trace ID**，用于跨云链路排障。

---

# 第十部分 — 性能基线与发展

## 10.1 性能目标


| 指标          | 目标                 |
| ------------- | -------------------- |
| 最大并发连接  | 2,000                |
| 平均处理耗时  | < 15 ms              |
| P99 投递延迟  | < 80 ms              |
| 端到端延迟    | < 50 ms              |
| 单 Pod 连接数 | 500（4 Pod 实现 HA） |


## 10.2 演进方向


| 方向         | 说明                                           |
| ------------ | ---------------------------------------------- |
| **下游解耦** | 新业务以 Kafka Consumer Group 接入；网关零侵入 |
| **多云灵活** | 遵循 API 契约即可接入其他云/引擎               |


---

# 第十一部分 — FAQ 与技术辩护

## Q1：为何选 Python 异步而非多线程？


| 维度               | 多线程                | Asyncio        |
| ------------------ | --------------------- | -------------- |
| 执行模式           | 伪并行（受 GIL 限制） | 协作式多任务   |
| 上下文切换         | 内核态，开销大        | 用户态，开销低 |
| 并发上限           | 数百线程              | C10K+ 连接     |
| 内存（1,000 连接） | 约 8 GB（栈）         | 数 MB（协程）  |
| 数据安全           | 需加锁，易死锁        | 天然线程安全   |


**多核利用**：每 vCPU 一个 Worker；多进程规避 GIL，实现多核并行。

## Q2：单线程内阻塞会拖慢所有连接吗？

会。防御措施：

1. **轻量逻辑**：仅做序列校验与转发，无重量级计算
2. **全异步 I/O**：使用 aiokafka、redis.asyncio；绝不阻塞事件循环
3. **多进程分担**：单进程繁忙时，其他 Worker 仍可正常响应

---

# 第十二部分 — 实现参考

## 12.1 核心事实

- **技术栈**：Python 3.10+ / FastAPI / redis.asyncio / aiokafka
- **并发**：单线程 Asyncio，每 vCPU 一 Worker
- **状态**：Redis Lua 两阶段提交
- **Kafka**：sessionId 为 Key，acks=all，enable_idempotence=True
- **保活**：20 秒 Ping/Pong

## 12.2 Redis Lua 脚本

**预检（仅校验，不自增）：**

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

**提交（Kafka Ack 后调用）：**

```lua
redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
return 1
```

## 12.3 实现清单

1. **WebSocket 路由**：`/ws/v1/realtime-transcriptions`（或按内部约定使用 `/ws/transcribe`）；握手校验 `Authorization: Bearer`
2. **消息循环**：validate_sequence → 若 IDEMPOTENT 则回 ACK 不写 Kafka；否则写 Kafka（2 秒超时）→ commit_sequence → 回 ACK
3. **异常处理**：KafkaTimeout / 熔断开启 → E1009；未捕获异常 → E1008
4. **优雅停机**：收到 SIGTERM 后 flush Kafka，再关闭连接
5. **日志**：JSON 格式，含 sessionId、seq
6. **初始化**：Kafka Producer、Lua `script_load` 仅在 lifespan/startup 执行一次
7. **客户端断开**：正确感知并释放 Redis 连接

---

# 附录 A — API 契约完整规格

## A.1 请求字段契约

### metaData


| 字段               | 必填 | 类型   | 最大长度 | 格式            | 说明                                |
| ------------------ | ---- | ------ | -------- | --------------- | ----------------------------------- |
| sessionId          | 是   | string | 64       | 每通电话唯一    | Genesys call id                     |
| agentId            | 是   | string | 32       | —               | 坐席 Staff ID                       |
| staffId            | 是   | string | 32       | —               | 员工 ID                             |
| customerId         | 是   | string | 64       | —               | 客户 ID                             |
| callStartTimeStamp | 是   | string | 32       | ISO-8601 UTC    | 通话开始时间                        |
| callEndTimeStamp   | 条件 | string | 32       | ISO-8601 UTC    | 通话结束时间（仅 SESSION_COMPLETE） |
| eventType          | 是   | string | 32       | SESSION_ONGOING | SESSION_COMPLETE                    | 事件类型 |


### payload


| 字段               | 必填 | 类型    | 最大长度 | 格式          | 说明               |
| ------------------ | ---- | ------- | -------- | ------------- | ------------------ |
| sequenceNumber     | 是   | integer | —        | ≥ 0，单调递增 | 序列号             |
| speaker            | 是   | string  | 16       | Agent         | Customer           | 说话人角色 |
| transcript         | 是   | string  | 8000     | —             | 转写文本           |
| engineProvider     | 是   | string  | 64       | 如 Fanolab    | STT 引擎提供商     |
| dialect            | 否   | string  | 32       | BCP-47        | 语言/方言          |
| isFinal            | 是   | boolean | —        | true          | 是否为最终假设     |
| createdAtTimeStamp | 是   | string  | 32       | ISO-8601 UTC  | 客户端转写创建时间 |


## A.2 业务规则

1. **序列号**：同一 `sessionId` 下 `sequenceNumber` 必须严格单调递增
2. **SESSION_ONGOING**：`callEndTimeStamp` 必须为 null
3. **SESSION_COMPLETE**：`callEndTimeStamp` 必须提供
4. **幂等性**：`(sessionId, sequenceNumber)` 视为幂等；服务端可再次返回 ACK

## A.3 错误响应 Schema

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

## A.4 HTTP 与 WebSocket 状态映射


| 场景       | HTTP | WS Close  |
| ---------- | ---- | --------- |
| 升级成功   | 101  | —         |
| 错误请求   | 400  | 1007/1008 |
| 未授权     | 401  | 1008      |
| 禁止访问   | 403  | 1008      |
| 负载过大   | 413  | 1009      |
| 限流       | 429  | 1013      |
| 内部错误   | 500  | 1011      |
| 服务不可用 | 503  | 1013      |
| 稍后重试   | —    | 1013      |


---

*— 文档结束 —*