# yudao-cloud 权限机制分析

## 一、权限体系概述

yudao-cloud 采用**双层权限叠加机制**：

1. **接口访问权限**（功能权限）：决定用户是否能访问某个接口
2. **SQL数据权限**（数据权限）：决定用户能查询到哪些数据行

两者在运行时**串行执行**，先通过接口权限校验，再在 SQL 执行时应用数据权限过滤。

---

## 二、关键组件职责边界

### 2.1 YudaoWebSecurityConfigurerAdapter

**职责**：Web 层的安全配置，负责 URL 级别的访问控制。

**核心逻辑**（`yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/config/YudaoWebSecurityConfigurerAdapter.java:110-153`）：

```
URL 权限规则优先级（从上到下匹配）：
├── ① 全局共享规则
│   ├── 静态资源（html/css/js）→ permitAll
│   ├── @PermitAll 注解的 URL → permitAll
│   └── yudao.security.permit-all-urls 配置的 URL → permitAll
├── ② 每个项目的自定义规则（AuthorizeRequestsCustomizer）
└── ③ 兜底规则：anyRequest() → authenticated（必须登录）
```

**关键方法**：
- `getPermitAllUrlsFromAnnotations()`：扫描所有 `@PermitAll` 注解的 Controller 方法，提取 URL 并加入免登录列表
- `filterChain()`：构建 Spring Security 过滤链，配置认证规则

**作用层**：**Web 层 / Servlet 过滤器层**

---

### 2.2 SecurityFrameworkServiceImpl

**职责**：提供细粒度的权限校验服务，被 `@PreAuthorize` 注解调用。

**核心方法**（`yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/service/SecurityFrameworkServiceImpl.java`）：

| 方法 | 作用 | 实现逻辑 |
|------|------|----------|
| `hasPermission(String)` | 判断是否有某权限 | 调用 permissionApi.hasAnyPermissions，带缓存 |
| `hasAnyPermissions(String...)` | 判断是否有任一权限 | 同上，带缓存 |
| `hasRole(String)` | 判断是否有某角色 | 调用 permissionApi.hasAnyRoles，带缓存 |
| `hasAnyRoles(String...)` | 判断是否有任一角色 | 同上，带缓存 |
| `hasScope(String)` | 判断是否有某授权范围 | 检查 LoginUser.scopes |
| `hasAnyScopes(String...)` | 判断是否有任一授权范围 | 同上 |

**作用层**：**Service 层 / AOP 代理层**（被 `@PreAuthorize` 切面调用）

---

### 2.3 DataPermissionAnnotationInterceptor

**职责**：AOP 拦截器，管理 `@DataPermission` 注解的上下文生命周期。

**核心逻辑**（`yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/core/aop/DataPermissionAnnotationInterceptor.java:32-48`）：

```
方法执行流程：
1. 入栈：DataPermissionContextHolder.add(dataPermission)
2. 执行原方法逻辑（可能包含数据库查询）
3. 出栈（finally 确保）：DataPermissionContextHolder.remove()
```

**查找注解优先级**（`DataPermissionAnnotationInterceptor.java:50-70`）：
1. 从缓存获取
2. 从方法上获取 `@DataPermission`
3. 从类上获取 `@DataPermission`

**配合的上下文容器**（`DataPermissionContextHolder`）：
- 使用 `LinkedList` 栈结构支持方法嵌套调用
- 使用 `TransmittableThreadLocal` 支持线程池传递

**作用层**：**DAO/Mapper 层**（拦截包含 `@DataPermission` 的方法）

---

### 2.4 DataPermissionRuleHandler

**职责**：MyBatis Plus 数据权限处理器，在 SQL 执行前动态注入权限条件。

**核心逻辑**（`yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/core/db/DataPermissionRuleHandler.java:31-62`）：

```
getSqlSegment() 执行流程：
1. skipPermissionCheck() → 是 → 返回 null（不追加条件）
2. 获取该 Mapper 对应的规则列表 → 空 → 返回 null
3. 遍历每个规则：
   - 表名不匹配 → 跳过
   - 调用 rule.getExpression() 获取条件 → null → 跳过
   - 用 AND 拼接到总条件
4. 返回拼接后的 Expression
```

