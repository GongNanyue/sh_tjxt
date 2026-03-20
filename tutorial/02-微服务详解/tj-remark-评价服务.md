# tj-remark 评价/点赞服务模块详解

## 一、服务概述

`tj-remark` 是天机学堂的评价/点赞服务模块，服务名称为 `remark-service`，运行在 **8091** 端口。该模块的核心功能是提供**通用化的点赞系统**，支持对不同业务类型（如问答回答 QA、笔记 NOTE 等）的点赞与取消点赞操作。

该模块体现了从"数据库直写方案"到"Redis 缓存 + 定时任务 + MQ 异步持久化"方案的架构演进，是一个典型的**高并发读写场景优化**案例。

### 包结构

```
com.tianji.remark
├── RemarkApplication.java          # 启动类，开启 @EnableScheduling
├── constants/
│   └── RedisConstants.java         # Redis Key 常量定义
├── controller/
│   └── LikedRecordController.java  # 点赞 REST 接口
├── domain/
│   ├── dto/
│   │   └── LikeRecordFormDTO.java  # 点赞请求 DTO
│   └── po/
│       └── LikedRecord.java        # 点赞记录持久化实体
├── mapper/
│   └── LikedRecordMapper.java      # MyBatis-Plus Mapper
├── service/
│   ├── ILikedRecordService.java    # 服务接口
│   └── impl/
│       ├── LikedRecordServiceImpl.java       # 基于数据库的实现（已废弃）
│       └── LikedRecordServiceRedisImpl.java  # 基于 Redis 的实现（当前生效）
└── task/
    └── LikedTimesCheckTask.java    # 定时任务：同步点赞数到下游
```

---

## 二、依赖分析

### Maven 依赖（pom.xml）

| 依赖 | 说明 |
|------|------|
| `tj-auth-resource-sdk` | 认证鉴权 SDK，提供登录拦截与用户上下文 |
| `tj-api` | 内部 API 模块，包含 `LikedTimesDTO` 等跨服务 DTO |
| `spring-boot-starter-web` | Web 框架 |
| `mybatis-plus-boot-starter` | MyBatis-Plus ORM 框架 |
| `mysql-connector-java` | MySQL 驱动 |
| `spring-boot-starter-data-redis` | Redis 客户端 |
| `spring-cloud-starter-alibaba-nacos-discovery` | Nacos 服务注册与发现 |
| `spring-cloud-starter-alibaba-nacos-config` | Nacos 配置中心 |
| `spring-boot-starter-amqp` | RabbitMQ 消息队列 |
| `spring-cloud-starter-loadbalancer` | 负载均衡 |

### 关键技术栈

- **缓存层**: Redis（Set + ZSet 数据结构）
- **消息队列**: RabbitMQ（Topic 交换机）
- **定时任务**: Spring `@Scheduled`
- **ORM**: MyBatis-Plus
- **注册中心/配置中心**: Nacos

---

## 三、配置文件分析

### bootstrap.yml 核心配置

```yaml
server:
  port: 8091

spring:
  application:
    name: remark-service
  cloud:
    nacos:
      config:
        shared-configs:
          - shared-spring.yaml   # 公共 Spring 配置
          - shared-redis.yaml    # 公共 Redis 配置
          - shared-mybatis.yaml  # 公共 MyBatis 配置
          - shared-logs.yaml     # 公共日志配置
          - shared-feign.yaml    # 公共 Feign 配置
          - shared-mq.yaml       # 公共 MQ 配置

tj:
  jdbc:
    database: tj_remark          # 独立数据库
  auth:
    resource:
      enable: true               # 开启登录拦截
  swagger:
    enable: true                 # 开启 Swagger 文档
    package-path: com.tianji.remark.controller
```

**要点**: 该服务使用 Nacos 管理 6 个共享配置文件，Redis、MQ、MyBatis 等中间件的连接信息均从 Nacos 集中下发，符合微服务配置中心化的最佳实践。

---

## 四、数据模型

### 数据库表：`liked_record`（数据库：`tj_remark`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | BIGINT, AUTO_INCREMENT | 主键 |
| `user_id` | BIGINT, NOT NULL | 点赞用户 ID |
| `biz_id` | BIGINT, NOT NULL | 被点赞的业务对象 ID |
| `biz_type` | VARCHAR(16), NOT NULL | 业务类型（如 QA、NOTE） |
| `create_time` | DATETIME | 创建时间（自动填充） |
| `update_time` | DATETIME | 更新时间（自动更新） |

