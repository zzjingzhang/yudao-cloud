# yudao-cloud @RateLimiter 运行机制与并发语义分析

## 一、核心组件协作关系

`@RateLimiter` 限流功能由以下核心组件协作完成：

| 组件 | 位置 | 职责 |
|------|------|------|
| `@RateLimiter` | `annotation/RateLimiter.java` | 声明限流配置（时间窗口、次数、Key 解析器等） |
| `RateLimiterAspect` | `aop/RateLimiterAspect.java` | AOP 切面，拦截带注解的方法，协调 Key 解析与限流判断 |
| `RateLimiterRedisDAO` | `redis/RateLimiterRedisDAO.java` | 基于 Redisson `RRateLimiter` 实现分布式限流逻辑 |
| `RateLimiterKeyResolver` | `keyresolver/RateLimiterKeyResolver.java` | Key 解析器接口 |
| `DefaultRateLimiterKeyResolver` | `keyresolver/impl/` | 全局级别，基于方法签名+参数 |
| `UserRateLimiterKeyResolver` | `keyresolver/impl/` | 用户级别，基于方法签名+参数+userId+userType |
| `ClientIpRateLimiterKeyResolver` | `keyresolver/impl/` | IP 级别，基于方法签名+参数+IP |
| `ServerNodeRateLimiterKeyResolver` | `keyresolver/impl/` | 服务节点级别，基于方法签名+参数+服务器地址+PID |
| `ExpressionRateLimiterKeyResolver` | `keyresolver/impl/` | 表达式级别，基于 Spring EL 表达式解析 |

**执行流程**：

```
请求进入
    ↓
RateLimiterAspect.beforePointCut() 拦截
    ↓
根据注解的 keyResolver() 从 Map 中获取对应的 KeyResolver 实例
    ↓
KeyResolver.resolver() 计算限流 Key（方法签名 + 维度标识）
    ↓
RateLimiterRedisDAO.tryAcquire(key, count, time, timeUnit)
    ↓
getRRateLimiter() 获取/创建 RRateLimiter 并配置速率
    ↓
tryAcquire() 尝试获取令牌
    ↓
成功 → 执行业务逻辑
失败 → 抛出 ServiceException(TOO_MANY_REQUESTS)
```

---

## 二、问题解答

### 1. keyResolver 的选择机制与 @Idempotent 有何相似和不同

#### 相似点

**（1）基于注解 Class 的 Map 查找机制**

两者的 Aspect 都采用相同的设计模式：

```java
// RateLimiterAspect.java:35-37
public RateLimiterAspect(List<RateLimiterKeyResolver> keyResolvers, ...) {
    this.keyResolvers = CollectionUtils.convertMap(keyResolvers, RateLimiterKeyResolver::getClass);
}

// IdempotentAspect.java:34-36
public IdempotentAspect(List<IdempotentKeyResolver> keyResolvers, ...) {
    this.keyResolvers = CollectionUtils.convertMap(keyResolvers, IdempotentKeyResolver::getClass);
}
```

- Spring 注入所有 KeyResolver 实现类（List）
- 以 `Class` 为 key 转为 Map 存储
- 运行时通过注解的 `keyResolver()` 属性获取对应实例

**（2）注解属性驱动**

```java
// RateLimiter.java:56
Class<? extends RateLimiterKeyResolver> keyResolver() default DefaultRateLimiterKeyResolver.class;

// Idempotent.java:46
Class<? extends IdempotentKeyResolver> keyResolver() default DefaultIdempotentKeyResolver.class;
```

- 都通过注解的 `keyResolver` 属性指定解析器
- 都提供 `keyArg()` 属性用于 Expression 解析器
- 都有 Default、User、Expression 三类解析器

**（3）解析器接口设计一致**

```java
// RateLimiterKeyResolver.java:20
String resolver(JoinPoint joinPoint, RateLimiter rateLimiter);

// IdempotentKeyResolver（同理）
String resolver(JoinPoint joinPoint, Idempotent idempotent);
```

#### 不同点

**（1）提供的解析器种类不同**

| 解析器类型 | @RateLimiter | @Idempotent |
|-----------|-------------|-------------|
| Default（全局） | ✅ | ✅ |
| User（用户） | ✅ | ✅ |
| Expression（表达式） | ✅ | ✅ |
| ClientIp（客户端 IP） | ✅ | ❌ |
| ServerNode（服务节点） | ✅ | ❌ |

限流场景更强调"谁在调用"（IP）和"在哪台机器"（节点），而幂等更关注"同一业务请求"。

