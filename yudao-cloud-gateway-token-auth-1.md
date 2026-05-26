# yudao-cloud 网关 Token 认证流程分析

## 一、整体架构概览

一次携带 Authorization token 的请求从进入 yudao-gateway 到转发到后端服务，登录态的建立、缓存、透传和兜底校验涉及以下核心组件：

| 组件 | 所属模块 | 核心职责 |
|------|----------|----------|
| TokenAuthenticationFilter | yudao-gateway | 网关层 token 校验、缓存、用户信息透传 |
| TokenAuthenticationFilter | yudao-spring-boot-starter-security | 服务层登录态建立、SecurityContext 填充 |
| SecurityFrameworkUtils | yudao-gateway | 网关层安全工具：token 提取、login-user header 操作 |
| SecurityFrameworkUtils | yudao-spring-boot-starter-security | 服务层安全工具：SecurityContext 读写 |
| WebFrameworkUtils | yudao-gateway | 网关层 Web 工具：租户 ID 提取 |
| WebFrameworkUtils | yudao-spring-boot-starter-web | 服务层 Web 工具：URL 前缀推断 userType |

---

## 二、请求处理流程详解

### 2.1 网关层处理流程

**文件位置**：`yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/TokenAuthenticationFilter.java`

```
请求进入网关
    ↓
[步骤1] 移除 login-user header（防止伪造）
    ↓
[步骤2] 从 Authorization header 解析 token
    ↓
[步骤3] 构建缓存 key = KeyValue<tenantId, token>
    ↓
[步骤4] 先查本地 LoadingCache（1分钟缓存，异步刷新）
    ↓ 缓存未命中
[步骤5] 调用 checkAccessToken 远程校验 token
    ↓
[步骤6] buildUser 解析结果
    ├─ 成功 → 缓存 LoginUser → 设置 login-user header → 转发
    ├─ 401过期 → 返回 LOGIN_USER_EMPTY → 不设置 header → 继续转发
    ├─ 其它错误 → 返回 null → 不设置 header → 继续转发
    └─ 解析失败 → 返回 null → 不设置 header → 继续转发
```

**关键代码说明**：

```java
// TokenAuthenticationFilter.java:85-86
// 移除 login-user 的请求头，避免伪造模拟
exchange = SecurityFrameworkUtils.removeLoginUser(exchange);

// TokenAuthenticationFilter.java:115-116
// 缓存 key 包含 tenantId 和 token
Long tenantId = WebFrameworkUtils.getTenantId(exchange);
KeyValue<Long, String> cacheKey = new KeyValue<Long, String>().setKey(tenantId).setValue(token);

// TokenAuthenticationFilter.java:141-161
// buildUser 方法的错误处理逻辑
private LoginUser buildUser(String body) {
    CommonResult<OAuth2AccessTokenCheckRespDTO> result = JsonUtils.parseObject(body, CHECK_RESULT_TYPE_REFERENCE);
    if (result == null) return null;                    // 解析失败
    if (result.isError()) {
        if (Objects.equals(result.getCode(), HttpStatus.UNAUTHORIZED.value())) {
            return LOGIN_USER_EMPTY;                    // 401 过期，特殊标记
        }
        return null;                                    // 其它错误
    }
    // 成功：构建 LoginUser
    return new LoginUser().setId(tokenInfo.getUserId())...
}
```

### 2.2 服务层处理流程

**文件位置**：`yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/filter/TokenAuthenticationFilter.java`

```
请求到达后端服务
    ↓
[步骤1] 优先从 login-user header 解析（网关透传场景）
    ├─ 解析成功 → 校验 userType → 设置到 SecurityContext
    └─ 解析失败/不存在 → 进入步骤2
    ↓
[步骤2] 兜底：从 Authorization header 直接校验（绕过网关直连场景）
    ├─ 校验成功 → 校验 userType → 设置到 SecurityContext
    └─ 校验失败 → 不设置 SecurityContext → 继续请求
    ↓
[步骤3] 继续 FilterChain（业务接口由 Spring Security 配置决定是否放行）
```

