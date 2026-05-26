# yudao-cloud OAuth2 授权、Token、CheckToken 与 Scope 权限校验核心链路分析

## 1. 整体架构与关键组件协作关系

### 1.1 核心组件一览

| 组件 | 职责 | 关键文件位置 |
|------|------|--------------|
| `OAuth2OpenController` | 对外暴露 OAuth2 标准接口（authorize、token、check-token、revoke） | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/controller/admin/oauth2/OAuth2OpenController.java` |
| `OAuth2UserController` | 提供受 OAuth2 scope 保护的用户相关接口 | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/controller/admin/oauth2/OAuth2UserController.java` |
| `OAuth2GrantServiceImpl` | 处理各种 OAuth2 授权模式的业务逻辑（授权码、密码、客户端、刷新令牌） | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/oauth2/OAuth2GrantServiceImpl.java` |
| `OAuth2TokenServiceImpl` | 负责 access_token 和 refresh_token 的创建、刷新、校验、删除 | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/oauth2/OAuth2TokenServiceImpl.java` |
| `OAuth2CodeServiceImpl` | 授权码模式中 code 的生成与消费（一次性使用） | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/oauth2/OAuth2CodeServiceImpl.java` |
| `OAuth2ClientServiceImpl` | 客户端管理与校验（client、secret、redirectUri、scope、grantType） | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/oauth2/OAuth2ClientServiceImpl.java` |
| `OAuth2AccessTokenRedisDAO` | access_token 的 Redis 缓存管理 | `yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/redis/oauth2/OAuth2AccessTokenRedisDAO.java` |
| `SecurityFrameworkServiceImpl` | 实现 `@PreAuthorize` 中的权限校验逻辑（permission、role、scope） | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/service/SecurityFrameworkServiceImpl.java` |

---

## 2. 授权码模式校验顺序详解

### 2.1 第一阶段：/system/oauth2/authorize (GET + POST)

在 `OAuth2OpenController.approveOrDeny()` 方法中，校验顺序如下：

#### 步骤 1：response_type 校验 (Controller 层)
```java
// OAuth2OpenController.java:228
OAuth2GrantTypeEnum grantTypeEnum = getGrantTypeEnum(responseType);
// 只允许 "code" 或 "token"
```

#### 步骤 2：client + redirectUri + scope 综合校验 (ClientService 层)
```java
// OAuth2OpenController.java:230-231
OAuth2ClientDO client = oauth2ClientService.validOAuthClientFromCache(
    clientId, null,
    grantTypeEnum.getGrantType(), 
    scopes.keySet(), 
    redirectUri
);
```

在 `OAuth2ClientServiceImpl.validOAuthClientFromCache()` 内部的严格顺序：
1. **client 存在且启用** → 不存在/禁用报错
2. **client_secret 校验**（如果提供）→ 错误报错
3. **authorizedGrantType 校验** → 不支持的 grant_type 报错
4. **scope 校验** → 请求的 scope 必须是 client 已配置 scope 的子集
5. **redirectUri 校验** → 必须以 client 配置的任一 redirectUri 开头

#### 步骤 3：user approval 用户授权校验 (ApproveService 层)
```java
// OAuth2OpenController.java:234-245
if (Boolean.TRUE.equals(autoApprove)) {
    // 场景一：自动授权
    if (!oauth2ApproveService.checkForPreApproval(...)) {
        return success(null); // 不通过，前端不跳转
    }
} else {
    // 场景二：手动授权
    if (!oauth2ApproveService.updateAfterApproval(...)) {
        return success(错误重定向URL);
    }
}
```

**授权码模式的完整校验顺序**：
```
client_id/response_type 校验 
    ↓
client.validOAuthClientFromCache()：
    (1) client 存在 + 启用
    (2) client_secret（如提供）
    (3) authorizedGrantType 支持
    (4) scope 在 client 范围内
    (5) redirectUri 匹配
    ↓
OAuth2ApproveService：用户授权 approval（自动或手动）
    ↓
