# tj-search 搜索服务详解

## 一、服务概述

tj-search 是天机学堂的搜索与推荐服务，基于 Elasticsearch 实现课程全文检索和个性化推荐。该服务同时提供用户兴趣管理功能，结合用户兴趣爱好实现个性化课程推荐，并通过 MQ 监听课程上下架和订单事件实现 ES 索引的实时同步。

**启动类：** `com.tianji.search.SearchApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-api | 公共 API 模块 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| spring-boot-starter-web | Web 框架 |
| spring-boot-starter-data-redis + redisson + commons-pool2 | Redis 缓存 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| elasticsearch-rest-high-level-client | Elasticsearch 高级客户端 |
| mybatis-plus-boot-starter + mysql | ORM 框架（用户兴趣表存储在MySQL） |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |

## 三、核心业务逻辑

### 3.1 课程搜索（SearchServiceImpl）

- **用户端课程搜索**：支持关键词搜索（match_phrase 短语匹配）+ 多维度过滤（分类、免费/付费、课程类型、时间范围）+ 排序 + 分页 + 高亮显示
- **课程ID按名称查询**：通过 match_phrase 查询匹配课程名称，返回课程ID列表（内部调用）

### 3.2 课程推荐（RecommendController + SearchServiceImpl）

- **精品好课推荐**：根据用户兴趣分类查询销量最高的 TopN 课程；未登录或无兴趣则查询全站热门
- **新课推荐**：按发布时间倒序查询 TopN 课程，结合用户兴趣过滤
- **精品公开课推荐**：查询免费课程中销量最高的 TopN
- **按分类查课程 Top10**：根据二级分类ID查询该分类下最新发布的10门课程

### 3.3 兴趣管理（InterestsController + InterestsServiceImpl）

- **保存兴趣爱好**：记录用户感兴趣的二级分类ID列表
- **查询我的兴趣爱好**：返回当前用户已选择的兴趣分类

### 3.4 MQ事件监听

- **CourseEventListener**：监听课程上架/下架事件，执行 ES 索引的新增/删除
- **OrderEventListener**：监听订单支付成功事件，更新 ES 中课程的销量（sold）字段

## 四、数据模型

### 4.1 Course（ES索引文档）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 课程ID |
| name | String | 课程名称（全文检索字段） |
| categoryIdLv1/Lv2/Lv3 | Long | 一级/二级/三级分类ID |
| free | Boolean | 是否免费 |
| type | Integer | 课程类型：1直播课、2录播课 |
| sold | Integer | 课程销量（报名人数） |
| price | Integer | 价格（分） |
| score | Integer | 课程评分 |
| teacher | Long | 老师ID |
| sections | Integer | 章节数量 |
| coverUrl | String | 课程封面 |
| publishTime | LocalDateTime | 发布时间 |

### 4.2 Interests（用户兴趣表，MySQL）

| 字段 | 类型 | 说明 |
|------|------|------|
| userId | Long | 用户ID |
| categoryId | Long | 感兴趣的二级分类ID |

## 五、API接口清单

### 课程搜索接口（/courses）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /courses/portal | 用户端课程搜索 |
| GET | /courses/name | 按名称查询课程ID列表（内部调用） |
| POST | /courses/up | 处理课程上架（手动触发） |
| POST | /courses/down | 处理课程下架（手动触发） |

### 推荐接口（/recommend）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /recommend/best | 精品好课推荐 |
| GET | /recommend/new | 新课推荐 |
| GET | /recommend/free | 精品公开课推荐 |

### 兴趣接口（/interests）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /interests | 保存兴趣爱好 |
| GET | /interests | 查询我的兴趣爱好 |
| GET | /interests/{id}/courses | 根据二级分类ID查询课程Top10 |

## 六、技术方案

1. **Elasticsearch 全文检索**：使用 RestHighLevelClient 构建复杂 DSL 查询，支持 BoolQuery 组合查询、短语匹配、高亮显示等。
2. **个性化推荐**：基于用户兴趣标签（二级分类ID）实现简单的个性化推荐，未登录或无兴趣时降级为全站热门推荐。
3. **MQ事件驱动索引同步**：通过 RabbitMQ 监听课程上下架事件和订单事件，实现 ES 索引的实时增删改，保证搜索数据与业务数据的最终一致性。
4. **教师信息聚合**：搜索结果中教师ID通过 UserClient 远程查询转换为教师姓名，实现数据聚合展示。
