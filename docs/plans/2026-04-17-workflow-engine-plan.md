# 工作流引擎实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 storeability 微服务体系中实现一套状态机驱动的通用工作流引擎，支持流程定义、审批（含会签/或签/加签/转签）、节点回滚、通知集成和流程监控。

**Architecture:** 状态机驱动的三层模型（流程→节点→任务）。节点下任务支持并行（PARALLEL）和串行（SERIAL）两种执行模式。引擎通过 Dubbo RPC 对外暴露服务，通过 RocketMQ 异步推送事件通知，通过 Redis 分布式锁保证并发安全。

**Tech Stack:** Java 1.8, Spring Boot 2.3.7, Dubbo 2.6.7, MyBatis-Plus 3.4.0, MySQL 8.0.17, RocketMQ, Redis, MapStruct 1.4.1, Lombok 1.18.10, Fastjson 1.2.67

**Design Spec:** `docs/2026-04-17-workflow-engine-design.md`

---

## 文件结构

### storeability-common 模块

- Create: `common/constant/WorkflowEnum.java` — 流程相关枚举（节点类型、任务状态、流程状态、操作类型、任务执行模式）
- Create: `common/constant/WorkflowConst.java` — 流程相关常量（RocketMQ Topic/Tag、Redis Key 前缀）

### storeability-storelife-api 模块

- Create: `storelife/api/workflow/ProcessDefinitionFacade.java` — 流程定义 Facade 接口
- Create: `storelife/api/workflow/ProcessInstanceFacade.java` — 流程实例 Facade 接口
- Create: `storelife/api/workflow/TaskFacade.java` — 任务操作 Facade 接口
- Create: `storelife/api/workflow/MonitorFacade.java` — 流程监控 Facade 接口
- Create: `storelife/api/workflow/po/*.java` — 所有入参 PO 对象
- Create: `storelife/api/workflow/ro/*.java` — 所有出参 RO 对象

### storeability-storelife 模块

- Create: `storelife/workflow/repository/dos/*.java` — DO 对象（5个表）
- Create: `storelife/workflow/repository/mapper/*.java` — MyBatis Mapper 接口
- Create: `storelife/workflow/repository/query/*.java` — 查询条件对象
- Create: `storelife/workflow/service/converter/*.java` — MapStruct BO↔DO 转换器
- Create: `storelife/workflow/engine/StateMachineEngine.java` — 状态机引擎核心
- Create: `storelife/workflow/engine/TransitionEvaluator.java` — 条件评估器
- Create: `storelife/workflow/engine/AssigneeResolver.java` — 审批人解析器
- Create: `storelife/workflow/service/ProcessDefinitionService.java` — 流程定义服务
- Create: `storelife/workflow/service/ProcessInstanceService.java` — 流程实例服务
- Create: `storelife/workflow/service/TaskService.java` — 任务服务
- Create: `storelife/workflow/service/MonitorService.java` — 监控服务
- Create: `storelife/workflow/facade/assembler/*.java` — PO/RO↔BO 转换器
- Create: `storelife/workflow/facade/ProcessDefinitionFacadeImpl.java` — 流程定义 Facade 实现
- Create: `storelife/workflow/facade/ProcessInstanceFacadeImpl.java` — 流程实例 Facade 实现
- Create: `storelife/workflow/facade/TaskFacadeImpl.java` — 任务 Facade 实现
- Create: `storelife/workflow/facade/MonitorFacadeImpl.java` — 监控 Facade 实现
- Create: `storelife/workflow/event/WorkflowEvent.java` — 事件定义
- Create: `storelife/workflow/event/WorkflowEventPublisher.java` — RocketMQ 生产者
- Create: `storelife/workflow/listener/NotificationListener.java` — 通知消费者
- Create: `storelife/workflow/listener/AuditLogListener.java` — 审计日志消费者
- Create: `storelife/workflow/job/WorkflowTimeoutJob.java` — 超时预警定时任务

---

### Task 1: 数据库建表 SQL

**Files:**
- Create: `docs/sql/workflow_ddl.sql`

- [ ] **Step 1: 创建建表 SQL 文件**