**关键代码说明**：

```java
// TokenAuthenticationFilter.java:50-73
protected void doFilterInternal(HttpServletRequest request, ...) {
    // 情况一：优先从 header[login-user] 获得用户（网关透传）
    LoginUser loginUser = buildLoginUserByHeader(request);
    
    // 情况二：兜底，基于 Token 获得用户（绕过网关直连）
    if (loginUser == null) {
        String token = SecurityFrameworkUtils.obtainAuthorization(request, ...);
        if (StrUtil.isNotEmpty(token)) {
            Integer userType = WebFrameworkUtils.getLoginUserType(request);
            loginUser = buildLoginUserByToken(token, userType);
            if (loginUser == null) {
                loginUser = mockLoginUser(request, token, userType);  // 开发调试用
            }
        }
    }
    
    if (loginUser != null) {
        SecurityFrameworkUtils.setLoginUser(loginUser, request);
    }
    chain.doFilter(request, response);
}
```

### 2.3 SecurityFrameworkUtils 协作

**网关版**：`yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/SecurityFrameworkUtils.java`

- `obtainAuthorization()`：从 `Authorization: Bearer xxx` 提取 token
- `removeLoginUser()`：移除 `login-user` header（防伪造）
- `setLoginUser()`：设置到 exchange attributes（网关内部使用）
- `setLoginUserHeader()`：将 LoginUser JSON 编码后设置到 `login-user` header（透传给后端）

**服务版**：`yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/util/SecurityFrameworkUtils.java`

- `obtainAuthorization()`：从 Header/Parameter 提取 token（支持 WebSocket 参数方式）
- `setLoginUser()`：将 LoginUser 包装为 Authentication 放入 SecurityContextHolder
- `getLoginUser()`：从 SecurityContext 读取当前用户

### 2.4 WebFrameworkUtils 协作

**网关版**：`yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/WebFrameworkUtils.java`

- `getTenantId()`：从 `tenant-id` header 提取租户 ID
- `setTenantIdHeader()`：设置 `tenant-id` header（远程调用 checkAccessToken 时使用）

**服务版**：`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/util/WebFrameworkUtils.java`

- `getLoginUserType()`：**核心逻辑**——基于 URL 前缀推断 userType
  - `/admin-api/*` → `UserTypeEnum.ADMIN`（管理员）
  - `/app-api/*` → `UserTypeEnum.MEMBER`（会员/用户）
  - 其它路径（如 `/ws/*`）→ `null`（不校验）

```java
// WebFrameworkUtils.java:114-121
public static Integer getLoginUserType(HttpServletRequest request) {
    // 1. 优先，从 Attribute 中获取
    Integer userType = (Integer) request.getAttribute(REQUEST_ATTRIBUTE_LOGIN_USER_TYPE);
    if (userType != null) return userType;
    // 2. 其次，基于 URL 前缀的约定
    if (request.getServletPath().startsWith(properties.getAdminApi().getPrefix())) {
        return UserTypeEnum.ADMIN.getValue();
    }
    if (request.getServletPath().startsWith(properties.getAppApi().getPrefix())) {
        return UserTypeEnum.MEMBER.getValue();
    }
    return null;
}
```

---

## 三、问题解答

### 1）网关为什么先移除 login-user header？

**原因分析**：

- **安全防伪造**：`login-user` header 是网关与后端服务之间的"内部信任协议"。如果不移除，恶意客户端可以直接在请求中携带伪造的 `login-user` header（如 `{"id":1,"userType":1}`），跳过 token 校验直接获得管理员权限。

- **网关是唯一可信入口**：只有网关才有权限校验 token 并生成 `login-user` header。后端服务信任这个 header 是因为它知道这个 header 只能来自网关（前提是网络层面限制服务端口不对外暴露）。

- **代码注释佐证**：`TokenAuthenticationFilter.java:85-86` 明确写着"移除 login-user 的请求头，避免伪造模拟"。

