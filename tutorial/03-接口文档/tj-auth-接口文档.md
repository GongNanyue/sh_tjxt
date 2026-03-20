# tj-auth 认证授权服务接口文档

## 一、账户接口（/accounts）

### 1.1 学员端登录
- **路径：** `POST /accounts/login`
- **请求参数：**
  ```json
  {
    "cellPhone": "13800138000",  // 手机号
    "password": "123456",         // 密码
    "rememberMe": false           // 是否记住我
  }
  ```
- **返回值：** `String`（JWT access-token）
- **功能说明：** 学员端登录接口。验证用户信息后返回 JWT access-token，同时生成 refresh-token 写入 HttpOnly Cookie。记住我为 true 时 refresh-token 有效期为 7 天，否则为 30 分钟。
- **响应头：** Set-Cookie（refresh-token）

### 1.2 管理端登录
- **路径：** `POST /accounts/admin/login`
- **请求参数：** 同 1.1
- **返回值：** `String`（JWT access-token）
- **功能说明：** 管理端（后台员工/管理员）登录接口，逻辑与学员端登录相同，但 Cookie 名称不同（`admin-refresh-token`）。

### 1.3 退出登录
- **路径：** `POST /accounts/logout`
- **请求参数：** 无（需携带 access-token）
- **返回值：** 无
- **功能说明：** 退出登录，删除 Redis 中的 JWT JTI 缓存（使 refresh-token 失效），清除浏览器中的 Cookie。

### 1.4 刷新Token
- **路径：** `GET /accounts/refresh`
- **请求参数：** 无（需携带 refresh-token Cookie）
- **返回值：** `String`（新的 JWT access-token）
- **功能说明：** 当 access-token 过期时调用此接口。从 Cookie 中读取 refresh-token，校验签名和 JTI 一致性后生成新的 access-token 和 refresh-token。根据请求来源（www 开头为学员端，否则为管理端）选择对应的 Cookie。

---

## 二、角色接口（/roles）

### 2.1 查询所有角色列表
- **路径：** `GET /roles/list`
- **请求参数：** 无
- **返回值：** `List<RoleDTO>`
  ```json
  [
    { "id": 1, "code": "admin", "name": "管理员" },
    { "id": 2, "code": "student", "name": "学员" }
  ]
  ```
- **功能说明：** 查询系统中所有角色（含固定角色和自定义角色）。

### 2.2 查询员工角色列表
- **路径：** `GET /roles`
- **请求参数：** 无
- **返回值：** `List<RoleDTO>`
- **功能说明：** 仅查询自定义角色（type=CUSTOM），用于员工角色分配。

### 2.3 根据ID查询角色
- **路径：** `GET /roles/{id}`
- **请求参数：** `id`（路径参数）- 角色ID
- **返回值：** `RoleDTO`

### 2.4 新增角色
- **路径：** `POST /roles`
- **请求参数：**
  ```json
  {
    "code": "teacher",
    "name": "教师"
  }
  ```
- **返回值：** `RoleDTO`（含生成的ID）
- **功能说明：** 新增自定义角色。

### 2.5 修改角色信息
- **路径：** `PUT /roles/{id}`
- **请求参数：** `id`（路径参数）+ `RoleDTO`（请求体）
- **返回值：** 无

### 2.6 删除角色
- **路径：** `DELETE /roles/{id}`
- **请求参数：** `id`（路径参数）
- **返回值：** 无

---

## 三、菜单接口（/menus）

### 3.1 根据父菜单ID查询子菜单
- **路径：** `GET /menus/parent/{pid}`
- **请求参数：** `pid`（路径参数）- 父菜单ID（0 为查询一级菜单）
- **返回值：** `List<MenuOptionVO>`

### 3.2 根据ID查询菜单
- **路径：** `GET /menus/{id}`
- **请求参数：** `id`（路径参数）- 菜单ID
- **返回值：** `MenuOptionVO`

### 3.3 查询菜单树结构
- **路径：** `GET /menus`
- **请求参数：** 无
- **返回值：** `List<MenuOptionVO>`（树状结构，含 subMenus 子菜单列表）
- **功能说明：** 查询所有菜单并组成树状结构，按优先级排序。

### 3.4 查询我的菜单树
- **路径：** `GET /menus/me`
- **请求参数：** 无（需登录）
- **返回值：** `List<MenuOptionVO>`（树状结构）
- **功能说明：** 根据当前登录用户的角色权限过滤菜单，返回该用户可见的菜单树。

