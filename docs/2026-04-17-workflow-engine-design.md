# 工作流引擎设计文档

> 创建时间：2026-04-17
> 状态：已评审通过

---

## 一、项目概述

### 1.1 目标

在现有 `storeability` 微服务体系中，设计并实现一套通用的业务流程引擎，支持企业内部审批流、OA 流程等场景。

### 1.2 技术约束（遵循 project.md）

| 约束 | 值 |
|------|------|
| Java 版本 | 1.8（禁止 Java 8+ API） |
| Spring Boot | 2.3.7 |
| Dubbo | 2.6.7 |
| MyBatis-Plus | 3.4.0 |
| 数据库 | MySQL 8.0.17 |
| 消息队列 | RocketMQ |
| 分布式锁 | Redis |
| 所属模块 | storeability-storelife / storeability-storelife-api |
| 包路径 | com.tmallyc.storeability.storelife.workflow |

### 1.3 核心能力

- **流程定义**：通过 JSON 格式定义流程模板（节点、任务、转移条件）
- **审批功能**：多级审批、会签、或签、加签、转签
- **节点回滚**：支持退回上一节点、退回指定节点
- **通知集成**：通过 RocketMQ 异步推送待办/催办通知
- **权限体系**：内置简单用户-角色-部门模型，通过防腐层对接已有用户服务
- **流程监控**：流程实例状态查询、超时预警、流程统计

### 1.4 架构方案

**状态机驱动**：每个流程实例是一个有限状态机，节点是状态，审批/填写动作触发状态转移。

---

## 二、整体架构

### 2.1 分层架构

```
┌──────────────────────────────────────────────────────────────┐
│  storeability-storelife-api（Dubbo Facade 接口 + PO/RO）     │
│  storelife.api.workflow/                                     │
│    ├── ProcessDefinitionFacade        流程定义管理            │
│    ├── ProcessInstanceFacade          流程实例管理            │
│    ├── TaskFacade                     任务操作                │
│    ├── MonitorFacade                  流程监控                │
│    ├── po/                            入参对象                │
│    └── ro/                            出参对象                │
├──────────────────────────────────────────────────────────────┤
│  storeability-storelife（实现模块）                           │
│  storelife.workflow/                                         │
│    ├── engine/                        状态机引擎核心          │
│    │   ├── StateMachineEngine             引擎入口            │
│    │   ├── NodeExecutor                   节点执行器          │
│    │   ├── TransitionEvaluator            转移条件评估        │
│    │   └── AssigneeResolver               审批人解析          │
│    ├── service/                       业务服务层              │
│    │   ├── ProcessDefinitionService       流程定义服务        │
│    │   ├── ProcessInstanceService         流程实例服务        │
│    │   ├── TaskService                    任务服务            │
│    │   ├── MonitorService                 监控服务            │
│    │   └── converter/                     BO↔DO 转换          │
│    ├── repository/                    数据访问层              │
│    │   ├── dos/                            DO 对象            │
│    │   ├── mapper/                         MyBatis Mapper     │
│    │   └── query/                          查询条件           │
│    ├── facade/                        Facade 实现             │
│    │   ├── assembler/                     PO/RO↔BO 转换      │
│    │   └── *FacadeImpl.java               Facade 实现类      │
│    ├── event/                         事件定义与生产者        │
│    │   ├── WorkflowEvent.java             事件基类            │
│    │   └── WorkflowEventPublisher.java    RocketMQ 生产者    │
│    └── listener/                      RocketMQ 消费者         │
│        ├── NotificationListener.java      通知消费            │
│        └── AuditLogListener.java          审计日志消费        │
├──────────────────────────────────────────────────────────────┤
│  storeability-common                                         │
│    ├── constant/WorkflowEnum.java     流程相关枚举            │
│    └── constant/WorkflowConst.java    流程相关常量            │
├──────────────────────────────────────────────────────────────┤
│  storeability-integration                                    │
│    └── 外部用户/部门/角色服务调用（已有服务的防腐层）          │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 事件流（全部走 RocketMQ）

```
引擎状态流转 → WorkflowEventPublisher → RocketMQ
                                           ├── Topic: workflow-notification → 待办/通知推送
                                           ├── Topic: workflow-audit       → 审计日志记录
                                           └── Topic: workflow-timeout     → 超时预警检测
