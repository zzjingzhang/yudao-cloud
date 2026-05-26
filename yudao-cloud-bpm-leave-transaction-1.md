# BpmOALeaveServiceImpl.createLeave 发起请假流程事务边界与一致性风险分析

## 一、核心代码协作关系

### 1.1 涉及类与文件位置

| 类名 | 文件路径 | 职责 |
|------|----------|------|
| BpmOALeaveServiceImpl | `yudao-module-bpm-server/src/main/java/cn/iocoder/yudao/module/bpm/service/oa/BpmOALeaveServiceImpl.java` | 请假申请业务服务，实现 createLeave 方法 |
| BpmProcessInstanceApi | `yudao-module-bpm-api/src/main/java/cn/iocoder/yudao/module/bpm/api/task/BpmProcessInstanceApi.java` | Feign 接口定义，声明 createProcessInstance 方法 |
| BpmProcessInstanceApiImpl | `yudao-module-bpm-server/src/main/java/cn/iocoder/yudao/module/bpm/api/task/BpmProcessInstanceApiImpl.java` | API 实现类，委托给 BpmProcessInstanceService |
| BpmProcessInstanceServiceImpl | `yudao-module-bpm-server/src/main/java/cn/iocoder/yudao/module/bpm/service/task/BpmProcessInstanceServiceImpl.java` | 流程实例核心服务，调用 Flowable 引擎创建流程 |
| BpmOALeaveDO | `yudao-module-bpm-server/src/main/java/cn/iocoder/yudao/module/bpm/dal/dataobject/oa/BpmOALeaveDO.java` | 请假单数据对象 |
| BpmOALeaveMapper | `yudao-module-bpm-server/src/main/java/cn/iocoder/yudao/module/bpm/dal/mysql/oa/BpmOALeaveMapper.java` | MyBatis Mapper，操作 bpm_oa_leave 表 |

### 1.2 调用链路

```
BpmOALeaveServiceImpl.createLeave()
    ├── leaveMapper.insert(BpmOALeaveDO)        // 第1步：插入请假单
    ├── processInstanceApi.createProcessInstance()  // 第2步：发起BPM流程
    │       └── BpmProcessInstanceApiImpl.createProcessInstance()
    │               └── BpmProcessInstanceServiceImpl.createProcessInstance()
    │                       └── Flowable RuntimeService.start()
    └── leaveMapper.updateById()                // 第3步：更新请假单 processInstanceId
```

---

## 二、三步操作的顺序和数据依赖分析

### 2.1 createLeave 方法核心逻辑（BpmOALeaveServiceImpl.java:46-65）

```java
@Override
@Transactional(rollbackFor = Exception.class)
public Long createLeave(Long userId, BpmOALeaveCreateReqVO createReqVO) {
    // 第1步：插入 OA 请假单
    long day = LocalDateTimeUtil.between(createReqVO.getStartTime(), createReqVO.getEndTime()).toDays();
    BpmOALeaveDO leave = BeanUtils.toBean(createReqVO, BpmOALeaveDO.class)
            .setUserId(userId).setDay(day).setStatus(BpmTaskStatusEnum.RUNNING.getStatus());
    leaveMapper.insert(leave);  // 此时 leave.id 已生成（主键自增）

    // 第2步：发起 BPM 流程
    Map<String, Object> processInstanceVariables = new HashMap<>();
    processInstanceVariables.put("day", day);
    String processInstanceId = processInstanceApi.createProcessInstance(userId,
            new BpmProcessInstanceCreateReqDTO().setProcessDefinitionKey(PROCESS_KEY)
                    .setVariables(processInstanceVariables)
                    .setBusinessKey(String.valueOf(leave.getId()))  // 依赖第1步生成的 leave.id
                    .setStartUserSelectAssignees(createReqVO.getStartUserSelectAssignees())).getCheckedData();

    // 第3步：将工作流的编号，更新到 OA 请假单中
    leaveMapper.updateById(new BpmOALeaveDO()
            .setId(leave.getId())  // 依赖第1步生成的 leave.id
            .setProcessInstanceId(processInstanceId));  // 依赖第2步返回的 processInstanceId
    return leave.getId();
}
```

