# tj-promotion 促销服务 -- 深度架构分析

## 一、服务概述

`tj-promotion` 是天机学堂微服务体系中的**促销中心服务**，负责优惠券全生命周期管理，包括优惠券的创建、发放、领取、兑换、核销、退还，以及订单下单时的**优惠券叠加方案智能推荐**。

- **服务名**：`promotion-service`
- **端口**：`8092`
- **数据库**：`tj_promotion`
- **启动类**：`com.tianji.promotion.PromotionApplication`

### 核心能力

| 能力域 | 说明 |
|--------|------|
| 优惠券管理 | 优惠券CRUD、发放/暂停/删除，支持草稿-未开始-发放中-已结束-暂停的完整状态机 |
| 兑换码 | 支持20亿量级的兑换码生成，基于自研位运算+Base32编码算法 |
| 领券 | 基于Redis LUA脚本的原子性校验 + RabbitMQ异步写库，保证高并发安全 |
| 兑换码兑换 | 基于Bitmap + ZSet的Redis数据结构设计 + LUA脚本原子操作 |
| 优惠方案推荐 | 基于排列组合+多线程并行计算的最优优惠券叠加方案推荐算法 |
| 定时任务 | 基于XXL-JOB分片广播的优惠券自动开始发放 |

---

## 二、依赖分析

### 2.1 Maven依赖清单

```xml
<!-- pom.xml 核心依赖 -->
tj-auth-resource-sdk    -- 认证鉴权SDK（登录拦截）
tj-api                  -- 服务间调用API（Feign客户端）
spring-boot-starter-web -- Web框架
mybatis-plus-boot-starter -- ORM框架（MyBatis-Plus）
mysql-connector-java    -- MySQL驱动
spring-boot-starter-data-redis -- Redis操作
redisson               -- Redisson分布式锁
spring-cloud-starter-alibaba-nacos-discovery -- Nacos服务发现
spring-cloud-starter-alibaba-nacos-config    -- Nacos配置中心
caffeine               -- Caffeine本地缓存
xxl-job-core           -- XXL-JOB分布式定时任务
spring-cloud-starter-loadbalancer -- 负载均衡
aspectjweaver          -- AOP切面
spring-boot-starter-amqp -- RabbitMQ消息队列
```

### 2.2 Nacos共享配置

服务从Nacos拉取7组共享配置：`shared-spring`、`shared-redis`、`shared-mybatis`、`shared-logs`、`shared-feign`、`shared-xxljob`、`shared-mq`。

### 2.3 与其他服务的交互

- **Feign调用**：通过 `CourseClient` 查询课程信息（用于优惠券使用范围名称解析）
- **缓存调用**：通过 `CategoryCache` 查询分类名称
- **MQ消息**：向 `PROMOTION_EXCHANGE` 发送领券/兑换消息，由自身MQ消费者消费写库

---

## 三、数据模型

### 3.1 核心实体（PO）

#### Coupon -- 优惠券规则表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券ID（雪花算法） |
| name | String | 优惠券名称 |
| type | Integer | 优惠券类型（1：普通券，保留字段） |
| discountType | DiscountType | 折扣类型：1-每满减 2-折扣 3-无门槛 4-满减 |
| specific | Boolean | 是否限定使用范围 |
| discountValue | Integer | 折扣值（满减存金额，折扣存折扣率如80=8折） |
| thresholdAmount | Integer | 使用门槛（0=无门槛） |
| maxDiscountAmount | Integer | 最高优惠金额（0=无限制） |
| obtainWay | ObtainType | 获取方式：1-手动领取 2-兑换码 |
| issueBeginTime | LocalDateTime | 发放开始时间 |
| issueEndTime | LocalDateTime | 发放结束时间 |
| termDays | Integer | 有效期天数（0=指定日期有效期） |
| termBeginTime | LocalDateTime | 有效期开始时间 |
| termEndTime | LocalDateTime | 有效期结束时间 |
| status | CouponStatus | 状态：1-待发放 2-未开始 3-发放中 4-已结束 5-暂停 |
| totalNum | Integer | 总数量（不超过5000） |
| issueNum | Integer | 已发放数量 |
| usedNum | Integer | 已使用数量 |
| userLimit | Integer | 每人限领数量（默认1） |

