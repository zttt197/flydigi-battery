# 飞智八爪鱼手柄电量显示器 · Windows 实现方案

> 交付对象：Windows 端 agent。本文自包含，不需要任何前置对话。
> 需求原文：「给飞智八爪鱼手柄做电量显示，只需要显示电量即可。」
> 协议来源：两套独立实现交叉验证（SDL 的 `SDL_hidapi_flydigi.c` 与 Linux 内核 `hid-flydigi` 补丁），
> 电量字节偏移、状态码、换算系数完全一致。见文末「参考资料」。

---

## 0. 给 agent 的十行摘要

1. 不要做驱动，不需要管理员权限：走 **HID 厂商接口**（usage page `0xFFA0`），用户态收发报文即可。
2. 枚举 VID `0x37D7`（新协议）/ `0x04B4`+PID `0x2412`（老协议 Apex2/3 早期）。
3. 发一条 32 字节输出报告：`RR 5A A5 01 02 00 00…`，`RR` = 输出 Report ID（可能是 `0x03`/`0x21`/`0x00`，需探测）。
4. 读 32 字节输入报告，若首字节不是 `0x5A` 说明前面有 Report ID，剥掉一字节。
5. 报文形态必须是 `5A A5 01 …`（`01` = GET_INFO 回复）。
6. **电量在第 11 字节**：高半字节 = 状态（0 放电 / 1 充电 / 2 充满 / 其他 未知），低半字节 = 档位。
7. 百分比 = 档位 × 20；八爪鱼 6（Apex 6）为档位 × 10；状态 2 直接 100%；最后 clamp 0–100。
8. UI：托盘图标画数字 + 悬停 tooltip + 右键菜单，常驻后台，30 秒轮询一次。
9. 第一步先做 `--diag` 十六进制诊断模式，把真实报文抓出来校准（本方案未能实测，见 §7 已知风险）。
10. 验收：断电/插拔/充电/满电四种状态切换，托盘数字与手柄本体显示一致（±20% 档位内）。

---

## 1. 目标与非目标

### 目标（P0）
- 手柄以 **2.4G 接收器** 或 **USB 线** 连接时，显示剩余电量百分比。
- 区分「放电中 / 充电中 / 已充满」三种状态。
- 断开连接时明确显示「未连接」，恢复连接后自动重连，无需重启软件。
- 极低资源占用：常驻托盘，无主窗口，轮询间隔 30–60 秒。

### 非目标（明确不做）
- 不做按键映射、不做宏、不做灯效、不做震动调节（这些需要「接管手柄」语义，复杂度完全不同）。
- 不修改固件、不装内核驱动、不 hook 游戏进程。
- 不追求 1% 精度：**手柄本身只上报 5 档（20% 一步）**，UI 必须诚实显示档位，不要伪造 `87%` 这种数字。

---

## 2. 硬件与设备识别

### 2.1 USB ID

| VID | PID | 协议 | 备注 |
|---|---|---|---|
| `0x37D7` | `0x2501` | V2 | Apex 4 / Apex 5 主 PID（靠 deviceID 字段区分具体型号） |
| `0x37D7` | `0x2401` | V2 | Vader 5 Pro 等，使用**未编号报告**（Report ID 0） |
| `0x37D7` | `0x2502` | V2 | Apex 6，使用**未编号报告**，电量按 10% 档位 |
| `0x04B4` | `0x2412` | V1 | 老控制器（Apex 2/3 早期、Vader 2/3），厂商协议在 **interface 2** |

**重要**：不要靠 PID 猜型号。型号从 GET_INFO 回复的第 5 字节 `deviceID` 读（详见 §3.5）。

### 2.2 设备枚举形态

手柄在 Windows 上会同时出现多个 HID 集合（XInput 手柄、厂商配置接口、消费类按键）。需要挑出厂商接口：