### 2.2 三步顺序与数据依赖

| 步骤 | 操作 | 前置依赖 | 产生数据 | 后续依赖 |
|------|------|----------|----------|----------|
| 第1步 | `leaveMapper.insert(leave)` | 无 | `leave.id`（主键自增） | 第2步（businessKey）、第3步（更新条件） |
| 第2步 | `processInstanceApi.createProcessInstance()` | `leave.id`（用于 businessKey） | `processInstanceId` | 第3步（更新字段值） |
| 第3步 | `leaveMapper.updateById()` | `leave.id`（WHERE条件） + `processInstanceId`（SET值） | 请假单完整数据 | 无 |

### 2.3 关键数据结构（BpmOALeaveDO）

```java
@TableName("bpm_oa_leave")
public class BpmOALeaveDO extends BaseDO {
    @TableId
    private Long id;                    // 请假单主键
    private Long userId;                // 申请人
    private Integer type;               // 请假类型
    private String reason;              // 原因
    private LocalDateTime startTime;    // 开始时间
    private LocalDateTime endTime;      // 结束时间
    private Long day;                   // 请假天数
    private Integer status;             // 审批状态（RUNNING等）
    private String processInstanceId;   // 关联的流程实例ID（第3步才填充）
}
```

---

## 三、@Transactional 事务边界分析

### 3.1 调用性质判定

**关键点：`processInstanceApi` 的调用性质**

从代码分析：
1. `BpmProcessInstanceApi` 接口标注了 `@FeignClient(name = ApiConstants.NAME)`，理论上支持微服务远程调用
2. 但 `BpmOALeaveServiceImpl` 和 `BpmProcessInstanceApiImpl`、`BpmProcessInstanceServiceImpl` 都在 **同一个模块（yudao-module-bpm-server）** 内
3. `BpmProcessInstanceApiImpl` 是 `@RestController`，直接注入 `BpmProcessInstanceService`
4. 实际运行时，Spring 会选择本地 Bean 注入（同一 JVM），**不会走 Feign 远程调用**

**结论：当前是单体/同模块本地调用，不是跨服务远程调用。**

### 3.2 事务传播机制

| 方法 | @Transactional | 事务行为 |
|------|----------------|----------|
| `BpmOALeaveServiceImpl.createLeave()` | `@Transactional(rollbackFor = Exception.class)` | 开启主事务（默认 REQUIRED） |
| `BpmProcessInstanceApiImpl.createProcessInstance()` | 无 | 加入主事务 |
| `BpmProcessInstanceServiceImpl.createProcessInstance(DTO)` | 无 | 加入主事务 |
| `BpmProcessInstanceServiceImpl.createProcessInstance(VO)` | `@Transactional(rollbackFor = Exception.class)` | DTO 入口不会走此方法 |
| `BpmProcessInstanceServiceImpl.createProcessInstance0()` | 无 | 加入主事务 |

### 3.3 事务覆盖范围

**问题2答案：@Transactional 只覆盖调用方本地库，但由于是同模块本地调用，BPM 服务内部操作也在同一事务中。**

```
事务边界（同一个物理事务）：
┌─────────────────────────────────────────────────────────────┐
│  BpmOALeaveServiceImpl.createLeave()  @Transactional        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  ① leaveMapper.insert()    ──→  DB: bpm_oa_leave 插入  │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  ② processInstanceApi.createProcessInstance()          │  │
│  │     └→ BpmProcessInstanceApiImpl                       │  │
│  │        └→ BpmProcessInstanceServiceImpl                │  │
│  │           └→ Flowable RuntimeService.start()          │  │
│  │              ──→  DB: Flowable 系列表（ACT_*）写入      │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  ③ leaveMapper.updateById()  ──→  DB: bpm_oa_leave 更新 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↑ 任一异常 → 整个事务回滚（包括 Flowable 数据）
```

