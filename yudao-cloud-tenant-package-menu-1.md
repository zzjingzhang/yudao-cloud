# yudao-cloud 系统租户模块分析

## 一、核心数据模型

### 1. TenantDO（租户信息）

**文件位置**：`yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/tenant/TenantDO.java

| 字段 | 含义 | 说明 |
|------|------|------|
| id | 租户编号 | 自增主键 |
| name | 租户名称 | 唯一 |
| status | 租户状态 | 枚举 CommonStatusEnum（启用/禁用） |
| websites | 绑定域名列表 | 兼容微信小程序 appid |
| packageId | 租户套餐编号 | 关联 TenantPackageDO.id；系统内置租户使用 PACKAGE_ID_SYSTEM(0) |
| expireTime | 过期时间 | 租户有效期 |
| accountCount | 账号数量 | 允许创建的最大账号数 |

### 2. TenantPackageDO（租户套餐）

**文件位置**：`yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/tenant/TenantPackageDO.java

| 字段 | 含义 | 说明 |
|------|------|------|
| id | 套餐编号 | 自增主键 |
| name | 套餐名称 | 唯一 |
| status | 套餐状态 | 枚举 CommonStatusEnum（启用/禁用） |
| menuIds | 关联的菜单编号集合 | 使用 JacksonTypeHandler 序列化存储 |
| remark | 备注 | 套餐说明 |

## 二、关键组件关系图

```
TenantController (管理后台)        AppTenantController (用户App)
           ↓                             ↓
           └─────────────┬──────────────┘
                         ↓
                    TenantService (Service层)
                    /         \
                   /           \
      TenantPackageService     TenantApiImpl (Feign RPC)
             |                    |
             ↓                    ↓
    TenantPackageDO            TenantFrameworkServiceImpl (框架层)
             |                    |
             ↓                    ↓
    (套餐菜单管理         TenantSecurityWebFilter (Web过滤器)
```

## 三、问题解答

### 问题1：创建或更新租户套餐时菜单权限如何保存并影响租户

#### 1.1 创建租户套餐

**调用链路：`TenantPackageServiceImpl.createTenantPackage()`

```
TenantController.create()
    → TenantPackageServiceImpl.createTenantPackage()
        1. 校验套餐名称唯一
        2. BeanUtils.toBean() 将 VO 转换为 DO
        3. tenantPackageMapper.insert() 保存到数据库
        4. 返回套餐ID
```

**关键点**：创建套餐时，仅将 `menuIds` 以 JSON 格式存储在 `system_tenant_package` 表中。此时不会影响任何现有租户。

#### 1.2 更新租户套餐

**调用链路**：`TenantPackageServiceImpl.updateTenantPackage()`

```
TenantController.update()
    → TenantPackageServiceImpl.updateTenantPackage()
        1. 校验套餐存在
        2. 校验套餐名称唯一
        3. 更新套餐信息（含 menuIds）
        4. 关键判断：如果菜单发生变化
           if (!CollUtil.isEqualList(tenantPackage.getMenuIds(), updateReqVO.getMenuIds())
           {
               5. 查询所有使用该套餐的租户
                  List<TenantDO> tenants = tenantService.getTenantListByPackageId(tenantPackage.getId());
               6. 遍历租户，同步更新角色菜单
                  tenants.forEach(tenant -> 
                      tenantService.updateTenantRoleMenu(tenant.getId(), updateReqVO.getMenuIds())
                  );
           }
```

#### 1.3 updateTenantRoleMenu 实现

**文件位置**：`TenantServiceImpl.updateTenantRoleMenu()`

```java
@DSTransactional
public void updateTenantRoleMenu(Long tenantId, Set<Long> menuIds) {
    TenantUtils.execute(tenantId, () -> {
        // 1. 获取该租户下所有角色
        List<RoleDO> roles = roleService.getRoleList();
        
        roles.forEach(role -> {
            // 2. 租户管理员角色：直接分配套餐全部菜单
            if (Objects.equals(role.getCode(), RoleCodeEnum.TENANT_ADMIN.getCode())) {
                permissionService.assignRoleMenu(role.getId(), menuIds);
                return;
            }
            // 3. 其他角色：取交集（保留原有权限与套餐权限的交集）
            Set<Long> roleMenuIds = permissionService.getRoleMenuListByRoleId(role.getId());
            roleMenuIds = CollUtil.intersectionDistinct(roleMenuIds, menuIds);
            permissionService.assignRoleMenu(role.getId(), roleMenuIds);
        });
    });
}
```