**唯一索引**: `idx_biz_user(biz_id, user_id)` -- 保证同一用户对同一业务对象只能点赞一次。

### Java 实体类

- **LikedRecord (PO)**: 与数据库表一一对应，使用 `@TableName("liked_record")` 映射。
- **LikeRecordFormDTO**: 前端提交的点赞/取消点赞请求体，包含 `bizId`、`bizType`、`liked` 三个字段，均使用 `@NotNull` 校验。
- **LikedTimesDTO** (在 `tj-api` 模块): 跨服务传递的点赞数 DTO，包含 `bizId` 和 `likedTimes`，用于 MQ 消息体。

---

## 五、核心业务逻辑详解——点赞系统设计

### 5.1 架构演进：两套实现方案

本模块同时保留了两套 `ILikedRecordService` 的实现，体现了一次清晰的架构重构：

| 类 | 注解状态 | 方案 | 状态 |
|----|---------|------|------|
| `LikedRecordServiceImpl` | `@Service` 被注释 | 数据库直写 | **已废弃** |
| `LikedRecordServiceRedisImpl` | `@Service` 生效 | Redis 缓存 + 异步持久化 | **当前方案** |

这种设计允许通过切换 `@Service` 注解来回退方案，是一种简单有效的策略模式应用。

---

### 5.2 方案一（已废弃）：数据库直写方案

`LikedRecordServiceImpl` 的核心流程：

```
用户点赞/取消 → 直接操作 MySQL → 查询 COUNT → 发送 MQ 通知下游
```

**点赞操作 (`like`)**:
1. 查询 `liked_record` 表，判断当前用户是否已对该 `bizId` 点赞
2. 若已点赞，返回 `false`（幂等保护）
3. 若未点赞，插入一条新记录

**取消点赞 (`unlike`)**:
1. 根据 `userId + bizId` 删除记录
2. 返回删除是否成功

**统计与通知**:
1. 使用 `lambdaQuery().count()` 查询该 `bizId` 的总点赞数
2. 通过 `RabbitMqHelper.send()` 发送到 `like.record.topic` 交换机

**缺陷**: 每次点赞都产生数据库写操作 + COUNT 查询，高并发下数据库压力大。

---

### 5.3 方案二（当前方案）：Redis 缓存 + 异步持久化

`LikedRecordServiceRedisImpl` 的核心设计：

```
用户点赞/取消 → 操作 Redis Set → 写入 Redis ZSet → 定时任务读取 ZSet → 发送 MQ
```

#### 5.3.1 点赞操作 (`like`)

```java
Long result = redisTemplate.opsForSet().add(key, userId.toString());
return result != null && result > 0;
```

- 使用 **Redis Set** 存储每个业务对象的点赞用户集合
- Key: `likes:set:biz:{bizId}`，Value: 用户 ID 集合
- `SADD` 命令天然幂等：如果用户已在集合中，返回 0，操作被判定为失败（重复点赞），不会继续执行后续逻辑

#### 5.3.2 取消点赞 (`unlike`)

```java
Long result = redisTemplate.opsForSet().remove(key, userId.toString());
return result != null && result > 0;
```

- 使用 `SREM` 从 Set 中移除用户 ID
- 同样天然幂等：用户不在集合中时返回 0

#### 5.3.3 统计点赞数并缓存

点赞/取消成功后：

```java
// 获取 Set 的大小作为点赞总数
Long likedTimes = redisTemplate.opsForSet().size("likes:set:biz:" + bizId);

// 将点赞数写入 ZSet
redisTemplate.opsForZSet().add(
    "likes:times:type:" + bizType,   // Key
    bizId.toString(),                  // Member
    likedTimes                         // Score
);
```

- 使用 **Redis ZSet (Sorted Set)** 缓存变更过的业务对象的最新点赞数
- Key: `likes:times:type:{bizType}`，Member: bizId，Score: 点赞数
- ZSet 的好处：自动去重（同一 bizId 多次点赞只保留最新的 score）、支持按 score 排序弹出

#### 5.3.4 查询点赞状态 (`isBizLiked`)

```java
List<Object> objects = redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    StringRedisConnection src = (StringRedisConnection) connection;
    for (Long bizId : bizIds) {
        src.sIsMember("likes:set:biz:" + bizId, userId.toString());
    }
    return null;
});
```

- 使用 **Redis Pipeline（管道）** 批量查询多个业务对象的点赞状态
- 对每个 bizId 执行 `SISMEMBER` 判断当前用户是否在 Set 中
- Pipeline 将多个命令打包一次发送，大幅减少网络往返次数（RTT），提升批量查询性能

