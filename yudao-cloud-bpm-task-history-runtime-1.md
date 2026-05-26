# Yudao-Cloud BPM 任务 Runtime 与 Historic 协作分析

## 一、各接口查询的 Flowable 对象分析

### 1.1 待办任务查询

**接口**：`BpmTaskController.getTaskTodoPage()` (line 63-80)

**查询的 Flowable 对象**：

| 对象类型 | 来源 | 查询方式 |
|---------|------|---------|
| **Task (运行时任务)** | Flowable Runtime | `taskService.createTaskQuery().taskAssignee().active()` |
| **ProcessInstance (运行时流程实例)** | Flowable Runtime | `runtimeService.createProcessInstanceQuery()` |
| **BpmProcessDefinitionInfoDO** | 业务表 | `processDefinitionService.getProcessDefinitionInfoMap()` |

**核心代码**：
- `BpmTaskServiceImpl.getTaskTodoPage()` (line 113-140)：使用 `TaskQuery.active()` 只查活跃任务
- `BpmTaskController` (line 73-78)：补充查询运行时流程实例

---

### 1.2 已办任务查询

**接口**：`BpmTaskController.getTaskDonePage()` (line 82-99)

**查询的 Flowable 对象**：

| 对象类型 | 来源 | 查询方式 |
|---------|------|---------|
| **HistoricTaskInstance (历史任务)** | Flowable History | `historyService.createHistoricTaskInstanceQuery().finished()` |
| **HistoricProcessInstance (历史流程实例)** | Flowable History | `historyService.createHistoricProcessInstanceQuery()` |
| **BpmProcessDefinitionInfoDO** | 业务表 | `processDefinitionService.getProcessDefinitionInfoMap()` |

**核心代码**：
- `BpmTaskServiceImpl.getTaskDonePage()` (line 223-257)：使用 `HistoricTaskInstanceQuery.finished()` 只查已完成任务
- `BpmTaskController` (line 92-98)：补充查询历史流程实例

---

### 1.3 全部任务查询

**接口**：`BpmTaskController.getTaskManagerPage()` (line 101-122)

**查询的 Flowable 对象**：

| 对象类型 | 来源 | 查询方式 |
|---------|------|---------|
| **HistoricTaskInstance (历史任务)** | Flowable History | `historyService.createHistoricTaskInstanceQuery()` (无状态过滤) |
| **HistoricProcessInstance (历史流程实例)** | Flowable History | `historyService.createHistoricProcessInstanceQuery()` |
| **BpmProcessDefinitionInfoDO** | 业务表 | `processDefinitionService.getProcessDefinitionInfoMap()` |

**核心代码**：
- `BpmTaskServiceImpl.getTaskPage()` (line 259-288)：不使用 `finished()` 过滤，可查全部任务
- 注意：Flowable 的 `HistoricTaskInstance` 包含运行中和已完成的所有任务

---

### 1.4 流程详情查询

**接口**：`BpmProcessInstanceController.getProcessInstance()` (line 131-153)

**查询的 Flowable 对象**：

| 对象类型 | 来源 | 查询方式 |
|---------|------|---------|
| **HistoricProcessInstance (历史流程实例)** | Flowable History | `historyService.createHistoricProcessInstanceQuery().processInstanceId()` |
| **ProcessDefinition (流程定义)** | Flowable Repository | `repositoryService.createProcessDefinitionQuery()` |
| **BpmProcessDefinitionInfoDO** | 业务表 | `processDefinitionService.getProcessDefinitionInfo()` |

**核心代码**：
- `BpmProcessInstanceServiceImpl.getHistoricProcessInstance()` (line 141-145)：使用历史查询
- 为什么用 Historic？因为 `HistoricProcessInstance` 包含运行中和已结束的所有流程实例

---

### 1.5 审批详情查询

**接口**：`BpmProcessInstanceController.getApprovalDetail()` (line 173-183)