- 优先筛选 `usage_page == 0xFFA0`（厂商自定义页）。
- 该集合的特征：输入报告 32 字节、输出报告 32 字节（HID report descriptor 里可见 `0x95 0x20 … 0x81` 与 `… 0x91`）。
- 若拿不到 usage_page，就遍历同 VID 的所有集合逐个探测（发请求、等回复），第一个回 `5A A5` 的就是。探测失败要能快速超时（每个 ≤ 300ms），不要卡住 UI。

> 有线连接与 2.4G 接收器的协议**完全一致**，只是 USB 描述符里的名称不同。蓝牙模式下该厂商集合通常不存在 → 走 §5.3 降级策略。

---

## 3. 协议规格（核心）

### 3.1 传输层

- 接口：HID，**输出报告（Output Report）+ 输入报告（Input Report）**，不用 Feature Report。
- 报文定长 **32 字节**。写入时用 32 字节、尾部补 0（参考内核驱动实现；SDL 只发 6 字节也能工作，但 32 字节兼容性更好）。
- 设备**同时**在此接口上持续推送游戏输入报告（`… EF …`，500–1000 Hz）。因此读取循环必须**过滤**，不能假设「下一条就是回复」。

### 3.2 请求：GET_INFO

```
裸字节（Report ID = 0x03 的情形）:
03 5A A5 01 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
│  │  │  │  └──┴─ 固定参数 0x02 0x00
│  │  │  └──────── 命令字 0x01 = GET_INFO
│  │  └─────────── magic 0xA5
│  └────────────── magic 0x5A
└───────────────── 输出 Report ID
```

- Report ID 有固件差异，三种取值：`0x03`、`0x21`、`0x00`（无编号报告）。
  - HID report descriptor 里出现序列 `85 21 95 1F` → Report ID 为 `0x21`。
  - `0x2401` / `0x2502` 这两类设备 SDL 使用 `0x00`（把 `0x03` 置零）。
  - 其余按 `0x03` 优先。
- 实现上做**探测**：依次尝试 `[descriptor 推断值, 0x03, 0x21, 0x00]`，每个尝试发 3 次、每次等 200ms，收到合法回复就锁定该 ID，本次运行期不再变。

### 3.3 回复：GET_INFO

原始报告 32 字节。**先归一化**：

```
if (raw[0] == 0x5A && raw[1] == 0xA5)  → payload = raw        // 未编号报告
else                                    → payload = raw[1..]  // 剥掉 1 字节 Report ID
校验: payload[0] == 0x5A && payload[1] == 0xA5 && payload[2] == 0x01
```

归一化后的字段表（`payload[i]`）：

| 偏移 | 含义 | 说明 |
|---|---|---|
| 0, 1 | magic | `5A A5` |
| 2 | 命令字 | `0x01` = GET_INFO 回复 |
| 3, 4 | 保留 | 忽略 |
| 5 | **deviceID** | 型号，见 §3.5 |
| 6 | 连接方式 | 有线/无线；新老架构取值不同（见下） |
| 11 | **电量字节** | 见 §3.4，本方案唯一关心的字段 |
| 15, 16 | 固件版本 | 小端：`fw = payload[15] | payload[16] << 8` |
| 其余 | 序列号等 | 忽略 |

> 连接方式：新架构 `0 = 有线, 1 = 无线`；老架构 `1 = 有线, 2 = 无线`。是否新架构由另一条命令（`0x07` CHECK_ARCHITECTURE）判定。**电量显示不需要它**——只在 tooltip 里想写「有线/2.4G」时才需要，建议先不做，或直接显示设备名。

### 3.4 电量字节解析（唯一需要精确实现的部分）

```csharp
byte b      = payload[11];
int  status = (b >> 4) & 0x0F;   // 0=放电, 1=充电, 2=充满, 其他=未知
int  level  = b & 0x0F;          // 0..5（Apex6 为 0..10）

int percent =
      status == 2                ? 100
    : status == 0 || status == 1 ? level * (isApex6 ? 10 : 20)
    :                              0;   // 未知状态
percent = Math.Clamp(percent, 0, 100);
```

