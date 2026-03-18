# Transcribe Service 详细设计与技术规格说明书 (Detailed Design)

## 1. 系统交互时序图 (System Sequence Diagram)

*这一部分是用图示（逻辑描述）来定义每一毫秒数据是怎么跑的，解决“谁先发起、谁后返回”的问题。*

## 1.1 正常流转与两阶段提交时序

1. **GCP Side**: 发起 WSS 连接 -> 校验 Token 成功。
2. **GCP Side**: 推送 `SESSION_ONGOING` (seq=N)。
3. **Transcribe Service**: 拦截消息 -> 调用 Redis Lua 脚本。
4. **Redis**: 校验 `seq == expected` -> 返回 `PRE_CHECK_OK` (不自增)。
5. **Transcribe Service**: 异步调用 Kafka Producer 发送数据。
6. **AWS MSK**: 返回 `RecordMetadata` (Ack)。
7. **Transcribe Service**: 再次调用 Redis 脚本 -> 期望值自增 1 (Commit)。
8. **Transcribe Service**: 回复 `TRANSCRIPT_ACK` 给 GCP。

------

## 2. 数据契约与存储 Schema (Data Schema & Storage)

*把每一行代码要读写的格式定死。*

## 2.1 Redis 存储结构 (State Store)

- **Key**: `transcript:session:{sessionId}`
- **Value Type**: `Hash`
- **Fields**:
  - `expected_seq`: (int) 下一个期望接收的序列号。
  - `start_time`: (timestamp) 首次建连时间。
  - `last_active`: (timestamp) 最后一次活跃时间，用于监控。
- **TTL策略**: 900秒（滚动过期）。

## 2.2 Kafka 生产规格

- **Topic Name**: `cc.transcript.realtime.v1`
- **Message Key**: `sessionId` (String)
- **Compression**: `lz4` (平衡 CPU 与带宽)。
- **Acks**: `all` (确保三副本落盘)。

------

## 3. 核心逻辑处理流程 (Logic Deep Dive)

*这一部分是开发最关心的，包含了你提到的 Lua 和熔断逻辑。*

## 3.1 Redis Lua 原子校验逻辑（伪代码）

Lua

```
-- 入参: KEYS[1]=sessionId, ARGV[1]=incomingSeq, ARGV[2]=ttl
local current = redis.call('get', KEYS[1])
if not current then
    -- 初始化会话
    if tonumber(ARGV[1]) == 0 then
        redis.call('setex', KEYS[1], ARGV[2], 1)
        return 1 -- OK
    else return -1 -- 乱序（首包非0）
    end
end
if tonumber(ARGV[1]) == tonumber(current) then
    return 1 -- 匹配，准许处理
elseif tonumber(ARGV[1]) < tonumber(current) then
    return 0 -- 重复包，幂等处理
else
    return -1 -- 跳号，阻断
end
```

## 3.2 熔断与自保策略阈值

- **熔断器状态转换**:
  - **Closed**: 正常运行。
  - **Open**: 当 10 秒内 Kafka 写入失败率 > 15%，拒绝连接，秒回 `E1009`。
  - **Half-Open**: 30 秒后尝试放行 10% 的流量。

------

## 4. 基础设施参数与配额 (Infrastructure Specs)

*这一部分给 Cloud Engineer 配置环境。*

## 4.1 ECS Fargate 任务规格

- **CPU**: 1 vCPU / **Memory**: 2 GB (WebSocket 是 IO 密集，不需要高配)。
- **Ephemeral Storage**: 20 GB。
- **Max Concurrency per Task**: 建议 500 个连接。

## 4.2 自动扩缩容 (Auto-scaling) 详细参数

- **Scale-out (扩容)**: `ALB ActiveConnectionCount > 400` 持续 2 分钟 -> 增加 1 个实例。
- **Scale-in (缩容)**: `ALB ActiveConnectionCount < 100` 持续 10 分钟 -> 减少 1 个实例。

------

## 5. 故障模式与影响分析 (FMEA)

*这一部分是方案说明书的灵魂，告诉老板我们考虑了所有死法。*

| **故障场景**             | **影响**                | **应对/自愈机制**                                            |
| ------------------------ | ----------------------- | ------------------------------------------------------------ |
| **GCP 专线瞬断**         | 所有 WebSocket 连接掉线 | 客户端 (GCP) 触发指数退避重连；Service B 依靠 Redis 续存状态，实现断点续传。 |
| **Redis 主从切换**       | 约 2-5 秒的写入阻塞     | Python 捕获异常，通过 API 抛出 E1008，让 GCP 端重试，不影响数据一致性。 |
| **Kafka 集群压力过载**   | 消息写入延迟增加        | 第一时间触发 P99 延迟报警，系统自动通过 E1009 实施“背压”，将压力推回源头。 |
| **Fargate 实例意外宕机** | 承载的连接断开          | ALB 自动剔除健康检查失败的实例，其余实例自动承载重连流量，数据依靠 Redis 恢复。 |

------

## 6. 操作手册与维护 (Ops Manual)

- **监控看板**: CloudWatch Dashboard "Transcript-Gate-Overview"。
- **报警阈值**:
  - `5xx Error Rate > 1%` (严重)。
  - `Kafka Cluster Disk Usage > 80%` (警告)。