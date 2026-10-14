# SCPI 与 GPIB 介绍及 C# 交互方式

本文档介绍仪器程控中的核心概念——**SCPI**（命令语言）、**GPIB**（物理总线），以及 **VISA / VISA.NET / IVI.NET** 之间的分层关系，并说明如何用 C# 与它们交互。文中示例与仓库 `是德Keysight/CSharp/VISA.NET` 下的示例代码风格一致，可直接参考运行。

---

## 1. 术语体系与分层关系（先理清全局）

仪器程控领域缩写极多，初学最容易被 `SCPI / VISA / VISA.NET / IVI / IVI.NET / GPIB` 绕晕。本节先用一句话概括，再给出分层图，然后逐个概念展开，附上术语速查表，最后说明这些标准各自由谁制定。

### 1.1 一句话理清

| 概念 | 一句话定位 | 回答的问题 |
| --- | --- | --- |
| **SCPI** | 仪器**命令语言**（文本命令） | 对仪器**说什么** |
| **VISA** | 仪器**通信抽象层**（C API） | 通过什么**通道**把命令送出去 |
| **VISA.NET** | VISA 的 **.NET 托管封装** | 怎么在 **C#** 里调用 VISA |
| **IVI / IVI.NET** | 建立在 VISA 之上的**仪器类驱动标准** | 怎么**换同类型仪器不改代码** |
| **GPIB** | 一种**物理总线**（硬件接口） | 命令实际走哪根**线** |

关键点：**SCPI 决定"说什么"，VISA 决定"怎么传"，VISA.NET 是 VISA 的 C# 门面，IVI.NET 是比 VISA 更高一层的"标准化仪器驱动"，GPIB 只是众多物理接口中的一种。**

### 1.2 分层架构图

从下往上，每一层都依赖它下面的一层：

```
┌──────────────────────────────────────────────────────────┐
│ 应用程序 / 测试代码（C#）                                  │
├──────────────────────────────────────────────────────────┤
│ IVI.NET / IVI-C  ：按仪器类别统一的“高层驱动 API”           │  ← 可选；换同类型仪器不改代码
├──────────────────────────────────────────────────────────┤
│ SCPI             ：仪器命令语言（“说什么”）                 │  ← 文本命令，跨接口通用
├──────────────────────────────────────────────────────────┤
│ VISA.NET（Ivi.Visa）：VISA 的 .NET 托管封装                 │  ← C# 面向对象调用
├──────────────────────────────────────────────────────────┤
│ VISA             ：统一 I/O 抽象层（“怎么传”）              │  ← 屏蔽接口差异
├──────────────────────────────────────────────────────────┤
│ GPIB / USB / LAN(TCPIP) / 串口(ASRL) ：物理接口             │  ← 实际的“线”
├──────────────────────────────────────────────────────────┤
│ 仪器硬件                                                    │
└──────────────────────────────────────────────────────────┘
```

理解这张图的一个要点：**SCPI 不依赖 VISA，VISA 也不依赖 SCPI**——两者是"内容"与"管道"的正交关系。VISA 可以传 SCPI，也可以传厂商私有命令；SCPI 可以走 VISA，也可以直接用 `TcpClient` / `SerialPort` 发（见 5.2 方案 B）。

### 1.3 SCPI 与 VISA 的关系

- **SCPI 是应用层的"语言"**：规定仪器能听懂哪些文本命令、返回什么格式，与接口无关。
- **VISA 是传输层的"管道"**：规定程序如何打开会话、读写数据，与命令内容无关。
- 两者组合才构成完整的一次交互：`VISA 打开会话 → 通过会话发送 SCPI 文本 → 通过会话读回 SCPI 响应`。
- 同一条 `MEAS:VOLT:DC?` 既能走 GPIB，也能走 USB / LAN / 串口，正是因为 VISA 把物理接口差异屏蔽掉了。

> 类比：SCPI 好比"英语"，VISA 好比"国际邮政系统"。你会说英语（SCPI），邮政负责把信送到（VISA），走空运还是海运（GPIB/LAN/USB）由邮政决定，你只管写信。