#### CouponScope -- 优惠券作用范围表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 自增主键 |
| type | Integer | 范围类型：1-分类 2-课程 |
| couponId | Long | 优惠券ID |
| bizId | Long | 业务ID（分类ID或课程ID） |

#### ExchangeCode -- 兑换码表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 兑换码序列号（手动指定，即全局递增序列号） |
| code | String | 兑换码字符串（10位Base32） |
| status | ExchangeCodeStatus | 状态：1-待兑换 2-已兑换 3-活动已结束 |
| userId | Long | 兑换人 |
| type | Integer | 兑换类型（1-优惠券） |
| exchangeTargetId | Long | 目标ID（优惠券ID） |
| expiredTime | LocalDateTime | 过期时间 |

#### UserCoupon -- 用户优惠券记录表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 雪花算法主键 |
| userId | Long | 用户ID |
| couponId | Long | 优惠券模板ID |
| termBeginTime | LocalDateTime | 有效期开始 |
| termEndTime | LocalDateTime | 有效期结束 |
| usedTime | LocalDateTime | 核销时间 |
| status | UserCouponStatus | 状态：1-未使用 2-已使用 3-已过期 |

#### Promotion -- 促销活动表

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 活动ID |
| name | String | 活动名称 |
| type | Integer | 类型：1-优惠券 2-分销 |
| hot | Integer | 是否热门 |
| beginTime/endTime | LocalDateTime | 活动时间范围 |

### 3.2 枚举定义

| 枚举类 | 值 | 说明 |
|--------|-----|------|
| CouponStatus | DRAFT(1), UN_ISSUE(2), ISSUING(3), FINISHED(4), PAUSE(5) | 优惠券状态 |
| DiscountType | PER_PRICE_DISCOUNT(1), RATE_DISCOUNT(2), NO_THRESHOLD(3), PRICE_DISCOUNT(4) | 折扣类型 |
| ObtainType | PUBLIC(1), ISSUE(2) | 获取方式 |
| ExchangeCodeStatus | UNUSED(1), USED(2), EXPIRED(3) | 兑换码状态 |
| UserCouponStatus | UNUSED(1), USED(2), EXPIRED(3) | 用户券状态 |
| ScopeType | ALL(0), CATEGORY(1), COURSE(2) | 作用范围类型 |

### 3.3 DTO / VO / Query

**DTO（入参）**：
- `CouponFormDTO` -- 新增优惠券表单（含名称、类型、折扣值、门槛、总量、限领数、获取方式、范围）
- `CouponIssueFormDTO` -- 发放优惠券表单（含发放时间、有效期）
- `CouponScopeDTO` -- 优惠券使用范围
- `UserCouponDTO` -- MQ消息体（userId + couponId + serialNum）

**VO（出参）**：
- `CouponVO` -- 用户端优惠券信息（含available可领取、received已领取标记）
- `CouponPageVO` -- 管理端分页数据
- `CouponDetailVO` -- 优惠券详情（含范围列表）
- `CouponScopeVO` -- 范围项（id + name）
- `ExchangeCodeVO` -- 兑换码（id + code）
- `UserCouponVO` -- 用户优惠券

**Query（查询条件）**：
- `CouponQuery` -- 按type/status/name分页查询
- `CodeQuery` -- 按couponId/status分页查询兑换码
- `UserCouponQuery` -- 按status分页查询用户券

---

## 四、核心业务逻辑详解

### 4.1 优惠券状态机与CRUD管理

优惠券的完整生命周期状态流转如下：

```
DRAFT(待发放) --[发放]--> ISSUING(发放中) / UN_ISSUE(未开始)
UN_ISSUE(未开始) --[定时任务触发]--> ISSUING(发放中)
ISSUING(发放中) --[暂停]--> PAUSE(暂停)
PAUSE(暂停) --[重新发放]--> ISSUING(发放中)
ISSUING(发放中) --[到期/发完]--> FINISHED(已结束)
DRAFT(待发放) --[删除]--> 物理删除
```