### 3.5 新增菜单
- **路径：** `POST /menus`
- **请求参数：** `MenuDTO`
- **返回值：** 无

### 3.6 更新菜单
- **路径：** `PUT /menus/{id}`
- **请求参数：** `id`（路径参数）+ `MenuDTO`（请求体）
- **返回值：** 无

### 3.7 删除菜单
- **路径：** `DELETE /menus/{id}`
- **请求参数：** `id`（路径参数）
- **返回值：** 无

### 3.8 绑定角色与菜单权限
- **路径：** `POST /menus/role/{roleId}`
- **请求参数：**
  | 参数 | 类型 | 说明 |
  |------|------|------|
  | roleId | Long | 角色ID（路径参数） |
  | body | List<Long> | 菜单ID集合 |
- **返回值：** 无
- **功能说明：** 将菜单权限绑定到指定角色。

### 3.9 解除角色的菜单权限
- **路径：** `DELETE /menus/role/{roleId}`
- **请求参数：** 同 3.8
- **返回值：** 无

---

## 四、权限接口（/privileges）

### 4.1 分页查询所有权限
- **路径：** `GET /privileges`
- **请求参数：** `PageQuery`（pageNo、pageSize）
- **返回值：** `PageDTO<PrivilegeDTO>`
  ```json
  {
    "total": 100,
    "pages": 10,
    "list": [
      {
        "id": 1,
        "menuId": 10,
        "intro": "查询用户列表",
        "method": "GET",
        "uri": "/users/list",
        "internal": false
      }
    ]
  }
  ```

### 4.2 查询菜单下的权限选项
- **路径：** `GET /privileges/options/{menuId}`
- **请求参数：** `menuId`（路径参数）- 菜单ID
- **返回值：** `List<PrivilegeOptionVO>`
- **功能说明：** 查询指定菜单下的非内部权限，用于下拉选框。

### 4.3 查询角色在菜单下的权限
- **路径：** `GET /privileges/roles/{roleId}/{menuId}`
- **请求参数：** `roleId`、`menuId`（路径参数）
- **返回值：** `List<PrivilegeOptionVO>`（含 checked 字段标记角色是否已拥有该权限）
- **功能说明：** 查询菜单下所有权限，并标记哪些已分配给指定角色。

### 4.4 新增权限
- **路径：** `POST /privileges`
- **请求参数：** `PrivilegeDTO`
- **返回值：** `PrivilegeDTO`（含生成的ID）

### 4.5 修改权限
- **路径：** `PUT /privileges/{id}`
- **请求参数：** `id`（路径参数）+ `PrivilegeDTO`
- **返回值：** `PrivilegeDTO`

### 4.6 删除权限
- **路径：** `DELETE /privileges/{id}`
- **请求参数：** `id`（路径参数）
- **返回值：** 无

### 4.7 绑定角色与API权限
- **路径：** `POST /privileges/role/{roleId}`
- **请求参数：**
  | 参数 | 类型 | 说明 |
  |------|------|------|
  | roleId | Long | 角色ID（路径参数） |
  | body | List<Long> | API权限ID集合 |
- **返回值：** 无

### 4.8 解除角色的API权限
- **路径：** `DELETE /privileges/role/{roleId}`
- **请求参数：** 同 4.7
- **返回值：** 无

---

## 五、JWK接口（/jwks）

### 5.1 获取公钥
- **路径：** `GET /jwks`
- **请求参数：** 无
- **返回值：** `String`（Base64 编码的 RSA 公钥）
- **功能说明：** 内部调用接口，供网关 SDK 获取 JWT 签名验证公钥。该接口不展示在 Swagger 文档中（标注 @ApiIgnore）。

---

## 附录：JWT Token 说明

### Access Token
- **算法：** RS256（RSA + SHA-256）
- **Payload：** 包含 `user` 字段（userId、roleId、rememberMe 等）
- **有效期：** 短期（由 `JwtConstants.JWT_TOKEN_TTL` 定义）

### Refresh Token
- **算法：** RS256
- **Payload：** 包含 `user` 字段和 `jti`（JWT ID）字段
- **有效期：** 记住我模式 7 天，否则 30 分钟（由 `JwtConstants.JWT_REFRESH_TTL` / `JWT_REMEMBER_ME_TTL` 定义）
- **存储方式：** HttpOnly Cookie
- **JTI 校验：** Redis 中以 `jwt:uid:${userId}` 为 key 存储 JTI，刷新时校验一致性