### 1.4 VISA 与 VISA.NET 的关系

- **VISA** 本身是一套 **C 语言 API**（由 IVI Foundation 制定规范，NI-VISA、Keysight IO Libraries 等提供具体实现），函数名形如 `viOpen`、`viWrite`、`viRead`。
- **VISA.NET** 是 VISA 的 **.NET 托管封装**：把 C 风格的 `viXxx` 函数包装成面向对象的 .NET 接口（`GlobalResourceManager.Open`、`session.Write` 等），以 `Ivi.Visa.dll` 程序集形式提供。
- **重要**：`Ivi.Visa.dll` 只是"壳"，**真正的通信能力来自本机安装的 VISA 运行时**（NI-VISA 或 Keysight IO Libraries Suite）。只拷 DLL 不装运行时，程序会在访问 `GlobalResourceManager` 时报"类型初始值设定项引发异常"。
- 因此 VISA.NET **不是**独立于 VISA 的另一套东西，而是"用 C# 写 VISA"的官方方式。

### 1.5 IVI 与 VISA 的关系（IVI.NET 是什么）

- **VISA 只管"通"，不管"懂"**：它能收发任意字节，但不知道"示波器该怎么配置时基"。
- **IVI**（Interchangeable Virtual Instrument，可互换虚拟仪器）是 IVI Foundation 制定的**仪器驱动标准**，它在 VISA 之上，为**同一类仪器**（示波器、万用表、电源、函数发生器等）定义**统一的编程接口**。
- 好处：程序面向"示波器接口"编程，换成另一品牌的示波器时，只要它也有对应 IVI 驱动，**上层代码基本不用改**。
- 两种形态：
  - **IVI-C**：C 语言接口（早期主流）。
  - **IVI.NET**：.NET 接口，基于 `Ivi.Driver` 程序集，是 IVI-C 在 .NET 上的对应物。
- **层次关系**：`IVI.NET →（构建于）→ VISA.NET →（构建于）→ VISA`。IVI.NET 内部仍通过 VISA.NET 收发 SCPI/底层命令，只是帮你把"某型号仪器的一堆 SCPI"封装成了标准方法。

> 类比：VISA 是"电话线路"，谁都能打；IVI 是"标准话术模板"——不管你打给哪家客服（哪台仪器），都按同一套问题问，答案也按同一格式返回。

**一句话区分 VISA 与 IVI**：VISA 统一的是**接口/传输**（怎么连），IVI 统一的是**仪器类别 API**（怎么用）。二者不是替代关系，而是上下层关系。

### 1.6 术语速查表

按主题分组，把前面出现的专业术语逐个解释。

**① 标准与命令语言**

| 术语 | 全称 | 说明 |
| --- | --- | --- |
| **IEEE 488.1** | — | 定义了 GPIB 的**硬件/电气层**：24 针接口、8 位并行数据线、握手时序等。 |
| **IEEE 488.2** | — | 在 488.1 之上定义了**数据格式与公用命令**（如 `*IDN?`、`*RST`、`*CLS`）以及状态报告模型。 |
| **SCPI** | Standard Commands for Programmable Instruments | 可编程仪器标准命令。基于 IEEE 488.2 扩展出的**树形文本命令语言**，规定"对仪器说什么"。 |
| **公用命令** | Common Commands | IEEE 488.2 规定的、以 `*` 开头的命令（`*IDN?`/`*RST`/`*CLS`/`*OPC?`），所有合规仪器通用。 |
| **查询** | Query | 以 `?` 结尾的 SCPI 命令，会**返回数据**；不以 `?` 结尾则为设置/执行，通常不返回。 |

**② VISA 体系**

