# tj-pay 支付服务详解

## 一、服务概述

tj-pay 是天机学堂的底层支付服务，对接第三方支付平台（支付宝、微信支付），提供统一的支付申请、支付结果查询、退款申请、退款结果查询等能力。该服务采用**多模块架构**设计，通过 MQ 通知业务服务支付和退款的结果变更。

| 子模块 | 说明 |
|--------|------|
| tj-pay-api | Feign 客户端模块（PayClient），供交易服务等上层服务调用 |
| tj-pay-domain | 领域对象模块，定义 PayApplyDTO、PayResultDTO、RefundApplyDTO 等 |
| tj-pay-service | 服务实现模块，包含支付对接逻辑、控制器和定时任务 |

**启动类：** `com.tianji.pay.PayApplication`

## 二、依赖分析

### tj-pay-service 依赖
| 依赖 | 说明 |
|------|------|
| tj-pay-domain | 支付领域对象 |
| tj-common | 公共基础模块 |
| spring-boot-starter-web | Web 框架 |
| spring-cloud-starter-alibaba-sentinel | Sentinel 限流熔断 |
| spring-boot-starter-data-redis + redisson | Redis 缓存与分布式锁 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| alipay-easysdk | 支付宝支付SDK |
| wechatpay-apache-httpclient | 微信支付SDK（V3版本） |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| xxl-job-core | XXL-JOB 分布式任务调度 |

## 三、核心业务逻辑

### 3.1 支付订单管理（PayOrderController + PayOrderServiceImpl）
- **支付申请**：接收 PayApplyDTO -> 创建/更新支付订单 -> 调用第三方支付API生成预支付单 -> 返回支付二维码URL
- **查询支付结果**：根据业务订单ID查询支付订单状态，返回 PayResultDTO

### 3.2 退款订单管理（RefundOrderController + RefundOrderServiceImpl）
- **申请退款**：接收 RefundApplyDTO -> 创建退款订单 -> 调用第三方支付API发起退款 -> 返回退款结果
- **查询退款结果**：根据业务退款订单ID查询退款状态

### 3.3 支付渠道管理（PayChannelController + PayChannelServiceImpl）
- **查询支付渠道列表**：返回所有可用的支付渠道
- **新增/修改支付渠道**

### 3.4 支付回调处理（NotifyController + NotifyServiceImpl）
- **支付宝回调**：接收支付宝异步通知 -> 验签 -> 更新支付订单状态 -> 发送MQ消息通知业务服务
- **微信支付回调**：接收微信支付异步通知 -> 验签解密 -> 更新支付订单状态 -> 发送MQ消息
- **微信退款回调**：接收微信退款结果通知 -> 更新退款订单状态 -> 发送MQ消息

### 3.5 第三方支付对接

通过 `IPayService` 接口抽象第三方支付操作：

| 实现类 | 平台 | 功能 |
|--------|------|------|
| AliPayService | 支付宝 | 预支付、支付状态查询、退款、退款查询 |
| WxPayService | 微信支付 | 预支付（Native）、支付状态查询、退款、退款查询 |

### 3.6 定时任务

| 任务类 | 功能 |
|--------|------|
| PayOrderCheckTask | 定时检查待支付订单状态，超时未支付的订单主动查询第三方结果 |
| RefundOrderCheckTask | 定时检查退款中的订单，主动查询第三方退款结果 |

## 四、数据模型

### 4.1 PayOrder（支付订单表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 支付订单ID |
| bizOrderNo | Long | 业务订单号 |
| payOrderNo | Long | 支付单号 |
| bizUserId | Long | 支付用户ID |
| payChannelCode | String | 支付渠道代码 |
| amount | Integer | 支付金额（分） |
| payType | Integer | 支付类型：1H5、2小程序、3公众号、4扫码 |
| status | Integer | 支付状态：0待提交、1待支付、2支付成功、3支付超时/取消 |
| qrCodeUrl | String | 支付二维码URL |
| notifyUrl | String | 业务端回调接口 |
| notifyTimes | Integer | 回调次数 |
| notifyStatus | Integer | 回调状态：0待回调、1回调成功、2回调失败 |
| paySuccessTime | LocalDateTime | 支付成功时间 |
| payOverTime | LocalDateTime | 支付超时时间 |

### 4.2 RefundOrder（退款订单表）
退款订单记录，包含退款金额、退款状态、退款渠道等信息。

### 4.3 PayChannel（支付渠道表）
支付渠道配置信息，如支付宝、微信支付等渠道的状态和描述。

## 五、API接口清单

### 支付订单接口（/pay-orders）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /pay-orders | 扫码支付申请，返回二维码URL |
| GET | /pay-orders/{bizOrderId}/status | 根据业务订单ID查询支付结果 |

### 退款订单接口（/refund-orders）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /refund-orders | 申请退款 |
| GET | /refund-orders/{bizRefundOrderId}/status | 查询退款结果 |

### 支付渠道接口（/pay-channels）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /pay-channels/list | 查询支付渠道列表 |
| POST | /pay-channels | 添加支付渠道 |
| PUT | /pay-channels/{id} | 修改支付渠道 |

### 回调接口（/notify）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /notify/aliPay | 支付宝支付回调 |
| POST | /notify/wxPay | 微信支付回调 |
| POST | /notify/refund/wxPay | 微信退款回调 |

## 六、技术方案

1. **策略模式支付对接**：通过 `IPayService` 接口抽象支付操作，AliPayService 和 WxPayService 分别实现，运行时根据支付渠道选择对应实现。
2. **MQ异步通知**：支付/退款结果变更后通过 RabbitMQ 发送消息（pay.topic交换机），交易服务监听消息处理业务逻辑，实现支付服务与业务服务的解耦。
3. **定时任务对账**：通过 XXL-JOB 定时检查待处理的支付/退款订单，主动向第三方查询状态，解决回调丢失的问题。
4. **Sentinel限流**：对支付核心接口进行限流保护。
5. **微信支付V3签名验证**：使用 wechatpay-apache-httpclient 处理微信支付V3版本的签名和解密。
6. **回调幂等处理**：支付回调接口支持重复调用（幂等），避免重复处理支付结果。