**查询的 Flowable 对象**：

| 对象类型 | 来源 | 查询方式 |
|---------|------|---------|
| **HistoricProcessInstance** | Flowable History | `historyService.createHistoricProcessInstanceQuery()` |
| **HistoricActivityInstance** | Flowable History | `historyService.createHistoricActivityInstanceQuery()` |
| **HistoricTaskInstance** | Flowable History | `historyService.createHistoricTaskInstanceQuery()` |
| **Task (运行时待办)** | Flowable Runtime | `taskService.createTaskQuery()` |
| **ProcessInstance Variables** | Flowable Runtime | `runtimeService.getVariable()` |
| **ProcessDefinition** | Flowable Repository | `repositoryService.createProcessDefinitionQuery()` |

**核心代码**：
- `BpmProcessInstanceServiceImpl.getApprovalDetail()` (line 169-246)：混合使用历史和运行时对象
- 历史对象用于查询已完成的审批记录
- 运行时对象用于查询当前待办任务和最新流程变量

---

## 二、转换 VO 时补充字段的原因

### 2.1 startUser（发起人信息）

**补充位置**：
- `BpmTaskConvert.buildTodoTaskPage()` (line 55-56)
- `BpmTaskConvert.buildTaskPage()` (line 81-83)
- `BpmProcessInstanceConvert.buildProcessInstancePage()` (line 76-81)

**原因分析**：

1. **Flowable 存储限制**：Flowable 的 `ProcessInstance` 和 `HistoricProcessInstance` 只存储 `startUserId`（字符串形式的用户ID），不存储用户的完整信息（昵称、头像等）

2. **系统解耦**：用户信息存储在 `system` 模块的 `admin_user` 表中，BPM 模块通过 `AdminUserApi` 远程调用获取

3. **展示需求**：前端需要展示发起人的昵称、部门等信息，仅靠 `startUserId` 无法满足

---

### 2.2 dept（部门信息）

**补充位置**：
- `BpmTaskConvert.buildTaskPage()` (line 75-77)
- `BpmProcessInstanceConvert.buildProcessInstancePage()` (line 79-81)

**原因分析**：

1. **Flowable 不存储部门**：Flowable 引擎只关注流程执行，不存储组织架构信息

2. **组织架构独立**：部门信息存储在 `system` 模块的 `system_dept` 表中

3. **审批上下文**：展示审批人所属部门有助于理解审批上下文，特别是在跨部门审批场景

---

### 2.3 summary（流程摘要）

**补充位置**：
- `BpmTaskConvert.buildTodoTaskPage()` (line 58-60)
- `BpmTaskConvert.buildTaskPage()` (line 84-86)
- `BpmProcessInstanceConvert.buildProcessInstancePage()` (line 92-94)

**原因分析**：

1. **自定义配置**：摘要是 yudao-cloud 在 `BpmProcessDefinitionInfoDO` 中扩展的配置项，支持自定义摘要字段

2. **动态计算**：摘要需要结合两部分信息：
   - 流程定义中的摘要配置（`summarySetting`）
   - 流程实例中的变量值（`processVariables`）

3. **FlowableUtils.getSummary()** (line 239-273)：实现了摘要计算逻辑
   - 若配置了自定义摘要字段，使用配置的字段
   - 否则默认取前 3 个表单字段

---

### 2.4 formVariables（表单变量）

**补充位置**：
- `BpmTaskConvert.buildTaskListByProcessInstanceId()` (line 106-110)
- `BpmProcessInstanceConvert.buildProcessInstance()` (line 107-108)
- `BpmProcessInstanceConvert.buildProcessInstancePage()` (line 96)

**原因分析**：

1. **变量过滤**：`processVariables` 和 `taskLocalVariables` 中包含大量系统内部变量（如状态、原因等），需要过滤后才能展示给用户