| 术语 | 全称 | 说明 |
| --- | --- | --- |
| **VISA** | Virtual Instrument Software Architecture | 虚拟仪器软件架构。统一的仪器 I/O **C API 规范**，屏蔽 GPIB/USB/LAN/串口差异。 |
| **VISA 运行时** | VISA Runtime | VISA 规范的**具体实现**，如 NI-VISA、Keysight IO Libraries Suite。必须安装，否则 VISA.NET 无法工作。 |
| **VISA.NET** | — | VISA 的 **.NET 托管封装**，以 `Ivi.Visa.dll` 程序集提供，让 C# 面向对象地使用 VISA。 |
| **Resource Manager** | Resource Manager | 资源管理器，VISA 的入口对象，负责查找/打开/解析资源。.NET 中通过 `GlobalResourceManager` 静态访问。 |
| **资源** | Resource | VISA 对"一台可通信设备"的抽象，用**资源名**（地址字符串）标识，如 `GPIB0::22::INSTR`。 |
| **会话** | Session | 打开资源后得到的通信句柄，所有读写都通过会话进行。 |
| **消息型会话** | Message-Based Session | 以"整条消息"为单位收发字符串的会话，对应 `IMessageBasedSession`。仪器程控绝大多数用它。 |
| **寄存器型会话** | Register-Based Session | 直接读写设备寄存器的会话，对应 `IRegisterBasedSession`，用于高速/底层控制，较少用。 |
| **格式化 I/O** | Formatted I/O | 在会话之上提供按行/按格式收发的封装，对应 `MessageBasedFormattedIO`（`WriteLine`/`ReadLine`）。 |
| **终止字符** | Termination Character | 消息结束标志字符（通常 `\n`）。串口/socket 必须启用，否则读取会一直等到超时。 |
| **超时** | Timeout | 单次读写的最大等待时间（`TimeoutMilliseconds`），防止仪器无应答时程序卡死。 |

**③ IVI 体系**

| 术语 | 全称 | 说明 |
| --- | --- | --- |
| **IVI Foundation** | — | 制定 VISA、IVI 等仪器互操作标准的行业组织。 |
| **IVI** | Interchangeable Virtual Instrument | 可互换虚拟仪器。建立在 VISA 之上的**仪器类驱动标准**，为同类仪器定义统一 API。 |
| **IVI-C** | — | IVI 的 C 语言接口形态。 |
| **IVI.NET** | — | IVI 的 .NET 接口形态，基于 `Ivi.Driver` 程序集。 |
| **仪器驱动** | Instrument Driver | 针对具体型号/类别封装的软件库，把 SCPI 细节隐藏成方法调用。IVI 是其中一种标准化形式。 |

**④ 接口与传输**

| 术语 | 全称 | 说明 |
| --- | --- | --- |
| **GPIB** | General Purpose Interface Bus | 通用接口总线，又称 IEEE 488。8 位**并行**总线，一条最多 15 台设备。 |
| **USB (USBTMC)** | USB Test & Measurement Class | USB 上的仪器控制协议类，VISA 地址形如 `USB0::...::INSTR`。 |
| **LAN / TCPIP** | — | 网络接口。VISA 支持两种：**socket（SCPI-RAW，端口常为 5025）** 与 **VXI-11**（`inst0::INSTR`）。 |
| **VXI-11** | — | 基于 ONC RPC 的 LAN 仪器控制协议，地址形如 `TCPIP0::192.168.1.50::inst0::INSTR`。 |
| **HiSLIP** | High-Speed LAN Instrument Protocol | 比 VXI-11 更快的 LAN 仪器协议，较新仪器支持。 |
| **LXI** | LAN eXtensions for Instrumentation | 基于 LAN 的仪器标准，强调用网络替代 GPIB 机架总线。 |
| **ASRL** | Asynchronous Serial | 异步串口，即 RS-232。VISA 地址形如 `ASRL3::INSTR`。 |
| **EOI** | End Or Identify | GPIB 上标记"消息结束"的**硬件信号线**，替代串口的终止字符。 |
| **SRQ** | Service Request | GPIB 上仪器主动请求服务的信号线，用于异步事件通知。 |

### 1.7 这些标准/术语分别由谁制定

上面这些名词背后是不同的标准化组织，理解"谁说了算"有助于判断某个特性是否可靠、是否通用。