状态文案：`0 → 使用中`、`1 → 充电中 ⚡`、`2 → 已充满`、`其他 → 未知`。

### 3.5 型号表（deviceID → 中文名）

用于 tooltip 与「是否 Apex6（10% 档位）」判定。

| deviceID (十进制) | 型号 |
|---|---|
| 19 | 八爪鱼 2 / Apex 2 |
| 24, 26, 29 | 八爪鱼 3 / Apex 3 |
| 84 | **八爪鱼 4 / Apex 4** |
| 128, 129, 133, 134 | 八爪鱼 5 / Apex 5 |
| 149, 150, 152 | 八爪鱼 6 / Apex 6（含 Pro / 幻影）**→ 10% 档位** |
| 20, 21, 23 | 黑武士 2 / Vader 2 |
| 22 | 黑武士 2 Pro |
| 28 | 黑武士 3 |
| 80, 81 | 黑武士 3 Pro |
| 85, 91, 105 | 黑武士 4 Pro |
| 130 | 黑武士 5 Pro |
| 其他 | 「飞智手柄」+ 原始 ID |

---

## 4. Windows 实现方案

### 4.1 技术选型

| 方案 | 结论 | 理由 |
|---|---|---|
| **C# / .NET 8 + WinForms 托盘 + HidApi.Net** | ✅ **推荐** | 单文件发布、无运行时依赖（`PublishSingleFile`）、托盘 API 最省事；HidApi.Net 自带 win/linux 原生 hidapi，`DeviceInfo` 直接给出 `UsagePage` 与 `Path` |
| C# + 手写 hidapi P/Invoke | ✅ 备选 | 只有 5 个函数，零第三方依赖（下方 §4.2 给了可用代码） |
| HidSharp | ⚠️ 可用但注意 | 纯托管，能读 report descriptor；但对无编号报告设备的 Write 语义要实测 |
| 直接调 `XInputGetBatteryInformation` | ❌ 不能作为主方案 | 只返回 EMPTY/LOW/MEDIUM/FULL 四档，且蓝牙/部分模式下没有；仅作降级 |
| Python + hidapi | ⚠️ 可行 | 分发麻烦（要 PyInstaller），不如 .NET 适合托盘常驻 |
| 复用飞智空间站内部接口 | ❌ 禁止 | 私有、随版本变、可能违反 EULA |

> 选 .NET 8 `net8.0-windows` + WinForms（不用 WPF，省掉托盘库），发布：
> `dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true`
> 体积约 150 KB（框架依赖）或 ~15 MB（self-contained）。

### 4.2 关键代码骨架

**（a）hidapi P/Invoke（x64 Windows 布局已验证）**

```csharp
using System.Runtime.InteropServices;

static class Hid
{
    const string Dll = "hidapi";

    [StructLayout(LayoutKind.Sequential)]
    public struct DeviceInfo
    {
        public IntPtr path;            // 0
        public ushort vendor_id;       // 8
        public ushort product_id;      // 10
        public IntPtr serial_number;   // 16
        public ushort release_number;  // 24
        public IntPtr manufacturer_string; // 32
        public IntPtr product_string;      // 40
        public ushort usage_page;      // 48
        public ushort usage;           // 50
        public int    interface_number;// 52
        public IntPtr next;            // 56
    }

    [DllImport(Dll)] public static extern int  hid_init();
    [DllImport(Dll)] public static extern IntPtr hid_enumerate(ushort vid, ushort pid);
    [DllImport(Dll)] public static extern void hid_free_enumeration(IntPtr list);
    [DllImport(Dll)] public static extern IntPtr hid_open_path([MarshalAs(UnmanagedType.LPUTF8Str)] string path);
    [DllImport(Dll)] public static extern void hid_close(IntPtr dev);
    [DllImport(Dll)] public static extern int  hid_write(IntPtr dev, byte[] data, UIntPtr len);
    [DllImport(Dll)] public static extern int  hid_read_timeout(IntPtr dev, byte[] data, UIntPtr len, int ms);
    [DllImport(Dll)] public static extern int  hid_set_nonblocking(IntPtr dev, int nonblock);
}
```