**（2）切面时机不同**

```java
// RateLimiterAspect.java:40-41
@Before("@annotation(rateLimiter)")
public void beforePointCut(JoinPoint joinPoint, RateLimiter rateLimiter)

// IdempotentAspect.java:39-40
@Around(value = "@annotation(idempotent)")
public Object aroundPointCut(ProceedingJoinPoint joinPoint, Idempotent idempotent)
```

- `@RateLimiter`：`@Before`，仅在方法执行前拦截，失败直接抛异常
- `@Idempotent`：`@Around`，环绕整个方法，支持异常时删除 Key 的逻辑

**（3）Default 解析器的隐含语义不同**

虽然两者的 Default 解析器实现相同（方法签名 + 参数 → MD5），但语义差异巨大：

- **限流 Default**：**全局级别的方法限流**。所有用户调用同一个方法（带相同参数）共享一个令牌桶。
- **幂等 Default**：**防止重复提交**。相同参数的请求在 timeout 内只能执行一次。

---

### 2. RateLimiterRedisDAO 如何获得 RRateLimiter 并设置 rate

核心逻辑在 `RateLimiterRedisDAO.java:40-64` 的 `getRRateLimiter()` 方法：

#### （1）Key 格式化

```java
// RateLimiterRedisDAO.java:36-38
private static String formatKey(String key) {
    return String.format(RATE_LIMITER, key);  // "rate_limiter:%s"
}
```

所有限流 Key 会加上 `rate_limiter:` 前缀存储在 Redis 中。

#### （2）获取 RRateLimiter 实例

```java
// RateLimiterRedisDAO.java:41-42
String redisKey = formatKey(key);
RRateLimiter rateLimiter = redissonClient.getRateLimiter(redisKey);
```

`redissonClient.getRateLimiter()` 是**懒加载**的，不会立即访问 Redis，只是创建一个本地代理对象。

#### （3）配置速率的三分支逻辑

```java
// RateLimiterRedisDAO.java:43-64
long rateInterval = timeUnit.toSeconds(time);  // 统一转为秒
Duration duration = Duration.ofSeconds(rateInterval);

// 分支 1：Redis 中不存在配置 → 初次创建
RateLimiterConfig config = rateLimiter.getConfig();
if (config == null) {
    rateLimiter.trySetRate(RateType.OVERALL, count, duration);
    rateLimiter.expire(duration);
    return rateLimiter;
}

// 分支 2：配置相同 → 直接复用
if (config.getRateType() == RateType.OVERALL
        && Objects.equals(config.getRate(), count)
        && Objects.equals(config.getRateInterval(), TimeUnit.SECONDS.toMillis(rateInterval))) {
    return rateLimiter;
}

// 分支 3：配置不同 → 更新配置
rateLimiter.setRate(RateType.OVERALL, count, duration);
rateLimiter.expire(duration);
return rateLimiter;
```

**关键点**：

- 始终使用 `RateType.OVERALL`（全局限流模式），而非 `PER_CLIENT`
- `getConfig()` 会真正访问 Redis 获取当前配置
- `trySetRate()` 与 `setRate()` 的区别：
  - `trySetRate()`：仅当不存在时设置，不会覆盖已有配置
  - `setRate()`：强制覆盖，无论是否存在

---

### 3. 当注解 count、time、timeUnit 改变时，已有 Redis 限流器配置会不会更新，依据是什么

**答案：会更新。**

#### 依据

```java
// RateLimiterRedisDAO.java:54-63
if (config.getRateType() == RateType.OVERALL
        && Objects.equals(config.getRate(), count)
        && Objects.equals(config.getRateInterval(), TimeUnit.SECONDS.toMillis(rateInterval))) {
    return rateLimiter;  // 配置相同，不更新
}
// 配置不同，强制更新
rateLimiter.setRate(RateType.OVERALL, count, duration);
rateLimiter.expire(duration);
```

#### 具体比较逻辑

| 注解属性 | Redis 中存储的配置 | 比较方式 |
|---------|-------------------|---------|
| `count` | `config.getRate()` | `Objects.equals()` |
| `time + timeUnit` | `config.getRateInterval()`（毫秒） | 先 `timeUnit.toSeconds(time)` 再转毫秒比较 |

#### 容易误读的并发语义

**问题场景**：假设某接口限流配置从 `count=10, time=1, timeUnit=SECONDS` 改为 `count=100, time=10, timeUnit=SECONDS`。

**并发情况下会发生什么？**

