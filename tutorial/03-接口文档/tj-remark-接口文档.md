# tj-remark 接口文档

> 服务名称：`remark-service` | 端口：`8091` | 基础路径：`/likes`

---

## 接口总览

| 序号 | 方法 | 路径 | 描述 | 认证 |
|------|------|------|------|------|
| 1 | POST | `/likes` | 点赞或取消点赞 | 需要登录 |
| 2 | GET | `/likes/list` | 批量查询点赞状态 | 需要登录 |

---

## 接口详情

### 1. 点赞或取消点赞

对指定业务对象执行点赞或取消点赞操作。

**基本信息**

| 项目 | 说明 |
|------|------|
| 接口路径 | `POST /likes` |
| 接口描述 | 点赞或取消点赞 |
| 认证方式 | 需要登录（通过 `tj-auth-resource-sdk` 拦截，从请求头获取用户信息） |
| Controller | `LikedRecordController.addLikeRecord()` |
| Service | `ILikedRecordService.addLikeRecord()` |

**请求参数**

Content-Type: `application/json`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `bizId` | Long | 是 | 点赞的业务对象 ID（如问答回答 ID、笔记 ID） |
| `bizType` | String | 是 | 业务类型，取值如 `"QA"`（问答）、`"NOTE"`（笔记） |
| `liked` | Boolean | 是 | `true` 表示点赞，`false` 表示取消点赞 |

**请求示例**

```json
{
  "bizId": 1001,
  "bizType": "QA",
  "liked": true
}
```

**响应**

无响应体（HTTP 200 表示成功）。

**业务逻辑说明**

1. 根据 `liked` 字段判断是点赞还是取消点赞
2. **点赞操作**: 使用 Redis `SADD` 将当前用户 ID 加入 `likes:set:biz:{bizId}` 集合。若用户已在集合中（重复点赞），`SADD` 返回 0，操作终止
3. **取消点赞操作**: 使用 Redis `SREM` 从集合中移除当前用户 ID。若用户不在集合中，`SREM` 返回 0，操作终止
4. 操作成功后，使用 `SCARD` 获取 Set 的大小作为最新点赞总数
5. 将 `{bizId: 点赞数}` 写入 Redis ZSet `likes:times:type:{bizType}`，供定时任务异步同步到下游

**幂等性**: 重复点赞或重复取消均安全，不会产生副作用。

**错误场景**

| HTTP 状态码 | 场景 |
|------------|------|
| 400 | 请求参数校验失败（`bizId`/`bizType`/`liked` 为空） |
| 401 | 未登录 |

---

### 2. 批量查询点赞状态

查询当前登录用户对一组业务对象的点赞状态，返回已点赞的业务 ID 集合。

**基本信息**

| 项目 | 说明 |
|------|------|
| 接口路径 | `GET /likes/list` |
| 接口描述 | 查询指定业务 ID 的点赞状态 |
| 认证方式 | 需要登录 |
| Controller | `LikedRecordController.isBizLiked()` |
| Service | `ILikedRecordService.isBizLiked()` |

**请求参数**

Query Parameters:

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `bizIds` | List\<Long\> | 是 | 要查询的业务对象 ID 列表，逗号分隔 |

**请求示例**

```
GET /likes/list?bizIds=1001,1002,1003,1004
```

**响应参数**

Content-Type: `application/json`

返回类型: `Set<Long>` -- 当前用户已点赞的业务 ID 集合。

**响应示例**

```json
[1001, 1003]
```

表示当前用户对 bizId=1001 和 bizId=1003 点过赞，对 1002 和 1004 未点赞。

**业务逻辑说明**

1. 从用户上下文 `UserContext.getUser()` 获取当前登录用户 ID
2. 使用 Redis **Pipeline** 批量执行 `SISMEMBER` 命令，对每个 bizId 检查用户是否在 `likes:set:biz:{bizId}` 集合中
3. 过滤出结果为 `true` 的 bizId，组装为 `Set<Long>` 返回

**性能说明**: 使用 Redis Pipeline 将 N 次网络请求合并为 1 次，即使 bizIds 列表较长也能保持高性能。

**错误场景**

| HTTP 状态码 | 场景 |
|------------|------|
| 400 | `bizIds` 参数缺失 |
| 401 | 未登录 |

---

## 数据模型

### LikeRecordFormDTO（请求体）

```java
@Data
@ApiModel(description = "点赞记录表单实体")
public class LikeRecordFormDTO {
    @NotNull(message = "业务id不能为空")
    @ApiModelProperty("点赞业务id")
    private Long bizId;

    @NotNull(message = "业务类型不能为空")
    @ApiModelProperty("点赞业务类型")
    private String bizType;

    @NotNull(message = "是否点赞不能为空")
    @ApiModelProperty("是否点赞，true：点赞；false：取消点赞")
    private Boolean liked;
}
```

### LikedTimesDTO（MQ 消息体，跨服务传递）

```java
@Data
@NoArgsConstructor
@AllArgsConstructor(staticName = "of")
public class LikedTimesDTO {
    private Long bizId;       // 业务对象 ID
    private Integer likedTimes; // 最新点赞总数
}
```

---

## 内部机制

### 定时任务：点赞数同步

该服务包含一个定时任务 `LikedTimesCheckTask`，不对外暴露 HTTP 接口，但作为点赞数据流转的核心环节：

| 项目 | 说明 |
|------|------|
| 类 | `LikedTimesCheckTask` |
| 调度方式 | `@Scheduled(fixedDelay = 20000)` -- 每 20 秒执行一次 |
| 业务类型 | `["QA", "NOTE"]` |
| 每次处理量 | 每种类型最多 30 条 |

**执行流程**:
1. 遍历业务类型列表 `["QA", "NOTE"]`
2. 对每种类型，从 Redis ZSet `likes:times:type:{bizType}` 中 `ZPOPMIN` 弹出最多 30 条记录
3. 将记录转换为 `List<LikedTimesDTO>`
4. 发送到 RabbitMQ 交换机 `like.record.topic`，RoutingKey 为 `{bizType}.times.changed`

### MQ 消息规范

| 项目 | 说明 |
|------|------|
| 交换机 | `like.record.topic` (Topic 类型) |
| RoutingKey | `QA.times.changed` 或 `NOTE.times.changed` |
| 消息体 | `List<LikedTimesDTO>` |
| 消费者 | `tj-learning` 模块 `LikeTimesChangeListener` |

---

## bizType 取值说明

| bizType 值 | 含义 | 对应的 RoutingKey | 消费者队列 |
|-----------|------|------------------|-----------|
| `QA` | 问答/回答 | `QA.times.changed` | `qa.liked.times.queue` |
| `NOTE` | 笔记 | `NOTE.times.changed` | （待消费者实现） |

---

## 认证说明

所有接口均需要用户登录。认证通过 `tj-auth-resource-sdk` 实现：
- 请求头中携带认证 Token
- SDK 自动拦截并解析用户信息
- 业务代码通过 `UserContext.getUser()` 获取当前用户 ID
- 配置项: `tj.auth.resource.enable: true`
