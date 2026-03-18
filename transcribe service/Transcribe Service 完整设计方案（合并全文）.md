# Transcribe Service 完整设计方案（合并全文）

> 本文档将架构白皮书、详细设计说明书、全量技术实施方案及开发指令中的全部内容，按统一大纲合并为一份完整的端到端技术文档。

---

# 第一部分：架构白皮书 (Architecture Whitepaper)

*侧重于业务边界、合规性与宏观设计原则*

---

## 1. 执行摘要与系统边界 (Executive Summary & System Context)

### 1.1 业务背景与系统定位

在呼叫中心（Call Center）智能化升级的背景下，本行引入了第三方厂商（FanoLab）的实时语音识别（ASR）引擎。该引擎作为第三方应用，**由本行代为托管并部署于本行内部的 GCP (Google Cloud Platform) 环境中**，负责将实时的客服通话语音转化为文本（Transcript）。 **Transcribe Service (本系统，简称 Service B)** 部署于本行 AWS 环境，作为连接本行 GCP 环境与本行 AWS 内部数据生态的**核心多云实时数据网关**。

### 1.2 业务目标

支撑本行 Call Center 早高峰 **1000路并发通话** 的实时转写流，保证端到端延迟（从 GCP 接收到进入 AWS Kafka）在 **50ms** 以内，且数据**绝对保序、零丢失**。

### 1.3 核心目标与职责边界 (In-Scope / Out-of-Scope)

为确保系统的高可用性、可扩展性以及未来下游业务的灵活接入，本系统遵循严格的"单一职责原则"。

- **In-Scope (系统内职责)**：
  - **多云长连接管理**：作为 WebSocket 服务端，承载来自本行 GCP 环境的海量并发长连接（设计早高峰容量：700~1000 并发通话）。
  - **事务级状态与保序校验**：基于业务会话（`sessionId`）和序列号（`sequenceNumber`），在分布式并发场景下拦截重复包、驳回乱序包，确保单通电话文本流的绝对时序正确性。
  - **可靠投递与解耦**：将清洗及保序后的文本数据，实时、可靠地投递至本行内部的 AWS MSK (Kafka) 消息总线。
- **Out-of-Scope (系统外职责 - 明确不包含)**：
  - **音频流处理**：本系统仅接收及处理翻译完成的**纯文本数据 (Text)**，不接触任何原始语音流。
  - **业务语义处理**：本系统属于纯粹的数据基础设施（Data Funnel），**绝对不包含**任何意图识别（Intent Recognition）、情感分析或下游业务消费逻辑。下游 AI 或业务团队需自行作为 Consumer 订阅 Kafka 获取数据。

---

## 2. 总体物理与网络架构 (Physical & Network Architecture)

虽然通信双方（Service A 与 Service B）均部署于本行控制的云环境内，但作为处理核心客服通话数据的多云链路，本系统的网络架构设计依然以"零信任 (Zero Trust)"和"网段级隔离"为最高指导原则，确保数据在传输与存储阶段的绝对安全合规。

### 2.1 内部多云骨干网拓扑设计

本系统彻底摒弃公网互联方案。本行的 GCP 租户与 AWS 租户之间通过**内部多云骨干网/专线**（如 AWS Direct Connect 结合 GCP Cloud Interconnect）进行物理级连通。

- 所有的 WebSocket 握手与数据推流，均在本行封闭的内部路由网络内完成，不经过任何公共互联网，彻底杜绝公网层面的中间人攻击与数据嗅探风险。

**跨云网络路由轨迹 (Network Trace)**：

1. **GCP 出口**：GCP GKE 内部 Service A 发起建连，通过 **Cloud Interconnect** 路由。
2. **专线穿越**：数据包通过专用 10Gbps 链路。
3. **AWS 入口**：进入 **AWS Direct Connect Gateway**。
4. **内部网关**：ALB 监听 443 端口，挂载内网 SSL 证书，执行 SSL Termination。
5. **目标转发**：ALB 根据权重将请求路由至 Fargate 任务的 8080 端口。

### 2.2 AWS 内部 VPC 隔离与安全组规划

系统内部组件在 AWS VPC 中采用严格的公私子网分离部署策略：

- **接入层 (Access Layer)**：采用 **Internal ALB (内部应用负载均衡器)**。ALB 部署于内部隔离子网，其安全组 (Security Group) 实行严格的白名单策略，**仅放行本行 GCP 环境中 GKE 集群所在 CIDR 网段**的入站流量。
- **计算层 (Compute Layer)**：Service B 采用 **ECS Fargate** 部署于私有子网 (Private Subnet)。安全组仅允许来自 Internal ALB 的入口流量。
- **状态与存储层 (State & Storage Layer)**：**AWS ElastiCache (Redis)** 与 **AWS MSK (Kafka)** 同样部署于深度私有子网。其安全组仅对计算层 (Fargate) 的业务安全组开放对应的读写端口（如 6379, 9092），拒绝任何其他越权访问。