---

### 5.4 Redis 数据结构设计总结

| 用途 | 数据结构 | Key 格式 | Value/Member | Score |
|------|---------|----------|-------------|-------|
| 点赞记录 | **Set** | `likes:set:biz:{bizId}` | userId 集合 | - |
| 点赞数缓存 | **ZSet** | `likes:times:type:{bizType}` | bizId | 点赞总数 |

**设计亮点**:

1. **Set 结构的选择**: 天然支持去重（一个用户只能点赞一次）、`SADD/SREM` 原子操作保证并发安全、`SCARD` (size) O(1) 复杂度获取点赞数、`SISMEMBER` O(1) 复杂度判断是否点赞。

2. **ZSet 结构的选择**: 用作变更队列，同一 bizId 的多次变更自动合并（覆盖 score），`popMin` 可以批量弹出并移除，避免重复消费。

---

## 六、定时任务：点赞数异步同步

### `LikedTimesCheckTask`

```java
@Scheduled(fixedDelay = 20000)  // 每 20 秒执行一次
public void checkLikedTimes() {
    for (String bizType : BIZ_TYPES) {  // ["QA", "NOTE"]
        recordService.readLikedTimesAndSendMessage(bizType, MAX_BIZ_SIZE);
    }
}
```

### `readLikedTimesAndSendMessage` 方法

```java
// 1. 从 ZSet 中弹出最多 30 个元素（popMin 是原子操作）
Set<TypedTuple<String>> tuples = redisTemplate.opsForZSet().popMin(key, maxBizSize);

// 2. 转换为 LikedTimesDTO 列表
List<LikedTimesDTO> list = ...;

// 3. 批量发送 MQ 消息
mqHelper.send(LIKE_RECORD_EXCHANGE, "{bizType}.times.changed", list);
```

**执行流程**:

```
[定时任务 每20秒]
    |
    ├── 遍历业务类型 ["QA", "NOTE"]
    │     |
    │     ├── 从 ZSet `likes:times:type:QA` popMin 最多 30 条
    │     ├── 转换为 List<LikedTimesDTO>
    │     └── 发送到 MQ: exchange=like.record.topic, routingKey=QA.times.changed
    │
    └── 对 NOTE 类型重复上述操作
           routingKey=NOTE.times.changed
```

**设计要点**:

- `popMin` 是 Redis 原子操作，弹出并删除，不会重复消费
- 每次最多处理 30 条（`MAX_BIZ_SIZE = 30`），避免单次处理量过大
- `fixedDelay = 20000` 表示上次执行完毕后等待 20 秒再执行，避免任务堆叠
- 批量发送减少 MQ 消息数量

---

## 七、MQ 消息流转

### 消息发送（tj-remark 模块）

| 交换机 | 类型 | RoutingKey 模板 |
|--------|------|----------------|
| `like.record.topic` | Topic | `{bizType}.times.changed` |

实际 RoutingKey:
- `QA.times.changed` -- 问答回答的点赞数变更
- `NOTE.times.changed` -- 笔记的点赞数变更

消息体: `List<LikedTimesDTO>`（包含 bizId 和最新点赞数）

### 消息消费（tj-learning 模块）

在 `tj-learning` 模块的 `LikeTimesChangeListener` 中：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "qa.liked.times.queue", durable = "true"),
    exchange = @Exchange(name = "like.record.topic", type = ExchangeTypes.TOPIC),
    key = "QA.times.changed"
))
public void listenReplyLikedTimesChange(List<LikedTimesDTO> likedTimesDTOs) {
    // 批量更新 interaction_reply 表的 liked_times 字段
    replyService.updateBatchById(list);
}
```

**完整数据流**:

```
用户点赞
  → Redis Set (SADD) 记录谁点赞了
  → Redis ZSet 缓存最新点赞数
  → [每20秒] 定时任务从 ZSet popMin
  → RabbitMQ (like.record.topic)
  → tj-learning 消费者更新业务表的点赞数字段
