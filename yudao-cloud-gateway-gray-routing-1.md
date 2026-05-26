# yudao-gateway 灰度路由逻辑深度分析

## 一、整体架构与协作关系

### 1.1 涉及的核心类

| 类名 | 职责 | 文件路径 |
|------|------|----------|
| `GrayReactiveLoadBalancerClientFilter` | 灰度负载均衡过滤器，处理 `grayLb` scheme 的请求 | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/grey/GrayReactiveLoadBalancerClientFilter.java` |
| `GrayLoadBalancer` | 灰度负载均衡器，实现 version/tag 过滤和实例选择 | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/grey/GrayLoadBalancer.java` |
| `EnvUtils` | 环境工具类，处理 tag 的获取 | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/EnvUtils.java` |
| `TokenAuthenticationFilter` | Token 认证过滤器 | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/TokenAuthenticationFilter.java` |
| `AccessLogFilter` | 访问日志过滤器 | `yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/logging/AccessLogFilter.java` |

### 1.2 请求处理流程

```
请求进入 Gateway
    ↓
AccessLogFilter (order = Integer.MIN_VALUE) - 记录请求开始，包装 response
    ↓
TokenAuthenticationFilter (order = -100) - Token 解析和验证
    ↓
GrayReactiveLoadBalancerClientFilter (order = LOAD_BALANCER_CLIENT_FILTER_ORDER - 50) - 灰度路由判断
    ↓
GrayLoadBalancer.choose() - 选择服务实例
    ├─ 检查 instances 是否为空
    ├─ version 过滤
    ├─ tag 过滤
    └─ Nacos 权重选择
    ↓
默认 ReactiveLoadBalancerClientFilter (order = 10150) - 被灰度过滤器抢先处理 grayLb scheme
    ↓
AccessLogFilter (响应阶段) - 记录响应结束，打印日志
```

---

## 二、问题1：为什么复制 ReactiveLoadBalancerClientFilter 代码而不是继承？

### 2.1 核心原因

在 `GrayReactiveLoadBalancerClientFilter.java:28-30` 的类注释中明确说明了原因：

```java
/**
 * 由于 {@link ReactiveLoadBalancerClientFilter#choose(Request, String, Set)} 是 private 方法，无法进行重写。
 * 因此，这里只好 copy 它所有的代码，手动重写 choose 方法
 */
```

### 2.2 技术分析

`ReactiveLoadBalancerClientFilter` 的 `choose` 方法是 `private` 的：

```java
// Spring Cloud Gateway 源码中的 ReactiveLoadBalancerClientFilter
private Mono<Response<ServiceInstance>> choose(Request<RequestDataContext> lbRequest, 
                                               String serviceId,
                                               Set<LoadBalancerLifecycle> supportedLifecycleProcessors) {
    // ...
}
```

由于私有方法无法被子类重写，导致：
- 无法通过继承方式替换负载均衡器实现
- 只能通过复制整个类的方式进行定制

### 2.3 灰度过滤器的定制点

`GrayReactiveLoadBalancerClientFilter.choose()` 方法（第 122-129 行）是唯一的定制点：

```java
private Mono<Response<ServiceInstance>> choose(Request<RequestDataContext> lbRequest, String serviceId,
                                               Set<LoadBalancerLifecycle> supportedLifecycleProcessors) {
    // 修改 by 芋道源码：直接创建 GrayLoadBalancer 对象
    GrayLoadBalancer loadBalancer = new GrayLoadBalancer(
            clientFactory.getLazyProvider(serviceId, ServiceInstanceListSupplier.class), serviceId);
    supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
    return loadBalancer.choose(lbRequest);
}
```

对比默认实现，它直接创建了 `GrayLoadBalancer` 而不是通过工厂获取。

---

## 三、问题2：路由 scheme 为什么是 grayLb，什么情况下直接放行？

### 3.1 为什么使用 grayLb

从 `application.yaml` 中可以看到所有路由都使用 `grayLb://service-name` 的 scheme：

```yaml
routes:
  - id: system-admin-api
    uri: grayLb://system-server
    predicates:
      - Path=/admin-api/system/**
```

