# 3C Demo 仓库导航

> 一个面向 **3C / 半导体自动化** 的资料型仓库，用来沉淀 `PC-based 运动控制` 与 `CIM 集成 Demo` 的学习材料、示例项目与现场经验。

## 仓库定位

这个仓库不是单一软件产品，而是一个适合公开展示、学习入门、内部培训与二次开发的资料集合。当前内容主要围绕两个方向展开：

1. **`PCBase/`**：偏“设备控制与运动控制基础”，帮助读者建立从控制卡、伺服、EtherCAT 到开发/调试/排障的整体认知。
2. **`CodeDemo/CIM/`**：偏“产线系统集成 Demo”，用一个 .NET 8 示例演示 3C 行业中 CIM 层如何与设备、接口、数据库和消息流结合。

如果你刚接触 3C 自动化，建议把它当成一套“**从设备控制到工厂集成**”的学习路线；如果你已经有项目经验，也可以把它当成“**文档模板 + Demo 参考 + 现场知识库**”。

## 内容概览

| 路径 | 定位 | 你能获得什么 |
| --- | --- | --- |
| [`PCBase/`](PCBase/) | PC-based Motion Control 学习资料库 | 概念入门、术语解释、控制卡选型、开发分层、EtherCAT/限位/回零、调试与排障清单 |
| [`CodeDemo/CIM/`](CodeDemo/CIM/) | 3C CIM Integration Demo | .NET 8 多项目示例、REST API、SQLite、设备模拟、消息处理、演示脚本 |
| [`CodeDemo/SQL/`](CodeDemo/SQL/) | 制造场景 SQL 资料 | 工业生产场景下常见查询与 SQL 速查示例 |
| [`CodeDemo/EthercatEsi/`](CodeDemo/EthercatEsi/) | EtherCAT 相关代码示例 | 适合继续补充 ESI、主站/从站或协议相关实践 |
| [`工具类/`](工具类/) | 个人常用工具记录 | 可作为配套工具清单的补充入口 |

## 推荐阅读顺序

### 路线 A：零基础想进入 3C / 自动化控制

1. 先读 [`PCBase/README.md`](PCBase/README.md)
2. 再读 [`PCBase/docs/01-认识PCBase与运动控制.md`](PCBase/docs/01-认识PCBase与运动控制.md)
3. 再读 [`PCBase/docs/03-术语表.md`](PCBase/docs/03-术语表.md)
4. 然后看 [`PCBase/docs/07-EtherCAT限位回零与复位.md`](PCBase/docs/07-EtherCAT限位回零与复位.md)
5. 最后配合 [`PCBase/docs/04-开发文档.md`](PCBase/docs/04-开发文档.md) 与 [`PCBase/docs/05-调试与排障清单.md`](PCBase/docs/05-调试与排障清单.md)

### 路线 B：想快速理解 3C 产线系统集成 / CIM Demo

1. 先读 [`CodeDemo/CIM/README.md`](CodeDemo/CIM/README.md)
2. 再读 [`CodeDemo/CIM/docs/CIM_vs_MES.md`](CodeDemo/CIM/docs/CIM_vs_MES.md)
3. 跟着 README 的“演示流程”跑一遍 API、数据库和模拟器
4. 最后看 [`CodeDemo/CIM/调试工具清单.md`](CodeDemo/CIM/调试工具清单.md) 做本地调试与二次开发

### 路线 C：有项目经验，想按主题查资料

- **控制卡/总线选型** → [`PCBase/docs/02-常见运动控制卡型号与选型.md`](PCBase/docs/02-常见运动控制卡型号与选型.md)
- **开发分层/抽象设计** → [`PCBase/docs/04-开发文档.md`](PCBase/docs/04-开发文档.md)
- **现场排障** → [`PCBase/docs/05-调试与排障清单.md`](PCBase/docs/05-调试与排障清单.md)
- **CIM 与 MES 边界** → [`CodeDemo/CIM/docs/CIM_vs_MES.md`](CodeDemo/CIM/docs/CIM_vs_MES.md)
- **接口与演示调用** → [`CodeDemo/CIM/README.md`](CodeDemo/CIM/README.md)