2. **FlowableUtils.filterProcessInstanceFormVariable()** (line 174-177)：
   - 移除 `PROCESS_INSTANCE_VARIABLE_STATUS` 等系统变量
   - 只保留业务表单字段

3. **FlowableUtils.filterTaskFormVariable()** (line 347-351)：
   - 移除 `TASK_VARIABLE_STATUS`、`TASK_VARIABLE_REASON` 等系统变量

---

## 三、状态与原因依赖字段分析

### 3.1 getTaskStatus 依赖字段

**方法定义**：`FlowableUtils.getTaskStatus()` (line 303-305)

```java
public static Integer getTaskStatus(TaskInfo task) {
    return (Integer) task.getTaskLocalVariables().get(BpmnVariableConstants.TASK_VARIABLE_STATUS);
}
```

**依赖字段**：
- 来源：`task.getTaskLocalVariables()`
- 变量名：`BpmnVariableConstants.TASK_VARIABLE_STATUS` = `"bpm_task_status"`
- 类型：`Integer`

**状态值定义**（`BpmTaskStatusEnum`）：
- `RUNNING` (1)：审批中
- `APPROVE` (2)：审批通过
- `REJECT` (3)：审批不通过
- `CANCEL` (4)：已取消
- `RETURN` (5)：已退回
- `DELEGATE` (6)：委派中
- `TRANSFER` (7)：已转办
- `APPROVING` (8)：审批中（加签场景）
- `SKIP` (9)：已跳过
- `NOT_START` (10)：未开始

**设置位置**：
- `BpmTaskServiceImpl.updateTaskStatus()` (line 848-850)
- 审批通过/拒绝/退回/委派等操作时设置

---

### 3.2 getTaskReason 依赖字段

**方法定义**：`FlowableUtils.getTaskReason()` (line 313-315)

```java
public static String getTaskReason(TaskInfo task) {
    return (String) task.getTaskLocalVariables().get(BpmnVariableConstants.TASK_VARIABLE_REASON);
}
```

**依赖字段**：
- 来源：`task.getTaskLocalVariables()`
- 变量名：`BpmnVariableConstants.TASK_VARIABLE_REASON` = `"bpm_task_reason"`
- 类型：`String`

**设置位置**：
- `BpmTaskServiceImpl.updateTaskStatusAndReason()` (line 859-862)
- 审批通过/拒绝/退回等操作时，用户填写的审批意见

---

### 3.3 getProcessInstanceStatus 依赖字段

**方法定义**：`FlowableUtils.getProcessInstanceStatus()` (line 116-132)

```java
public static Integer getProcessInstanceStatus(ProcessInstance processInstance) {
    return getProcessInstanceStatus(processInstance.getProcessVariables());
}

public static Integer getProcessInstanceStatus(HistoricProcessInstance processInstance) {
    return getProcessInstanceStatus(processInstance.getProcessVariables());
}

private static Integer getProcessInstanceStatus(Map<String, Object> processVariables) {
    return (Integer) processVariables.get(BpmnVariableConstants.PROCESS_INSTANCE_VARIABLE_STATUS);
}
```

**依赖字段**：
- 来源：`processInstance.getProcessVariables()`
- 变量名：`BpmnVariableConstants.PROCESS_INSTANCE_VARIABLE_STATUS` = `"bpm_process_instance_status"`
- 类型：`Integer`

**状态值定义**（`BpmProcessInstanceStatusEnum`）：
- `NOT_START` (-1)：未开始
- `RUNNING` (1)：审批中
- `APPROVE` (2)：审批通过
- `REJECT` (3)：审批不通过
- `CANCEL` (4)：已取消

**设置位置**：
- 发起流程时：`BpmProcessInstanceServiceImpl.createProcessInstance0()` (line 808-809) 设置为 `RUNNING`
- 审批通过时：流程自然结束，状态由 Flowable 维护
- 审批拒绝时：`BpmTaskServiceImpl.rejectTask()` (line 838) 调用 `processInstanceService.updateProcessInstanceReject()`
- 取消流程时：设置为 `CANCEL`

