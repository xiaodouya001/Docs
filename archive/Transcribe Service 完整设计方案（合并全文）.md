# Transcribe Service 完整设计方案（合并全文）

> 本文档整合架构白皮书、详细设计说明书、全量技术实施方案及开发指令，提供端到端技术设计参考。API 契约详见《Transcribe Service API Contract 契约文档》。

---

## 文档结构

| 章节 | 内容 |
|------|------|
| 一、执行摘要与系统边界 | 业务背景、目标、职责边界 |
| 二、网络架构与安全 | 多云拓扑、VPC 隔离、零信任加固 |
| 三、核心应用逻辑 | 技术栈、连接生命周期、乐观锁、2PC、高可用 |
| 四、存储与通信 | Kafka、Redis 配置与消息结构 |
| 五、基础设施配额 | Fargate、ALB、弹性伸缩 |
| 六、运维与可观测性 | 日志、监控、FMEA |
| 七、API 契约与错误码 | 详见 API Contract 契约文档 |
| 八、性能、安全与扩展 | 性能基线、IAM、演进方向 |
| 九、FAQ | 异步选型辩护、阻塞风险 |
| 附录 | 参考代码与实现要求 |

---

# 一、执行摘要与系统边界

## 1.1 业务背景与系统定位

本行在 GCP 环境托管第三方 ASR 引擎（FanoLab），负责将客服通话语音实时转为文本。**Transcribe Service (Service B)** 部署于 AWS，作为 **GCP ↔ AWS 多云实时数据网关**，承接转写文本并投递至内部 Kafka。

## 1.2 业务目标

- 支撑早高峰 **700~1000 路** 并发通话实时转写
- 端到端延迟 < **50ms**（GCP 接收 → AWS Kafka）
- 数据**绝对保序、零丢失**

## 1.3 职责边界

| 类型 | 内容 |
|------|------|
| **In-Scope** | 多云长连接管理；基于 sessionId/sequenceNumber 的保序校验；可靠投递至 Kafka |
| **Out-of-Scope** | 不处理音频流；不包含意图识别、情感分析等业务逻辑；下游需自行订阅 Kafka |

---

# 二、网络架构与安全

## 2.1 多云骨干网拓扑

- GCP 与 AWS 通过 **Direct Connect + Cloud Interconnect** 专线互联，不经过公网
- **路由轨迹**：GCP GKE → Cloud Interconnect → 10Gbps 专线 → Direct Connect Gateway → Internal ALB (443) → Fargate (8080)

## 2.2 VPC 隔离与安全组

| 层级 | 组件 | 策略 |
|------|------|------|
| 接入层 | Internal ALB | 仅放行 GKE 集群 CIDR |
| 计算层 | ECS Fargate | 仅允许 ALB 入站 |
| 存储层 | Redis、Kafka | 仅对 Fargate 开放 6379、9092 |

## 2.3 纵深防御

- **传输加密**：ALB TLS 1.2+，本行 CA 证书
- **应用鉴权**：WebSocket 握手校验 `Authorization: Bearer`
- **落盘加密**：MSK、Redis 启用 KMS 加密

---

# 三、核心应用逻辑

## 3.1 技术栈与并发模型

- **技术栈**：Python 3.10+ / FastAPI / Uvicorn / redis.asyncio / aiokafka
- **并发模型**：单线程 Asyncio，单 vCPU 对应单 Worker 进程
- **连接控制**：websockets，`max_size=1MB`
- **选型理由**：I/O 密集型场景下，异步协程在上下文切换、内存占用上优于多线程，且天然线程安全

## 3.2 连接生命周期与保活

- **业务信号**：监听 `SESSION_ONGOING`、`SESSION_COMPLETE`
- **协议保活**：20 秒/次 Ping/Pong，规避 ALB 60 秒空闲超时

## 3.3 乐观数据锁 (Sequence Guard)

状态全下沉 Redis，无本地会话状态。基于序列号的原子校验：

