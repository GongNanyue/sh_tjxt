# tj-learning 学习服务 - 接口文档

> 服务名称：`learning-service` | 端口：`8090` | 基础路径：无统一前缀

---

## 一、我的课表相关接口

### 1.1 分页查询我的课表

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/page` |
| 接口说明 | 分页查询当前用户的课表列表，按最近学习时间倒序排列 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| pageNo | Integer | 否 | 页码，默认1 |
| pageSize | Integer | 否 | 每页条数，默认10 |
| sortBy | String | 否 | 排序字段 |
| isAsc | Boolean | 否 | 是否升序 |

**返回值**：`PageDTO<LearningLessonVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "课表ID",
      "courseId": "课程ID",
      "courseName": "课程名称",
      "courseCoverUrl": "课程封面URL",
      "sections": "课程章节总数",
      "status": "课程状态：0-未学习，1-学习中，2-已学完，3-已失效",
      "learnedSections": "已学习章节数",
      "createTime": "购买时间",
      "expireTime": "过期时间，null代表永久有效",
      "weekFreq": "每周计划学习频率",
      "planStatus": "学习计划状态：0-没有计划，1-计划进行中"
    }
  ]
}
```

---

### 1.2 查询我正在学习的课程

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/now` |
| 接口说明 | 查询当前用户最近正在学习的一门课程，包含课程详情和最近学习的小节信息 |
| 权限要求 | 需要登录 |

**请求参数**：无

**返回值**：`LearningLessonVO`

```json
{
  "id": "课表ID",
  "courseId": "课程ID",
  "courseName": "课程名称",
  "courseCoverUrl": "课程封面URL",
  "sections": "课程章节总数",
  "status": 1,
  "learnedSections": "已学习章节数",
  "courseAmount": "总已报名课程数",
  "createTime": "购买时间",
  "expireTime": "过期时间",
  "latestSectionName": "最近学习的小节名称",
  "latestSectionIndex": "最近学习的小节编号"
}
```

---

### 1.3 查询指定课程信息

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/{courseId}` |
| 接口说明 | 根据课程ID查询当前用户该课程的课表信息 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |

**返回值**：`LearningLessonVO`

返回该课程在课表中的基本信息，包含 id, courseId, status, learnedSections, weekFreq, planStatus 等字段。

---

### 1.4 删除指定课程

| 属性 | 值 |
|------|-----|
| 接口路径 | `DELETE /lessons/{courseId}` |
| 接口说明 | 从当前用户的课表中删除指定课程 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |

**返回值**：无（void）

---

### 1.5 统计课程学习人数

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/{courseId}/count` |
| 接口说明 | 统计指定课程的学习人数（状态为未学习、学习中、已学完的用户数） |
| 权限要求 | 无需登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |

**返回值**：`Integer` -- 学习人数

---

### 1.6 校验当前课程是否已经报名

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/{courseId}/valid` |
| 接口说明 | 校验当前用户是否已报名指定课程，返回课表ID |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |

**返回值**：`Long` -- 课表ID。如果未报名返回null。

---

### 1.7 创建学习计划

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /lessons/plans` |
| 接口说明 | 为指定课程创建学习计划，设置每周学习频率 |
| 权限要求 | 需要登录 |

**请求参数（Body - JSON）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID，最小值1 |
| freq | Integer | 是 | 每周学习频率，范围1~50 |

**请求示例**：
```json
{
  "courseId": 1,
  "freq": 6
}
```

**返回值**：无（void）

---

### 1.8 查询我的学习计划

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /lessons/plans` |
| 接口说明 | 分页查询当前用户的学习计划，包含本周学习统计 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| pageNo | Integer | 否 | 页码，默认1 |
| pageSize | Integer | 否 | 每页条数，默认10 |

**返回值**：`LearningPlanPageVO`

```json
{
  "weekPoints": "本周积分值",
  "weekFinished": "本周完成的计划数量",
  "weekTotalPlan": "总的计划学习数量",
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "课表ID",
      "courseId": "课程ID",
      "courseName": "课程名称",
      "weekFreq": "每周计划学习章节数",
      "sections": "课程章节总数",
      "weekLearnedSections": "本周已学习章节数",
      "learnedSections": "总已学习章节数",
      "latestLearnTime": "最近一次学习时间"
    }
  ]
}
```

---

## 二、学习记录相关接口

### 2.1 查询指定课程的学习记录

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /learning-records/course/{courseId}` |
| 接口说明 | 查询当前用户在指定课程下的所有学习记录（每个小节的学习进度） |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |

