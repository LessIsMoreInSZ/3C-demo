# VISA.NET 示例说明（VisaNetDemo）

本文档介绍本目录下的 **VISA.NET 示例程序 `VisaNetDemo`**：用 C# + VISA.NET（`Ivi.Visa` 程序集）与任意支持 SCPI 的仪器通信。示例不绑定具体仪器型号，核心流程是 **查找资源 → 打开会话 → SCPI 收发 → 关闭会话**。

> 相关背景知识（SCPI 命令语言、GPIB 总线、VISA 抽象层）见同目录 `SCPI与GPIB介绍及C#交互.md`；本仓库既有的 Keysight 官方示例见 `../是德Keysight/CSharp/VISA.NET/`。

---

## 1. 目录结构

```
NI/
├─ VisaNetDemo/                 示例工程（.NET Framework 4.8 控制台）
│  ├─ VisaNetDemo.sln           解决方案（可用 VS 直接打开）
│  ├─ VisaNetDemo.csproj
│  ├─ Program.cs                全部示例代码
│  ├─ Properties/AssemblyInfo.cs
│  └─ lib/Ivi.Visa.dll          VISA.NET 程序集（5.11.3422.0，随工程附带）
└─ VISA.NET示例说明.md           本文档
```

---

## 2. 前置条件

| 项目 | 说明 |
| --- | --- |
| VISA 运行时 | 必须安装 **NI-VISA** 或 **Keysight IO Libraries Suite**，它提供 `GlobalResourceManager` 的底层实现和仪器驱动 |
| 程序集 | 工程已引用 `lib\Ivi.Visa.dll`（IVI Foundation 的 VISA.NET 共享组件，版本 5.11.3422.0） |
| 框架 | .NET Framework 4.8（与本仓库其它示例一致） |
| 开发环境 | Visual Studio 2022，或 MSBuild 命令行 |

> 注意：`Ivi.Visa.dll` 只是**托管封装**，真正的通信能力来自本机安装的 VISA 运行时。若未安装运行时，程序会在访问 `GlobalResourceManager` 时报“类型初始值设定项引发异常”。

---

## 3. 编译与运行

### 3.1 用 Visual Studio

打开 `VisaNetDemo\VisaNetDemo.sln`，直接 F5 运行。可在 **项目属性 → 调试 → 命令行参数** 里填写地址或 `find`。

### 3.2 用命令行（MSBuild）

```bat
:: 编译（在 VisaNetDemo 目录下）
msbuild VisaNetDemo.sln -p:Configuration=Debug

:: 运行：列出当前可见的 VISA 资源
bin\AnyCPU\Debug\VisaNetDemo.exe find

:: 运行：用默认地址 GPIB0::22::INSTR 连接
bin\AnyCPU\Debug\VisaNetDemo.exe

:: 运行：用指定地址连接
bin\AnyCPU\Debug\VisaNetDemo.exe TCPIP0::192.168.1.50::5025::SOCKET
```

### 3.3 程序行为

1. 打印 `== 打开会话：<地址> ==`；
2. 发送 `*IDN?` 并回显仪器标识，确认链路正常；
3. 发送 `*CLS` 清状态；
4. 进入**交互模式**：输入任意 SCPI 命令回车发送，以 `?` 结尾的命令会读回响应；输入 `exit` 或直接回车退出。

```
== 打开会话：GPIB0::22::INSTR ==
*IDN? -> KEYSIGHT TECHNOLOGIES,34465A,MY12345678,A.03.00

输入 SCPI 命令回车发送；以 ? 结尾的命令会读回响应。
输入 exit 或直接回车退出。
SCPI> MEAS:VOLT:DC?
  <- +1.23456789E+01
SCPI> exit
```

---

## 4. 核心 API 速查