### 2.3 纵深防御与加解密策略 (零信任安全加固)

- **传输层加密 (Data in Transit)**：虽然处于本行内网，Internal ALB 仍强制配置 TLS 1.2+ 加密策略。使用本行内部 CA 签发的 SSL 证书终结 WSS 协议，保障内网多云传输的绝对机密性。
- **应用层鉴权 (Application Auth)**：Service B 在 WebSocket 握手阶段，强制校验 HTTP Header 中的 `Authorization: Bearer <Token>`，防止内网其他未经授权的微服务非法建连。
- **落盘静态加密 (Data at Rest)**：AWS MSK (Kafka) 存储卷与 ElastiCache (Redis) 节点均强制开启基于 AWS KMS (Key Management Service) 的落盘加密功能，防止底层物理介质泄露导致的数据安全事件。

---

# 第二部分：详细设计说明书 (Detailed Design Specification)

*侧重于具体逻辑实现、并发控制与存储策略*

---

## 1. 核心应用逻辑与流式处理 (Core Application Logic)

作为承载高并发长连接与实时文本流转的核心枢纽，Transcribe Service 在计算层彻底抛弃了传统的同步阻塞模型，采用"纯异步 I/O + 状态全外包"的设计理念，以极低的资源损耗实现高吞吐与绝对的数据一致性。

### 1.1 纯异步技术栈选型与高并发处理

面对早高峰 700~1000 级别的实时语音转写并发推流，Transcribe Service 采用纯异步的 Python 技术栈构建：

- **技术栈**：Python 3.10+ / FastAPI / redis.asyncio / aiokafka。
- **核心框架**：基于 ASGI 标准的异步 Web 框架（FastAPI）配合 Uvicorn 服务器。
- **异步生态**：全面引入 `redis.asyncio` 与 `aiokafka` 等纯异步客户端。
- **并发模型**：单线程异步协程（Asyncio）。单 vCPU 对应单进程 Worker，避免多线程竞争产生的 GIL（全局解释器锁）开销。
- **连接控制**：使用 `websockets` 库，并配置 `max_size=1MB` 以防止大包攻击。
- **架构收益**：利用单线程事件循环（Event Loop）机制，系统在处理海量 WebSocket 连接的 I/O 等待时不会阻塞底层计算线程，避免了传统多线程模型带来的上下文切换开销与内存暴涨问题，天生契合容器化环境下的水平扩容。

### 1.2 连接生命周期与协议级保活 (Keep-Alive)

为适应呼叫中心业务中常见的"通话静默期"（如客服查阅资料时的长达数分钟的无语音输入），本系统在应用与协议层实施了双重生命周期管理：

- **业务生命周期闭环**：严格监听上游 ASR 引擎发送的 `SESSION_ONGOING` 与 `SESSION_COMPLETE` 信号，完成会话状态的建立与精准销毁。
- **底层协议保活防绞杀**：针对 AWS ALB 默认 60 秒空闲连接超时（Idle Timeout）的物理限制，Transcribe Service 在 WebSocket 协议层强制开启 **Ping/Pong 控制帧（频率设定为 20 秒/次）**。此机制持续刷新底层网络设备的状态表，确保静默期内的合法业务连接绝对不会被网络基建静默阻断。

### 1.3 无状态架构与防并发穿透设计——乐观数据锁 (Sequence Guard)

由于 ECS Fargate 节点会根据流量动态扩缩容，Transcribe Service 自身内存中**不持有任何通话连接状态**，所有状态管理下沉至 AWS ElastiCache (Redis)。 针对网络极短瞬断导致上游触发重连并发的"脑裂"场景，系统放弃了存在极大业务假死风险的"物理连接排他锁"，全面拥抱基于序列号的**乐观数据锁 (Sequence Lock)**：

- **原子校验引擎**：通过预先注入 Redis 的 Lua 脚本，将"读取期望序列号 -> 比对 -> 更新状态"打包为绝对的原子操作。
- **并发路由规则**：
  - **匹配期望值**：放行数据进入投递队列。
  - **小于期望值 (重复包/旧连接残影)**：触发幂等逻辑，拦截投递，直接向上游回复 `TRANSCRIPT_ACK`。
  - **大于期望值 (乱序跳号)**：硬性阻断，向上游抛出 `E1006` (Invalid sequence number) 倒逼其重传。
- **架构收益**：彻底解耦物理连接与业务状态。无论上游请求漂移至哪个 Fargate 节点，只要携带的 `sequenceNumber` 合法，即可实现零延迟的断点续传。

#### 1.3.1 Redis Lua 原子校验脚本详情

**脚本 A：预检逻辑 (Check Only) — 只读不写，极速执行**

