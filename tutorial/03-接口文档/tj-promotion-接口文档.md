# tj-promotion 促销服务 -- 接口文档

> 服务名：promotion-service | 端口：8092 | 基础路径：/

---

## 一、优惠券管理接口（CouponController）

**路径前缀**：`/coupons`
**Swagger Tag**：优惠券相关接口

---

### 1.1 新增优惠券

- **接口**：`POST /coupons`
- **描述**：新增一张优惠券模板（规则配置），状态默认为待发放(DRAFT)
- **请求体**：`application/json`

**请求参数（CouponFormDTO）**：

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| id | Long | 否 | - | 优惠券id，新增不传，更新必填 |
| name | String | 是 | 长度4~20 | 优惠券名称 |
| specific | Boolean | 否 | - | 是否限定使用范围，true=限定 |
| scopes | List\<Long\> | 条件必填 | specific=true时不能为空 | 使用范围id集合（分类id或课程id） |
| discountType | Integer | 是 | 枚举{1,2,3,4} | 折扣类型：1-每满减 2-折扣 3-无门槛 4-满减 |
| thresholdAmount | Integer | 否 | - | 使用门槛金额（单位：分），0=无门槛 |
| discountValue | Integer | 否 | - | 折扣值：满减填金额，折扣填折扣率(80=8折) |
| maxDiscountAmount | Integer | 否 | - | 最高优惠金额（单位：分） |
| totalNum | Integer | 否 | 范围1~5000 | 优惠券总量 |
| userLimit | Integer | 否 | 范围1~10 | 每人限领数量 |
| obtainWay | Integer | 是 | 枚举{1,2} | 获取方式：1-手动领取 2-兑换码 |

**请求示例**：

```json
{
  "name": "新用户满100减20",
  "specific": true,
  "scopes": [1001, 1002],
  "discountType": 4,
  "thresholdAmount": 10000,
  "discountValue": 2000,
  "maxDiscountAmount": 2000,
  "totalNum": 1000,
  "userLimit": 1,
  "obtainWay": 1
}
```

**响应**：无（`void`，HTTP 200表示成功）

---

### 1.2 分页查询优惠券

- **接口**：`GET /coupons/page`
- **描述**：管理端分页查询优惠券列表，支持按类型、状态、名称筛选

**请求参数（CouponQuery，Query String）**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNo | Integer | 否 | 页码，默认1 |
| pageSize | Integer | 否 | 每页条数，默认10 |
| type | Integer | 否 | 折扣类型：1-每满减 2-折扣 3-无门槛 4-满减 |
| status | Integer | 否 | 状态：1-待发放 2-未开始 3-发放中 4-已结束 5-暂停 |
| name | String | 否 | 优惠券名称（模糊查询） |

**响应**：`PageDTO<CouponPageVO>`

```json
{
  "total": 100,
  "pages": 10,
  "list": [
    {
      "id": 1234567890,
      "name": "新用户满100减20",
      "specific": true,
      "discountType": 4,
      "thresholdAmount": 10000,
      "discountValue": 2000,
      "maxDiscountAmount": 2000,
      "obtainWay": 1,
      "usedNum": 50,
      "issueNum": 200,
      "totalNum": 1000,
      "createTime": "2024-01-01T10:00:00",
      "issueBeginTime": "2024-01-05T00:00:00",
      "issueEndTime": "2024-02-05T23:59:59",
      "termDays": 30,
      "termBeginTime": null,
      "termEndTime": null,
      "status": 3
    }
  ]
}
```

**CouponPageVO 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |
| name | String | 优惠券名称 |
| specific | Boolean | 是否限定范围 |
| discountType | DiscountType | 折扣类型 |
| thresholdAmount | Integer | 门槛金额 |
| discountValue | Integer | 折扣值 |
| maxDiscountAmount | Integer | 最高优惠金额 |
| obtainWay | ObtainType | 获取方式 |
| usedNum | Integer | 已使用数量 |
| issueNum | Integer | 已发放数量 |
| totalNum | Integer | 总量 |
| createTime | LocalDateTime | 创建时间 |
| issueBeginTime | LocalDateTime | 发放开始时间 |
| issueEndTime | LocalDateTime | 发放结束时间 |
| termDays | Integer | 有效天数 |
| termBeginTime | LocalDateTime | 使用有效期开始 |
| termEndTime | LocalDateTime | 使用有效期结束 |
| status | CouponStatus | 状态 |

