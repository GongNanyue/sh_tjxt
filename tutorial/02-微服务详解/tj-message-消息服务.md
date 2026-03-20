# tj-message 消息服务详解

## 一、服务概述

tj-message 是天机学堂的消息通知服务，采用**多模块架构**设计，负责短信发送、站内通知、消息模板管理、通知任务调度等功能。该服务集成阿里云短信服务，支持异步短信发送和定时通知任务。

| 子模块 | 说明 |
|--------|------|
| tj-message-api | Feign 客户端模块，供其他微服务调用（AsyncSmsClient、MessageClient） |
| tj-message-domain | 领域对象模块，定义 SmsInfoDTO、SmsTemplate 等共享类 |
| tj-message-service | 服务实现模块，包含所有业务逻辑和控制器 |

**启动类：** `com.tianji.message.MessageApplication`

## 二、依赖分析

### tj-message-service 依赖
| 依赖 | 说明 |
|------|------|
| spring-boot-starter-web | Web 框架 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-boot-starter-data-redis | Redis 缓存 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| knife4j-spring-boot-starter | Swagger 增强文档 |
| tj-api | 公共 API 模块 |
| tj-message-domain | 消息领域对象 |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| caffeine | 本地缓存 |
| xxl-job-core | XXL-JOB 分布式任务调度 |
| alibabacloud-dysmsapi20170525 | 阿里云短信SDK |

## 三、核心业务逻辑

### 3.1 短信发送（SmsController + SmsServiceImpl）
- **同步/异步发送短信**：接收 SmsInfoDTO（手机号、模板、参数），调用阿里云短信 API 发送
- **MQ异步处理**：通过 SmsMessageHandler 监听 MQ 消息实现短信的异步发送

### 3.2 短信模板管理（MessageTemplateController + MessageTemplateServiceImpl）
- **短信模板CRUD**：管理第三方短信平台的模板信息
- **分页查询/根据ID查询**

### 3.3 通知模板管理（NoticeTemplateController + NoticeTemplateServiceImpl）
- **通知模板CRUD**：管理系统通知的模板内容
- **分页查询/根据ID查询**

### 3.4 通知任务管理（NoticeTaskController + NoticeTaskServiceImpl）
- **新增/更新通知任务**：支持即时发送和定时发送
- **分页查询/根据ID查询通知任务**

### 3.5 用户收件箱（UserInboxController + UserInboxServiceImpl）
- **发送私信**：向指定用户发送站内私信
- **分页查询收件箱**：查询当前用户的消息列表

### 3.6 短信平台管理（SmsThirdPlatformController + SmsThirdPlatformServiceImpl）
- **短信平台CRUD**：管理第三方云通讯平台的配置信息

### 3.7 定时任务（NoticeJobHandler）
- 通过 XXL-JOB 定时扫描待发送的通知任务，到达发送时间后执行通知发送

## 四、数据模型

### 4.1 MessageTemplate（短信模板表）
第三方短信平台模板信息。

### 4.2 NoticeTemplate（通知模板表）
系统内部通知模板信息。

### 4.3 NoticeTask（通知任务表）
定时/延时通知任务记录。

### 4.4 UserInbox（用户收件箱表）
用户站内消息记录。

### 4.5 SmsThirdPlatform（第三方短信平台表）
第三方云通讯平台配置信息。

### 4.6 PublicNotice（公告表）
公共通知公告记录。

## 五、API接口清单

### 短信接口（/sms）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /sms/message | 发送短信 |

### 短信模板接口（/message-templates）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /message-templates | 新增短信模板 |
| PUT | /message-templates/{id} | 更新短信模板 |
| GET | /message-templates | 分页查询短信模板 |
| GET | /message-templates/{id} | 根据ID查询短信模板 |

### 通知模板接口（/notice-templates）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /notice-templates | 新增通知模板 |
| PUT | /notice-templates/{id} | 更新通知模板 |
| GET | /notice-templates | 分页查询通知模板 |
| GET | /notice-templates/{id} | 根据ID查询通知模板 |

### 通知任务接口（/notice-tasks）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /notice-tasks | 新增通知任务 |
| PUT | /notice-tasks/{id} | 更新通知任务 |
| GET | /notice-tasks | 分页查询通知任务 |
| GET | /notice-tasks/{id} | 根据ID查询通知任务 |

### 用户收件箱接口（/inboxes）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /inboxes | 发送私信 |
| GET | /inboxes | 分页查询收件箱 |

### 短信平台接口（/sms-platforms）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /sms-platforms | 新增短信平台 |
| PUT | /sms-platforms/{id} | 更新短信平台 |
| GET | /sms-platforms | 分页查询短信平台 |
| GET | /sms-platforms/{id} | 根据ID查询短信平台 |

## 六、技术方案

1. **多模块设计**：api（Feign客户端）+ domain（领域对象）+ service（服务实现）三层分离，其他微服务只需引入 api 和 domain 模块即可调用。
2. **阿里云短信集成**：使用阿里云 Dysms SDK 实现短信发送。
3. **MQ异步发送**：短信发送通过 RabbitMQ 异步处理，避免阻塞业务主流程。
4. **Caffeine本地缓存**：使用 Caffeine 对短信模板等热点数据进行本地缓存，减少数据库查询。
5. **XXL-JOB定时任务**：定时扫描待发送的通知任务，支持定时/延时发送功能。