**先认识几个组织：**

| 组织 | 性质 | 负责/曾负责的标准 |
| --- | --- | --- |
| **IEEE**（电气电子工程师学会） | 国际专业学会、标准制定机构 | IEEE 488（GPIB）、IEEE 488.1、IEEE 488.2 |
| **IVI Foundation**（可互换虚拟仪器基金会） | 仪器厂商组成的非营利联盟 | IVI / IVI-C / IVI.NET、**VISA**、**VISA.NET**、**SCPI**、**HiSLIP** |
| **VXIbus Consortium** | 仪器厂商联盟 | VXI 总线、**VXI-11**（LAN 仪器协议） |
| **LXI Consortium** | 仪器厂商联盟 | **LXI**（基于 LAN 的仪器标准） |
| **USB-IF**（USB 开发者论坛） | 行业协会 | USB 规范、**USBTMC**（USB 仪器类） |
| **厂商自身**（NI、Keysight/HP 等） | 商业公司 | VISA/VISA.NET 的**具体实现**（NI-VISA、Keysight IO Libraries）、NI-MAX、Connection Expert 等工具 |

**逐项对应：**

| 标准 / 术语 | 制定或维护方 | 背景与年代 |
| --- | --- | --- |
| **GPIB / IEEE 488** | 由 **HP（惠普）** 于 1960 年代发明，原名 **HP-IB**；1975 年被 **IEEE** 采纳为标准 **IEEE 488** | HP 为自家仪器互联而设计，后成为行业通用总线 |
| **IEEE 488.1** | **IEEE** | 1987 年发布，规定 GPIB 的硬件/电气层（接口、信号线、握手时序） |
| **IEEE 488.2** | **IEEE** | 1992 年发布，规定数据格式、状态模型与 `*IDN?` 等公用命令 |
| **SCPI** | 最初由 **SCPI Consortium**（1990 年成立的仪器厂商联盟）制定；**2004 年该联盟并入 IVI Foundation**，现由 **IVI Foundation** 维护 | 在 IEEE 488.2 之上扩展出树形命令语言，最新正式版本为 **SCPI-1999**（常称 SCPI-99） |
| **VISA** | 最初由 **VXIplug&play Systems Alliance**（1993 年成立，NI、Tektronix、HP 等发起）制定；现规范由 **IVI Foundation** 维护 | 统一的仪器 I/O API 规范 |
| **VISA.NET** | **IVI Foundation** | VISA 的 .NET 语言绑定规范，对应 `Ivi.Visa.dll` |
| **IVI / IVI-C / IVI.NET** | **IVI Foundation**（1998 年成立） | 为同类仪器定义可互换的统一 API，.NET 版对应 `Ivi.Driver` |
| **HiSLIP** | **IVI Foundation** | 2011 年发布的 High-Speed LAN Instrument Protocol，VXI-11 的高速替代 |
| **VXI / VXI-11** | **VXIbus Consortium** | VXI 为 1987 年的模块化仪器总线；VXI-11 是其 LAN 仪器控制协议 |
| **LXI** | **LXI Consortium**（2004 年成立，Agilent 与 VXI Technology 发起） | 用 LAN 替代 GPIB 机架总线的仪器标准 |
| **USBTMC** | **USB-IF** | 2003 年的 USB Test & Measurement Class 规范，让 USB 像 GPIB 一样传仪器消息 |
| **SCPI-RAW socket（端口 5025）** | 无正式标准，属**业界事实约定** | 各厂商普遍把 LAN 上的原始 SCPI 走 TCP 5025 端口，VISA 以 `...::SOCKET` 地址支持 |
| **NI-VISA / NI-MAX** | **NI（National Instruments）** | VISA 规范的实现之一，附带 MAX 配置工具 |
| **Keysight IO Libraries / Connection Expert** | **Keysight**（1999 年从 HP 分拆的 Agilent，2014 年更名 Keysight） | 另一套 VISA 实现，附带 Connection Expert 工具 |

**两点提示：**

