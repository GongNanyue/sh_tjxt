# tj-auth 认证授权服务详解

## 一、服务概述

tj-auth 是天机学堂的认证授权核心模块，采用**多模块架构**设计，包含以下子模块：

| 子模块 | 说明 |
|--------|------|
| tj-auth-common | 公共模块，定义JWT常量、权限DTO等共享类 |
| tj-auth-service | 认证服务本体，提供登录认证、角色管理、菜单管理、权限管理等功能 |
| tj-auth-gateway-sdk | 网关鉴权SDK，供网关服务使用，负责JWT解析和权限路径匹配 |
| tj-auth-resource-sdk | 资源服务鉴权SDK，供所有业务微服务引入，实现用户信息传递和登录拦截 |

**启动类：** `com.tianji.auth.AuthApplication`

## 二、依赖分析

### tj-auth-service 依赖
| 依赖 | 说明 |
|------|------|
| tj-api | 公共 API 模块，包含 UserClient |
| tj-auth-resource-sdk | 自身也引入资源鉴权SDK |
| spring-boot-starter-web | Web 框架 |
| mybatis-plus-boot-starter + mysql | ORM 框架与数据库 |
| spring-boot-starter-data-redis + redisson + commons-pool2 | Redis（JWT JTI缓存） |
| spring-cloud-starter-alibaba-nacos-discovery | 服务注册与发现 |
| spring-cloud-starter-alibaba-nacos-config | 配置中心 |
| spring-cloud-starter-loadbalancer | 客户端负载均衡 |

### tj-auth-gateway-sdk 依赖
主要依赖 tj-auth-common、hutool-jwt、spring-data-redis，作为SDK被网关服务引入。

### tj-auth-resource-sdk 依赖
轻量级SDK，依赖 tj-auth-common 和 Spring MVC 拦截器机制。

## 三、核心业务逻辑

### 3.1 登录认证（AccountServiceImpl + JwtTool）

**登录流程：**
1. 接收登录表单（手机号+密码），区分学员端和管理端登录
2. 通过 UserClient 远程调用用户服务验证用户信息
3. 使用 RS256 非对称加密算法生成 JWT access-token（短有效期）
4. 生成 refresh-token 并将其 JTI（JWT ID）存入 Redis（用于刷新token时的校验）
5. 将 refresh-token 写入 HttpOnly Cookie，根据"记住我"设置不同的有效期
6. 记录登录日志

**Token 刷新流程：**
1. 从 Cookie 中读取 refresh-token（区分学员端和管理端使用不同 Cookie 名称）
2. 校验 refresh-token 签名、有效期、JTI 一致性
3. 生成新的 access-token 和 refresh-token

**退出登录：**
1. 删除 Redis 中的 JTI 缓存（使 refresh-token 失效）
2. 清除浏览器 Cookie

### 3.2 角色管理（RoleController + RoleService）

- 查询角色列表（支持全量和仅自定义角色）
- 角色CRUD操作
- 角色类型：CONSTANT（固定角色，不可选）、CUSTOM（自定义角色）

### 3.3 菜单管理（MenuController + MenuService）

- 查询菜单树结构（支持按父菜单ID查询子菜单）
- 查询当前用户的菜单树（根据权限过滤）
- 菜单CRUD操作
- 绑定/解绑角色与菜单权限

### 3.4 权限管理（PrivilegeController + PrivilegeService）

- 分页查询所有权限
- 查询菜单下的权限列表
- 查询角色拥有的权限（标记已选中）
- 权限CRUD操作
- 绑定/解绑角色与API权限

### 3.5 权限数据加载（LoadPrivilegeRunner）

服务启动时通过 `ApplicationRunner` 加载所有权限数据到 Redis（Hash结构），包含权限路径和关联的角色ID集合，供网关SDK使用。

## 四、SDK设计详解

### 4.1 网关鉴权SDK（tj-auth-gateway-sdk）

**核心类：AuthUtil**

该SDK被网关服务引入，在网关层完成JWT校验和权限判断：

1. **Token解析（parseToken）**：
   - 校验JWT签名有效性（使用公钥验证，公钥通过 JwkController 接口从认证服务获取）
   - 校验JWT是否过期
   - 解析 payload 中的用户信息（userId、roleId等）

2. **权限校验（checkAuth）**：
   - 使用 AntPathMatcher 匹配请求路径是否在权限列表中
   - 如果路径需要权限，校验用户角色是否匹配
   - 不在权限列表中的路径直接放行

3. **权限缓存刷新（refreshTask）**：
   - 每20秒定时从 Redis 加载最新权限数据（Hash结构）
   - 通过版本号机制减少不必要的缓存处理

**核心类：JwtSignerHolder**

- 通过 DiscoveryClient 发现认证服务实例
- 调用 `/jwks` 接口获取公钥
- 构建 JWT 签名验证器

### 4.2 资源服务鉴权SDK（tj-auth-resource-sdk）