| API | 作用 |
| --- | --- |
| `GlobalResourceManager.Find()` | 列出所有可见的 VISA 资源地址 |
| `GlobalResourceManager.Find("GPIB?*INSTR")` | 按类型/接口查找资源（支持通配符） |
| `GlobalResourceManager.Parse(addr).AliasIfExists` | 取资源的别名（没有则为空） |
| `GlobalResourceManager.Open(addr)` | 打开会话，返回 `ISession`；仪器需强转为 `IMessageBasedSession` |
| `session.ResourceName` | 会话对应的资源地址 |
| `session.TimeoutMilliseconds` | 读写超时（毫秒），防止仪器无应答时卡死 |
| `session.TerminationCharacterEnabled` | 串口 / socket 必须置 `true`，否则 `ReadLine` 会超时 |
| `new MessageBasedFormattedIO(session)` | 格式化 I/O 对象，提供 `WriteLine` / `ReadLine` 等行级收发 |
| `io.WriteLine("*IDN?")` / `io.ReadLine()` | 发送命令 / 读回响应 |
| `session.Dispose()` | 关闭会话、释放资源 |

### 常见 VISA 地址格式

| 接口 | 地址格式示例 |
| --- | --- |
| GPIB | `GPIB0::22::INSTR`（0 为接口号，22 为仪器地址） |
| LAN（socket） | `TCPIP0::192.168.1.50::5025::SOCKET` |
| LAN（VXI-11） | `TCPIP0::192.168.1.50::inst0::INSTR` |
| USB | `USB0::0x2A8D::0x1301::MY12345678::INSTR` |
| 串口（RS-232） | `ASRL1::INSTR` |

---

## 5. 常用 SCPI 命令（以数字万用表为例）

| 命令 | 作用 |
| --- | --- |
| `*IDN?` | 查询仪器标识 |
| `*RST` / `*CLS` | 复位到出厂状态 / 清状态寄存器 |
| `MEAS:VOLT:DC?` | 测直流电压（单位 V） |
| `MEAS:VOLT:AC?` | 测交流电压 |
| `MEAS:CURR:DC?` / `MEAS:CURR:AC?` | 测直流 / 交流电流（单位 A） |
| `MEAS:RES?` / `MEAS:FRES?` | 测二线 / 四线电阻（单位 Ω） |
| `CONF:VOLT:DC 10` + `READ?` | 先配置量程再触发读取（适合固定量程场景） |

> `MEASure:...?` 是“配置 + 触发 + 读取”一步到位；其它仪器的命令不同，请查对应型号的 SCPI 编程手册。

---

## 6. 与仓库既有示例的关系

| 位置 | 说明 |
| --- | --- |
| `NI/VisaNetDemo/`（本示例） | 通用 VISA.NET 演示：查找、连接、`*IDN?`、交互式 SCPI 收发、异常与超时处理 |
| `是德Keysight/CSharp/VISA.NET/GettingStarted` | 最小连接示例（`*IDN?`） |
| `是德Keysight/CSharp/VISA.NET/FindDevices` | 各类接口（含 GPIB）设备查找 |
| `是德Keysight/CSharp/VISA.NET/Socket_Example` | LAN socket + 电压/电流/电阻测量 |
| `是德Keysight/CSharp/VISA.NET/Serial_Example` | 串口 SCPI 通信 |
| `是德Keysight/CSharp/VISA.NET/ErrorHandling` | VISA 异常处理 |

本示例把上述要点整合成一个可交互的命令行工具，可直接用于现场调试仪器。

---

## 7. 常见问题

| 现象 | 原因 / 解决 |
| --- | --- |
| “类型初始值设定项引发异常” | 本机未安装 VISA 运行时（NI-VISA / Keysight IO Libraries Suite），安装后即可 |
| `GlobalResourceManager.Open` 报错 | 地址或接口号错误；或该地址被其它程序占用 |
| 读取超时 | 串口 / socket 未启用 `TerminationCharacterEnabled`；或仪器无应答 |
| GPIB 找不到仪器 | 仪器 GPIB 地址不对，或未安装 GPIB 卡驱动 |
| 查询读不到数据 | 命令必须以 `?` 结尾；或命令拼写 / 大小写不符合 SCPI 规范 |
| 中文显示乱码 | 控制台代码页问题；程序已设置 `Console.OutputEncoding = UTF8`，在旧终端可执行 `chcp 65001` |