| 条件 | 行为 |
|------|------|
| seq == expected | 放行投递 |
| seq < expected | 幂等，直接回 ACK，不写 Kafka |
| seq > expected | 阻断，返回 E1006 |

**Redis Lua 预检脚本**（返回码：100 初始化 / 200 通过 / 300 幂等 / 400 首包非法 / 500 跳号）：

```lua
local current = redis.call('GET', KEYS[1])
if not current then
    if tonumber(ARGV[1]) == 0 then
        redis.call('SETEX', KEYS[1], ARGV[2], 1)
        return 100
    else return 400 end
end
if tonumber(ARGV[1]) == tonumber(current) then return 200
elseif tonumber(ARGV[1]) < tonumber(current) then return 300
else return 500 end
```

**推进脚本**（仅 Kafka Ack 后调用）：

```lua
redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], ARGV[1])
return 1
```

## 3.4 两阶段提交 (2PC)

| 阶段 | 操作 |
|------|------|
| Prepare | Lua 预检，不自增 |
| Persistence | 写入 Kafka，sessionId 为 Key，acks=all |
| Commit | Kafka Ack 后调用 Redis INCR |
| Ack | 回复 TRANSCRIPT_ACK |

Kafka 失败则不 Commit，上游重试时 Redis 仍为原值，实现无损重试。

## 3.5 高可用与自保

| 机制 | 说明 |
|------|------|
| **Rolling TTL** | 每次合法推送将 sessionId TTL 顺延 15 分钟，僵尸会话自动回收 |
| **背压** | Kafka 写入超时 2s 即失败，返回 E1009 |
| **熔断器** | 10s 内失败率 > 15% 熔断，秒回 E1009；30s 后 Half-Open 放行 10% 流量 |
| **优雅停机** | 收 SIGTERM 后停新连接，发 1001/1012 通知存量连接，flush Kafka 后退出 |

---

# 四、存储与通信

## 4.1 Kafka

| 项目 | 配置 |
|------|------|
| Topic | `transcription.raw.stream` 或 `cc.transcript.realtime.v1` |
| Partition Key | sessionId |
| Partition 数量 | **定死 50 或 100**（Hash 基数不可变） |
| 可靠性 | acks=all, enable_idempotence=True, max_in_flight=1 |
| 压缩 | zstd |
| 保留 | 7 天 |

**消息结构**：`header`（sessionId, version, producerId）+ `payload`（seq, text, timestamp）

## 4.2 Redis

| 项目 | 配置 |
|------|------|
| Key | `ts:sess:{sessionId}` |
| TTL | 900s 滚动续租 |
| 内存 | 约 100B/session，1000 并发 < 1MB |
| 部署 | 主从 / 多 AZ 集群 |

---

# 五、基础设施配额

## 5.1 ECS Fargate

| 参数 | 值 |
|------|------|
| CPU / Memory | 1 vCPU / 2 GB |
| Ephemeral Storage | 20 GB |
| nofile | 65535 |
| 单 Task 建议连接数 | 500 |

## 5.2 弹性伸缩

- **扩容**：`ActiveConnectionCount > 400` 持续 2 分钟 → +1 实例
- **缩容**：`ActiveConnectionCount < 100` 持续 10 分钟 → -1 实例
- **原则**：基于连接数，不用 CPU/内存

## 5.3 ALB

- 协议：WSS (TLS 1.2+)
- Idle Timeout：3600s
- Sticky Sessions：禁用

---

# 六、运维与可观测性

## 6.1 结构化日志

JSON 格式，必含 `sessionId`、`sequenceNumber`、`eventType`、`latency_ms`。对接 CloudWatch Logs Insights。

**示例**：`[2026-03-17 10:26:00] [INFO] [sid:38422] [seq:45] Action:PUBLISH_TO_KAFKA Result:SUCCESS Latency:12ms`

## 6.2 监控与告警

| 指标 | 告警 |
|------|------|
| 建连成功率、活跃连接数 | — |
| Kafka P99 延迟 | > 80ms |
| E1006 / E1009 频率 | — |
| 5xx Error Rate | > 1% |
| Kafka Disk Usage | > 80% |