- **规范与实现是两回事**：VISA 是 IVI Foundation 的**规范**，NI-VISA、Keysight IO Libraries 才是**具体实现**；规范由基金会定，程序运行起来靠的是本机装的哪家实现（见 1.4）。
- **"事实标准"要留意**：像 `5025` socket 端口这类没有正式标准背书、只是业界约定俗成的做法，不同仪器可能不遵守，接入前最好查仪器手册确认端口与协议。

---

## 2. SCPI 是什么

**SCPI**（Standard Commands for Programmable Instruments，可编程仪器标准命令）是一套**基于文本的仪器控制命令语言**，由 IEEE 488.2 标准扩展而来。它规定了"对仪器说什么"，与用什么物理接口（GPIB / USB / LAN / 串口）无关。

### 2.1 命令结构

SCPI 命令呈**树形分层结构**，用冒号 `:` 分隔层级：

```
MEASure:VOLTage:DC?
└─根──┘ └─子节点─┘ └叶节点
```

- **大小写**：命令中大写字母是**必写部分**（短形式），完整单词是**长形式**。例如 `MEASure` 可写作 `MEAS` 或 `MEASURE`，两者等价。
- **查询**：命令末尾加 `?` 表示**查询**（会返回数据）；不加 `?` 表示**设置/执行**（不返回数据）。
- **参数**：设置类命令后跟参数，如 `CONF:VOLT:DC 10`（配置 10V 直流电压量程）。

### 2.2 常见命令

| 命令 | 作用 |
| --- | --- |
| `*IDN?` | 查询仪器标识（厂商、型号、序列号、固件版本） |
| `*RST` | 复位仪器到出厂默认状态 |
| `*CLS` | 清除状态寄存器 |
| `MEAS:VOLT:DC?` | 测量直流电压（自动量程，触发一次并返回读数） |
| `MEAS:CURR:AC?` | 测量交流电流 |
| `MEAS:RES?` / `MEAS:FRES?` | 测二线电阻 / 四线电阻 |
| `CONF:VOLT:DC 10` | 配置直流电压量程为 10V |
| `READ?` | 触发并读取当前配置下的测量值 |

> 规律：`MEASure:...?` 是"配置 + 触发 + 读取"一步到位；`CONFigure:...` + `READ?` 则把配置与读取分开，适合需要固定量程的场景。

---

## 3. GPIB 是什么

**GPIB**（General Purpose Interface Bus，通用接口总线）又称 **IEEE 488**，是一种用于连接仪器与计算机的**并行总线标准**。

- **物理层**：8 位并行数据线 + 若干控制/握手线，总线式连接，一条总线最多挂 **15 台设备**。
- **地址**：每台仪器有一个 GPIB 地址（通常 0~30），如地址 22 的仪器地址字符串为 `GPIB0::22::INSTR`。
- **硬件**：计算机需要安装 **GPIB 接口卡**（PCI/PCIe/USB-GPIB 转换器），再用 GPIB 线缆级联仪器。
- **地位**：GPIB 历史悠久、稳定可靠，大量老式/工业仪器仍在使用；但速度与便利性不如 LAN 和 USB，新仪器多优先提供 LAN/USB 接口。

### 3.1 GPIB 与 SCPI 的关系

- **SCPI 是"说什么"**（命令语言），**GPIB 是"怎么传"**（物理传输方式）。
- 同一条 SCPI 命令 `MEAS:VOLT:DC?` 可以通过 GPIB、USB、LAN 任意一种接口发送。
- 为屏蔽接口差异，业界用 **VISA** 做统一抽象层：程序只面向 VISA 地址和 SCPI 命令，底层是 GPIB 还是 LAN 由 VISA 处理。

---

## 4. GPIB 与串口（RS-232）通信的差异

GPIB 与串口（RS-232）都是常见的仪器通信物理接口，但设计目标和用法差别很大。下面先总览对比，再逐项说明，最后落到对 C# / VISA 编程的实际影响。

### 4.1 总览对比

