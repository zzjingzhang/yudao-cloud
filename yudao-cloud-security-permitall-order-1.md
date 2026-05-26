# Yudao-Cloud URL 安全配置深入分析

## 概述

本文档深入分析 yudao-cloud 项目中 `YudaoWebSecurityConfigurerAdapter` 的 URL 安全配置机制，包括 `TokenAuthenticationFilter`、`AuthenticationEntryPointImpl`、`AccessDeniedHandlerImpl` 以及 `@PermitAll` 注解之间的协作关系。

---

## 1. getPermitAllUrlsFromAnnotations 如何从 HandlerMethod 收集类级和方法级 @PermitAll

### 核心机制

`getPermitAllUrlsFromAnnotations()` 方法位于 `YudaoWebSecurityConfigurerAdapter.java:159-219`，负责从 Spring MVC 的请求映射信息中收集所有带有 `@PermitAll` 注解的接口。

### 工作流程

#### 步骤 1：获取所有请求映射

```java
RequestMappingHandlerMapping requestMappingHandlerMapping = (RequestMappingHandlerMapping)
        applicationContext.getBean("requestMappingHandlerMapping");
Map<RequestMappingInfo, HandlerMethod> handlerMethodMap = requestMappingHandlerMapping.getHandlerMethods();
```

- 通过 `RequestMappingHandlerMapping` 获取所有已注册的请求映射
- `handlerMethodMap` 的 key 是 `RequestMappingInfo`（包含 URL 模式、HTTP 方法等），value 是 `HandlerMethod`（控制器方法的元信息）

#### 步骤 2：双重检查 @PermitAll 注解

```java
HandlerMethod handlerMethod = entry.getValue();
if (!handlerMethod.hasMethodAnnotation(PermitAll.class) // 方法级
        && !handlerMethod.getBeanType().isAnnotationPresent(PermitAll.class)) { // 类级
    continue;
}
```

- **OR 逻辑**：只要方法级或类级任一存在 `@PermitAll`，该接口就被视为免登录
- 方法级优先检查：`handlerMethod.hasMethodAnnotation(PermitAll.class)`
- 类级检查：`handlerMethod.getBeanType().isAnnotationPresent(PermitAll.class)`

#### 步骤 3：收集 URL 路径

```java
Set<String> urls = new HashSet<>();
if (entry.getKey().getPatternsCondition() != null) {
    urls.addAll(entry.getKey().getPatternsCondition().getPatterns());
}
if (entry.getKey().getPathPatternsCondition() != null) {
    urls.addAll(convertList(entry.getKey().getPathPatternsCondition().getPatterns(), PathPattern::getPatternString));
}
```

- 支持两种 URL 模式：
  - `PatternsCondition`：传统的 Ant 路径模式（Spring MVC 默认）
  - `PathPatternsCondition`：Spring 5.3+ 引入的新 URL 匹配器（性能更优）

#### 步骤 4：方法映射

根据 `RequestMappingInfo` 中声明的 HTTP 方法，将 URL 分类存储到 `Multimap<HttpMethod, String>` 中。

### 示例

假设有以下控制器：

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @PostMapping("/login")
    @PermitAll
    public void login() {}
    
    @GetMapping("/profile")
    public void profile() {}
}

@PermitAll
@RestController
@RequestMapping("/api/public")
public class PublicController {
    
    @GetMapping("/info")
    public void info() {}
    