**规则组合方式**：**AND**（多个规则的条件用 `AND` 连接）

**作用层**：**MyBatis 拦截层**（SQL 执行前拦截）

---

### 2.5 DeptDataPermissionRule

**职责**：基于部门的数据权限规则实现。

**核心逻辑**（`yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/core/rule/dept/DeptDataPermissionRule.java:89-144`）：

```
getExpression() 执行流程：
1. loginUser == null → 返回 null
2. userType != ADMIN → 返回 null  ← 注意：非管理员不处理！
3. 从上下文获取数据权限（无则调用 permissionApi 获取）
4. all == true → 返回 null（可查看全部，无需过滤）
5. deptIds 空 且 self == false → 返回 null=null（确保查不到）
6. 构建 deptExpression（dept_id IN ?）和 userExpression（user_id = ?）
7. 两者都为空 → 返回 null=null
8. 任一为空 → 返回非空那个
9. 都不为空 → 返回 (deptExpression OR userExpression)
```

**字段配置**：
- `deptColumns`：表名 → 部门字段名映射（默认 `dept_id`）
- `userColumns`：表名 → 用户字段名映射（默认 `user_id`）
- `TABLE_NAMES`：两表集合的并集

**作用层**：**数据权限规则层**（被 DataPermissionRuleHandler 调用）

---

## 三、问题回答

### 3.1 @PermitAll、authenticated、@PreAuthorize、@DataPermission 分别在哪些层生效？

| 注解/机制 | 生效层 | 触发时机 | 决策点 |
|-----------|--------|----------|--------|
| `@PermitAll` | **Web 层（Servlet Filter）** | 请求进入 Controller 之前 | YudaoWebSecurityConfigurerAdapter 扫描注解，将对应 URL 加入 `permitAll()` 列表，由 Spring Security FilterChain 判定 |
| `authenticated` | **Web 层（Servlet Filter）** | 请求进入 Controller 之前 | Spring Security FilterChain 兜底规则 `.anyRequest().authenticated()`，检查 SecurityContext 中是否有认证信息 |
| `@PreAuthorize` | **Service 层（Spring AOP）** | 方法调用时（进入 Controller 之后，执行业务逻辑之前） | Spring MethodSecurityInterceptor 切面，通过 SpEL 表达式调用 SecurityFrameworkService 的方法进行校验 |
| `@DataPermission` | **DAO/Mapper 层（MyBatis 拦截）** | SQL 执行前（业务逻辑中调用 Mapper 查询时） | DataPermissionAnnotationInterceptor AOP 入栈 → DataPermissionRuleFactory 根据上下文过滤规则 → DataPermissionRuleHandler 注入 SQL 条件 |

**执行时序图**：

```
HTTP 请求
    ↓
[FilterChain]
    ├─ @PermitAll？→ permitAll，直接通过
    └─ 否则 → authenticated？→ 未登录 → 401
                    ↓
[Controller]
    ↓
[MethodSecurityInterceptor]
    └─ @PreAuthorize(hasPermission/hasScope)？→ 不通过 → 403
                    ↓
[Service 业务逻辑]
    ↓
[Mapper 调用]
    ├─ DataPermissionAnnotationInterceptor 入栈 @DataPermission
    ├─ DataPermissionRuleHandler 拦截 SQL
    └─ DeptDataPermissionRule 构建 WHERE 条件
                    ↓
数据库查询（带数据权限条件）
```

---

### 3.2 skipPermissionCheck 对 hasPermission、hasScope、DataPermissionRuleHandler.getSqlSegment 分别有什么影响？

首先看 `skipPermissionCheck()` 的定义（`SecurityFrameworkUtils.java:150-160`）：

```java
public static boolean skipPermissionCheck() {
    LoginUser loginUser = getLoginUser();
    if (loginUser == null) {
        return false;
    }
    if (loginUser.getVisitTenantId() == null) {
        return false;
    }
    // 重点：跨租户访问时，无法进行权限校验
    return ObjUtil.notEqual(loginUser.getVisitTenantId(), loginUser.getTenantId());
}
```