**返回值**：`LearningLessonDTO`

```json
{
  "id": "课表ID",
  "latestSectionId": "最近学习的小节ID",
  "records": [
    {
      "sectionId": "小节ID",
      "finished": true,
      "moment": 120
    }
  ]
}
```

---

### 2.2 提交学习记录

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /learning-records` |
| 接口说明 | 提交视频播放进度或考试记录。视频类型会记录播放进度，超过50%视为完成。 |
| 权限要求 | 需要登录 |

**请求参数（Body - JSON）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| sectionType | Integer | 是 | 小节类型：1-视频，2-考试 |
| lessonId | Long | 是 | 课表ID |
| sectionId | Long | 是 | 小节ID |
| duration | Integer | 否 | 视频总时长（秒），视频类型必填 |
| moment | Integer | 否 | 视频当前观看时长（秒），视频类型必填，第一次提交填0 |
| commitTime | LocalDateTime | 否 | 提交时间 |

**请求示例**：
```json
{
  "sectionType": 1,
  "lessonId": 1234567890,
  "sectionId": 9876543210,
  "duration": 600,
  "moment": 320,
  "commitTime": "2024-01-15 10:30:00"
}
```

**返回值**：无（void）

**业务规则**：
- 视频类型：moment * 2 >= duration 时判定为完成
- 考试类型：提交即完成
- 小节首次完成后会更新课表的学习进度

---

## 三、互动问答相关接口（用户端）

### 3.1 新增互动问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /questions` |
| 接口说明 | 在指定课程的小节下新增互动问题 |
| 权限要求 | 需要登录 |

**请求参数（Body - JSON）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 是 | 课程ID |
| chapterId | Long | 是 | 章ID |
| sectionId | Long | 是 | 小节ID |
| title | String | 是 | 问题标题，长度1~254 |
| description | String | 是 | 问题描述 |
| anonymity | Boolean | 否 | 是否匿名提问 |

**返回值**：无（void）

---

### 3.2 修改提问

| 属性 | 值 |
|------|-----|
| 接口路径 | `PUT /questions/{id}` |
| 接口说明 | 修改自己的提问内容，不能修改他人问题 |
| 权限要求 | 需要登录 |

**请求参数**：

| 参数名 | 位置 | 类型 | 必填 | 说明 |
|--------|------|------|------|------|
| id | Path | Long | 是 | 问题ID |
| (body) | Body | QuestionFormDTO | 是 | 同新增问题参数 |

**返回值**：无（void）

---

### 3.3 分页查询互动问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /questions/page` |
| 接口说明 | 分页查询指定课程或小节下的互动问题，自动过滤隐藏问题 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 否 | 课程ID（与sectionId至少传一个） |
| sectionId | Long | 否 | 小节ID（与courseId至少传一个） |
| onlyMine | Boolean | 否 | 是否只查询我的问题，默认false |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PageDTO<QuestionVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "问题ID",
      "title": "问题标题",
      "description": "问题描述",
      "answerTimes": "回答数量",
      "createTime": "创建时间",
      "anonymity": false,
      "userId": "提问者ID",
      "userName": "提问者昵称（匿名时为null）",
      "userIcon": "提问者头像（匿名时为null）",
      "latestReplyContent": "最新回答内容",
      "latestReplyUser": "最新回答者昵称"
    }
  ]
}
```

---

### 3.4 根据ID查询互动问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /questions/{id}` |
| 接口说明 | 根据ID查询问题详情，隐藏的问题返回null |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 问题ID |

**返回值**：`QuestionVO` -- 同3.3中的列表项结构

---

### 3.5 删除当前用户问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `DELETE /questions/{id}` |
| 接口说明 | 删除自己的问题，同时级联删除该问题下的所有回答和评论 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 问题ID |