**（b）枚举 + 打开**

```csharp
sealed record Candidate(string Path, ushort UsagePage, ushort Vid, ushort Pid);

static List<Candidate> Enumerate()
{
    var list = new List<Candidate>();
    IntPtr head = Hid.hid_enumerate(0x37D7, 0);
    if (head == IntPtr.Zero)
        head = Hid.hid_enumerate(0x04B4, 0x2412);  // 老协议兜底
    try
    {
        for (IntPtr p = head; p != IntPtr.Zero; )
        {
            var di = Marshal.PtrToStructure<Hid.DeviceInfo>(p);
            var path = Marshal.PtrToStringUTF8(di.path) ?? "";
            // 厂商接口优先，其余留给探测
            list.Add(new Candidate(path, di.usage_page, di.vendor_id, di.product_id));
            p = di.next;
        }
    }
    finally { if (head != IntPtr.Zero) Hid.hid_free_enumeration(head); }

    return list
        .OrderByDescending(c => c.UsagePage == 0xFFA0)
        .ThenByDescending(c => c.UsagePage >= 0xFF00)
        .ToList();
}
```

**（c）发请求 + 收回复（含 Report ID 探测）**

```csharp
const byte MAGIC1 = 0x5A, MAGIC2 = 0xA5, CMD_INFO = 0x01;

static byte[] BuildInfoRequest(byte reportId)
{
    var buf = new byte[32];
    buf[0] = reportId;
    buf[1] = MAGIC1; buf[2] = MAGIC2; buf[3] = CMD_INFO;
    buf[4] = 0x02;   buf[5] = 0x00;
    return buf;
}

static bool TryReadInfo(IntPtr dev, out byte[] payload)
{
    payload = Array.Empty<byte>();
    var buf = new byte[64];
    var deadline = Environment.TickCount64 + 300;   // 300ms 内找回复
    while (Environment.TickCount64 < deadline)
    {
        int n = Hid.hid_read_timeout(dev, buf, (UIntPtr)buf.Length, 15);
        if (n <= 0) continue;

        Span<byte> raw = buf.AsSpan(0, n);
        // 归一化：未编号报告首字节直接是 0x5A
        Span<byte> p = (raw.Length >= 2 && raw[0] == MAGIC1 && raw[1] == MAGIC2)
                     ? raw
                     : (raw.Length > 1 ? raw[1..] : default);
        if (p.Length < 31) continue;
        if (p[0] != MAGIC1 || p[1] != MAGIC2 || p[2] != CMD_INFO) continue;  // 丢弃 0xEF 输入报告

        payload = p.ToArray();
        return true;
    }
    return false;
}

static byte DetectReportId(IntPtr dev)
{
    foreach (byte id in new byte[] { 0x03, 0x21, 0x00 })
    {
        for (int i = 0; i < 3; i++)
        {
            Hid.hid_write(dev, BuildInfoRequest(id), (UIntPtr)32);
            if (TryReadInfo(dev, out _)) return id;
        }
    }
    throw new IOException("no reply to GET_INFO");
}
```

**（d）解析**