**发放流程（beginIssue）关键逻辑**：

1. 校验优惠券存在且状态为 DRAFT 或 PAUSE
2. 判断是否立即发放（issueBeginTime为null或已过期表示立即发放）
3. 如果立即发放，状态更新为 ISSUING，否则更新为 UN_ISSUE
4. 立即发放时缓存优惠券信息到Redis（Hash结构）
5. 如果获取方式是兑换码且原状态是 DRAFT，异步生成兑换码

**删除**只允许对 DRAFT 状态的优惠券执行，同时清理关联的 CouponScope 记录。

### 4.2 兑换码生成算法（支持20亿量级）

这是整个促销服务最精巧的算法设计。兑换码是一个**10位Base32编码字符串**，对应50位二进制明文。

#### 4.2.1 明文结构（50位二进制）

```
| 校验码(14位) | 新鲜值(4位) | 序列号(32位) |
```

- **序列号（32位）**：通过Redis INCR全局递增生成，最大值 2^32 = **4,294,967,296**（约43亿），远超20亿需求
- **新鲜值（4位）**：取优惠券ID的低4位，同一优惠券的兑换码共享标记
- **载荷（36位）**：新鲜值(4位) + 序列号(32位) 拼接而成
- **校验码（14位）**：载荷每4位一组乘以加权系数求和，对2^14取余

#### 4.2.2 加密过程

```
1. 计算新鲜值 f = couponId & 0xF
2. 拼接载荷 payload = f << 32 | serialNum
3. 以 f 为角标从 PRIME_TABLE[16组] 选取加权码表
4. 计算校验码 checkCode = calcCheckCode(payload, f)
5. 用 checkCode 的低5位做角标，从 XOR_TABLE[32组] 取异或密钥
6. payload ^= XOR_TABLE[checkCode & 0b11111]  -- 混淆
7. 拼接明文 code = checkCode << 36 | payload
8. Base32编码，输出10位字符串
```

#### 4.2.3 解密过程

```
1. Base32解码得到 num
2. 取高14位得 checkCode，取低36位得 payload
3. 用 checkCode & 0b11111 做角标取 XOR_TABLE 密钥
4. payload ^= 密钥  -- 反混淆
5. 取 payload 高4位得 fresh，用 fresh 选取码表
6. 重新计算校验码与 checkCode 比对，通过则提取低32位序列号
```

#### 4.2.4 Base32编码

使用自定义字符表 `"6CSB7H8DAKXZF3N95RTMVUQG2YE4JWPL"`（32个字符），每5个bit对应一个字符，50位 / 5 = 10个字符。

#### 4.2.5 异步生成流程

```java
@Async("generateExchangeCodeExecutor")
public void asyncGenerateCode(Coupon coupon) {
    // 1. Redis INCR 批量获取序列号区间
    Long maxSerialNum = serialOps.increment(totalNum);
    // 2. 循环生成兑换码并批量入库
    for (int serialNum = maxSerialNum - totalNum + 1; serialNum <= maxSerialNum; serialNum++) {
        String code = CodeUtil.generateCode(serialNum, coupon.getId());
        ...
    }
    saveBatch(list);
    // 3. 写Redis ZSet（member=couponId, score=最大序列号）
    redisTemplate.opsForZSet().add(COUPON_RANGE_KEY, couponId, maxSerialNum);
}
```

使用独立线程池 `generateExchangeCodeExecutor`（核心2，最大5，队列200，CallerRunsPolicy拒绝策略）异步执行。

### 4.3 领券功能的并发安全优化

领券是高并发热点操作，采用 **Redis LUA脚本原子校验 + RabbitMQ异步写库** 的架构：

#### 4.3.1 领券LUA脚本（receive_coupon.lua）