    @PostMapping("/submit")
    public void submit() {}
}
```

收集结果：

| URL 路径 | HTTP 方法 | 来源 |
|---------|-----------|------|
| /api/user/login | POST | 方法级 @PermitAll |
| /api/public/info | GET | 类级 @PermitAll |
| /api/public/submit | POST | 类级 @PermitAll |

---

## 2. @RequestMapping 未声明 method 时为什么会映射到 GET、POST、PUT、DELETE、HEAD、PATCH

### 代码位置

`YudaoWebSecurityConfigurerAdapter.java:183-192`

```java
// 特殊：使用 @RequestMapping 注解，并且未写 method 属性，此时认为都需要免登录
Set<RequestMethod> methods = entry.getKey().getMethodsCondition().getMethods();
if (CollUtil.isEmpty(methods)) {
    result.putAll(HttpMethod.GET, urls);
    result.putAll(HttpMethod.POST, urls);
    result.putAll(HttpMethod.PUT, urls);
    result.putAll(HttpMethod.DELETE, urls);
    result.putAll(HttpMethod.HEAD, urls);
    result.putAll(HttpMethod.PATCH, urls);
    continue;
}
```

### 技术原因

#### Spring MVC 的行为

当使用 `@RequestMapping` 但不显式指定 `method` 属性时：

```java
@RequestMapping("/api/all-methods")
public void allMethods() {}
```

Spring MVC 会将该方法映射到**所有 HTTP 方法**。这是因为：

1. `RequestMappingInfo.getMethodsCondition().getMethods()` 返回空集合
2. 空集合表示"未限制"，即接受所有 HTTP 方法

#### yudao-cloud 的处理策略

为了确保安全配置与 Spring MVC 的实际路由行为一致，yudao-cloud 选择**显式列出所有常见 HTTP 方法**：

- **GET**：用于资源获取
- **POST**：用于资源创建
- **PUT**：用于资源更新
- **DELETE**：用于资源删除
- **HEAD**：类似于 GET，但只返回响应头
- **PATCH**：用于资源部分更新

**注意**：`OPTIONS` 和 `TRACE` 方法未被包含，这是有意为之，因为它们通常用于 CORS 预检和调试，不涉及业务逻辑。

### 实际影响

假设有：

```java
@PermitAll
@RequestMapping("/api/callback")
public void callback() {}
```

该接口将在以下所有 HTTP 方法上免登录：
- `GET /api/callback` ✓
- `POST /api/callback` ✓
- `PUT /api/callback` ✓
- `DELETE /api/callback` ✓
- `HEAD /api/callback` ✓
- `PATCH /api/callback` ✓

---

## 3. 静态资源、注解免登录、配置项 permit-all-urls、自定义 AuthorizeRequestsCustomizer、兜底 authenticated 的先后关系

### 配置顺序（代码位置：YudaoWebSecurityConfigurerAdapter.java:128-148）

```java
// 设置每个请求的权限
httpSecurity
    // ①：全局共享规则
    .authorizeHttpRequests(c -> c
        // 1.1 静态资源，可匿名访问
        .requestMatchers(HttpMethod.GET, "/*.html", "/*.css", "/*.js").permitAll()
        // 1.2 设置 @PermitAll 无需认证
        .requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()
        .requestMatchers(HttpMethod.POST, permitAllUrls.get(HttpMethod.POST).toArray(new String[0])).permitAll()
        // ... 其他方法
        // 1.3 基于 yudao.security.permit-all-urls 无需认证
        .requestMatchers(securityProperties.getPermitAllUrls().toArray(new String[0])).permitAll()
    )
    // ②：每个项目的自定义规则
    .authorizeHttpRequests(c -> authorizeRequestsCustomizers.forEach(customizer -> customizer.customize(c)))
    // ③：兜底规则，必须认证
    .authorizeHttpRequests(c -> c
            .dispatcherTypeMatchers(DispatcherType.ASYNC).permitAll() // WebFlux 异步请求
            .anyRequest().authenticated());
