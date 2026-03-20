# tj-course 课程服务接口文档

## 一、课程管理接口（/courses）

### 1.1 获取课程基础信息
- **路径：** `GET /courses/baseInfo/{id}`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | id | Long | 是 | 课程ID（路径参数） |
  | see | Boolean | 否 | 是否查看模式，默认true。false为编辑模式 |
- **返回值：** `CourseBaseInfoVO`（课程名称、分类、价格、封面、有效期等基本信息）
- **功能说明：** 获取课程基本信息。查看模式查询正式数据，编辑模式查询草稿数据。

### 1.2 保存课程基本信息
- **路径：** `POST /courses/baseInfo/save`
- **请求参数：** `CourseBaseInfoSaveDTO`
  ```json
  {
    "id": null,              // 课程ID，新增时为空
    "name": "Java基础入门",   // 课程名称
    "courseType": 2,          // 课程类型：1直播课、2录播课
    "coverUrl": "http://...", // 封面链接
    "firstCateId": 1,        // 一级分类ID
    "secondCateId": 2,       // 二级分类ID
    "thirdCateId": 3,        // 三级分类ID
    "free": 0,               // 0付费、1免费
    "price": 9900             // 价格（分）
  }
  ```
- **返回值：** `CourseSaveVO`（含课程ID）
- **功能说明：** 保存课程基本信息到草稿表。

### 1.3 获取课程的章节
- **路径：** `GET /courses/catas/{id}`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | id | Long | 是 | 课程ID |
  | see | Boolean | 否 | 查看模式，默认true |
  | withPractice | Boolean | 否 | 是否包含练习，默认true |
- **返回值：** `List<CataVO>`（树状结构的章节列表，含章-节-练习）
- **功能说明：** 获取课程的完整章节目录结构。

### 1.4 保存章节
- **路径：** `POST /courses/catas/save/{id}/{step}`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | id | Long | 是 | 课程ID（路径参数） |
  | step | Integer | 否 | 填写步骤 |
  | body | List<CataSaveDTO> | 是 | 章节数据列表 |
- **返回值：** 无
- **功能说明：** 保存课程的章节目录到草稿表。

### 1.5 保存课程视频
- **路径：** `POST /courses/media/save/{id}`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | id | Long | 是 | 课程ID（路径参数） |
  | body | List<CourseMediaDTO> | 是 | 视频媒资关联信息 |
- **返回值：** 无
- **功能说明：** 保存小节与媒资视频的关联关系。

### 1.6 保存小节题目
- **路径：** `POST /courses/subjects/save/{id}`
- **请求参数：** `id`（路径参数） + `List<CataSubjectDTO>`（请求体）
- **返回值：** 无
- **功能说明：** 保存小节或练习中关联的题目。

### 1.7 获取小节题目（用于编辑）
- **路径：** `GET /courses/subjects/get/{id}`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** `List<CataSimpleSubjectVO>`
- **功能说明：** 获取课程中小节/练习关联的题目列表。

### 1.8 查询课程老师信息
- **路径：** `GET /courses/teachers/{id}`
- **请求参数：**
  | 参数 | 类型 | 必填 | 说明 |
  |------|------|------|------|
  | id | Long | 是 | 课程ID |
  | see | Boolean | 否 | 查看模式，默认true |
- **返回值：** `List<CourseTeacherVO>`
- **功能说明：** 查询课程关联的教师列表。

### 1.9 保存老师信息
- **路径：** `POST /courses/teachers/save`
- **请求参数：** `CourseTeacherSaveDTO`
- **返回值：** 无
- **功能说明：** 保存课程与教师的关联关系。

### 1.10 课程上架
- **路径：** `POST /courses/upShelf`
- **请求参数：**
  ```json
  { "id": 1 }
  ```
- **返回值：** 无
- **功能说明：** 将课程从草稿状态发布上架，草稿数据同步到正式表，发送MQ通知搜索服务。

### 1.11 课程上架前校验
- **路径：** `GET /courses/checkBeforeUpShelf/{id}`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** 无（校验不通过会抛异常）
- **功能说明：** 校验课程信息完整性，确保可以上架。

### 1.12 课程下架
- **路径：** `POST /courses/downShelf`
- **请求参数：**
  ```json
  { "id": 1 }
  ```
- **返回值：** 无
- **功能说明：** 将课程下架，发送MQ通知搜索服务删除索引。

### 1.13 课程删除
- **路径：** `DELETE /courses/delete/{id}`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** 无
- **功能说明：** 删除课程及相关草稿数据。

### 1.14 根据条件获取课程简要信息
- **路径：** `GET /courses/simpleInfo/list`
- **请求参数：** `CourseSimpleInfoListDTO`（查询条件）
- **返回值：** `List<CourseSimpleInfoDTO>`
- **功能说明：** 内部调用接口，按条件批量查询课程简要信息。