**网关安全策略**：
```
客户端请求（可携带任意 header）
    ↓
网关移除 login-user header（清空不可信来源）
    ↓
网关校验 token 成功 → 重新生成 login-user header（可信来源）
    ↓
转发给后端服务（后端信任这个 header）
```

---

### 2）网关本地 LoadingCache 的 key 为什么包含 tenantId 和 token？

**缓存定义**：
```java
// TokenAuthenticationFilter.java:64
private final LoadingCache<KeyValue<Long, String>, LoginUser> loginUserCache = 
    buildAsyncReloadingCache(Duration.ofMinutes(1), new CacheLoader<>() {
        public LoginUser load(KeyValue<Long, String> token) {
            // token.getKey() = tenantId, token.getValue() = accessToken
            String body = checkAccessToken(token.getKey(), token.getValue()).block();
            return buildUser(body);
        }
    });
```

**原因分析**：

- **多租户隔离**：yudao-cloud 是多租户 SaaS 架构。同一个 accessToken 在不同租户下可能代表不同的用户，或者说不同租户的 token 管理是隔离的。

- **checkAccessToken 需要 tenant-id**：远程校验 token 时，网关通过 `tenant-id` header 告诉 system 服务当前属于哪个租户。system 服务的 token 校验逻辑是租户隔离的（查看 `tenant_id` 字段）。

- **缓存命中率与正确性平衡**：
  - 如果只用 token 做 key：不同租户相同 token 会命中错误缓存
  - 如果只用 tenantId 做 key：同一个租户下不同 token 会互相覆盖
  - 用组合 key：每个租户的每个 token 独立缓存，既正确又高效

**远程调用佐证**：
```java
// TokenAuthenticationFilter.java:135-138
return webClient.get()
    .uri(OAuth2TokenCommonApi.URL_CHECK, ...)
    .headers(httpHeaders -> WebFrameworkUtils.setTenantIdHeader(tenantId, httpHeaders))
    .retrieve().bodyToMono(String.class);
```

---

### 3）checkAccessToken 返回 401、其它错误、null、过期 token、缓存命中 LOGIN_USER_EMPTY 分别如何影响后续请求？

**设计原则**：网关只做 token 校验和信息透传，**不拦截请求**。接口是否需要登录的校验交给后端服务自身的 Spring Security 配置处理。

| 场景 | 网关行为 | 对后续请求的影响 |
|------|----------|------------------|
| **401（token 过期）** | `buildUser` 返回 `LOGIN_USER_EMPTY`，不设置 `login-user` header | 请求继续转发，后端服务的 `buildLoginUserByHeader` 获取不到用户，走兜底 `buildLoginUserByToken` 也会失败。最终 SecurityContext 为空，由后端 Spring Security 决定是否放行（`permitAll` 接口可通过，需登录接口返回 401） |
| **其它错误（如 token 不存在）** | `buildUser` 返回 `null`，不设置 `login-user` header | 同上，请求继续，后端兜底校验也失败 |
| **null（响应解析失败）** | `buildUser` 返回 `null`，不设置 `login-user` header | 同上，请求继续，视为无登录态 |
| **过期 token（expiresTime 已过）** | 即使缓存中有数据，`filter()` 方法也会检查 `expiresTime`，若已过期则不设置 header | 同上，请求继续，无登录态 |
| **缓存命中 LOGIN_USER_EMPTY** | 命中后继续走 `flatMap`，判断 `user == LOGIN_USER_EMPTY` 为 true，不设置 header | 同上，请求继续，无登录态 |

**关键判断逻辑**：
```java
// TokenAuthenticationFilter.java:97-102
return getLoginUser(exchange, token).defaultIfEmpty(LOGIN_USER_EMPTY).flatMap(user -> {
    // 无用户，直接 filter 继续请求
    if (user == LOGIN_USER_EMPTY || 
            user.getExpiresTime() == null || LocalDateTimeUtils.beforeNow(user.getExpiresTime())) {
        return chain.filter(finalExchange);  // 继续转发，不拦截
    }
    // 有用户，设置 login-user header 后转发
    SecurityFrameworkUtils.setLoginUser(finalExchange, user);
    ServerWebExchange newExchange = finalExchange.mutate()
            .request(builder -> SecurityFrameworkUtils.setLoginUserHeader(builder, user)).build();
    return chain.filter(newExchange);
});
```