```

### 2.3 与外部系统交互

| 交互方 | 方式 | 说明 |
|--------|------|------|
| 前端/其他微服务 | Dubbo RPC | 通过 Facade 接口调用 |
| 用户/角色/部门服务 | Dubbo RPC（防腐层） | 通过 integration 模块调用已有服务 |
| 通知服务 | RocketMQ | 发送待办消息，由通知服务消费 |
| 审计日志 | RocketMQ | 记录流程操作日志 |

---

## 三、核心领域模型

### 3.1 概念关系

```
定义态（模板）                          运行态（实例）
ProcessDefinition ──1:N──────────── ProcessInstance
  └── nodes (JSON内嵌)                └── 1:N → NodeInstance
        └── tasks (JSON内嵌)                     └── 1:N → TaskInstance
              ├── businessParams                       ├── businessData
              └── approvalParams                       └── approvalData
```

**核心规则**：
- 一个流程包含多个有序节点
- 一个节点下包含多个任务，任务有两种执行模式：
  - **PARALLEL（并行）**：所有任务同时激活，全部完成后节点才完成（如：招商端和加盟商各自补充信息）
  - **SERIAL（串行）**：任务按 sortOrder 顺序依次执行，前一个完成才激活下一个（如：区域经理面试 → 大区经理面试）
- 节点下所有任务完成后，引擎才推进到下一个节点
- 每个任务有独立的业务参数（businessData）和审批参数（approvalData）

### 3.2 节点类型

| 节点类型 | 枚举值 | 说明 |
|---------|--------|------|
| 开始节点 | START | 流程起点 |
| 结束节点 | END | 流程终点 |
| 审批节点 | APPROVAL | 单人/多人审批 |
| 填写节点 | FILL | 指定角色填写/补充信息 |
| 会签节点 | COUNTERSIGN | 多人同时审批，全部通过才通过 |
| 或签节点 | OR_SIGN | 多人同时审批，任一通过即通过 |
| 并行节点 | PARALLEL | 多个任务并行执行（不同角色各做各的） |
| 条件节点 | CONDITION | 根据条件选择分支 |
| 等待节点 | WAIT | 等待外部事件/条件满足 |

---

## 四、数据库设计

### 4.1 wf_process_definition — 流程定义表

```sql
CREATE TABLE wf_process_definition (
    id              BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    process_key     VARCHAR(64)     NOT NULL COMMENT '流程唯一标识',
    process_name    VARCHAR(128)    NOT NULL COMMENT '流程名称',
    version         INT             NOT NULL DEFAULT 1 COMMENT '版本号',
    status          TINYINT         NOT NULL DEFAULT 0 COMMENT '状态：0-草稿 1-已发布 2-已停用',
    definition_json TEXT            NOT NULL COMMENT '流程定义JSON',
    remark          VARCHAR(512)    DEFAULT NULL COMMENT '备注',
    creator         VARCHAR(64)     NOT NULL COMMENT '创建人',
    gmt_create      DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    modifier        VARCHAR(64)     DEFAULT NULL COMMENT '修改人',
    gmt_modified    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    is_deleted      TINYINT         NOT NULL DEFAULT 0 COMMENT '逻辑删除',
    PRIMARY KEY (id),
    UNIQUE KEY uk_key_version (process_key, version, is_deleted)
) COMMENT '流程定义表';
```

### 4.2 wf_process_instance — 流程实例表

```sql
CREATE TABLE wf_process_instance (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    process_key         VARCHAR(64)     NOT NULL COMMENT '流程标识',
    definition_id       BIGINT          NOT NULL COMMENT '流程定义ID',
    business_key        VARCHAR(128)    NOT NULL COMMENT '业务主键',
    business_type       VARCHAR(64)     DEFAULT NULL COMMENT '业务类型',
    title               VARCHAR(256)    DEFAULT NULL COMMENT '流程标题',
    status              TINYINT         NOT NULL DEFAULT 0 COMMENT '状态：0-进行中 1-已完成 2-已拒绝 3-已撤销 4-已挂起',
    current_node_id     VARCHAR(64)     DEFAULT NULL COMMENT '当前节点ID',
    initiator           VARCHAR(64)     NOT NULL COMMENT '发起人',
    variables           TEXT            DEFAULT NULL COMMENT '流程变量JSON',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    gmt_modified        DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    gmt_end             DATETIME        DEFAULT NULL COMMENT '结束时间',
    is_deleted          TINYINT         NOT NULL DEFAULT 0 COMMENT '逻辑删除',
    PRIMARY KEY (id),
    KEY idx_process_key (process_key),
    KEY idx_business_key (business_key),
    KEY idx_initiator (initiator),
    KEY idx_status (status)
) COMMENT '流程实例表';
```

### 4.3 wf_node_instance — 节点实例表

```sql
CREATE TABLE wf_node_instance (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    instance_id         BIGINT          NOT NULL COMMENT '流程实例ID',
    node_id             VARCHAR(64)     NOT NULL COMMENT '节点定义ID',
    node_name           VARCHAR(128)    DEFAULT NULL COMMENT '节点名称',
    node_type           VARCHAR(32)     NOT NULL COMMENT '节点类型',
    status              TINYINT         NOT NULL DEFAULT 0 COMMENT '状态：0-待执行 1-执行中 2-已完成 3-已跳过 4-已拒绝 5-已退回',
    execution_round     INT             NOT NULL DEFAULT 1 COMMENT '执行轮次（退回后+1）',
    rollback_from_node  VARCHAR(64)     DEFAULT NULL COMMENT '从哪个节点退回而来',
    operator            VARCHAR(64)     DEFAULT NULL COMMENT '操作人',
    remark              VARCHAR(512)    DEFAULT NULL COMMENT '审批意见',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    gmt_modified        DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    gmt_end             DATETIME        DEFAULT NULL COMMENT '完成时间',
    PRIMARY KEY (id),
    KEY idx_instance_id (instance_id),
    KEY idx_node_id (node_id)
) COMMENT '节点实例表';
```

### 4.4 wf_task_instance — 任务实例表

```sql
CREATE TABLE wf_task_instance (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    instance_id         BIGINT          NOT NULL COMMENT '流程实例ID',
    node_instance_id    BIGINT          NOT NULL COMMENT '节点实例ID',
    task_def_id         VARCHAR(64)     NOT NULL COMMENT '任务定义ID',
    task_name           VARCHAR(128)    DEFAULT NULL COMMENT '任务名称',
    task_type           VARCHAR(32)     NOT NULL COMMENT '任务类型：APPROVAL/FILL',
    assignee            VARCHAR(64)     NOT NULL COMMENT '处理人',
    assignee_role       VARCHAR(64)     DEFAULT NULL COMMENT '处理人角色',
    status              TINYINT         NOT NULL DEFAULT 0 COMMENT '状态：0-待处理 1-已完成 2-已拒绝 3-已转签 4-已撤回 5-已退回 6-待激活',
    sort_order          INT             NOT NULL DEFAULT 1 COMMENT '任务排序（串行模式下决定执行顺序）',
    business_data       TEXT            DEFAULT NULL COMMENT '业务参数JSON',
    approval_data       TEXT            DEFAULT NULL COMMENT '审批参数JSON',
    remark              VARCHAR(512)    DEFAULT NULL COMMENT '处理备注',
    due_time            DATETIME        DEFAULT NULL COMMENT '截止时间',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    gmt_modified        DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    gmt_end             DATETIME        DEFAULT NULL COMMENT '完成时间',
    PRIMARY KEY (id),
    KEY idx_instance_id (instance_id),
    KEY idx_node_instance_id (node_instance_id),
    KEY idx_assignee (assignee),
    KEY idx_status (status)
) COMMENT '任务实例表';
```

### 4.5 wf_operation_log — 操作日志表

```sql
CREATE TABLE wf_operation_log (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    instance_id         BIGINT          NOT NULL COMMENT '流程实例ID',
    node_instance_id    BIGINT          DEFAULT NULL COMMENT '节点实例ID',
    task_id             BIGINT          DEFAULT NULL COMMENT '任务ID',
    operation_type      VARCHAR(32)     NOT NULL COMMENT '操作类型：START/APPROVE/REJECT/TRANSFER/WITHDRAW/ROLLBACK/COUNTERSIGN',
    operator            VARCHAR(64)     NOT NULL COMMENT '操作人',
    remark              VARCHAR(512)    DEFAULT NULL COMMENT '操作备注',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_instance_id (instance_id)
) COMMENT '流程操作日志表';
```

---

## 五、流程定义 JSON 格式

### 5.1 完整示例（加盟签约流程）

```json
{
  "processKey": "franchise_signing",
  "processName": "加盟签约流程",
  "nodes": [
    {
      "nodeId": "start",
      "nodeName": "开始",
      "nodeType": "START",
      "sortOrder": 0,
      "transitions": [{"targetNodeId": "submit_application"}]
    },
    {
      "nodeId": "submit_application",
      "nodeName": "提交申请",
      "sortOrder": 1,
      "nodeType": "APPROVAL",
      "tasks": [
        {
          "taskId": "submit_application_oa",
          "taskName": "提交加盟申请",
          "taskType": "APPROVAL",
          "assignee": {"type": "ROLE", "value": "招商端"},
          "businessParams": [
            {"key": "applicantName", "name": "申请人姓名", "type": "STRING", "required": true},
            {"key": "applicantPhone", "name": "联系电话", "type": "STRING", "required": true},
            {"key": "intendedCity", "name": "意向城市", "type": "STRING", "required": true}
          ],
          "approvalParams": [
            {"key": "result", "name": "审批结果", "type": "ENUM", "options": ["通过", "拒绝"]},
            {"key": "opinion", "name": "审批意见", "type": "TEXT"}
          ]
        }
      ],
      "transitions": [{"targetNodeId": "pay_fee"}]
    },
    {
      "nodeId": "pay_fee",
      "nodeName": "缴纳费用",
      "sortOrder": 2,
      "nodeType": "CONDITION",
      "tasks": [
        {
          "taskId": "pay_fee_task",
          "taskName": "缴费",
          "taskType": "FILL",
          "assignee": {"type": "ROLE", "value": "加盟商"},
          "businessParams": [
            {"key": "payType", "name": "缴费方式", "type": "ENUM", "options": ["全款缴纳", "分期"]},
            {"key": "amount", "name": "缴费金额", "type": "DECIMAL", "required": true}
          ],
          "approvalParams": []
        }
      ],
      "transitions": [
        {"targetNodeId": "lock_area", "condition": "payType == '全款缴纳'"},
        {"targetNodeId": "study_club", "condition": "payType != '全款缴纳'"}
      ]
    },
    {
      "nodeId": "lock_area",
      "nodeName": "锁定商圈",
      "sortOrder": 3,
      "nodeType": "FILL",
      "tasks": [
        {
          "taskId": "lock_area_task",
          "taskName": "锁定商圈",
          "taskType": "FILL",
          "assignee": {"type": "ROLE", "value": "加盟商"},
          "businessParams": [
            {"key": "areaName", "name": "商圈名称", "type": "STRING", "required": true}
          ],
          "approvalParams": []
        }
      ],
      "transitions": [{"targetNodeId": "supplement_info"}]
    },
    {
      "nodeId": "supplement_info",
      "nodeName": "补充信息",
      "sortOrder": 4,
      "nodeType": "PARALLEL",
      "taskExecutionMode": "PARALLEL",
      "tasks": [
        {
          "taskId": "fill_background",
          "taskName": "补充背景信息",
          "taskType": "FILL",
          "sortOrder": 1,
          "assignee": {"type": "ROLE", "value": "招商端"},
          "businessParams": [
            {"key": "backgroundInfo", "name": "背景资料", "type": "TEXT", "required": true},
            {"key": "attachments", "name": "附件", "type": "FILE_LIST"}
          ],
          "approvalParams": []
        },
        {
          "taskId": "fill_property",
          "taskName": "补充物业信息",
          "taskType": "FILL",
          "sortOrder": 2,
          "assignee": {"type": "ROLE", "value": "加盟商"},
          "businessParams": [
            {"key": "propertyAddress", "name": "物业地址", "type": "STRING", "required": true},
            {"key": "propertyArea", "name": "面积(㎡)", "type": "DECIMAL"},
            {"key": "leaseContract", "name": "租赁合同", "type": "FILE"}
          ],
          "approvalParams": []
        }
      ],
      "transitions": [{"targetNodeId": "review_info"}]
    },
    {
      "nodeId": "review_info",
      "nodeName": "审核补充信息",
      "sortOrder": 5,
      "nodeType": "APPROVAL",
      "tasks": [
        {
          "taskId": "review_info_task",
          "taskName": "审核补充信息",
          "taskType": "APPROVAL",
          "assignee": {"type": "ROLE", "value": "招商端"},
          "businessParams": [],
          "approvalParams": [
            {"key": "result", "name": "审核结果", "type": "ENUM", "options": ["通过", "拒绝"]},
            {"key": "opinion", "name": "审核意见", "type": "TEXT"}
          ]
        }
      ],
      "transitions": [{"targetNodeId": "region_manager_interview"}]
    },
    {
      "nodeId": "interview",
      "nodeName": "加盟面试",
      "sortOrder": 6,
      "nodeType": "APPROVAL",
      "taskExecutionMode": "SERIAL",
      "tasks": [
        {
          "taskId": "region_interview_task",
          "taskName": "区域经理面试",
          "taskType": "APPROVAL",
          "sortOrder": 1,
          "assignee": {"type": "ROLE", "value": "区域经理"},
          "businessParams": [
            {"key": "interviewScore", "name": "面试评分", "type": "INTEGER"},
            {"key": "interviewRecord", "name": "面试记录", "type": "TEXT"}
          ],
          "approvalParams": [
            {"key": "result", "name": "面试结果", "type": "ENUM", "options": ["通过", "待定", "拒绝"]},
            {"key": "opinion", "name": "面试意见", "type": "TEXT"}
          ]
        },
        {
          "taskId": "district_interview_task",
          "taskName": "大区经理面试",
          "taskType": "APPROVAL",
          "sortOrder": 2,
          "assignee": {"type": "ROLE", "value": "大区经理"},
          "businessParams": [
            {"key": "interviewScore", "name": "面试评分", "type": "INTEGER"},
            {"key": "interviewRecord", "name": "面试记录", "type": "TEXT"}
          ],
          "approvalParams": [
            {"key": "result", "name": "面试结果", "type": "ENUM", "options": ["通过", "拒绝"]},
            {"key": "opinion", "name": "面试意见", "type": "TEXT"}
          ]
        }
      ],
      "transitions": [{"targetNodeId": "franchise_approval"}]
    },
    {
      "nodeId": "franchise_approval",
      "nodeName": "加盟审批",
      "sortOrder": 7,
      "nodeType": "APPROVAL",
      "tasks": [
        {
          "taskId": "franchise_approval_task",
          "taskName": "加盟审批",
          "taskType": "APPROVAL",
          "assignee": {"type": "ROLE", "value": "审批人"},
          "businessParams": [],
          "approvalParams": [
            {"key": "result", "name": "审批结果", "type": "ENUM", "options": ["通过", "拒绝"]},
            {"key": "opinion", "name": "审批意见", "type": "TEXT"}
          ]
        }
      ],
      "transitions": [{"targetNodeId": "submit_contract_material"}]
    },
    {
      "nodeId": "submit_contract_material",
      "nodeName": "提交签约资料",
      "sortOrder": 8,
      "nodeType": "FILL",
      "tasks": [
        {
          "taskId": "submit_material_task",
          "taskName": "提交签约资料",
          "taskType": "FILL",
          "assignee": {"type": "ROLE", "value": "加盟商"},
          "businessParams": [
            {"key": "contractMaterial", "name": "签约资料", "type": "FILE_LIST", "required": true},
            {"key": "leaseContract", "name": "房屋租赁合同", "type": "FILE", "required": true}
          ],
          "approvalParams": [],
          "validation": {"rule": "房屋租赁合同已上传", "expression": "leaseContract != null"}
        }
      ],
      "transitions": [{"targetNodeId": "material_review"}]
    },
    {
      "nodeId": "material_review",
      "nodeName": "资料审核",
      "sortOrder": 9,
      "nodeType": "APPROVAL",
      "tasks": [
        {
          "taskId": "material_review_task",
          "taskName": "资料审核",
          "taskType": "APPROVAL",
          "assignee": {"type": "ROLE", "value": "招商端"},
          "businessParams": [],
          "approvalParams": [
            {"key": "result", "name": "审核结果", "type": "ENUM", "options": ["通过", "拒绝"]},
            {"key": "opinion", "name": "审核意见", "type": "TEXT"}
          ]
        }
      ],
      "transitions": [{"targetNodeId": "contract_signing"}]
    },
    {
      "nodeId": "contract_signing",
      "nodeName": "合同签约",
      "sortOrder": 10,
      "nodeType": "FILL",
      "tasks": [
        {
          "taskId": "contract_signing_task",
          "taskName": "合同签约",
          "taskType": "FILL",
          "assignee": {"type": "ROLE", "value": "招商端"},
          "businessParams": [
            {"key": "contractNo", "name": "合同编号", "type": "STRING", "required": true},
            {"key": "contractFile", "name": "合同文件", "type": "FILE", "required": true}
          ],
          "approvalParams": []
        }
      ],
      "transitions": [{"targetNodeId": "end"}]
    },
    {
      "nodeId": "end",
      "nodeName": "结束",
      "sortOrder": 11,
      "nodeType": "END"
    }
  ]
}
```

---

## 六、Dubbo Facade 接口设计

### 6.1 ProcessDefinitionFacade — 流程定义管理

```java
public interface ProcessDefinitionFacade {
    Result<Long> createDefinition(CreateProcessDefinitionPO po);
    Result<Boolean> updateDefinition(UpdateProcessDefinitionPO po);
    Result<Boolean> publishDefinition(PublishProcessDefinitionPO po);
    Result<Boolean> disableDefinition(DisableProcessDefinitionPO po);
    Result<ProcessDefinitionRO> getDefinition(GetProcessDefinitionPO po);
    Result<PageRO<ProcessDefinitionRO>> pageQueryDefinitions(QueryProcessDefinitionPO po);
}
```

### 6.2 ProcessInstanceFacade — 流程实例管理

```java
public interface ProcessInstanceFacade {
    Result<Long> startProcess(StartProcessPO po);
    Result<Boolean> withdrawProcess(WithdrawProcessPO po);
    Result<Boolean> suspendProcess(SuspendProcessPO po);
    Result<Boolean> resumeProcess(ResumeProcessPO po);
    Result<ProcessInstanceDetailRO> getProcessDetail(GetProcessDetailPO po);
    Result<PageRO<ProcessInstanceRO>> pageQueryMyInitiated(QueryMyInitiatedPO po);
    Result<List<OperationLogRO>> listOperationLogs(ListOperationLogPO po);
}
```

### 6.3 TaskFacade — 任务操作

```java
public interface TaskFacade {
    Result<Boolean> approve(ApproveTaskPO po);
    Result<Boolean> reject(RejectTaskPO po);
    Result<Boolean> submitTask(SubmitTaskPO po);
    Result<Boolean> transferTask(TransferTaskPO po);
    Result<Boolean> addCountersign(AddCountersignPO po);
    Result<Boolean> rollbackToPrevious(RollbackTaskPO po);
    Result<Boolean> rollbackToNode(RollbackToNodePO po);
    Result<List<RollbackableNodeRO>> listRollbackableNodes(ListRollbackableNodePO po);
    Result<PageRO<TaskInstanceRO>> pageQueryMyTodoTasks(QueryMyTodoTaskPO po);
    Result<PageRO<TaskInstanceRO>> pageQueryMyDoneTasks(QueryMyDoneTaskPO po);
    Result<TaskInstanceDetailRO> getTaskDetail(GetTaskDetailPO po);
}
```

### 6.4 MonitorFacade — 流程监控

```java
public interface MonitorFacade {
    Result<ProcessStatisticsRO> getProcessStatistics(ProcessStatisticsPO po);
    Result<PageRO<TimeoutTaskRO>> pageQueryTimeoutTasks(QueryTimeoutTaskPO po);
    Result<List<NodeTraceRO>> listNodeTrace(ListNodeTracePO po);
}
```

### 6.5 核心 PO/RO

#### StartProcessPO

```java
@Data
public class StartProcessPO implements Serializable {
    private String processKey;
    private String businessKey;
    private String businessType;
    private String title;
    private String initiator;
    private Map<String, Object> variables;
}
```

#### ApproveTaskPO

```java
@Data
public class ApproveTaskPO implements Serializable {
    private Long taskId;
    private String operator;
    private Map<String, Object> businessData;
    private Map<String, Object> approvalData;
    private String remark;
}
```

#### TaskInstanceDetailRO

```java
@Data
public class TaskInstanceDetailRO implements Serializable {
    private Long taskId;
    private Long instanceId;
    private Long nodeInstanceId;
    private String taskDefId;
    private String taskName;
    private String taskType;
    private String assignee;
    private String assigneeRole;
    private Integer status;
    private Map<String, Object> businessData;
    private Map<String, Object> approvalData;
    private List<ParamDefinitionRO> businessParamDefinitions;
    private List<ParamDefinitionRO> approvalParamDefinitions;
    private String remark;
    private Date dueTime;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

---

## 七、引擎核心流转逻辑

### 7.1 StateMachineEngine 核心方法

#### startProcess — 发起流程

```
1. 查找已发布的最新版 ProcessDefinition
2. 解析 definitionJson → 内存中的节点图
3. 创建 ProcessInstance (status=RUNNING)
4. 找到第一个非 START 节点
5. 调用 enterNode(firstNode)
```

#### enterNode — 进入节点

```
1. 创建 NodeInstance (status=RUNNING)
2. 如果是 CONDITION 节点:
   → TransitionEvaluator 评估条件
   → 直接跳转到目标节点 (不创建任务)
3. 如果是 END 节点:
   → 标记 ProcessInstance.status = COMPLETED
   → 发送 PROCESS_COMPLETED 事件
4. 其他节点:
   → 根据 tasks 定义创建 TaskInstance 列表
   → 判断 taskExecutionMode:
     a. PARALLEL（默认）: 所有 TaskInstance 状态设为"待处理"，同时激活
     b. SERIAL: 仅第一个 TaskInstance（sortOrder 最小）状态设为"待处理"，
        其余设为"待激活"(status=6)
   → AssigneeResolver 解析已激活任务的处理人
   → 发送 TASK_CREATED 事件 (仅已激活的任务)
```

#### completeTask — 完成任务

```
1. 校验任务状态 (必须是待处理)
2. 校验操作人 (必须是 assignee)
3. 执行参数校验 (validation rules)
4. 更新 TaskInstance (status, businessData, approvalData)
5. 记录操作日志
6. 调用 checkNodeCompletion(nodeInstanceId)
```

#### checkNodeCompletion — 检查节点完成

```
1. 查询该节点下所有 TaskInstance
2. 判断 taskExecutionMode:
   a. PARALLEL 模式:
      IF 全部 status == COMPLETED:
        → 标记 NodeInstance.status = COMPLETED → 推进到下一节点
      IF 存在 status == REJECTED:
        → 根据节点配置决定: 整个流程拒绝 or 仅标记该任务
      ELSE: 等待其他任务完成
   b. SERIAL 模式:
      IF 当前任务 status == COMPLETED:
        → 查找下一个待激活任务（sortOrder 更大的）
        → IF 存在下一个任务:
            激活下一个 TaskInstance (status → 待处理)
            AssigneeResolver 解析处理人
            发送 TASK_CREATED 事件
        → IF 不存在（已是最后一个任务）:
            标记 NodeInstance.status = COMPLETED → 推进到下一节点
      IF 当前任务 status == REJECTED:
        → 根据节点配置决定: 整个流程拒绝 or 仅标记该任务
3. 推进到下一节点时:
   a. 汇总所有任务的 businessData + approvalData
   b. 合并到 ProcessInstance.variables
   c. TransitionEvaluator 评估转移条件
   d. 确定 targetNodeId
   e. 调用 enterNode(targetNode)
```

#### rollbackToNode — 退回到指定节点

```
1. 校验 targetNodeId 是否在已执行节点列表中
2. 校验 targetNodeId 不是 START/END/CONDITION
3. 标记当前 TaskInstance.status = ROLLBACK
4. 标记当前 NodeInstance.status = ROLLBACK
5. 标记中间节点的 NodeInstance.status = SKIPPED
6. 更新 ProcessInstance.current_node_id = targetNodeId
7. 调用 enterNode(targetNode) (execution_round + 1)
8. 发送 TASK_ROLLBACK 事件
```

### 7.2 TransitionEvaluator — 条件评估器

```
支持的表达式语法:
  - 等值判断:  payType == '全款缴纳'
  - 不等判断:  result != '拒绝'
  - 空值判断:  leaseContract != null
  - 逻辑组合:  result == '通过' && score >= 60
  - 默认转移:  condition 为空表示默认路径（兜底）

实现方式: 自研简单规则解析器（字符串分割 + 反射取值）
```

### 7.3 AssigneeResolver — 审批人解析器

```
assignee 配置格式:
  {"type": "USER",  "value": "zhangsan"}       → 直接指定用户
  {"type": "ROLE",  "value": "区域经理"}        → 按角色查找
  {"type": "INITIATOR"}                         → 流程发起人

通过 storeability-integration 防腐层调用已有的用户-角色-部门服务
```

---

## 八、并发安全

使用 Redis 分布式锁 + 数据库乐观锁双重保障。

| 场景 | Redis 锁 Key | 说明 |
|------|-------------|------|
| 同一任务被重复提交 | `wf:task:lock:{taskId}` | 防止同一任务被并发操作 |
| 同一节点多任务并发完成 | `wf:node:lock:{nodeInstanceId}` | 保证 checkNodeCompletion 串行执行 |
| 流程实例级操作互斥 | `wf:instance:lock:{instanceId}` | 撤销、挂起等操作互斥 |

乐观锁兜底：`UPDATE ... WHERE id = ? AND status = 原状态`，更新行数为 0 则抛出异常。

锁超时时间：30 秒。

---

## 九、事务边界

```
@Transactional 范围:
  1. 更新 TaskInstance
  2. 检查节点完成 + 更新 NodeInstance
  3. 创建下一个 NodeInstance + TaskInstance
  4. 更新 ProcessInstance
  5. 写入 OperationLog

事务提交后 (TransactionSynchronizationManager afterCommit):
  6. 发送 RocketMQ 事件
```

---

## 十、RocketMQ 事件设计

| Topic | Tag | 触发时机 | 消费方 |
|-------|-----|---------|--------|
| workflow-event | TASK_CREATED | 新任务创建时 | 通知服务 |
| workflow-event | TASK_COMPLETED | 任务完成时 | 通知服务 |
| workflow-event | TASK_ROLLBACK | 任务被退回时 | 通知服务 |
| workflow-event | TASK_TIMEOUT | 任务超时时 | 通知服务 |
| workflow-event | PROCESS_COMPLETED | 流程结束时 | 通知服务 + 业务系统 |
| workflow-event | PROCESS_REJECTED | 流程被拒绝时 | 通知服务 + 业务系统 |
| workflow-audit | OPERATION_LOG | 任何操作发生时 | 审计日志服务 |

---

## 十一、错误处理

| 错误类型 | 处理方式 |
|---------|---------|
| 流程定义不存在/未发布 | WorkflowException(DEFINITION_NOT_FOUND) |
| 任务不存在/状态不对 | WorkflowException(TASK_STATUS_INVALID) |
| 操作人无权限 | WorkflowException(OPERATOR_NOT_AUTHORIZED) |
| 参数校验失败 | WorkflowException(PARAM_VALIDATION_FAILED) |
| 条件评估无匹配 | 走默认转移路径；若无默认路径则 WorkflowException(NO_MATCHING_TRANSITION) |
| 退回目标节点非法 | WorkflowException(INVALID_ROLLBACK_TARGET) |
| RocketMQ 发送失败 | 记录错误日志，不影响主流程 |
| 数据库操作失败 | Spring 事务回滚 |

---

## 十二、超时预警

```
WorkflowTimeoutJob (定时任务，每5分钟执行):
  1. 查询所有 status=待处理 且 due_time < NOW() 的 TaskInstance
  2. 对每个超时任务发送 RocketMQ TASK_TIMEOUT 事件
  3. 通知服务消费后推送催办消息
```

---

## 十三、节点回滚规则

| 规则 | 说明 |
|------|------|
| 只能退回已执行过的节点 | 不能退回到从未执行过的节点 |
| 退回后中间节点全部作废 | 如从节点5退回到节点2，节点3、4的实例标记为"已跳过" |
| 退回不影响历史记录 | 所有历史 NodeInstance/TaskInstance 保留，新建新的实例 |
| 条件节点不可作为退回目标 | CONDITION 类型节点是自动判断的 |
| START/END 节点不可退回 | 起止节点不参与退回 |

---

## 十四、代码组织（遵循 project.md）

### api 模块（storeability-storelife-api）

```
storelife/api/workflow/
├── po/
│   ├── CreateProcessDefinitionPO.java
│   ├── StartProcessPO.java
│   ├── ApproveTaskPO.java
│   ├── RejectTaskPO.java
│   ├── SubmitTaskPO.java
│   ├── TransferTaskPO.java
│   ├── AddCountersignPO.java
│   ├── RollbackTaskPO.java
│   ├── RollbackToNodePO.java
│   └── ... (其他 PO)
├── ro/
│   ├── ProcessDefinitionRO.java
│   ├── ProcessInstanceRO.java
│   ├── ProcessInstanceDetailRO.java
│   ├── TaskInstanceRO.java
│   ├── TaskInstanceDetailRO.java
│   ├── ParamDefinitionRO.java
│   ├── RollbackableNodeRO.java
│   ├── NodeTraceRO.java
│   └── ... (其他 RO)
├── ProcessDefinitionFacade.java
├── ProcessInstanceFacade.java
├── TaskFacade.java
└── MonitorFacade.java
```

### 实现模块（storeability-storelife）

```
storelife/workflow/
├── engine/
│   ├── StateMachineEngine.java
│   ├── NodeExecutor.java
│   ├── TransitionEvaluator.java
│   └── AssigneeResolver.java
├── service/
│   ├── ProcessDefinitionService.java
│   ├── ProcessInstanceService.java
│   ├── TaskService.java
│   ├── MonitorService.java
│   └── converter/
│       ├── ProcessDefinitionConverter.java
│       ├── ProcessInstanceConverter.java
│       └── TaskInstanceConverter.java
├── repository/
│   ├── dos/
│   │   ├── ProcessDefinitionDO.java
│   │   ├── ProcessInstanceDO.java
│   │   ├── NodeInstanceDO.java
│   │   ├── TaskInstanceDO.java
│   │   └── OperationLogDO.java
│   ├── mapper/
│   │   ├── ProcessDefinitionMapper.java
│   │   ├── ProcessInstanceMapper.java
│   │   ├── NodeInstanceMapper.java
│   │   ├── TaskInstanceMapper.java
│   │   └── OperationLogMapper.java
│   └── query/
│       ├── QueryProcessDefinition.java
│       ├── QueryProcessInstance.java
│       └── QueryTaskInstance.java
├── facade/
│   ├── assembler/
│   │   ├── ProcessDefinitionAssembler.java
│   │   ├── ProcessInstanceAssembler.java
│   │   └── TaskInstanceAssembler.java
│   ├── ProcessDefinitionFacadeImpl.java
│   ├── ProcessInstanceFacadeImpl.java
│   ├── TaskFacadeImpl.java
│   └── MonitorFacadeImpl.java
├── event/
│   ├── WorkflowEvent.java
│   └── WorkflowEventPublisher.java
├── listener/
│   ├── NotificationListener.java
│   └── AuditLogListener.java
└── job/
    └── WorkflowTimeoutJob.java
```

### common 模块

```
common/constant/
├── WorkflowEnum.java       # NodeTypeEnum, TaskStatusEnum, ProcessStatusEnum 等
└── WorkflowConst.java      # RocketMQ Topic, Redis Key 前缀等常量
```
