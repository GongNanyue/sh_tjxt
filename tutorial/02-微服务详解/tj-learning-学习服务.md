# tj-learning 学习服务 - 微服务详解

## 一、服务概述

### 1.1 基本信息

| 属性 | 值 |
|------|-----|
| 服务名称 | `learning-service` |
| 端口 | `8090` |
| ArtifactId | `tj-learning` |
| 主类 | `com.tianji.learning.LearningApplication` |
| 数据库 | `tj_learning` |
| JDK版本 | 11 |

### 1.2 服务定位

学习服务是天机学堂平台的核心业务服务之一，承担了用户学习全流程的管理职责。主要功能覆盖以下六大领域：

1. **我的课表管理** -- 用户购买课程后的课表维护、学习计划制定与跟踪
2. **学习记录跟踪** -- 视频播放进度记录、考试记录、小节完成状态管理
3. **互动问答系统** -- 课程学习中的提问、回答、评论功能
4. **学习笔记系统** -- 笔记的新增、采集、管理
5. **积分与排行榜** -- 签到积分、学习积分、积分排行榜（学霸天梯榜）
6. **签到系统** -- 每日签到、连续签到奖励

---

## 二、依赖分析

### 2.1 核心依赖列表

| 依赖 | 用途 |
|------|------|
| `tj-auth-resource-sdk` | 认证鉴权资源SDK，实现接口权限控制 |
| `tj-api` | 内部API模块，包含Feign客户端接口和公共DTO |
| `spring-boot-starter-web` | Web服务基础框架 |
| `mybatis-plus-boot-starter` | MyBatis-Plus ORM框架 |
| `mysql-connector-java` | MySQL数据库驱动 |
| `spring-boot-starter-data-redis` | Redis缓存客户端 |
| `redisson` | Redis分布式锁和高级数据结构 |
| `spring-cloud-starter-alibaba-nacos-discovery` | Nacos服务注册与发现 |
| `spring-cloud-starter-alibaba-nacos-config` | Nacos配置中心 |
| `caffeine` | 本地缓存（用于高频访问的热点数据） |
| `spring-boot-starter-amqp` | RabbitMQ消息队列 |
| `xxl-job-core` | XXL-JOB分布式定时任务 |
| `spring-cloud-starter-loadbalancer` | 服务负载均衡 |

### 2.2 Nacos共享配置

服务通过Nacos加载以下共享配置文件：

- `shared-spring.yaml` -- Spring公共配置
- `shared-redis.yaml` -- Redis连接配置
- `shared-mybatis.yaml` -- MyBatis配置
- `shared-logs.yaml` -- 日志配置
- `shared-feign.yaml` -- Feign远程调用配置
- `shared-mq.yaml` -- RabbitMQ配置
- `shared-xxljob.yaml` -- XXL-JOB定时任务配置

### 2.3 远程服务调用（Feign Client）

| Feign客户端 | 调用目标服务 | 用途 |
|-------------|-------------|------|
| `CourseClient` | 课程服务 | 查询课程详情、简要信息 |
| `CatalogueClient` | 课程服务 | 查询章节目录信息 |
| `UserClient` | 用户服务 | 查询用户基本信息 |
| `SearchClient` | 搜索服务 | 根据课程名称搜索课程ID |
| `RemarkClient` | 评价服务 | 查询用户点赞状态 |
| `CategoryCache` | 分类缓存 | 查询课程分类名称 |

---

## 三、数据模型

### 3.1 核心实体（PO）一览

#### 3.1.1 LearningLesson（学生课程表）

表名：`learning_lesson`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键，雪花算法生成 |
| userId | Long | 学员ID |
| courseId | Long | 课程ID |
| status | LessonStatus | 课程状态：0-未学习，1-学习中，2-已学完，3-已失效 |
| weekFreq | Integer | 每周学习频率 |
| planStatus | PlanStatus | 学习计划状态：0-没有计划，1-计划进行中 |
| learnedSections | Integer | 已学习小节数量 |
| latestSectionId | Long | 最近一次学习的小节ID |
| latestLearnTime | LocalDateTime | 最近一次学习的时间 |
| createTime | LocalDateTime | 创建时间 |
| expireTime | LocalDateTime | 过期时间 |

#### 3.1.2 LearningRecord（学习记录表）