**结论**：只有**跨租户访问**时（visitTenantId ≠ tenantId）才返回 `true`，其他情况均为 `false`。

---

**对 `hasPermission` 的影响**（`SecurityFrameworkServiceImpl.java:60-78`）：

```java
public boolean hasAnyPermissions(String... permissions) {
    if (skipPermissionCheck()) {
        return true;  // ← 直接放行
    }
    Long userId = getLoginUserId();
    if (userId == null) {
        return false;
    }
    return hasAnyPermissionsCache.get(...);  // 实际校验
}
```

| skipPermissionCheck | hasPermission 行为 |
|---------------------|-------------------|
| `true`（跨租户） | **直接返回 `true`**，跳过权限校验 |
| `false`（本租户） | 正常调用 permissionApi 校验用户权限 |

---

**对 `hasScope` 的影响**（`SecurityFrameworkServiceImpl.java:102-119`）：

```java
public boolean hasAnyScopes(String... scope) {
    if (skipPermissionCheck()) {
        return true;  // ← 直接放行
    }
    LoginUser user = SecurityFrameworkUtils.getLoginUser();
    if (user == null) {
        return false;
    }
    return CollUtil.containsAny(user.getScopes(), Arrays.asList(scope));  // 实际校验
}
```

| skipPermissionCheck | hasScope 行为 |
|---------------------|---------------|
| `true`（跨租户） | **直接返回 `true`**，跳过授权范围校验 |
| `false`（本租户） | 正常检查 LoginUser 的 scopes 集合 |

---

**对 `DataPermissionRuleHandler.getSqlSegment` 的影响**（`DataPermissionRuleHandler.java:31-62`）：

```java
public Expression getSqlSegment(Table table, Expression where, String mappedStatementId) {
    if (skipPermissionCheck()) {
        return null;  // ← 直接返回 null，不追加任何条件
    }
    // ... 后续规则处理
}
```

| skipPermissionCheck | getSqlSegment 行为 |
|---------------------|-------------------|
| `true`（跨租户） | **直接返回 `null`**，不处理任何数据权限规则，SQL 不追加任何条件（可查询全部数据） |
| `false`（本租户） | 正常获取规则列表，构建数据权限条件并注入 SQL |

---

**总结**：

| 方法 | skipPermissionCheck=true 时 | skipPermissionCheck=false 时 |
|------|----------------------------|-----------------------------|
| `hasPermission` | 直接 `return true` | 正常校验权限 |
| `hasScope` | 直接 `return true` | 正常校验授权范围 |
| `DataPermissionRuleHandler.getSqlSegment` | 直接 `return null`（SQL 无过滤） | 正常构建数据权限条件 |

**设计意图**：跨租户访问场景下，系统无法获取目标租户的权限配置，因此选择"完全放行"策略，由调用方确保安全性。

---

### 3.3 DeptDataPermissionRule 在各种情况下分别返回什么？

基于 `DeptDataPermissionRule.java:89-144` 的 `getExpression()` 方法：

```java
public Expression getExpression(String tableName, Alias tableAlias) {
    LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
    if (loginUser == null) {  // 情况 1
        return null;
    }
    if (ObjectUtil.notEqual(loginUser.getUserType(), UserTypeEnum.ADMIN.getValue())) {  // 情况 2
        return null;
    }
    DeptDataPermissionRespDTO deptDataPermission = ...;  // 获取数据权限
    if (deptDataPermission.getAll()) {  // 情况 3
        return null;
    }
    if (CollUtil.isEmpty(deptDataPermission.getDeptIds())
        && Boolean.FALSE.equals(deptDataPermission.getSelf())) {  // 情况 4
        return new EqualsTo(null, null);
    }
    Expression deptExpression = buildDeptExpression(...);
    Expression userExpression = buildUserExpression(...);
    if (deptExpression == null && userExpression == null) {  // 情况 5
        return new EqualsTo(null, null);
    }
    // ... 返回可用的表达式
}
```

**各分支分析**：

