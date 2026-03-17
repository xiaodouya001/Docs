# 📚 Transcribe Service 实时语音转写网关全量技术实施方案 (V1.0)

## 1. 引言与背景 (Introduction)

## 1.1 文档目的

本说明书旨在为“实时语音转写网关（以下简称 Transcribe Service）”提供极尽详细的工程实施指导。涵盖应用层异步逻辑、分布式状态一致性、多云网络路由及 AWS 基础设施的精确配额。

## 1.2 业务目标

支撑本行 Call Center 早高峰 **1000路并发通话** 的实时转写流，保证端到端延迟（从 GCP 接收到进入 AWS Kafka）在 **50ms** 以内，且数据**绝对保序、零丢失**。

------

## 2. 系统深度架构设计 (Deep Dive Architecture)

## 2.1 异步事件驱动模型

Transcribe Service 采用 Python **FastAPI + Uvicorn** 的非阻塞 I/O 模型。

- **Worker 机制**：采用单进程单线程（Single-process, Single-threaded）模式，通过 `asyncio` 事件循环调度。在 Fargate 容器中，每个 vCPU 对应一个 Worker 进程，避免多线程竞争产生的 GIL（全局解释器锁）开销。
- **连接控制**：使用 `websockets` 库，并配置 `max_size=1MB` 以防止大包攻击。

## 2.2 跨云网络路由轨迹 (Network Trace)

1. **GCP 出口**：GCP GKE 内部 Service A 发起建连，通过 **Cloud Interconnect** 路由。
2. **专线穿越**：数据包通过专用 10Gbps 链路。
3. **AWS 入口**：进入 **AWS Direct Connect Gateway**。
4. **内部网关**：ALB 监听 443 端口，挂载内网 SSL 证书，执行 SSL Termination。
5. **目标转发**：ALB 根据权重将请求路由至 Fargate 任务的 8080 端口。

------

## 3. 核心业务逻辑实现规格 (Implementation Specs)

## 3.1 增强型 Redis Lua 原子校验 (The "Sequence Guard")

为防止“脑裂”和重复投递，Lua 脚本必须处理复杂的边界情况。

**脚本逻辑详情：**

Lua

```
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

## 3.2 两阶段提交 (2PC) 与断点续传逻辑

系统不使用分布式事务，而是通过“状态滞后推进”实现一致性：

1. **Prepare**: Service A 发送 `seq=5`。Transcribe Service 调用 Lua 预检。
2. **Persistence**: 写入 Kafka。设置 `acks=all`。
3. **Commit**: 收到 Kafka Ack。调用 Redis `INCR` 脚本将期望值推至 6。
4. **Ack**: 回复 `TRANSCRIPT_ACK`。

- **异常处理**：若 Kafka 写入失败，不执行第 3 步。上游超时后重发 `seq=5`，Redis 此时存的仍是 5，预检依然通过，实现无损重试。

------

## 4. 存储与通信协议细节 (Data & Protocol)

## 4.1 Kafka 消息定义

- **Topic**: `transcription.raw.stream`

- **Partition Key**: `sessionId` (String)

- **Message Value (Protobuf/JSON)**:

  JSON

  ```
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

- **Retention**: 7天。

## 4.2 Redis 键空间管理 (Keyspace)

- **Key 命名**: `ts:sess:{sessionId}`
- **内存估算**: 每个 Session 占用约 100 字节。1000 并发下，Redis 内存占用不足 1MB，极度轻量。

------

## 5. 基础设施配额与调优 (Infrastructure & Tuning)

## 5.1 ECS Fargate 详细配置

- **Task Definition**:
  - `Cpu`: 1024 (1 vCPU)
  - `Memory`: 2048 (2 GB)
- **Soft Limit (Ulimit)**: `nofile 65535` (确保单实例能承载高并发连接)。

## 5.2 ALB (Application Load Balancer) 设置

- **Idle Timeout**: 3600s (通过底层 Ping/Pong 维系，ALB 不主动切断)。
- **Sticky Sessions**: **禁用**。因为我们依靠 Redis 做全局状态，Session 漂移不影响业务。

## 5.3 MSK (Kafka) 参数调优

- **min.insync.replicas**: 2 (配合 acks=all)。
- **compression.type**: `zstd` (针对文本转写有极高的压缩比，节省 60% 带宽)。

------

## 6. 故障模式影响分析 (FMEA)

| **组件**         | **故障场景**        | **探测机制**               | **自动修复/降级动作**                          |
| ---------------- | ------------------- | -------------------------- | ---------------------------------------------- |
| **GCP 侧引擎**   | 停止发包            | Ping/Pong 超时             | 15 分钟后自动回收 Redis 状态。                 |
| **Fargate 实例** | OOM 崩溃            | ALB Health Check (HTTP 80) | 30 秒内 ALB 摘除节点，Fargate 自动拉起新实例。 |
| **Redis 节点**   | 主从切换 (Failover) | 驱动抛出 `ReadOnlyError`   | 应用捕获异常，抛出 `E1008` 触发上游延迟重试。  |
| **Kafka 阻塞**   | 磁盘满或网络慢      | `KafkaTimeoutException`    | 触发熔断器，向全量连接下发 `E1009` 进行背压。  |

------

## 7. 安全与合规审计 (Security & Compliance)

## 7.1 IAM 最小权限原则

- **Fargate Role**:
  - `kms:Decrypt` (解密专线证书)。
  - `kafka:Publish` (仅限特定 Topic)。
  - `elasticache:Connect` (仅限 Redis 端口)。

## 7.2 日志审计 (Log Specs)

日志必须包含 **Trace ID**，格式如下：

```
[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms
```

------

## 8. 性能基线与压测目标 (Performance Baseline)

| **指标**          | **目标值 (Target)** | **备注**                      |
| ----------------- | ------------------- | ----------------------------- |
| **最大并发连接**  | 2,000               | 预留 2 倍余量。               |
| **平均处理耗时**  | < 15ms              | 不含 Kafka 传输开销。         |
| **P99 投递延迟**  | < 80ms              | 含 Kafka 三副本确认。         |
| **单 Pod 承载力** | 500 连接            | 建议部署 4 个 Pod 以实现 HA。 |