```sql
-- ============================================
-- 工作流引擎建表 SQL
-- ============================================

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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '流程定义表';

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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '流程实例表';

CREATE TABLE wf_node_instance (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    instance_id         BIGINT          NOT NULL COMMENT '流程实例ID',
    node_id             VARCHAR(64)     NOT NULL COMMENT '节点定义ID',
    node_name           VARCHAR(128)    DEFAULT NULL COMMENT '节点名称',
    node_type           VARCHAR(32)     NOT NULL COMMENT '节点类型',
    status              TINYINT         NOT NULL DEFAULT 0 COMMENT '状态：0-待执行 1-执行中 2-已完成 3-已跳过 4-已拒绝 5-已退回',
    execution_round     INT             NOT NULL DEFAULT 1 COMMENT '执行轮次',
    rollback_from_node  VARCHAR(64)     DEFAULT NULL COMMENT '从哪个节点退回而来',
    operator            VARCHAR(64)     DEFAULT NULL COMMENT '操作人',
    remark              VARCHAR(512)    DEFAULT NULL COMMENT '审批意见',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    gmt_modified        DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
    gmt_end             DATETIME        DEFAULT NULL COMMENT '完成时间',
    PRIMARY KEY (id),
    KEY idx_instance_id (instance_id),
    KEY idx_node_id (node_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '节点实例表';

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
    sort_order          INT             NOT NULL DEFAULT 1 COMMENT '任务排序',
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
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '任务实例表';

CREATE TABLE wf_operation_log (
    id                  BIGINT          NOT NULL AUTO_INCREMENT COMMENT '主键',
    instance_id         BIGINT          NOT NULL COMMENT '流程实例ID',
    node_instance_id    BIGINT          DEFAULT NULL COMMENT '节点实例ID',
    task_id             BIGINT          DEFAULT NULL COMMENT '任务ID',
    operation_type      VARCHAR(32)     NOT NULL COMMENT '操作类型',
    operator            VARCHAR(64)     NOT NULL COMMENT '操作人',
    remark              VARCHAR(512)    DEFAULT NULL COMMENT '操作备注',
    gmt_create          DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    PRIMARY KEY (id),
    KEY idx_instance_id (instance_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT '流程操作日志表';
```

- [ ] **Step 2: 确认 SQL 可执行**

在 MySQL 中执行建表 SQL，确认无语法错误。

---

### Task 2: Common 模块 — 枚举和常量

**Files:**
- Create: `com.tmallyc.storeability.common.constant.WorkflowEnum`
- Create: `com.tmallyc.storeability.common.constant.WorkflowConst`

- [ ] **Step 1: 创建 WorkflowEnum**