**影响范围**：
- 租户管理员角色（TENANT_ADMIN）：权限被重置为套餐菜单权限
- 其他自定义角色：只保留在套餐权限范围内的权限（取交集）

#### 1.4 创建租户时的套餐关联

**调用链路**：`TenantServiceImpl.createTenant()`

```java
@DSTransactional
public Long createTenant(TenantSaveReqVO createReqVO) {
    1. 校验名称、域名唯一
    2. 校验套餐未禁用
       TenantPackageDO tenantPackage = tenantPackageService.validTenantPackage(createReqVO.getPackageId());
    
    3. 插入租户
       TenantDO tenant = BeanUtils.toBean(createReqVO, TenantDO.class);
       tenantMapper.insert(tenant);
    
    4. 在租户上下文中创建租户管理员
       TenantUtils.execute(tenant.getId(), () -> {
           // 创建租户管理员角色，分配套餐菜单
           Long roleId = createRole(tenantPackage);
           // 创建用户并分配角色
       });
}
```

**createRole 实现**：
```java
private Long createRole(TenantPackageDO tenantPackage) {
    1. 创建 TENANT_ADMIN 角色
    2. 分配套餐菜单
       permissionService.assignRoleMenu(roleId, tenantPackage.getMenuIds());
}
```

**结论**：创建租户时，套餐的菜单会作为租户管理员角色的初始权限。

#### 1.5 更新租户时的套餐变更

**调用链路**：`TenantServiceImpl.updateTenant()`

```java
@DSTransactional
public void updateTenant(TenantSaveReqVO updateReqVO) {
    1. 校验存在、名称、域名
    2. 校验套餐未禁用
    3. 更新租户信息
    4. 关键判断：如果套餐ID变化
       if (ObjectUtil.notEqual(tenant.getPackageId(), updateReqVO.getPackageId())) {
           updateTenantRoleMenu(tenant.getId(), tenantPackage.getMenuIds());
       }
}
```

### 问题2：租户过期、禁用、域名或 website 查询时分别如何校验

#### 2.1 租户过期和禁用校验

**核心校验方法**：`TenantServiceImpl.validTenant()`

```java
public void validTenant(Long id) {
    1. 校验存在
       TenantDO tenant = getTenant(id);
       if (tenant == null) {
           throw exception(TENANT_NOT_EXISTS);
       }
    2. 校验状态：禁用
       if (tenant.getStatus().equals(CommonStatusEnum.DISABLE.getStatus())) {
           throw exception(TENANT_DISABLE, tenant.getName());
       }
    3. 校验过期时间
       if (DateUtils.isExpired(tenant.getExpireTime())) {
           throw exception(TENANT_EXPIRE, tenant.getName());
       }
}
```

**校验时机**：
- 其他模块通过 `TenantApiImpl.validTenant()` → `TenantServiceImpl.validTenant()
- Web请求通过 `TenantSecurityWebFilter` 拦截

#### 2.2 域名（websites）校验

**创建/更新租户时**：`TenantServiceImpl.validTenantWebsiteDuplicate()

```java
private void validTenantWebsiteDuplicate(List<String> websites, Long excludeId) {
    if (CollUtil.isEmpty(websites)) {
        return;
    }
    websites.forEach(website -> {
        1. 查询使用该域名的租户
           List<TenantDO> tenants = tenantMapper.selectListByWebsite(website);
        2. 排除当前租户
           if (excludeId != null) {
               tenants.removeIf(tenant -> tenant.getId().equals(excludeId));
           }
        3. 如果还有其他租户使用该域名，抛出异常
           if (CollUtil.isNotEmpty(tenants)) {
               throw exception(TENANT_WEBSITE_DUPLICATE, website);
           }
    });
}
```