**返回值**：无（void）

---

## 四、互动问答相关接口（管理端）

### 4.1 管理端分页查询互动问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/questions/page` |
| 接口说明 | 管理端分页查询互动问题，支持多条件筛选 |
| 权限要求 | 管理员 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseName | String | 否 | 课程名称搜索关键字 |
| status | Integer | 否 | 管理端问题状态：0-未查看，1-已查看 |
| beginTime | LocalDateTime | 否 | 更新时间区间开始（格式：yyyy-MM-dd HH:mm:ss） |
| endTime | LocalDateTime | 否 | 更新时间区间结束 |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PageDTO<QuestionAdminVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "问题ID",
      "title": "问题标题",
      "description": "问题描述",
      "answerTimes": "回答数量",
      "createTime": "创建时间",
      "status": "0-未查看，1-已查看",
      "hidden": false,
      "userName": "提问者昵称",
      "userIcon": "提问者头像",
      "courseName": "课程名称",
      "teacherName": "教师名称",
      "chapterName": "章名称",
      "sectionName": "节名称",
      "categoryName": "三级分类名称（/分隔）"
    }
  ]
}
```

---

### 4.2 管理端根据ID查询互动问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/questions/{id}` |
| 接口说明 | 管理端查询问题详情，包含教师信息、课程分类等完整信息 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 问题ID |

**返回值**：`QuestionAdminVO` -- 同4.1中的列表项结构

---

### 4.3 隐藏或显示问题

| 属性 | 值 |
|------|-----|
| 接口路径 | `PUT /admin/questions/{id}/hidden/{hidden}` |
| 接口说明 | 设置问题的隐藏/显示状态 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 问题ID |
| hidden | Boolean | 是 | true-隐藏，false-显示 |

**返回值**：无（void）

---

## 五、互动回答/评论相关接口（用户端）

### 5.1 新增回答或评论

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /replies` |
| 接口说明 | 新增回答或评论。answerId为空时表示回答，不为空时表示评论 |
| 权限要求 | 需要登录 |

**请求参数（Body - JSON）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| content | String | 是 | 回答内容 |
| anonymity | Boolean | 否 | 是否匿名 |
| questionId | Long | 是 | 互动问题ID |
| answerId | Long | 否 | 回复的上级回答ID（评论时必填） |
| targetReplyId | Long | 否 | 回复的目标回复ID |
| targetUserId | Long | 否 | 回复的目标用户ID |
| isStudent | Boolean | 否 | 是否学生提交，默认true |

**请求示例**：
```json
{
  "content": "这个问题可以这样理解...",
  "anonymity": false,
  "questionId": 12345,
  "isStudent": true
}
```

**返回值**：无（void）

**业务规则**：
- 回答会累加问题的回答数并更新最新回答ID
- 评论会累加上级回答的评论数
- 学生回答后异步发放5积分

---

### 5.2 分页查询回答或评论

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /replies/page` |
| 接口说明 | 分页查询回答或评论，支持按问题查回答或按回答查评论 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| questionId | Long | 否 | 问题ID，不为空则查询该问题下的回答（与answerId至少传一个） |
| answerId | Long | 否 | 回答ID，不为空则查询该回答下的评论（与questionId至少传一个） |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PageDTO<ReplyVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "回答/评论ID",
      "content": "回答内容",
      "anonymity": false,
      "hidden": false,
      "replyTimes": "评论数量",
      "createTime": "创建时间",
      "userId": "回复者ID",
      "userName": "回复者昵称",
      "userIcon": "回复者头像",
      "userType": "回复者类型：2-学员，其它-老师",
      "liked": true,
      "likedTimes": 15,
      "targetUserName": "目标用户名字"
    }
  ]
}
```

**排序规则**：先按点赞数倒序，点赞数相同则按创建时间正序。

---

## 六、互动回答/评论相关接口（管理端）

### 6.1 隐藏或显示评论

| 属性 | 值 |
|------|-----|
| 接口路径 | `PUT /admin/replies/{id}/hidden/{hidden}` |
| 接口说明 | 隐藏或显示评论/回答。如果隐藏的是回答，其下所有评论也会被隐藏 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 回答/评论ID |
| hidden | Boolean | 是 | true-隐藏，false-显示 |

**返回值**：无（void）

---

### 6.2 管理端分页查询回答或评论

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/replies/page` |
| 接口说明 | 管理端分页查询回答或评论，可查看被隐藏的内容，匿名用户也会显示真实信息 |
| 权限要求 | 管理员 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| questionId | Long | 否 | 问题ID |
| answerId | Long | 否 | 回答ID |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PageDTO<ReplyVO>` -- 同5.2返回值结构

---

### 6.3 根据ID查询回答或评论

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/replies/{id}` |
| 接口说明 | 根据ID查询回答或评论的详细信息 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 回答/评论ID |