生成授权码 code
```

### 2.2 第二阶段：/system/oauth2/token (POST) - 用 code 换 token

在 `OAuth2OpenController.postAccessToken()` 方法中：

#### 步骤 1：grant_type 校验 (Controller 层)
```java
// OAuth2OpenController.java:109-115
OAuth2GrantTypeEnum grantTypeEnum = OAuth2GrantTypeEnum.getByGrantType(grantType);
// 检查枚举存在且不是 implicit 模式
```

#### 步骤 2：client + secret + redirectUri + scope 校验 (ClientService 层)
```java
// OAuth2OpenController.java:118-120
String[] clientIdAndSecret = obtainBasicAuthorization(request);
OAuth2ClientDO client = oauth2ClientService.validOAuthClientFromCache(
    clientIdAndSecret[0], clientIdAndSecret[1],
    grantType, scopes, redirectUri
);
```

#### 步骤 3：授权码 code 消费与二次校验 (GrantService + CodeService 层)
```java
// OAuth2GrantServiceImpl.java:49-70
public OAuth2AccessTokenDO grantAuthorizationCodeForAccessToken(...) {
    // 消费 code（一次性，用完即删）
    OAuth2CodeDO codeDO = oauth2CodeService.consumeAuthorizationCode(code);
    
    // 再次校验 clientId 匹配
    if (!StrUtil.equals(clientId, codeDO.getClientId())) {
        throw exception(OAUTH2_GRANT_CLIENT_ID_MISMATCH);
    }
    // 再次校验 redirectUri 匹配
    if (!StrUtil.equals(redirectUri, codeDO.getRedirectUri())) {
        throw exception(OAUTH2_GRANT_REDIRECT_URI_MISMATCH);
    }
    // 校验 state 匹配
    if (!StrUtil.equals(state, codeDO.getState())) {
        throw exception(OAUTH2_GRANT_STATE_MISMATCH);
    }
}
```

在 `OAuth2CodeServiceImpl.consumeAuthorizationCode()` 内部还会校验：
- code 是否存在
- code 是否已过期（默认 5 分钟）

---

## 3. Access Token 与 Refresh Token 的创建、缓存、过期、刷新关系

### 3.1 创建流程

```java
// OAuth2TokenServiceImpl.java:62-69
@Transactional
public OAuth2AccessTokenDO createAccessToken(...) {
    // 1. 校验 client 存在
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    
    // 2. 先创建 refresh_token
    OAuth2RefreshTokenDO refreshTokenDO = createOAuth2RefreshToken(userId, userType, clientDO, scopes);
    
    // 3. 再基于 refresh_token 创建 access_token
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

**关键属性与关系**：
- `access_token`：UUID，存在 MySQL + Redis
- `refresh_token`：UUID，只存在 MySQL
- `access_token` 引用 `refresh_token`（多对一关系）
- 过期时间取自 `clientDO` 配置：
  - `accessTokenValiditySeconds` → access_token 过期时间
  - `refreshTokenValiditySeconds` → refresh_token 过期时间

### 3.2 缓存策略

**Redis 缓存位置**：`OAuth2AccessTokenRedisDAO`

```java
// OAuth2AccessTokenRedisDAO.java:30-42
public OAuth2AccessTokenDO get(String accessToken) {
    String redisKey = formatKey(accessToken);
    return JsonUtils.parseObject(..., OAuth2AccessTokenDO.class);
}

public void set(OAuth2AccessTokenDO accessTokenDO) {
    // TTL = expiresTime - now，精确到秒
    long time = LocalDateTimeUtil.between(LocalDateTime.now(), accessTokenDO.getExpiresTime(), ChronoUnit.SECONDS);
    if (time > 0) {
        stringRedisTemplate.opsForValue().set(redisKey, json, time, TimeUnit.SECONDS);
    }
}
```

**缓存命中策略**（`OAuth2TokenServiceImpl.getAccessToken()`）：
1. 先查 Redis → 命中直接返回
2. Redis 未命中 → 查 MySQL
3. MySQL 也没找到 → 特殊逻辑：尝试用 `refresh_token` 当 access_token 查（兼容积木报表、WebSocket 等场景）
4. MySQL 命中且未过期 → 回写 Redis 缓存

### 3.3 刷新流程

```java
// OAuth2TokenServiceImpl.java:72-101
@Transactional
public OAuth2AccessTokenDO refreshAccessToken(String refreshToken, String clientId) {
    // 1. 根据 refresh_token 查找
    OAuth2RefreshTokenDO refreshTokenDO = oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    
    // 2. 校验 client 匹配
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    if (ObjectUtil.notEqual(clientId, refreshTokenDO.getClientId())) {
        throw "刷新令牌的客户端编号不正确";
    }
    
    // 3. 删除旧的 access_token（MySQL + Redis 双删）
    List<OAuth2AccessTokenDO> accessTokenDOs = 
        oauth2AccessTokenMapper.selectListByRefreshToken(refreshToken);
    oauth2AccessTokenMapper.deleteByIds(...);
    oauth2AccessTokenRedisDAO.deleteList(...);
    
    // 4. 检查 refresh_token 是否过期
    if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
        oauth2RefreshTokenMapper.deleteById(refreshTokenDO.getId());
        throw "刷新令牌已过期"; // 401
    }
    
    // 5. 创建新的 access_token（复用同一个 refresh_token！）
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

**关键设计点**：
- 刷新时 **refresh_token 保持不变**，只重新生成 access_token
- 旧的 access_token 被立即删除（MySQL + Redis）
- 如果 refresh_token 已过期，也会被从数据库删除

### 3.4 生命周期关系图

```
用户授权通过
    ↓
创建 refresh_token (MySQL) 
    ├── 过期时间：refreshTokenValiditySeconds
    └── ↓
创建 access_token (MySQL + Redis)
    ├── 过期时间：accessTokenValiditySeconds
    ├── 引用：refresh_token
    └── Redis TTL = 过期时间
    ↓
access_token 过期 → 客户端调用 refresh_token 接口
    ↓
删除旧 access_token (MySQL + Redis)
    ↓
创建新 access_token (复用同一个 refresh_token)
    ↓
... 重复 N 次 ...
    ↓
refresh_token 过期 → 抛出 401，需重新登录授权
    ↓
删除 refresh_token + 关联的 access_token
```

---

## 4. checkToken 返回数据被网关和服务端使用的方式

### 4.1 checkToken 接口返回结构

**HTTP 接口**：`POST /system/oauth2/check-token` → `OAuth2OpenCheckTokenRespVO`

**RPC 接口**：`OAuth2TokenCommonApi.checkAccessToken()` → `OAuth2AccessTokenCheckRespDTO`

返回的核心字段：
```
userId        → 用户ID
userType      → 用户类型（ADMIN/MEMBER）
userInfo      → 用户扩展信息（nickname、deptId 等）
tenantId      → 租户ID
scopes        → 授权范围列表
expiresTime   → 过期时间
```

### 4.2 网关层：Gateway TokenAuthenticationFilter

```java
// yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/TokenAuthenticationFilter.java

@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 1. 先移除伪造的 login-user 请求头
    exchange = SecurityFrameworkUtils.removeLoginUser(exchange);
    
    // 2. 从请求中提取 token (Authorization: Bearer xxx)
    String token = SecurityFrameworkUtils.obtainAuthorization(exchange);
    if (StrUtil.isEmpty(token)) {
        return chain.filter(exchange); // 无 token，继续（接口是否需登录由服务端自己判断）
    }
    
    // 3. 通过 WebClient 调用 system 服务的 checkToken 接口
    // 带本地缓存（Guava LoadingCache，1 分钟）
    return getLoginUser(exchange, token).defaultIfEmpty(LOGIN_USER_EMPTY).flatMap(user -> {
        // 4. 如果 token 无效或过期 → 不设置用户信息，继续放行
        if (user == LOGIN_USER_EMPTY || user.getExpiresTime() == null 
                || LocalDateTimeUtils.beforeNow(user.getExpiresTime())) {
            return chain.filter(finalExchange);
        }
        
        // 5. 有效 → 构建 LoginUser 对象
        // 6. 将 LoginUser 序列化后设置到请求头 login-user
        ServerWebExchange newExchange = finalExchange.mutate()
            .request(builder -> SecurityFrameworkUtils.setLoginUserHeader(builder, user)).build();
        
        // 7. 转发给下游服务
        return chain.filter(newExchange);
    });
}
```

**网关行为总结**：
- 有 token 就去校验，但**校验失败不拦截**，只是不设置用户信息
- 校验成功则把 `LoginUser` 放到 `login-user` 请求头（JSON 格式）
- 带 **Guava LoadingCache** 本地缓存（key = tenantId + token，TTL = 1 分钟）
- 缓存处理过期 token 的特殊逻辑：如果 checkToken 返回 401，则缓存 `LOGIN_USER_EMPTY` 占位符

### 4.3 服务端：TokenAuthenticationFilter

```java
// yudao-framework/yudao-spring-boot-starter-security/src/main/java/.../TokenAuthenticationFilter.java