#### 2.3 通过 website 查询租户

**管理后台**：`TenantController.getTenantByWebsite()`

```java
@GetMapping("/get-by-website")
@PermitAll
@TenantIgnore
public CommonResult<TenantRespVO> getTenantByWebsite(@RequestParam("website") String website) {
    1. 查询租户
       TenantDO tenant = tenantService.getTenantByWebsite(website);
    2. 如果租户不存在或禁用，返回 null
       if (tenant == null || CommonStatusEnum.isDisable(tenant.getStatus())) {
           return success(null);
       }
    3. 返回租户ID和名称
       return success(new TenantRespVO().setId(tenant.getId()).setName(tenant.getName()));
}
```

**用户App**：`AppTenantController.getTenantByWebsite()`

```java
@GetMapping("/get-by-website")
@PermitAll
@TenantIgnore
public CommonResult<AppTenantRespVO> getTenantByWebsite(@RequestParam("website") String website) {
    1. 查询租户
       TenantDO tenant = tenantService.getTenantByWebsite(website);
    2. 如果租户不存在或禁用，返回 null
       if (tenant == null || CommonStatusEnum.isDisable(tenant.getStatus())) {
           return success(null);
       }
    3. 返回完整租户信息
       return success(BeanUtils.toBean(tenant, AppTenantRespVO.class));
}
```

**关键点**：通过 website 查询时，只校验状态（禁用的租户返回 null），**不校验过期时间。

#### 2.4 通过租户名称查询

**管理后台**：`TenantController.getTenantIdByName()`

```java
@GetMapping("/get-id-by-name")
@PermitAll
@TenantIgnore
public CommonResult<Long> getTenantIdByName(@RequestParam("name") String name) {
    TenantDO tenant = tenantService.getTenantByName(name);
    return success(tenant != null ? tenant.getId() : null);
}
```

**关键点**：通过名称查询时，**不校验状态和过期时间**，只要存在就返回。

### 问题3：TenantApiImpl 被其它模块调用时返回哪些租户信息

**文件位置**：`yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/api/tenant/TenantApiImpl.java

#### 3.1 提供的接口

```java
@RestController
@Validated
public class TenantApiImpl implements TenantCommonApi {

    @Resource
    private TenantService tenantService;

    // 1. 获取所有租户ID列表
    @Override
    @TenantIgnore
    public CommonResult<List<Long>> getTenantIdList() {
        return success(tenantService.getTenantIdList());
    }

    // 2. 校验租户是否有效
    @Override
    @TenantIgnore
    public CommonResult<Boolean> validTenant(Long id) {
        tenantService.validTenant(id);
        return success(true);
    }
}
```

#### 3.2 接口说明

| 接口 | 返回值 | 说明 |
|------|--------|------|
| getTenantIdList() | List<Long> | 返回所有租户的 ID 列表 |
| validTenant(Long id) | Boolean | 校验通过返回 true；校验失败抛出异常 |

#### 3.3 调用方：`TenantFrameworkServiceImpl`

**文件位置**：`yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/service/TenantFrameworkServiceImpl.java

```java
@RequiredArgsConstructor
public class TenantFrameworkServiceImpl implements TenantFrameworkService {

    private final TenantCommonApi tenantApi;

    // 缓存：租户ID列表缓存，1分钟过期
    private final LoadingCache<Object, List<Long>> getTenantIdsCache = buildAsyncReloadingCache(
            Duration.ofMinutes(1L),
            new CacheLoader<Object, List<Long>>() {
                @Override
                public List<Long> load(Object key) {
                    return tenantApi.getTenantIdList().getCheckedData();
                }
            });

    // 缓存：租户校验结果缓存，1分钟过期
    private final LoadingCache<Long, CommonResult<Boolean>> validTenantCache = buildAsyncReloadingCache(
            Duration.ofMinutes(1L),
            new CacheLoader<Long, CommonResult<Boolean>>() {
                @Override
                public CommonResult<Boolean> load(Long id) {
                    return tenantApi.validTenant(id);
                }
            });

    @Override
    @SneakyThrows
    public List<Long> getTenantIds() {
        return getTenantIdsCache.get(Boolean.TRUE);
    }

    @Override
    @SneakyThrows
    public void validTenant(Long id) {
        validTenantCache.get(id).checkError();
    }
}
```