表名：`learning_record`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键，雪花算法生成 |
| lessonId | Long | 对应课表ID |
| sectionId | Long | 对应小节ID |
| userId | Long | 用户ID |
| moment | Integer | 视频当前观看时间点（秒） |
| finished | Boolean | 是否完成学习 |
| createTime | LocalDateTime | 第一次观看时间 |
| finishTime | LocalDateTime | 完成学习的时间 |

#### 3.1.3 InteractionQuestion（互动问题表）

表名：`interaction_question`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| title | String | 问题标题 |
| description | String | 问题描述 |
| courseId | Long | 所属课程ID |
| chapterId | Long | 所属章ID |
| sectionId | Long | 所属节ID |
| userId | Long | 提问学员ID |
| latestAnswerId | Long | 最新回答ID |
| answerTimes | Integer | 回答数量 |
| anonymity | Boolean | 是否匿名 |
| hidden | Boolean | 是否隐藏 |
| status | QuestionStatus | 管理端状态：0-未查看，1-已查看 |

#### 3.1.4 InteractionReply（互动回答/评论表）

表名：`interaction_reply`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| questionId | Long | 问题ID |
| answerId | Long | 上级回答ID（为0或null表示是回答，非0表示是评论） |
| userId | Long | 回答者ID |
| content | String | 回答内容 |
| targetUserId | Long | 回复目标用户ID |
| targetReplyId | Long | 回复目标回复ID |
| replyTimes | Integer | 评论数量 |
| likedTimes | Integer | 点赞数量 |
| hidden | Boolean | 是否隐藏 |
| anonymity | Boolean | 是否匿名 |

#### 3.1.5 Note（笔记表）

表名：`note`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| userId | Long | 用户ID |
| courseId | Long | 课程ID |
| chapterId | Long | 章ID |
| sectionId | Long | 小节ID |
| noteMoment | Integer | 记录笔记时的视频播放时间（秒） |
| content | String | 笔记内容 |
| isPrivate | Boolean | 是否隐私笔记 |
| hidden | Boolean | 是否被管理端隐藏 |
| authorId | Long | 笔记原始作者ID |
| gatheredNoteId | Long | 被采集的原始笔记ID |
| isGathered | Boolean | 是否是采集笔记 |

#### 3.1.6 PointsBoard（积分排行榜）

表名：`points_board`（动态表名，按赛季分表，如 `points_board_1`）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 榜单ID（存储排名值） |
| userId | Long | 学生ID |
| points | Integer | 积分值 |
| rank | Integer | 名次（仅内存使用，不持久化） |
| season | Integer | 赛季ID（仅内存使用，不持久化） |

#### 3.1.7 PointsBoardSeason（赛季信息表）

表名：`points_board_season`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 自增主键，赛季标识 |
| name | String | 赛季名称（如"第1赛季"） |
| beginTime | LocalDate | 赛季开始时间 |
| endTime | LocalDate | 赛季结束时间 |

#### 3.1.8 PointsRecord（积分记录表）

表名：`points_record`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 自增主键 |
| userId | Long | 用户ID |
| type | PointsRecordType | 积分方式：1-课程学习，2-签到，3-问答，4-笔记，5-评价 |
| points | Integer | 积分值 |
| createTime | LocalDateTime | 创建时间 |

### 3.2 枚举定义

| 枚举 | 值 | 说明 |
|------|-----|------|
| **LessonStatus** | 0-NOT_BEGIN, 1-LEARNING, 2-FINISHED, 3-EXPIRED | 课程学习状态 |
| **PlanStatus** | 0-NO_PLAN, 1-PLAN_RUNNING | 学习计划状态 |
| **SectionType** | 1-VIDEO, 2-EXAM | 小节类型 |
| **QuestionStatus** | 0-UN_CHECK, 1-CHECKED | 问题状态（管理端） |
| **PointsRecordType** | 1-LEARNING(上限50), 2-SIGN(无上限), 3-QA(上限20), 4-NOTE(上限20), 5-COMMENT(无上限) | 积分类型及每日上限 |

---

## 四、核心业务逻辑详解

### 4.1 课表管理 (LearningLessonServiceImpl)

#### 4.1.1 自动添加课程到课表

**触发方式**：通过MQ监听订单支付成功消息（`LessonChangeListener`）

**业务流程**：
1. 监听订单交换机 `ORDER_EXCHANGE` 的支付成功消息（RoutingKey: `order.pay`）
2. 接收 `OrderBasicDTO`，包含 userId 和 courseIds
3. 调用课程服务获取课程有效期
4. 根据课程有效期计算过期时间（当前时间 + validDuration 月）
5. 批量插入 `learning_lesson` 表