```lua
-- KEYS[1]: sessionId_key
-- ARGV[1]: incoming_seq
-- ARGV[2]: expire_seconds (900s)

local current_expected = redis.call('GET', KEYS[1])

-- 1. 首次建连初始化
if not current_expected then
    if tonumber(ARGV[1]) == 0 then
        redis.call('SETEX', KEYS[1], ARGV[2], 1)
        return 100 -- SUCCESS_INIT
    else
        return 400 -- ERROR_INVALID_START (首包序列号必须为0)
    end
end

-- 2. 正常时序判定
if tonumber(ARGV[1]) == tonumber(current_expected) then
    -- 仅校验，不在此处自增（等待 Kafka Ack 后再通过第二个脚本自增）
    return 200 -- PRE_CHECK_OK

-- 3. 幂等判定（旧消息）
elseif tonumber(ARGV[1]) < tonumber(current_expected) then
    return 300 -- IDEMPOTENT_REPLAY (已经处理过，直接回ACK)

-- 4. 异常跳号（乱序）
else
    return 500 -- ERROR_OUT_OF_ORDER
end
```

**脚本 B：推进逻辑 (Commit & TTL) — 仅在 Kafka 成功后调用**

```lua
local key = KEYS[1]
local ttl = tonumber(ARGV[1])
redis.call('INCR', key)
redis.call('EXPIRE', key, ttl)
return 1
```

### 1.4 严格保序与两阶段提交机制 (Strict Ordering & 2PC)

系统向内部 AWS MSK (Kafka) 投递转写文本时，面临"不可靠网络"与"绝对不能丢数据"的核心矛盾。Transcribe Service 实施了轻量级的两阶段提交（Two-Phase Commit）雏形。系统不使用分布式事务，而是通过"状态滞后推进"实现一致性：

#### 1.4.1 正常流转时序

1. **GCP Side**: 发起 WSS 连接 -> 校验 Token 成功。
2. **GCP Side**: 推送 `SESSION_ONGOING` (seq=N)。
3. **Transcribe Service**: 拦截消息 -> 调用 Redis Lua 脚本。
4. **Redis**: 校验 `seq == expected` -> 返回 `PRE_CHECK_OK` (不自增)。
5. **Transcribe Service**: 异步调用 Kafka Producer 发送数据。
6. **AWS MSK**: 返回 `RecordMetadata` (Ack)。
7. **Transcribe Service**: 再次调用 Redis 脚本 -> 期望值自增 1 (Commit)。
8. **Transcribe Service**: 回复 `TRANSCRIPT_ACK` 给 GCP。

#### 1.4.2 2PC 阶段分解

| 阶段 | 操作 | 说明 |
|------|------|------|
| **Prepare (预检)** | Service A 发送 `seq=5`。Transcribe Service 调用 Lua 预检。 | 此时不在 Redis 中自增期望值。 |
| **Persistence (执行)** | 携带短超时配置，异步写入 MSK。`acks=all`。 | 强制将 `sessionId` 设定为 Kafka 的 Message Key，Hash 路由确保同一通电话的所有文本严格串行落入同一个 Partition。 |
| **Commit (提交)** | 收到 Kafka Ack。调用 Redis `INCR` 脚本将期望值推至 6。 | 仅当 Kafka 明确返回写入成功 Ack 后才推进状态。 |
| **Ack (回复)** | 回复 `TRANSCRIPT_ACK`。 | 向上游返回最终确认。 |

- **异常处理**：若 Kafka 写入失败，不执行 Commit 步骤。上游超时后重发 `seq=5`，Redis 此时存的仍是 5，预检依然通过，实现无损重试。
- **架构收益**：若 Kafka 写入失败，Redis 状态不会推进。上游重试时可持原序列号无缝接续，彻底杜绝了服务端宕机或底层基建抖动导致的数据黑洞。

---

## 2. 存储与通信协议细节 (Storage & Protocol)

### 2.1 Kafka 生产端深度配置

#### 2.1.1 消息定义

- **Topic**: `transcription.raw.stream` (或 `cc.transcript.realtime.v1`，视环境配置)
- **Partition Key**: `sessionId` (String) — 以 `sessionId` 为 Partition Key 的 FIFO 机制确保严格保序。
- **Compression**: `zstd` (针对文本转写有极高的压缩比，节省 60% 带宽) 或 `lz4` (平衡 CPU 与带宽)。
- **Retention**: 7 天。

#### 2.1.2 消息结构 (Message Value - Protobuf/JSON)

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

#### 2.1.3 可靠性配置

| 参数 | 值 | 说明 |
|------|------|------|
| `acks` | `all` | 确保三副本落盘 |
| `enable_idempotence` | `True` | 开启幂等性生产 |
| `max_in_flight_requests_per_connection` | `1` | 防止乱序 |
| `min.insync.replicas` | `2` | 配合 `acks=all` |