| 维度 | GPIB（IEEE 488） | 串口（RS-232） |
| --- | --- | --- |
| 拓扑 | 总线型，**一条总线可挂多台**（最多 15 台） | **点对点**，一个口接一台仪器 |
| 寻址 | 每台有 GPIB 地址（0~30），如 `GPIB0::22::INSTR` | 无地址，靠 **COM 端口号**区分，如 `ASRL3::INSTR` |
| 传输方式 | **8 位并行** + 握手/控制线（DIO、DAV、NRFD、NDAC、EOI…） | **串行**，仅 TX / RX / GND（可加流控线），异步起止位 |
| 速率 | 高，约 1 MB/s，HS488 可达 8 MB/s | 低，常见 9600~115200 bps（≈1~11.5 KB/s） |
| 距离 | 单段 ≤ 2 m、整条总线 ≤ 20 m | 标准约 15 m（降低波特率可略长） |
| 连接器 | 24 针（IEEE 488 / Centronics） | DB9 / DB25（或接线端子） |
| 消息结束标志 | **EOI 硬件线**（辅以 LF） | **终止字符**（通常 `\n` 或 `\r\n`） |
| 握手 / 流控 | 硬件三线握手，逐字节确认，天然可靠 | 无握手，可选软件（XON/XOFF）或硬件（RTS/CTS）流控 |
| 硬件需求 | 需 GPIB 接口卡或 USB-GPIB 转换器 | 需 COM 口或 USB 转串口线 |
| 多机切换 | 总线内按地址点名，无需改接线 | 每台各占一个口，多机需多个口/转换器 |
| 典型场景 | 老式/工业仪器、机架式系统、多仪器同步 | 台式仪器、简单控制器、低带宽场景 |

### 4.2 逐项说明

**拓扑与寻址**：GPIB 是共享总线，主机通过地址“点名”某台仪器收发，一条线缆链上可挂多台；串口是点对点，一个端口只对应一台仪器，没有地址概念，选设备其实就是选 COM 口。因此多仪器系统里 GPIB 更省口、更整齐，串口则要占用多个 COM 口（或改用 RS-485 总线变体做多点）。

**电气与信号**：GPIB 用 8 根数据线并行传一个字节，另有 DAV / NRFD / NDAC 三线握手保证每字节可靠送达，还有 EOI（结束）、SRQ（服务请求）、ATN（注意）等管理线。串口只有 TX / RX 两根信号线，字节按位串行发送，靠起始位/停止位同步，没有握手线，出错检测主要靠奇偶校验。

**速率**：GPIB 并行 + 硬件握手，吞吐通常 1 MB/s 量级，HS488 可达 8 MB/s；串口受波特率限制，115200 bps 约合 11.5 KB/s，慢 1~2 个数量级。传波形、批量读数等大数据时差异明显。

**距离**：GPIB 单段 ≤ 2 m、总线总长 ≤ 20 m，且相邻两台之间不超过 2 m；RS-232 标准约 15 m，降波特率可延长。两者都属短距离，远距离或组网应改用 LAN（`TCPIP`）。

**消息结束标志**：GPIB 用专门的 **EOI 线**在硬件层标记“本条消息结束”，不依赖字符；串口没有这条线，只能在数据里用**终止字符**（`\n`、`\r\n` 等）分隔消息。这是编程时最容易踩坑的差异，见 4.3。

**硬件与成本**：GPIB 需要接口卡 / USB-GPIB 适配器，线缆和卡相对贵；串口可用主板自带 COM 口或几元钱的 USB 转串口线，成本低。

### 4.3 对 C# / VISA 编程的影响

在 VISA 统一抽象下，GPIB 与串口的代码几乎一致，差异集中在**打开前的参数**和**终止字符**：

- **串口必须启用读取终止字符**，否则 `ReadLine` 会一直等到超时；GPIB 靠 EOI 线，通常不需要（设了也无害）。

  ```csharp
  session.TerminationCharacterEnabled = true;   // 串口 / socket 必设
  ```