**退款处理**：监听退款消息（RoutingKey: `order.refund`），直接从课表中删除对应课程。

#### 4.1.2 分页查询我的课表

- 根据当前用户ID分页查询课表数据
- 按最近学习时间倒序排列
- 通过Feign调用课程服务，填充课程名称、封面、章节数

#### 4.1.3 查询正在学习的课程

- 查询当前用户状态为"学习中"的课程
- 按最近学习时间倒序，取第一条
- 同时统计用户总报名课程数
- 通过Feign查询最近学习小节的名称和序号

#### 4.1.4 学习计划

- **创建学习计划**：设置每周学习频率（weekFreq），将计划状态改为"计划进行中"
- **查询学习计划**：统计本周已学习小节数、本周计划总数，分页展示每门课的学习计划完成情况

### 4.2 学习记录管理 (LearningRecordServiceImpl)

#### 4.2.1 视频播放进度记录（核心功能）

这是系统中最关键的功能之一，采用了**延迟任务 + Redis缓存**的方案来优化高频写入。

**业务流程**：

```
用户播放视频 --> 前端定时提交播放进度
                    |
                    v
            判断小节类型（视频/考试）
                    |
        +-----------+-----------+
        |                       |
    视频类型                  考试类型
        |                       |
   查询旧记录               直接新增记录
   (先查缓存)               标记为已完成
        |
    +---+---+
    |       |
  不存在   存在
    |       |
  新增     判断是否首次完成
  记录     (moment*2 >= duration)
            |
     +------+------+
     |              |
   未完成        首次完成
     |              |
  写入Redis       更新DB
  +延迟任务       标记finished=true
  (20秒后        清理缓存
   持久化)
```

**关键设计**：

1. **Redis缓存**：使用 `learning:record:{lessonId}` 作为Hash Key，sectionId为Hash Field，存储学习记录的 id/moment/finished。缓存过期时间1分钟。
2. **延迟任务队列**：使用JDK内置 `DelayQueue`，延迟20秒后执行持久化。当用户持续观看视频时，频繁提交的进度只写入Redis缓存，不直接写数据库，大幅降低数据库压力。
3. **完成判定**：当视频观看时长 * 2 >= 视频总时长时（即观看超过50%），判定为完成该小节。
4. **课表更新**：小节完成后，更新课表的 `learned_sections`（+1），如果所有小节学完则将课程状态改为"已学完"。

#### 4.2.2 延迟任务处理器 (LearningRecordDelayTaskHandler)

| 方法 | 说明 |
|------|------|
| `addLearningRecordTask` | 将学习记录写入Redis缓存，并提交20秒延迟任务 |
| `handleDelayTask` | 异步循环消费延迟队列，比较moment值判断是否需要持久化 |
| `writeRecordCache` | 将学习记录写入Redis Hash |
| `readRecordCache` | 从Redis Hash读取学习记录缓存 |
| `cleanRecordCache` | 当小节完成时，清理Redis中的缓存 |

**防重复写入机制**：延迟任务到期后，会比较任务中的moment值与Redis缓存中的最新moment值。如果不一致，说明用户在这20秒内又提交了新进度，当前任务放弃，由后续任务处理。

### 4.3 互动问答系统 (InteractionQuestionServiceImpl / InteractionReplyServiceImpl)

#### 4.3.1 问题管理

- **用户端**：新增问题（关联课程/章/节）、修改/删除自己的问题、分页查询（支持按课程/小节/仅我的筛选）
- **管理端**：分页查询（支持按课程名称/状态/时间筛选）、隐藏/显示问题、查看问题详情（含教师信息、分类信息）
- **删除级联**：删除问题时同步删除其下所有回答和评论

#### 4.3.2 回答与评论

- **数据模型**：回答和评论共用 `interaction_reply` 表，通过 `answerId` 区分：
  - `answerId = 0 或 null`：表示回答（直接回复问题）
  - `answerId != 0`：表示评论（回复某个回答）
- **新增回答**：同时更新问题的回答数量和最新回答ID
- **新增评论**：同时更新上级回答的评论数量
- **积分奖励**：学生回答问题后通过MQ发送积分消息（5积分/次）
- **排序规则**：先按点赞数倒序，再按创建时间正序
- **匿名处理**：匿名提问/回答时不返回用户信息
- **点赞集成**：通过 `RemarkClient` 查询当前用户对回答的点赞状态

