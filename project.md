---
alwaysApply: false
description: 当需要新增、修改、重构项目中的 Java 代码（含命名、文件放置位置），或被询问项目结构、模块职责、技术栈时，读取并遵守本规范。
---

# Storeability 项目规范

- **GroupId**: `com.tmallyc.storeability` | **Java**: 1.8（禁止 Java 8+ API） | **Spring Boot**: 2.3.7
- **Dubbo**: 2.6.7 | **MyBatis-Plus**: 3.4.0 | **MapStruct**: 1.4.1 | **Lombok**: 1.18.10
- **Fastjson**: 1.2.67 | **Hutool**: 4.6.7 | **ES**: 7.7.1 | **MySQL**: 8.0.17

---

## 模块职责与依赖

代码必须放到正确的模块中，**严禁反向依赖**（如 common 不能依赖 core，api 不能依赖 server）。

| 模块 | 包路径 | 职责 | 允许依赖 |
|------|--------|------|----------|
| `storeability-api` | `api.{业务}` | Dubbo Facade 接口、PO/RO | 仅 common |
| `storeability-common` | `common` | 常量、枚举、异常、工具类 | 无 |
| `storeability-core` | `core.{业务}` | 核心业务逻辑、数据访问 | common, api, shared, integration |
| `storeability-server` | `server.dubbo.{业务}` | 主业务 Facade 实现 | core, api, common |
| `storeability-integration` | `intergration`（历史拼写，不要改） | 防腐层 | common |
| `storeability-shared` | `shared` | 共享能力（锁、审计、待办） | common |
| `storeability-launcher` | `launcher` | 启动类、配置 | 所有 |
| `storeability-smap` / `-api` | `smap` | 地图商圈业务 | common, shared |
| `storeability-storelife` / `-api` | `storelife` / `storelife.api` | 流程工作流业务 | common, shared |

> 所有包路径前缀为 `com.tmallyc.storeability.`

---

## 代码组织模式（以 storelife 为例）

> 所有模块遵循相同模式，根据业务所属模块替换路径即可，不要全部放到 storelife 中。

### 完整链路示例：新增"城市级别管理"

**api 模块**（`storeability-storelife-api`）：
```
storelife/api/city/
├── po/QueryCityLevelPO.java       # 入参，实现 Serializable
├── ro/CityLevelRO.java            # 出参，实现 Serializable
└── CityLevelFacade.java           # 接口（复杂时拆为 CmdFacade / QueryFacade）
```

**实现模块**（`storeability-storelife`）：
```
storelife/city/
├── service/
│   ├── CityLevelService.java      # 业务服务（复杂时拆为 CmdService / QueryService）
│   └── converter/
│       └── CityLevelConverter.java # MapStruct BO↔DO 转换
├── repository/
│   ├── query/QueryCityLevel.java   # 查询条件
│   ├── dos/CityLevelDO.java        # 数据库对象
│   └── mapper/CityLevelMapper.java # 继承 BaseMapper<DO>
└── facade/
    ├── assembler/CityLevelAssembler.java  # PO/RO↔BO 转换
    └── CityLevelFacadeImpl.java           # Facade 实现，禁止写业务逻辑
```

### 模块对应关系

| 业务所属 | api 模块 | 实现模块 | Facade 实现位置 |
|----------|----------|----------|-----------------|
| 主业务 | `api.{业务}` | `core.{业务}` | `server.dubbo.{业务}` |
| 地图商圈 | `smap.{业务}` | `smap.{业务}` | `smap.{业务}.facade` |
| 流程工作流 | `storelife.api.{业务}` | `storelife.{业务}` | `storelife.{业务}.facade` |

> 主业务 Facade 实现在独立的 `storeability-server`；smap/storelife 的 Facade 实现在各自模块的 `facade/` 包下。

### 其他新增场景

- **定时任务**: `core/job/{业务}/`（smap: `smap/job/`，storelife: `storelife/{业务}/job/`），命名 `{描述}Job`
- **ONS 消费者**: `core/ons/{业务}/`，消息体在 `ons/body/`
- **枚举/常量**: `common/constant/`，命名 `{描述}Enum` 或 `{描述}Const`
- **工具类**: 公共放 `common/util/`，模块内放各自 `util/`
- **外部调用**: `intergration/`（注意历史拼写）

---

## 命名规范

**包**: `com.tmallyc.storeability.{module}.{business}.{layer}`，全小写

**类命名**:

| 类型 | 模式 | 示例 |
|------|------|------|
| Facade 接口/实现 | `{Business}CmdFacade` / `Impl` | `TaskTemplateCmdFacadeImpl` |
| 业务服务 | `{Business}CmdService` / `QueryService` | `StrategyCmdService` |
| 入参/出参 | `{描述}PO` / `{描述}RO` | `TaskTemplatePO` / `StrategyRO` |
| 业务/数据库对象 | `{描述}BO` / `{描述}DO` | `StrategyBO` / `StrategyDO` |
| 转换器/装配器 | `{Business}Converter` / `{Business}Assembler` | `CityLevelConverter` |
| 定时任务/处理器 | `{描述}Job` / `{描述}Handler` / `{描述}Action` | `PortraitJob` |

**方法**: 新增 `add/create`、编辑 `edit/update`、删除 `delete/remove`、查询 `get/list/pageQuery`、失效 `invalid`

---

## 代码约束

**必须**:
- `@Slf4j` 记录日志 | `@Data` 标注数据对象 | MapStruct 做对象转换 | PO/RO 实现 `Serializable`
- Dubbo `@Service` 暴露服务 | `@Resource` 或 `@Autowired` 注入

**禁止**:
- Facade 实现类中写业务逻辑 | common 中引入业务逻辑 | api 中引入 Spring 依赖
- Java 8+ API（`var`、`List.of()`、`String.isBlank()` 等） | 空 catch 块