## 6.3 FMEA

| 组件 | 故障 | 应对 |
|------|------|------|
| GCP 引擎 | 停止发包 | 15 分钟后 Redis 回收 |
| GCP 专线 | 连接掉线 | 客户端指数退避重连，Redis 断点续传 |
| Fargate | OOM/宕机 | ALB 摘除，自动拉起新实例 |
| Redis | 主从切换 | 捕获 ReadOnlyError，返回 E1008 |
| Kafka | 阻塞/过载 | 熔断，返回 E1009 背压 |

---

# 七、API 契约与错误码

详见 [Transcribe Service API Contract 契约文档](./Transcribe%20Service%20API%20Contract%20契约文档.md)。

---

# 八、性能、安全与扩展

## 8.1 性能基线

| 指标 | 目标 |
|------|------|
| 最大并发 | 2,000 |
| 平均处理耗时 | < 15ms |
| P99 投递延迟 | < 80ms |
| 端到端延迟 | < 50ms |

## 8.2 IAM 最小权限

Fargate Role：`kms:Decrypt`、`kafka:Publish`、`elasticache:Connect`

## 8.3 演进方向

- **下游解耦**：新业务以 Kafka Consumer Group 接入，零侵入
- **多云灵活**：遵循 API 契约即可接入其他云/引擎

---

# 九、FAQ

## 9.1 为何选 Python 异步而非多线程？

- **I/O 密集型**：99% 时间在等网络，异步在 I/O 间隙切换任务，用户态切换成本低
- **GIL**：多线程受 GIL 限制，上下文切换为内核态，开销大
- **内存**：1000 线程约 8GB 栈；1000 协程仅数 MB
- **多核**：每 vCPU 一 Worker 进程，规避 GIL，实现多核并行

## 9.2 单线程阻塞会拖慢所有连接吗？

会。防御措施：轻量逻辑、全异步库（aiokafka/redis.asyncio）、多进程分担。

---

# 附录：核心参考代码与实现要求

## 核心事实清单

- 技术栈：Python 3.10+ / FastAPI / redis.asyncio / aiokafka
- 并发：单线程 Asyncio，单 vCPU 单 Worker
- 状态：Redis Lua 两阶段提交
- Kafka：sessionId 为 Key，acks=all，enable_idempotence=True
- 保活：20s Ping/Pong

## TranscribeStateManager 参考实现

```python
class TranscribeStateManager:
    LUA_PRE_CHECK = """
    local current = redis.call('GET', KEYS[1])
    local incoming = tonumber(ARGV[1])
    if not current then
        if incoming == 0 then return 1 end
        return -1
    end
    if incoming == tonumber(current) then return 1
    elseif incoming < tonumber(current) then return 0
    else return -2 end
    """

    LUA_COMMIT_SEQ = """
    redis.call('INCR', KEYS[1])
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))
    return 1
    """

    async def validate_sequence(self, session_id: str, incoming_seq: int) -> Tuple[bool, Optional[str]]:
        # 返回 (准许写Kafka, 错误码)
        # result: 1=通过, 0=幂等, -1/-2=E1006
        ...

    async def commit_sequence(self, session_id: str):
        # Kafka Ack 后调用
        ...
```

## 实现要求清单

1. WebSocket 路由 `/ws/transcribe`，握手校验 `Authorization: Bearer`
2. 消息循环：validate_sequence → 若 IDEMPOTENT 则回 ACK 不写 Kafka；否则写 Kafka(2s 超时) → commit_sequence → 回 ACK
3. 异常：KafkaTimeout/熔断 → E1009；未捕获异常 → E1008
4. 优雅停机：SIGTERM → flush Kafka → 关闭连接
5. 日志：JSON，含 sessionId、seq
6. Kafka Producer、Lua script_load 仅在 lifespan 初始化一次
7. 正确感知客户端断开并释放 Redis 连接

---

*— 文档结束 —*