#### 4.3.3 点赞数同步 (LikeTimesChangeListener)

- 监听点赞服务的 `LIKE_RECORD_EXCHANGE` 交换机
- 接收 `LikedTimesDTO` 列表，批量更新回答/评论的点赞数

### 4.4 笔记系统 (NoteServiceImpl)

#### 4.4.1 核心功能

- **新增笔记**：记录笔记内容、视频播放时间点、所属课程/章/节，支持私密笔记。新增后通过MQ发送积分消息
- **采集笔记**：复制他人公开笔记到自己的笔记本，标记为采集笔记，强制设为私密。采集后通过MQ发送积分消息
- **取消采集**：根据 `gatheredNoteId` 删除采集记录
- **更新笔记**：只能更新自己的笔记，采集笔记不能设置为公开
- **删除笔记**：只能删除自己的笔记

#### 4.4.2 管理端功能

- 分页查询所有笔记（支持按课程名称、隐藏状态、时间筛选）
- 查看笔记详情（含课程分类、章节信息、作者手机号、采集者列表）
- 隐藏/显示笔记

### 4.5 签到系统 (SignRecordServiceImpl)

#### 4.5.1 签到实现（基于Redis BitMap）

**存储方案**：使用Redis的BitMap数据结构，Key格式为 `sign:uid:{userId}{yyyyMM}`

**签到流程**：
1. 计算当月第几天（offset = dayOfMonth - 1）
2. 使用 `setBit(key, offset, true)` 写入签到记录
3. 如果bit位已经是1，说明重复签到，抛出异常
4. 计算连续签到天数（从今天往前数，遇到0则停止）
5. 根据连续签到天数计算奖励积分：
   - 连续7天：+10分
   - 连续14天：+20分
   - 连续28天：+40分
6. 通过MQ发送签到积分消息

**优势**：每个用户每月仅占用约4字节Redis内存（31个bit位），极其高效。

#### 4.5.2 查询签到记录

使用 `bitField` 命令一次性读取当月所有签到数据，通过位运算解析为Byte数组返回前端（1=已签，0=未签）。

### 4.6 积分系统 (PointsRecordServiceImpl)

#### 4.6.1 积分来源与上限

| 积分方式 | 每次积分 | 每日上限 |
|----------|---------|---------|
| 课程学习 | 10 | 50 |
| 每日签到 | 1 + 连续奖励 | 无上限 |
| 课程问答 | 5 | 20 |
| 课程笔记（新增） | 3 | 20 |
| 笔记被采集 | 2 | 20 |
| 课程评价 | -- | 无上限 |

#### 4.6.2 积分上限控制

对于有上限的积分类型，每次新增积分前：
1. 查询该用户今日已获取的该类型积分总数
2. 如果已达上限，直接返回不再累加
3. 如果累加后超过上限，只给予剩余可获得的积分

#### 4.6.3 积分榜单（Redis ZSet）

- 使用Redis的 `ZSet` 存储当前赛季排行榜
- Key格式：`boards:{yyyyMM}`
- 每次新增积分后，使用 `incrementScore` 原子递增用户积分
- 查询排名使用 `reverseRank`，查询积分使用 `score`
- 分页查询使用 `reverseRangeWithScores`

### 4.7 排行榜系统 (PointsBoardServiceImpl)

#### 4.7.1 当前赛季 vs 历史赛季

| 场景 | 数据源 | 说明 |
|------|--------|------|
| 当前赛季 | Redis ZSet | 实时数据，高性能 |
| 历史赛季 | MySQL分表 | 持久化数据，按赛季分表 |

#### 4.7.2 动态分表

通过 `TableInfoContext`（ThreadLocal）在运行时动态指定查询的表名，格式为 `points_board_{season}`。这种方式避免了使用分库分表中间件带来的额外复杂度。

---

## 五、MQ消息机制

### 5.1 消费者列表