**注意：Flowable 的数据库操作也参与同一 JDBC 事务，因为使用的是同一个 DataSource 和 Connection。**

---

## 四、一致性风险场景分析

### 4.1 场景一：processInstanceApi 成功但 updateById 失败

**问题3答案：在当前实现下，不会出现此情况。**

因为：
1. 三步在 **同一个物理事务** 中
2. 如果 `updateById` 失败（抛出异常），Spring 事务管理器会标记事务回滚
3. 回滚范围包括：
   - `leaveMapper.insert()` 的插入
   - Flowable `RuntimeService.start()` 对 ACT_* 表的写入
   - （自然也包括 `updateById` 本身，虽然它还没提交）

**最终结果：请假单不存在，流程实例也不存在，数据一致。**

### 4.2 场景二：processInstanceApi 抛异常时

**问题4答案：请假单会回滚。**

执行流程：
1. `leaveMapper.insert()` 执行成功（未提交）
2. `processInstanceApi.createProcessInstance()` 抛出异常（如流程定义不存在、校验失败等）
3. 异常传播到 `createLeave()` 方法
4. `@Transactional` 触发回滚
5. `insert` 操作被撤销

**最终结果：请假单不存在，流程实例也不存在，数据一致。**

### 4.3 潜在风险：系统崩溃/网络分区（极端场景）

虽然代码层面在同一事务，但存在以下极端情况：

**风险点：事务提交瞬间崩溃**

```
时间线：
T1: insert(leave) 执行完成
T2: createProcessInstance() 执行完成，Flowable 数据写入
T3: updateById() 执行完成
T4: 事务管理器执行 commit
    ├─ 发送 COMMIT 指令到 MySQL
    └─ [崩溃点] MySQL 收到 COMMIT 但响应丢失 / 应用在收到响应前崩溃
```

**影响：**
- 数据库侧：事务可能已提交（取决于崩溃时机）
- 应用侧：认为操作失败，可能触发重试
- 结果：可能重复创建请假单和流程实例

**风险点：如果未来拆分为微服务（processInstanceApi 走 Feign）**

如果 `BpmOALeaveServiceImpl` 和 `BpmProcessInstanceApiImpl` 部署在不同服务：

```
服务A（请假服务）               服务B（BPM服务）
┌─────────────────┐           ┌─────────────────┐
│ @Transactional  │  Feign    │ @Transactional  │
│ 事务A           │ ───────→  │ 事务B（独立）    │
│                 │           │                 │
│ ① insert(leave) │           │ ② 创建流程实例   │
│                 │ ←───────  │ 事务B提交成功    │
│ ③ updateById()  │           │                 │
│ 事务A回滚 ×     │           │ 事务B无法回滚 √  │
└─────────────────┘           └─────────────────┘
```

**此时问题3的答案会变成：是的，可能出现。**
- 流程实例已创建（服务B事务已提交）
- 请假单回滚或 processInstanceId 未更新
- 数据不一致

---

## 五、businessKey 的作用及回调依赖

### 5.1 businessKey 的定义与赋值

**BpmProcessInstanceCreateReqDTO.java:21-23**
```java
@Schema(description = "业务的唯一标识", requiredMode = Schema.RequiredMode.REQUIRED)
@NotEmpty(message = "业务的唯一标识不能为空")
private String businessKey; // 例如说，请假申请的编号。通过它，可以查询到对应的实例
```

**在 createLeave 中的赋值（BpmOALeaveServiceImpl.java:59）**
```java
.setBusinessKey(String.valueOf(leave.getId()))
```