**LOGIN_USER_EMPTY 的特殊作用**（代码注释说明）：

1. 解决 `Mono.empty()` 导致后续 `flatMap` 无法处理的问题（用 `defaultIfEmpty(LOGIN_USER_EMPTY)` 兜底）
2. token 过期时缓存 `LOGIN_USER_EMPTY`，避免缓存一直持有过期状态被误判为有效（下次缓存命中后仍会正确判断为无效）

---

### 4）后端服务 buildLoginUserByHeader 与 buildLoginUserByToken 的优先级？

**优先级结论**：`buildLoginUserByHeader` > `buildLoginUserByToken`

**执行顺序**：
```java
// TokenAuthenticationFilter.java:50-73
LoginUser loginUser = buildLoginUserByHeader(request);  // 步骤1：优先走这个

if (loginUser == null) {                                // 步骤2：只有 null 才走兜底
    String token = SecurityFrameworkUtils.obtainAuthorization(request, ...);
    if (StrUtil.isNotEmpty(token)) {
        loginUser = buildLoginUserByToken(token, userType);
    }
}
```

**设计意图**：

- **正常流量（经网关）**：请求经过网关时，网关已完成 token 校验并在 `login-user` header 中透传了完整用户信息。后端直接解析 header 即可，无需再次远程调用校验 token——**性能优化**。

- **旁路流量（绕过网关）**：开发/测试环境可能直接请求服务端口（绕过网关），此时没有 `login-user` header，需要 `buildLoginUserByToken` 兜底，直接校验 token——**开发便利**。

**两者差异对比**：

| 维度 | buildLoginUserByHeader | buildLoginUserByToken |
|------|------------------------|-----------------------|
| 触发场景 | 请求经网关转发 | 绕过网关直连服务 |
| 数据来源 | `login-user` header（JSON） | `Authorization` header（token） |
| 是否远程调用 | 否（直接解析 JSON） | 是（调用 `oauth2TokenApi.checkAccessToken`） |
| userType 校验 | 有（对比 header 中 userType 与 URL 前缀推断） | 有（对比 token 返回 userType 与 URL 前缀推断） |

---

### 5）userType 在经过网关和绕过网关直连服务时分别在哪里校验？

**userType 校验的统一逻辑**：基于 URL 前缀推断期望的 userType，然后与实际登录用户的 userType 对比。

#### 场景 A：经过网关

```
请求经网关 → login-user header 中包含 userType → 后端 buildLoginUserByHeader 校验
```

**校验位置**：`yudao-framework/yudao-spring-boot-starter-security/.../TokenAuthenticationFilter.java:133-155`

```java
private LoginUser buildLoginUserByHeader(HttpServletRequest request) {
    String loginUserStr = request.getHeader(SecurityFrameworkUtils.LOGIN_USER_HEADER);
    if (StrUtil.isEmpty(loginUserStr)) return null;
    
    LoginUser loginUser = JsonUtils.parseObject(loginUserStr, LoginUser.class);
    
    // userType 校验
    Integer userType = WebFrameworkUtils.getLoginUserType(request);  // 从 URL 前缀推断
    if (userType != null && loginUser != null 
            && ObjectUtil.notEqual(loginUser.getUserType(), userType)) {
        throw new AccessDeniedException("错误的用户类型");  // 直接抛出，中断请求
    }
    return loginUser;
}
```

#### 场景 B：绕过网关直连服务

```
请求直连服务 → 无 login-user header → 后端 buildLoginUserByToken 校验
```

**校验位置**：`yudao-framework/yudao-spring-boot-starter-security/.../TokenAuthenticationFilter.java:83-106`

```java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    try {
        OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
        if (accessToken == null) return null;
        
        // userType 校验
        if (userType != null 
                && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
            throw new AccessDeniedException("错误的用户类型");  // 直接抛出，中断请求
        }
        return new LoginUser().setId(accessToken.getUserId())...
    } catch (ServiceException serviceException) {
        return null;  // token 校验失败，返回 null（视为未登录）
    }
}
```