@Override
protected void doFilterInternal(...) {
    // 情况一：优先从请求头 login-user 解析（来自网关透传）
    LoginUser loginUser = buildLoginUserByHeader(request);
    
    // 情况二：如果没有 login-user 头，则自己调用 checkToken
    if (loginUser == null) {
        String token = SecurityFrameworkUtils.obtainAuthorization(request, ...);
        if (StrUtil.isNotEmpty(token)) {
            loginUser = buildLoginUserByToken(token, userType);
            // 模拟登录（开发调试用）
            if (loginUser == null) {
                loginUser = mockLoginUser(request, token, userType);
            }
        }
    }
    
    // 设置到 Spring Security 上下文
    if (loginUser != null) {
        SecurityFrameworkUtils.setLoginUser(loginUser, request);
    }
    chain.doFilter(request, response);
}

private LoginUser buildLoginUserByToken(String token, Integer userType) {
    // 通过 Feign 调用 OAuth2TokenCommonApi.checkAccessToken
    OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
    if (accessToken == null) {
        return null;
    }
    // 校验 userType 匹配（/admin-api/ vs /app-api/）
    if (userType != null && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
        throw new AccessDeniedException("错误的用户类型");
    }
    // 构建 LoginUser
    return new LoginUser()
        .setId(accessToken.getUserId())
        .setUserType(accessToken.getUserType())
        .setInfo(accessToken.getUserInfo())
        .setTenantId(accessToken.getTenantId())
        .setScopes(accessToken.getScopes())  // ← scopes 被放入 LoginUser
        .setExpiresTime(accessToken.getExpiresTime());
}
```

**服务端行为总结**：
- 优先信任网关透传的 `login-user` 头（减少重复 RPC）
- 也支持直连场景（Nginx 直接转发，不走网关）
- `scopes` 字段从 checkToken 结果提取并放入 `LoginUser` 对象，供后续 `@PreAuthorize` 使用
- 校验失败返回 `null`（不拦截，接口是否需登录由 `@PreAuthorize` 或配置决定）

---

## 5. @PreAuthorize 的 hasScope 与普通 permission/role 校验的区别

### 5.1 使用示例对比

```java
// OAuth2UserController.java:53
@PreAuthorize("@ss.hasScope('user.read')")  // scope 校验
public CommonResult<OAuth2UserInfoRespVO> getUserInfo() { ... }