```

### Spring Security 的匹配机制

Spring Security 使用**首次匹配原则（First Match Wins）**：

1. 请求按顺序匹配 `requestMatchers` 定义的规则
2. **第一个匹配的规则生效**，后续规则不再检查
3. 如果所有显式规则都不匹配，则由 `anyRequest()` 的兜底规则处理

### 优先级从高到低

| 优先级 | 规则类型 | 代码位置 | 说明 |
|-------|---------|---------|------|
| 1 | 静态资源 | 132 行 | GET 方法的 HTML/CSS/JS 文件 |
| 2 | @PermitAll 注解（方法级/类级） | 134-139 行 | 按 HTTP 方法精确匹配 |
| 3 | 配置项 `yudao.security.permit-all-urls` | 141 行 | `application.yml` 中配置的免登录 URL |
| 4 | 自定义 `AuthorizeRequestsCustomizer` | 144 行 | 各模块（System、Pay、Member 等）自定义的规则 |
| 5 | 兜底规则 `anyRequest().authenticated()` | 148 行 | 所有未匹配的请求必须认证 |

### 自定义 AuthorizeRequestsCustomizer 的顺序

`AuthorizeRequestsCustomizer` 接口继承了 `Ordered`，默认 order 为 0：

```java
public abstract class AuthorizeRequestsCustomizer
        implements Customizer<AuthorizeHttpRequestsConfigurer<HttpSecurity>.AuthorizationManagerRequestMatcherRegistry>, Ordered {
    
    @Override
    public int getOrder() {
        return 0;
    }
}
```

如果有多个自定义实现，它们的执行顺序由 `@Order` 或 `getOrder()` 的返回值决定，**order 值越小优先级越高**。

以 System 模块为例（`SecurityConfiguration.java:16-38`）：

```java
@Bean("systemAuthorizeRequestsCustomizer")
public AuthorizeRequestsCustomizer authorizeRequestsCustomizer() {
    return new AuthorizeRequestsCustomizer() {
        @Override
        public void customize(...) {
            // Swagger 接口文档
            registry.requestMatchers("/v3/api-docs/**").permitAll()
                    .requestMatchers("/webjars/**").permitAll()
                    .requestMatchers("/swagger-ui").permitAll()
                    .requestMatchers("/swagger-ui/**").permitAll();
            // Druid 监控
            registry.requestMatchers("/druid/**").permitAll();
            // Spring Boot Actuator
            registry.requestMatchers("/actuator").permitAll()
                    .requestMatchers("/actuator/**").permitAll();
            // RPC 服务
            registry.requestMatchers(ApiConstants.PREFIX + "/**").permitAll();
        }
    };
}
```

### 注意事项

1. **多次调用 `authorizeHttpRequests` 的合并**：在 Spring Security 6+ 中，多次调用 `authorizeHttpRequests()` 会合并规则，而不是覆盖
2. **同一优先级内的顺序**：在同一个 `authorizeHttpRequests` lambda 中，规则按声明顺序匹配
3. **ASYN 类型请求例外**：`dispatcherTypeMatchers(DispatcherType.ASYNC).permitAll()` 放在兜底规则之前，确保 SSE 等异步场景正常工作

---

## 4. TokenAuthenticationFilter 被 addFilterBefore 到 UsernamePasswordAuthenticationFilter 前，对 @PermitAll 接口是否仍会尝试解析 token

### 关键代码

**过滤器注册位置**（YudaoWebSecurityConfigurerAdapter.java:151）：
```java
httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
```

**TokenAuthenticationFilter 实现**（TokenAuthenticationFilter.java:46-81）：
```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
    // 情况一，基于 header[login-user] 获得用户
    LoginUser loginUser = buildLoginUserByHeader(request);

    // 情况二，基于 Token 获得用户
    if (loginUser == null) {
        String token = SecurityFrameworkUtils.obtainAuthorization(request,
                securityProperties.getTokenHeader(), securityProperties.getTokenParameter());
        if (StrUtil.isNotEmpty(token)) {
            Integer userType = WebFrameworkUtils.getLoginUserType(request);
            try {
                loginUser = buildLoginUserByToken(token, userType);
                if (loginUser == null) {
                    loginUser = mockLoginUser(request, token, userType);
                }
            } catch (Throwable ex) {
                CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
                ServletUtils.writeJSON(response, result);
                return;
            }
        }
    }

    // 设置当前用户
    if (loginUser != null) {
        SecurityFrameworkUtils.setLoginUser(loginUser, request);
    }
    // 继续过滤链
    chain.doFilter(request, response);
}
```

### 结论

**是的，对 @PermitAll 接口仍会尝试解析 token。**

### 原因分析

#### 1. 过滤器链的执行时机

Spring Security 的过滤器链在**授权决策之前**执行：

```
请求 → Filter Chain (包含 TokenAuthenticationFilter) 
      → AuthorizationFilter (检查 permitAll/authenticated)
      → Controller