1. 第一个请求到达时，`getConfig()` 读取到旧配置 `rate=10, interval=1000ms`
2. 与注解新配置 `count=100, interval=10000ms` 比较 → 不相等
3. 执行 `setRate()` 更新配置

**潜在问题**：

- **更新时机不确定**：如果没有请求进来，旧配置会一直留在 Redis
- **并发更新**：多个请求同时发现配置不一致，会**多次执行 `setRate()`**
- **expire 覆盖**：每次更新都会重新设置过期时间，可能导致 Key 长期不失效

**与 Redisson 底层行为的差异**：

Redisson 的 `RRateLimiter` 本身是基于 Redis 原子命令实现的令牌桶算法。`setRate()` 会重置整个速率配置，但**不会清空当前已消耗的令牌**。这意味着：

- 配置从紧变松：已消耗的令牌可能导致短期仍被限流
- 配置从松变紧：短期内可能仍能获取令牌直到下一个时间窗口

---

### 4. 不同 keyResolver 的限流粒度分别是什么

#### （1）DefaultRateLimiterKeyResolver

```java
// DefaultRateLimiterKeyResolver.java:19-23
public String resolver(JoinPoint joinPoint, RateLimiter rateLimiter) {
    String methodName = joinPoint.getSignature().toString();
    String argsStr = StrUtils.joinMethodArgs(joinPoint);
    return SecureUtil.md5(methodName + argsStr);
}
```

**限流粒度**：**方法签名 + 参数值**

| 场景 | 是否共享令牌桶 |
|------|--------------|
| 不同用户调用同一方法（同参数） | ✅ 共享 |
| 同一用户调用同一方法（不同参数） | ❌ 不共享 |
| 不同方法 | ❌ 不共享 |

**容易误读**：名为"Default"（默认），但实际上是**参数敏感**的。如果方法参数包含请求 ID、时间戳等每次变化的值，会导致每个请求独享一个令牌桶，限流完全失效。

---

#### （2）UserRateLimiterKeyResolver

```java
// UserRateLimiterKeyResolver.java:20-26
public String resolver(JoinPoint joinPoint, RateLimiter rateLimiter) {
    String methodName = joinPoint.getSignature().toString();
    String argsStr = StrUtils.joinMethodArgs(joinPoint);
    Long userId = WebFrameworkUtils.getLoginUserId();
    Integer userType = WebFrameworkUtils.getLoginUserType();
    return SecureUtil.md5(methodName + argsStr + userId + userType);
}
```

**限流粒度**：**方法签名 + 参数值 + userId + userType**

| 场景 | 是否共享令牌桶 |
|------|--------------|
| 同一用户（同 userType）调用同一方法（同参数） | ✅ 共享 |
| 不同用户调用同一方法（同参数） | ❌ 不共享 |
| 同一用户不同 userType（管理端 vs 客户端） | ❌ 不共享 |

---

#### （3）ClientIpRateLimiterKeyResolver

```java
// ClientIpRateLimiterKeyResolver.java:20-25
public String resolver(JoinPoint joinPoint, RateLimiter rateLimiter) {
    String methodName = joinPoint.getSignature().toString();
    String argsStr = StrUtils.joinMethodArgs(joinPoint);
    String clientIp = ServletUtils.getClientIP();
    return SecureUtil.md5(methodName + argsStr + clientIp);
}
```

**限流粒度**：**方法签名 + 参数值 + 客户端 IP**

| 场景 | 是否共享令牌桶 |
|------|--------------|
| 同一 IP 调用同一方法（同参数） | ✅ 共享 |
| 同一 NAT 出口下的多个用户 | ✅ 共享（误杀风险） |
| 切换网络（Wi-Fi → 4G） | ❌ 不共享（绕过风险） |

---

#### （4）ServerNodeRateLimiterKeyResolver

```java
// ServerNodeRateLimiterKeyResolver.java:20-25
public String resolver(JoinPoint joinPoint, RateLimiter rateLimiter) {
    String methodName = joinPoint.getSignature().toString();
    String argsStr = StrUtils.joinMethodArgs(joinPoint);
    String serverNode = String.format("%s@%d", SystemUtil.getHostInfo().getAddress(), SystemUtil.getCurrentPID());
    return SecureUtil.md5(methodName + argsStr + serverNode);
}
```

**限流粒度**：**方法签名 + 参数值 + 服务器 IP + PID**

