# tj-course 课程服务详解

## 一、服务概述

tj-course 是天机学堂的课程管理核心服务，负责课程全生命周期管理，包括课程基本信息、章节目录、师资关联、课程分类、题目关联等。该服务采用**草稿-正式双表设计**，课程编辑时操作草稿表，上架发布时将草稿同步到正式表，实现了课程编辑与线上数据隔离。

**启动类：** `com.tianji.course.CourseApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-api | 公共 API 模块，包含 Feign 客户端接口定义 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| spring-boot-starter-web | Web 框架 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库驱动 |
| spring-boot-starter-data-redis + redisson + commons-pool2 | Redis 缓存、分布式锁、连接池 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-boot-starter-amqp | RabbitMQ 消息队列（课程上下架事件通知） |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |
| spring-cloud-starter-alibaba-seata | Seata 分布式事务（课程上架涉及多服务数据一致性） |
| xxl-job-core | XXL-JOB 分布式任务调度 |

## 三、核心业务逻辑

### 3.1 课程管理（CourseController + CourseDraftService + CourseService）

- **保存课程基本信息**：校验课程名称唯一性 -> 写入草稿表（CourseDraft）
- **课程上架**：校验课程信息完整性 -> 将草稿表数据同步到正式表 -> 发送MQ消息通知搜索服务添加索引
- **课程下架**：更新正式表状态 -> 发送MQ消息通知搜索服务删除索引
- **课程删除**：先删除正式表数据，再删除草稿数据
- **课程搜索（管理端）**：根据状态区分查询源 - 待上架/已下架查草稿表，已上架/已完结查正式表
- **课程查询（内部调用）**：支持按ID查询课程完整信息（可选包含目录和教师信息）、按名称查询课程ID列表等

### 3.2 章节目录管理（CatalogueDraftService + CatalogueService）

- **保存章节目录**：写入课程目录草稿表（CourseCatalogueDraft），支持章（chapter）、节（section）、练习（practice）三级结构
- **保存视频信息**：将媒资ID与小节关联
- **保存题目关联**：将题目与小节/练习关联（写入 CourseCataSubjectDraft）
- **批量查询章节基础信息**：根据章节ID批量查询名称、序号等信息（内部调用）

### 3.3 课程分类管理（CategoryController + CategoryService）

- **分类树查询**：查询所有课程分类并组成树状结构（支持管理端和前台不同场景）
- **分类增删改**：新增/删除/更新课程分类，支持启用/停用
- **扁平化查询**：获取所有分类不分层（allOfOneLevel）

### 3.4 课程教师管理（CourseTeacherDraftService）

- **查询课程关联老师**：根据课程ID查询教师列表
- **保存教师信息**：将教师与课程关联

### 3.5 内部服务调用接口（CourseInfoController）

- 根据教师ID统计课程数和题目数
- 根据小节ID获取媒资ID和课程ID
- 根据媒资ID列表统计媒资被引用次数
- 课程上架时提供搜索信息
- 按三级分类ID查询分类名称映射

## 四、数据模型

### 4.1 Course（课程正式表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 课程ID |
| name | String | 课程名称 |
| courseType | Integer | 课程类型：1直播课、2录播课 |
| coverUrl | String | 封面链接 |
| firstCateId / secondCateId / thirdCateId | Long | 一级/二级/三级课程分类ID |
| free | Integer | 售卖方式：0付费、1免费 |
| price | Integer | 课程价格（分） |
| status | Integer | 课程状态：1待上架、2已上架、3下架、4已完结 |
| purchaseStartTime / purchaseEndTime | LocalDateTime | 购买有效期 |
| score | Integer | 课程评分（45代表4.5星） |
| mediaDuration | Integer | 课程总时长（秒） |
| validDuration | Integer | 课程有效期（月） |
| sectionNum | Integer | 课程总节数 |
| publishTimes | Integer | 发布次数 |

### 4.2 CourseCatalogue（课程目录表）

章节目录表，支持章/节/练习三级结构，正式表与草稿表分离。

### 4.3 Category（课程分类表）

树状结构的课程分类，支持多级分类。

### 4.4 CourseTeacher（课程教师表）

记录课程与教师的多对多关联关系。

### 4.5 Subject（题目表）

课程关联的题目信息，通过 CourseCataSubject 表建立题目与章节的关联。

## 五、API接口清单

### 课程管理接口（/courses）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /courses/baseInfo/{id} | 获取课程基础信息 |
| POST | /courses/baseInfo/save | 保存课程基本信息 |
| GET | /courses/catas/{id} | 获取课程的章节 |
| POST | /courses/catas/save/{id}/{step} | 保存章节 |
| POST | /courses/media/save/{id} | 保存课程视频 |
| POST | /courses/subjects/save/{id} | 保存小节中的题目 |
| GET | /courses/subjects/get/{id} | 获取小节中的题目 |
| GET | /courses/teachers/{id} | 查询课程老师信息 |
| POST | /courses/teachers/save | 保存老师信息 |
| POST | /courses/upShelf | 课程上架 |
| GET | /courses/checkBeforeUpShelf/{id} | 课程上架前校验 |
| POST | /courses/downShelf | 课程下架 |
| DELETE | /courses/delete/{id} | 课程删除 |
| GET | /courses/simpleInfo/list | 根据条件获取课程简要信息 |
| GET | /courses/catas/index/list/{id} | 查询章节序号列表 |
| GET | /courses/generator | 生成练习ID |
| GET | /courses/page | 管理端课程搜索 |
| GET | /courses/checkName | 校验课程名称是否存在 |
| GET | /courses/{id}/catalogs | 查询课程基本信息和目录 |

### 分类管理接口（/categorys）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /categorys/list | 查询课程分类信息（树结构） |
| GET | /categorys/{id} | 获取分类详情 |
| POST | /categorys/add | 新增课程分类 |
| DELETE | /categorys/{id} | 删除分类 |
| PUT | /categorys/disableOrEnable | 分类启用/停用 |
| PUT | /categorys/update | 更新课程分类 |
| GET | /categorys/all | 获取所有分类（简要，含层级关系） |
| GET | /categorys/getAllOfOneLevel | 获取所有分类（不分层） |

### 目录接口（/catalogues）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /catalogues/batchQuery | 批量查询章节目录基础信息 |
| GET | /catalogues/querySectionInfoById/{id} | 获取小节信息 |

### 内部调用接口（/course）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /course/infoByTeacherIds | 通过老师ID获取课程和出题数量 |
| GET | /course/section/{id} | 根据小节ID获取媒资ID和课程ID |
| GET | /course/media/useInfo | 根据媒资ID列表查询引用次数 |
| GET | /course/{id}/searchInfo | 课程上架时查询搜索信息 |
| GET | /course/{id} | 获取课程完整信息 |
| GET | /course/getCateNameMap | 按三级分类ID查询分类名称 |
| GET | /course/name | 按名称查询课程ID列表 |

## 六、技术方案

1. **草稿-正式双表设计**：CourseDraft/Course、CourseCatalogueDraft/CourseCatalogue 等均采用双表设计，编辑操作只影响草稿表，上架时将草稿同步到正式表，确保线上数据稳定。
2. **分布式事务（Seata）**：课程上架涉及正式表数据写入和搜索索引更新等跨服务操作，使用 Seata 保障数据一致性。
3. **MQ事件驱动**：课程上架/下架通过 RabbitMQ 发送事件，搜索服务监听事件进行 ES 索引的增删。
4. **Redis缓存**：课程分类等热点数据使用 Redis 缓存加速查询。
5. **XXL-JOB定时任务**：通过 CourseJobHandler 执行定时任务（如过期课程自动完结等）。
