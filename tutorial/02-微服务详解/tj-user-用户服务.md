# tj-user 用户服务详解

## 一、服务概述

tj-user 是天机学堂的用户管理核心服务，负责用户账号管理、学员注册、教师管理、员工管理等功能。该服务同时为认证服务提供用户验证接口，是登录认证流程中的关键一环。

**启动类：** `com.tianji.user.UserApplication`

## 二、依赖分析

| 依赖 | 说明 |
|------|------|
| spring-boot-starter-web | Web 框架 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-boot-starter-data-redis + commons-pool2 | Redis 缓存 |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| tj-auth-resource-sdk | 资源服务鉴权 SDK |
| tj-api | 公共 API 模块 |
| tj-message-api | 消息服务 Feign 客户端（用于发送验证码短信） |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |

## 三、核心业务逻辑

### 3.1 用户管理（UserController + UserServiceImpl）
- **新增用户**：新增员工或教师账号
- **更新用户信息**：管理员更新指定用户信息
- **修改当前用户信息**：支持修改密码
- **重置密码**：管理员重置指定用户密码为默认密码
- **修改用户状态**：禁用/启用用户账号
- **获取当前用户信息**：返回当前登录用户的详细信息
- **用户验证接口（内部调用）**：接收 LoginFormDTO，校验手机号和密码，返回 LoginUserDTO（含 userId、roleId 等）
- **根据ID批量查询用户**：供其他微服务调用
- **手机号查询用户ID / 检查手机号存在性**

### 3.2 学员管理（StudentController + StudentServiceImpl）
- **学员注册**：接收手机号和密码，创建学员账号
- **分页查询学员**：管理端分页查询学员信息
- **修改学员密码**

### 3.3 教师管理（TeacherController + TeacherServiceImpl）
- **分页查询教师**：管理端分页查询教师信息

### 3.4 员工管理（StaffController + StaffServiceImpl）
- **分页查询员工**：管理端分页查询员工信息

### 3.5 验证码服务（CodeServiceImpl）
- 生成并发送短信验证码（通过 MessageClient 调用消息服务）
- 验证码校验

## 四、数据模型

### 4.1 User（用户表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 用户主键 |
| username | String | 用户名 |
| cellPhone | String | 手机号 |
| password | String | 密码（加密存储） |
| status | UserStatus | 账户状态：0禁用、1正常 |
| type | UserType | 用户类型：1其他员工、2普通学员、3老师 |
| createTime | LocalDateTime | 创建时间 |
| updateTime | LocalDateTime | 更新时间 |

### 4.2 UserDetail（用户详情表）

存储用户的详细信息，如姓名、头像、个人简介等，与 User 表通过 userId 关联。

## 五、API接口清单

### 用户接口（/users）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /users | 新增用户（员工/教师） |
| PUT | /users/{id} | 更新用户信息 |
| PUT | /users | 更新当前登录用户信息（含密码修改） |
| PUT | /users/{id}/password/default | 重置密码 |
| PUT | /users/{id}/status/{status} | 修改用户状态 |
| GET | /users/me | 获取当前登录用户信息 |
| GET | /users/{id} | 根据ID查询用户信息 |
| POST | /users/detail/{isStaff} | 用户登录验证（内部调用） |
| GET | /users/list | 根据ID批量查询用户 |
| GET | /users/{id}/type | 查询用户类型 |
| GET | /users/ids | 根据手机号查询用户ID |
| GET | /users/checkCellphone | 检查手机号是否已存在 |

### 学员接口（/students）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /students/page | 分页查询学生信息 |
| POST | /students/register | 学员注册 |
| PUT | /students/password | 修改学员密码 |

### 教师接口（/teachers）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /teachers/page | 分页查询教师信息 |

### 员工接口（/staffs）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /staffs/page | 分页查询员工信息 |

## 六、技术方案

1. **密码安全**：使用 Spring Security 提供的 BCryptPasswordEncoder 进行密码加密存储（SecurityConfig 配置）。
2. **用户类型分角色管理**：通过 UserType 枚举区分学员、教师、员工，不同类型有不同的注册和管理流程。
3. **内部接口设计**：登录验证接口使用 `@ApiIgnore` 注解隐藏，仅供认证服务内部调用。
4. **消息服务集成**：引入 tj-message-api 依赖，通过 Feign 调用消息服务发送短信验证码。
