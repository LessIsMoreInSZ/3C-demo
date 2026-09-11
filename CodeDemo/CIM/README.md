# 3C Industry CIM Integration Demo

> 一个基于 **.NET 8 + SQLite + REST API + 设备模拟** 的 3C 行业 CIM 集成演示项目，适合用于理解 `CIM 是什么`、`MES / 设备 / 接口 / 数据库如何串起来`，以及如何从 Demo 继续扩展到真实项目。

## 项目定位

这个 Demo 不是完整的工厂系统，而是一个“**足够小、足够完整**”的教学型集成样例，用来演示 3C 产线中常见的这些能力：

- 设备状态查询
- Recipe 校验
- TrackIn / TrackOut 过站记录
- 测试结果归档
- 设备告警与状态事件持久化
- 用模拟器替代真实设备，便于本地演示与二次开发

如果你需要先理解 `CIM` 与 `MES` 的职责边界，建议先看：

- [`docs/CIM_vs_MES.md`](docs/CIM_vs_MES.md)

## 适合谁阅读

- 刚开始接触 **3C / 半导体工厂系统集成** 的开发者
- 想用一个最小项目理解 **MES / CIM / 设备接口 / 追溯库** 关系的人
- 想做 **.NET 工业集成 Demo**、培训课件或二次开发样板的人
- 想把当前 Demo 替换成 **RabbitMQ / Kafka / 真正设备接入** 的工程师

## 先看什么

