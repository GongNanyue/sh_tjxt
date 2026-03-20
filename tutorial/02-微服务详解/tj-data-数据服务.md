# tj-data 数据服务详解

## 一、服务概述

tj-data 是天机学堂的数据统计与可视化服务，为管理后台提供工作台看板数据、今日运营数据、课程TOP10排行等数据展示功能。数据通过 Redis 缓存，支持数据的手动设置和自动统计。

**启动类：** `com.tianji.data.DataCenterApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| tj-api | 公共 API 模块 |
| spring-boot-starter-web | Web 框架 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-boot-starter-data-redis + redisson | Redis 缓存 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| xxl-job-core | XXL-JOB 分布式任务调度 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |

## 三、核心业务逻辑

### 3.1 看板数据（BoardController + BoardServiceImpl）
- **获取看板数据**：根据数据类型（types）参数查询不同维度的看板图表数据，返回 ECharts 格式的数据（含坐标轴和数据系列）
- **设置看板数据**：管理员手动设置线上运营看板数据

### 3.2 今日数据（TodayDataController + TodayDataServiceImpl）
- **获取今日数据**：查询当天的核心运营指标
- **设置今日数据**：管理员手动设置今日运营数据

### 3.3 Top10排行（Top10Controller + Top10ServiceImpl）
- **获取Top10数据**：查询课程排行榜等 Top10 数据
- **设置Top10数据**：管理员手动设置 Top10 数据

## 四、数据模型

### 4.1 TodayDataInfo

| 字段 | 类型 | 说明 |
|------|------|------|
| 核心运营指标数据 | 各类 | 如注册用户数、活跃用户数、订单数、收入等 |

### 4.2 CourseInfo
课程排行相关数据。

### 4.3 视图对象

- **EchartsVO**：ECharts 图表数据格式，包含 AxisVO（坐标轴）和 SerierVO（数据系列）
- **TodayDataVO**：今日数据展示对象
- **Top10DataVO**：Top10 排行数据展示对象

## 五、API接口清单

### 看板数据接口（/data/board）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /data/board | 获取看板数据 |
| PUT | /data/board/set | 设置看板数据 |

### 今日数据接口（/data/today）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /data/today | 获取今日数据 |
| PUT | /data/today/set | 设置线上数据 |

### Top10数据接口（/data/top10）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /data/top10 | 获取Top10数据 |
| PUT | /data/top10/set | 设置Top10数据 |

## 六、技术方案

1. **Redis数据缓存**：统计数据存储在 Redis 中，避免频繁查询数据库，使用 Redisson 保证分布式环境下的数据一致性。
2. **XXL-JOB定时统计**：通过定时任务定期从各业务库中聚合数据，更新 Redis 缓存。
3. **ECharts数据格式**：返回结果直接匹配 ECharts 图表组件的数据格式（AxisVO + SerierVO），前端可直接渲染。
4. **手动+自动双模式**：支持管理员手动设置数据（用于演示或调整）和定时任务自动统计数据。