### 2.2 Redis 键空间管理 (Keyspace)

#### 2.2.1 存储结构

- **Key 命名**: `ts:sess:{sessionId}` (或 `transcript:session:{sessionId}`)
- **Value Type**: `Hash` (或简单 String，视实现选择)
- **Fields** (Hash 模式):
  - `expected_seq`: (int) 下一个期望接收的序列号。
  - `start_time`: (timestamp) 首次建连时间。
  - `last_active`: (timestamp) 最后一次活跃时间，用于监控。
- **TTL 策略**: 900 秒（15 分钟滚动过期）。

#### 2.2.2 内存估算

每个 Session 占用约 100 字节。1000 并发下，Redis 内存占用不足 1MB，极度轻量。仅存储极轻量的 `sessionId` 与 `seq` 键值对，配合 15 分钟 Rolling TTL，内存开销极低。重点保障网络高吞吐与多 AZ 级高可用。

---

## 3. 高可用、容灾与自保机制 (Resilience & Self-Preservation)

作为实时链路的咽喉要道，Transcribe Service 必须具备极强的自愈能力与防御性边界。系统在设计上充分接受"底层网络与组件必然会发生抖动"的现实，通过严格的背压（Backpressure）与降级机制，保护本行 AWS 核心基础设施不被海量积压流量拖垮。

### 3.1 流量心跳与 Redis 动态内存回收 (Rolling TTL)

在长连接场景下，因客户端网络死机或未发送 `SESSION_COMPLETE` 信号导致的"僵尸会话"是引发内存泄漏（OOM）的元凶。

Transcribe Service 摒弃了粗放的长效固定过期策略，采用基于**流量心跳的滚动续租 (Rolling TTL)** 机制：

- **动态生命周期**：在执行防并发 Lua 脚本的同时，原生集成 `SETEX/EXPIRE` 指令。每次收到合法的业务推送包，Redis 中对应 `sessionId` 的 TTL 锁即**自动向后顺延 15 分钟**。
- **架构收益**：既能完美支撑长达数小时的马拉松式合规通话，又能确保在对端异常死机且无任何业务流转的 15 分钟后，Redis 自动释放内存。以零额外性能开销，实现状态缓存的极致健康运转。

### 3.2 背压机制与 Kafka 异常熔断 (Backpressure & Circuit Breaking)

面对下游 AWS MSK (Kafka) 可能发生的短暂延迟（如 Leader 重新选举）或彻底不可用，Transcribe Service 严禁在自身 Fargate 节点内存中无限制堆积待发送的文本消息，强制将缓存压力反向传导至源头 GCP 环境。

- **单次超时快速失败**：Kafka 异步写入强制配置极短的超时阈值（2000ms）。一旦超时，提交流程立即中断，系统利用 API 契约向 GCP 推流端返回 `E1009` (Downstream unavailable) 错误码，逼迫对端在源头暂存重试。
- **断路器降级 (Circuit Breaker)**：若 Kafka 连续报错触发预设阈值，Transcribe Service 内存断路器"熔断 (Open)"。在冷却期内，直接拒收新推流并秒回 `E1009`，保护自身容器不被耗尽资源。
- **物理切断与自保**：若底层故障持续超限，系统将主动下发 WebSocket Close Code `1013` (Try Again Later) 强行踢掉异常连接，彻底释放本地 TCP 句柄与内存。待基建恢复后，对端可携历史状态无缝重连续传。

#### 3.2.1 熔断器状态转换

| 状态 | 条件 | 行为 |
|------|------|------|
| **Closed** | 正常运行 | 放行全量流量 |
| **Open** | 10 秒内 Kafka 写入失败率 > 15% | 拒绝连接，秒回 `E1009` |
| **Half-Open** | 熔断 30 秒后 | 尝试放行 10% 的流量探测恢复 |

### 3.3 容器漂移与优雅停机 (Graceful Shutdown)

在 ECS Fargate 触发版本发布或缩容时，系统必须平滑处理正在处理中的长连接。

- 接收到容器编排发出的 `SIGTERM` 信号后，Transcribe Service 立即停止接收新连接。
- 对存量 WebSocket 连接主动发送特定的 Close 帧（如 Code 1001/1012），通知对端暂停发送并准备重连至新节点。
- 阻塞主进程退出，直至内存缓冲区中已校验的最后几条记录安全落盘至 Kafka（`flush()`），确保应用漂移期间的绝对零数据丢失。

---

# 第三部分：技术附件与规格 (Technical Annexes & Specs)

---

## 1. API 契约与错误处理定义 (Appendix A: API Contract)

### 1.1 正常响应码

| 响应类型 | 说明 |
|---------|------|
| `TRANSCRIPT_ACK` | 数据已成功投递至 Kafka 并完成状态推进 |

### 1.2 异常响应码 (The "E" Series)

