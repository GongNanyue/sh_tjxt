# tj-exam 考试服务详解

## 一、服务概述

tj-exam 是天机学堂的题目与考试管理服务，负责题目的CRUD操作、题目与业务（如课程小节、练习）的关联管理、题目分值统计等功能。该服务通过 Seata 实现与课程服务之间的分布式事务一致性。

**启动类：** `com.tianji.exam.ExamApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-common | 公共基础模块 |
| tj-api | 公共 API 模块 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| spring-boot-starter-web | Web 框架 |
| spring-boot-starter-data-redis | Redis |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| xxl-job-core | XXL-JOB 分布式任务调度 |
| spring-cloud-starter-alibaba-seata | Seata 分布式事务 |

## 三、核心业务逻辑

### 3.1 题目管理（QuestionController + QuestionServiceImpl）
- **新增/修改/删除题目**：CRUD操作，题目包含题干、题目类型、分类、难度等
- **分页查询题目**：管理端分页查询题目列表
- **查询题目详情**：获取题目完整信息含题目详情（选项、答案等）
- **批量查询题目**：根据ID列表查询题目信息（内部调用）
- **查询题目分值**：根据ID列表返回题目分值映射
- **统计教师出题数量**：按创建人分组统计
- **根据业务ID查询关联题目**：查询某个小节/练习下的所有题目
- **校验题目名称唯一性**

### 3.2 题目业务关联管理（QuestionBizController + QuestionBizServiceImpl）
- **批量保存题目业务关系**：将题目与业务（小节/练习）绑定
- **查询业务关联的题目ID列表**：支持单个和批量查询
- **统计业务下的题目分数总和**

## 四、数据模型

### 4.1 Question（题目表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 题目ID |
| name | String | 题干 |
| type | Integer | 题目类型：1单选、2多选、3不定向选择、4判断、5主观题 |
| cateId1 / cateId2 / cateId3 | Long | 一级/二级/三级课程分类ID |
| difficulty | Integer | 难度：1简单、2中等、3困难 |
| correctTimes | Integer | 回答正确次数 |
| answerTimes | Integer | 回答次数 |
| score | Integer | 分值 |

### 4.2 QuestionDetail（题目详情表）
存储题目的选项和答案信息。

### 4.3 QuestionBiz（题目业务关联表）
记录题目与业务实体（如课程小节）的多对多关联。

## 五、API接口清单

### 题目接口（/questions）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /questions | 新增题目 |
| PUT | /questions/{id} | 修改题目 |
| DELETE | /questions/{id} | 删除题目 |
| GET | /questions/page | 分页查询题目 |
| GET | /questions/{id} | 查询题目详情 |
| GET | /questions/list | 批量查询题目（内部调用） |
| GET | /questions/scores | 查询题目分值 |
| GET | /questions/numOfTeacher | 查询老师出题数量 |
| GET | /questions/listOfBiz | 查询业务关联的题目列表 |
| GET | /questions/checkName | 校验名称是否有效 |

### 题目业务关联接口（/question-biz）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /question-biz/list | 批量保存题目业务关系 |
| GET | /question-biz/biz/{id} | 查询业务关联的题目ID |
| GET | /question-biz/biz/list | 批量查询业务关联的题目ID |
| GET | /question-biz/scores | 查询业务下的题目分数总和 |

## 六、技术方案

1. **Seata分布式事务**：题目与课程小节的关联操作涉及课程服务和考试服务两个微服务，使用 Seata AT 模式保障数据一致性。
2. **分表设计**：题目基本信息（Question）和题目详情（QuestionDetail）分表存储，减少单表数据量。
3. **统计查询优化**：correctTimes 和 answerTimes 字段支持快速计算题目正确率。