```lua
-- KEYS[1] = prs:coupon:{couponId}    优惠券Hash
-- KEYS[2] = prs:user:coupon:{couponId}  用户领券Hash
-- ARGV[1] = userId

-- 1. 校验优惠券存在
if(redis.call('exists', KEYS[1]) == 0) then return 1 end
-- 2. 校验库存
if(tonumber(redis.call('hget', KEYS[1], 'totalNum')) <= 0) then return 2 end
-- 3. 校验活动是否结束（对比Redis服务器时间戳）
if(tonumber(redis.call('time')[1]) > tonumber(redis.call('hget', KEYS[1], 'issueEndTime'))) then return 3 end
-- 4. 校验每人限领数量（先自增再校验，原子操作）
if(tonumber(redis.call('hget', KEYS[1], 'userLimit')) < redis.call('hincrby', KEYS[2], ARGV[1], 1)) then return 4 end
-- 5. 扣减库存
redis.call('hincrby', KEYS[1], "totalNum", "-1")
return 0
```

**返回值含义**：0=成功 1=优惠券不存在 2=库存不足 3=活动已结束 4=超出限领数量

错误信息映射：

```java
String[] RECEIVE_COUPON_ERROR_MSG = {"活动未开始", "库存不足", "活动已经结束", "领取次数过多"};
```

#### 4.3.2 兑换码兑换LUA脚本（exchange_coupon.lua）

```lua
-- KEYS[1] = coupon:code:map    Bitmap记录兑换状态
-- KEYS[2] = coupon:code:range  ZSet记录优惠券序列号范围
-- ARGV[1] = serialNum  ARGV[2] = serialNum+5000  ARGV[3] = userId

-- 1. 检查是否已兑换（Bitmap）
if(redis.call('GETBIT', KEYS[1], ARGV[1]) == 1) then return "1" end
-- 2. 通过ZSet查找对应的优惠券ID
local arr = redis.call('ZRANGEBYSCORE', KEYS[2], ARGV[1], ARGV[2], 'LIMIT', 0, 1)
if(#arr == 0) then return "2" end
local cid = arr[1]
-- 3-4. 校验优惠券存在和过期时间
...
-- 5. 校验限领数量
...
-- 6. 标记为已兑换
redis.call('SETBIT', KEYS[1], ARGV[1], "1")
return cid  -- 返回优惠券ID（>=10的值表示成功）
```

#### 4.3.3 异步写库流程

LUA脚本校验通过后，发送MQ消息到 `coupon.receive.queue`：

```
receiveCoupon/exchangeCoupon
    --> LUA脚本原子校验
    --> 发送MQ消息 (UserCouponDTO: userId, couponId, serialNum)
    --> MQ消费者: checkAndCreateUserCoupon
        --> couponMapper.incrIssueNum（乐观锁：issue_num < total_num）
        --> 保存UserCoupon记录
        --> 更新兑换码状态（如有）
```

### 4.4 Redis数据结构设计

| Redis Key | 类型 | 说明 |
|-----------|------|------|
| `prs:coupon:{couponId}` | Hash | 优惠券缓存：issueBeginTime, issueEndTime, totalNum, userLimit |
| `prs:user:coupon:{couponId}` | Hash | 用户领券计数：field=userId, value=已领数量 |
| `coupon:code:serial` | String | 兑换码全局序列号（INCR递增） |
| `coupon:code:map` | Bitmap | 兑换码使用标记：bit位=序列号，1=已兑换 |
| `coupon:code:range` | ZSet | 优惠券序列号范围：member=couponId, score=该券最大序列号 |

**ZSet巧妙设计**：多个优惠券的兑换码共享一个递增序列号空间，通过 ZRANGEBYSCORE 在范围内查找第一个 score >= serialNum 的优惠券，从而确定一个序列号属于哪张优惠券。

### 4.5 优惠券叠加方案推荐算法

这是订单结算时最核心的计算逻辑，位于 `DiscountServiceImpl.findDiscountSolution()`。

#### 4.5.1 算法流程