**关键点**：
- `getTenantIdList()` 返回所有租户 ID，不区分状态和过期时间
- `validTenant()` 校验租户是否存在、启用、未过期，校验失败抛出异常
- 两个接口都有 1 分钟的缓存

### 问题4：租户套餐变更后已有租户的菜单权限是否自动变化，依据是什么

#### 4.1 答案：是，依据

**自动变化的场景**：

| 场景 | 是否自动变化 |
|------|-------------|
| 套餐菜单变更（更新套餐） | ✅ 自动变化 |
| 租户套餐变更（更新租户的 packageId） | ✅ 自动变化 |

#### 4.2 套餐菜单变更时的自动同步

**依据代码**：`TenantPackageServiceImpl.updateTenantPackage()`

```java
@DSTransactional
public void updateTenantPackage(TenantPackageSaveReqVO updateReqVO) {
    1. 校验存在、名称
    2. 更新套餐
    3. **关键**：如果菜单发生变化
       if (!CollUtil.isEqualList(tenantPackage.getMenuIds(), updateReqVO.getMenuIds())) {
           // 查询所有使用该套餐的租户
           List<TenantDO> tenants = tenantService.getTenantListByPackageId(tenantPackage.getId());
           // 遍历租户，同步更新角色菜单
           tenants.forEach(tenant ->
               tenantService.updateTenantRoleMenu(tenant.getId(), updateReqVO.getMenuIds())
           );
       }
}
```

**执行逻辑**：
- 找到所有使用该套餐的租户
- 对每个租户调用 `updateTenantRoleMenu()` 更新所有角色的权限

#### 4.3 租户套餐变更时的自动同步

**依据代码**：`TenantServiceImpl.updateTenant()`

```java
@DSTransactional
public void updateTenant(TenantSaveReqVO updateReqVO) {
    1. 校验存在、名称、域名、套餐
    2. 更新租户
    3. **关键**：如果套餐ID变化
       if (ObjectUtil.notEqual(tenant.getPackageId(), updateReqVO.getPackageId())) {
           updateTenantRoleMenu(tenant.getId(), tenantPackage.getMenuIds());
       }
}
```

#### 4.4 自动同步的权限策略

**依据代码**：`TenantServiceImpl.updateTenantRoleMenu()`

```java
@DSTransactional
public void updateTenantRoleMenu(Long tenantId, Set<Long> menuIds) {
    TenantUtils.execute(tenantId, () -> {
        List<RoleDO> roles = roleService.getRoleList();
        
        roles.forEach(role -> {
            // 租户管理员：直接分配套餐菜单
            if (Objects.equals(role.getCode(), RoleCodeEnum.TENANT_ADMIN.getCode())) {
                permissionService.assignRoleMenu(role.getId(), menuIds);
                return;
            }
            // 其他角色：取交集（去掉超过套餐的权限
            Set<Long> roleMenuIds = permissionService.getRoleMenuListByRoleId(role.getId());
            roleMenuIds = CollUtil.intersectionDistinct(roleMenuIds, menuIds);
            permissionService.assignRoleMenu(role.getId(), roleMenuIds);
        });
    });
}
```

**策略说明**：
- 租户管理员角色（TENANT_ADMIN）：权限被重置为新套餐的菜单
- 其他角色：保留原有权限与新套餐权限的交集（收缩权限只减不增
- 事务保证：使用 `@DSTransactional 保证多数据源事务

### 问题5：TenantInfoHandler 和 TenantMenuHandler 在初始化或刷新缓存时分别做什么

#### 5.1 接口定义

**TenantInfoHandler**

```java
public interface TenantInfoHandler {
    /**
     * 基于传入的租户信息，进行相关逻辑的执行
     * 例如说，创建用户时，超过最大账户配额
     */
    void handle(TenantDO tenant);
}
```

**TenantMenuHandler**

```java
public interface TenantMenuHandler {
    /**
     * 基于传入的租户菜单【全】列表，进行相关逻辑的执行
     * 例如说，返回可分配菜单的时候，可以移除多余的
     */
    void handle(Set<Long> menuIds);
}
```

#### 5.2 TenantService 中的调用

**文件位置**：`TenantServiceImpl`

```java
@Override
public void handleTenantInfo(TenantInfoHandler handler) {
    // 如果禁用，则不执行逻辑
    if (isTenantDisable()) {
        return;
    }
    // 获得租户
    TenantDO tenant = getTenant(TenantContextHolder.getRequiredTenantId());
    // 执行处理器
    handler.handle(tenant);
}