```

`TokenAuthenticationFilter` 被添加到 `UsernamePasswordAuthenticationFilter` 之前，这意味着：
- 它在**所有请求**进入时都会执行
- **不区分**该请求是否配置了 `permitAll`

#### 2. TokenAuthenticationFilter 的无条件执行

`TokenAuthenticationFilter` 继承自 `OncePerRequestFilter`，它：
- 没有 `shouldNotFilter()` 的排除逻辑
- 不在 `doFilterInternal` 中检查 SecurityContext 或请求路径
- 只要请求中包含 token/header，就会尝试解析

#### 3. 两种解析方式

**方式一：Header 透传（Gateway/内部服务）**
```java
LoginUser buildLoginUserByHeader(HttpServletRequest request) {
    String loginUserStr = request.getHeader(SecurityFrameworkUtils.LOGIN_USER_HEADER);
    if (StrUtil.isEmpty(loginUserStr)) {
        return null;
    }
    // 解析 JSON → LoginUser
    // 校验 userType
    return loginUser;
}
```

**方式二：Token 解析（直连 Nginx）**
```java
LoginUser buildLoginUserByToken(String token, Integer userType) {
    try {
        OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
        if (accessToken == null) {
            return null;  // Token 无效，静默返回
        }
        // 校验 userType
        return new LoginUser(...);
    } catch (ServiceException serviceException) {
        // Token 校验失败，考虑到一些接口是无需登录的，所以直接返回 null
        return null;
    }
}
```

### 行为差异

| 场景 | 行为 | 原因 |
|-----|------|------|
| @PermitAll + 无 Token | 正常通过，`SecurityContext` 为 null | 过滤器不报错，直接放行 |
| @PermitAll + 有效 Token | 正常通过，`SecurityContext` 包含用户 | Token 解析成功，设置用户信息 |
| @PermitAll + 无效 Token | 正常通过，`SecurityContext` 为 null | `buildLoginUserByToken` 捕获 `ServiceException`，返回 null |
| @PermitAll + Header 透传（userType 不匹配） | **抛出 AccessDeniedException** | `buildLoginUserByHeader` 不捕获异常，直接抛出 |

### 设计意图

这种"即使 permitAll 也解析 token"的设计有以下好处：

1. **用户信息可访问**：在免登录接口中，仍然可以通过 `SecurityFrameworkUtils.getLoginUser()` 获取用户信息（如果用户已登录）
2. **支持可选登录**：例如商品详情接口，未登录时返回基础信息，已登录时返回收藏状态等个性化信息
3. **mock 模式支持**：`mockLoginUser` 方法允许在开发环境中模拟登录

---

## 5. AuthenticationEntryPoint 与 AccessDeniedHandler 的触发差异

### 类位置与实现

**AuthenticationEntryPointImpl**（AuthenticationEntryPointImpl.java:26-33）：
```java
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) {
        log.debug("[commence][访问 URL({}) 时，没有登录]", request.getRequestURI(), e);
        // 返回 401
        ServletUtils.writeJSON(response, CommonResult.error(UNAUTHORIZED));
    }
}
```

**AccessDeniedHandlerImpl**（AccessDeniedHandlerImpl.java:31-41）：
```java
public class AccessDeniedHandlerImpl implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException e)
            throws IOException, ServletException {
        log.warn("[commence][访问 URL({}) 时，用户({}) 权限不够]", request.getRequestURI(),
                SecurityFrameworkUtils.getLoginUserId(), e);
        // 返回 403
        ServletUtils.writeJSON(response, CommonResult.error(FORBIDDEN));
    }
}
```

### 触发时机

Spring Security 通过 `ExceptionTranslationFilter` 来统一处理异常：

```
ExceptionTranslationFilter 逻辑：