| 场景 | 是否共享令牌桶 |
|------|--------------|
| 同一 JVM 进程处理同一方法（同参数） | ✅ 共享 |
| 多实例部署（不同机器） | ❌ 不共享 |
| 同一机器多实例（不同 PID） | ❌ 不共享 |

**容易误读**：虽然使用了 Redis（分布式），但 ServerNode 粒度会**绕过分布式限流**。每个实例独立计算限流，总限流数 = `count × 实例数`。

---

#### （5）ExpressionRateLimiterKeyResolver

```java
// ExpressionRateLimiterKeyResolver.java:29-45
public String resolver(JoinPoint joinPoint, RateLimiter rateLimiter) {
    Method method = getMethod(joinPoint);
    Object[] args = joinPoint.getArgs();
    String[] parameterNames = this.parameterNameDiscoverer.getParameterNames(method);
    StandardEvaluationContext evaluationContext = new StandardEvaluationContext();
    if (ArrayUtil.isNotEmpty(parameterNames)) {
        for (int i = 0; i < parameterNames.length; i++) {
            evaluationContext.setVariable(parameterNames[i], args[i]);
        }
    }
    Expression expression = expressionParser.parseExpression(rateLimiter.keyArg());
    return expression.getValue(evaluationContext, String.class);
}
```

**限流粒度**：**完全由 SpEL 表达式 `keyArg()` 决定**

示例：
- `@RateLimiter(keyResolver = ExpressionRateLimiterKeyResolver.class, keyArg = "#userId")` → 按 userId 限流
- `@RateLimiter(keyResolver = ExpressionRateLimiterKeyResolver.class, keyArg = "#order.productId")` → 按商品 ID 限流

**注意**：Expression 解析器**不包含**方法签名和参数。如果表达式仅返回一个简单值（如 `"my-key"`），所有请求共享同一个令牌桶。

---

### 5. 用户未登录时 UserRateLimiterKeyResolver 可能造成什么结果

#### 分析 WebFrameworkUtils 返回值

```java
// WebFrameworkUtils.java:91-96
public static Long getLoginUserId(HttpServletRequest request) {
    if (request == null) {
        return null;
    }
    return (Long) request.getAttribute(REQUEST_ATTRIBUTE_LOGIN_USER_ID);
}

// WebFrameworkUtils.java:105-122
public static Integer getLoginUserType(HttpServletRequest request) {
    if (request == null) {
        return null;
    }
    Integer userType = (Integer) request.getAttribute(REQUEST_ATTRIBUTE_LOGIN_USER_TYPE);
    if (userType != null) {
        return userType;
    }
    // 基于 URL 前缀推断
    if (request.getServletPath().startsWith(properties.getAdminApi().getPrefix())) {
        return UserTypeEnum.ADMIN.getValue();
    }
    if (request.getServletPath().startsWith(properties.getAppApi().getPrefix())) {
        return UserTypeEnum.MEMBER.getValue();
    }
    return null;
}
```

未登录时：
- `getLoginUserId()` → `null`
- `getLoginUserType()` → 可能是 `null`，也可能根据 URL 前缀推断为 ADMIN/MEMBER

#### 对 UserRateLimiterKeyResolver 的影响

```java
// UserRateLimiterKeyResolver.java:25
return SecureUtil.md5(methodName + argsStr + userId + userType);
```

Java 字符串拼接中，`null` 会被转为 `"null"` 字符串。

#### 后果

| 场景 | userId | userType | 拼接结果 | 限流行为 |
|------|--------|----------|---------|---------|
| 未登录 + `/admin-api/**` | `null` | `1`（ADMIN） | `...null1` | **所有未登录的管理端请求共享一个令牌桶** |
| 未登录 + `/app-api/**` | `null` | `2`（MEMBER） | `...null2` | **所有未登录的客户端请求共享一个令牌桶** |
| 未登录 + 其他路径 | `null` | `null` | `...nullnull` | **所有未登录请求共享一个令牌桶** |

#### 风险

1. **放大误杀**：如果限流配置为 `count=5`，那么所有未登录用户加起来只能调用 5 次/秒，单个合法用户可能被误伤
2. **绕过攻击**：攻击者切换 IP 没用（User 限流不看 IP），但可以通过登录不同账号绕过
3. **不一致行为**：登录用户按 userId 独立限流，未登录用户全部挤在一个桶里，用户体验不一致

---

### 6. 设计 4 个请求样例，推导最终 key、限流桶和是否被拒绝