@Override
public void handleTenantMenu(TenantMenuHandler handler) {
    // 如果禁用，则不执行逻辑
    if (isTenantDisable()) {
        return;
    }
    // 获得租户，然后获得菜单
    TenantDO tenant = getTenant(TenantContextHolder.getRequiredTenantId());
    Set<Long> menuIds;
    if (isSystemTenant(tenant)) { // 系统租户，菜单是全量的
        menuIds = CollectionUtils.convertSet(menuService.getMenuList(), MenuDO::getId);
    } else {
        menuIds = tenantPackageService.getTenantPackage(tenant.getPackageId()).getMenuIds();
    }
    // 执行处理器
    handler.handle(menuIds);
}
```

#### 5.3 实际使用场景

**场景1：菜单列表查询**：`MenuServiceImpl.getMenuListByTenant()`

```java
@Override
public List<MenuDO> getMenuListByTenant(MenuListReqVO reqVO) {
    // 查询所有菜单
    List<MenuDO> menus = getMenuList(reqVO);
    // 开启多租户的情况下，需要过滤掉未开通的菜单
    tenantService.handleTenantMenu(menuIds -> 
        menus.removeIf(menu -> !CollUtil.contains(menuIds, menu.getId())));
    return menus;
}
```

**作用**：过滤掉租户套餐中不包含的菜单。

**场景2：角色菜单分配**：`PermissionController.assignRoleMenu()

```java
@PostMapping("/assign-role-menu")
public CommonResult<Boolean> assignRoleMenu(@Validated @RequestBody PermissionAssignRoleMenuReqVO reqVO) {
    // 开启多租户的情况下，需要过滤掉未开通的菜单
    tenantService.handleTenantMenu(menuIds -> 
        reqVO.getMenuIds().removeIf(menuId -> !CollUtil.contains(menuIds, menuId)));
    
    // 执行菜单的分配
    permissionService.assignRoleMenu(reqVO.getRoleId(), reqVO.getMenuIds());
    return success(true);
}
```

**作用**：防止分配租户套餐外的菜单权限。

**场景3：用户创建校验**（假设的）：`AdminUserServiceImpl` 中可能使用 handleTenantInfo 校验账号数量

#### 5.4 初始化和刷新缓存

**关键点**：
- 这两个接口是**策略模式**的设计，用于将租户逻辑与业务逻辑解耦
- **没有直接的缓存机制
- `handleTenantInfo()` 每次调用时实时查询数据库获取租户信息
- `handleTenantMenu()` 每次调用时实时查询租户，然后获取菜单列表
- 系统租户（packageId=0）获取全量菜单；其他租户获取套餐菜单

### 问题6：构造一个"租户套餐删除、禁用或过期但 token 仍有效"的请求场景，推导最终会在哪些层被拦截或放行

#### 6.1 场景构造

**场景描述**：
1. 用户 A 属于租户 T1，套餐 P1
2. 用户 A 登录成功，获得有效 token（JWT）
3. 管理员操作：
   - 情况1：禁用租户 T1
   - 情况2：租户 T1 过期
   - 情况3：删除套餐 P1（实际上无法删除，因为有租户使用
   - 情况4：禁用套餐 P1
4. 用户 A 使用 token 发送请求

#### 6.2 各层校验流程

**请求流程**：

```
请求 → TenantSecurityWebFilter → Controller → Service → ...
```

**详细流程**：

##### 第1层：TenantSecurityWebFilter