// 普通权限/角色校验示例
@PreAuthorize("@ss.hasPermission('system:user:list')")  // permission 校验
@PreAuthorize("@ss.hasRole('admin')")                  // role 校验
```

### 5.2 实现机制对比

**位置**：`SecurityFrameworkServiceImpl.java`

#### hasScope 实现：
```java
// SecurityFrameworkServiceImpl.java:102-119
@Override
public boolean hasScope(String scope) {
    return hasAnyScopes(scope);
}

@Override
public boolean hasAnyScopes(String... scope) {
    if (skipPermissionCheck()) {
        return true;  // 跨租户访问特殊处理
    }
    
    // 直接从当前线程的 LoginUser 中取 scopes 列表进行本地集合判断
    LoginUser user = SecurityFrameworkUtils.getLoginUser();
    if (user == null) {
        return false;
    }
    return CollUtil.containsAny(user.getScopes(), Arrays.asList(scope));
}
```

#### hasPermission 实现：
```java
// SecurityFrameworkServiceImpl.java:65-78
@Override
public boolean hasAnyPermissions(String... permissions) {
    if (skipPermissionCheck()) {
        return true;
    }
    
    Long userId = getLoginUserId();
    if (userId == null) {
        return false;
    }
    // 走 RPC 调用 PermissionCommonApi，查数据库
    // 带 Guava LoadingCache（key = userId + permissions，TTL = 1 分钟）
    return hasAnyPermissionsCache.get(new KeyValue<>(userId, Arrays.asList(permissions)));
}
```

#### hasRole 实现：
```java
// SecurityFrameworkServiceImpl.java:86-99
@Override
public boolean hasAnyRoles(String... roles) {
    if (skipPermissionCheck()) {
        return true;
    }
    
    Long userId = getLoginUserId();
    if (userId == null) {
        return false;
    }
    // 同样走 RPC 调用 PermissionCommonApi
    // 带 Guava LoadingCache（TTL = 1 分钟）
    return hasAnyRolesCache.get(new KeyValue<>(userId, Arrays.asList(roles)));
}
```

### 5.3 核心区别总结表

| 维度 | hasScope | hasPermission / hasRole |
|------|----------|-------------------------|
| **数据来源** | `LoginUser.getScopes()`（来自 token） | 远程 RPC 调用 `PermissionCommonApi`（查数据库） |
| **作用域** | OAuth2 令牌级别，不同 client 的 token 可获得不同 scope | 系统级 RBAC，同一用户固定的权限/角色 |
| **缓存策略** | 无额外缓存（scopes 已在 LoginUser 内存中） | Guava LoadingCache（1 分钟） |
| **主要场景** | 开放平台 / 第三方应用接入的细粒度控制 | 内部管理后台的常规权限控制 |
| **是否需要用户登录** | 是（需要 LoginUser） | 是（需要 userId） |
| **灵活性** | 同一用户对不同 client 可授权不同 scope | 用户权限相对固定 |

### 5.4 典型使用场景

```
内部管理后台接口（管理员操作）:
    → 使用 hasPermission / hasRole
    → 依据：用户在系统中的角色和权限配置
    