该SDK被所有业务微服务引入，提供以下功能：

1. **UserInfoInterceptor**（用户信息拦截器）：
   - 从请求头（`user-info`）中读取网关传递的用户ID
   - 将用户ID存入 `UserContext`（基于ThreadLocal）
   - 请求完成后清理ThreadLocal，防止内存泄漏

2. **LoginAuthInterceptor**（登录认证拦截器）：
   - 检查 `UserContext` 中是否有用户信息
   - 如果没有则返回401未授权
   - 可配置哪些路径需要登录拦截

3. **FeignRelayUserInterceptor**（Feign用户信息传递拦截器）：
   - 实现 `RequestInterceptor`
   - 在微服务间Feign调用时，将当前用户ID通过请求头传递
   - 确保跨服务调用时用户身份不丢失

### 4.3 认证鉴权整体流程

```
用户请求 -> 网关（gateway-sdk解析JWT、校验权限）
         -> 将用户ID写入请求头 user-info
         -> 业务微服务（resource-sdk读取user-info、存入UserContext）
         -> Feign调用其他微服务（resource-sdk透传user-info）
```

## 五、数据模型

### 5.1 Role（角色表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 角色ID |
| code | String | 角色代号（如admin） |
| name | String | 角色名称 |
| type | RoleType | 角色类型：0固定角色、1自定义角色 |

### 5.2 Privilege（权限表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 权限ID |
| menuId | Long | 所属菜单ID |
| intro | String | 权限说明 |
| method | String | API请求方式（GET/POST/PUT/DELETE） |
| uri | String | API请求路径 |
| internal | Boolean | 是否为内部接口 |

### 5.3 Menu（菜单表）

菜单表，支持多级菜单结构，通过 parentId 建立父子关系。

### 5.4 AccountRole（账户角色关联表）

记录用户与角色的多对多关系。

### 5.5 RolePrivilege / RoleMenu（角色权限/菜单关联表）

记录角色与权限、菜单的多对多关系。

### 5.6 LoginRecord（登录记录表）

记录用户登录日志。

## 六、API接口清单

### 账户接口（/accounts）
| 方法 | 路径 | 功能 |
|------|------|------|
| POST | /accounts/login | 学员端登录获取Token |
| POST | /accounts/admin/login | 管理端登录获取Token |
| POST | /accounts/logout | 退出登录 |
| GET | /accounts/refresh | 刷新Token |

### 角色接口（/roles）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /roles/list | 查询所有角色列表 |
| GET | /roles | 查询员工角色列表（自定义角色） |
| GET | /roles/{id} | 根据ID查询角色 |
| POST | /roles | 新增角色 |
| PUT | /roles/{id} | 修改角色信息 |
| DELETE | /roles/{id} | 删除角色 |

### 菜单接口（/menus）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /menus/parent/{pid} | 根据父菜单ID查询子菜单 |
| GET | /menus/{id} | 根据ID查询菜单 |
| GET | /menus | 查询菜单树结构 |
| GET | /menus/me | 查询我的菜单树 |
| POST | /menus | 新增菜单 |
| PUT | /menus/{id} | 更新菜单 |
| DELETE | /menus/{id} | 删除菜单 |
| POST | /menus/role/{roleId} | 绑定角色与菜单权限 |
| DELETE | /menus/role/{roleId} | 解除角色的菜单权限 |

### 权限接口（/privileges）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /privileges | 分页查询所有权限 |
| GET | /privileges/options/{menuId} | 查询菜单下的权限选项 |
| GET | /privileges/roles/{roleId}/{menuId} | 查询角色在菜单下的权限 |
| POST | /privileges | 新增权限 |
| PUT | /privileges/{id} | 修改权限 |
| DELETE | /privileges/{id} | 删除权限 |
| POST | /privileges/role/{roleId} | 绑定角色与API权限 |
| DELETE | /privileges/role/{roleId} | 解除角色的API权限 |

### JWK接口（/jwks）
| 方法 | 路径 | 功能 |
|------|------|------|
| GET | /jwks | 获取公钥（Base64编码），供网关SDK验证JWT |

## 七、技术方案

1. **RS256 非对称加密JWT**：使用 RSA 密钥对，私钥签发Token（认证服务持有），公钥验证Token（网关SDK通过接口获取公钥）。
2. **双Token机制**：短有效期的 access-token + 长有效期的 refresh-token，access-token 过期后通过 refresh-token 无感刷新。
3. **JTI防篡改**：refresh-token 中包含 JTI，Redis 中存储对应 JTI，刷新时校验一致性，退出时删除JTI使Token失效。
4. **权限版本化**：Redis 中维护权限版本号，权限变更时版本号递增，网关SDK定时对比版本号决定是否刷新缓存。
5. **SDK分层设计**：
   - gateway-sdk：网关层使用，负责JWT校验和权限判断
   - resource-sdk：业务层使用，负责用户信息传递（网关->微服务->微服务）
   - common：公共模块，定义共享常量和DTO