```java
package com.tmallyc.storeability.common.constant;

import lombok.AllArgsConstructor;
import lombok.Getter;

public class WorkflowEnum {

    @Getter
    @AllArgsConstructor
    public enum NodeTypeEnum {
        START("START", "开始节点"),
        END("END", "结束节点"),
        APPROVAL("APPROVAL", "审批节点"),
        FILL("FILL", "填写节点"),
        COUNTERSIGN("COUNTERSIGN", "会签节点"),
        OR_SIGN("OR_SIGN", "或签节点"),
        PARALLEL("PARALLEL", "并行节点"),
        CONDITION("CONDITION", "条件节点"),
        WAIT("WAIT", "等待节点");

        private final String code;
        private final String desc;

        public static NodeTypeEnum getByCode(String code) {
            for (NodeTypeEnum value : values()) {
                if (value.getCode().equals(code)) {
                    return value;
                }
            }
            return null;
        }
    }

    @Getter
    @AllArgsConstructor
    public enum ProcessStatusEnum {
        RUNNING(0, "进行中"),
        COMPLETED(1, "已完成"),
        REJECTED(2, "已拒绝"),
        WITHDRAWN(3, "已撤销"),
        SUSPENDED(4, "已挂起");

        private final int code;
        private final String desc;

        public static ProcessStatusEnum getByCode(int code) {
            for (ProcessStatusEnum value : values()) {
                if (value.getCode() == code) {
                    return value;
                }
            }
            return null;
        }
    }

    @Getter
    @AllArgsConstructor
    public enum NodeStatusEnum {
        PENDING(0, "待执行"),
        RUNNING(1, "执行中"),
        COMPLETED(2, "已完成"),
        SKIPPED(3, "已跳过"),
        REJECTED(4, "已拒绝"),
        ROLLBACK(5, "已退回");

        private final int code;
        private final String desc;

        public static NodeStatusEnum getByCode(int code) {
            for (NodeStatusEnum value : values()) {
                if (value.getCode() == code) {
                    return value;
                }
            }
            return null;
        }
    }

    @Getter
    @AllArgsConstructor
    public enum TaskStatusEnum {
        PENDING(0, "待处理"),
        COMPLETED(1, "已完成"),
        REJECTED(2, "已拒绝"),
        TRANSFERRED(3, "已转签"),
        WITHDRAWN(4, "已撤回"),
        ROLLBACK(5, "已退回"),
        INACTIVE(6, "待激活");

        private final int code;
        private final String desc;

        public static TaskStatusEnum getByCode(int code) {
            for (TaskStatusEnum value : values()) {
                if (value.getCode() == code) {
                    return value;
                }
            }
            return null;
        }
    }

    @Getter
    @AllArgsConstructor
    public enum DefinitionStatusEnum {
        DRAFT(0, "草稿"),
        PUBLISHED(1, "已发布"),
        DISABLED(2, "已停用");

        private final int code;
        private final String desc;
    }

    @Getter
    @AllArgsConstructor
    public enum OperationTypeEnum {
        START("START", "发起流程"),
        APPROVE("APPROVE", "审批通过"),
        REJECT("REJECT", "审批拒绝"),
        SUBMIT("SUBMIT", "提交任务"),
        TRANSFER("TRANSFER", "转签"),
        ADD_COUNTERSIGN("ADD_COUNTERSIGN", "加签"),
        WITHDRAW("WITHDRAW", "撤销"),
        ROLLBACK("ROLLBACK", "退回"),
        SUSPEND("SUSPEND", "挂起"),
        RESUME("RESUME", "恢复");

        private final String code;
        private final String desc;
    }

    @Getter
    @AllArgsConstructor
    public enum TaskExecutionModeEnum {
        PARALLEL("PARALLEL", "并行执行"),
        SERIAL("SERIAL", "串行执行");

        private final String code;
        private final String desc;

        public static TaskExecutionModeEnum getByCode(String code) {
            for (TaskExecutionModeEnum value : values()) {
                if (value.getCode().equals(code)) {
                    return value;
                }
            }
            return PARALLEL;
        }
    }

    @Getter
    @AllArgsConstructor
    public enum TaskTypeEnum {
        APPROVAL("APPROVAL", "审批"),
        FILL("FILL", "填写");

        private final String code;
        private final String desc;
    }

    @Getter
    @AllArgsConstructor
    public enum AssigneeTypeEnum {
        USER("USER", "指定用户"),
        ROLE("ROLE", "指定角色"),
        INITIATOR("INITIATOR", "流程发起人");

        private final String code;
        private final String desc;
    }
}
```

- [ ] **Step 2: 创建 WorkflowConst**

```java
package com.tmallyc.storeability.common.constant;

public class WorkflowConst {

    /** RocketMQ Topic */
    public static final String TOPIC_WORKFLOW_EVENT = "workflow-event";
    public static final String TOPIC_WORKFLOW_AUDIT = "workflow-audit";

    /** RocketMQ Tag */
    public static final String TAG_TASK_CREATED = "TASK_CREATED";
    public static final String TAG_TASK_COMPLETED = "TASK_COMPLETED";
    public static final String TAG_TASK_ROLLBACK = "TASK_ROLLBACK";
    public static final String TAG_TASK_TIMEOUT = "TASK_TIMEOUT";
    public static final String TAG_PROCESS_COMPLETED = "PROCESS_COMPLETED";
    public static final String TAG_PROCESS_REJECTED = "PROCESS_REJECTED";
    public static final String TAG_OPERATION_LOG = "OPERATION_LOG";

    /** Redis Key 前缀 */
    public static final String REDIS_KEY_TASK_LOCK = "wf:task:lock:";
    public static final String REDIS_KEY_NODE_LOCK = "wf:node:lock:";
    public static final String REDIS_KEY_INSTANCE_LOCK = "wf:instance:lock:";

    /** Redis 锁超时时间（秒） */
    public static final int LOCK_TIMEOUT_SECONDS = 30;

    private WorkflowConst() {
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add .
git commit -m "feat(workflow): add workflow enums and constants"
```