**共同特点**：

- `userType = null` 时不校验（如 WebSocket `/ws/*` 路径）
- 校验失败直接抛出 `AccessDeniedException`，由 `GlobalExceptionHandler` 返回错误响应
- 网关**不校验** userType，只透传；userType 校验统一在**后端服务**完成

---

### 6）哪些情况下请求会继续到业务接口但 SecurityContext 中没有登录用户？

**核心机制**：网关和 TokenAuthenticationFilter 都**不拦截无登录态的请求**，只负责建立登录态。最终是否拦截由 Spring Security 的 `HttpSecurity` 配置（`antMatchers().permitAll()` / `authenticated()`）决定。

#### 情况列表：

| 情况 | 原因说明 | 对业务的影响 |
|------|----------|--------------|
| **无 token** | 请求头没有 `Authorization`，或解析不出有效 token | SecurityContext 为空。业务代码 `SecurityFrameworkUtils.getLoginUser()` 返回 null |
| **token 无效** | token 不存在、已过期、签名错误等，网关 `buildUser` 返回 null 或 LOGIN_USER_EMPTY，后端兜底校验也失败 | 同上，视为未登录 |
| **token 校验异常被 catch** | `buildLoginUserByToken` 中 `ServiceException` 被 catch，返回 null | 同上 |
| **token 对应用户类型不匹配** | 如：管理员 token 请求 `/app-api/*` 路径，抛出 `AccessDeniedException` | **特殊**：直接返回错误，不会到业务接口 |
| **接口配置为 permitAll** | Spring Security 配置了 `.antMatchers("/public/**").permitAll()` | 即使无登录态也放行。业务代码需主动判断 `getLoginUser() == null` |
| **mock 模式关闭且 token 不是 mock 格式** | 开发环境 `mockEnable=false`，但前端传了一个看起来像 token 但实际无效的值 | 网关校验失败 → 后端兜底也失败 → SecurityContext 为空 |

#### 关键代码佐证：

**网关不拦截**：
```java
// TokenAuthenticationFilter.java:99-101
if (user == LOGIN_USER_EMPTY || user.getExpiresTime() == null 
        || LocalDateTimeUtils.beforeNow(user.getExpiresTime())) {
    return chain.filter(finalExchange);  // 无效登录态也继续转发，不拦截
}
```

**服务层 TokenAuthenticationFilter 也不拦截**：
```java
// TokenAuthenticationFilter.java:75-80
if (loginUser != null) {
    SecurityFrameworkUtils.setLoginUser(loginUser, request);
}
// 无论 loginUser 是否 null，都继续 FilterChain
chain.doFilter(request, response);
```

**真正的拦截发生在 Spring Security FilterChainProxy**：

- 对于 `.antMatchers("/admin-api/**").authenticated()` 配置的路径：`FilterSecurityInterceptor` 发现 SecurityContext 为空 → 抛出 `AuthenticationCredentialsNotFoundException` → 返回 401

- 对于 `.antMatchers("/admin-api/login").permitAll()` 配置的路径：直接放行，即使 SecurityContext 为空也能到业务 Controller

#### 业务代码如何感知？

```java
@GetMapping("/profile")
public CommonResult<UserProfileRespVO> getProfile() {
    LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
    if (loginUser == null) {
        // 说明：1）接口配置了 permitAll 且请求无登录态
        // 或 2）token 无效但接口 permitAll
        return error(UNAUTHORIZED);  // 业务层自己决定是否需要登录
    }
    return success(userService.getProfile(loginUser.getId()));
}
```

---