**设计原因：**

1. **区分灰度路由与普通路由**：通过 `grayLb` scheme 明确标记需要灰度路由的请求
2. **避免与默认 `lb` scheme 冲突**：默认的 `ReactiveLoadBalancerClientFilter` 会处理 `lb` scheme
3. **支持混合路由**：理论上可以部分路由用 `grayLb`（灰度），部分用 `lb`（普通）

### 3.2 判断逻辑

在 `GrayReactiveLoadBalancerClientFilter.filter()` 方法第 55-61 行：

```java
@Override
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
    String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
    // 修改 by 芋道源码：将 lb 替换成 grayLb，表示灰度负载均衡
    if (url == null || (!"grayLb".equals(url.getScheme()) && !"grayLb".equals(schemePrefix))) {
        return chain.filter(exchange);
    }
    // ... 灰度路由逻辑
}
```

### 3.3 直接放行的条件

**当以下任一条件满足时，直接放行（不处理灰度路由）：**

| 条件 | 说明 |
|------|------|
| `url == null` | 没有请求 URL 属性 |
| `url.getScheme() != "grayLb"` | URL 的 scheme 不是 `grayLb` |
| `schemePrefix != "grayLb"` | scheme prefix 也不是 `grayLb` |

**实际场景：**
- 使用 `http://` 或 `https://` 直接转发的请求
- 使用 `lb://` scheme 的请求（由默认过滤器处理）
- 非服务发现的静态路由

---

## 四、问题3：version 匹配逻辑，不匹配时是否回退？

### 4.1 version 匹配逻辑

在 `GrayLoadBalancer.getInstanceResponse()` 方法第 66-77 行：

```java
private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances, HttpHeaders headers) {
    // ... 空检查
    
    // 筛选满足 version 条件的实例列表
    String version = headers.getFirst(VERSION);  // VERSION = "version"
    List<ServiceInstance> chooseInstances;
    if (StrUtil.isEmpty(version)) {
        chooseInstances = instances;
    } else {
        chooseInstances = CollectionUtils.filterList(instances, 
                instance -> version.equals(instance.getMetadata().get("version")));
        if (CollUtil.isEmpty(chooseInstances)) {
            log.warn("[getInstanceResponse][serviceId({}) 没有满足版本({})的服务实例列表，直接使用所有服务实例列表]", serviceId, version);
            chooseInstances = instances;  // 回退！
        }
    }
    // ... tag 过滤和权重选择
}
```

### 4.2 version 匹配流程

```
请求 header 中获取 version
    ↓
version 为空？
    ├─ 是 → 使用全部实例
    └─ 否 → 过滤 metadata.version == header.version 的实例
                ↓
        过滤后有实例？
            ├─ 是 → 使用过滤后的实例
            └─ 否 → 回退使用全部实例（打印 WARN 日志）
```

### 4.3 关键结论

**是的，不匹配时会回退到全部实例。**

具体行为：
- **请求带了 version**：优先匹配 `metadata.version` 等于该值的实例
- **没有匹配的实例**：打印 WARN 日志，然后回退使用全部实例
- **请求没带 version**：不进行 version 过滤，使用全部实例

---

## 五、问题4：tag 过滤规则，header 无 tag 时为什么优先过滤有 tag 的实例？

### 5.1 tag 过滤逻辑

在 `GrayLoadBalancer.filterTagServiceInstances()` 方法第 95-115 行：