```

---

## 八、并发处理与技术亮点

### 8.1 Redis 原子性保障并发安全

- **SADD/SREM 原子操作**: Redis 单线程模型保证同一用户的并发点赞请求只会有一次成功（SADD 返回 1），其余返回 0 后被 `if (!success) return` 拦截。无需加锁。
- **popMin 原子操作**: 定时任务读取 ZSet 时使用 `popMin`，弹出即删除，即使多个实例并发执行也不会重复消费。

### 8.2 Pipeline 批量查询优化

查询点赞状态时，使用 `executePipelined` 将多个 `SISMEMBER` 命令打包发送：
- 将 N 次网络往返降为 1 次
- 适用于列表页展示多个内容的点赞状态场景

### 8.3 通用化点赞系统设计

- **业务类型抽象**: 通过 `bizType` 字段（QA、NOTE 等）实现通用化，新业务类型只需在 `BIZ_TYPES` 列表和 MQ RoutingKey 中新增即可
- **解耦设计**: 点赞服务只负责记录和统计，不关心具体业务逻辑，通过 MQ 通知下游各业务服务自行更新

### 8.4 双层缓存 + 异步持久化

- **第一层（Set）**: 存储点赞明细，支持实时查询"是否点赞"
- **第二层（ZSet）**: 存储变更过的点赞计数，作为异步同步的缓冲队列
- **MQ 异步**: 最终通过 MQ 将点赞数同步到业务表，实现最终一致性

### 8.5 幂等性设计

- 数据库层面: `UNIQUE KEY idx_biz_user(biz_id, user_id)` 防止重复插入
- Redis 层面: `SADD` 对已存在元素返回 0，天然幂等
- 两层保障，无论使用哪套实现方案都能保证幂等

### 8.6 ZSet 的妙用——变更合并

当同一个 bizId 在 20 秒内被多次点赞/取消：
- 每次操作都会 `ZADD` 更新 ZSet 中该 bizId 的 score（最新点赞数）
- 定时任务只读取最终值，自动合并了多次变更
- 大幅减少了发送到 MQ 的消息量

---

## 九、架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        tj-remark 服务                               │
│                                                                     │
│  ┌──────────────┐    ┌───────────────────────────────────────────┐  │
│  │  Controller   │───▶│  LikedRecordServiceRedisImpl              │  │
│  │  POST /likes  │    │                                           │  │
│  │  GET  /likes  │    │  addLikeRecord()                          │  │
│  │       /list   │    │    ├── SADD/SREM (Redis Set)              │  │
│  └──────────────┘    │    ├── SCARD (获取点赞数)                  │  │
│                       │    └── ZADD (Redis ZSet 缓存变更)         │  │
│                       │                                           │  │
│                       │  isBizLiked()                             │  │
│                       │    └── Pipeline + SISMEMBER               │  │
│                       │                                           │  │
│                       │  readLikedTimesAndSendMessage()           │  │
│                       │    ├── ZPOPMIN (弹出 ZSet)                │  │
│                       │    └── 发送 MQ 消息                       │  │
│                       └───────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────┐                                           │
│  │ LikedTimesCheckTask   │  @Scheduled(fixedDelay=20s)              │
│  │ 遍历 [QA, NOTE]       │──▶ readLikedTimesAndSendMessage()       │
│  └──────────────────────┘                                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ MQ: like.record.topic
                               │ Key: {bizType}.times.changed
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  tj-learning 服务                                                    │
│  LikeTimesChangeListener                                             │
│  └── 消费消息 → 批量更新 interaction_reply.liked_times               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 十、Redis Key 规范

定义在 `RedisConstants` 接口中：

```java
public interface RedisConstants {
    String LIKES_BIZ_KEY_PREFIX  = "likes:set:biz:";    // + bizId
    String LIKES_TIMES_KEY_PREFIX = "likes:times:type:"; // + bizType
}
```

### Key 实例

| Key | 示例 | 类型 | 说明 |
|-----|------|------|------|
| `likes:set:biz:1001` | Set{1, 2, 5, 8} | SET | 业务ID=1001 的点赞用户集合 |
| `likes:set:biz:1002` | Set{3, 7} | SET | 业务ID=1002 的点赞用户集合 |
| `likes:times:type:QA` | {1001: 4, 1002: 2} | ZSET | QA 类型中有变更的 bizId 及其最新点赞数 |
| `likes:times:type:NOTE` | {2001: 10} | ZSET | NOTE 类型中有变更的 bizId 及其最新点赞数 |

---

## 十一、总结

`tj-remark` 模块虽然代码量精简，但架构设计考究，涵盖了高并发场景下点赞系统的典型解决方案：

1. **Redis Set** 实现高性能、天然幂等的点赞/取消操作
2. **Redis ZSet** 作为变更缓冲队列，自动合并高频更新
3. **定时任务** 批量消费 ZSet，控制下游处理频率
4. **RabbitMQ** 实现服务间解耦，通过 Topic 交换机路由到不同业务消费者
5. **Pipeline** 优化批量点赞状态查询
6. **通用化设计** 通过 `bizType` 支持多种业务类型扩展