**返回值**：`ReplyVO` -- 同5.2中的列表项结构

---

## 七、笔记相关接口（用户端）

### 7.1 新增笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /notes` |
| 接口说明 | 在指定课程小节下新增学习笔记 |
| 权限要求 | 需要登录 |

**请求参数（Body - JSON）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| content | String | 是 | 笔记内容 |
| noteMoment | Integer | 是 | 记录笔记时的视频播放时间（秒） |
| isPrivate | Boolean | 是 | 是否隐私笔记 |
| courseId | Long | 是 | 课程ID |
| chapterId | Long | 是 | 章ID |
| sectionId | Long | 是 | 小节ID |

**返回值**：无（void）

**业务规则**：新增笔记后异步发放3积分

---

### 7.2 采集笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /notes/gathers/{id}` |
| 接口说明 | 采集他人的公开笔记到自己的笔记本，采集的笔记强制设为私密 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 被采集的笔记ID |

**返回值**：无（void）

**业务规则**：
- 不能采集私密笔记和已隐藏笔记
- 采集后异步发放2积分

---

### 7.3 取消采集笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `DELETE /notes/gathers/{id}` |
| 接口说明 | 取消采集笔记 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 原始笔记ID |

**返回值**：无（void）

---

### 7.4 更新笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `PUT /notes/{id}` |
| 接口说明 | 更新自己的笔记内容。采集笔记不能设置为公开。 |
| 权限要求 | 需要登录 |

**请求参数**：

| 参数名 | 位置 | 类型 | 必填 | 说明 |
|--------|------|------|------|------|
| id | Path | Long | 是 | 笔记ID |
| (body) | Body | NoteFormDTO | 是 | 笔记内容等信息 |

**返回值**：无（void）

---

### 7.5 删除我的笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `DELETE /notes/{id}` |
| 接口说明 | 删除自己的笔记，不能删除他人笔记 |
| 权限要求 | 需要登录 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 笔记ID |

**返回值**：无（void）

---

### 7.6 用户端分页查询笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /notes/page` |
| 接口说明 | 分页查询笔记列表，支持按课程/小节/仅我的笔记筛选 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| courseId | Long | 否 | 课程ID（与sectionId至少传一个） |
| sectionId | Long | 否 | 小节ID（与courseId至少传一个） |
| onlyMine | Boolean | 否 | 是否只查询我的笔记，默认false |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PageDTO<NoteVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "笔记ID",
      "content": "笔记内容",
      "noteMoment": "视频播放时间点（秒）",
      "isPrivate": false,
      "isGathered": false,
      "authorId": "作者ID",
      "authorName": "作者名字",
      "authorIcon": "作者头像",
      "createTime": "发布时间"
    }
  ]
}
```

---

## 八、笔记相关接口（管理端）

### 8.1 管理端分页查询笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/notes/page` |
| 接口说明 | 管理端分页查询笔记，支持按课程名称、隐藏状态、时间筛选 |
| 权限要求 | 管理员 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| name | String | 否 | 搜索关键字（课程名称） |
| hidden | Boolean | 否 | 笔记是否隐藏 |
| beginTime | LocalDateTime | 否 | 开始时间 |
| endTime | LocalDateTime | 否 | 结束时间 |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |
| sortBy | String | 否 | 排序字段 |
| isAsc | Boolean | 否 | 是否升序 |