**传入 Flowable（BpmProcessInstanceServiceImpl.java:818-821）**
```java
ProcessInstanceBuilder processInstanceBuilder = runtimeService.createProcessInstanceBuilder()
        .processDefinitionId(definition.getId())
        .businessKey(businessKey)  // 传入 Flowable 引擎
        .variables(variables);
```

### 5.2 businessKey = leave.id 的作用

| 作用 | 说明 |
|------|------|
| **关联业务数据与流程实例** | 通过 `businessKey = leave.id` 建立请假单 ↔ 流程实例的双向关联 |
| **流程侧查询业务** | 流程实例中存储 `businessKey`，可通过 `HistoricProcessInstance.getBusinessKey()` 找到对应请假单 |
| **业务侧查询流程** | 请假单中存储 `processInstanceId`（第3步更新），可通过流程ID查询流程详情 |
| **事件回调的桥梁** | 流程状态变化事件中携带 `businessKey`，监听器据此定位要更新的业务数据 |

### 5.3 回调机制与状态更新

#### 5.3.1 事件发布（BpmProcessInstanceEventPublisher）

```java
public class BpmProcessInstanceEventPublisher {
    private final ApplicationEventPublisher publisher;
    
    public void sendProcessInstanceResultEvent(@Valid BpmProcessInstanceStatusEvent event) {
        publisher.publishEvent(event);
    }
}
```

#### 5.3.2 事件结构（BpmProcessInstanceStatusEvent）

```java
public class BpmProcessInstanceStatusEvent extends ApplicationEvent {
    private String id;                    // 流程实例ID
    private String processDefinitionKey;  // 流程定义Key（如 "oa_leave"）
    private Integer status;               // 流程状态（通过/拒绝等）
    private String reason;                // 结束原因
    private String businessKey;           // ← 关键：业务标识 = leave.id
}
```

#### 5.3.3 监听器实现（BpmOALeaveStatusListener）

```java
@Component
public class BpmOALeaveStatusListener extends BpmProcessInstanceStatusEventListener {
    
    @Resource
    private BpmOALeaveService leaveService;
    
    @Override
    protected String getProcessDefinitionKey() {
        return BpmOALeaveServiceImpl.PROCESS_KEY;  // "oa_leave"
    }
    
    @Override
    protected void onEvent(BpmProcessInstanceStatusEvent event) {
        // 核心：使用 businessKey 定位请假单
        leaveService.updateLeaveStatus(
            Long.parseLong(event.getBusinessKey()),  // leave.id
            event.getStatus()                        // 新状态
        );
    }
}
```

#### 5.3.4 更新状态（BpmOALeaveServiceImpl.updateLeaveStatus）

```java
@Override
public void updateLeaveStatus(Long id, Integer status) {
    validateLeaveExists(id);
    leaveMapper.updateById(new BpmOALeaveDO().setId(id).setStatus(status));
}
```

### 5.4 完整回调链路

```
流程审批结束（通过/拒绝）
    ↓
BpmProcessInstanceEventPublisher.sendProcessInstanceResultEvent()
    ↓ 发布 Spring ApplicationEvent
BpmProcessInstanceStatusEventListener.onApplicationEvent()
    ↓ 过滤 processDefinitionKey = "oa_leave"
BpmOALeaveStatusListener.onEvent()
    ↓ 解析 businessKey = leave.id
leaveService.updateLeaveStatus(leaveId, newStatus)
    ↓
leaveMapper.updateById()  →  更新 bpm_oa_leave.status
```

### 5.5 回调依赖 businessKey 的关键点

**问题5答案：businessKey 是连接流程实例与业务数据的核心桥梁。**

1. **发起时**：`businessKey = leave.id` 写入 Flowable 流程实例
2. **回调时**：监听器从事件中获取 `businessKey`，解析为 `leave.id`
3. **更新时**：以 `leave.id` 为条件更新请假单状态