---

### 1.3 根据ID查询优惠券详情

- **接口**：`GET /coupons/{id}`
- **描述**：查询优惠券详情，包含使用范围信息

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |

**响应**：`CouponDetailVO`

```json
{
  "id": 1234567890,
  "name": "新用户满100减20",
  "scopes": [
    { "id": 1001, "name": "Java课程" },
    { "id": 1002, "name": "Python课程" }
  ],
  "discountType": 4,
  "thresholdAmount": 10000,
  "discountValue": 2000,
  "maxDiscountAmount": 2000,
  "issueBeginTime": "2024-01-05T00:00:00",
  "issueEndTime": "2024-02-05T23:59:59",
  "termDays": 30,
  "termBeginTime": null,
  "termEndTime": null,
  "totalNum": 1000,
  "userLimit": 1,
  "obtainWay": 1
}
```

**CouponDetailVO 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |
| name | String | 优惠券名称 |
| scopes | List\<CouponScopeVO\> | 使用范围列表（id + name） |
| discountType | DiscountType | 折扣类型 |
| thresholdAmount | Integer | 门槛金额 |
| discountValue | Integer | 折扣值 |
| maxDiscountAmount | Integer | 最高优惠 |
| issueBeginTime | LocalDateTime | 发放开始时间 |
| issueEndTime | LocalDateTime | 发放结束时间 |
| termDays | Integer | 有效天数 |
| termBeginTime | LocalDateTime | 使用有效期开始 |
| termEndTime | LocalDateTime | 使用有效期结束 |
| totalNum | Integer | 总量 |
| userLimit | Integer | 每人限领 |
| obtainWay | ObtainType | 获取方式 |

---

### 1.4 发放优惠券

- **接口**：`PUT /coupons/{id}/issue`
- **描述**：开始发放优惠券。如果开始时间为空或已过期则立即发放，否则设为未开始状态等待定时任务触发。对于兑换码类型优惠券（原状态为DRAFT时），会异步生成兑换码。
- **请求体**：`application/json`

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id（路径中传递，实际以请求体为准） |

**请求参数（CouponIssueFormDTO）**：

| 字段 | 类型 | 必填 | 校验规则 | 说明 |
|------|------|------|----------|------|
| id | Long | 是 | - | 优惠券id |
| issueBeginTime | LocalDateTime | 否 | 必须晚于当前时间 | 发放开始时间（为空则立即发放） |
| issueEndTime | LocalDateTime | 是 | 必须晚于当前时间 | 发放结束时间 |
| termDays | Integer | 否 | - | 有效天数（与termBeginTime二选一） |
| termBeginTime | LocalDateTime | 否 | - | 使用有效期开始 |
| termEndTime | LocalDateTime | 否 | - | 使用有效期结束 |

**请求示例**：

```json
{
  "id": 1234567890,
  "issueBeginTime": "2024-01-05T00:00:00",
  "issueEndTime": "2024-02-05T23:59:59",
  "termDays": 30
}
```

**响应**：无（`void`）

**业务规则**：
- 优惠券状态必须为 DRAFT(待发放) 或 PAUSE(暂停)
- 立即发放：状态更新为 ISSUING，缓存到Redis
- 延迟发放：状态更新为 UN_ISSUE，等待XXL-JOB定时任务触发
- 兑换码类型且原状态为DRAFT：异步生成兑换码

---

### 1.5 暂停发放优惠券

- **接口**：`PUT /coupons/{id}/pause`
- **描述**：暂停正在发放或未开始的优惠券

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |

**响应**：无（`void`）

**业务规则**：
- 优惠券状态必须为 UN_ISSUE(未开始) 或 ISSUING(发放中)
- 状态更新为 PAUSE
- 删除Redis中的优惠券缓存

---

### 1.6 查询发放中的优惠券列表

- **接口**：`GET /coupons/list`
- **描述**：用户端查询当前可领取的优惠券列表（状态为发放中、获取方式为手动领取），包含当前用户是否可领取/已领取的标记