| 监听器类 | 队列 | 交换机 | RoutingKey | 触发动作 |
|----------|------|--------|-----------|---------|
| `LessonChangeListener` | `learning.lesson.pay.queue` | `ORDER_EXCHANGE` | `order.pay` | 订单支付成功，添加课程到课表 |
| `LessonChangeListener` | `learning.lesson.refund.queue` | `ORDER_EXCHANGE` | `order.refund` | 订单退款，从课表删除课程 |
| `LearningPointsListener` | `qa.points.queue` | `LEARNING_EXCHANGE` | `write.reply` | 问答积分 +5 |
| `LearningPointsListener` | `sign.points.queue` | `LEARNING_EXCHANGE` | `sign.in` | 签到积分 |
| `LearningPointsListener` | `learning.points.queue` | `LEARNING_EXCHANGE` | `learn.section` | 学习积分 +10 |
| `LearningPointsListener` | `note.new.points.queue` | `LEARNING_EXCHANGE` | `write.note` | 写笔记积分 +3 |
| `LearningPointsListener` | `note.gathered.points.queue` | `LEARNING_EXCHANGE` | `note.gathered` | 笔记被采集积分 +2 |
| `LikeTimesChangeListener` | `qa.liked.times.queue` | `LIKE_RECORD_EXCHANGE` | `QA_LIKED_TIMES_KEY` | 同步回答点赞数 |

### 5.2 生产者

| 场景 | 交换机 | RoutingKey | 消息体 |
|------|--------|-----------|--------|
| 签到成功 | `LEARNING_EXCHANGE` | `sign.in` | `SignInMessage(userId, points)` |
| 回答问题 | `LEARNING_EXCHANGE` | `write.reply` | `userId (Long)` |
| 新增笔记 | `LEARNING_EXCHANGE` | `write.note` | `userId (Long)` |
| 采集笔记 | `LEARNING_EXCHANGE` | `note.gathered` | `userId (Long)` |

---

## 六、定时任务（XXL-JOB）

### 6.1 任务列表 (PointsBoardPersistentHandler)

| 任务名 | 方法 | 说明 |
|--------|------|------|
| `createTableJob` | `createPointsBoardTableOfLastSeason` | 在每月初创建上个月赛季的积分排行榜分表 |
| `savePointsBoard2DB` | `savePointsBoard2DB` | 将Redis中上月的排行榜数据持久化到MySQL分表中，支持分片执行 |
| `clearPointsBoardFromRedis` | `clearPointsBoardFromRedis` | 清理Redis中上月的排行榜数据 |

### 6.2 排行榜持久化流程

```
每月初定时任务执行:
1. createTableJob     --> 创建 points_board_{season} 分表
2. savePointsBoard2DB --> 从Redis ZSet分页读取数据，写入MySQL分表（支持分片并行）
3. clearPointsBoardFromRedis --> 删除Redis中上月的ZSet Key
```

支持XXL-JOB的分片广播模式，多个服务实例可并行处理不同分页的数据。

---

## 七、Redis使用汇总

| 用途 | Key格式 | 数据结构 | 说明 |
|------|---------|---------|------|
| 签到记录 | `sign:uid:{userId}{yyyyMM}` | BitMap | 每月一个Key，每天一个bit位 |
| 积分排行榜 | `boards:{yyyyMM}` | ZSet | member=userId，score=积分 |
| 学习记录缓存 | `learning:record:{lessonId}` | Hash | field=sectionId，value=记录JSON |

---

## 八、技术方案总结

### 8.1 高频写入优化（学习记录）

**问题**：视频播放时前端频繁提交播放进度（可能每隔几秒一次），如果直接写数据库会造成巨大压力。

**解决方案**：Redis缓存 + JDK DelayQueue

- 首次提交写入数据库，后续提交只更新Redis缓存
- 通过延迟队列合并写入，每20秒最多持久化一次
- 延迟任务到期时比较moment值，避免重复写入
- 小节完成时直接写入数据库并清理缓存

### 8.2 签到方案（Redis BitMap）

- 利用Redis BitMap的位操作，每个用户每月仅需约4字节存储
- 连续签到天数通过位运算高效计算
- 签到与积分解耦，通过MQ异步发放积分

### 8.3 排行榜方案（Redis ZSet + MySQL分表）

- 当前赛季使用Redis ZSet实时排名，O(logN)时间复杂度
- 每月底通过定时任务将排行榜持久化到MySQL分表
- 历史赛季查询从MySQL分表中读取
- 通过ThreadLocal动态切换表名，轻量级实现分表

### 8.4 积分上限控制

- 每种积分类型可配置每日上限（通过枚举 `maxPoints` 字段）
- 每次积分新增前查询今日已得积分，确保不超过上限
- 签到积分无上限，通过连续签到天数给予额外奖励

### 8.5 事件驱动架构

积分系统采用事件驱动模式，各业务操作（签到、回答、写笔记等）通过MQ发送消息，积分服务统一消费处理，实现了业务逻辑与积分逻辑的解耦。