## 两大主线分别适合谁

| 角色 | 建议先看什么 |
| --- | --- |
| 刚入行的自动化/上位机工程师 | `PCBase/` 全套文档 |
| 做设备联调、现场排障的工程师 | `PCBase/docs/05-*`、`PCBase/docs/07-*` |
| 做 MES / CIM / 集成接口的开发者 | `CodeDemo/CIM/README.md`、`CodeDemo/CIM/docs/CIM_vs_MES.md` |
| 团队负责人 / 架构设计者 | 根 README + `PCBase/docs/04-*` + `CodeDemo/CIM/README.md` |
| 想做二次开发或扩展 Demo 的读者 | `CodeDemo/CIM/README.md` 中的目录索引、配置说明、扩展建议 |

## 本地运行 / 预览指南

### 文档预览

- 直接在 GitHub 网页端阅读各级 `README.md`
- 推荐按目录入口阅读，而不是直接从零散文件开始
- 若本地使用 VS Code，可安装 Markdown Preview 进行目录式预览

### `CodeDemo/CIM/` 本地运行

仓库中最适合直接运行演示的项目是 [`CodeDemo/CIM/`](CodeDemo/CIM/)。

```bash
cd CodeDemo/CIM
dotnet build Cim3CIntegrationDemo.slnx
```

更完整的启动顺序、接口示例、数据库说明和演示流程，请直接查看：

- [`CodeDemo/CIM/README.md`](CodeDemo/CIM/README.md)
- [`CodeDemo/CIM/调试工具清单.md`](CodeDemo/CIM/调试工具清单.md)

## 如何贡献

欢迎把这个仓库继续打磨成“可学习、可演示、可扩展”的资料仓库。比较适合的贡献方向包括：

- 补充更多 **PCBase 实战经验**：控制卡示例、驱动器报警码、状态机模板、现场 SOP
- 补充更多 **CIM Demo 扩展**：对接真实消息中间件、增加接口、增加工艺流转场景
- 完善 **目录导航与文档结构**：让新读者更快找到入口
- 提交 **修正文档、补图、补示例** 的 PR

建议的贡献方式：

1. 先通过 Issue 说明你准备补充的方向
2. 尽量沿用现有目录结构与中文文档风格
3. 对示例项目的扩展优先补充“背景 → 场景 → 操作步骤 → 验证方式”
4. 若引入新 Demo，尽量提供最小可运行说明

## 许可与使用说明

- 许可证与再分发边界，请以仓库根目录中的 `LICENSE` 或其他许可文件为准。
- 如果仓库尚未提供明确许可证，请不要默认其等同于“可任意商用、可任意再分发”；若计划长期公开协作，建议补充明确的开源许可证。
- 文档中的厂商、型号、总线、工艺与架构说明主要用于**学习、交流、原型演示与方案讨论**。

## 免责声明 / 适用范围

- 本仓库中的资料与 Demo 主要用于 **学习、演示、培训和二次开发参考**。
- 其中涉及的自动化控制、设备接口、总线与现场排障经验，不应直接替代真实项目中的安全评审、电气评审和设备验收流程。
- 真实产线落地时，请结合现场硬件手册、驱动器参数、工艺要求、安全规范与团队标准进行二次确认。

## 仓库维护建议

如果你准备继续升级这个仓库，下一步最值得做的事情通常是：

1. 给 `CodeDemo/EthercatEsi/` 补一份 README，说明用途、运行方式和与 `PCBase/` 的关系
2. 为 `CodeDemo/SQL/` 增加目录入口说明与适用场景
3. 补充根目录 `LICENSE`
4. 在 `CodeDemo/CIM/` 中增加更完整的请求/响应样例与时序图

欢迎继续完善这个仓库，把它从 Demo 集合逐步沉淀为面向 3C 自动化工程的公开知识库。