```java
private List<ServiceInstance> filterTagServiceInstances(List<ServiceInstance> instances, HttpHeaders headers) {
    // 情况一，没有 tag 时，过滤掉有 tag 的节点。目的：避免 test 环境，打到本地有 tag 的实例
    String tag = EnvUtils.getTag(headers);
    if (StrUtil.isEmpty(tag)) {
        List<ServiceInstance> chooseInstances = CollectionUtils.filterList(instances, 
                instance -> StrUtil.isEmpty(EnvUtils.getTag(instance)));
        // 【重要】补充说明：如果希望在 chooseInstances 为空时，不允许打到有 tag 的实例，可以取消注释下面的代码
        if (CollUtil.isEmpty(chooseInstances)) {
            log.warn("[filterTagServiceInstances][serviceId({}) 没有不带 tag 的服务实例列表，直接使用所有服务实例列表]", serviceId);
            chooseInstances = instances;
        }
        return chooseInstances;
    }

    // 情况二，有 tag 时，使用 tag 匹配服务实例
    List<ServiceInstance> chooseInstances = CollectionUtils.filterList(instances, 
            instance -> tag.equals(EnvUtils.getTag(instance)));
    if (CollUtil.isEmpty(chooseInstances)) {
        log.warn("[filterTagServiceInstances][serviceId({}) 没有满足 tag({}) 的服务实例列表，直接使用所有服务实例列表]", serviceId, tag);
        chooseInstances = instances;
    }
    return chooseInstances;
}
```

### 5.2 EnvUtils 的 tag 获取

在 `EnvUtils.java` 第 24-32 行：

```java
private static final String HEADER_TAG = "tag";

public static String getTag(HttpHeaders headers) {
    String tag = headers.getFirst(HEADER_TAG);
    // 如果请求的是 "${HOSTNAME}"，则解析成对应的本地主机名
    return Objects.equals(tag, HOST_NAME_VALUE) ? getHostName() : tag;
}

public static String getTag(ServiceInstance instance) {
    return instance.getMetadata().get(HEADER_TAG);
}
```

### 5.3 tag 过滤规则总结

| 场景 | 过滤规则 | 回退策略 |
|------|----------|----------|
| **请求无 tag** | 只保留实例 metadata.tag 为空的实例 | 如果全部实例都有 tag，则回退使用全部实例 |
| **请求有 tag** | 只保留实例 metadata.tag == header.tag 的实例 | 如果没有匹配的，回退使用全部实例 |

### 5.4 为什么 header 无 tag 时过滤有 tag 的实例？

从代码注释可以看出设计意图（第 96 行）：

```java
// 情况一，没有 tag 时，过滤掉有 tag 的节点。目的：避免 test 环境，打到本地有 tag 的实例
```

**设计背景：**

1. **本地开发场景**：开发者在本地启动服务时，通常会给实例设置一个 `tag`（比如自己的名字或主机名）
2. **测试环境隔离**：测试环境的请求不应该被路由到本地开发实例
3. **默认安全策略**：不带 tag 的请求应该走"标准"实例（没有 tag 的实例），而不是"特殊"实例（有 tag 的实例）

**典型场景：**
- 测试环境实例：无 tag
- 开发者本地实例：`tag = "zhangsan"` 或 `tag = "${HOSTNAME}"`
- 测试人员请求：无 tag → 只会打到测试环境实例
- 张三调试请求：`tag = "zhangsan"` → 打到张三的本地实例

---

## 六、问题5：各种异常场景的返回值

### 6.1 整体返回流程

`getInstanceResponse()` 方法第 59-84 行：

```java
private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances, HttpHeaders headers) {
    // 1. 无实例检查
    if (CollUtil.isEmpty(instances)) {
        log.warn("[getInstanceResponse][serviceId({}) 服务实例列表为空]", serviceId);
        return new EmptyResponse();
    }
    
    // 2. version 过滤（不匹配回退）
    // ...
    
    // 3. tag 过滤（不匹配回退）
    chooseInstances = filterTagServiceInstances(chooseInstances, headers);
    
    // 4. Nacos 权重选择
    return new DefaultResponse(NacosBalancer.getHostByRandomWeight3(chooseInstances));
}
```

### 6.2 各场景返回值

| 场景 | 返回类型 | 说明 |
|------|----------|------|
| **无实例** | `EmptyResponse` | `response.hasServer()` 返回 `false`，会抛出 `NotFoundException` |
| **version 无匹配** | 回退到全部实例 → `DefaultResponse` | 打印 WARN 日志，继续使用全部实例 |
| **tag 无匹配** | 回退到全部实例 → `DefaultResponse` | 打印 WARN 日志，继续使用全部实例 |
| **Nacos 权重选择** | `DefaultResponse` | 返回通过权重算法选中的实例 |

### 6.3 详细分析

#### 6.3.1 无实例场景