---

## 四、特殊操作时只在历史对象中的信息

### 4.1 流程结束

**信息只在历史对象中的原因**：

1. **运行时对象被删除**：Flowable 在流程结束后，会将 `ProcessInstance` 和 `Task` 从运行时表中删除
2. **历史表保留**：所有信息保留在 `HistoricProcessInstance` 和 `HistoricTaskInstance` 表中

**影响**：
- 已结束的流程只能通过 `HistoryService` 查询
- 不能使用 `RuntimeService` 查询已结束流程的实例和任务

**代码验证**：
- `BpmProcessInstanceServiceImpl.getHistoricProcessInstance()` (line 141-145)：使用历史查询获取所有状态的流程实例

---

### 4.2 流程取消

**只在历史对象中的信息**：

1. **取消状态记录**：
   - 任务状态：`BpmTaskStatusEnum.CANCEL`
   - 流程状态：`BpmProcessInstanceStatusEnum.CANCEL`

2. **取消原因**：存储在流程变量 `PROCESS_INSTANCE_VARIABLE_REASON` 中

**历史任务特征**：
- `HistoricTaskInstance.getEndTime()` 有值
- `taskLocalVariables` 中包含取消状态

**代码验证**：
- `BpmTaskServiceImpl.getFinishedTaskListByProcessInstanceIdWithoutCancel()` (line 503-511)：排除取消状态的任务
- `BpmTaskConvert.buildTaskListByProcessInstanceId()` (line 98-103)：过滤已取消的任务

---

### 4.3 流程退回

**只在历史对象中的信息**：

1. **退回任务记录**：
   - 状态：`BpmTaskStatusEnum.RETURN`
   - 原因：退回理由

2. **被取消的任务**：退回时，当前节点之后的任务会被取消，这些任务只在历史表中

3. **退回评论**：
   - 类型：`BpmCommentTypeEnum.RETURN`
   - 内容：退回理由

**代码验证**：
- `BpmTaskServiceImpl.returnTask()` (line 927-943)：
  - 当前用户任务标记为 `RETURN`
  - 其他节点任务标记为 `CANCEL`
- `BpmTaskServiceImpl.getNeedSimulateTaskDefinitionKeys()` (line 964-989)：使用历史任务计算需要预测的节点

---

### 4.4 任务委派

**只在历史对象中的信息**：

1. **委派评论记录**：
   - `DELEGATE_START`：发起委派
   - `DELEGATE_END`：完成委派

2. **委派状态**：
   - 运行时任务：`DelegationState.PENDING`（委派中）
   - 历史任务：状态变更记录

**代码验证**：
- `BpmTaskServiceImpl.approveDelegateTask()` (line 781-794)：添加委派评论
- `BpmTaskServiceImpl.delegateTask()` (line 991+)：发起委派逻辑

---

### 4.5 任务转办

**只在历史对象中的信息**：

1. **任务分配历史**：原审批人和新审批人的变更记录
2. **转办评论**（如果有实现）

**注意**：转办操作直接修改运行时任务的 `assignee`，历史表中会保留任务的完整生命周期记录

---

### 4.6 任务加签

**只在历史对象中的信息**：

1. **子任务关联**：通过 `parentTaskId` 关联父子任务
2. **加签类型**：`scopeType` 字段标识前后加签
3. **子任务状态历史**：子任务的审批状态记录

**代码验证**：
- `BpmTaskServiceImpl.getAllChildrenTaskListByParentTaskId()` (line 388-418)：递归查询子任务
- `BpmTaskServiceImpl.getTaskListByParentTaskId()` (line 449-454)：原生 SQL 查询子任务（Flowable API 不支持）
- `BpmTaskConvert.copyTo()` (line 208-220)：父任务属性复制到子任务

---

## 五、流程详情接口数据聚合调用链

