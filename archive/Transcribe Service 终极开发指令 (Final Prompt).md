**角色设定**：你是一位精通 Python 异步编程和分布式系统架构的高级资深工程师。

**任务目标**：请基于以下【核心事实清单】和【逻辑封装类】，补全 **Transcribe Service** 的完整生产代码实现。

## 1. 核心事实清单 (Grounding Truths)

- **技术栈**：Python 3.10+ / FastAPI / redis.asyncio / aiokafka。
- **并发模型**：单线程异步协程（Asyncio）。单 vCPU 对应单进程 Worker。
- **状态管理**：使用 Redis Lua 脚本实现“两阶段提交”（预检 -> 写入 Kafka -> 提交状态）。
- **Kafka 策略**：以 sessionId 为 Partition Key 确保严格保序；acks=all, enable_idempotence=True。
- **错误码规范**：
- E1006: 序列号非法（Lua 返回 -1/-2）。
- E1008: 内部故障（Redis/应用异常）。
- E1009: 下游不可用（Kafka 超时 2s 或断路器开启）。
- **保活机制**：WebSocket 开启 20s/次 Ping/Pong。

## 2. 核心逻辑封装 (核心参考代码)

```python
import asyncio
import logging
from redis.asyncio import Redis, ConnectionPool
from typing import Optional, Tuple

# 结构化日志配置 (推荐使用 structlog)
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
        # 使用 EVALSHA 减少网络负载
        result = await self.redis.evalsha(self._pre_check_sha, 1, key, incoming_seq)
        
        if result == 1:
            return True, None
        elif result == 0:
            return False, "IDEMPOTENT" # 幂等重复，不报错但也不写Kafka
        elif result == -1 or result == -2:
            return False, "E1006" # 序列号非法
            
    except Exception as e:
        logger.error(f"Redis pre-check failed for {session_id}: {e}")
        return False, "E1008" # 内部组件故障 (Redis连接问题)

async def commit_sequence(self, session_id: str):
    """
    第二阶段：提交状态
    只有在 Kafka 返回 Ack 后调用
    """
    key = f"ts:sess:{session_id}"
    try:
        await self.redis.evalsha(self._commit_sha, 1, key, self.ttl)
    except Exception as e:
        # 这里的失败需要记录 Critical Log，因为 Kafka 已经写了，但状态没更新，会导致重试
        logger.critical(f"FATAL: Kafka synced but Redis state failed to advance for {session_id}: {e}")
        # 注意：此处不返回 E1008 给 Vendor，因为数据其实已经到 Kafka 了
```

## 3. 代码实现要求 (Detailed Requirements)

1. **WebSocket 路由**：实现 /ws/transcribe 接口。需包含握手阶段的 Authorization: Bearer 校验。
2. **消息循环**：

- 收到消息后先调用 validate_sequence。
- 若为 IDEMPOTENT，记录 Debug 日志并回 Ack，不写 Kafka。
- 若验证通过，异步写入 Kafka 并设置 2s 超时。
- Kafka 成功后调用 commit_sequence。
3. **异常处理中间件**：
- 捕获 KafkaTimeoutError 或断路器异常，返回 E1009。
- 捕获所有未定义异常，记录 Error 日志并返回 E1008。
4. **优雅停机**：监听 SIGTERM 信号，确保在关闭连接前，Kafka 生产缓冲区的所有消息已完成 flush()。
5. **结构化日志**：使用 logging 或 structlog 输出 JSON 格式日志，每行必须携带 sessionId 和 seq。
6. **Kafka Producer 的初始化**：确保它是在 FastAPI 的 `lifespan`（或 `on_event("startup")`）里初始化的，而不是每个请求都初始化一遍。
7. **Lua 脚本的加载**：确保它只在启动时 `script_load` 一次。
8. **循环引用**：确保 WebSocket 的 `receive` 循环能正确感知到客户端断开，并释放 Redis 连接。