```java
if (CollUtil.isEmpty(instances)) {
    log.warn("[getInstanceResponse][serviceId({}) 服务实例列表为空]", serviceId);
    return new EmptyResponse();
}
```

返回 `EmptyResponse` 后，在 `GrayReactiveLoadBalancerClientFilter.filter()` 第 78-82 行处理：

```java
if (!response.hasServer()) {
    supportedLifecycleProcessors.forEach(lifecycle -> lifecycle
            .onComplete(new CompletionContext<>(CompletionContext.Status.DISCARD, lbRequest, response)));
    throw NotFoundException.create(properties.isUse404(), "Unable to find instance for " + url.getHost());
}
```

最终抛出 `NotFoundException`，根据配置返回 404 或 503。

#### 6.3.2 version 无匹配

```java
chooseInstances = CollectionUtils.filterList(instances, 
        instance -> version.equals(instance.getMetadata().get("version")));
if (CollUtil.isEmpty(chooseInstances)) {
    log.warn("[getInstanceResponse][serviceId({}) 没有满足版本({})的服务实例列表，直接使用所有服务实例列表]", serviceId, version);
    chooseInstances = instances;  // 回退
}
```

#### 6.3.3 tag 无匹配（两种情况）

**情况一：请求无 tag，但所有实例都有 tag**
```java
if (StrUtil.isEmpty(tag)) {
    List<ServiceInstance> chooseInstances = CollectionUtils.filterList(instances, 
            instance -> StrUtil.isEmpty(EnvUtils.getTag(instance)));
    if (CollUtil.isEmpty(chooseInstances)) {
        log.warn("[filterTagServiceInstances][serviceId({}) 没有不带 tag 的服务实例列表，直接使用所有服务实例列表]", serviceId);
        chooseInstances = instances;  // 回退
    }
    return chooseInstances;
}
```

**情况二：请求有 tag，但没有匹配的实例**
```java
List<ServiceInstance> chooseInstances = CollectionUtils.filterList(instances, 
        instance -> tag.equals(EnvUtils.getTag(instance)));
if (CollUtil.isEmpty(chooseInstances)) {
    log.warn("[filterTagServiceInstances][serviceId({}) 没有满足 tag({}) 的服务实例列表，直接使用所有服务实例列表]", serviceId, tag);
    chooseInstances = instances;  // 回退
}
```

#### 6.3.4 Nacos 权重选择

```java
return new DefaultResponse(NacosBalancer.getHostByRandomWeight3(chooseInstances));
```

使用 Nacos 提供的 `nacos.weight` metadata 进行加权随机选择。

---

## 七、问题6：过滤器顺序与调试关系

### 7.1 各过滤器的 order 值

| 过滤器 | Order 值 | 说明 |
|--------|----------|------|
| `AccessLogFilter` | `Ordered.HIGHEST_PRECEDENCE` (Integer.MIN_VALUE) | 最早执行 |
| `TokenAuthenticationFilter` | `-100` | 认证过滤器 |
| `GrayReactiveLoadBalancerClientFilter` | `LOAD_BALANCER_CLIENT_FILTER_ORDER - 50` | 灰度负载均衡过滤器 |
| 默认 `ReactiveLoadBalancerClientFilter` | `LOAD_BALANCER_CLIENT_FILTER_ORDER` (10150) | 默认负载均衡过滤器 |

### 7.2 执行顺序

```
Integer.MIN_VALUE (AccessLogFilter)
    ↓
-100 (TokenAuthenticationFilter)
    ↓
10150 - 50 = 10100 (GrayReactiveLoadBalancerClientFilter)
    ↓
10150 (ReactiveLoadBalancerClientFilter) - 对于 grayLb scheme 不会执行
```

### 7.3 协作关系与调试要点

#### 7.3.1 AccessLogFilter 与灰度过滤器

```
AccessLogFilter (请求阶段):
- 记录 startTime
- 记录 requestUrl、requestHeaders、queryParams
- 不修改请求，不影响灰度路由

GrayReactiveLoadBalancerClientFilter:
- 读取原始请求的 header (version, tag)
- 选择实例
- 设置 GATEWAY_REQUEST_URL_ATTR

AccessLogFilter (响应阶段):
- 记录 endTime、duration
- 记录 responseBody、httpStatus
- 记录 userId/userType（由 TokenAuthenticationFilter 设置）
```