| 场景 | 判断条件 | 返回值 | 实际效果 |
|------|----------|--------|----------|
| **1. loginUser 为空** | `loginUser == null` | `null` | **不追加任何条件**。通常出现在未登录或跨服务无上下文的场景 |
| **2. 非 ADMIN 用户** | `userType != ADMIN(2)` | `null` | **不追加任何条件**。注意：这里的逻辑是"只有管理员才需要数据权限过滤"，非管理员不处理（这似乎是个设计决策，可能依赖其他机制或配置） |
| **3. all 为 true** | `deptDataPermission.getAll() == true` | `null` | **不追加任何条件**。可查看全部数据，无需过滤 |
| **4. deptIds 空 且 self 为 false** | `deptIds 为空` 且 `self == false` | `EqualsTo(null, null)` | **追加 `WHERE null = null`**，该条件恒为 false，确保查询结果为空 |
| **5. dept 和 user 表达式都无法构造** | `deptExpression == null` 且 `userExpression == null` | `EqualsTo(null, null)` | **追加 `WHERE null = null`**，同样确保查不到数据 |
| **正常情况** | 非上述情况 | 部门/用户条件表达式 | 追加 `WHERE (dept_id IN ? OR user_id = ?)` 等条件 |

---

**深入分析情况 4 和 5 的触发条件**：

**情况 4（deptIds 空且 self 为 false）**：
- 含义：该用户既不能查看任何部门的数据，也不能查看自己的数据
- 实际业务含义：完全没有数据权限
- 结果：`null = null` → 查不到任何数据

**情况 5（表达式都无法构造）**：
- `buildDeptExpression()` 返回 `null` 的条件：
  - 该表未配置 `deptColumns`（即没有部门字段）
  - 或 `deptIds` 为空（但这种情况已被情况 4 处理了）
- `buildUserExpression()` 返回 `null` 的条件：
  - 该表未配置 `userColumns`（即没有用户字段）
  - 或 `self == false`（但这种情况已被情况 4 处理了）
- 所以情况 5 实际触发于：**该表既没有 dept_id 也没有 user_id 字段，但数据权限要求过滤**
- 结果：`null = null` → 查不到任何数据

---

**总结**（`DeptDataPermissionRule` 返回值语义）：

| 返回值 | SQL 效果 | 业务含义 |
|--------|----------|----------|
| `null` | 不追加条件 | 不过滤，可查全部 |
| `EqualsTo(null, null)` | `WHERE null = null` | 过滤掉所有行，查不到数据 |
| `InExpression(dept_id IN ...)` | `WHERE dept_id IN (?,?)` | 只能查指定部门的数据 |
| `EqualsTo(user_id = ?)` | `WHERE user_id = ?` | 只能查自己的数据 |
| `ParenthesedExpressionList(OR)` | `WHERE (dept_id IN ? OR user_id = ?)` | 可查指定部门或自己的数据 |

---

### 3.4 多个 DataPermissionRule 为什么用 AND 组合？

**源码位置**（`DataPermissionRuleHandler.java:43-60`）：

```java
Expression allExpression = null;
for (DataPermissionRule rule : rules) {
    if (!rule.getTableNames().contains(tableName)) {
        continue;
    }
    Expression oneExpress = rule.getExpression(tableName, table.getAlias());
    if (oneExpress == null) {
        continue;
    }
    // 拼接到 allExpression 中
    allExpression = allExpression == null ? oneExpress
            : new AndExpression(allExpression, oneExpress);  // ← AND 连接
}
```

**AND 组合的设计逻辑**：

#### 3.4.1 安全性原则（默认"最严格"策略）

数据权限的核心目标是**防止越权访问**。在多规则场景下：

- **AND**：必须同时满足所有规则的条件 → 结果集是各规则的**交集**
- **OR**：满足任一规则即可 → 结果集是各规则的**并集**

**示例**：
- 规则 A：只能看本部门数据 → `dept_id IN (1,2)`
- 规则 B：只能看未删除数据 → `deleted = 0`

| 组合方式 | SQL 条件 | 结果 |
|----------|----------|------|
| AND（实际） | `dept_id IN (1,2) AND deleted = 0` | 本部门的未删除数据 ✅ |
| OR（假设） | `dept_id IN (1,2) OR deleted = 0` | 本部门数据 **或** 任何未删除数据 ❌（越权风险） |