- **串口要配波特率、数据位、校验、停止位**（如 9600/8/N/1），这些通过 VISA 运行时的串口配置（NI-MAX / Keysight Connection Expert）或会话属性设置；GPIB 没有这些参数。
- **地址格式不同**：串口是 `ASRL<端口号>::INSTR`（如 `ASRL3::INSTR`），GPIB 是 `GPIB<接口号>::<地址>::INSTR`（如 `GPIB0::22::INSTR`），详见 5.1 的地址格式表。
- 两者都属**消息型会话**，都用 `IMessageBasedSession` + `MessageBasedFormattedIO` 收发 SCPI，`*IDN?` 等命令完全通用。
- 都建议设置 `session.TimeoutMilliseconds`，避免仪器无应答时卡死。

> 小结：**协议（SCPI）不变，变的是“怎么传”**。用 VISA 时换接口主要就是换地址字符串，串口再多一步终止字符设置；若不用 VISA 直接编程，串口用 `SerialPort`（自己处理终止符与超时，见 5.2），而 GPIB 一般只能经厂商库或 VISA。

### 4.4 选型建议

- **多台仪器、要同步触发、数据量大**：优先 GPIB（或用 LAN）。
- **单台台式仪器、带宽要求低、想低成本**：串口即可。
- **距离较远或需要组网**：两者都不合适，选 LAN（`TCPIP` 的 socket 或 VXI-11）。
- **不确定用哪种**：先看仪器背板有哪些接口；只要有 LAN，通常优先 LAN，其次 USB，再次 GPIB / 串口。

---

## 5. C# 交互方式

### 5.1 方案 A：VISA.NET（推荐，仓库采用）

**VISA**（Virtual Instrument Software Architecture）是仪器通信的统一 API。C# 通过 **VISA.NET**（`Ivi.Visa` 程序集）访问仪器。这是本仓库示例采用的方式。

**前置条件**：

1. 安装厂商的 VISA 运行时：Keysight IO Libraries Suite 或 NI-VISA（提供 `GlobalResourceManager` 底层实现）。
2. 工程引用 `Ivi.Visa`（VISA.NET 程序集，或对应 NuGet 包）。

**通用流程**（见 `Socket_Example/Program.cs`）：

```csharp
using Ivi.Visa;
using Ivi.Visa.FormattedIO;

// 1. 打开会话（地址可以是 GPIB / TCPIP / USB / ASRL 等）
string address = "GPIB0::22::INSTR";   // 例：GPIB 地址 22 的仪器
IMessageBasedSession session =
    GlobalResourceManager.Open(address) as IMessageBasedSession;

// 2. 串口/socket 必须启用读取终止字符，否则读取会超时
session.TerminationCharacterEnabled = true;

// 3. 设置超时，避免无应答时程序卡死
session.TimeoutMilliseconds = 3000;

// 4. 用格式化 IO 收发 SCPI
MessageBasedFormattedIO io = new MessageBasedFormattedIO(session);

io.WriteLine("*IDN?");                    // 发查询
string idn = io.ReadLine();               // 读回响应

io.WriteLine("MEAS:VOLT:DC?");            // 发测量命令
string value = io.ReadLine();             // 读回测量值

// 5. 关闭会话
session.Dispose();
```

**查找设备**（见 `FindDevices/Program.cs`）：

```csharp
// 列出所有仪器
foreach (string dev in GlobalResourceManager.Find())
    Console.WriteLine(dev);

// 按类型查找
GlobalResourceManager.Find("GPIB?*INSTR");   // GPIB 仪器
GlobalResourceManager.Find("TCPIP?*SOCKET"); // LAN socket 仪器
GlobalResourceManager.Find("USB?*INSTR");    // USB 仪器
GlobalResourceManager.Find("ASRL?*INSTR");   // 串口仪器
```

**常见 VISA 地址格式**：

| 接口 | 地址格式示例 |
| --- | --- |
| GPIB | `GPIB0::22::INSTR`（0 为接口号，22 为仪器地址） |
| LAN（socket） | `TCPIP0::192.168.1.50::5025::SOCKET` |
| LAN（VXI-11） | `TCPIP0::192.168.1.50::inst0::INSTR` |
| USB | `USB0::0x2A8D::0x1301::MY12345678::INSTR` |
| 串口（RS-232） | `ASRL1::INSTR` |