### 1.15 查询章节序号列表
- **路径：** `GET /courses/catas/index/list/{id}`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** `List<CataSimpleInfoVO>`
- **功能说明：** 获取课程所有章节的序号信息。

### 1.16 生成练习ID
- **路径：** `GET /courses/generator`
- **请求参数：** 无
- **返回值：** `CourseCataIdVO`（含雪花算法生成的ID）
- **功能说明：** 前端创建练习时使用，预生成一个唯一ID。

### 1.17 管理端课程搜索
- **路径：** `GET /courses/page`
- **请求参数：** `CoursePageQuery`（含分页参数、课程状态、关键词等）
- **返回值：** `PageDTO<CoursePageVO>`
- **功能说明：** 管理端课程搜索，根据状态自动选择查询草稿表或正式表。

### 1.18 校验课程名称是否存在
- **路径：** `GET /courses/checkName?id=1&name=Java基础`
- **请求参数：** `id`（可选，编辑时传入排除自身）、`name`（课程名称）
- **返回值：** `NameExistVO`
- **功能说明：** 校验课程名称唯一性。

### 1.19 查询课程基本信息和目录
- **路径：** `GET /courses/{id}/catalogs`
- **请求参数：** `id`（路径参数）- 课程ID
- **返回值：** `CourseAndSectionVO`
- **功能说明：** 用户端查询课程基本信息和章节目录。

---

## 二、课程分类接口（/categorys）

### 2.1 查询课程分类（树结构）
- **路径：** `GET /categorys/list`
- **请求参数：** `CategoryListDTO`
- **返回值：** `List<CategoryVO>`
- **功能说明：** 查询课程分类信息，返回树状结构。

### 2.2 获取分类详情
- **路径：** `GET /categorys/{id}`
- **请求参数：** `id`（路径参数）- 分类ID
- **返回值：** `CategoryInfoVO`

### 2.3 新增课程分类
- **路径：** `POST /categorys/add`
- **请求参数：** `CategoryAddDTO`
- **返回值：** 无

### 2.4 删除分类
- **路径：** `DELETE /categorys/{id}`
- **请求参数：** `id`（路径参数）
- **返回值：** 无

### 2.5 分类启用/停用
- **路径：** `PUT /categorys/disableOrEnable`
- **请求参数：** `CategoryDisableOrEnableDTO`
- **返回值：** 无

### 2.6 更新课程分类
- **路径：** `PUT /categorys/update`
- **请求参数：** `CategoryUpdateDTO`
- **返回值：** 无

### 2.7 获取所有分类（简要，含层级关系）
- **路径：** `GET /categorys/all?admin=false`
- **请求参数：** `admin`（是否管理端，默认false）
- **返回值：** `List<SimpleCategoryVO>`

### 2.8 获取所有分类（不分层）
- **路径：** `GET /categorys/getAllOfOneLevel`
- **返回值：** `List<CategoryVO>`

---

## 三、章节目录接口（/catalogues）

### 3.1 批量查询章节目录基础信息
- **路径：** `GET /catalogues/batchQuery?ids=1,2,3`
- **请求参数：** `ids`（查询参数）- 章节目录ID列表
- **返回值：** `List<CataSimpleInfoVO>`
- **功能说明：** 内部调用，根据章节目录ID批量查询名称、序号等基础信息。

### 3.2 获取小节信息
- **路径：** `GET /catalogues/querySectionInfoById/{id}`
- **请求参数：** `id`（路径参数）- 小节ID
- **返回值：** `CataSimpleInfoVO`

---

## 四、内部调用接口（/course）

### 4.1 通过老师ID获取课程和出题数量
- **路径：** `GET /course/infoByTeacherIds?teacherIds=1,2`
- **返回值：** `List<SubNumAndCourseNumDTO>`

### 4.2 根据小节ID获取媒资和课程信息
- **路径：** `GET /course/section/{id}`
- **返回值：** `SectionInfoDTO`

### 4.3 查询媒资引用次数
- **路径：** `GET /course/media/useInfo?mediaIds=1,2`
- **返回值：** `List<MediaQuoteDTO>`

### 4.4 获取课程搜索信息
- **路径：** `GET /course/{id}/searchInfo`
- **返回值：** `CourseDTO`（课程上架时供搜索服务使用）

### 4.5 获取课程完整信息
- **路径：** `GET /course/{id}?withCatalogue=true&withTeachers=true`
- **返回值：** `CourseFullInfoDTO`

### 4.6 按三级分类ID查询分类名称
- **路径：** `GET /course/getCateNameMap?thirdCateIdList=1,2,3`
- **返回值：** `Map<Long, String>`

### 4.7 按名称查询课程ID列表
- **路径：** `GET /course/name?name=Java`
- **返回值：** `List<Long>`