**响应**：`List<CouponVO>`

```json
[
  {
    "id": 1234567890,
    "name": "新用户满100减20",
    "specific": true,
    "discountType": 4,
    "thresholdAmount": 10000,
    "discountValue": 2000,
    "maxDiscountAmount": 2000,
    "termDays": 30,
    "termEndTime": null,
    "available": true,
    "received": false
  }
]
```

**CouponVO 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |
| name | String | 名称 |
| specific | Boolean | 是否限定范围 |
| discountType | DiscountType | 折扣类型 |
| thresholdAmount | Integer | 门槛金额 |
| discountValue | Integer | 折扣值 |
| maxDiscountAmount | Integer | 最高优惠 |
| termDays | Integer | 有效天数 |
| termEndTime | LocalDateTime | 使用截止时间 |
| available | Boolean | 是否可领取（库存充足且未达限领数） |
| received | Boolean | 是否已领取未使用 |

---

### 1.7 删除优惠券

- **接口**：`DELETE /coupons/{id}`
- **描述**：删除优惠券（仅允许删除待发放状态的优惠券）

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| id | Long | 优惠券id |

**响应**：无（`void`）

**业务规则**：
- 仅 DRAFT(待发放) 状态可删除
- 同时删除关联的优惠券作用范围记录（coupon_scope）

---

## 二、兑换码管理接口（ExchangeCodeController）

**路径前缀**：`/codes`

---

### 2.1 分页查询兑换码

- **接口**：`GET /codes/page`
- **描述**：分页查询指定优惠券的兑换码列表

**请求参数（CodeQuery，Query String）**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNo | Integer | 否 | 页码，默认1 |
| pageSize | Integer | 否 | 每页条数 |
| couponId | Long | 是 | 兑换码对应的优惠券id |
| status | Integer | 是 | 兑换码状态：1-未兑换 2-已兑换 |

**响应**：`PageDTO<ExchangeCodeVO>`

```json
{
  "total": 500,
  "pages": 50,
  "list": [
    {
      "id": 10001,
      "code": "ABCDE12345"
    }
  ]
}
```

**ExchangeCodeVO 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer | 兑换码序列号 |
| code | String | 兑换码字符串（10位Base32编码） |

---

## 三、用户优惠券接口（UserCouponController）

**路径前缀**：`/user-coupons`
**Swagger Tag**：优惠券相关接口

---

### 3.1 领取优惠券

- **接口**：`POST /user-coupons/{couponId}/receive`
- **描述**：用户手动领取优惠券。通过Redis LUA脚本原子校验（优惠券存在性、库存、发放时间、限领数量），校验通过后发送MQ消息异步写库。

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| couponId | Long | 优惠券id |

**响应**：无（`void`）

**业务规则**：
- 优惠券必须存在且已缓存到Redis
- 优惠券库存（totalNum）必须大于0
- 当前时间必须在发放时间范围内
- 用户领取次数不能超过 userLimit 限制
- 校验和扣减在LUA脚本中原子完成
- 成功后发送MQ消息异步创建 UserCoupon 记录

**可能的异常**：
- 活动未开始
- 库存不足
- 活动已经结束
- 领取次数过多

---

### 3.2 兑换码兑换优惠券

- **接口**：`POST /user-coupons/{code}/exchange`
- **描述**：用户通过兑换码兑换优惠券。先解析兑换码获取序列号，再通过LUA脚本原子校验兑换状态和优惠券有效性。

**路径参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| code | String | 兑换码（10位Base32编码字符串） |

**响应**：无（`void`）

**业务规则**：
- 兑换码格式校验（Base32解码+校验码验证）
- Bitmap检查兑换码是否已使用
- ZSet查找兑换码对应的优惠券ID
- 校验优惠券缓存存在、未过期、未超领
- 标记Bitmap为已兑换
- 发送MQ消息异步创建UserCoupon并更新ExchangeCode状态

**可能的异常**：
- 无效兑换码（格式错误或校验码不匹配）
- 兑换码已兑换
- 无效兑换码（未找到对应优惠券）
- 活动未开始
- 活动已经结束
- 领取次数过多

---