**返回值**：`PageDTO<NoteAdminVO>`

```json
{
  "total": 10,
  "pages": 2,
  "list": [
    {
      "id": "笔记ID",
      "courseName": "课程名称",
      "chapterName": "章名称",
      "sectionName": "节名称",
      "content": "笔记内容",
      "hidden": false,
      "authorName": "作者名称",
      "createTime": "发布时间"
    }
  ]
}
```

---

### 8.2 管理端查询笔记详情

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /admin/notes/{id}` |
| 接口说明 | 查询笔记详情，包含完整的课程分类、作者联系方式、采集者信息 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 笔记ID |

**返回值**：`NoteAdminDetailVO`

```json
{
  "id": "笔记ID",
  "courseName": "课程名称",
  "chapterName": "章名称",
  "sectionName": "节名称",
  "categoryNames": "课程分类名称（/拼接）",
  "content": "笔记内容",
  "noteMoment": "视频播放时间点（秒）",
  "usedTimes": "被采集次数",
  "hidden": false,
  "authorName": "作者名称",
  "authorPhone": "作者手机号",
  "createTime": "发布时间",
  "gathers": ["采集者1", "采集者2"]
}
```

---

### 8.3 隐藏指定笔记

| 属性 | 值 |
|------|-----|
| 接口路径 | `PUT /admin/notes/{id}/hidden/{hidden}` |
| 接口说明 | 隐藏或显示指定笔记 |
| 权限要求 | 管理员 |

**请求参数（Path）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| id | Long | 是 | 笔记ID |
| hidden | Boolean | 是 | true-隐藏，false-显示 |

**返回值**：无（void）

---

## 九、积分排行榜相关接口

### 9.1 分页查询积分排行榜

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /boards` |
| 接口说明 | 分页查询指定赛季的积分排行榜。当前赛季从Redis查询，历史赛季从MySQL查询。 |
| 权限要求 | 需要登录 |

**请求参数（Query）**：

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| season | Long | 否 | 赛季ID，为null或0代表查询当前赛季 |
| pageNo | Integer | 否 | 页码 |
| pageSize | Integer | 否 | 每页条数 |

**返回值**：`PointsBoardVO`

```json
{
  "rank": "我的排名",
  "points": "我的积分",
  "boardList": [
    {
      "rank": 1,
      "points": 520,
      "name": "学生姓名"
    },
    {
      "rank": 2,
      "points": 480,
      "name": "学生姓名"
    }
  ]
}
```

---

### 9.2 查询赛季信息列表

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /boards/seasons/list` |
| 接口说明 | 查询所有已开始的赛季信息列表（开始时间 <= 当前时间） |
| 权限要求 | 无 |

**请求参数**：无

**返回值**：`List<PointsBoardSeasonVO>`

```json
[
  {
    "id": 1,
    "name": "第1赛季",
    "beginTime": "2024-01-01",
    "endTime": "2024-01-31"
  },
  {
    "id": 2,
    "name": "第2赛季",
    "beginTime": "2024-02-01",
    "endTime": "2024-02-29"
  }
]
```

---

## 十、积分记录相关接口

### 10.1 查询我的今日积分

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /points/today` |
| 接口说明 | 查询当前用户今日各种方式获取的积分汇总及上限信息 |
| 权限要求 | 需要登录 |

**请求参数**：无

**返回值**：`List<PointsStatisticsVO>`

```json
[
  {
    "type": "课程学习",
    "points": 30,
    "maxPoints": 50
  },
  {
    "type": "每日签到",
    "points": 1,
    "maxPoints": 0
  },
  {
    "type": "课程问答",
    "points": 10,
    "maxPoints": 20
  }
]
```

**说明**：`maxPoints` 为0表示无上限。

---

## 十一、签到相关接口

### 11.1 签到

| 属性 | 值 |
|------|-----|
| 接口路径 | `POST /sign-records` |
| 接口说明 | 执行签到操作，每天只能签到一次。连续签到7/14/28天有额外奖励。 |
| 权限要求 | 需要登录 |

**请求参数**：无

**返回值**：`SignResultVO`