假设环境：
- 某方法：`public String testMethod(String param1)`
- 注解：`@RateLimiter(count = 3, time = 1, timeUnit = TimeUnit.SECONDS)`
- 时间窗口：1 秒内最多 3 次
- 用户 A（userId=100, userType=2）已登录
- 用户 B（userId=200, userType=2）已登录
- 客户端 IP：`192.168.1.100`
- 服务器节点：`10.0.0.1@8080`

---

#### 样例 1：DefaultRateLimiterKeyResolver

**场景**：连续 4 次调用，参数相同

| 请求 | keyResolver | 方法签名 | 参数 | 计算过程 | 最终 Key（MD5 前） | Redis Key | 结果 |
|-----|-------------|---------|------|---------|------------------|-----------|------|
| 1 | Default | `testMethod(String)` | `"abc"` | `md5(方法签名 + "abc")` | `testMethod(String)abc` | `rate_limiter:xxx` | ✅ 通过（第 1 次） |
| 2 | Default | `testMethod(String)` | `"abc"` | 同请求 1 | 同请求 1 | 同请求 1 | ✅ 通过（第 2 次） |
| 3 | Default | `testMethod(String)` | `"abc"` | 同请求 1 | 同请求 1 | 同请求 1 | ✅ 通过（第 3 次） |
| 4 | Default | `testMethod(String)` | `"abc"` | 同请求 1 | 同请求 1 | 同请求 1 | ❌ 拒绝（第 4 次） |

**关键观察**：所有请求共享一个令牌桶。参数相同则 Key 相同。

---

#### 样例 2：UserRateLimiterKeyResolver

**场景**：用户 A 调用 2 次，用户 B 调用 2 次，未登录用户调用 2 次

| 请求 | 调用者 | 参数 | userId | userType | 最终 Key（MD5 前） | Redis Key | 结果 |
|-----|--------|------|--------|----------|------------------|-----------|------|
| 1 | 用户 A | `"x"` | 100 | 2 | `testMethod(String)x1002` | `rate_limiter:xxx1` | ✅ 通过 |
| 2 | 用户 A | `"x"` | 100 | 2 | 同请求 1 | 同请求 1 | ✅ 通过 |
| 3 | 用户 B | `"x"` | 200 | 2 | `testMethod(String)x2002` | `rate_limiter:xxx2` | ✅ 通过（不同桶） |
| 4 | 用户 B | `"x"` | 200 | 2 | 同请求 3 | 同请求 3 | ✅ 通过 |
| 5 | 未登录 | `"x"` | null | 2 | `testMethod(String)xnull2` | `rate_limiter:xxx3` | ✅ 通过（未登录桶） |
| 6 | 未登录 | `"x"` | null | 2 | 同请求 5 | 同请求 5 | ✅ 通过 |
| 7 | 未登录 | `"x"` | null | 2 | 同请求 5 | 同请求 5 | ❌ 拒绝（未登录桶满） |

**关键观察**：
- 用户 A 和 B 有各自独立的令牌桶
- 所有未登录用户共享同一个桶（`null` 转为 `"null"`）
- 第 7 次请求被拒绝，因为未登录桶已耗尽

---

#### 样例 3：ClientIpRateLimiterKeyResolver vs ServerNodeRateLimiterKeyResolver

**场景**：同一 IP 的两个用户，在双实例部署（Instance1、Instance2）下各调用 3 次

##### 3.1 ClientIp 粒度

| 请求 | 实例 | 调用者 | IP | 最终 Key | Redis Key | 结果 |
|-----|------|--------|----|---------|-----------|------|
| 1 | Instance1 | 用户 A | 192.168.1.100 | `testMethod(String)x192.168.1.100` | `rate_limiter:ip1` | ✅ 通过 |
| 2 | Instance2 | 用户 B | 192.168.1.100 | 同请求 1 | 同请求 1 | ✅ 通过 |
| 3 | Instance1 | 用户 A | 192.168.1.100 | 同请求 1 | 同请求 1 | ✅ 通过 |
| 4 | Instance2 | 用户 B | 192.168.1.100 | 同请求 1 | 同请求 1 | ❌ 拒绝 |

##### 3.2 ServerNode 粒度