**调试要点：**
- AccessLogFilter 记录的是**原始请求**的 URL 和 Header
- 灰度路由选择的实例信息不会出现在 AccessLog 中
- 但日志会记录最终的 `httpStatus`，可以判断请求是否成功

#### 7.3.2 TokenAuthenticationFilter 与灰度过滤器

```
TokenAuthenticationFilter (order = -100):
- 移除 login-user header（防止伪造）
- 解析 Token
- 设置 login-user header（认证通过时）
- 不会修改 version、tag 等灰度相关 header

GrayReactiveLoadBalancerClientFilter (order = 10100):
- 读取请求 header 中的 version、tag
- 这些 header 没有被 TokenAuthenticationFilter 修改
```

**调试要点：**
- Token 认证的成功与否**不影响**灰度路由逻辑
- 灰度路由完全依赖请求 header 中的 `version` 和 `tag`
- 即使 Token 无效，灰度路由仍会正常执行

#### 7.3.3 灰度过滤器与默认负载均衡过滤器

```
GrayReactiveLoadBalancerClientFilter (order = 10100):
- 只处理 grayLb scheme
- 处理后设置 GATEWAY_REQUEST_URL_ATTR
- 继续 chain.filter(exchange)

ReactiveLoadBalancerClientFilter (order = 10150):
- 处理 lb scheme
- 对于 grayLb scheme：GATEWAY_REQUEST_URL_ATTR 的 scheme 已被改为 http/https
- 所以默认过滤器会跳过（因为 scheme 不是 lb 或 grayLb）
```

**调试要点：**
- 两个过滤器的 `filter()` 方法都会被调用
- 但对于 `grayLb` 请求，默认过滤器会检查 scheme 并直接跳过
- 灰度过滤器在第 98-104 行修改了请求 URL：

```java
URI requestUrl = reconstructURI(serviceInstance, uri);
// ...
exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
```

---

## 八、问题7："明明带了 version 却没进灰度实例"排查清单

### 8.1 逐步排查清单

#### Step 1：确认请求真的带了 version header

**排查点：**
1. 检查请求是否真的包含 `version` header
2. 检查 header 名称是否正确（是 `version` 不是 `Version` 或 `VERSION`？虽然 HttpHeaders 不区分大小写）
3. 检查值是否有空格或特殊字符

**验证方法：**
- 在 AccessLog 中查看 `requestHeaders`
- 在 GrayLoadBalancer.getInstanceResponse() 第 67 行打断点或加日志：
  ```java
  String version = headers.getFirst(VERSION);
  log.info("请求 version = {}", version);  // 临时添加
  ```

#### Step 2：确认服务实例 metadata 中有 version

**排查点：**
1. 检查 Nacos 控制台中实例的 metadata
2. 确认实例 metadata 中有 `version` 字段
3. 确认值与请求中的 version 完全一致（区分大小写）

**验证方法：**
- 登录 Nacos 控制台 → 服务列表 → 查看实例详情 → Metadata
- 或者在 GrayLoadBalancer.getInstanceResponse() 第 72 行加日志：
  ```java
  chooseInstances = CollectionUtils.filterList(instances, instance -> {
      String metadataVersion = instance.getMetadata().get("version");
      log.info("实例 {} 的 version = {}", instance.getHost(), metadataVersion);
      return version.equals(metadataVersion);
  });
  ```

#### Step 3：确认路由使用了 grayLb scheme

**排查点：**
1. 检查 application.yaml 中的路由配置
2. 确认 uri 使用的是 `grayLb://service-name` 而不是 `lb://service-name`

**验证方法：**
- 检查配置文件第 34 行等：`uri: grayLb://system-server`
- 在 GrayReactiveLoadBalancerClientFilter.filter() 第 59 行打断点：
  ```java
  if (url == null || (!"grayLb".equals(url.getScheme()) && !"grayLb".equals(schemePrefix))) {
      log.info("跳过灰度路由，scheme = {}", url != null ? url.getScheme() : "null");
      return chain.filter(exchange);
  }
  ```