**如果没有 businessKey：**
- 事件中没有业务标识
- 监听器无法知道该更新哪条业务数据
- 只能通过 `processInstanceId` 反查，但业务表可能没有索引或需要额外查询

---

## 六、一致性修复/补偿方案对比

### 6.1 方案一：状态机 + 定时任务补偿（推荐）

**核心思想**

在 `bpm_oa_leave` 表增加 `init_status` 字段，标记初始化状态：
- `INIT`: 已插入但流程未启动/未关联
- `PROCESSING`: 流程已启动，等待关联
- `SUCCESS`: 初始化完成
- `FAILED`: 初始化失败

**实现步骤**

1. **第1步 insert 时**：设置 `init_status = INIT`
2. **第2步 启动流程后**：设置 `init_status = PROCESSING`
3. **第3步 updateById 后**：设置 `init_status = SUCCESS`
4. **定时任务**：每分钟扫描 `init_status = PROCESSING` 超过 N 秒的记录
   - 查询 Flowable：是否存在 `businessKey = leave.id` 的流程实例
   - 若存在：补填 `processInstanceId`，设置 `init_status = SUCCESS`
   - 若不存在：设置 `init_status = FAILED`，或发起补偿流程

**优缺点**

| 优点 | 缺点 |
|------|------|
| 不依赖分布式事务，最终一致性 | 有短暂不一致窗口 |
| 实现简单，侵入性小 | 需要额外字段和定时任务 |
| 适用于单体和微服务架构 | 定时任务有延迟 |

**代价评估**：低（开发1-2天，运维成本低）

---

### 6.2 方案二：事件驱动 + 本地消息表（可靠事件模式）

**核心思想**

利用"本地消息表"模式保证业务操作和事件发送的原子性。

**实现步骤**

1. **创建本地消息表** `bpm_event_message`
   - `id`, `business_key`, `event_type`, `payload`, `status(NEW/SENT/FAILED)`, `retry_count`

2. **createLeave 改造**（同一事务）：
   ```java
   @Transactional
   public Long createLeave(...) {
       // 1. 插入请假单
       leaveMapper.insert(leave);
       
       // 2. 插入"待启动流程"消息（同一事务）
       eventMessageMapper.insert(new EventMessage()
           .setBusinessKey(String.valueOf(leave.getId()))
           .setEventType("START_PROCESS")
           .setStatus("NEW"));
       
       return leave.getId();
   }
   ```

3. **独立的消息发布器**（异步）：
   - 定时扫描 `status = NEW` 的消息
   - 调用 `processInstanceApi.createProcessInstance()`
   - 成功后更新请假单 `processInstanceId`，标记消息 `SENT`
   - 失败则重试，达到上限标记 `FAILED` + 告警

**优缺点**

| 优点 | 缺点 |
|------|------|
| 业务操作和消息发送原子性强 | 实现复杂度较高 |
| 天然支持重试机制 | 需要消息表、发布器、重试策略 |
| 可扩展到其他业务场景 | 异步化，调用方不能立即拿到 processInstanceId |

**代价评估**：中（开发3-5天，需设计消息状态机）

---

### 6.3 方案三：Seata AT 模式（分布式事务）

**核心思想**

如果未来拆分为微服务，引入 Seata AT 模式实现分布式事务。

**架构**

```
TM（事务管理器） ──→ 协调全局事务
    ↑
服务A（请假服务）──┐
                  ├── RM（资源管理器）→ 执行分支事务
服务B（BPM服务） ──┘
```

**实现步骤**

1. 引入 Seata 依赖，配置 TC（事务协调器）
2. 每个服务的数据库创建 `undo_log` 表
3. 入口方法加 `@GlobalTransactional`
4. Feign 调用传递 XID

**优缺点**

| 优点 | 缺点 |
|------|------|
| 强一致性，编程模型简单 | 引入额外中间件（Seata Server） |
| 对业务代码侵入小 | 运维复杂度高 |
| AT 模式性能较好 | 锁粒度可能影响并发 |
| 支持回滚 | 需要 undo_log 表 |

