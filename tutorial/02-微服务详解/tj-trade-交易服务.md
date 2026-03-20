# tj-trade 交易服务详解

## 一、服务概述

tj-trade 是天机学堂的交易核心服务，负责购物车管理、订单生命周期管理、支付发起以及退款审批等完整交易链路。该服务通过 RabbitMQ 与支付服务（tj-pay）、课程服务、促销服务进行异步通信，实现了支付结果回调处理、订单超时自动取消、退款异步申请等核心业务流程。

**服务端口：** 由 Nacos 配置中心统一管理
**注册中心：** Nacos
**配置中心：** Nacos

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| tj-pay-api | 支付服务 Feign 客户端，用于发起支付、退款、查询支付结果 |
| tj-api | 公共 API 模块，包含 CourseClient、UserClient、PromotionClient 等 Feign 客户端 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK，实现用户登录状态拦截和用户信息传递 |
| spring-boot-starter-web | Web 框架 |
| spring-cloud-starter-alibaba-sentinel | Sentinel 限流熔断 |
| spring-boot-starter-data-redis + redisson | Redis 缓存与分布式锁 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库驱动 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-boot-starter-amqp | RabbitMQ 消息队列 |
| xxl-job-core | XXL-JOB 分布式任务调度 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |

## 三、核心业务逻辑

### 3.1 购物车模块（CartServiceImpl）

- **添加课程到购物车**：校验课程是否已在购物车 -> 校验购物车数量上限（默认10）-> 远程查询课程信息 -> 校验课程是否过期 -> 写入购物车表
- **获取购物车列表**：查询当前用户购物车 -> 远程查询最新课程信息（价格、有效期）-> 排序返回（未过期在前，新添加在前）
- **删除购物车条目**：支持单个和批量删除，均校验用户归属

### 3.2 订单模块（OrderServiceImpl）

- **预下单**：查询课程信息 -> 计算总价 -> 通过 PromotionClient 查询可用优惠券方案 -> 生成订单ID -> 返回确认信息
- **下单（placeOrder）**：校验课程在售状态 -> 计算总价与优惠金额 -> 创建订单和订单明细 -> 删除购物车 -> 核销优惠券 -> 返回下单结果（含超时时间）
- **免费课报名（enrolledFreeCourse）**：校验免费课程 -> 创建零元订单 -> 发送MQ消息（ORDER_PAY_KEY）通知报名成功
- **支付成功处理（handlePaySuccess）**：更新订单状态为已支付 -> 更新订单明细 -> 发送MQ消息通知报名成功
- **取消订单**：幂等校验 -> 仅允许未支付订单取消 -> 更新状态为已关闭 -> 退还优惠券
- **订单查询**：支持分页查询我的订单、按ID查询详情（含进度节点和优惠明细）、查询订单支付状态

### 3.3 支付模块（PayServiceImpl）

- **发起支付**：校验订单状态（未支付）和超时 -> 封装支付参数调用 PayClient -> 发送延迟消息轮询支付状态
- **延迟查询支付结果**：收到延迟消息 -> 查询支付状态 -> 成功则处理支付回调 -> 未成功则继续发延迟消息重试 -> 重试耗尽则自动取消订单

### 3.4 退款模块（RefundApplyServiceImpl）

- **申请退款**：校验订单归属与状态 -> 检查退款次数限制（学员最多2次）-> 创建退款申请记录 -> 更新订单/明细状态 -> 管理员直接退款时异步发送退款请求
- **审批退款**：校验退款申请状态 -> 记录审批结果 -> 同意则异步调用 PayClient 发起退款
- **退款结果处理**：接收MQ消息 -> 更新退款状态 -> 退款成功则发送MQ消息取消用户课程报名

### 3.5 MQ消息处理（PayMessageHandler）

| 队列 | Exchange | RoutingKey | 功能 |
|------|----------|------------|------|
| trade.pay.success.queue | pay.topic | pay.success | 接收支付成功通知 |
| trade.refund.result.queue | pay.topic | refund.change | 接收退款结果变更通知 |
| trade.delay.order.query | trade.delay.topic（延迟交换机） | order.delay | 延迟查询订单支付状态 |

## 四、数据模型