| 请求 | 实例 | 服务器节点 | 最终 Key | Redis Key | 结果 |
|-----|------|-----------|---------|-----------|------|
| 1 | Instance1 | 10.0.0.1@8080 | `testMethod(String)x10.0.0.1@8080` | `rate_limiter:node1` | ✅ 通过 |
| 2 | Instance2 | 10.0.0.2@8081 | `testMethod(String)x10.0.0.2@8081` | `rate_limiter:node2` | ✅ 通过 |
| 3 | Instance1 | 10.0.0.1@8080 | 同请求 1 | 同请求 1 | ✅ 通过 |
| 4 | Instance2 | 10.0.0.2@8081 | 同请求 2 | 同请求 2 | ✅ 通过 |
| 5 | Instance1 | 10.0.0.1@8080 | 同请求 1 | 同请求 1 | ✅ 通过 |
| 6 | Instance2 | 10.0.0.2@8081 | 同请求 2 | 同请求 2 | ✅ 通过 |
| 7 | Instance1 | 10.0.0.1@8080 | 同请求 1 | 同请求 1 | ❌ 拒绝（Instance1 桶满） |
| 8 | Instance2 | 10.0.0.2@8081 | 同请求 2 | 同请求 2 | ❌ 拒绝（Instance2 桶满） |

**关键对比**：
- **ClientIp**：真正的分布式限流（Redis 全局一个桶），2 实例总共只能 3 次
- **ServerNode**：每个实例独立一个桶，2 实例总共可通过 6 次（`3 × 2`）

---

#### 样例 4：ExpressionRateLimiterKeyResolver

**场景**：注解配置为 `@RateLimiter(count = 3, keyResolver = ExpressionRateLimiterKeyResolver.class, keyArg = "#userId")`

方法签名：`public String createOrder(Long userId, String productId)`

| 请求 | userId | productId | SpEL 表达式 | 表达式值 | Redis Key | 结果 |
|-----|--------|-----------|------------|---------|-----------|------|
| 1 | 100 | `"P001"` | `#userId` | `"100"` | `rate_limiter:100` | ✅ 通过 |
| 2 | 100 | `"P002"` | `#userId` | `"100"` | `rate_limiter:100` | ✅ 通过 |
| 3 | 100 | `"P003"` | `#userId` | `"100"` | `rate_limiter:100` | ✅ 通过 |
| 4 | 100 | `"P004"` | `#userId` | `"100"` | `rate_limiter:100` | ❌ 拒绝 |
| 5 | 200 | `"P001"` | `#userId` | `"200"` | `rate_limiter:200` | ✅ 通过（不同用户） |

**关键观察**：
- Expression 解析器**不包含**方法签名和参数（除非你在表达式里显式写）
- 同一 userId 无论购买什么商品，都共享一个令牌桶
- 这实现了"每个用户每秒最多下 3 单"的业务语义

---

## 三、容易误读的并发语义总结

| 误读点 | 实际行为 | 风险 |
|-------|---------|------|
| "Default 是全局限流，不管参数" | Default 包含参数，参数变 Key 变 | 动态参数导致限流失效 |
| "ServerNode 是分布式限流" | ServerNode 是实例级，Redis 只是存储 | 多实例下总限流 = count × 实例数 |
| "User 限流对未登录用户不起作用" | 未登录用户 userId=null，全部共享 `"null"` | 所有未登录用户挤在一个桶 |
| "配置修改后所有桶立即更新" | 只有请求触发时才检查并更新 | 旧配置可能长期残留 |
| "Expression 像其他解析器一样包含方法签名" | Expression 仅返回表达式值 | 表达式简单时所有请求共享 |
| "count 是每个实例的配额" | 除 ServerNode 外都是全局配额 | 多实例部署时预期不符 |

---

## 四、文件位置索引

| 组件 | 路径 |
|------|------|
| @RateLimiter | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/annotation/RateLimiter.java` |
| RateLimiterAspect | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/aop/RateLimiterAspect.java` |
| RateLimiterRedisDAO | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/redis/RateLimiterRedisDAO.java` |
| RateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/RateLimiterKeyResolver.java` |
| DefaultRateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/impl/DefaultRateLimiterKeyResolver.java` |
| UserRateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/impl/UserRateLimiterKeyResolver.java` |
| ClientIpRateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/impl/ClientIpRateLimiterKeyResolver.java` |
| ServerNodeRateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/impl/ServerNodeRateLimiterKeyResolver.java` |
| ExpressionRateLimiterKeyResolver | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/ratelimiter/core/keyresolver/impl/ExpressionRateLimiterKeyResolver.java` |
| @Idempotent | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/idempotent/core/annotation/Idempotent.java` |
| IdempotentAspect | `yudao-framework/yudao-spring-boot-starter-protection/src/main/java/cn/iocoder/yudao/framework/idempotent/core/aop/IdempotentAspect.java` |
| WebFrameworkUtils | `yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/util/WebFrameworkUtils.java` |