---

#### 3.4.2 规则独立性

每个 `DataPermissionRule` 代表**一个独立的权限维度**：

| 规则类型 | 作用 |
|----------|------|
| `DeptDataPermissionRule` | 基于部门的数据隔离 |
| （可自定义）租户规则 | 基于租户的数据隔离 |
| （可自定义）业务规则 | 基于业务状态的数据过滤 |

这些规则之间是**独立约束**，而非**可选替代**。用户必须同时满足所有维度的权限要求。

---

#### 3.4.3 空值（null）的语义

规则返回 `null` 表示**"该规则不约束此查询"**，而非"该规则拒绝"：

```
规则 A 返回 null（不约束）
规则 B 返回 dept_id IN (1,2)
规则 C 返回 null（不约束）
→ 最终：dept_id IN (1,2)（只应用 B 的约束）
```

这配合 AND 逻辑非常合理：跳过不相关的规则，只应用相关规则的约束。

---

#### 3.4.4 与 `@DataPermission` 的配合

通过 `@DataPermission` 注解可以**细粒度控制规则应用**：

```java
// 只启用部门权限规则，禁用其他规则
@DataPermission(includeRules = DeptDataPermissionRule.class)
public List<User> getUsers() { ... }

// 禁用数据权限
@DataPermission(enable = false)
public List<AllData> getAllData() { ... }
```

这让开发者可以在方法级别精确控制**哪些规则生效**，而无需改变 AND 的默认组合逻辑。

---

**总结**：AND 组合是数据权限的安全默认策略，确保多维度约束同时满足，配合 `null` 语义和注解控制，既安全又灵活。

---

### 3.5 给出"接口权限通过但 SQL 查不到数据"的推导

#### 场景设定

用户：张三（普通管理员，userType=ADMIN）
- 部门：研发部（dept_id=100）
- 数据权限配置：
  - `all = false`（不能看全部）
  - `deptIds = [100]`（只能看研发部）
  - `self = false`（不能只看自己）
- 接口权限：有 `user:list` 权限

#### 推导步骤

**步骤 1：接口权限校验（通过）**

```
HTTP 请求 GET /admin-api/system/user/list
    ↓
[FilterChain]
    ├─ 不是 @PermitAll
    └─ authenticated → 有登录态，通过
                    ↓
[Controller]
    ↓
[MethodSecurityInterceptor]
    └─ @PreAuthorize("hasAuthority('user:list')")
        └─ SecurityFrameworkServiceImpl.hasPermission("user:list")
            ├─ skipPermissionCheck() → false（本租户）
            └─ permissionApi.hasAnyPermissions(1, ["user:list"]) → true
                                    ↓
                          ✅ 接口权限通过
```

**步骤 2：业务逻辑执行**

```java
// UserService.java
public PageResult<UserRespVO> getUserPage(UserPageReqVO reqVO) {
    // ...
    return userMapper.selectPage(reqVO);  // ← 触发数据权限
}
```

**步骤 3：数据权限拦截**

```
[Mapper 调用]
    ├─ 方法有 @DataPermission（默认 enable=true）
    └─ DataPermissionAnnotationInterceptor 入栈
                    ↓
[DataPermissionRuleHandler.getSqlSegment()]
    ├─ skipPermissionCheck() → false
    ├─ ruleFactory.getDataPermissionRule() → [DeptDataPermissionRule]
    └─ 遍历规则：
        └─ DeptDataPermissionRule.getExpression("system_user", alias)
                    ↓
[DeptDataPermissionRule.getExpression()]
    ├─ loginUser != null ✅
    ├─ userType == ADMIN(2) ✅
    ├─ deptDataPermission.all → false
    ├─ deptIds=[100], self=false
    ├─ buildDeptExpression("system_user", ...)
    │   ├─ deptColumns.get("system_user") → "dept_id" ✅（表有 dept_id 字段）
    │   └─ deptIds=[100] 非空
    │   └─ 返回：dept_id IN (100)
    ├─ buildUserExpression(...)
    │   └─ self=false → 返回 null
    ├─ deptExpression 非空，userExpression 为空
    └─ 返回：dept_id IN (100)
                    ↓
[SQL 注入]
    原 SQL：SELECT * FROM system_user WHERE status = 0
    注入后：SELECT * FROM system_user WHERE status = 0 AND dept_id IN (100)
```