| 错误码 | 名称 | 触发条件 | 对端处理建议 |
|--------|------|---------|-------------|
| **E1006** | 序列号非法 | Lua 返回 -1（首包非 0）或 -2（跳号乱序） | 检查发送序列号，重新同步状态后重发 |
| **E1008** | 内部组件故障 | Redis 连接失败 / ReadOnlyError（主从切换） / 应用未捕获异常 | 延迟重试（指数退避） |
| **E1009** | 下游组件不可用 | Kafka 写入超时 (2s) / 断路器已开启 (Open) | 在源头暂存数据，延迟重试 |

### 1.3 协议级 Close Code

| Close Code | 含义 | 场景 |
|-----------|------|------|
| `1001` | Going Away | 服务端优雅停机，通知客户端重连至新节点 |
| `1012` | Service Restart | 服务版本更新 / 容器漂移 |
| `1013` | Try Again Later | 严重负载 / 持续故障，强制踢断连接释放资源 |

---

## 2. 基础设施配额与调优 (Appendix B: Infra & Capacity)

本章明确了在 AWS 环境中通过基础设施即代码（IaC，如 Terraform/CloudFormation）进行落地时的关键参数基线。所有配额规划均围绕"支撑早高峰 700~1000 并发通话"及"严守绝对保序"两个核心目标制定。

### 2.1 ECS Fargate 计算节点规格

| 参数 | 值 | 说明 |
|------|------|------|
| CPU | 1024 (1 vCPU) | WebSocket 是 IO 密集型，不需要高配 |
| Memory | 2048 (2 GB) | 单 Worker 足够应对数百连接 |
| Ephemeral Storage | 20 GB | 日志缓冲与临时数据 |
| Soft Limit (Ulimit) | `nofile 65535` | 确保单实例能承载高并发连接 |
| 建议单 Task 最大连接数 | 500 | 建议部署 4 个 Pod 以实现 HA |

### 2.2 弹性伸缩逻辑 (Auto-scaling)

放弃 CPU/内存指标，采用基于连接数的扩容策略：

| 动作 | 触发条件 | 响应 |
|------|---------|------|
| **Scale-out (扩容)** | `ALB ActiveConnectionCount > 400` 或 `RequestCountPerTarget` 逼近阈值，持续 2 分钟 | 增加 1 个实例 |
| **Scale-in (缩容)** | `ALB ActiveConnectionCount < 100`，持续 10 分钟 | 减少 1 个实例 |

> **核心原则**：彻底放弃 CPU/内存指标，采用基于 ALB `RequestCountPerTarget`（单目标活跃连接数）或网络吞吐量的策略。一旦逼近阈值，直接触发横向无脑扩容，完美应对瞬时话务高峰。

### 2.3 Kafka Topic 规划

| 参数 | 值 | 说明 |
|------|------|------|
| **Partition 数量** | **定死 50 或 100** | 确保 Hash 取模不变，同一 `sessionId` 绝对保序 |
| 副本因子 (Replication Factor) | 3 | 三副本高可用 |
| `enable_idempotence` | `True` | 生产端幂等性 |
| `min.insync.replicas` | 2 | 配合 `acks=all` |
| `compression.type` | `zstd` | 针对文本转写有极高的压缩比，节省 60% 带宽 |

> **架构底线**：Topic 创建时必须一次性分配足量 Partition。50~100 的数量既无元数据压力，又为下游 AI 消费团队预留了最高 100 个并发线程的充裕吞吐水位。

### 2.4 ALB (Internal Application Load Balancer) 设置

| 参数 | 值 | 说明 |
|------|------|------|
| 监听协议 | WSS (TLS 1.2+) | 多云专线隔离入口 |
| Idle Timeout | 3600s | 通过底层 Ping/Pong 维系，ALB 不主动切断 |
| Sticky Sessions | **禁用** | 依靠 Redis 做全局状态，Session 漂移不影响业务 |

### 2.5 ElastiCache (Redis) 配置

| 参数 | 值 | 说明 |
|------|------|------|
| 引擎类型 | Redis | 极轻量键值存储 |
| 部署架构 | 主从 / 多 AZ 集群 | 保障高可用与自动故障切换 |
| 端口 | 6379 | 仅对 Fargate 安全组开放 |

### 2.6 核心组件配额总览

| **核心组件 (AWS Services)** | **核心配置与配额约束** | **架构规划依据与触发策略** |
|---|---|---|
| **ECS Fargate** (计算节点) | 初始/最小实例数：N；最大实例数：M；vCPU/内存：视压测调优 | 基于 `RequestCountPerTarget` 扩容，建议单 Pod 安全并发连接上限 200 |
| **MSK (Kafka)** (消息总线) | Partition 数量：定死 50 或 100；副本因子：3 | Hash 路由基数不可更改，为下游预留充裕吞吐水位 |
| **ElastiCache (Redis)** (状态中心) | Redis 主从/多 AZ 集群 | 极轻量键值对存储，重点保障网络吞吐与高可用 |
| **Internal ALB** (内部网关) | WSS (TLS 1.2+) | 配合 20 秒 Ping/Pong 规避 ALB 60 秒空闲连接绞杀 |

