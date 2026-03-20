# tj-trade 交易服务接口文档

## 一、购物车接口

### 1.1 添加课程到购物车
- **路径：** `POST /carts`
- **请求参数：**
  ```json
  {
    "courseId": 1  // 课程ID，必填
  }
  ```
- **返回值：** 无（void）
- **功能说明：** 将指定课程添加到当前用户的购物车。校验课程是否已在购物车、购物车是否已满（默认上限10个）、课程是否存在且未过期。

### 1.2 获取购物车中的课程
- **路径：** `GET /carts`
- **请求参数：** 无
- **返回值：** `List<CartVO>`
  ```json
  [
    {
      "id": 1,           // 购物车条目ID
      "courseId": 100,    // 课程ID
      "courseName": "Java基础",  // 课程名称
      "coverUrl": "http://...",   // 课程封面
      "price": 9900,      // 原价（分）
      "nowPrice": 9900,   // 当前价格（分）
      "expired": false,   // 是否过期
      "courseValidDate": "2025-12-31T00:00:00"  // 课程有效期
    }
  ]
  ```
- **功能说明：** 获取当前用户购物车中的所有课程，并查询最新课程价格和有效期。未过期的排在前面，新添加的排在前面。

### 1.3 删除指定购物车条目
- **路径：** `DELETE /carts/{id}`
- **请求参数：** `id`（路径参数）- 购物车条目ID
- **返回值：** 无
- **功能说明：** 删除当前用户购物车中的指定条目，校验归属。

### 1.4 批量删除购物车条目
- **路径：** `DELETE /carts?ids=1,2,3`
- **请求参数：** `ids`（查询参数）- 购物车条目ID集合
- **返回值：** 无
- **功能说明：** 批量删除购物车条目。

---

## 二、订单接口

### 2.1 分页查询我的订单
- **路径：** `GET /orders/page`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | pageNo | Integer | 否 | 页码，默认1 |
  | pageSize | Integer | 否 | 每页数量，默认10 |
  | status | Integer | 否 | 订单状态过滤 |
- **返回值：** `PageDTO<OrderPageVO>`
  ```json
  {
    "total": 100,
    "pages": 10,
    "list": [
      {
        "id": 1,
        "totalAmount": 9900,
        "realAmount": 8900,
        "status": 2,
        "statusDesc": "已支付",
        "createTime": "2024-01-01T12:00:00",
        "details": [
          {
            "id": 1,
            "courseId": 100,
            "name": "Java基础",
            "coverUrl": "http://...",
            "price": 9900,
            "realPayAmount": 8900
          }
        ]
      }
    ]
  }
  ```
- **功能说明：** 分页查询当前用户的订单列表，包含订单明细信息。

### 2.2 根据ID查询订单详情
- **路径：** `GET /orders/{id}`
- **请求参数：** `id`（路径参数）- 订单ID
- **返回值：** `OrderVO`（含订单详情、进度节点、优惠明细等完整信息）
- **功能说明：** 查询订单完整详情，包括订单基本信息、订单明细列表、订单进度节点和优惠券使用明细。

### 2.3 查询订单支付状态
- **路径：** `GET /orders/{id}/status`
- **请求参数：** `id`（路径参数）- 订单ID
- **返回值：** `PlaceOrderResultVO`
  ```json
  {
    "orderId": 1,
    "payAmount": 8900,
    "status": 1,
    "payOutTime": "2024-01-01T12:30:00"  // 支付超时时间
  }
  ```
- **功能说明：** 查询订单的当前支付状态和剩余支付时间。

### 2.4 预下单接口
- **路径：** `GET /orders/prePlaceOrder?courseIds=1,2,3`
- **请求参数：** `courseIds`（查询参数）- 课程ID列表
- **返回值：** `OrderConfirmVO`
  ```json
  {
    "orderId": 123456789,
    "totalAmount": 19800,
    "courses": [...],
    "discounts": [...]  // 可用优惠券方案
  }
  ```
- **功能说明：** 生成订单ID，计算总价，查询可用优惠券方案，返回确认订单信息。

### 2.5 下单接口
- **路径：** `POST /orders/placeOrder`
- **请求参数：**
  ```json
  {
    "orderId": 123456789,  // 预下单生成的订单ID
    "courseIds": [1, 2],   // 课程ID列表
    "couponIds": [10]      // 使用的优惠券ID列表（可选）
  }
  ```
- **返回值：** `PlaceOrderResultVO`
- **功能说明：** 创建订单，计算优惠金额，写入数据库，删除购物车，核销优惠券。

### 2.6 免费课立刻报名
- **路径：** `POST /orders/freeCourse/{courseId}`
- **请求参数：** `courseId`（路径参数）- 免费课程ID
- **返回值：** `PlaceOrderResultVO`
- **功能说明：** 免费课程一键报名，创建零元订单并直接完成报名，发送MQ消息通知报名成功。

### 2.7 取消订单
- **路径：** `PUT /orders/{id}/cancel`
- **请求参数：** `id`（路径参数）- 订单ID
- **返回值：** 无
- **功能说明：** 取消未支付订单，更新状态为已关闭，退还优惠券。