```json
{
  "signDays": 7,
  "signPoints": 1,
  "rewardPoints": 10
}
```

**字段说明**：
- `signDays`：连续签到天数
- `signPoints`：签到基础积分（固定1分）
- `rewardPoints`：连续签到奖励积分（7天=10分，14天=20分，28天=40分）

**错误情况**：
- 重复签到：抛出异常"不允许重复签到！"

---

### 11.2 查询签到记录

| 属性 | 值 |
|------|-----|
| 接口路径 | `GET /sign-records` |
| 接口说明 | 查询当前用户本月的签到记录 |
| 权限要求 | 需要登录 |

**请求参数**：无

**返回值**：`Byte[]`

返回一个Byte数组，长度等于本月已过天数，每个元素值为1（已签到）或0（未签到）。

**示例**（假设今天是1月5日）：
```json
[1, 1, 1, 0, 1]
```
表示1月1日、2日、3日已签，4日未签，5日已签。

---

## 附录：接口路径汇总

| 模块 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 课表 | GET | `/lessons/page` | 分页查询我的课表 |
| 课表 | GET | `/lessons/now` | 查询正在学习的课程 |
| 课表 | GET | `/lessons/{courseId}` | 查询指定课程信息 |
| 课表 | DELETE | `/lessons/{courseId}` | 删除指定课程 |
| 课表 | GET | `/lessons/{courseId}/count` | 统计课程学习人数 |
| 课表 | GET | `/lessons/{courseId}/valid` | 校验是否已报名 |
| 课表 | POST | `/lessons/plans` | 创建学习计划 |
| 课表 | GET | `/lessons/plans` | 查询我的学习计划 |
| 学习记录 | GET | `/learning-records/course/{courseId}` | 查询课程学习记录 |
| 学习记录 | POST | `/learning-records` | 提交学习记录 |
| 问答(用户) | POST | `/questions` | 新增问题 |
| 问答(用户) | PUT | `/questions/{id}` | 修改问题 |
| 问答(用户) | GET | `/questions/page` | 分页查询问题 |
| 问答(用户) | GET | `/questions/{id}` | 查询问题详情 |
| 问答(用户) | DELETE | `/questions/{id}` | 删除问题 |
| 问答(管理) | GET | `/admin/questions/page` | 管理端查询问题 |
| 问答(管理) | GET | `/admin/questions/{id}` | 管理端查询详情 |
| 问答(管理) | PUT | `/admin/questions/{id}/hidden/{hidden}` | 隐藏/显示问题 |
| 回答(用户) | POST | `/replies` | 新增回答/评论 |
| 回答(用户) | GET | `/replies/page` | 分页查询回答 |
| 回答(管理) | PUT | `/admin/replies/{id}/hidden/{hidden}` | 隐藏/显示回答 |
| 回答(管理) | GET | `/admin/replies/page` | 管理端查询回答 |
| 回答(管理) | GET | `/admin/replies/{id}` | 查询回答详情 |
| 笔记(用户) | POST | `/notes` | 新增笔记 |
| 笔记(用户) | POST | `/notes/gathers/{id}` | 采集笔记 |
| 笔记(用户) | DELETE | `/notes/gathers/{id}` | 取消采集 |
| 笔记(用户) | PUT | `/notes/{id}` | 更新笔记 |
| 笔记(用户) | DELETE | `/notes/{id}` | 删除笔记 |
| 笔记(用户) | GET | `/notes/page` | 分页查询笔记 |
| 笔记(管理) | GET | `/admin/notes/page` | 管理端查询笔记 |
| 笔记(管理) | GET | `/admin/notes/{id}` | 查询笔记详情 |
| 笔记(管理) | PUT | `/admin/notes/{id}/hidden/{hidden}` | 隐藏/显示笔记 |
| 排行榜 | GET | `/boards` | 查询积分排行榜 |
| 排行榜 | GET | `/boards/seasons/list` | 查询赛季列表 |
| 积分 | GET | `/points/today` | 查询今日积分 |
| 签到 | POST | `/sign-records` | 签到 |
| 签到 | GET | `/sign-records` | 查询签到记录 |