---

### Task 3: API 模块 — PO/RO 对象

**Files:**
- Create: `com.tmallyc.storeability.storelife.api.workflow.po.*`
- Create: `com.tmallyc.storeability.storelife.api.workflow.ro.*`

- [ ] **Step 1: 创建流程定义相关 PO**

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class CreateProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    /** 流程唯一标识 */
    private String processKey;
    /** 流程名称 */
    private String processName;
    /** 流程定义JSON */
    private String definitionJson;
    /** 备注 */
    private String remark;
    /** 创建人 */
    private String creator;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class UpdateProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String processName;
    private String definitionJson;
    private String remark;
    private String modifier;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class PublishProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String modifier;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class DisableProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String modifier;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class GetProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String processKey;
    private Integer version;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class QueryProcessDefinitionPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String processKey;
    private String processName;
    private Integer status;
    private Integer pageNum;
    private Integer pageSize;
}
```

- [ ] **Step 2: 创建流程实例相关 PO**

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;
import java.util.Map;

@Data
public class StartProcessPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String processKey;
    private String businessKey;
    private String businessType;
    private String title;
    private String initiator;
    private Map<String, Object> variables;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class WithdrawProcessPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
    private String operator;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class SuspendProcessPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
    private String operator;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class ResumeProcessPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
    private String operator;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class GetProcessDetailPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class QueryMyInitiatedPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String initiator;
    private String processKey;
    private Integer status;
    private Integer pageNum;
    private Integer pageSize;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class ListOperationLogPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
}
```

- [ ] **Step 3: 创建任务相关 PO**

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;
import java.util.Map;

@Data
public class ApproveTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private Map<String, Object> businessData;
    private Map<String, Object> approvalData;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;
import java.util.Map;

@Data
public class RejectTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private Map<String, Object> approvalData;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;
import java.util.Map;

@Data
public class SubmitTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private Map<String, Object> businessData;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class TransferTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private String targetAssignee;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class AddCountersignPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private String additionalAssignee;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class RollbackTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String operator;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class RollbackToNodePO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private String targetNodeId;
    private String operator;
    private String remark;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class ListRollbackableNodePO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class QueryMyTodoTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String assignee;
    private String processKey;
    private Integer pageNum;
    private Integer pageSize;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class QueryMyDoneTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String assignee;
    private String processKey;
    private Integer pageNum;
    private Integer pageSize;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class GetTaskDetailPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
}
```

- [ ] **Step 4: 创建监控相关 PO**

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class ProcessStatisticsPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String processKey;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class QueryTimeoutTaskPO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String processKey;
    private Integer pageNum;
    private Integer pageSize;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.po;

import lombok.Data;
import java.io.Serializable;

@Data
public class ListNodeTracePO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long instanceId;
}
```

- [ ] **Step 5: 创建 RO 对象**

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;

@Data
public class ProcessDefinitionRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String processKey;
    private String processName;
    private Integer version;
    private Integer status;
    private String definitionJson;
    private String remark;
    private String creator;
    private Date gmtCreate;
    private Date gmtModified;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;

@Data
public class ProcessInstanceRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String processKey;
    private String title;
    private Integer status;
    private String currentNodeId;
    private String initiator;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;
import java.util.List;
import java.util.Map;

@Data
public class ProcessInstanceDetailRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String processKey;
    private Long definitionId;
    private String businessKey;
    private String businessType;
    private String title;
    private Integer status;
    private String currentNodeId;
    private String initiator;
    private Map<String, Object> variables;
    private List<NodeInstanceRO> nodeInstances;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;
import java.util.List;

@Data
public class NodeInstanceRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private String nodeId;
    private String nodeName;
    private String nodeType;
    private Integer status;
    private Integer executionRound;
    private List<TaskInstanceRO> taskInstances;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;
import java.util.Map;

@Data
public class TaskInstanceRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private Long instanceId;
    private Long nodeInstanceId;
    private String taskDefId;
    private String taskName;
    private String taskType;
    private String assignee;
    private String assigneeRole;
    private Integer status;
    private Integer sortOrder;
    private String remark;
    private Date dueTime;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;