#### Step 4：确认没有被 tag 过滤掉

**排查点：**
1. 检查请求是否带了 `tag` header
2. 检查实例 metadata 中是否有 `tag`
3. 理解 tag 过滤逻辑可能会缩小实例范围

**验证方法：**
- 在 GrayLoadBalancer.filterTagServiceInstances() 第 97 行加日志：
  ```java
  String tag = EnvUtils.getTag(headers);
  log.info("请求 tag = {}", tag);
  ```

**注意：**
- 如果请求**没有** tag，会过滤掉**有** tag 的实例
- 如果请求**有** tag，只会匹配**相同** tag 的实例

#### Step 5：确认版本匹配后有实例存活

**排查点：**
1. 检查 version 匹配后的实例列表是否为空
2. 检查是否回退到了全部实例

**验证方法：**
- 查看日志中是否有：
  ```
  [getInstanceResponse][serviceId(xxx) 没有满足版本(v2)的服务实例列表，直接使用所有服务实例列表]
  ```

#### Step 6：确认 Nacos 权重配置

**排查点：**
1. 即使 version 匹配，Nacos 权重选择也可能选中"非预期"的实例
2. 检查实例的 `nacos.weight` 配置

**验证方法：**
- 在 Nacos 控制台查看实例权重
- 权重为 0 的实例不会被选中
- 权重相同的实例概率相等

#### Step 7：确认过滤器顺序和执行

**排查点：**
1. 确认 GrayReactiveLoadBalancerClientFilter 被正确注册
2. 确认没有被其他过滤器修改 header

**验证方法：**
- 检查类上是否有 `@Component` 注解
- 在 filter 方法入口加日志确认被调用

### 8.2 快速排查 Checklist

```
□ 请求 header 中确实有 version 字段
□ version 值与实例 metadata.version 完全一致（区分大小写）
□ 路由配置使用的是 grayLb:// 而非 lb://
□ version 匹配的实例确实在 Nacos 中健康存活
□ 没有被 tag 过滤逻辑意外排除
□ 匹配的实例权重不为 0
□ GrayReactiveLoadBalancerClientFilter 类被 Spring 扫描到
□ 查看日志中是否有 "没有满足版本" 的 WARN 信息
```

### 8.3 常见问题与解决方案

| 问题现象 | 可能原因 | 解决方案 |
|----------|----------|----------|
| 请求没走灰度过滤器 | 路由使用了 `lb://` 而非 `grayLb://` | 修改 application.yaml 中的 uri |
| version 匹配不到 | 实例 metadata 没有 version 或值不匹配 | 检查 Nacos 实例 metadata |
| 被 tag 过滤掉 | 请求无 tag 但实例有 tag，或 tag 值不匹配 | 检查 tag header 和实例 tag |
| 总是回退 | 灰度实例未注册或不健康 | 检查 Nacos 服务健康状态 |
| 选中权重低的实例 | 权重随机选择的正常现象 | 调高灰度实例权重便于测试 |

---

## 九、附录：核心代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| 灰度过滤器 order | `GrayReactiveLoadBalancerClientFilter.java` | 49-52 |
| grayLb scheme 判断 | `GrayReactiveLoadBalancerClientFilter.java` | 55-61 |
| 自定义 choose 方法 | `GrayReactiveLoadBalancerClientFilter.java` | 122-129 |
| version 过滤逻辑 | `GrayLoadBalancer.java` | 66-77 |
| tag 过滤逻辑 | `GrayLoadBalancer.java` | 79-115 |
| 无实例返回 | `GrayLoadBalancer.java` | 61-64 |
| Nacos 权重选择 | `GrayLoadBalancer.java` | 83 |
| EnvUtils tag 获取 | `EnvUtils.java` | 24-33 |
| Token 过滤器 order | `TokenAuthenticationFilter.java` | 164-166 |
| AccessLog 过滤器 order | `AccessLogFilter.java` | 105-107 |
| 路由配置 | `application.yaml` | 31-201 |