try {
    filterChain.doFilter(request, response);
} catch (AccessDeniedException ex) {
    // 已认证但权限不足 → AccessDeniedHandler
    handleAccessDeniedException(request, response, chain, ex);
} catch (AuthenticationException ex) {
    // 未认证 → AuthenticationEntryPoint
    sendStartAuthentication(request, response, chain, ex);
}
```

### 核心差异

| 维度 | AuthenticationEntryPointImpl | AccessDeniedHandlerImpl |
|-----|-----------------------------|------------------------|
| **触发条件** | 未认证（Anonymous） | 已认证但权限不足 |
| **异常类型** | `AuthenticationException` | `AccessDeniedException` |
| **HTTP 状态码** | 401 UNAUTHORIZED | 403 FORBIDDEN |
| **日志级别** | DEBUG | WARN |
| **日志内容** | "没有登录" | "权限不够" + 用户ID |
| **处理方法名** | `commence()` | `handle()` |

### 具体场景示例

#### 场景 1：访问需要认证的接口，无 token

- 请求：`GET /api/user/profile`（需认证）
- Header：无 Authorization
- 结果：
  - `AuthorizationFilter` 检测到 `SecurityContext` 为空
  - 抛出 `InsufficientAuthenticationException`（继承自 `AuthenticationException`）
  - `ExceptionTranslationFilter` 捕获后调用 `AuthenticationEntryPointImpl.commence()`
  - 返回 **401**

#### 场景 2：已登录但权限不足

- 请求：`POST /api/admin/user/delete`（需 ADMIN 角色）
- Header：Authorization: Bearer <普通用户token>
- 结果：
  - `TokenAuthenticationFilter` 解析 token 成功，设置普通用户
  - `MethodSecurityInterceptor`（方法级权限）检测到角色不匹配
  - 抛出 `AccessDeniedException`
  - `ExceptionTranslationFilter` 捕获后调用 `AccessDeniedHandlerImpl.handle()`
  - 返回 **403**

#### 场景 3：Token 过期

- 请求：`GET /api/user/profile`
- Header：Authorization: Bearer <过期token>
- 结果（取决于过滤器行为）：
  - `TokenAuthenticationFilter.buildLoginUserByToken()` 捕获 `ServiceException`，返回 null
  - `SecurityContext` 为空
  - 最终触发 `AuthenticationEntryPointImpl`，返回 **401**

### ExceptionTranslationFilter 的详细逻辑

```
请求 → 前置过滤器
     → AuthorizationFilter（权限检查）
     → 抛出异常
     → ExceptionTranslationFilter 捕获
        ├── AuthenticationException → AuthenticationEntryPoint.commence() → 401
        └── AccessDeniedException
              ├── SecurityContext 为空 → AuthenticationEntryPoint.commence() → 401
              └── SecurityContext 有用户 → AccessDeniedHandler.handle() → 403