import java.util.List;
import java.util.Map;

@Data
public class TaskInstanceDetailRO implements Serializable {
    private static final long serialVersionUID = 1L;
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

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.List;

@Data
public class ParamDefinitionRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String key;
    private String name;
    private String type;
    private Boolean required;
    private List<String> options;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;

@Data
public class RollbackableNodeRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String nodeId;
    private String nodeName;
    private Integer lastExecutionRound;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;

@Data
public class OperationLogRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long id;
    private Long instanceId;
    private Long nodeInstanceId;
    private Long taskId;
    private String operationType;
    private String operator;
    private String remark;
    private Date gmtCreate;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;

@Data
public class ProcessStatisticsRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Integer runningCount;
    private Integer completedCount;
    private Integer rejectedCount;
    private Integer withdrawnCount;
    private Integer totalCount;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;

@Data
public class TimeoutTaskRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private Long taskId;
    private Long instanceId;
    private String taskName;
    private String assignee;
    private String processKey;
    private String title;
    private Date dueTime;
    private Date gmtCreate;
}
```

```java
package com.tmallyc.storeability.storelife.api.workflow.ro;

import lombok.Data;
import java.io.Serializable;
import java.util.Date;

@Data
public class NodeTraceRO implements Serializable {
    private static final long serialVersionUID = 1L;
    private String nodeId;
    private String nodeName;
    private String nodeType;
    private Integer status;
    private Integer executionRound;
    private String operator;
    private String remark;
    private Date gmtCreate;
    private Date gmtEnd;
}
```

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat(workflow): add workflow PO and RO objects"
```

---

### Task 4: API 模块 — Facade 接口

**Files:**
- Create: `com.tmallyc.storeability.storelife.api.workflow.ProcessDefinitionFacade`
- Create: `com.tmallyc.storeability.storelife.api.workflow.ProcessInstanceFacade`
- Create: `com.tmallyc.storeability.storelife.api.workflow.TaskFacade`
- Create: `com.tmallyc.storeability.storelife.api.workflow.MonitorFacade`

- [ ] **Step 1: 创建 ProcessDefinitionFacade**

```java
package com.tmallyc.storeability.storelife.api.workflow;

import com.tmallyc.storeability.storelife.api.workflow.po.*;
import com.tmallyc.storeability.storelife.api.workflow.ro.ProcessDefinitionRO;
import com.tmallyc.storeability.common.result.Result;
import com.tmallyc.storeability.common.result.PageRO;

public interface ProcessDefinitionFacade {

    Result<Long> createDefinition(CreateProcessDefinitionPO po);

    Result<Boolean> updateDefinition(UpdateProcessDefinitionPO po);

    Result<Boolean> publishDefinition(PublishProcessDefinitionPO po);

    Result<Boolean> disableDefinition(DisableProcessDefinitionPO po);

    Result<ProcessDefinitionRO> getDefinition(GetProcessDefinitionPO po);

    Result<PageRO<ProcessDefinitionRO>> pageQueryDefinitions(QueryProcessDefinitionPO po);
}
```

- [ ] **Step 2: 创建 ProcessInstanceFacade**

```java
package com.tmallyc.storeability.storelife.api.workflow;

import com.tmallyc.storeability.storelife.api.workflow.po.*;
import com.tmallyc.storeability.storelife.api.workflow.ro.*;
import com.tmallyc.storeability.common.result.Result;
import com.tmallyc.storeability.common.result.PageRO;
import java.util.List;

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

- [ ] **Step 3: 创建 TaskFacade**

```java
package com.tmallyc.storeability.storelife.api.workflow;

import com.tmallyc.storeability.storelife.api.workflow.po.*;
import com.tmallyc.storeability.storelife.api.workflow.ro.*;
import com.tmallyc.storeability.common.result.Result;
import com.tmallyc.storeability.common.result.PageRO;
import java.util.List;

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

- [ ] **Step 4: 创建 MonitorFacade**