---

## 3. 运维与可观测性 (Appendix C: Observability)

Transcribe Service 作为数据流转的咽喉，其运行状态的透明度至关重要。为解决跨云链路定位难、多部门扯皮成本高的问题，系统在设计上强制推行"结构化、可追溯"的监控审计方案。

### 3.1 结构化 JSON 审计日志 (Structured Logging)

系统抛弃传统的非结构化文本 Log，全面采用 **JSON 格式的结构化日志**。每一条日志均作为一个独立的、可检索的数据对象，直接对接 AWS CloudWatch Logs。

- **核心字段规范**：
  - `sessionId`: 全局唯一会话 ID，用于串联单通电话的全链路轨迹。
  - `sequenceNumber`: 业务序列号，用于核对转写文本的连续性。
  - `eventType`: 事件类型（如 `WS_CONNECTED`, `KAFKA_PUBLISHED`, `ERROR_E1006`）。
  - `latency_ms`: 单次处理耗时（从接收 WebSocket 到收到 Kafka Ack 的时长）。

**日志格式示例**：

```
[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms
```

- **架构收益**：运维团队无需编写复杂的正则表达式，即可通过 **CloudWatch Logs Insights** 使用类似 SQL 的语句对特定会话进行秒级排障或性能分析。

### 3.2 核心监控指标与告警 (Golden Signals)

系统在 CloudWatch Dashboard "Transcript-Gate-Overview" 中重点监控以下核心"黄金指标"，并设定实时告警：

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| **建连成功率** | 监控 WebSocket 握手异常，及时发现多云专线抖动 | — |
| **Kafka 投递 P99 延迟** | 延迟持续走高时自动触发熔断降级 | P99 > 80ms |
| **契约错误触发频率** | 重点关注 `E1006` 和 `E1009` 的抛出频率 | — |
| **活跃连接数 (Active Connections)** | 直接关联 Fargate Auto-scaling 状态 | — |
| **5xx Error Rate** | 严重 | > 1% |
| **Kafka Cluster Disk Usage** | 警告 | > 80% |

### 3.3 故障模式与影响分析 (FMEA)

| **组件** | **故障场景** | **探测机制** | **自动修复/降级动作** |
|---|---|---|---|
| **GCP 侧引擎** | 停止发包 | Ping/Pong 超时 | 15 分钟后自动回收 Redis 状态 |
| **GCP 专线瞬断** | 所有 WebSocket 连接掉线 | 连接异常事件 | 客户端触发指数退避重连；Service B 依靠 Redis 续存状态，断点续传 |
| **Fargate 实例** | OOM 崩溃 / 意外宕机 | ALB Health Check (HTTP 80) | 30 秒内 ALB 摘除节点，Fargate 自动拉起新实例；其余实例承载重连流量 |
| **Redis 节点** | 主从切换 (Failover) | 驱动抛出 `ReadOnlyError` | 应用捕获异常，抛出 `E1008` 触发上游延迟重试；约 2-5 秒写入阻塞 |
| **Kafka 阻塞** | 磁盘满或网络慢 / 集群压力过载 | `KafkaTimeoutException` / P99 延迟报警 | 触发熔断器，向全量连接下发 `E1009` 进行背压 |

---

## 4. 技术辩护与常见问题 (Appendix D: FAQ)

### 4.1 选型辩护：为什么 Python 单线程异步 + 多进程 Worker 优于传统多线程？

**Q: Transcribe Service 采用单线程异步模型（Asyncio），在高并发场景下性能是否会成为瓶颈？为什么不采用 Java 风格的多线程模型？**

**A：** 这是一个常见的认知误区。在处理"长连接、高并发 I/O"场景时，Python 的单线程异步模型在资源利用率和吞吐量上通常优于多线程模型。理由如下：

#### 4.1.1 I/O 密集型场景的本质

Transcribe Service 的业务逻辑属于典型的 **I/O 密集型（I/O Bound）**。系统 99% 的时间都在等待网络数据包（来自 GCP 的 WebSocket 或投递给 AWS Kafka 的确认）。

- **多线程模型的问题**：由于 Python **GIL（全局解释器锁）** 的限制，即使开启多线程，同一时刻也只有一个线程能占用 CPU。在数千个长连接并发时，操作系统会频繁进行**上下文切换（Context Switch）**，这种内核态的切换开销会产生巨大的 CPU 浪费。
- **异步协程模型**：通过 `asyncio` 事件循环（Event Loop），系统在等待 I/O 的间隙可以立即处理其他连接的任务。这种切换发生在用户态，成本极低（仅相当于函数调用）。