### 5.1 流程详情接口调用链

**接口入口**：`BpmProcessInstanceController.getProcessInstance()` (line 135)

```
┌─────────────────────────────────────────────────────────────────┐
│ BpmProcessInstanceController.getProcessInstance(id)             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
┌──────────────────────────┐  ┌───────────────────────────────────┐
│ 1. 查询历史流程实例       │  │ 2. 查询流程定义信息               │
│ processInstanceService   │  │ processDefinitionService          │
│ .getHistoricProcessInstance│ │ .getProcessDefinition()          │
│ (id)                     │  │ .getProcessDefinitionInfo()       │
└───────────┬──────────────┘  └───────────────┬───────────────────┘
            │                                 │
            └─────────────────┬───────────────┘
                              ▼
              ┌─────────────────────────────────────────┐
              │ 3. 查询发起人及部门信息                  │
              │ adminUserApi.getUser(startUserId)       │
              │ deptApi.getDept(startUser.deptId)       │
              └─────────────────────┬───────────────────┘
                                    │
                                    ▼
              ┌─────────────────────────────────────────┐
              │ 4. 转换 VO（BpmProcessInstanceConvert） │
              │    buildProcessInstance()               │
              │    ├─ 设置流程状态                      │
              │    │   FlowableUtils.getProcessInstance│
              │    │   Status()                        │
              │    ├─ 设置表单变量                      │
              │    │   FlowableUtils.getProcessInstance│
              │    │   FormVariable()                  │
              │    └─ 组装发起人信息                    │
              └─────────────────────────────────────────┘
```

**详细调用步骤**：

1. **查询历史流程实例** (`BpmProcessInstanceController:136`)
   ```java
   HistoricProcessInstance processInstance = processInstanceService.getHistoricProcessInstance(id);
   ```
   → `BpmProcessInstanceServiceImpl.getHistoricProcessInstance()` (line 141-145)

2. **查询流程定义** (`BpmProcessInstanceController:142`)
   ```java
   ProcessDefinition processDefinition = processDefinitionService.getProcessDefinition(
       processInstance.getProcessDefinitionId());
   ```

3. **查询流程定义扩展信息** (`BpmProcessInstanceController:144`)
   ```java
   BpmProcessDefinitionInfoDO processDefinitionInfo = processDefinitionService.getProcessDefinitionInfo(
       processInstance.getProcessDefinitionId());
   ```

4. **查询发起人信息** (`BpmProcessInstanceController:146`)
   ```java
   AdminUserRespDTO startUser = adminUserApi.getUser(
       NumberUtils.parseLong(processInstance.getStartUserId())).getCheckedData();
   ```

5. **查询部门信息** (`BpmProcessInstanceController:148-150`)
   ```java
   DeptRespDTO dept = deptApi.getDept(startUser.getDeptId()).getCheckedData();
   ```

6. **转换 VO** (`BpmProcessInstanceController:151`)
   ```java
   BpmProcessInstanceConvert.INSTANCE.buildProcessInstance(
       processInstance, processDefinition, processDefinitionInfo, startUser, dept);
   ```

7. **转换内部调用** (`BpmProcessInstanceConvert.buildProcessInstance()` line 101-120)
   - 设置流程状态：`FlowableUtils.getProcessInstanceStatus(processInstance)`
   - 设置表单变量：`FlowableUtils.getProcessInstanceFormVariable(processInstance)`
   - 组装发起人信息

---

### 5.2 审批详情接口调用链

**接口入口**：`BpmProcessInstanceController.getApprovalDetail()` (line 178)