| 你的问题 | 先看哪一节 |
| --- | --- |
| 这个 Demo 到底演示什么？ | [项目定位](#项目定位) + [演示范围](#演示范围与非目标) |
| 整个系统有哪些项目？ | [目录索引](#目录索引) |
| 数据和消息是怎么流的？ | [架构说明](#架构说明) + [数据库与消息流说明](#数据库与消息流说明) |
| 本地怎么跑起来？ | [快速启动](#快速启动) |
| 接口怎么调用？ | [接口说明](#接口说明) |
| 如何演示给别人看？ | [推荐演示流程](#推荐演示流程) |
| 想继续扩展成真实项目 | [扩展建议](#扩展建议) |

## 目录索引

```text
CodeDemo/CIM/
├── Cim3CIntegrationDemo.slnx          # 解决方案入口
├── README.md                          # 项目说明、启动与演示入口
├── docs/
│   └── CIM_vs_MES.md                  # 解释 CIM 与 MES 的区别与关系
├── src/
│   ├── Cim.DbAdapter/                 # SQLite/Dapper、领域模型、Schema、进程内事件总线接口
│   ├── Cim.RestApi/                   # ASP.NET Core Minimal API，对外集成接口
│   ├── Cim.MqWorker/                  # 事件订阅与持久化处理示例
│   └── Cim.DeviceSimulator/           # 设备/PLC 模拟器 + TCP 控制端口
└── 调试工具清单.md                     # 本地调试、观测、接口验证建议
```

## Solution Structure

```text
Cim3CIntegrationDemo.slnx
└── src/
    ├── Cim.DbAdapter/          Class library – SQLite/Dapper persistence, shared models, event bus abstractions
    ├── Cim.RestApi/            ASP.NET Core Minimal API – CIM integration endpoints + Swagger
    ├── Cim.MqWorker/           Worker Service – event consumption, normalized DB writes
    └── Cim.DeviceSimulator/    Console App – shop-floor equipment/PLC simulation, TCP control port
```

## 架构说明

### 1. 逻辑角色

| 组件 | 职责 | 对应真实场景 |
| --- | --- | --- |
| `Cim.RestApi` | 对外暴露集成接口 | 被 MES / QMS / 测试系统调用 |
| `Cim.DeviceSimulator` | 模拟 SMT、AOI、TEST 设备状态与事件 | 真实现场中的设备、PLC、工站控制程序 |
| `Cim.MqWorker` | 订阅事件并归一化落库 | 真实项目中的消息消费、事件处理、解耦层 |
| `Cim.DbAdapter` | 提供 Repository、Schema、模型、事件总线接口 | 共享基础设施层 |
| SQLite | 保存状态、过站、测试结果、告警 | 真实项目中的集成数据库 / 追溯库 |

### 2. 当前 Demo 的重要边界

当前仓库使用的是 **`InMemoryEventBus`（进程内事件总线）**，它只在**同一个进程内**共享消息。

这意味着：

- `Cim.RestApi`、`Cim.MqWorker`、`Cim.DeviceSimulator` **分别单独启动时，并不会跨进程共享同一个消息总线实例**。
- `Cim.DeviceSimulator` 为了方便本地演示，已经在自身进程内直接订阅事件并写入数据库，因此**即使不启动 `Cim.MqWorker`，模拟器事件也能落库**。
- `Cim.MqWorker` 更适合被理解为“**未来替换成 RabbitMQ / Kafka 后的处理层样板**”，或用于演示消息订阅代码组织方式。

如果要实现真正的“设备 → 消息总线 → Worker → 数据库”跨进程链路，需要把 `InMemoryEventBus` 换成外部消息中间件实现。

### 3. 当前 Demo 的两种理解方式

#### 模式 A：本地教学 / 单机演示（当前仓库默认更贴近这个模式）

```text
DeviceSimulator ──(进程内事件订阅)──► SQLite
RestApi         ────────────────────► SQLite
```

适合：

- 本地快速跑通 Demo
- 演示设备状态变化、接口调用、数据库结果
- 培训新人理解“设备 + 接口 + 数据”闭环

#### 模式 B：未来真实项目扩展示意

```text
Device / PLC / Adapter
        │
        ▼
 Message Broker (RabbitMQ / Kafka)
        │
        ▼
    Cim.MqWorker
        │
        ▼
      SQLite / DB
        ▲
        │
     Cim.RestApi
```

适合：

- 扩展为真实集成架构
- 跨进程、跨服务解耦
- 接入更多设备、更多工站、更多事件类型

## 演示范围与非目标

### 当前已经覆盖

- 设备状态查询
- Recipe 兼容性校验
- TrackIn / TrackOut 过站
- 测试结果写入与查询
- 设备模拟器周期运行
- TCP 控制端口手动触发状态/报警/测试事件
- SQLite Schema 自动初始化
- Swagger 接口浏览

### 当前未覆盖或仅做概念示意

- 真正跨进程共享的消息总线
- 用户认证、权限模型、审计
- 复杂 Route / WIP / Hold / Release 规则
- 真实 SECS/GEM、PLC、AMHS、EAP 协议接入
- 高并发、分布式部署、生产级容错

## 快速启动

### 运行前提

- .NET 8 SDK: https://dotnet.microsoft.com/download/dotnet/8.0

### 1. 构建解决方案

```bash
cd CodeDemo/CIM
dotnet build Cim3CIntegrationDemo.slnx
```

### 2. 启动 RestApi

```bash
cd CodeDemo/CIM/src/Cim.RestApi
dotnet run
```

- 默认地址：`http://localhost:5100`
- Swagger：`http://localhost:5100/swagger`

### 3. 启动 DeviceSimulator

```bash
cd CodeDemo/CIM/src/Cim.DeviceSimulator
dotnet run
```

- 默认 TCP 控制端口：`7001`
- 会周期性模拟 `SMT-01`、`AOI-01`、`TEST-01` 三台设备

### 4. （可选）启动 MqWorker

```bash
cd CodeDemo/CIM/src/Cim.MqWorker
dotnet run
```

> 说明：在当前 `InMemoryEventBus` 实现下，`MqWorker` 更偏“架构演示 / 将来替换消息中间件的样板”；它不是当前本地演示跑通的强制前置条件。

## 配置说明

### Environment Variables

| Variable | Default | Description | 适用项目 |
|---|---|---|---|
| `Database__Path` | `./data/cim.db` | SQLite 数据库文件路径 | 全部 |
| `Urls` | `http://localhost:5100` | REST API 监听地址 | `Cim.RestApi` |
| `TcpControlPort` | `7001` | 模拟器 TCP 控制端口 | `Cim.DeviceSimulator` |
| `CycleIntervalSeconds` | `5` | 模拟器循环周期（秒） | `Cim.DeviceSimulator` |

### appsettings.json

每个项目都带有自己的 `appsettings.json`，可以通过环境变量或 `appsettings.{Environment}.json` 覆盖：

- `src/Cim.RestApi/appsettings.json`
- `src/Cim.MqWorker/appsettings.json`
- `src/Cim.DeviceSimulator/appsettings.json`

### 数据文件路径要注意什么

默认 `Database:Path` 是 `./data/cim.db`，它是**相对于各自进程的工作目录**解析的。为了避免多个进程各自写到不同位置，建议统一从**同一个工作目录**启动，或更直接地**始终显式指定同一个绝对路径**；对于多进程演示，更推荐后者，例如：

```bash
Database__Path=/tmp/cim-demo/cim.db dotnet run
```

## 接口说明

Swagger UI: **http://localhost:5100/swagger**

| Method | Path | Description |
|---|---|---|
| GET | `/api/equipment/{equipmentId}/status` | Get current equipment status |
| POST | `/api/recipe/verify` | Verify recipe compatibility |
| POST | `/api/trackin` | Record a TrackIn event |
| POST | `/api/trackout` | Record a TrackOut event |
| POST | `/api/testresults/upsert` | Upsert test result (idempotent) |
| GET | `/api/testresults/{sn}` | Query test result by serial number |

### Idempotency

`Cim.RestApi` 中包含 `IdempotencyMiddleware`。对于 `POST` 接口，建议带上：

```http
Idempotency-Key: <unique-key>
```

重复请求相同 key 时，会返回缓存响应，避免重复写入。

### 最常用的 3 类接口场景

1. **设备状态查询**：给 MES、调度系统或演示页面查当前设备状态
2. **过站与履历**：用 `trackin / trackout` 记录 SN/Lot 在工站的流转
3. **测试结果归档**：把 FCT/ICT/AOI 的判定和测量值写入追溯库

## Sample curl Commands

### Get equipment status
```bash
curl -s http://localhost:5100/api/equipment/SMT-01/status | jq
```

### Verify recipe
```bash
curl -s -X POST http://localhost:5100/api/recipe/verify \
  -H "Content-Type: application/json" \
  -d '{"equipmentId":"SMT-01","recipeId":"RCP-001"}' | jq
```

### TrackIn
```bash
curl -s -X POST http://localhost:5100/api/trackin \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: trackin-001" \
  -d '{
    "serialNumber": "SN-TEST-001",
    "lotId": "LOT-20240101",
    "equipmentId": "SMT-01",
    "stationId": "SMT-01-ST1",
    "recipeId": "RCP-001",
    "operator": "OP001"
  }' | jq
```

### TrackOut
```bash
curl -s -X POST http://localhost:5100/api/trackout \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: trackout-001" \
  -d '{
    "serialNumber": "SN-TEST-001",
    "lotId": "LOT-20240101",
    "equipmentId": "SMT-01",
    "stationId": "SMT-01-ST1",
    "operator": "OP001"
  }' | jq
```

### Upsert test result
```bash
curl -s -X POST http://localhost:5100/api/testresults/upsert \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: result-sn001-st1" \
  -d '{
    "serialNumber": "SN-TEST-001",
    "lotId": "LOT-20240101",
    "equipmentId": "TEST-01",
    "stationId": "TEST-01-ST1",
    "testProgram": "FCT-v1.0",
    "verdict": "PASS",
    "operator": "OP001",
    "items": [
      {"itemName":"Voltage","measuredValue":5.01,"lowerLimit":4.80,"upperLimit":5.20,"unit":"V","verdict":"PASS"},
      {"itemName":"Current","measuredValue":0.99,"lowerLimit":0.90,"upperLimit":1.10,"unit":"A","verdict":"PASS"}
    ]
  }' | jq
```

### Query test result by serial number
```bash
curl -s http://localhost:5100/api/testresults/SN-TEST-001 | jq
```

## TCP Control Port (DeviceSimulator)

Connect via telnet or nc:

```bash
nc 127.0.0.1 7001
```

Available commands:

```text
STATE <equipmentId> <state>                        Force state transition
ALARM <equipmentId> <code> <level> <description>  Raise an alarm
CLEAR <equipmentId> <code>                        Clear an alarm
TEST <equipmentId>                                Emit a test result
STATUS                                            List all equipment states
QUIT                                              Close connection
```

Examples:

```text
STATE SMT-01 MAINTENANCE
ALARM AOI-01 ALM-002 ERROR Camera focus failed
CLEAR AOI-01 ALM-002
TEST TEST-01
STATUS
```

## 数据库与消息流说明

### Database Schema

SQLite database at `./data/cim.db` (or your overridden `Database:Path`):

| Table | Description | 用途 |
|---|---|---|
| `EquipmentStatus` | Current state of each equipment (upsert by `EquipmentId`) | 保存设备当前状态、配方、批次 |
| `TrackEvents` | TRACKIN/TRACKOUT history | 保存 SN/Lot 过站履历 |
| `TestResults` | Test result headers (upsert by `SerialNumber + StationId`) | 保存测试结果主表 |
| `TestItems` | Individual test measurements per result | 保存明细测量项 |
| `Alarms` | Alarm history (`ACTIVE`/`CLEARED`) | 保存设备告警生命周期 |

### 当前 Demo 中的消息/数据流

#### 1. API 驱动的数据流

```text
MES / Client
    │
    ▼
Cim.RestApi
    │
    ├── TrackIn / TrackOut ─────► TrackEvents
    ├── TestResults Upsert ────► TestResults + TestItems
    └── Equipment Query ◄────── EquipmentStatus
```

#### 2. 模拟器驱动的数据流

```text
Cim.DeviceSimulator
    │
    ├── 周期状态变化 ───────────► EquipmentStatus
    ├── ALARM / CLEAR 命令 ────► Alarms
    └── TEST 命令 ─────────────► TestResults + TestItems
```

#### 3. 将来扩展为外部消息总线后的目标流

```text
Device / Adapter -> Broker -> MqWorker -> SQLite
```

## 推荐演示流程

如果你要把这个项目演示给同事、客户或新人，可以按下面顺序走：

1. 打开 `docs/CIM_vs_MES.md`，先讲清楚“为什么需要 CIM”
2. 启动 `RestApi`，打开 Swagger，让读者先看到接口边界
3. 启动 `DeviceSimulator`，说明有哪些设备：`SMT-01`、`AOI-01`、`TEST-01`
4. 用 `STATUS` 或 `STATE` 命令演示设备状态变化
5. 调用 `POST /api/trackin` 演示 SN 进站
6. 调用 `POST /api/recipe/verify` 演示 Recipe 校验
7. 用 `TEST TEST-01` 或 `POST /api/testresults/upsert` 演示测试结果归档
8. 查询 `GET /api/testresults/{sn}`，展示追溯结果
9. 最后解释：当前是教学型 Demo，若要生产化，需要替换外部消息中间件、权限、审计和真实协议接入

## FAQ

### 1. 为什么 README 同时提到了 DeviceSimulator 和 MqWorker，感觉消息流不完全一样？

因为当前实现使用的是进程内 `InMemoryEventBus`。`MqWorker` 展示的是“订阅处理层的代码组织方式”，而 `DeviceSimulator` 为了本地演示已经在自身进程内内联订阅并落库，所以两者表达的是“当前教学模式”和“未来生产模式”两个层次。

### 2. 本地只启动 `RestApi` 和 `DeviceSimulator` 可以吗？

可以。对当前仓库来说，这是最容易跑通的演示组合。

### 3. 为什么同样配置了 `./data/cim.db`，不同进程可能看不到同一份数据？

因为 `./data/cim.db` 是相对路径，具体落到哪里取决于你从哪个工作目录运行。建议统一使用绝对路径配置。

### 4. 这个 Demo 更偏 3C 还是半导体？

当前示例更偏 **3C 产线集成**（SMT、AOI、测试工站），但它采用的很多思路同样适合作为半导体/泛制造集成 Demo 的入门模板。

### 5. 能不能直接把它当生产代码用？

不建议。它更适合用作教学、PoC、方案说明和二次开发基础，生产化还需要补充认证授权、消息可靠性、审计、配置中心、监控、异常重试等能力。

## 扩展建议

### 优先级 1：把 Demo 变成更完整的学习样板

- 增加请求/响应示例截图或时序图
- 增加 `sqlite3` 查询样例
- 增加更多工站与事件类型（如 Hold、Release、Scrap、Rework）

### 优先级 2：把 Demo 变成更真实的集成骨架

- 用 RabbitMQ / Kafka 替换 `InMemoryEventBus`
- 将 `DeviceSimulator` 拆成独立协议适配层
- 为 `RestApi` 增加认证、版本化和统一错误码规范
- 增加后台查询接口（履历、告警、设备看板）

### 优先级 3：把 Demo 变成团队模板

- 增加 `tests/`，为 Repository、API 和幂等逻辑补测试
- 增加 Docker / docker-compose 启动方式
- 增加 `.http` / Postman / Bruno 请求集合
- 增加贡献规范和数据初始化脚本

## 对二次开发者的建议

如果你准备在当前仓库上继续开发，建议优先明确以下 4 件事：

1. 你要演示的是“接口调用闭环”，还是“消息驱动架构闭环”
2. 你的数据库是只做 Demo 存档，还是要做真正追溯查询
3. 你要不要接入真实设备协议，还是继续用模拟器培训新人
4. 你是否准备把当前最小模型扩展为更完整的 Lot / WIP / Recipe / Alarm 域模型

## 贡献指南

欢迎继续补充这个 Demo。推荐的贡献方式：

1. 优先保持“**最小可运行 + 易于讲解**”的风格
2. 新增功能时，尽量同步补充 README 对应章节
3. 若引入新配置、新端口或新表结构，请在 README 中补齐说明
4. 若补充真实中间件或更复杂架构，建议把“当前教学模式”和“生产扩展模式”清晰分开描述

## 项目依赖图

```text
Cim.RestApi ──────┐
Cim.MqWorker ─────┤──► Cim.DbAdapter
Cim.DeviceSimulator ┘
```