### 3.3 分页查询我的优惠券

- **接口**：`GET /user-coupons/page`
- **描述**：分页查询当前登录用户的优惠券列表

**请求参数（UserCouponQuery，Query String）**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |
| status | Integer | 是 | 状态：1-未使用 2-已使用 3-已过期 |

**响应**：`PageDTO<CouponVO>`

```json
{
  "total": 5,
  "pages": 1,
  "list": [
    {
      "id": 1234567890,
      "name": "新用户满100减20",
      "specific": true,
      "discountType": 4,
      "thresholdAmount": 10000,
      "discountValue": 2000,
      "maxDiscountAmount": 2000,
      "termDays": 30,
      "termEndTime": "2024-03-01T23:59:59",
      "available": null,
      "received": null
    }
  ]
}
```

**说明**：按使用截止时间（termEndTime）升序排列。

---

### 3.4 查询优惠券可用方案

- **接口**：`POST /user-coupons/available`
- **描述**：根据订单中的课程信息，查询当前用户所有可用的优惠券叠加方案，返回按优惠金额降序排列的最优方案列表。
- **请求体**：`application/json`

**请求参数**：`List<OrderCourseDTO>`

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 课程id |
| price | Integer | 课程价格（单位：分） |
| cateId | Long | 课程三级分类id |

**请求示例**：

```json
[
  { "id": 101, "price": 9900, "cateId": 1001 },
  { "id": 102, "price": 19900, "cateId": 1002 }
]
```

**响应**：`List<CouponDiscountDTO>`

```json
[
  {
    "ids": [100001, 100002],
    "rules": ["满100减20", "满200打9折"],
    "discountAmount": 4980,
    "discountDetail": {
      "101": 1660,
      "102": 3320
    }
  },
  {
    "ids": [100001],
    "rules": ["满100减20"],
    "discountAmount": 2000,
    "discountDetail": {
      "101": 667,
      "102": 1333
    }
  }
]
```

**CouponDiscountDTO 字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| ids | List\<Long\> | 用户优惠券id列表（UserCoupon.id） |
| rules | List\<String\> | 优惠规则描述（如"满100减20"、"满200打8折，上限50元"） |
| discountAmount | Integer | 总优惠金额（单位：分） |
| discountDetail | Map\<Long, Integer\> | 课程级别的优惠明细（key=课程id，value=该课程优惠金额） |

**算法说明**：
1. 查询用户所有未使用的优惠券
2. 按订单总价初筛，再按优惠券范围细筛
3. 全排列生成叠加方案 + 单券方案
4. 多线程并行计算每种方案的优惠明细
5. 双维度筛选最优解：同券组合取最大优惠，同优惠额取最少用券
6. 按优惠金额降序排列返回

---

### 3.5 根据券方案计算订单优惠明细

- **接口**：`POST /user-coupons/discount`
- **描述**：用户选定优惠券方案后，精确计算该方案下的订单优惠明细。供订单服务在下单时调用。
- **请求体**：`application/json`

**请求参数（OrderCouponDTO）**：

| 字段 | 类型 | 说明 |
|------|------|------|
| userCouponIds | List\<Long\> | 用户优惠券id列表 |
| courseList | List\<OrderCourseDTO\> | 订单课程列表 |

**请求示例**：

```json
{
  "userCouponIds": [100001, 100002],
  "courseList": [
    { "id": 101, "price": 9900, "cateId": 1001 },
    { "id": 102, "price": 19900, "cateId": 1002 }
  ]
}
```

**响应**：`CouponDiscountDTO`（同3.4响应格式中的单个对象）

---

### 3.6 核销优惠券

- **接口**：`PUT /user-coupons/use`
- **描述**：核销指定的用户优惠券（用户下单成功后调用）

**请求参数（Query String）**：

| 参数 | 类型 | 说明 |
|------|------|------|
| couponIds | List\<Long\> | 用户优惠券id集合（逗号分隔） |

**示例**：`PUT /user-coupons/use?couponIds=100001,100002`

**响应**：无（`void`）

**业务规则**：
- 过滤有效券：状态为 UNUSED 且在有效期内
- 更新 UserCoupon 状态为 USED
- 更新 Coupon 的 usedNum（已使用数量）+ 1