```
┌────────────────────────────────────────────────────────────────────┐
│ BpmProcessInstanceController.getApprovalDetail(reqVO)              │
└────────────────────────────┬───────────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
┌───────────────────────────┐  ┌────────────────────────────────────┐
│ 1. 查询历史流程实例        │  │ 2. 查询流程定义/BpmnModel          │
│ getHistoricProcessInstance│  │ getProcessDefinition()              │
│                           │  │ getProcessDefinitionBpmnModel()     │
└───────────┬───────────────┘  └─────────────────┬──────────────────┘
            │                                    │
            └──────────────────┬─────────────────┘
                               ▼
              ┌────────────────────────────────────────────┐
              │ 3. 查询活动和任务历史                      │
              │ taskService.getActivityListByProcessInst   │
              │ anceId()  ← HistoricActivityInstance       │
              │ taskService.getTaskListByProcessInstanceI  │
              │ d()  ← HistoricTaskInstance                │
              └────────────────────┬───────────────────────┘
                                   │
                                   ▼
              ┌────────────────────────────────────────────┐
              │ 4. 构建已结束/进行中节点                    │
              │ getEndActivityNodeList()                   │
              │ getRunApproveNodeList()                    │
              └────────────────────┬───────────────────────┘
                                   │
                                   ▼
              ┌────────────────────────────────────────────┐
              │ 5. 预测未来节点（如未结束）                 │
              │ getSimulateApproveNodeList()               │
              │ ├─ BPMN 设计器：simulateProcess()          │
              │ └─ SIMPLE 设计器：SimpleModelUtils         │
              └────────────────────┬───────────────────────┘
                                   │
                                   ▼
              ┌────────────────────────────────────────────┐
              │ 6. 查询用户/部门信息并转换 VO              │
              │ parseUserIds()                             │
              │ adminUserApi.getUserMap()                  │
              │ deptApi.getDeptMap()                       │
              │ buildApprovalDetail()                      │
              └────────────────────────────────────────────┘
```

---

## 六、Runtime 与 Historic 混淆导致的错误结论

### 6.1 错误结论 1：查询已结束流程只查 Runtime

**场景**：统计已完成的流程数量时，使用 `RuntimeService.createProcessInstanceQuery()`

**错误结果**：
- 已结束的流程实例从 Runtime 表中被删除
- 统计结果为 0，与实际情况严重不符

**正确做法**：
- 使用 `HistoryService.createHistoricProcessInstanceQuery().finished()`
- 或使用 `HistoricProcessInstanceQuery` 配合状态过滤

**代码验证**：
- `BpmProcessInstanceServiceImpl.getProcessInstancePage()` (line 312-363)：使用 `HistoricProcessInstanceQuery` 支持所有状态查询

---

### 6.2 错误结论 2：查询待办任务查 Historic

**场景**：查询待办任务时，使用 `HistoricTaskInstanceQuery` 且不加 `unfinished()` 过滤

**错误结果**：
- `HistoricTaskInstance` 包含已完成和运行中的所有任务
- 待办列表中会混入已完成的任务
- 用户看到的待办数量远大于实际需要处理的任务

**正确做法**：
- 待办任务：使用 `TaskQuery.active().taskAssignee()`
- 已办任务：使用 `HistoricTaskInstanceQuery.finished().taskAssignee()`

**代码验证**：
- `BpmTaskServiceImpl.getTaskTodoPage()` (line 113-140)：`taskQuery.active()`
- `BpmTaskServiceImpl.getTaskDonePage()` (line 223-257)：`taskQuery.finished()`

---

### 6.3 错误结论 3：用 Historic 获取实时变量

**场景**：审批时需要获取最新流程变量，却从 `HistoricProcessInstance.getProcessVariables()` 读取

**错误结果**：
- `HistoricProcessInstance` 中的变量是快照，不是实时的
- 可能读取到过期的变量值，导致决策错误
- 例如：加签后新增的子任务信息不会反映在历史变量中

**正确做法**：
- 获取实时变量：使用 `RuntimeService.getVariable(processInstanceId, variableName)`
- 获取历史变量：使用 `HistoricProcessInstance.getProcessVariables()`