```
1. 查询用户所有可用优惠券（status=UNUSED的UserCoupon JOIN Coupon）
2. 初筛：计算订单总价，过滤不满足门槛的券
3. 细筛：对每张券匹配可用课程（通过CouponScope关联分类/课程）
4. 排列组合：用PermuteUtil生成所有券的全排列方案 + 单券方案
5. 多线程并行计算：CompletableFuture + CountDownLatch(1秒超时)
6. 计算每种方案的优惠明细
7. 筛选最优解：相同用券组合取最大优惠，相同优惠金额取最少用券数
8. 返回优惠金额降序排列的方案列表
```

#### 4.5.2 全排列工具（PermuteUtil）

基于**回溯算法**实现全排列：

```java
private static <T> void backtrack(int n, List<T> input, List<List<T>> res, int first) {
    if (first == n) {
        res.add(new ArrayList<>(input));
    }
    for (int i = first; i < n; i++) {
        Collections.swap(input, first, i);   // 交换
        backtrack(n, input, res, first + 1);  // 递归
        Collections.swap(input, first, i);    // 撤销
    }
}
```

排列方案数量为 n!（阶乘），因此优惠券总量受限于5000，每人限领10张，实际组合数可控。

#### 4.5.3 并行计算设计

```java
// 线程池：12核心/12最大线程，队列99999
Executor discountSolutionExecutor;

// 使用CountDownLatch + CompletableFuture并行计算
CountDownLatch latch = new CountDownLatch(solutions.size());
for (List<Coupon> solution : solutions) {
    CompletableFuture
        .supplyAsync(() -> calculateSolutionDiscount(...), discountSolutionExecutor)
        .thenAccept(dto -> { list.add(dto); latch.countDown(); });
}
latch.await(1, TimeUnit.SECONDS);  // 最多等待1秒
```

#### 4.5.4 优惠明细计算

计算每个方案时，采用**按比例分摊**的策略：

- 对方案中的每张优惠券，计算其适用课程的总价
- 减去前面券已经产生的折扣金额，得到"当前剩余总价"
- 判断是否满足该券的使用门槛
- 计算该券的优惠金额
- 按每门课在总价中的**占比**分摊优惠到课程明细
- 最后一门课用"总折扣 - 前面课的折扣之和"避免精度误差

#### 4.5.5 最优解筛选

双维度筛选：

```java
// 维度1：同券组合，取最大优惠金额
Map<String, CouponDiscountDTO> moreDiscountMap;
// 维度2：同优惠金额，取最少用券数
Map<Integer, CouponDiscountDTO> lessCouponMap;
// 取两者交集，按优惠金额降序排列
Collection<CouponDiscountDTO> bestSolutions = CollUtils.intersection(
    moreDiscountMap.values(), lessCouponMap.values());
```

### 4.6 折扣策略模式

采用**策略模式**实现四种折扣类型：

```
Discount (接口)
  ├── NoThresholdDiscount  -- 无门槛抵扣（totalAmount > discountValue 即可用）
  ├── PriceDiscount        -- 满减（满 thresholdAmount 减 discountValue）
  ├── PerPriceDiscount     -- 每满减（循环减，上限 maxDiscountAmount）
  └── RateDiscount         -- 折扣（满 thresholdAmount 打 discountValue/100 折，上限 maxDiscountAmount）
```

通过 `DiscountStrategy` 工厂类（EnumMap）根据 `DiscountType` 枚举获取对应策略：

```java
static {
    strategies = new EnumMap<>(DiscountType.class);
    strategies.put(DiscountType.NO_THRESHOLD, new NoThresholdDiscount());
    strategies.put(DiscountType.PER_PRICE_DISCOUNT, new PerPriceDiscount());
    strategies.put(DiscountType.RATE_DISCOUNT, new RateDiscount());
    strategies.put(DiscountType.PRICE_DISCOUNT, new PriceDiscount());
}
```

**每满减**的计算逻辑示例：

```java
public int calculateDiscount(int totalAmount, Coupon coupon) {
    int discount = 0;
    while (totalAmount >= thresholdAmount) {
        discount += discountValue;
        totalAmount -= thresholdAmount;
    }
    return Math.min(discount, coupon.getMaxDiscountAmount());
}
```

### 4.7 使用范围策略

优惠券使用范围也采用策略模式：