**步骤 4：数据查询结果**

| 表中实际数据 | 过滤条件 | 查询结果 |
|-------------|----------|----------|
| id=1, dept_id=100, name=张三 | `dept_id IN (100)` ✅ | 查到 |
| id=2, dept_id=100, name=李四 | `dept_id IN (100)` ✅ | 查到 |
| id=3, dept_id=200, name=王五 | `dept_id IN (100)` ❌ | **查不到** |
| id=4, dept_id=200, name=赵六 | `dept_id IN (100)` ❌ | **查不到** |

**最终结果**：
- ✅ 接口权限通过（200 OK）
- ❌ SQL 只能查到 dept_id=100 的数据（王五、赵六的数据被过滤掉）
- 表现为：接口调用成功，但返回的数据列表不完整

---

**另一种极端场景：完全查不到**

如果张三的数据权限配置为：
- `deptIds = []`（空）
- `self = false`

则 `DeptDataPermissionRule.getExpression()` 走到：
```java
if (CollUtil.isEmpty(deptDataPermission.getDeptIds())
    && Boolean.FALSE.equals(deptDataPermission.getSelf())) {
    return new EqualsTo(null, null);  // WHERE null = null
}
```

最终 SQL：`SELECT * FROM system_user WHERE status = 0 AND null = null`

`null = null` 在 SQL 中是 `UNKNOWN`，等效于 `false`，**查不到任何数据**。

---

### 3.6 给出"SQL 数据权限理论上允许但方法权限先拒绝"的推导

#### 场景设定

用户：李四（超级管理员，userType=ADMIN）
- 数据权限配置：`all = true`（可看全部数据）
- 接口权限：**没有** `user:list` 权限
- 数据表中：有 100 条用户数据（李四理论上都能看到）

#### 推导步骤

**步骤 1：接口权限校验（失败）**

```
HTTP 请求 GET /admin-api/system/user/list
    ↓
[FilterChain]
    ├─ 不是 @PermitAll
    └─ authenticated → 有登录态，通过
                    ↓
[Controller]
    ↓
[MethodSecurityInterceptor]
    └─ @PreAuthorize("hasAuthority('user:list')")
        └─ SecurityFrameworkServiceImpl.hasPermission("user:list")
            ├─ skipPermissionCheck() → false（本租户）
            ├─ userId = 2（李四）
            └─ permissionApi.hasAnyPermissions(2, ["user:list"]) → false
                                    ↓
                          ❌ 接口权限拒绝
                                    ↓
                          AccessDeniedException
                                    ↓
                          HTTP 403 Forbidden
```

**请求在这一步就被拦截了，后续流程不再执行。**

---

**如果接口权限通过后，数据权限会如何（理论推导）**

假设李四有 `user:list` 权限，继续往下走：

```
[Mapper 调用]
    ↓
[DeptDataPermissionRule.getExpression()]
    ├─ loginUser != null ✅
    ├─ userType == ADMIN(2) ✅
    └─ deptDataPermission.all → true  ← 可看全部
        └─ return null  // 不追加任何条件
                    ↓
[SQL 注入]
    原 SQL：SELECT * FROM system_user WHERE status = 0
    注入后：SELECT * FROM system_user WHERE status = 0  // 无额外条件
                    ↓
    查询结果：100 条数据全部返回
```

---

**时序对比**

| 阶段 | 实际场景（无接口权限） | 假设场景（有接口权限） |
|------|------------------------|------------------------|
| FilterChain authenticated | ✅ 通过 | ✅ 通过 |
| @PreAuthorize hasAuthority | ❌ **拒绝（403）** | ✅ 通过 |
| Service 执行 | 未执行 | 执行 |
| Mapper 调用 | 未执行 | 执行 |
| DeptDataPermissionRule | 未执行 | 返回 null（不过滤） |
| SQL 执行 | 未执行 | `SELECT * FROM system_user WHERE status = 0` |
| 数据返回 | 403 错误 | 100 条数据 |