```java
package com.tmallyc.storeability.storelife.api.workflow;

import com.tmallyc.storeability.storelife.api.workflow.po.*;
import com.tmallyc.storeability.storelife.api.workflow.ro.*;
import com.tmallyc.storeability.common.result.Result;
import com.tmallyc.storeability.common.result.PageRO;
import java.util.List;

public interface MonitorFacade {

    Result<ProcessStatisticsRO> getProcessStatistics(ProcessStatisticsPO po);

    Result<PageRO<TimeoutTaskRO>> pageQueryTimeoutTasks(QueryTimeoutTaskPO po);

    Result<List<NodeTraceRO>> listNodeTrace(ListNodeTracePO po);
}
```

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat(workflow): add workflow Facade interfaces"
```

---

### Task 5: 数据访问层 — DO、Mapper、Query

**Files:**
- Create: `com.tmallyc.storeability.storelife.workflow.repository.dos.*`
- Create: `com.tmallyc.storeability.storelife.workflow.repository.mapper.*`
- Create: `com.tmallyc.storeability.storelife.workflow.repository.query.*`

- [ ] **Step 1: 创建 DO 对象**

```java
package com.tmallyc.storeability.storelife.workflow.repository.dos;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.util.Date;

@Data
@TableName("wf_process_definition")
public class ProcessDefinitionDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String processKey;
    private String processName;
    private Integer version;
    private Integer status;
    private String definitionJson;
    private String remark;
    private String creator;
    private Date gmtCreate;
    private String modifier;
    private Date gmtModified;
    @TableLogic
    private Integer isDeleted;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.dos;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.util.Date;

@Data
@TableName("wf_process_instance")
public class ProcessInstanceDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String processKey;
    private Long definitionId;
    private String businessKey;
    private String businessType;
    private String title;
    private Integer status;
    private String currentNodeId;
    private String initiator;
    private String variables;
    private Date gmtCreate;
    private Date gmtModified;
    private Date gmtEnd;
    @TableLogic
    private Integer isDeleted;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.dos;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.util.Date;

@Data
@TableName("wf_node_instance")
public class NodeInstanceDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long instanceId;
    private String nodeId;
    private String nodeName;
    private String nodeType;
    private Integer status;
    private Integer executionRound;
    private String rollbackFromNode;
    private String operator;
    private String remark;
    private Date gmtCreate;
    private Date gmtModified;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.dos;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.util.Date;

@Data
@TableName("wf_task_instance")
public class TaskInstanceDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long instanceId;
    private Long nodeInstanceId;
    private String taskDefId;
    private String taskName;
    private String taskType;
    private String assignee;
    private String assigneeRole;
    private Integer status;
    private Integer sortOrder;
    private String businessData;
    private String approvalData;
    private String remark;
    private Date dueTime;
    private Date gmtCreate;
    private Date gmtModified;
    private Date gmtEnd;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.dos;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.util.Date;

@Data
@TableName("wf_operation_log")
public class OperationLogDO {
    @TableId(type = IdType.AUTO)
    private Long id;
    private Long instanceId;
    private Long nodeInstanceId;
    private Long taskId;
    private String operationType;
    private String operator;
    private String remark;
    private Date gmtCreate;
}
```

- [ ] **Step 2: 创建 Mapper 接口**

```java
package com.tmallyc.storeability.storelife.workflow.repository.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.tmallyc.storeability.storelife.workflow.repository.dos.ProcessDefinitionDO;