OAuth2 开放接口（给第三方应用调用）:
    → 使用 hasScope
    → 依据：第三方应用在 OAuth2 授权时用户同意的 scope
    → 例如：user.read, user.write, order.read 等
```

---

## 6. 各类错误在哪一层报错

### 6.1 scope 不足

**报错位置**：**服务端方法调用层**（`@PreAuthorize` AOP 拦截）

- 触发点：`SecurityFrameworkServiceImpl.hasScope()` → `CollUtil.containsAny()` 返回 false
- 表现：Spring Security 的 `AccessDeniedException`，最终响应 403 Forbidden
- **不会**在 checkToken 或 token 创建时报错
- 只有当实际访问被 `@PreAuthorize("@ss.hasScope('xxx')")` 保护的接口时才校验

### 6.2 token 过期

**分两个阶段**：

**阶段一：checkToken 时** → **TokenService 层**
```java
// OAuth2TokenServiceImpl.java:131-140
public OAuth2AccessTokenDO checkAccessToken(String accessToken) {
    OAuth2AccessTokenDO accessTokenDO = getAccessToken(accessToken);
    if (accessTokenDO == null) {
        throw exception0(UNAUTHORIZED, "访问令牌不存在");  // 401
    }
    if (DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        throw exception0(UNAUTHORIZED, "访问令牌已过期");  // 401
    }
    return accessTokenDO;
}
```

**阶段二：refresh_token 刷新时** → **TokenService 层**
```java
// OAuth2TokenServiceImpl.java:93-97
if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
    oauth2RefreshTokenMapper.deleteById(refreshTokenDO.getId());
    throw exception0(UNAUTHORIZED, "刷新令牌已过期");  // 401
}
```

### 6.3 client 不匹配

**分多个位置**：

**位置 1：token 接口 client 校验** → **ClientService 层**
```java
// OAuth2ClientServiceImpl.java:122-149
public OAuth2ClientDO validOAuthClientFromCache(...) {
    // client 不存在
    if (client == null) {
        throw exception(OAUTH2_CLIENT_NOT_EXISTS);
    }
    // client 禁用
    if (CommonStatusEnum.isDisable(client.getStatus())) {
        throw exception(OAUTH2_CLIENT_DISABLE);
    }
    // secret 错误
    if (StrUtil.isNotEmpty(clientSecret) && ObjectUtil.notEqual(client.getSecret(), clientSecret)) {
        throw exception(OAUTH2_CLIENT_CLIENT_SECRET_ERROR);
    }
}
```

**位置 2：授权码换 token 时二次校验** → **GrantService 层**
```java
// OAuth2GrantServiceImpl.java:53-56
if (!StrUtil.equals(clientId, codeDO.getClientId())) {
    throw exception(OAUTH2_GRANT_CLIENT_ID_MISMATCH);
}
```

**位置 3：refresh_token 刷新时** → **TokenService 层**
```java
// OAuth2TokenServiceImpl.java:81-84
if (ObjectUtil.notEqual(clientId, refreshTokenDO.getClientId())) {
    throw exception0(BAD_REQUEST, "刷新令牌的客户端编号不正确");  // 400
}
```

**位置 4：revokeToken 时** → **GrantService 层**
```java
// OAuth2GrantServiceImpl.java:94-102
OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.getAccessToken(accessToken);
if (accessTokenDO == null || ObjectUtil.notEqual(clientId, accessTokenDO.getClientId())) {
    return false;  // 静默失败，返回 false
}
```

### 6.4 redirectUri 不匹配

**分两个位置**：

**位置 1：authorize 阶段** → **ClientService 层**
```java
// OAuth2ClientServiceImpl.java:145-148
if (StrUtil.isNotEmpty(redirectUri) && !StrUtils.startWithAny(redirectUri, client.getRedirectUris())) {
    throw exception(OAUTH2_CLIENT_REDIRECT_URI_NOT_MATCH, redirectUri);
}
```

**位置 2：授权码换 token 二次校验** → **GrantService 层**
```java
// OAuth2GrantServiceImpl.java:57-60
if (!StrUtil.equals(redirectUri, codeDO.getRedirectUri())) {
    throw exception(OAUTH2_GRANT_REDIRECT_URI_MISMATCH);
}
```

### 6.5 错误层级汇总表

| 错误类型 | 报错层级 | 核心方法/位置 | 错误码特征 |
|----------|----------|---------------|------------|
| **scope 不足** | 服务端方法调用层（AOP） | `SecurityFrameworkServiceImpl.hasScope()` → `@PreAuthorize` | 403 AccessDenied |
| **access_token 过期** | TokenService 层 | `OAuth2TokenServiceImpl.checkAccessToken()` | 401 |
| **refresh_token 过期** | TokenService 层 | `OAuth2TokenServiceImpl.refreshAccessToken()` | 401 |
| **client 不存在/禁用** | ClientService 层 | `OAuth2ClientServiceImpl.validOAuthClientFromCache()` | OAUTH2_CLIENT_NOT_EXISTS / DISABLE |
| **client_secret 错误** | ClientService 层 | `OAuth2ClientServiceImpl.validOAuthClientFromCache()` | OAUTH2_CLIENT_CLIENT_SECRET_ERROR |
| **client_id 不匹配（code 换 token）** | GrantService 层 | `OAuth2GrantServiceImpl.grantAuthorizationCodeForAccessToken()` | OAUTH2_GRANT_CLIENT_ID_MISMATCH |
| **client_id 不匹配（refresh）** | TokenService 层 | `OAuth2TokenServiceImpl.refreshAccessToken()` | 400 BAD_REQUEST |
| **redirectUri 不匹配（authorize）** | ClientService 层 | `OAuth2ClientServiceImpl.validOAuthClientFromCache()` | OAUTH2_CLIENT_REDIRECT_URI_NOT_MATCH |
| **redirectUri 不匹配（code 换 token）** | GrantService 层 | `OAuth2GrantServiceImpl.grantAuthorizationCodeForAccessToken()` | OAUTH2_GRANT_REDIRECT_URI_MISMATCH |
| **grant_type 不支持** | ClientService 层 | `OAuth2ClientServiceImpl.validOAuthClientFromCache()` | OAUTH2_CLIENT_AUTHORIZED_GRANT_TYPE_NOT_EXISTS |
| **请求 scope 超出 client 配置** | ClientService 层 | `OAuth2ClientServiceImpl.validOAuthClientFromCache()` | OAUTH2_CLIENT_SCOPE_OVER |
| **授权码不存在/过期** | CodeService 层 | `OAuth2CodeServiceImpl.consumeAuthorizationCode()` | OAUTH2_CODE_NOT_EXISTS / EXPIRE |
| **state 不匹配** | GrantService 层 | `OAuth2GrantServiceImpl.grantAuthorizationCodeForAccessToken()` | OAUTH2_GRANT_STATE_MISMATCH |
| **用户拒绝授权** | Controller 层 | `OAuth2OpenController.approveOrDeny()` → 跳转 `access_denied` | 重定向，非异常 |

---

## 7. 完整链路时序图（授权码模式）

```
[第三方应用]                              [yudao-gateway]              [system-service]
     |                                          |                              |
     |---- GET /system/oauth2/authorize ------>|                              |
     |  ?client_id=xxx&redirect_uri=yyy         |                              |
     |                                          |-----> 用户需先登录 --------->|
     |<----- 登录页面（前端处理）---------------|                              |
     |                                          |                              |
     |---- POST /system/oauth2/authorize ----->|                              |
     |  client_id + redirect_uri + scope       |                              |
     |                                          |---- OAuth2OpenController    |
     |                                          |      .approveOrDeny()        |
     |                                          |         ↓                   |
     |                                          |   validOAuthClientFromCache |
     |                                          |     (client+redirect+scope) |
     |                                          |         ↓                   |
     |                                          |   OAuth2ApproveService      |
     |                                          |     (用户授权确认)            |
     |                                          |         ↓                   |
     |                                          |   OAuth2CodeService         |
     |                                          |     createAuthorizationCode |
     |<----- 重定向 redirect_uri?code=xxx ------|                              |
     |                                          |                              |
     |                                          |                              |
     |---- POST /system/oauth2/token --------->|                              |
     |  grant_type=authorization_code           |                              |
     |  code + redirect_uri + Basic Auth       |                              |
     |                                          |---- OAuth2OpenController    |
     |                                          |      .postAccessToken()      |
     |                                          |         ↓                   |
     |                                          |   validOAuthClientFromCache |
     |                                          |     (client+secret+...)     |
     |                                          |         ↓                   |
     |                                          |   OAuth2GrantService        |
     |                                          |      consumeAuthorizationCode|
     |                                          |         ↓                   |
     |                                          |   校验 clientId/redirectUri/state |
     |                                          |         ↓                   |
     |                                          |   OAuth2TokenService        |
     |                                          |     createAccessToken       |
     |                                          |       → createRefreshToken  |
     |                                          |       → createAccessToken   |
     |                                          |       → Redis 缓存          |
     |<----- {access_token, refresh_token, -----|                              |
     |       expires_in, scope} ---------------|                              |
     |                                          |                              |
     |                                          |                              |
     |---- GET /system/oauth2/user/get ------->|                              |
     |  Authorization: Bearer <access_token>   |                              |
     |                                          |---- Gateway                  |
     |                                          |   TokenAuthenticationFilter  |
     |                                          |      checkAccessToken        |
     |                                          |      (带本地缓存)             |
     |                                          |         ↓                   |
     |                                          |   构造 LoginUser             |
     |                                          |   设置 login-user header     |
     |                                          |         ↓                   |
     |                                          |---- 服务端                   |
     |                                          |   TokenAuthenticationFilter  |
     |                                          |   从 header 解析 LoginUser   |
     |                                          |         ↓                   |
     |                                          |   @PreAuthorize             |
     |                                          |   @ss.hasScope('user.read') |
     |                                          |         ↓                   |
     |                                          |   SecurityFrameworkService  |
     |                                          |   hasScope() 检查 scopes    |
     |<----- 用户信息 --------------------------|                              |
     |                                          |                              |
     |---- POST /system/oauth2/token --------->|                              |
     |  grant_type=refresh_token                |                              |
     |  refresh_token + Basic Auth             |                              |
     |                                          |---- OAuth2TokenService      |
     |                                          |   refreshAccessToken        |
     |                                          |     → 删除旧 access_token   |
     |                                          |     → 创建新 access_token   |
     |                                          |     → (refresh_token 不变)  |
     |<----- 新的 access_token -----------------|                              |
```

---

## 8. 关键设计洞察

1. **双重校验设计**：授权码模式在 authorize 和 token 两个阶段都会校验 client 和 redirectUri，防止中间篡改
2. **授权码一次性消费**：`consumeAuthorizationCode()` 读取后立即删除，防止重放攻击
3. **Token 双存储**：access_token 同时存在 MySQL（持久化）和 Redis（高性能读取）
4. **Redis TTL 与过期时间同步**：`OAuth2AccessTokenRedisDAO` 精确计算 TTL，避免缓存过期的 token
5. **刷新令牌复用机制**：refresh 时只重新生成 access_token，refresh_token 保持不变，减少数据库写入
6. **网关不拦截无效 token**：网关只负责解析和透传，是否需要登录由服务端通过 `@PreAuthorize` 或安全配置决定
7. **scope 与 permission 分层**：scope 是 OAuth2 令牌级别的（第三方应用维度），permission 是系统 RBAC 级别的（用户维度）
8. **checkToken 结果的 scopes 透传链路**：system-service → gateway → downstream-service → LoginUser → @PreAuthorize