### 4.1 Order（订单表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 订单ID，雪花算法生成 |
| payOrderNo | Long | 支付交易流水号 |
| userId | Long | 用户ID |
| status | Integer | 订单状态：1待支付、2已支付、3已关闭、4已完成、5已报名、6已申请退款 |
| message | String | 状态备注 |
| totalAmount | Integer | 订单总金额（分） |
| realAmount | Integer | 实付金额（分） |
| discountAmount | Integer | 优惠金额（分） |
| payChannel | String | 支付渠道 |
| couponIds | List<Long> | 优惠券ID集合（JSON存储） |
| createTime / payTime / closeTime / finishTime / refundTime | LocalDateTime | 各状态时间节点 |

### 4.2 OrderDetail（订单明细表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 订单明细ID |
| orderId | Long | 所属订单ID |
| userId | Long | 用户ID |
| courseId | Long | 课程ID |
| price | Integer | 课程价格（分） |
| name | String | 课程名称 |
| coverUrl | String | 封面地址 |
| validDuration | Integer | 课程学习有效期（月） |
| discountAmount | Integer | 折扣金额 |
| realPayAmount | Integer | 实付金额 |
| status | Integer | 订单详情状态 |
| refundStatus | Integer | 退款状态 |

### 4.3 Cart（购物车表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 购物车条目ID |
| userId | Long | 用户ID |
| courseId | Long | 课程ID |
| coverUrl | String | 课程封面路径 |
| courseName | String | 课程名称 |
| price | Integer | 单价（分） |

### 4.4 RefundApply（退款申请表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 退款ID |
| orderDetailId | Long | 订单明细ID |
| orderId | Long | 订单ID |
| refundOrderNo | Long | 退款单号 |
| userId | Long | 订单所属用户ID |
| refundAmount | Integer | 退款金额（分） |
| status | Integer | 退款状态：1待审批、2取消退款、3同意退款、4拒绝退款、5退款成功、6退款失败 |
| refundReason | String | 申请退款原因 |
| approver | Long | 审批人ID |
| approveOpinion | String | 审批意见 |

## 五、API接口清单

### 购物车接口（/carts）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /carts | 添加课程到购物车 |
| GET | /carts | 获取购物车中的课程 |
| DELETE | /carts/{id} | 删除指定购物车条目 |
| DELETE | /carts?ids= | 批量删除购物车条目 |

### 订单接口（/orders）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /orders/page | 分页查询我的订单 |
| GET | /orders/{id} | 根据ID查询订单详情 |
| GET | /orders/{id}/status | 查询订单支付状态 |
| GET | /orders/prePlaceOrder | 预下单（确认订单） |
| POST | /orders/placeOrder | 下单接口 |
| POST | /orders/freeCourse/{courseId} | 免费课立刻报名 |
| PUT | /orders/{id}/cancel | 取消订单 |
| DELETE | /orders/{id} | 删除订单 |

### 订单明细接口（/order-details）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /order-details/page | 分页查询订单明细 |
| GET | /order-details/{id} | 根据ID获取订单明细详情 |
| GET | /order-details/course/{id} | 校验课程是否购买及是否过期 |
| GET | /order-details/enrollNum | 统计课程报名人数 |
| GET | /order-details/enrollCourse | 统计学生报名课程数量 |
| GET | /order-details/purchaseInfo | 查询课程购买信息 |

### 支付接口（/pay）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /pay/order | 支付申请，返回支付二维码URL |
| GET | /pay/channels | 获取支付渠道列表 |

### 退款接口（/refund-apply）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /refund-apply | 退款申请 |
| PUT | /refund-apply/approval | 审批退款申请 |
| PUT | /refund-apply/cancel | 取消退款申请 |
| GET | /refund-apply/page | 分页查询退款申请 |
| GET | /refund-apply/{id} | 根据ID查询退款详情 |
| GET | /refund-apply/detail/{id} | 根据子订单ID查询退款详情 |
| GET | /refund-apply/next | 查询下一个待审批的退款申请 |

## 六、技术方案

1. **支付结果异步轮询**：采用 RabbitMQ 延迟消息队列，在用户发起支付后通过递增间隔的延迟消息反复查询支付状态，避免长轮询和频繁查询第三方支付平台。
2. **订单超时自动取消**：延迟消息重试次数耗尽后自动调用取消订单逻辑，实现订单超时关闭。
3. **退款异步处理**：退款审批通过后，通过独立线程池异步发送退款请求到支付服务，避免阻塞主流程。
4. **Sentinel 限流**：集成 Sentinel 对交易核心接口进行限流熔断保护。
5. **乐观锁防重**：订单状态更新使用 where status=原状态 的条件更新，实现乐观锁防止并发问题。
6. **优惠券联动**：下单时核销优惠券，取消/退款时退还优惠券，与促销服务通过 Feign 调用保持一致。