### 2.8 删除订单
- **路径：** `DELETE /orders/{id}`
- **请求参数：** `id`（路径参数）- 订单ID
- **返回值：** 无
- **功能说明：** 逻辑删除订单，仅允许删除自己的订单。

---

## 三、订单明细接口

### 3.1 分页查询订单明细
- **路径：** `GET /order-details/page`
- **请求参数：** 分页参数 + 筛选条件
- **返回值：** `PageDTO<OrderDetailPageVO>`
- **功能说明：** 管理端分页查询订单明细。

### 3.2 根据ID获取订单明细详情
- **路径：** `GET /order-details/{id}`
- **请求参数：** `id`（路径参数）- 订单明细ID
- **返回值：** `OrderDetailAdminVO`
- **功能说明：** 获取订单明细的完整信息和进度。

### 3.3 校验课程是否购买
- **路径：** `GET /order-details/course/{id}`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** `Boolean`（true表示已购买且未过期）
- **功能说明：** 内部调用接口，校验指定课程是否已购买且在有效期内。

### 3.4 统计课程报名人数
- **路径：** `GET /order-details/enrollNum?courseIdList=1,2,3`
- **请求参数：** `courseIdList`（查询参数）- 课程ID列表
- **返回值：** `Map<Long, Integer>`（课程ID -> 报名人数）
- **功能说明：** 内部调用接口，批量统计课程报名人数。

### 3.5 统计学生报名课程数量
- **路径：** `GET /order-details/enrollCourse?studentIds=1,2,3`
- **请求参数：** `studentIds`（查询参数）- 学生ID列表
- **返回值：** `Map<Long, Integer>`（学生ID -> 报名课程数）
- **功能说明：** 内部调用接口，批量统计学生报名的课程数量。

### 3.6 查询课程购买信息
- **路径：** `GET /order-details/purchaseInfo?courseId=1`
- **请求参数：** `courseId`（查询参数）- 课程ID
- **返回值：** `CoursePurchaseInfoDTO`
- **功能说明：** 内部调用接口，获取课程的购买统计信息。

---

## 四、支付接口

### 4.1 支付申请
- **路径：** `POST /pay/order`
- **请求参数：**
  ```json
  {
    "orderId": 123456789,     // 订单ID
    "payChannelCode": "aliPay"  // 支付渠道代码
  }
  ```
- **返回值：** `String`（支付二维码URL）
- **功能说明：** 发起支付申请，校验订单状态和超时，调用支付服务生成支付二维码，同时发送延迟消息轮询支付状态。

### 4.2 获取支付渠道列表
- **路径：** `GET /pay/channels`
- **请求参数：** 无
- **返回值：** `List<PayChannelVO>`
  ```json
  [
    {
      "channelCode": "aliPay",
      "channelName": "支付宝",
      "icon": "http://..."
    }
  ]
  ```
- **功能说明：** 获取当前可用的支付渠道列表。

---

## 五、退款接口

### 5.1 退款申请
- **路径：** `POST /refund-apply`
- **请求参数：**
  ```json
  {
    "orderDetailId": 1,      // 订单明细ID
    "refundReason": "不想学了",  // 退款原因
    "questionDesc": "..."     // 问题描述
  }
  ```
- **返回值：** 无
- **功能说明：** 提交退款申请。学员提交后需等待审批，管理员提交则直接发起退款。学员同一子订单最多申请2次退款。

### 5.2 审批退款申请
- **路径：** `PUT /refund-apply/approval`
- **请求参数：**
  ```json
  {
    "id": 1,               // 退款申请ID
    "approveType": 1,      // 1同意，2拒绝
    "approveOpinion": "同意退款",
    "remark": "..."
  }
  ```
- **返回值：** 无
- **功能说明：** 管理员审批退款申请，同意则异步发起退款请求。

### 5.3 取消退款申请
- **路径：** `PUT /refund-apply/cancel`
- **请求参数：**
  ```json
  {
    "id": 1,              // 退款申请ID（或 orderDetailId）
    "orderDetailId": 1
  }
  ```
- **返回值：** 无
- **功能说明：** 取消待审批的退款申请。

### 5.4 分页查询退款申请
- **路径：** `GET /refund-apply/page`
- **请求参数：** 分页参数 + 筛选条件（退款状态、申请时间范围、手机号等）
- **返回值：** `PageDTO<RefundApplyPageVO>`
- **功能说明：** 管理端分页查询退款申请列表。

### 5.5 根据ID查询退款详情
- **路径：** `GET /refund-apply/{id}`
- **请求参数：** `id`（路径参数）- 退款ID
- **返回值：** `RefundApplyVO`
- **功能说明：** 查询退款申请的完整详情，包含订单信息、用户信息等。

### 5.6 根据子订单ID查询退款详情
- **路径：** `GET /refund-apply/detail/{id}`
- **请求参数：** `id`（路径参数）- 子订单ID
- **返回值：** `RefundApplyVO`
- **功能说明：** 根据子订单ID查询最近一次退款申请详情。

### 5.7 查询下一个待审批的退款申请
- **路径：** `GET /refund-apply/next`
- **请求参数：** 无
- **返回值：** `RefundApplyVO`
- **功能说明：** 获取一个待审批的退款申请，用于审批工作流。