```csharp
sealed record Battery(int Percent, ChargeStatus Status, int DeviceId, ushort Firmware, bool IsApex6);

enum ChargeStatus { Unknown, Discharging, Charging, Full }

static Battery ParseInfo(byte[] p)
{
    int deviceId = p[5];
    bool apex6 = deviceId is 149 or 150 or 152;
    int status = (p[11] >> 4) & 0x0F;
    int level  = p[11] & 0x0F;

    // 偏移自愈：万一少剥/多剥了一字节，用合法性校验兜住
    if (status > 2 || level > (apex6 ? 10 : 5))
    {
        foreach (int alt in new[] { 10, 12 })
        {
            if (alt >= p.Length) continue;
            int s2 = (p[alt] >> 4) & 0x0F, l2 = p[alt] & 0x0F;
            if (s2 <= 2 && l2 <= (apex6 ? 10 : 5)) { status = s2; level = l2; break; }
        }
    }

    var st = status switch { 0 => ChargeStatus.Discharging, 1 => ChargeStatus.Charging,
                             2 => ChargeStatus.Full, _ => ChargeStatus.Unknown };
    int pct = st switch
    {
        ChargeStatus.Full => 100,
        ChargeStatus.Discharging or ChargeStatus.Charging => level * (apex6 ? 10 : 20),
        _ => 0
    };
    return new Battery(Math.Clamp(pct, 0, 100), st, deviceId,
                       (ushort)(p[15] | (p[16] << 8)), apex6);
}
```

**（e）轮询与生命周期**

```
启动 → 枚举 → 打开 → 探测 Report ID → 循环:
   每 30s（充电中 15s）:
       write(GET_INFO) → read(≤300ms) → parse → 更新托盘
   打开失败/hid_read 返回 -1:
       关闭句柄 → 灰色图标「未连接」→ 每 2s 重试枚举（指数退避到 10s 上限）
```

要点：
- **保持句柄常开**，不要每次轮询都 open/close（某些固件的 open 有几百毫秒延迟）。
- 必须持续 `hid_read_timeout` 抽干输入报告，否则驱动缓冲区堆积、回复被挤掉。
- 所有 HID 调用包在 try/catch 里；设备拔出会返回 -1，不要抛到 UI 线程。

### 4.3 UI 规格（托盘）

