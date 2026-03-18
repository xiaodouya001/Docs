# 📑 Transcribe Service 完整设计方案：内容索引大纲

## 第一部分：架构白皮书 (Architecture Whitepaper)

*侧重于业务边界、合规性与宏观设计原则*

## 1. 执行摘要 (Executive Summary)

- **1.1 业务背景**：呼叫中心 ASR 实时转写对接需求。
- **1.2 系统定位**：本行内部多云环境（GCP 托管端至 AWS 接收端）的实时数据网关。
- **1.3 核心职责 (In-Scope)**：长连接管理、事务保序、可靠投递。
- **1.4 边界声明 (Out-of-Scope)**：不触碰音频流、不执行意图识别、不处理下游业务逻辑。

## 2. 物理与多云网络架构 (Network Architecture)

- **2.1 多云骨干网拓扑**：GCP 与 AWS 之间的企业级专用互联（Direct Connect / Cloud Interconnect）。
- **2.2 网络隔离策略**：VPC 划分子网、Internal ALB 准入控制、Private Subnet 计算节点隔离。
- **2.3 零信任安全加固**：
  - WSS (TLS 1.2+) 传输加密与本行内部证书管理。
  - 应用层 Bearer Token 鉴权。
  - 静态数据 (KMS) 落盘加密。

------

## 第二部分：详细设计说明书 (Detailed Design Specification)

*侧重于具体逻辑实现、并发控制与存储策略*

## 3. 应用逻辑与流式处理 (Core Application Logic)

- **3.1 纯异步技术栈选型**：FastAPI + Uvicorn + Asyncio 异步生态优势。
- **3.2 连接生命周期管理**：
  - `SESSION_ONGOING` 与 `SESSION_COMPLETE` 状态闭环。
  - 应用级 Ping/Pong (20s) 应对 ALB 60s 闲置超时。
- **3.3 分布式状态锁 (Sequence Guard)**：
  - 基于 Redis Lua 脚本的原子校验机制（首包校验、跳号拦截）。
  - **乐观数据锁**：摒弃物理连接锁，拥抱逻辑序列锁。
- **3.4 轻量级两阶段提交 (2PC)**：
  - 第一阶段：Redis 预检 (Check)。
  - 第二阶段：Kafka 异步写入 (Produce)。
  - 第三阶段：Redis 状态自增与 Ack 回复 (Commit)。

## 4. 存储与通信协议细节 (Storage & Protocol)

- **4.1 Kafka 生产端深度配置**：
  - **保序策略**：以 `sessionId` 为 Partition Key 的 FIFO 机制。
  - **可靠性配置**：`acks=all`, `enable_idempotence=True`, `max_in_flight=1`。
- **4.2 Redis 键空间管理**：
  - Hash 存储结构优化、1000 并发下的内存预估。

## 5. 高可用、容灾与自保机制 (Resilience & Self-Preservation)

- **5.1 动态内存回收**：15 分钟滚动续租 (Rolling TTL) 策略防止 OOM。
- **5.2 背压与熔断 (Backpressure)**：
  - Kafka 写入超时（2s）触发快速失败。
  - **断路器设计**：失败率 > 15% 时开启熔断，保护 AWS 基建。
- **5.3 容器漂移处理**：`SIGTERM` 信号捕获与 WebSocket 优雅关闭码 (1012/1001)。

------

## 第三部分：技术附件与规格 (Technical Annexes & Specs)

## 6. API 契约与错误处理定义 (Appendix A: API Contract)

- **6.1 正常响应码**：`TRANSCRIPT_ACK`。
- **6.2 异常响应码 (The "E" Series)**：
  - **E1006**：序列号非法（乱序或重放）。
  - **E1008**：内部组件故障（Redis 连接失败或应用异常）。
  - **E1009**：下游组件不可用（Kafka 写入超时或熔断开启）。
- **6.3 协议级 Close Code**：`1013 (Try Again Later)` 用于严重负载。

## 7. 基础设施配额与调优 (Appendix B: Infra & Capacity)

- **7.1 计算节点规格**：Fargate 1vCPU / 2GB 内存配额依据。
- **7.2 弹性伸缩逻辑**：放弃 CPU 指标，基于 `RequestCountPerTarget` 扩容。
- **7.3 Kafka Topic 规划**：Partition 数量定死（50-100）以确保 Hash 取模不变。

## 8. 运维与可观测性 (Appendix C: Observability)

- **8.1 结构化日志规范**：JSON 日志 Schema 定义。
- **8.2 监控黄金指标**：P99 Latency、Connection Count、E-Code Rate。

## 9. 技术辩护与常见问题 (Appendix D: FAQ)

- **9.1 选型辩护**：为什么 Python 单线程异步 + 多进程 Worker 优于传统多线程？
- **9.2 性能深度解析**：GIL 限制、上下文切换成本、C10K 问题处理。

------

## 文档现状核对：

1. **已完成**：白皮书核心（1-2章）、逻辑大框架（3-4章）、伸缩逻辑（5章）、性能 FAQ（9章）。