---

### 3.7 退还优惠券

- **接口**：`PUT /user-coupons/refund`
- **描述**：退还已核销的优惠券（用户退款时调用）

**请求参数（Query String）**：

| 参数 | 类型 | 说明 |
|------|------|------|
| couponIds | List\<Long\> | 用户优惠券id集合（逗号分隔） |

**示例**：`PUT /user-coupons/refund?couponIds=100001,100002`

**响应**：无（`void`）

**业务规则**：
- 过滤已使用（USED）状态的券
- 判断有效期：未过期恢复为 UNUSED，已过期设为 EXPIRED
- 更新 Coupon 的 usedNum（已使用数量）- 1

---

### 3.8 查询优惠券折扣规则

- **接口**：`GET /user-coupons/rules`
- **描述**：查询指定用户优惠券的折扣规则描述文本

**请求参数（Query String）**：

| 参数 | 类型 | 说明 |
|------|------|------|
| couponIds | List\<Long\> | 用户优惠券id集合（逗号分隔） |

**示例**：`GET /user-coupons/rules?couponIds=100001,100002`

**响应**：`List<String>`

```json
[
  "满100减20",
  "满200打8折，上限50元"
]
```

**折扣规则描述格式**：

| 折扣类型 | 规则描述格式 |
|----------|-------------|
| 无门槛(3) | `无门槛抵{x}元` |
| 满减(4) | `满{x}减{y}` |
| 每满减(1) | `每满{x}减{y}，上限{z}` |
| 折扣(2) | `满{x}打{y}折，上限{z}元` |

---

## 四、接口调用关系总结

### 4.1 管理端接口（面向管理后台）

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /coupons | 新增优惠券 |
| GET | /coupons/page | 分页查询优惠券 |
| GET | /coupons/{id} | 查询优惠券详情 |
| PUT | /coupons/{id}/issue | 发放优惠券 |
| PUT | /coupons/{id}/pause | 暂停发放 |
| DELETE | /coupons/{id} | 删除优惠券 |
| GET | /codes/page | 分页查询兑换码 |

### 4.2 用户端接口（面向前端App/H5）

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /coupons/list | 查询可领取优惠券列表 |
| POST | /user-coupons/{couponId}/receive | 领取优惠券 |
| POST | /user-coupons/{code}/exchange | 兑换码兑换 |
| GET | /user-coupons/page | 查询我的优惠券 |

### 4.3 服务间调用接口（面向其他微服务，如订单服务）

| 方法 | 路径 | 功能 | 调用方 |
|------|------|------|--------|
| POST | /user-coupons/available | 查询可用优惠方案 | 订单服务（结算页） |
| POST | /user-coupons/discount | 计算优惠明细 | 订单服务（下单时） |
| PUT | /user-coupons/use | 核销优惠券 | 订单服务（下单成功后） |
| PUT | /user-coupons/refund | 退还优惠券 | 订单服务（退款时） |
| GET | /user-coupons/rules | 查询折扣规则 | 订单服务（订单详情页） |

---

## 五、枚举值速查

### 折扣类型（discountType）

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | PER_PRICE_DISCOUNT | 每满减 |
| 2 | RATE_DISCOUNT | 折扣 |
| 3 | NO_THRESHOLD | 无门槛 |
| 4 | PRICE_DISCOUNT | 满减 |

### 优惠券状态（status）

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | DRAFT | 待发放 |
| 2 | UN_ISSUE | 未开始 |
| 3 | ISSUING | 发放中 |
| 4 | FINISHED | 已结束 |
| 5 | PAUSE | 暂停 |

### 获取方式（obtainWay）

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | PUBLIC | 手动领取 |
| 2 | ISSUE | 兑换码兑换 |

### 用户优惠券状态（UserCoupon status）

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | UNUSED | 未使用 |
| 2 | USED | 已使用 |
| 3 | EXPIRED | 已过期 |

### 兑换码状态（ExchangeCode status）

| 值 | 名称 | 说明 |
|----|------|------|
| 1 | UNUSED | 待兑换 |
| 2 | USED | 已兑换 |
| 3 | EXPIRED | 活动已结束 |