#### 4.1.2 内存开销对比

- **多线程**：每个线程通常需要分配 8MB 的栈空间。1000 个并发连接如果开启 1000 个线程，仅线程栈就会占用约 8GB 内存，这在 Fargate 容器中是极大的资源浪费。
- **异步协程**：每个协程（Coroutine）仅占用几 KB 内存。1000 个并发连接的内存占用微乎其微。

#### 4.1.3 核心并发指标对比 (Benchmark Logic)

| **维度** | **多线程模型 (Multi-threading)** | **异步协程模型 (Asyncio)** |
|---|---|---|
| **执行模式** | 并发执行（受 GIL 限制，实际上是伪并行） | 协作式多任务（单线程内高效轮询） |
| **上下文切换** | **内核态切换**，开销大，易导致 CPU 抖动 | **用户态切换**，开销极低，CPU 曲线平滑 |
| **并发上限** | 受限于操作系统的最大线程数（通常几百个） | 轻松支持上万个并发长连接（C10K 级别） |
| **数据安全性** | 需频繁使用 Lock/Mutex，易产生死锁或竞态 | **天然线程安全**，无需复杂加锁逻辑 |

#### 4.1.4 如何利用多核 CPU？

单线程并不意味着只用一个核。在实施方案中，我们采用了 **"每 vCPU 一个独立 Worker 进程"** 的部署模式。例如，如果 Fargate 分配了 4 个 vCPU，我们会启动 4 个独立的异步 Worker 进程。这不仅规避了 GIL 限制，还实现了真正的多核并行处理（Parallelism）。

### 4.2 性能深度解析：单线程阻塞风险

**Q: 如果单线程内某个逻辑出现了阻塞（例如复杂的 JSON 解析），是否会拖慢所有连接？**

**A：** 是的，这是异步编程需要规避的风险。为此，本设计采取了以下防御措施：

1. **轻量化逻辑**：本网关仅做序列号校验和转发，不进行任何重量级计算或复杂的文本分析。
2. **异步库强制化**：所有涉及网络和存储的操作必须使用 `await` 关键字配合异步驱动（如 `aiokafka`, `redis.asyncio`），确保绝对不阻塞事件循环。
3. **多进程分担**：多进程架构确保了即使一个进程瞬时繁忙，其他核心上的进程仍能正常响应。

---

## 5. 性能基线与压测目标 (Performance Baseline)

| **指标** | **目标值 (Target)** | **备注** |
|---|---|---|
| **最大并发连接** | 2,000 | 预留 2 倍余量 |
| **平均处理耗时** | < 15ms | 不含 Kafka 传输开销 |
| **P99 投递延迟** | < 80ms | 含 Kafka 三副本确认 |
| **单 Pod 承载力** | 500 连接 | 建议部署 4 个 Pod 以实现 HA |
| **端到端延迟** | < 50ms | 从 GCP 接收到进入 AWS Kafka |

---

## 6. 安全与合规审计 (Security & Compliance)

### 6.1 IAM 最小权限原则

- **Fargate Role**:
  - `kms:Decrypt` (解密专线证书)。
  - `kafka:Publish` (仅限特定 Topic)。
  - `elasticache:Connect` (仅限 Redis 端口)。

### 6.2 日志审计 (Log Specs)

日志必须包含 **Trace ID**，格式如下：

```
[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms
```

---

## 7. 演进与可扩展性 (Future Extensibility)

本系统的架构设计并非一成不变的终点，而是为本行未来更丰富的 AI 业务生态预留了充足的"技术白板"。

### 7.1 "数据漏斗"式的下游解耦

Transcribe Service 坚持将文本流投递至 Kafka 这一标准工业总线，其核心价值在于**彻底解耦了生产者与消费者**：

- **零入侵扩展**：未来若本行 AI 团队需要新增"客户意图识别"、"实时风险监控"或"坐席辅助提示"等模块，只需作为独立的 **Kafka Consumer Group** 接入即可。
- **架构收益**：新增任何业务功能，均不需要对 Transcribe Service 的核心网关代码做任何修改（Zero Intrusion），确保了核心通信链路的绝对稳定性。

### 7.2 多云治理的灵活性

虽然目前的 ASR 引擎托管在 GCP 环境，但由于 Transcribe Service 基于标准的 WebSocket 协议与 WSS 加密链路，未来若有其他引擎（如部署在 AWS 原生环境或其他云平台）需要接入，只需遵循现有的 API 契约与内网路由规则，即可实现低成本的快速迁移与并入。

---

# 附录：核心参考代码 (Development Reference)

> 以下代码为核心逻辑封装的参考实现，供开发阶段对照使用。

## 核心事实清单 (Grounding Truths)

