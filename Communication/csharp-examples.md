# C# 仪器通信示例代码

本文提供几种常见仪器通信方式的 C# 示例，便于在实际项目中快速起步。

> 说明：以下代码偏向演示用途，实际工程中请根据仪器协议、超时、异常处理和线程模型进一步封装。

---

## 1. 通用 SCPI 命令封装

很多仪器都支持 SCPI（Standard Commands for Programmable Instruments）。

```csharp
using System;
using System.IO;
using System.Text;

public sealed class ScpiInstrumentClient : IDisposable
{
    private readonly Stream _stream;
    private readonly Encoding _encoding = Encoding.ASCII;

    public ScpiInstrumentClient(Stream stream)
    {
        _stream = stream;
    }

    public void Write(string command)
    {
        var data = _encoding.GetBytes(command + "\n");
        _stream.Write(data, 0, data.Length);
        _stream.Flush();
    }

    public string ReadLine()
    {
        using var ms = new MemoryStream();
        var buffer = new byte[1];
        while (true)
        {
            var read = _stream.Read(buffer, 0, 1);
            if (read == 0) break;
            if (buffer[0] == (byte)'\n') break;
            if (buffer[0] != (byte)'\r') ms.WriteByte(buffer[0]);
        }
        return _encoding.GetString(ms.ToArray());
    }

    public string Query(string command)
    {
        Write(command);
        return ReadLine();
    }

    public void Dispose()
    {
        _stream.Dispose();
    }
}
```

---

## 2. 串口通信示例

C# 自带 `SerialPort`，适合 RS-232、RS-485、虚拟串口。

```csharp
using System;
using System.IO.Ports;

public class SerialPortExample
{
    public static void Main()
    {
        using var port = new SerialPort("COM3", 9600, Parity.None, 8, StopBits.One)
        {
            Handshake = Handshake.None,
            ReadTimeout = 2000,
            WriteTimeout = 2000,
            NewLine = "\n"
        };

        port.Open();
        port.WriteLine("*IDN?");
        string response = port.ReadLine();
        Console.WriteLine($"IDN = {response}");
        port.Close();
    }
}
```

### 典型说明
- `ReadLine()` 适合仪器返回文本行
- 若设备返回二进制数据，可改为 `Read(byte[], ...)`
- 建议对每个命令添加超时和重试

---

## 3. TCP/IP（LAN）通信示例

适合支持 Raw Socket、SCPI over TCP/IP 的仪器。

```csharp
using System;
using System.Net.Sockets;
using System.Text;

public class TcpInstrumentExample
{
    public static void Main()
    {
        using var client = new TcpClient();
        client.Connect("192.168.1.100", 5025);

        using NetworkStream stream = client.GetStream();
        var encoding = Encoding.ASCII;

        void Send(string cmd)
        {
            var bytes = encoding.GetBytes(cmd + "\n");
            stream.Write(bytes, 0, bytes.Length);
            stream.Flush();
        }

        string ReadLine()
        {
            var buffer = new byte[1];
            using var ms = new System.IO.MemoryStream();
            while (true)
            {
                int read = stream.Read(buffer, 0, 1);
                if (read == 0) break;
                if (buffer[0] == (byte)'\n') break;
                if (buffer[0] != (byte)'\r') ms.WriteByte(buffer[0]);
            }
            return encoding.GetString(ms.ToArray());
        }

        Send("*IDN?");
        var idn = ReadLine();
        Console.WriteLine(idn);
    }
}
```

### 适用场景
- 台式仪器的 SCPI socket
- 远程实验室
- 需要长距离控制的系统

---

## 4. UDP 通信示例

适合设备广播状态、发现服务或特定厂商协议。

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;

public class UdpInstrumentExample
{
    public static void Main()
    {
        using var udp = new UdpClient();
        udp.Connect("192.168.1.100", 6000);

        var request = Encoding.ASCII.GetBytes("STATUS?");
        udp.Send(request, request.Length);

        IPEndPoint remote = null;
        var response = udp.Receive(ref remote);
        Console.WriteLine(Encoding.ASCII.GetString(response));
    }
}
```

---

## 5. VISA 调用示例（思路版）

不同厂商的 VISA .NET API 名称不完全相同，但基本思路一致：

```csharp
// 伪代码 / 结构示例，具体类名请以厂商 SDK 为准
using System;

public class VisaExample
{
    public static void Main()
    {
        // 1. 打开资源管理器
        // 2. 打开资源：GPIB0::22::INSTR / TCPIP0::192.168.1.100::inst0::INSTR / USB0::...
        // 3. 写命令、读响应
        // 4. 关闭资源

        Console.WriteLine("Use your VISA SDK here.");
    }
}
```

### VISA 资源示例
- GPIB：`GPIB0::22::INSTR`
- 串口：`ASRL3::INSTR`
- LAN：`TCPIP0::192.168.1.100::inst0::INSTR`
- USB：`USB0::0x1234::0x5678::INSTR`

---

## 6. USB 通信示例思路

USB 仪器通常有几种路径：

1. USB-TMC：通过 VISA 访问
2. USB CDC：按串口处理
3. HID：通过 `HidSharp` 或 Windows API
4. WinUSB：通过厂商 SDK 或自定义封装

### USB CDC 示例
如果设备枚举成虚拟串口，代码和串口示例相同。

### HID 示例思路
```csharp
// 示例仅展示思路：实际建议使用第三方库或厂商 SDK
// 例如 HidSharp、HidLibrary 等
```

---

## 7. 一个更实用的包装方式

建议为项目封装统一接口：

```csharp
public interface IInstrumentConnection : IDisposable
{
    void Open();
    void Close();
    void Write(string command);
    string Read();
    string Query(string command);
}
```

然后分别实现：
- `SerialInstrumentConnection`
- `TcpInstrumentConnection`
- `VisaInstrumentConnection`
- `UsbInstrumentConnection`

这样上层业务就不需要关心底层总线类型。

---

## 8. 示例：查询设备身份并读取测量值

```csharp
public static class ScpiWorkflow
{
    public static void Run(IInstrumentConnection conn)
    {
        conn.Open();
        var idn = conn.Query("*IDN?");
        Console.WriteLine($"IDN: {idn}");

        var voltage = conn.Query("MEAS:VOLT?");
        Console.WriteLine($"Voltage: {voltage}");

        conn.Close();
    }
}
```

---

## 9. 实战建议

- 优先明确设备是否支持 SCPI
- 统一加超时和重试机制
- 命令发送和响应读取都记录日志
- 对二进制数据与文本数据分开处理
- 把“通信层”和“业务层”解耦

---

## 10. 小结

对于 C# 仪器通信，最推荐的工程化思路是：

- 能用 VISA 时优先用 VISA
- 串口设备直接用 `SerialPort`
- LAN 场景优先 TCP/IP + SCPI
- USB 按设备类型选择 USB-TMC / CDC / HID / WinUSB

只要把通信层封装好，后续更换仪器型号通常只需要改适配层，不必重写业务逻辑。