**文件位置**：`yudao-framework/yudao-spring-boot-starter-biz-tenant/src/main/java/cn/iocoder/yudao/framework/tenant/core/security/TenantSecurityWebFilter.java

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
    Long tenantId = TenantContextHolder.getTenantId();
    
    // 1. 登录用户，获取租户ID
    LoginUser user = SecurityFrameworkUtils.getLoginUser();
    if (user != null) {
        if (tenantId == null) {
            tenantId = user.getTenantId();
            TenantContextHolder.setTenantId(tenantId);
        } else if (!Objects.equals(user.getTenantId(), TenantContextHolder.getTenantId())) {
            // 越权访问
            ServletUtils.writeJSON(response, CommonResult.error(FORBIDDEN, "您无权访问该租户的数据"));
            return;
        }
    }
    
    // 2. 非忽略URL，校验租户
    if (!isIgnoreUrl(request)) {
        if (tenantId == null) {
            // 未传递租户ID
            ServletUtils.writeJSON(response, CommonResult.error(BAD_REQUEST, "请求的租户标识未传递"));
            return;
        }
        // 3. 校验租户合法
        try {
            tenantFrameworkService.validTenant(tenantId);
        } catch (Throwable ex) {
            CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
            ServletUtils.writeJSON(response, result);
            return;
        }
    }
    
    chain.doFilter(request, response);
}
```

##### 第2层：TenantFrameworkServiceImpl

```java
@Override
@SneakyThrows
public void validTenant(Long id) {
    validTenantCache.get(id).checkError();
}
```

- 先查缓存（1分钟过期）
- 缓存命中：使用缓存结果
- 缓存未命中：调用 TenantApiImpl.validTenant()

##### 第3层：TenantApiImpl

```java
@Override
@TenantIgnore
public CommonResult<Boolean> validTenant(Long id) {
    tenantService.validTenant(id);
    return success(true);
}
```

##### 第4层：TenantServiceImpl

```java
@Override
public void validTenant(Long id) {
    TenantDO tenant = getTenant(id);
    if (tenant == null) {
        throw exception(TENANT_NOT_EXISTS);
    }
    if (tenant.getStatus().equals(CommonStatusEnum.DISABLE.getStatus())) {
        throw exception(TENANT_DISABLE, tenant.getName());
    }
    if (DateUtils.isExpired(tenant.getExpireTime())) {
        throw exception(TENANT_EXPIRE, tenant.getName());
    }
}
```

#### 6.3 各种情况的拦截分析

##### 情况1：禁用租户 T1

**拦截位置**：
1. `TenantSecurityWebFilter` → `TenantFrameworkServiceImpl` → `TenantApiImpl` → `TenantServiceImpl.validTenant()`

**拦截点**：
```java
if (tenant.getStatus().equals(CommonStatusEnum.DISABLE.getStatus())) {
    throw exception(TENANT_DISABLE, tenant.getName());
}
```

**结果**：抛出 `TENANT_DISABLE` 异常，返回给前端

**缓存影响**：
- 如果缓存未过期（1分钟内），可能放行（缓存了之前的校验成功的结果
- 缓存过期后，重新校验，拦截

##### 情况2：租户 T1 过期

**拦截位置**：同情况1

**拦截点**：
```java
if (DateUtils.isExpired(tenant.getExpireTime())) {
    throw exception(TENANT_EXPIRE, tenant.getName());
}
```

**结果**：抛出 `TENANT_EXPIRE` 异常

##### 情况3：删除套餐 P1

**实际上**：无法删除，因为有租户使用

**依据代码**：`TenantPackageServiceImpl.deleteTenantPackage()`

```java
@Override
public void deleteTenantPackage(Long id) {
    1. 校验存在
    2. **校验正在使用**
       validateTenantUsed(id);
    3. 删除
}

private void validateTenantUsed(Long id) {
    if (tenantService.getTenantCountByPackageId(id) > 0) {
        throw exception(TENANT_PACKAGE_USED);
    }
}
```

**结果**：删除套餐时会校验是否有租户使用，有则抛出异常，删除失败。