### 5.2 方案 B：不依赖 VISA（.NET 原生）

若不想安装 VISA，可对支持 LAN 或串口的仪器直接用 .NET 内置类收发 SCPI 文本。

**TCP/IP Socket**：

```csharp
using System.Net.Sockets;
using System.Text;

using var client = new TcpClient("192.168.1.50", 5025); // 仪器 SCPI 端口
using var stream = client.GetStream();

void Write(string scpi) =>
    stream.Write(Encoding.ASCII.GetBytes(scpi + "\n"));

string ReadLine()
{
    var sb = new StringBuilder();
    int b;
    while ((b = stream.ReadByte()) >= 0 && b != '\n') sb.Append((char)b);
    return sb.ToString().Trim();
}

Write("*IDN?");
Console.WriteLine(ReadLine());
```

**串口**（`System.IO.Ports.SerialPort`）：

```csharp
using System.IO.Ports;
var port = new SerialPort("COM3", 9600, Parity.None, 8, StopBits.One) { NewLine = "\n" };
port.Open();
port.WriteLine("*IDN?");
Console.WriteLine(port.ReadLine());
```

> 说明：方案 B 需要自己处理命令结束符（一般 `\n`）、超时和缓冲区，通用性和健壮性不如 VISA，但胜在零依赖、部署简单。

### 5.3 方案 C：厂商 .NET 驱动 / IVI.NET

- **厂商 .NET 驱动**：部分仪器提供官方 .NET 驱动库（如 Keysight、NI 的仪器驱动），封装了专用 API，无需手写 SCPI。缺点是**只支持特定型号**，换仪器要换驱动。
- **IVI.NET**：若追求"换同类型仪器不改代码"，可选用符合 IVI 标准的驱动（基于 `Ivi.Driver`）。它在 VISA.NET 之上再抽象一层，代价是需要相应仪器/类别已有 IVI 驱动支持。

---

## 6. 常见问题

| 问题 | 原因 / 解决 |
| --- | --- |
| 读取超时 | 串口/socket 未启用 `TerminationCharacterEnabled`；或仪器无应答 |
| `GlobalResourceManager.Open` 报错 | 未安装 VISA 运行时，或地址/接口号错误 |
| "类型初始值设定项引发异常" | 本机未安装 VISA 运行时（NI-VISA / Keysight IO Libraries Suite） |
| GPIB 找不到仪器 | 仪器地址不对，或未安装 GPIB 卡驱动 |
| 命令无响应 | 命令拼写/大小写不符合 SCPI 规范，或缺少结尾换行符 |
| 查询读不到数据 | 查询命令必须以 `?` 结尾 |
| 换同类型仪器就要改代码 | 未使用 IVI 驱动；直接用 SCPI/VISA 时各型号命令不同属正常 |

---

## 附：相关文件索引

| 文件 | 说明 |
| --- | --- |
| `02-Source/UnitTest/是德Keysight/CSharp/VISA.NET/GettingStarted/Program.cs` | VISA 连接、`*IDN?`、超时保护 |
| `02-Source/UnitTest/是德Keysight/CSharp/VISA.NET/FindDevices/Program.cs` | 各类接口（含 GPIB）设备查找 |
| `02-Source/UnitTest/是德Keysight/CSharp/VISA.NET/Socket_Example/Program.cs` | LAN socket + 电压/电流/电阻 SCPI 测量 |
| `02-Source/UnitTest/是德Keysight/CSharp/VISA.NET/Serial_Example/Program.cs` | 串口 SCPI 通信 |
| `02-Source/UnitTest/是德Keysight/CSharp/VISA.NET/ErrorHandling/Program.cs` | VISA 异常处理示例 |
| `NI/VisaNetDemo/Program.cs` | 通用 VISA.NET 可交互示例（查找、连接、SCPI 收发） |