```
Scope (接口)
  ├── NoScope        -- 不限范围（始终返回true）
  ├── CategoryScope  -- 限定分类（判断课程分类ID是否在范围内）
  └── CourseScope    -- 限定课程（判断课程ID是否在范围内）
```

`ScopeType` 枚举充当工厂，通过 `buildScope(scopeIds)` 方法创建具体策略实例。

范围名称解析通过 `ScopeNameHandler` 接口实现：
- `CategoryScopeNameHandler` -- 通过 `CategoryCache` 查分类名
- `CourseScopeNameHandler` -- 通过 `CourseClient`（Feign）查课程名

---

## 五、分布式锁设计

### 5.1 @MyLock 注解式分布式锁

服务实现了一套完整的声明式分布式锁框架：

**注解定义**（`@MyLock`）：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyLock {
    String name();                     // 锁名称（支持SpEL）
    long waitTime() default 1;         // 等待时间
    long leaseTime() default -1;       // 持有时间（-1=看门狗续期）
    TimeUnit unit() default SECONDS;
    MyLockType lockType() default RE_ENTRANT_LOCK;
    MyLockStrategy lockStrategy() default FAIL_AFTER_RETRY_TIMEOUT;
}
```

**锁类型**（`MyLockType`）：

| 类型 | 说明 |
|------|------|
| RE_ENTRANT_LOCK | 可重入锁（默认） |
| FAIR_LOCK | 公平锁 |
| READ_LOCK | 读写锁-读锁 |
| WRITE_LOCK | 读写锁-写锁 |

**锁策略**（`MyLockStrategy`）：

| 策略 | 行为 |
|------|------|
| SKIP_FAST | 获取不到立即跳过，返回null |
| FAIL_FAST | 获取不到立即抛异常 |
| KEEP_TRYING | 无限等待直到获取到 |
| SKIP_AFTER_RETRY_TIMEOUT | 超时后跳过 |
| FAIL_AFTER_RETRY_TIMEOUT | 超时后抛异常（默认） |

**锁工厂**（`MyLockFactory`）：根据 `MyLockType` 从 Redisson 获取对应的 `RLock` 实例。

**AOP切面**（`MyLockAspect`）：拦截 `@MyLock` 注解方法，自动加锁/解锁，实现 `Ordered` 接口确保切面顺序。

### 5.2 简易RedisLock

还提供了一个简化版的 `RedisLock`，基于 `SETNX` 实现：

```java
public boolean tryLock(long leaseTime, TimeUnit unit) {
    String value = Thread.currentThread().getName();
    Boolean success = redisTemplate.opsForValue().setIfAbsent(key, value, leaseTime, unit);
    return BooleanUtils.isTrue(success);
}
```

### 5.3 实际使用

代码中注释掉了基于 `@Lock` 注解的分布式锁方案（领券和兑换券方法），改用了更高效的 LUA 脚本方案。这表明在高并发场景下，LUA 脚本的原子性校验比分布式锁有更好的性能表现。

---

## 六、MQ消费者与定时任务

### 6.1 RabbitMQ消费者

**PromotionMqHandler** 监听领券/兑换消息：

```java
@RabbitListener(bindings = @QueueBinding(
    value = @Queue(name = "coupon.receive.queue", durable = "true"),
    exchange = @Exchange(name = PROMOTION_EXCHANGE, type = ExchangeTypes.TOPIC),
    key = COUPON_RECEIVE
))
public void listenCouponReceiveMessage(UserCouponDTO uc) {
    userCouponService.checkAndCreateUserCoupon(uc);
}
```

消费逻辑：
1. 查询优惠券是否存在
2. 乐观锁更新 issue_num（`WHERE issue_num < total_num`）
3. 创建 UserCoupon 记录（根据 termDays 或固定时间设置有效期）
4. 如果是兑换码兑换，更新兑换码状态为已兑换

### 6.2 XXL-JOB定时任务

**CouponIssueTaskHandler** -- 优惠券定时发放：

```java
@XxlJob("couponIssueJobHandler")
public void handleCouponIssueJob() {
    int index = XxlJobHelper.getShardIndex() + 1;  // 分片索引作为页码
    int size = Integer.parseInt(XxlJobHelper.getJobParam());  // 参数为每页数量
    // 查询状态为UN_ISSUE且开始时间已到的优惠券
    Page<Coupon> page = couponService.lambdaQuery()
        .eq(Coupon::getStatus, CouponStatus.UN_ISSUE)
        .le(Coupon::getTermBeginTime, LocalDateTime.now())
        .page(new Page<>(index, size));
    // 批量更新为发放中，并缓存到Redis（Pipeline批量写入）
    couponService.beginIssueBatch(records);
}
```

支持**分片广播**，多个执行器实例分别处理不同页的数据，批量缓存使用 Redis Pipeline 优化性能。

---

## 七、技术亮点总结

### 7.1 兑换码算法 -- 位运算+加权校验+异或混淆

- 50位二进制 = 14位校验码 + 4位新鲜值 + 32位序列号
- 16组素数加权码表 + 32组异或密钥表实现双重混淆
- Base32编码输出10位人类可读字符串
- 32位序列号支持 2^32（43亿）个不重复兑换码
- 解码时通过校验码逆向验证，篡改任意位都会校验失败

### 7.2 高并发领券 -- LUA脚本+MQ异步

- LUA脚本在Redis侧原子性完成5项校验（存在性、库存、时间、限领、扣减）
- 避免了分布式锁的性能瓶颈
- MQ异步写库削峰填谷，保证最终一致性
- 乐观锁兜底（`issue_num < total_num`）

### 7.3 优惠券叠加推荐 -- 排列组合+并行计算

- 全排列算法穷举所有可能方案
- CompletableFuture + 专用线程池（12线程）并行计算
- CountDownLatch 1秒超时保护，防止计算时间过长
- 双维度最优解筛选（最大优惠 AND 最少用券数）
- 按比例分摊优惠明细到课程级别

### 7.4 Redis数据结构精巧设计

- Hash存优惠券信息支持局部读写
- Bitmap以O(1)标记兑换码使用状态，内存极省
- ZSet存储优惠券序列号范围，支持范围查询定位优惠券
- Pipeline批量写入优化网络开销

### 7.5 声明式分布式锁框架

- 注解式使用，对业务代码零侵入
- 支持4种锁类型（可重入/公平/读/写）
- 5种获取策略（快速跳过/快速失败/无限等待/超时跳过/超时失败）
- 工厂模式 + 策略模式 + AOP切面的经典组合

### 7.6 策略模式的广泛应用

- **折扣策略**：四种折扣类型各自独立计算逻辑
- **范围策略**：三种作用范围各自独立匹配逻辑
- **锁策略**：五种获取失败的处理策略
- 枚举类充当工厂角色，代码简洁优雅

### 7.7 线程池隔离

两个业务专用线程池，避免相互影响：
- `generateExchangeCodeExecutor`：兑换码生成（2/5/200，CallerRunsPolicy）
- `discountSolutionExecutor`：优惠方案计算（12/12/99999，AbortPolicy）

---

## 八、配置文件

```yaml
# bootstrap.yml
server.port: 8092
spring.application.name: promotion-service
spring.profiles.active: dev

# Nacos共享配置
shared-configs: shared-spring, shared-redis, shared-mybatis,
                shared-logs, shared-feign, shared-xxljob, shared-mq

# Swagger
tj.swagger.package-path: com.tianji.promotion.controller
tj.swagger.title: 天机课堂 - 促销中心接口文档

# 数据库
tj.jdbc.database: tj_promotion

# 认证
tj.auth.resource.enable: true

# MQ消费者重试策略
tj.mq.listener.retry.stateless: false

# AES加密（外部化配置）
tj.promotion.aes.key: ${external}
tj.promotion.aes.iv: ${external}
```

---

## 九、测试

项目包含两个测试类：

1. **CodeUtilTest** -- 兑换码算法测试：对 4000~10000 的序列号批量生成兑换码并解码验证
2. **UserCouponMapperTest** -- 用户优惠券Mapper测试：验证 queryMyCoupons 关联查询