##### 情况4：禁用套餐 P1

**关键点**：
- 套餐禁用不影响已创建租户的现有权限
- 影响：
  - 创建新租户时会校验套餐是否禁用
  - 套餐禁用后，租户仍可使用（租户的权限不变

**依据代码**：
- 创建租户时校验：`TenantServiceImpl.createTenant()

```java
// 校验套餐被禁用
TenantPackageDO tenantPackage = tenantPackageService.validTenantPackage(createReqVO.getPackageId());
```

- 更新租户时校验：`TenantServiceImpl.updateTenant()`

```java
// 校验套餐被禁用
TenantPackageDO tenantPackage = tenantPackageService.validTenantPackage(updateReqVO.getPackageId());
```

- 更新套餐禁用校验：`TenantPackageServiceImpl.validTenantPackage()

```java
public TenantPackageDO validTenantPackage(Long id) {
    TenantPackageDO tenantPackage = tenantPackageMapper.selectById(id);
    if (tenantPackage == null) {
        throw exception(TENANT_PACKAGE_NOT_EXISTS);
    }
    if (tenantPackage.getStatus().equals(CommonStatusEnum.DISABLE.getStatus())) {
        throw exception(TENANT_PACKAGE_DISABLE, tenantPackage.getName());
    }
    return tenantPackage;
}
```

**结论**：
- 禁用套餐 P1 后，已有租户 T1 的现有请求**不会被拦截**
- 但无法新建租户、更新租户（换套餐时会被拦截

#### 6.4 放行情况总结

| 场景 | 拦截层 | 拦截点 | 结果 |
|------|--------|--------|------|
| 禁用租户 | WebFilter → FrameworkService → ApiImpl → Service | validTenant() 中 status 校验 | 拦截，TENANT_DISABLE |
| 租户过期 | 同上 | validTenant() 中 expireTime 校验 | 拦截，TENANT_EXPIRE |
| 删除套餐（有租户使用） | 无法删除，删除时已被阻止 | deleteTenantPackage() 中 validateTenantUsed | 删除失败 |
| 禁用套餐 | 不拦截现有租户请求 | 仅影响新建/更新租户 | 现有租户可继续使用 |

#### 6.5 特殊情况：缓存影响

**缓存机制**：`TenantFrameworkServiceImpl` 有 1 分钟缓存

**时间窗口问题**：
1. T=0：用户登录，token 有效，缓存校验结果（成功）
2. T=30s：管理员禁用租户
3. T=40s：用户请求，缓存未过期（60s），**放行**
4. T=70s：用户请求，缓存过期，重新校验，**拦截**

**结论**：
- 存在最多 1 分钟的时间窗口，禁用/过期后，缓存未过期时可能放行
- 缓存过期后才会被拦截

## 四、组件协作关系总结

### 4.1 菜单权限同步链路

```
套餐更新（menuIds变化）
    ↓
TenantPackageServiceImpl.updateTenantPackage()
    ↓
查询使用该套餐的所有租户
    ↓
遍历租户
    ↓
TenantServiceImpl.updateTenantRoleMenu()
    ↓
租户管理员角色：重置为套餐菜单
其他角色：取交集（收缩权限）
    ↓
PermissionService.assignRoleMenu()
    ↓
更新角色-菜单关联
```

### 4.2 租户校验链路

```
HTTP 请求（带 token）
    ↓
TenantSecurityWebFilter
    ↓
TenantFrameworkServiceImpl.validTenant()
    ↓
缓存命中？
    ├─ 是 → 使用缓存结果
    └─ 否 → TenantApiImpl.validTenant()
                    ↓
              TenantServiceImpl.validTenant()
                    ↓
              校验存在、状态、过期时间
                    ↓
              校验通过/失败
```

### 4.3 关键设计亮点

1. **策略模式解耦：TenantInfoHandler、TenantMenuHandler 接口
2. **多级缓存：1 分钟缓存减轻 RPC 调用
3. **事务保证：@DSTransactional 多数据源事务
4. **权限收缩：套餐变更时只减不增（其他角色取交集
5. **分层校验：WebFilter 层拦截，Service 层校验