public interface ProcessDefinitionMapper extends BaseMapper<ProcessDefinitionDO> {
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.tmallyc.storeability.storelife.workflow.repository.dos.ProcessInstanceDO;

public interface ProcessInstanceMapper extends BaseMapper<ProcessInstanceDO> {
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.tmallyc.storeability.storelife.workflow.repository.dos.NodeInstanceDO;

public interface NodeInstanceMapper extends BaseMapper<NodeInstanceDO> {
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.tmallyc.storeability.storelife.workflow.repository.dos.TaskInstanceDO;

public interface TaskInstanceMapper extends BaseMapper<TaskInstanceDO> {
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.tmallyc.storeability.storelife.workflow.repository.dos.OperationLogDO;

public interface OperationLogMapper extends BaseMapper<OperationLogDO> {
}
```

- [ ] **Step 3: 创建 Query 对象**

```java
package com.tmallyc.storeability.storelife.workflow.repository.query;

import lombok.Data;

@Data
public class QueryProcessDefinition {
    private String processKey;
    private String processName;
    private Integer status;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.query;

import lombok.Data;

@Data
public class QueryProcessInstance {
    private String processKey;
    private String initiator;
    private Integer status;
    private String businessKey;
}
```

```java
package com.tmallyc.storeability.storelife.workflow.repository.query;

import lombok.Data;

@Data
public class QueryTaskInstance {
    private String assignee;
    private String processKey;
    private Integer status;
}
```

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat(workflow): add DO, Mapper and Query objects"
```

---

### Task 6: 引擎核心 — TransitionEvaluator 条件评估器

**Files:**
- Create: `com.tmallyc.storeability.storelife.workflow.engine.TransitionEvaluator`

- [ ] **Step 1: 实现 TransitionEvaluator**

```java
package com.tmallyc.storeability.storelife.workflow.engine;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.Map;

@Slf4j
@Component
public class TransitionEvaluator {

    /**
     * 评估条件表达式
     * 支持: ==, !=, >, <, >=, <=, null 判断, && 和 || 组合
     */
    public boolean evaluate(String condition, Map<String, Object> variables) {
        if (condition == null || condition.trim().isEmpty()) {
            return true;
        }

        String trimmed = condition.trim();

        // 处理 && 组合
        if (trimmed.contains("&&")) {
            String[] parts = trimmed.split("&&");
            for (String part : parts) {
                if (!evaluateSingle(part.trim(), variables)) {
                    return false;
                }
            }
            return true;
        }

        // 处理 || 组合
        if (trimmed.contains("||")) {
            String[] parts = trimmed.split("\\|\\|");
            for (String part : parts) {
                if (evaluateSingle(part.trim(), variables)) {
                    return true;
                }
            }
            return false;
        }

        return evaluateSingle(trimmed, variables);
    }

    private boolean evaluateSingle(String expression, Map<String, Object> variables) {
        // != null
        if (expression.endsWith("!= null")) {
            String varName = expression.replace("!= null", "").trim();
            Object value = variables.get(varName);
            return value != null;
        }

        // == null
        if (expression.endsWith("== null")) {
            String varName = expression.replace("== null", "").trim();
            Object value = variables.get(varName);
            return value == null;
        }

        // != 判断
        if (expression.contains("!=")) {
            String[] parts = expression.split("!=", 2);
            String varName = parts[0].trim();
            String expected = stripQuotes(parts[1].trim());
            Object actual = variables.get(varName);
            return actual == null || !String.valueOf(actual).equals(expected);
        }

        // >= 判断
        if (expression.contains(">=")) {
            return compareNumeric(expression, ">=", variables);
        }

        // <= 判断
        if (expression.contains("<=")) {
            return compareNumeric(expression, "<=", variables);
        }

        // > 判断（排除 >= 已处理）
        if (expression.contains(">") && !expression.contains(">=")) {
            return compareNumeric(expression, ">", variables);
        }

        // < 判断（排除 <= 已处理）
        if (expression.contains("<") && !expression.contains("<=")) {
            return compareNumeric(expression, "<", variables);
        }

        // == 判断
        if (expression.contains("==")) {
            String[] parts = expression.split("==", 2);
            String varName = parts[0].trim();
            String expected = stripQuotes(parts[1].trim());
            Object actual = variables.get(varName);
            if (actual == null) {
                return false;
            }
            return String.valueOf(actual).equals(expected);
        }

        log.warn("无法解析的条件表达式: {}", expression);
        return false;
    }

    private boolean compareNumeric(String expression, String operator, Map<String, Object> variables) {
        String[] parts = expression.split(operator, 2);
        String varName = parts[0].trim();
        String expectedStr = parts[1].trim();

        Object actual = variables.get(varName);
        if (actual == null) {
            return false;
        }

        BigDecimal actualNum = new BigDecimal(String.valueOf(actual));
        BigDecimal expectedNum = new BigDecimal(expectedStr);
        int compareResult = actualNum.compareTo(expectedNum);

        switch (operator) {
            case ">":
                return compareResult > 0;
            case "<":
                return compareResult < 0;
            case ">=":
                return compareResult >= 0;
            case "