---

**关键结论**

两种权限的**校验顺序**是：

```
接口权限（@PreAuthorize） → SQL 数据权限（DataPermissionRule）
```

如果接口权限先拒绝，SQL 数据权限**根本不会执行**。

这意味着：
- 即使数据权限配置为 `all=true`（理论上可看全部）
- 如果没有 `@PreAuthorize` 要求的权限
- 请求会在更早阶段被拒绝，根本到不了 SQL 执行这一步

---

## 四、权限叠加流程图

```
                    ┌─────────────────────────┐
                    │     HTTP 请求入口       │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │  FilterChain (Web 层)   │
                    │                         │
                    │ 1. @PermitAll URL?      │
                    │    ├─ 是 → permitAll     │
                    │    └─ 否 → authenticated │
                    │       ├─ 未登录 → 401    │
                    │       └─ 已登录 → 继续   │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │ MethodSecurity (AOP层)  │
                    │                         │
                    │ @PreAuthorize 校验      │
                    │ hasPermission/hasScope  │
                    │    ├─ 不通过 → 403      │
                    │    └─ 通过 → 继续       │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Service 业务逻辑      │
                    │                         │
                    │  执行业务操作...         │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Mapper 数据访问       │
                    │                         │
                    │ @DataPermission AOP     │
                    │ DataPermissionRule      │
                    │    ├─ 构建过滤条件       │
                    │    └─ 注入 SQL WHERE    │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │      数据库查询          │
                    │                         │
                    │ 执行带权限条件的 SQL     │
                    └─────────────────────────┘
```

---

## 五、关键决策点速查表

| 决策点 | 类/方法 | 位置 | 核心逻辑 |
|--------|---------|------|----------|
| @PermitAll 免登录 | `getPermitAllUrlsFromAnnotations()` | YudaoWebSecurityConfigurerAdapter:159 | 扫描注解，加入 permitAll 列表 |
| 兜底必须登录 | `anyRequest().authenticated()` | YudaoWebSecurityConfigurerAdapter:148 | 非 permitAll 的 URL 必须认证 |
| 权限校验缓存 | `hasAnyPermissionsCache` | SecurityFrameworkServiceImpl:48-57 | Guava Cache，1 分钟过期 |
| 跨租户跳过校验 | `skipPermissionCheck()` | SecurityFrameworkUtils:150 | visitTenantId ≠ tenantId 时返回 true |
| 规则组合方式 | `new AndExpression()` | DataPermissionRuleHandler:59 | 多个规则用 AND 连接 |
| 无权限数据过滤 | `new EqualsTo(null, null)` | DeptDataPermissionRule:122,134 | 返回 null=null 确保查不到 |
| 全部数据权限 | `getAll() == true` | DeptDataPermissionRule:115 | 返回 null，不过滤 |
| 注解上下文管理 | `DataPermissionContextHolder` | DataPermissionContextHolder:19 | LinkedList 栈 + TransmittableThreadLocal |
| 规则过滤（include/exclude） | `getDataPermissionRule()` | DataPermissionRuleFactoryImpl:34-65 | 根据 @DataPermission 过滤规则列表 |

---

## 六、附录：用户类型判断的关键发现

**重要发现**：在 `DeptDataPermissionRule.java:96` 中：

```java
// 只有管理员类型的用户，才进行数据权限的处理
if (ObjectUtil.notEqual(loginUser.getUserType(), UserTypeEnum.ADMIN.getValue())) {
    return null;
}
```

结合 `UserTypeEnum.java:17-18`：

```java
MEMBER(1, "会员"),   // 面向 c 端，普通用户
ADMIN(2, "管理员");  // 面向 b 端，管理后台
```

**结论**：
- `userType = 1`（MEMBER/C 端会员）：`DeptDataPermissionRule` 返回 `null`，**不做数据权限过滤**
- `userType = 2`（ADMIN/B 端管理员）：才会执行部门数据权限逻辑

这意味着 C 端用户（会员）在设计上不使用基于部门的数据权限机制。