- **技术栈**：Python 3.10+ / FastAPI / redis.asyncio / aiokafka。
- **并发模型**：单线程异步协程（Asyncio）。单 vCPU 对应单进程 Worker。
- **状态管理**：使用 Redis Lua 脚本实现"两阶段提交"（预检 -> 写入 Kafka -> 提交状态）。
- **Kafka 策略**：以 sessionId 为 Partition Key 确保严格保序；acks=all, enable_idempotence=True。
- **保活机制**：WebSocket 开启 20s/次 Ping/Pong。

## TranscribeStateManager 参考实现

```python
import asyncio
import logging
from redis.asyncio import Redis, ConnectionPool
from typing import Optional, Tuple

logger = logging.getLogger("transcribe.service")

class TranscribeStateManager:
    """
    负责维护分布式环境下的会话状态与序列号校验
    实现逻辑：Lua 脚本下沉 + 两阶段提交
    """
    # 脚本 A：预检逻辑 (Check Only) - 只读不写，极速执行
    LUA_PRE_CHECK = """
    local current = redis.call('GET', KEYS[1])
    local incoming = tonumber(ARGV[1])

    if not current then
        if incoming == 0 then return 1 end -- 初始包合法
        return -1 -- 错误：会话未初始化但序号非0 (E1006)
    end

    if incoming == tonumber(current) then
        return 1 -- 准许投递 Kafka
    elseif incoming < tonumber(current) then
        return 0 -- 幂等重复 (已处理，直接回ACK)
    else
        return -2 -- 跳号错误 (E1006)
    end
    """

    # 脚本 B：推进逻辑 (Commit & TTL) - 只有 Kafka 成功后调用
    LUA_COMMIT_SEQ = """
    local key = KEYS[1]
    local ttl = tonumber(ARGV[1])
    redis.call('INCR', key)
    redis.call('EXPIRE', key, ttl)
    return 1
    """

    def __init__(self, redis_pool: ConnectionPool, ttl: int = 900):
        self.redis = Redis(connection_pool=redis_pool)
        self.ttl = ttl
        self._pre_check_sha = None
        self._commit_sha = None

    async def load_scripts(self):
        """预加载脚本到 Redis 缓存，获取 SHA1 以提升性能"""
        self._pre_check_sha = await self.redis.script_load(self.LUA_PRE_CHECK)
        self._commit_sha = await self.redis.script_load(self.LUA_COMMIT_SEQ)
        logger.info("Redis Lua scripts loaded into server cache.")

    async def validate_sequence(self, session_id: str, incoming_seq: int) -> Tuple[bool, Optional[str]]:
        """
        第一阶段：预检
        返回: (是否准许写入Kafka, 错误码)
        """
        key = f"ts:sess:{session_id}"
        try:
            result = await self.redis.evalsha(self._pre_check_sha, 1, key, incoming_seq)

            if result == 1:
                return True, None
            elif result == 0:
                return False, "IDEMPOTENT"
            elif result == -1 or result == -2:
                return False, "E1006"

        except Exception as e:
            logger.error(f"Redis pre-check failed for {session_id}: {e}")
            return False, "E1008"

    async def commit_sequence(self, session_id: str):
        """
        第二阶段：提交状态
        只有在 Kafka 返回 Ack 后调用
        """
        key = f"ts:sess:{session_id}"
        try:
            await self.redis.evalsha(self._commit_sha, 1, key, self.ttl)
        except Exception as e:
            logger.critical(f"FATAL: Kafka synced but Redis state failed to advance for {session_id}: {e}")
```

## 代码实现要求清单

1. **WebSocket 路由**：实现 `/ws/transcribe` 接口。需包含握手阶段的 `Authorization: Bearer` 校验。
2. **消息循环**：
   - 收到消息后先调用 `validate_sequence`。
   - 若为 `IDEMPOTENT`，记录 Debug 日志并回 Ack，不写 Kafka。
   - 若验证通过，异步写入 Kafka 并设置 2s 超时。
   - Kafka 成功后调用 `commit_sequence`。
3. **异常处理中间件**：
   - 捕获 `KafkaTimeoutError` 或断路器异常，返回 `E1009`。
   - 捕获所有未定义异常，记录 Error 日志并返回 `E1008`。
4. **优雅停机**：监听 `SIGTERM` 信号，确保在关闭连接前，Kafka 生产缓冲区的所有消息已完成 `flush()`。
5. **结构化日志**：使用 `logging` 或 `structlog` 输出 JSON 格式日志，每行必须携带 `sessionId` 和 `seq`。
6. **Kafka Producer 初始化**：确保在 FastAPI 的 `lifespan`（或 `on_event("startup")`）里初始化，而非每个请求都初始化。
7. **Lua 脚本加载**：确保只在启动时 `script_load` 一次。
8. **连接释放**：确保 WebSocket 的 `receive` 循环能正确感知到客户端断开，并释放 Redis 连接。

---

*— 文档结束 —*