**代价评估**：高（基础设施+运维成本，学习曲线陡峭）

---

### 6.4 方案四：事务消息 + RocketMQ/TxMQ

**核心思想**

使用支持事务消息的 MQ（如 RocketMQ）实现"半消息"机制。

**流程**

```
1. 发送"半消息"到 MQ（消息暂存，消费者不可见）
2. 执行本地事务（insert 请假单）
3. 本地事务成功 → 提交半消息 → 消费者可见
4. 本地事务失败 → 回滚半消息 → 消息丢弃
5. MQ 回调检查本地事务状态（兜底）
```

**消费者侧**：
- 消费"启动流程"消息
- 调用 BPM 服务创建流程实例
- 成功后回调更新请假单 `processInstanceId`
- 失败则消息重试

**优缺点**

| 优点 | 缺点 |
|------|------|
| 高吞吐、低延迟 | 需要特定 MQ（RocketMQ 等） |
| 最终一致性可靠 | 实现复杂度高 |
| 天然支持重试和死信 | 异步化，调用方不能立即拿到结果 |
| 解耦服务 | MQ 基础设施维护成本 |

**代价评估**：高（依赖特定 MQ，开发和运维成本高）

---

### 6.5 方案对比总结

| 维度 | 方案一：状态机+定时 | 方案二：本地消息表 | 方案三：Seata AT | 方案四：事务消息 |
|------|-------------------|-------------------|-----------------|-----------------|
| **一致性模型** | 最终一致 | 最终一致 | 强一致（AT） | 最终一致 |
| **适用架构** | 单体/微服务 | 单体/微服务 | 微服务 | 微服务 |
| **开发成本** | 低 | 中 | 中 | 高 |
| **运维成本** | 低 | 低 | 高 | 高 |
| **侵入性** | 低（加字段） | 中（加消息表） | 低（注解） | 中（MQ依赖） |
| **实时性** | 有延迟（分钟级） | 近实时 | 实时 | 近实时 |
| **推荐场景** | 当前单体架构 | 异步化需求 | 强一致要求 | 高吞吐解耦 |

### 6.6 针对当前项目的建议

**当前状态**：单体应用，同模块本地调用，已在同一事务中

**短期建议**（针对问题3/4提到的极端场景）：
- 保持现有实现（同一事务已经保证原子性）
- 增加 `processInstanceId` 字段的非空约束或校验
- 增加操作日志，便于问题排查

**长期建议**（为微服务拆分做准备）：
- 采用 **方案一（状态机+定时补偿）** 作为过渡
- 成本低，侵入性小，可平滑过渡到微服务架构
- 同时为 `businessKey` 建立索引，便于补偿任务快速查询

---

## 七、结论

### 7.1 问题答案汇总

| 问题 | 答案 |
|------|------|
| 1. 三步顺序和数据依赖 | insert(生成id) → createProcessInstance(依赖id生成businessKey) → updateById(依赖id和processInstanceId) |
| 2. @Transactional 覆盖范围 | 单体模式下同事务，覆盖本地和BPM内部；微服务模式下只覆盖本地 |
| 3. processInstanceApi成功但updateById失败 | 单体模式下不会（同事务回滚）；微服务模式下会（需补偿） |
| 4. processInstanceApi抛异常时请假单 | 单体模式下回滚；微服务模式下需看异常传播时机 |
| 5. businessKey的作用 | 关联业务与流程，回调时通过 businessKey=leave.id 定位要更新的请假单 |

### 7.2 核心风险

- **当前单体架构**：基本安全，极端崩溃场景有极小概率不一致
- **未来微服务拆分**：必须引入分布式事务或补偿机制
- **业务设计依赖**：`businessKey` 是整个回调机制的核心，必须保证其正确性和唯一性