| 元素 | 规格 |
|---|---|
| 图标 | 32×32（系统托盘自动缩放），深色圆角背景 + 白色百分比数字；充电时右下角加 ⚡ 角标 |
| 图标颜色 | ≥50% 绿 `#2ECC71`；20–49% 橙 `#F39C12`；<20% 红 `#E74C3C`；未连接 灰 `#7F8C8D` |
| Tooltip | `飞智八爪鱼4 · 80% · 使用中` / `… · 40% · 充电中` / `飞智手柄 · 未连接` |
| 右键菜单 | `立即刷新` / `电量 80%（充电中）`（禁用态信息项）/ `开机自启`（勾选）/ `诊断模式…` / `退出` |
| 数字渲染 | `Bitmap 32×32` → `Graphics.DrawString`（Segoe UI Bold 14）→ `Icon.FromHandle(bmp.GetHicon())`，**每次替换后必须 `DestroyIcon`**，否则 GDI 句柄泄漏（1 分钟级就会崩） |
| 双击 | 弹一个 5 秒自动消失的小气泡（`NotifyIcon.ShowBalloonTip`），可选 |
| 单实例 | `Mutex("Global\\FlydigiBatteryTray")`，第二个实例直接退出 |
| 自启 | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`，值是 exe 路径加 `--tray` |

**不要**做常驻悬浮窗（用户明确只需要「显示电量」）；如果后面要加，做成右键菜单可开的小窗。

### 4.4 热插拔与睡眠恢复

- 隐藏消息窗口 + `WM_DEVICECHANGE`（`DBT_DEVNODES_CHANGED`）→ 500ms 去抖 → 重新枚举。
- `SystemEvents.PowerModeChanged` → Resume 时强制刷新一次（部分固件休眠后需要重新 open）。
- 若 `WM_DEVICECHANGE` 收不到（WinForms 托盘应用常见），兜底用 30s 轮询即可，不要为此引入 WMI 订阅。

---

## 5. 坑与对策

| 现象 | 原因 | 对策 |
|---|---|---|
| 回复里找不到 `5A A5` | Report ID 猜错，设备没响应 | 探测 `0x03/0x21/0x00`；再不行抓 descriptor 里的 `85 xx` |
| 读到的报文偏移整体差 1 | 少剥了 Report ID | 归一化后再解析；用 §4.2(d) 的偏移自愈兜底 |
| 一直收到 `… EF …` | 输入报告洪流 | 循环读、按 `payload[2]` 过滤，别只读一次 |
| 电量数字抖动 | 手柄只报 5 档 | 缓存上一次值，只有变化才重绘；不要做平滑动画 |
| 显示 100% 但手柄显示 4 格 | 状态 `2`（充满）优先 | 状态 2 固定 100%，其余按档位 |
| 打开设备失败 `Access denied` | 飞智空间站/其他软件抢占 | 提示「请关闭飞智空间站后重试」；`hid_open_path` 是共享打开，一般能共存 |
| 蓝牙模式完全没反应 | 蓝牙不暴露厂商 HID 集合 | 降级：`XInputGetBatteryInformation` → 显示「低/中/满/满电」四档文字，tooltip 注明「蓝牙模式精度有限」 |
| 掉电重启后图标丢失 | 托盘重建失败 | 收到 `TaskbarCreated` 消息时重新 `NotifyIcon.Visible = true` |
| 安装后首启无反应 | 2.4G 接收器枚举慢（1–3s） | 启动时给 10 秒重试窗口，期间图标显示「连接中…」 |

---

## 6. 分阶段任务与验收

### P0 · 协议诊断（先做，产出真实抓包）
- CLI：`FlydigiBattery.exe --diag`
- 输出：枚举到的所有 HID 集合（path / vid / pid / usage_page）、探测到的 Report ID、每次请求的 32 字节 **hexdump**、解析结果。
- **验收**：能打印类似
  ```
  [0xFFA0] path=... vid=37D7 pid=2501
  report-id probe: 0x03 ok
  TX 03 5A A5 01 02 00 00 ...
  RX 03 5A A5 01 00 00 54 01 00 00 00 00 04 00 00 00 31 70 ...
  => deviceID=84 (Apex 4) fw=0x7031 status=0 level=4 percent=80%
  ```
- 把这段 hexdump 回传给上游校准（协议偏移以真实报文为准）。

### P1 · 解析层 + 单元测试
- `ParseInfo` 纯函数化，用 §8 的合成报文做 xUnit 测试（5 个用例必须全绿）。

### P2 · 托盘 UI
- 图标数字、颜色阈值、tooltip、右键菜单、单实例。
- **验收**：断网/拔接收器 → 3 秒内变灰「未连接」；插回 → 5 秒内恢复数字。

### P3 · 健壮性
- 热插拔、睡眠恢复、Access denied 提示、GDI 句柄不泄漏（任务管理器观察 30 分钟句柄数稳定）。

### P4 · 打包
- 单文件 exe + 图标 + `--autostart` 注册 + 可选 Inno Setup 安装包。
- **验收**：干净 Win10 22H2 / Win11 23H2 上双击即用，无 .NET 缺失弹窗（self-contained 发布）。

### 端到端验收矩阵

| 场景 | 期望 |
|---|---|
| 2.4G 接收器，电量 80% | 托盘 `80`，绿色，tooltip 含型号 |
| 插 USB 线充电，电量 40% | 托盘 `40`，橙色，⚡ 角标，tooltip「充电中」 |
| 充满 | 托盘 `100`，绿色，⚡ 保留，tooltip「已充满」 |
| 电量 <20% | 托盘 `20` 或 `0`，红色 |
| 拔掉接收器 | 灰色，`未连接`，无异常日志 |
| 插回接收器 | ≤5s 恢复读数 |
| 睡眠 8 小时后唤醒 | 仍能刷新（必要时自动 reopen） |
| 长跑 8 小时 | 内存/句柄无增长 |

---

## 7. 已知风险与不确定项（必须由实测消除）

1. **未在真实设备上验证**：本方案撰写环境没有手柄。协议字段来自两套独立开源实现，可信度高，但 **Report ID 与偏移必须用 P0 的 hexdump 实测确认**。
2. **Report ID 存在固件差异**：`0x03`（SDL 用于 Apex4/5）、`0x21`（内核驱动从 descriptor 推导，用于 Vader 5 Pro）、`0x00`（无编号，Apex6）。所以设计成运行时探测而非写死。
3. **电量精度上限 20%**：档位只有 5 档（Apex6 为 10 档）。UI 不要暗示更高精度。
4. **蓝牙模式**：厂商接口通常不存在，只能降级为四档文字。
5. **多手柄同时连接**：按 `path` 排序取第一个，或用「最近一次成功读取」的那个；本版只支持单手柄，UI 上不提。
6. **飞智空间站共存**：共享打开通常可行；若被独占，给出明确提示而不是静默失败。

---

## 8. 测试向量（合成报文，用于单元测试）

以下报文为**构造值**（非真实抓包），字段按 §3.3 布局填好，用于验证解析器逻辑：

| 用例 | 原始 32 字节（hex） | 期望 |
|---|---|---|
| 放电 80% · Apex4 (id 84) | `03 5A A5 01 00 00 54 01 00 00 00 00 04 00 00 00 31 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | status=0, level=4, **80%**, deviceID=84, fw=0x7031 |
| 充电 40% · Apex4 | `03 5A A5 01 00 00 54 01 00 00 00 00 12 00 00 00 31 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | status=1, level=2, **40%** |
| 充满 · Apex4 | `03 5A A5 01 00 00 54 01 00 00 00 00 20 00 00 00 31 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | status=2, **100%** |
| 放电 70% · Apex6 (id 149) | `03 5A A5 01 00 00 95 01 00 00 00 00 07 00 00 00 31 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | level=7 ×10 → **70%** |
| 未知状态（0xFF） | `03 5A A5 01 00 00 54 01 00 00 00 00 FF 00 00 00 31 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00` | status=15 → **0%，状态 Unknown** |

另需覆盖：
- 无 Report ID 版本（首字节即 `5A A5 01 …`，整体左移一位）必须解析出相同结果。
- 输入报告 `… EF …` 必须被丢弃且不影响下一次读取。
- 长度不足 31 字节的报文必须安全忽略（不越界）。

---

## 9. 参考资料

- SDL 飞智 HIDAPI 驱动（协议权威来源，含电量解析 `HandleInfoResponse`、Report ID 处理 `WritePacket`、报文归一化 `HandlePacketV2`）：
  <https://github.com/libsdl-org/SDL/blob/main/src/joystick/hidapi/SDL_hidapi_flydigi.c>
- SDL USB ID 定义：<https://github.com/libsdl-org/SDL/blob/main/src/joystick/usb_ids.h>
- Linux 内核 `hid-flydigi` 驱动补丁（独立实现同一协议，含 `flydigi_report_battery` 与 Report ID 从 descriptor 推导）：
  <https://patchew.org/linux/20260811125856.1692045-1-denis.benato@linux.dev/>
- hidapi 官方（Windows 预编译 `hidapi-win.zip`）：
  <https://github.com/libusb/hidapi/releases>
- NuGet：`HidApi.Net`（1.2.0，自带原生库）/ `HidSharp`（2.6.4，纯托管）

---

## 10. 一句话交接

> 先写 `--diag`，抓到真实 hexdump；确认字节 11 的 `status<<4 | level` 后，解析层就 20 行代码；
> 剩下的全是托盘 UI 和热插拔的工程细节。若实测偏移与本文不符，一律以真实抓包为准，并把结论回写到本文件。