```

### 关键发现

`AccessDeniedException` 有一个特殊的降级逻辑：

```java
// ExceptionTranslationFilter 源码（简化）
private void handleAccessDeniedException(HttpServletRequest request, HttpServletResponse response,
        FilterChain chain, AccessDeniedException exception) throws ServletException, IOException {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    if (authentication == null || isAnonymous(authentication)) {
        // 虽然抛出的是 AccessDeniedException，但用户未认证
        // 降级为 401
        sendStartAuthentication(request, response, chain,
                new InsufficientAuthenticationException("Full authentication is required to access this resource"));
    }
    else {
        // 已认证但权限不足
        accessDeniedHandler.handle(request, response, exception);
    }
}
```

这意味着：
- **401** = "你是谁？请先登录"
- **403** = "我知道你是谁，但你没有权限"

---

## 6. 5 个边界样例推导

### 前提假设

假设项目中有以下配置：

```yaml
# application.yml
yudao:
  security:
    permit-all-urls:
      - /api/config/**
      - /api/health
```

控制器：

```java
@RestController
@RequestMapping("/api")
public class TestController {
    
    @PermitAll
    @GetMapping("/public")
    public String publicEndpoint() { return "public"; }
    
    @PermitAll
    @RequestMapping("/all-methods")
    public String allMethods() { return "all methods"; }
    
    @GetMapping("/private")
    public String privateEndpoint() { return "private"; }
}

@Configuration
public class CustomSecurityConfig {
    @Bean
    public AuthorizeRequestsCustomizer customizer() {
        return registry -> registry.requestMatchers("/api/custom/**").permitAll();
    }
}
```

### 样例 1：@PermitAll + 无 Token + GET

**请求：**
```
GET /api/public
Authorization: (无)
```

**行为推导：**

1. `TokenAuthenticationFilter` 执行
   - 无 token，无 login-user header
   - `SecurityContext` 为空
   - `chain.doFilter()` 继续

2. `AuthorizationFilter` 匹配规则
   - 匹配到步骤 2（@PermitAll 注解收集的 URL）
   - 规则：`.requestMatchers(HttpMethod.GET, "/api/public").permitAll()`
   - 放行

**最终结果：** 200 OK，返回 "public"

---

### 样例 2：@PermitAll + 无效 Token + GET

**请求：**
```
GET /api/public
Authorization: Bearer invalid-token-12345
```

**行为推导：**

1. `TokenAuthenticationFilter` 执行
   - 提取到 token
   - 调用 `oauth2TokenApi.checkAccessToken()`
   - 远程服务返回无效，抛出 `ServiceException`
   - `buildLoginUserByToken()` 捕获异常，**返回 null**（静默失败）
   - `SecurityContext` 为空
   - `chain.doFilter()` 继续

2. `AuthorizationFilter` 匹配规则
   - 匹配到 `permitAll` 规则
   - 放行

**最终结果：** 200 OK，返回 "public"
**关键发现：** 无效 token 在 permitAll 接口中不会导致 401，因为 filter 内部静默处理了

---

### 样例 3：@RequestMapping 无 method + DELETE

**请求：**
```
DELETE /api/all-methods
Authorization: (无)
```

**行为推导：**

1. `getPermitAllUrlsFromAnnotations()` 收集阶段
   - 检测到 `@RequestMapping` 无 method
   - `methodsCondition.getMethods()` 返回空集合
   - 该 URL 被添加到 **所有 6 种 HTTP 方法** 的 permitAll 列表中

2. `AuthorizationFilter` 匹配规则
   - 匹配到步骤 2 中的：
     ```java
     .requestMatchers(HttpMethod.DELETE, "/api/all-methods").permitAll()
     ```
   - 放行

**最终结果：** 200 OK，返回 "all methods"

---

### 样例 4：配置项 permit-all-urls + 自定义规则 + 路径重叠

**请求：**
```
GET /api/config/details
Authorization: (无)
```

假设配置项：
```yaml
yudao.security.permit-all-urls:
  - /api/config/**
```

自定义规则：
```java
// AuthorizeRequestsCustomizer
registry.requestMatchers("/api/config/details").authenticated();  // 冲突！
```

**行为推导：**

1. 规则顺序回顾（从高到低）：
   - 静态资源（优先级 1）
   - @PermitAll 注解（优先级 2）
   - **配置项 yudao.security.permit-all-urls**（优先级 3）
   - **自定义 AuthorizeRequestsCustomizer**（优先级 4）
   - 兜底 authenticated（优先级 5）

2. `AuthorizationFilter` 按顺序匹配
   - 首先检查优先级 3：`/api/config/**` 匹配成功
   - 规则是 `permitAll`
   - **First Match Wins**，后续优先级 4 的规则不再检查

**最终结果：** 200 OK，配置项的 permitAll 生效

---

### 样例 5：Header 透传 + userType 不匹配 + @PermitAll

**请求：**
```
GET /api/public
login-user: {"id":1,"userType":2}  // userType=2 表示普通用户
tenant-id: 1
```

假设 `/api/public` 接口路径暗示需要 admin 用户类型（userType=1）

**行为推导：**

1. `TokenAuthenticationFilter` 执行
   - 检测到 `login-user` header
   - 调用 `buildLoginUserByHeader()`
   - 解析 JSON 成功，`loginUser.userType = 2`
   - **检查 userType 匹配**：
     ```java
     Integer userType = WebFrameworkUtils.getLoginUserType(request);
     if (userType != null && ObjectUtil.notEqual(loginUser.getUserType(), userType)) {
         throw new AccessDeniedException("错误的用户类型");
     }
     ```
   - 路径 `/api/public` 没有 userType 前缀（如 `/admin-api` 或 `/app-api`），所以 `userType = null`
   - **不校验**，继续

2. 设置用户到 SecurityContext
3. `AuthorizationFilter` 匹配 permitAll 规则，放行

**最终结果：** 200 OK

**但如果请求路径是 `/admin-api/public`：**

1. `WebFrameworkUtils.getLoginUserType(request)` 返回 1（admin）
2. `loginUser.userType = 2`（普通用户）
3. 类型不匹配，**抛出 `AccessDeniedException`**
4. 注意：`buildLoginUserByHeader()` **不捕获**该异常
5. `doFilterInternal()` 中的 catch 块捕获 `Throwable`：
   ```java
   try {
       loginUser = buildLoginUserByToken(token, userType);
       // ...
   } catch (Throwable ex) {
       CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
       ServletUtils.writeJSON(response, result);
       return;  // 直接返回，不继续 filter chain
   }
   ```
   - 等等，catch 块只包围 `buildLoginUserByToken` 和 `mockLoginUser`
   - `buildLoginUserByHeader()` 的异常会**向上抛出**

**修正推导（Header 透传 + userType 不匹配）：**

```
请求：GET /admin-api/public
Header: login-user: {"id":1,"userType":2}  // userType=2 是普通用户
```

1. `buildLoginUserByHeader()` 调用
   - 解析成功，`loginUser.userType = 2`
   - `WebFrameworkUtils.getLoginUserType(request)` → `/admin-api` → 返回 1
   - `2 != 1` → **抛出 `AccessDeniedException`**
   - 这个方法**没有 try-catch**

2. `doFilterInternal()` 中：
   ```java
   LoginUser loginUser = buildLoginUserByHeader(request);  // ← 这里抛出异常！
   // ...
   if (loginUser == null) {
       // 走 token 解析
   }
   ```
   - 第一行就抛出异常，没有被 catch 包围
   - 异常直接传播到 `ExceptionTranslationFilter`

3. `ExceptionTranslationFilter` 捕获
   - 异常类型：`AccessDeniedException`
   - 检查 `SecurityContext`：为空（还没来得及设置）
   - **降级逻辑**：虽然是 AccessDeniedException，但用户未认证
   - 调用 `AuthenticationEntryPointImpl.commence()`

**最终结果：** 返回 401 UNAUTHORIZED

---

## 总结

### 关键发现总览

| 问题 | 答案 |
|-----|------|
| getPermitAllUrlsFromAnnotations 如何收集 | 通过 RequestMappingHandlerMapping 获取所有 HandlerMethod，检查方法级或类级 @PermitAll（OR 逻辑） |
| @RequestMapping 无 method 时 | methodsCondition 返回空集合，yudao-cloud 显式添加到 GET/POST/PUT/DELETE/HEAD/PATCH |
| 规则优先级 | 静态资源 > @PermitAll 注解 > 配置项 permit-all-urls > 自定义 AuthorizeRequestsCustomizer > 兜底 authenticated |
| @PermitAll 接口是否解析 token | **是**，TokenAuthenticationFilter 在授权决策前执行，不区分是否 permitAll |
| AuthenticationEntryPoint vs AccessDeniedHandler | 401（未认证）vs 403（已认证但权限不足），由 ExceptionTranslationFilter 根据 SecurityContext 决定 |
| 无效 token + @PermitAll | 正常通过（filter 内部静默失败） |
| Header 透传 + userType 不匹配 | 抛出 AccessDeniedException，可能降级为 401 |

### 安全建议

1. **谨慎使用类级 @PermitAll**：会导致该类所有接口免登录，包括后续新增的方法
2. **显式声明 HTTP 方法**：避免 `@RequestMapping` 无 method 的情况，缩小攻击面
3. **注意自定义规则顺序**：配置项优先级高于自定义 AuthorizeRequestsCustomizer
4. **userType 校验严格**：Header 透传的 userType 不匹配会直接抛出异常，与 token 解析的静默失败行为不一致
5. **无效 token 静默失败**：在 permitAll 接口中，无效 token 不会报错，这是设计选择但需注意