**代码验证**：
- `BpmProcessInstanceServiceImpl.getApprovalDetail()` (line 230)：使用 `runtimeService.getVariable()` 获取实时变量

---

### 6.4 错误结论 4：用 Runtime Task 查询已完成任务的审批意见

**场景**：查询某个已完成任务的审批意见，使用 `TaskQuery.taskId(taskId)`

**错误结果**：
- 任务完成后，`Task` 对象从 Runtime 表删除
- 查询结果为 null，无法获取审批意见
- 审批状态和原因存储在 `taskLocalVariables` 中，只有历史任务保留

**正确做法**：
- 已完成任务：使用 `HistoryService.createHistoricTaskInstanceQuery().taskId(taskId)`
- 通过 `FlowableUtils.getTaskStatus()` 和 `getTaskReason()` 获取状态和原因

**代码验证**：
- `BpmTaskServiceImpl.getHistoricTask()` (line 337-339)：提供历史任务查询方法
- `BpmTaskConvert.buildTaskListByProcessInstanceId()` (line 93-116)：使用历史任务列表

---

### 6.5 错误结论 5：流程结束后查询 Runtime 变量导致空指针

**场景**：流程结束后，继续使用 `RuntimeService` 查询流程实例或任务

**错误结果**：
- `ProcessInstance` 为 null
- `Task` 列表为空
- 代码中未做 null 检查会导致 NullPointerException

**正确做法**：
- 流程结束后，所有查询应使用 `HistoryService`
- 查询前判断流程是否已结束

**判断流程是否结束**：
```java
HistoricProcessInstance historicPI = historyService.createHistoricProcessInstanceQuery()
    .processInstanceId(id)
    .singleResult();
boolean isEnded = historicPI.getEndTime() != null;
```

**代码验证**：
- `BpmProcessInstanceServiceImpl.getApprovalDetail()` (line 218)：判断 `isProcessEndStatus()` 后决定是否预测未来节点

---

## 七、关键协作关系总结

### 7.1 各组件职责

| 组件 | 主要职责 | 关键方法 |
|------|---------|---------|
| **BpmTaskController** | 任务接口入口 | getTaskTodoPage, getTaskDonePage, getTaskManagerPage |
| **BpmProcessInstanceController** | 流程实例接口入口 | getProcessInstance, getApprovalDetail |
| **BpmTaskServiceImpl** | 任务业务逻辑 | getTaskTodoPage, getTaskDonePage, getTaskPage |
| **BpmProcessInstanceServiceImpl** | 流程实例业务逻辑 | getHistoricProcessInstance, getApprovalDetail |
| **BpmTaskConvert** | 任务 VO 转换 | buildTodoTaskPage, buildTaskPage |
| **BpmProcessInstanceConvert** | 流程实例 VO 转换 | buildProcessInstance, buildApprovalDetail |
| **FlowableUtils** | Flowable 工具方法 | getTaskStatus, getTaskReason, getProcessInstanceStatus, getSummary |

### 7.2 Runtime 与 Historic 的使用原则

| 场景 | 应使用 | 不应使用 | 原因 |
|------|-------|---------|------|
| 查询待办任务 | Task (Runtime) | HistoricTaskInstance | 待办只存在于 Runtime |
| 查询已办任务 | HistoricTaskInstance | Task (Runtime) | 已完成任务已从 Runtime 删除 |
| 查询所有任务 | HistoricTaskInstance | Task (Runtime) | Task 只包含运行中 |
| 查询运行中流程 | ProcessInstance + HistoricProcessInstance | 仅 ProcessInstance | ProcessInstance 不包含已结束 |
| 查询历史流程 | HistoricProcessInstance | ProcessInstance | ProcessInstance 不包含已结束 |
| 获取实时变量 | RuntimeService.getVariable() | HistoricProcessInstance.getVariables() | 历史变量是快照 |
| 获取历史变量 | HistoricProcessInstance.getVariables() | - | 流程结束后 Runtime 无数据 |