## 四、登录态流转时序图

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐     ┌─────────────┐
│  客户端   │────>│ yudao-gateway │────>│ 后端服务(如system)│────>│ 业务Controller│
└──────────┘     └───────────────┘     └──────────────────┘     └─────────────┘
      │                  │                        │                      │
      │ Authorization:   │                        │                      │
      │ Bearer xxx       │                        │                      │
      │─────────────────>│                        │                      │
      │                  │ 1. removeLoginUser()   │                      │
      │                  │    (防伪造)             │                      │
      │                  │                        │                      │
      │                  │ 2. obtainAuthorization │                      │
      │                  │    (提取token)          │                      │
      │                  │                        │                      │
      │                  │ 3. LoadingCache        │                      │
      │                  │    key=tenantId+token  │                      │
      │                  │                        │                      │
      │                  │ 4. checkAccessToken()  │                      │
      │                  │    (远程校验token)      │                      │
      │                  │───────────────────────>│                      │
      │                  │<───────────────────────│                      │
      │                  │    {userId,userType,   │                      │
      │                  │     tenantId,expires}   │                      │
      │                  │                        │                      │
      │                  │ 5. setLoginUserHeader  │                      │
      │                  │    login-user: JSON    │                      │
      │                  │    (URLEncode编码)      │                      │
      │                  │──────────────────────────────────────────────>│
      │                  │                        │                      │
      │                  │                        │ 1. buildLoginUserBy  │
      │                  │                        │    Header            │
      │                  │                        │    (解析login-user)  │
      │                  │                        │                      │
      │                  │                        │ 2. 校验userType      │
      │                  │                        │    (URL前缀推断)      │
      │                  │                        │                      │
      │                  │                        │ 3. setLoginUser()    │
      │                  │                        │    (放入Security     │
      │                  │                        │     Context)         │
      │                  │                        │                      │
      │                  │                        │ 4. Spring Security  │
      │                  │                        │    判定是否放行       │
      │                  │                        │─────────────────────>│
      │                  │                        │                      │ 业务代码可通过
      │                  │                        │                      │ SecurityFramework-
      │                  │                        │                      │ Utils.getLoginUser()
      │                  │                        │                      │ 获取当前用户
```

---

## 五、设计亮点与安全考量

### 5.1 网关层设计亮点

1. **信任边界清晰**：网关是唯一的 token 校验点，后端服务信任网关透传的 `login-user` header
2. **防伪造机制**：先移除再重建 `login-user` header
3. **本地缓存**：1分钟 LoadingCache 减少远程调用，异步刷新保证缓存新鲜度
4. **失败不拦截**：网关不做权限决策，只做信息透传，权限决策下沉到服务层

### 5.2 服务层设计亮点

1. **双路径支持**：既支持网关透传（高效），也支持直连（开发便利）
2. **URL 约定优于配置**：通过 `/admin-api` / `/app-api` 前缀自动推断 userType
3. **统一 userType 校验**：无论走哪条路径，userType 校验逻辑一致
4. **mock 调试模式**：开发环境可配置 `yudao.security.mockEnable=true` 绕过 token 校验

### 5.3 安全边界

| 组件 | 安全职责 |
|------|----------|
| 网关 | token 有效性校验、防 header 伪造、租户隔离 |
| 后端 TokenAuthenticationFilter | userType 校验、SecurityContext 建立 |
| Spring Security | 接口级权限控制（permitAll vs authenticated） |
| 业务层 | 可通过 `SecurityFrameworkUtils.getLoginUser()` 做更细粒度控制 |

---

## 六、关键文件索引

| 文件 | 路径 |
|------|------|
| 网关 TokenAuthenticationFilter | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/TokenAuthenticationFilter.java` |
| 服务 TokenAuthenticationFilter | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/filter/TokenAuthenticationFilter.java` |
| 网关 SecurityFrameworkUtils | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/SecurityFrameworkUtils.java` |
| 服务 SecurityFrameworkUtils | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/util/SecurityFrameworkUtils.java` |
| 网关 WebFrameworkUtils | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/WebFrameworkUtils.java` |
| 服务 WebFrameworkUtils | `yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/util/WebFrameworkUtils.java` |
| OAuth2TokenCommonApi 接口 | `yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/biz/system/oauth2/OAuth2TokenCommonApi.java` |
| SecurityProperties 配置 | `yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/config/SecurityProperties.java` |
