# EndfieldCharge macOS 移植评估方案

> 状态：**已实施并通过实测验证** · 版本：1.1 · 起始代码基线：`main` @ `9c438eb`
> 目标机型：Apple Silicon (arm64) / macOS 26+，次要目标 Intel Mac (x64)
>
> **实施期修正**（本文档已同步）：全部实测结论与踩坑见 §3 「已实测的环境事实」及各节
> 「实现期修正」标注。验证清单见 [macos-verification.md](./macos-verification.md)。

---

## 0. TL;DR

| 项 | 结论 |
|---|---|
| 移植可行性 | **高**。UI / 动画 / 设置 / 本地化共约 **78% 代码（2493 行中约 1950 行）零改动复用** |
| 真正需要重写的部分 | **4 个文件、约 700 行**：`PowerNative.cs`(284) / `PowerWatcher.cs`(353) / `AutoStart.cs`(61) / `TrayMenuWindow.axaml.cs` 的定位段(~20) |
| 最大技术风险 | macOS 电源事件通知（Windows 用 `RegisterPowerSettingNotification`；macOS 用 `IOPSNotificationCreateRunLoopSource` + 轮询兜底）—— **已实测可用，并有降级方案** |
| 已实测验证的关键假设 | Avalonia 11.2.1 在本机 macOS 27 / arm64 / .NET 10 完整启动；`NSWindow` 可从 Avalonia 拿到并改级别；IOKit 能读到真实 mAh 容量与设计容量 |
| 建议方案 | csproj 条件多目标 + `Platform/` 抽象层 + 自定义编译符号隔离；macOS 端 IOKit + libobjc 互操作 + LaunchAgent + `.app` bundle |
| 工作量估算 | **代码约 1300 行新增（`Platform/` 下 12 个文件）/ 约 700 行迁移**，含文档、打包脚本与 CI |
| 实施结果 | ✅ 两平台严格构建 0 警告；`.app` + `.dmg` 打包通过；`--selftest` 数值与系统逐字段一致；HUD `NSWindow` 层级/集合行为/透明背景三项增强实测生效（level=101 / collectionBehavior=273 / transparent=True）；**真机 7/7 次插拔全部检出，端到端延迟 233–403 ms**（目标 ≤1.5s）；**每次物理动作恰好一次 HUD**（`HudDisplayCoordinator` 合并副作用事件）；**macOS 刘海出生与分离前导**（`NotchBirthGeometry`；`safeBottom=33`、`offY=-59`、最终顶点 67.4、窄条 180×28）；**单测 34/34**（事件合并 10 + 出生几何 24） |

---

## 1. 目标与成功标准

### 1.1 目标
把现有 Windows-only 的 Avalonia 电量 HUD 移植为 **macOS 原生可用的菜单栏常驻应用**，同时**不破坏**原有 Windows 构建与发布链路（同一仓库、同一份 UI 代码）。

### 1.2 成功标准（可验收）
1. `dotnet publish -c Release -r osx-arm64` 产出 `.app`；双击启动后 Dock 无图标、菜单栏出现闪电图标、无多余窗口。
2. 插入/拔出电源后 **≤1.5s** 内弹出对应 HUD（插电 = 完整三态，拔电 = 简化胶囊），动画与视觉与 Windows 版一致。
3. HUD 数值正确：`剩余 mWh / 满充 mWh` 与百分比来自真实电池；`<20%` 时电量圈变红。
4. 菜单栏图标左键弹出菜单，四项（预览电量 / 设置 / 检查更新 / 退出）均可用。
5. 设置窗口可打开、可保存；`全局缩放 / 显示时长 / HUD 位置 / 显示器 / 语言 / 低电量提醒 / 充满提醒 / 低电量模式提示 / 登录时自启` 全部生效并持久化。
6. `--selftest` 在 macOS 输出电池/电源/低电量模式诊断 JSON，退出码 0。
7. Windows 侧 `dotnet build -c Release /p:TreatWarningsAsErrors=true`、`dotnet publish`、Inno Setup 打包仍全部通过。

### 1.3 非目标（本次不做）
- Avalonia 11 → 12 升级（另立事项）
- Windows 侧 TFM 迁到 net10（仅预留开关，见 §4.1）
- 通知中心集成（`UNUserNotificationCenter`）取代现有卡牌弹窗
- 代码签名证书采购与公证密钥的 CI 配置（仅提供脚本入口与文档）
- Linux 移植（抽象层已留位置，但不实现）
- 视觉/图标重设计

---

## 2. 现状盘点

### 2.1 工程概览

| 项 | 值 |
|---|---|
| 类型 | Avalonia 11.2.1 桌面应用，C# / .NET 8 |
| TFM | `net8.0-windows`，`PlatformTarget=x64`，`RuntimeIdentifiers=win-x64` |
| 发布 | `PublishSingleFile=true` / `SelfContained=false`（框架依赖，需 .NET 8 Desktop Runtime） |
| 装机 | Inno Setup (`installer/EndfieldCharge.iss`) + 便携 zip |
| CI | `.github/workflows/build.yml`，`runs-on: windows-latest` |
| 第三方依赖 | `Avalonia` / `Avalonia.Desktop` / `Avalonia.Themes.Fluent` / `Avalonia.Fonts.Inter` / `System.Management`(WMI 兜底) |
| 代码量 | 25 个源文件，2493 行（含 XAML） |

### 2.2 代码结构

```
EndfieldCharge/
├─ Program.cs                 30   入口、单实例 Mutex、Avalonia 配置
├─ App.axaml / .axaml.cs      18/408  应用主体：托盘、电源监听、提醒、设置分发
├─ Localization.cs           130   中英文案（运行时切换）
├─ Animations/HudAnimations.cs 372 时间线动画轨道（KeySpline 逐段缓动）
├─ Styles/{HudTheme,Geometries}.axaml 44/18  主题色、字体族、图标几何
├─ Views/HudWindow.{axaml,cs} 277/443  HUD 三态动画 + 多显示器定位
├─ Views/TrayMenuWindow.{axaml,cs} 56/111  自绘托盘菜单
├─ Settings/AppSettings.cs    36   设置模型（record）
├─ Settings/SettingsManager.cs 48  设置读写（ApplicationData/settings.json）
├─ Settings/SettingsWindow.{axaml,cs} 315/523  设置窗口（4 个 Tab）
├─ Services/BatteryService.cs 126  电池采样（powrprof 主 + WMI 兜底）
├─ Services/PowerNative.cs    284  Windows 电源 P/Invoke（整个文件）
├─ Services/PowerWatcher.cs   353  电源事件监听（消息循环 + 去抖）
├─ Services/AutoStart.cs       61  注册表开机自启
├─ Services/Logger.cs          37  文件日志（%TEMP%）
├─ Services/UpdateChecker.cs   66  GitHub Releases 版本比较
└─ app.manifest                18  Windows 清单（DPI、UAC）
```

### 2.3 Windows 强耦合点（逐项已读码核实）

| # | 位置 | 依赖 | 性质 |
|---|---|---|---|
| 1 | `EndfieldCharge.csproj` | `net8.0-windows`、`PlatformTarget=x64`、`RuntimeIdentifiers=win-x64`、`ApplicationManifest`、`ApplicationIcon=.ico`、`PublishSingleFile` | 构建期硬编码 |
| 2 | `Services/PowerNative.cs`（整文件） | `powrprof!CallNtPowerInformation`、`kernel32!GetSystemPowerStatus`、`user32!RegisterPowerSettingNotification` + 隐藏 message-only 窗口 (`RegisterClassExW`/`CreateWindowExW`/`GetMessageW`/`DispatchMessageW`) | **需整体平台化** |
| 3 | `Services/PowerWatcher.cs`（整文件） | Windows 消息循环线程 + `WM_POWERBROADCAST` / `PBT_POWERSETTINGCHANGE`；`Environment.OSVersion.Version.Build >= 26100` 分流新旧节能 GUID | **需整体平台化** |
| 4 | `Services/BatteryService.cs` | 主路径 `PowerNative`；兜底 `System.Management` WMI `Win32_Battery` | 需平台化（WMI → IOKit） |
| 5 | `Services/AutoStart.cs`（整文件） | `Microsoft.Win32.Registry` → `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` | **需整体平台化** |
| 6 | `Views/TrayMenuWindow.axaml.cs:64-66` | `user32!GetCursorPos` 把菜单定位到托盘图标上方 | macOS 失效（且 macOS 托盘 `Clicked` 根本不触发，见 §5.5） |
| 7 | `Program.cs:9,15` | 单实例 `Mutex(@"Local\EndfieldCharge_SingleInstance_7C1D")` —— .NET 在 Unix 上 named mutex 为进程内，**跨进程无效** | 需换 `FileStream.Lock` |
| 8 | `Services/PowerNative.cs:117-144` | 读 `HKLM\SYSTEM\CurrentControlSet\Control\Power\EnergySaverState` 与 `GetSystemPowerStatus.SystemStatusFlag` | 需平台化 |
| 9 | `Styles/HudTheme.axaml:39,42`、`SettingsWindow.axaml:14`、`SettingsWindow.axaml.cs:453`、`App.axaml.cs:236` | 字体栈中文兜底为 `Microsoft YaHei UI` | 需补 `PingFang SC` |
| 10 | `Localization.cs:63` | `"登录 Windows 时自动启动"` 硬写 Windows | 需平台分支 |
| 11 | `.github/workflows/build.yml` | `runs-on: windows-latest`、`iscc`、`.iss` | 需加 macOS job |
| 12 | — | 无平台抽象层；无单元测试；单实例失败即静默 return | 需新增 |

### 2.4 可零改动复用的部分

`Animations/HudAnimations.cs`、`Views/HudWindow.axaml`（含全部动画轨道定义）、`Styles/*`、`Settings/AppSettings.cs`、`Localization.cs` 主体、`Services/Logger.cs`、`Services/UpdateChecker.cs`、`App.axaml`。

`Views/HudWindow.axaml.cs` 与 `Settings/SettingsWindow.axaml.cs` 仅需各改动 1–2 处调用点（见 §7）。

---

## 3. 已实测的环境事实

> 以下数据全部由本次在本机（Apple Silicon / macOS 27.0 build 26A428）实际执行命令与探针程序获得，非文档推断。

### 3.1 工具链

| 项 | 实测结果 | 影响 |
|---|---|---|
| .NET SDK | `10.0.203`（唯一） | 需用 SDK 10 构建 |
| 共享运行时 | **仅 `Microsoft.NETCore.App 10.0.7`**，无 net8 运行时 | macOS 侧必须 `net10.0`，或 self-contained |
| 参考包缓存 | `microsoft.netcore.app.ref` 6/7/8/9 齐全；`microsoft.windowsdesktop.app.ref 8.0.26` | 现有工程在本机可 cross-compile |
| 现有工程本机构建 | `dotnet build -c Debug` **成功**（net8.0-windows 产物） | 但该产物在本机**无法运行**（缺 net8 运行时） |
| Xcode | Xcode 27.0 / macOS SDK 27.0 | 可签名、可公证 |
| 签名身份 | `Apple Development: 烨 刘 (793F7H68S8)`；**无 Developer ID** | 首版只能 ad-hoc / 本地签名分发 |
| Avalonia 版本 | 11.2.1（工程）；最新 **12.1.2**（2026-09-02） | 见 §4.3 决策 |

### 3.2 运行探针结果

用独立探针工程（`net10.0` / `osx-arm64` / Avalonia 11.2.1）实测：

```
PLATFORM-OK   lifetime=ClassicDesktopStyleApplicationLifetime
HANDLE-KIND=NSWindow   ptr=500625410560
SCREENS=1     primary WorkingArea=0, 33, 1512, 890   scaling=1
TIMER-OK      EXIT-CLEAN
```

| 探针项 | 结果 | 结论 |
|---|---|---|
| Avalonia 11.2.1 在 macOS 27 / arm64 / .NET 10 | 完整启动并干净退出 | ✅ 无需升级 Avalonia |
| macOS 原生库架构 | `libAvaloniaNative.dylib`、`libSkiaSharp.dylib` 均为 **x86_64 + arm64 通用二进制** | ✅ 双架构可用 |
| `TryGetPlatformHandle()` | 返回 descriptor = `"NSWindow"` 的裸指针 | ✅ 可做原生窗口增强 |
| `objc_msgSend` 调 `setLevel:`(→101) / `setCollectionBehavior:`(128→273) | 成功 | ✅ HUD 可置于菜单栏之上、跨 Space 可见 |
| `NSScreen.mainScreen` / `NSBundle.mainBundle` | 可取；非 bundle 运行时 `bundleIdentifier` 为 `null` | ✅ 可用于自启路径解析 |
| `NSProcessInfo.lowPowerModeEnabled` | **`unrecognized selector` → 抛 `NSInvalidArgumentException` 崩溃** | ⚠️ **禁止裸调**，必须 `respondsToSelector:` 守卫 |
| `NSProcessInfo.isLowPowerModeEnabled` | `respondsToSelector:` 返回 `True` | ✅ 有可用选择子 |
| `Screens.Primary.WorkingArea` | `(0, 33, 1512, 890)` —— 已排除 33pt 菜单栏/刘海行 | ✅ HUD 现有 `area.Y + 4` 定位逻辑在 mac 直接可用 |
| `TryGetPlatformHandle()` 的对象类别 | `object_getClassName` 返回 **`AvnWindow`** | ✅ 它是 NSWindow 的**子类**，可直接发 `setLevel:` / `setCollectionBehavior:`，**无需** superview 解包 |
| `kIOGeneralInterest` / `kIOMainPortDefault` 符号 | `dlsym` 返回 **NULL**（未导出） | ⚠️ 不能走 IOKit 私有通知；改用公开的 `IOPSNotificationCreateRunLoopSource` |
| `IOPSPowerSourceStateKey` 的语义 | 「已接电但未充电」时报 `Battery Power` | ⚠️ **不能用它判断是否插电**，必须用 `AppleSmartBattery.ExternalConnected` |
| `Amperage` / `InstantAmperage` / `BatteryPower` 的符号 | 实测 `18446744073709550344`（实为 −2092 mA） | ⚠️ IOKit 以**无符号 32 位**存储负值补码，必须还原符号，否则 `RateWatts` 会得到天文数字 |
| `TimeRemaining` 的可信度 | 插电转放电后仍**滞留在哨兵值 65535**，真实值在 `AvgTimeToEmpty` | ⚠️ 需双来源回退 |
| `NSProcessInfo.lowPowerModeEnabled` vs `isLowPowerModeEnabled` | 前者 `unrecognized selector`（崩溃），后者存在 | ⚠️ 必须 `respondsToSelector:` 守卫 |
| Avalonia 托盘 `TrayIcon.Clicked`（macOS） | 源码级确认：`AvnTrayIcon` 只实现 `setMenu:` | ❌ 永不触发 → macOS 必须用原生菜单 |

### 3.3 电池数据源（本机实测值）

`ioreg -rn AppleSmartBattery`：

| 键 | 实测值 | 含义 |
|---|---|---|
| `Voltage` | `12958` | mV |
| `BatteryData.FullChargeCapacity` | `5419` | 当前满充容量 mAh |
| `BatteryData.RemainingCapacity` | `5419` | 剩余容量 mAh |
| `BatteryData.DesignCapacity` | `6249` | 设计容量 mAh |
| `MaxCapacity` / `CurrentCapacity` | `100` / `100` | 百分比（整数） |
| `ExternalConnected` | `Yes` | 是否接交流电 |
| `IsCharging` | `No` | 是否正在充电 |
| `FullyCharged` | `Yes` | 是否已充满 |
| `Amperage` / `InstantAmperage` | `0` | mA，正=流入 |
| `TimeRemaining` | `65535` | 未知哨兵值 |
| `CycleCount` / `DesignCycleCount9C` | `371` / `1000` | 循环次数 |

**换算验证**：健康度 = `5419 / 6249 = 86.7%`，与 `system_profiler SPPowerDataType` 报的 `Maximum Capacity: 89%` 同源（差异来自 Apple 的标称取整），因此 macOS 侧**能算出比 Windows 侧更完整的健康度指标**（Windows 版主路径 powrprof 取不到设计容量）。

**关键结论**：macOS 的 mAh + mV 数据足以精确还原 HUD 需要的 `mWh` 显示：
`mWh = mAh × mV / 1000`，即 `FullWh = FullChargeCapacity × Voltage / 1_000_000`。

### 3.4 macOS 数据源可靠性对比

| 数据源 | 延迟 | 权限 | 字段完整度 | 结论 |
|---|---|---|---|---|
| IORegistry `AppleSmartBattery` (IOKit P/Invoke) | 无（同步） | 无 | 容量/电压/电流/温度/循环/充电态 全 | **主路径** |
| `pmset -g batt` (子进程) | 需 fork，~30–80ms | 无 | 仅百分比 + 交流/电池 + 粗估剩余时间 | 兜底 |
| `system_profiler SPPowerDataType` | 秒级 | 无 | 完整但极慢 | 不用 |
| `NSProcessInfo.isLowPowerModeEnabled` | 无 | 无 | 仅低电量模式 | 低电量模式主路径 |

---

## 4. 技术选型与取舍（ADR）

### ADR-1 目标框架：`net8.0-windows`(Win) + `net10.0`(mac)

- 本机与 CI 均**无 net8 共享运行时**，macOS 只能跑 net10（或 self-contained）。
- Windows 侧用户基数已稳定，改 TFM 会连带 `System.Management`、`PublishSingleFile`、桌面运行时依赖与 Inno 打包全部重验，属高风险低收益。
- **决策**：TFM 用 MSBuild 条件拆分 —— `'$(OS)'=='Windows_NT'` 时 `net8.0-windows`，否则 `net10.0`；允许 `-p:TargetFramework=` 覆盖，并预留 `-p:PortToNet10Windows=true` 把 Windows 也迁到 net10（后续独立事项）。
- **代价**：两平台 TFM 不同 → 平台差异**不能**用 TFM 条件编译，必须用自定义符号（ADR-2）。
- **收益**：Windows 构建链路 0 风险；CI 的 Windows job 天然跑在 `windows-latest`，仍产出 net8.0-windows。

### ADR-2 平台隔离：显式自定义编译符号 + 目录级排除

- 不用 `#if WINDOWS`：Windows 目标下由 TFM 隐式定义、非 Windows 目标下未定义，语义含糊且易与 `WIN32`/`WINDOWS` 撞名。
- 项目**显式**定义 `ENDFIELD_WINDOWS` / `ENDFIELD_MACOS`，并用 `Compile Remove` 把对侧平台目录排除出编译（每个 TFM 下只有一套实现参与编译，IDE 不标红）：

```xml
<DefineConstants Condition="'$(TargetFramework)'=='net8.0-windows'">$(DefineConstants);ENDFIELD_WINDOWS</DefineConstants>
<DefineConstants Condition="'$(TargetFramework)'!='net8.0-windows'">$(DefineConstants);ENDFIELD_MACOS</DefineConstants>

<ItemGroup Condition="'$(TargetFramework)'=='net8.0-windows'">
  <Compile Remove="Platform/MacOS/**" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)'!='net8.0-windows'">
  <Compile Remove="Platform/Windows/**" />
  <Compile Remove="Views/TrayMenuWindow.*" />
  <PackageReference Remove="System.Management" />
</ItemGroup>
```

- 少量真正的运行时分发点用 `OperatingSystem.IsWindows()` / `IsMacOS()`（.NET 5+ 的 `[SupportedOSPlatform]` 守卫方法，CA1416 不会报警），集中在 `Platform/PlatformServices.cs` 一处。

### ADR-3 Avalonia 保持 11.2.1，不升 12

- 11.2.1 已实测在本机（macOS 27 / arm64 / .NET 10）完整启动并干净退出。
- Avalonia 12 引入破坏性变更：`SystemDecorations` → `WindowDecorations`（本项目 HUD/菜单/对话框全依赖前者）、默认客户端装饰（`TransparencyLevelHint` 行为变化）、SkiaSharp 2.88 → 3.0、Tizen/`Avalonia.Browser.Blazor`/`BinaryFormatter` 移除。
- **决策**：本次锁定 11.2.1。移植与框架升级混在一起会让"是移植引入的 bug 还是升级引入的 bug"无法二分定位。
- **后续**：11.3.x（同大版本、补丁级）可作为低成本跟进；12.x 另立独立事项。

### ADR-4 macOS 侧不用 `net10.0-macos` 工作负载

- `net*-macos` TFM 会让 `dotnet publish` 走 `.pkg` 生成路径，并干扰手写 `Info.plist`（社区多次报告 `LSUIElement` 被忽略），与"保持 Avalonia 标准发布流程 + 手写 bundle"冲突。
- 本项目对原生 API 的需求很窄（IOKit + CoreFoundation + 少量 AppKit），手写 P/Invoke 完全够用，且**不引入 `Microsoft.macOS` 大型绑定包与 workload 安装要求**。
- **决策**：`net10.0` + 手写互操作。

### ADR-5 托盘改用原生菜单（macOS）

- 事实：Avalonia macOS 的 `native/Avalonia.Native/src/OSX/trayicon.mm` 只实现 `setMenu:` / `setImage:` / `setVisible:` / `setToolTip:`，**没有任何 click 回调** → `TrayIcon.Clicked` 在 macOS 永不触发。
- 自绘菜单要模拟需要：拿 `NSEvent.mouseLocation` + 无锚点定位 + 处理菜单栏坐标系翻转，且与 macOS 交互习惯冲突。
- **决策**：macOS 使用 `NativeMenu`（`NSStatusItem` 左键自动弹出，符合平台习惯）；Windows 保留现有自绘深色菜单（行为零变化）。通过 `ITrayMenuPresenter` 抽象隔离。

### ADR-6 自启用 LaunchAgent，不用 `SMLoginItemSetEnabled` / `SMAppService`

| 方案 | 依赖 | 评价 |
|---|---|---|
| `SMLoginItemSetEnabled` | 需额外 login-item helper bundle | 结构复杂，收益低 |
| `SMAppService.mainApp` | macOS 13+，对签名/公证要求更高 | 首个版本无 Developer ID，风险高 |
| **`~/Library/LaunchAgents/*.plist`** | 无 | **零依赖、可审计、可回滚、用户可直接删除** |

- **决策**：LaunchAgent plist。`Enable` 时写文件并尽力 `launchctl bootstrap`，失败不影响登录时加载。

### ADR-7 低电量模式读取：`isLowPowerModeEnabled` + `respondsToSelector` 守卫

- macOS 27 实测 `lowPowerModeEnabled` 选择子已不存在，裸调直接抛异常崩溃。这是本方案最危险的坑。
- **决策**：先用 `respondsToSelector:` 探测 `isLowPowerModeEnabled`（已实测存在），否则回退解析 `pmset -g` 的 `powermode`（`1` = 低功耗）。**永不裸调未探测的选择子** —— 这条纪律同样适用于所有 AppKit 互操作。

---

## 5. macOS 平台实现设计

### 5.1 电池与功率读数

**主路径**：IOKit 直读 `AppleSmartBattery` 节点（无子进程、无 TCC 权限、无延迟）

```
IOServiceGetMatchingService(kIOMainPortDefault, IOServiceMatching("AppleSmartBattery"))
  → IORegistryEntryCreateCFProperties(entry, &props, kCFAllocatorDefault, 0)
    → CFDictionaryGetValue("Voltage")                     → 12958   (mV)
    → CFDictionaryGetValue("MaxCapacity"/"CurrentCapacity") → 100/100 (%)
    → CFDictionaryGetValue("ExternalConnected")            → Yes
    → CFDictionaryGetValue("IsCharging")                   → No
    → CFDictionaryGetValue("Amperage"/"InstantAmperage")   → 0       (mA)
    → CFDictionaryGetValue("TimeRemaining")                → 65535
    → CFDictionaryGetValue("BatteryData")  → 嵌套 dict:
         FullChargeCapacity 5419 / RemainingCapacity 5419 / DesignCapacity 6249 (mAh)
         FullyCharged 1 / BatteryPower 0 (mW)
```

**映射到现有 `BatterySnapshot`**（字段语义与 Windows 侧完全对齐）：

| Snapshot 字段 | macOS 计算 |
|---|---|
| `FullWh` | `FullChargeCapacity × Voltage / 1_000_000` |
| `RemainingWh` | `RemainingCapacity × Voltage / 1_000_000` |
| `Percent` | `clamp(round(RemainingCapacity×100/FullChargeCapacity), 0, 100)`；若满充 ≤0 退回 `CurrentCapacity` |
| `AcOnline` | `ExternalConnected` |
| `Charging` | `IsCharging` |
| `RateWatts` | `Amperage × Voltage / 1_000_000`（正 = 流入 = 充电）；`Amperage` 缺失时用 `InstantAmperage`；无电流数据置 null |
| `DesignCapacityWh` | `DesignCapacity × Voltage / 1_000_000`（→ 健康度可算） |
| `EstimatedRemaining` | `TimeRemaining` ∈ {0, 65535, 0x80000000} 视为未知 → null |
| `HasBattery` | `FullWh > 0` |

**兜底路径**：`/usr/bin/pmset -g batt` 解析
`Now drawing from 'AC Power'|'Battery Power'` 与 `NN%; [charged|charging|discharging|finishing charge]; H:MM remaining`
仅在 IOKit 失败时调用（避免常态 fork）。

**无电池机型**（Mac mini / iMac / Mac Studio）：`IOServiceMatching` 返回 0 → `GetSnapshot()` 返回 null → HUD 显示 `--`，不崩、不弹提醒（与 Windows 无电池行为一致）。

**内存纪律**：所有 `Create`/`Copy` 出的 CF 对象必须配对 `CFRelease`；用 `try/finally` 包裹；互操作层集中在一个 `sealed` 静态类，方便审计。

### 5.2 电源事件监听

采用与 Windows 版**完全同构**的架构，最大化逻辑复用与可测性：

| 层次 | Windows 现实现 | macOS 实现 |
|---|---|---|
| 事件主路径 | `RegisterPowerSettingNotification(GUID_ACDC_POWER_SOURCE)` + 隐藏消息窗 | `IOPSNotificationCreateRunLoopSource(cb, ctx)`（**公开 API**）挂到专用线程的 `CFRunLoop` |
| 底层轮询 | 2s `System.Threading.Timer`（主路径） | 5s `System.Threading.Timer`（纯兜底，push 通知才是主路径） |
| 去抖确认 | 400ms + `_confirmSeq` 序号，双向复读 | **完全照搬** |
| 启动静默 | 注册后记录初值，`!_initialized` 吞事件 | **完全照搬** |
| 线程模型 | 单消息循环线程，天然串行 | runloop 线程 + timer 线程 **→ 必须加锁** |

**唯一的语义差异（须显式标注）**：Windows 版靠单线程消息循环天然串行；macOS 版回调在 CF runloop 线程、轮询在 timer 线程，因此 `_lastAcOnline` / `_confirmSeq` / `_lastSaverEnabled` 必须用 `lock` 或 `Interlocked` 保护。

**GC 风险**：`IOServiceAddInterestNotification` 的回调委托会被 native 侧长期持有 → 必须 `GCHandle.Alloc(del, GCHandleType.Normal)` 钉住，`Dispose` 时 `Free`，否则 GC 回收后触发 native 栈崩溃。回调体全部 `try/catch` 包住，异常绝不外泄到 native 栈。

**降级方案（Plan B）**：若通知源创建失败，仅保留 5s 轮询 —— 功能等价，插拔最坏响应 `5s + 400ms`。

> **实现期修正（重要）**：原方案打算走 IOKit 私有的 `IOServiceAddInterestNotification(kIOGeneralInterest)`。实测 `kIOGeneralInterest` **未从 IOKit 导出**（`dlsym` 返回 NULL），无法用 P/Invoke 取得；改用公开的 `IOPSNotificationCreateRunLoopSource`（`IOKit/ps/IOPowerSources.h`），已实测可用。
> 同时注意：`IOPSPowerSourceStateKey` 在「已接电但未充电」时会报 `Battery Power`，**不能**用它判断是否插电，必须用 `AppleSmartBattery.ExternalConnected`。

**为什么选它**：`IOPSNotificationCreateRunLoopSource` 是唯一从 IOKit 导出的电源变化通知入口，语义正是「电源来源有变化」；它只负责唤醒，具体数值仍由 `AppleSmartBattery` 读取，两者共用同一套 CoreFoundation 互操作代码。

### 5.3 低电量模式

- 实现：`respondsToSelector:` 守卫 → 存在 `isLowPowerModeEnabled` 则走 ObjC；否则解析 `pmset -g` 的 `powermode`（`1` = 低功耗模式开启）。
- 映射到现有 `PowerSavingChanged` 语义：开 → 完整三态 HUD；关 → 简化胶囊。
- 文案：macOS 用「低电量模式 / Low Power Mode」，Windows 保持「省电模式 / Battery Saver」（平台相关文案在 `Localization` 中分支）。

### 5.4 单实例

- 现有 `Mutex(@"Local\...")` 在 Unix 上跨进程无效。
- macOS 改为：`FileStream(<AppData>/EndfieldCharge/.lock, OpenOrCreate, ReadWrite)` + `FileStream.Lock(0, 1)`；拿不到锁静默退出；锁随进程终止自动释放。
- Windows 保留 `Mutex` 原逻辑（`OperatingSystem.IsWindows()` 分支），不改动已发布行为。

### 5.5 托盘与菜单

- 事实（源码级核实）：Avalonia macOS `AvnTrayIcon` 只有 `setMenu:`，`TrayIcon.Clicked` 永不触发。
- `NSStatusItem` 左键自动弹出其 `menu` → 原生菜单是唯一且最贴合平台习惯的入口。
- 新增抽象 `ITrayMenuPresenter`：

| 平台 | 实现 | 行为 |
|---|---|---|
| macOS | 构造 `NativeMenu`（预览电量 HUD / 设置 / 检查更新 / — / 退出），赋给 `_tray.Menu` | 左键弹出系统菜单 |
| Windows | 包装现有 `TrayMenuWindow.ShowAtTray()` | **零变化** |

- `App.axaml.cs` 的 `OnTrayClicked` 拆为 4 个动作方法（`ShowPreviewHud` / `OpenSettingsWindow` / `CheckForUpdatesAsync` / `ExitApp`），两端共用。
- 语言切换时重建 `NativeMenu` 并重新赋值（macOS 原生菜单不支持就地改文案）。
- 图标建议 `IsTemplateIcon = true` 以获得亮/暗菜单栏自适应；`Assets/tray_bolt.png` 已存在，`.ico` 仅在 Windows 目标下引入。

### 5.6 窗口外观与激活策略

已实测可行（`setLevel:` / `setCollectionBehavior:` 调用成功）：

| 需求 | 手段 | 实测值 |
|---|---|---|
| HUD 盖住菜单栏与全屏应用 | `NSWindow.setLevel:` | `NSFloatingWindowLevel = 3`（Avalonia `Topmost` 仅到此）→ **提升到 `NSPopUpMenuWindowLevel = 101`** |
| 切 Space / 全屏应用上仍可见 | `NSWindow.setCollectionBehavior:` | `128` → **`273`**（`CanJoinAllSpaces(1) | Stationary(16) | FullScreenAuxiliary(256)`） |
| 不占 Dock、不抢焦点 | `NSApplication.setActivationPolicy:Accessory(1)` + `LSUIElement` | 二者叠加（非 bundle 运行时靠前者） |
| 设置窗/对话框到前台 | `-[NSApplication activate]`（Accessory 策略下必须显式激活） | — |

**定位：不改** `HudWindow.PositionTopCenter()`。实测 `WorkingArea = (0, 33, 1512, 890)` 已排除菜单栏/刘海行，`area.Y + 4 = 37` 正好落在其下方；`screen.Scaling` 与窗口 `Width` 同为 DIP，`pixelWidth = Width × scaling` 的换算在 macOS 同样成立。

### 5.7 自启

写入 `~/Library/LaunchAgents/com.lenkmat.endfieldcharge.plist`：

```xml
<key>Label</key><string>com.lenkmat.endfieldcharge</string>
<key>ProgramArguments</key>
<array><string>/Applications/EndfieldCharge.app/Contents/MacOS/EndfieldCharge</string></array>
<key>RunAtLoad</key><true/>
<key>ProcessType</key><string>Interactive</string>
```

- `Enable()`：写文件 + 尽力 `launchctl bootstrap gui/$UID <plist>`（失败忽略，登录时 launchd 仍会加载）
- `IsEnabled()`：文件存在且 `Label` 匹配
- `Disable()`：`launchctl bootout gui/$UID/<label>` + 删除 plist
- 可执行路径解析：`NSBundle.mainBundle.bundlePath`（`.app` 时）→ `Environment.ProcessPath` → `AppContext.BaseDirectory`
- **非 bundle 环境**（`dotnet run`）禁用该项并在设置页提示，避免写出错误路径

### 5.8 字体

统一替换为（4 处）：

```
正文：  "HarmonyOS Sans SC", "HarmonyOS Sans", Inter, "PingFang SC", "Microsoft YaHei UI", "Hiragino Sans GB", sans-serif
数字：  "Inter Medium", "HarmonyOS Sans SC Medium", "HarmonyOS Sans Medium", Inter, "SF Pro Text", "PingFang SC", "Microsoft YaHei UI", sans-serif
```

- 改动点：`Styles/HudTheme.axaml:39,42`、`Settings/SettingsWindow.axaml:14`、`Settings/SettingsWindow.axaml.cs:453`、`App.axaml.cs:236`。
- `.WithInterFont()` 保留（内置 Inter 保证跨平台一致），macOS 上 `Inter` 命中内置字体，中文回退 `PingFang SC`。

### 5.9 配置与日志路径

| 项 | Windows | macOS（建议） |
|---|---|---|
| 设置 | `%APPDATA%\EndfieldCharge\settings.json` | `~/Library/Application Support/EndfieldCharge/settings.json` |
| 日志 | `%TEMP%\EndfieldCharge\log-yyyyMMdd.txt` | `~/Library/Logs/EndfieldCharge/log-yyyyMMdd.txt`（`GetTempPath()` 在 mac 为每用户随机目录，不便排查） |
| 单实例锁 | `Mutex` | `<AppData>/EndfieldCharge/.lock` |

`settings.json` **schema 不变**，两平台可共用同一份结构，便于将来迁移。

### 5.10 自检模式 `--selftest`（新增）

输出 JSON 后退出（不进 UI 循环），用于 CI 冒烟与用户报障取证：

```json
{ "platform":"macOS", "bundled":true, "bundlePath":"/Applications/EndfieldCharge.app",
  "batteryPercent":100, "remainingWh":70.2, "fullWh":70.2, "designWh":81.0, "healthPercent":86.7,
  "acOnline":true, "charging":false, "rateWatts":0.0,
  "lowPowerMode":false, "lowPowerModeSource":"iokit",
  "loginItemEnabled":false, "settingsPath":"~/Library/Application Support/EndfieldCharge/settings.json" }
```

优先级高于现有 `--demo` / `--preview-unplug` / `--preview` / `--debug-ring` / `--power-log`。

### 5.11 不改动项

`Platform.Start(url)`（`Process.Start(UseShellExecute=true)` 在 macOS 等价 `open`）、`UpdateChecker` 逻辑（但 macOS CI 必须同样注入 `/p:Version=`，否则版本比较失准）、全部动画与 HUD XAML、设置窗口 XAML 布局。

---

## 6. 目标架构

```
EndfieldCharge.csproj                          ← 条件 TFM / 条件引用 / 条件符号
├─ Platform/                                   ← 新增：平台抽象与实现
│  ├─ PlatformServices.cs                      ← 唯一运行时分发点
│  ├─ IPowerMonitor.cs / IAutoStartManager.cs / IPlatformServices.cs / ITrayMenuPresenter.cs
│  ├─ BatterySnapshot.cs                       ← 从 BatteryService.cs 抽出的共享模型
│  ├─ HudDisplayCoordinator.cs                 ← HUD 事件合并（同一物理动作只播一次）
│  ├─ NotchBirthGeometry.cs                    ← macOS 刘海出生几何（纯函数，可单测）
│  ├─ AppPaths.cs / SelfTest.cs                ← 平台目录 / --selftest 诊断
│  ├─ Windows/
│  │  ├─ WindowsPlatformServices.cs
│  │  ├─ WindowsPowerMonitor.cs                ← 现 PowerWatcher.cs 迁入（逻辑零改动）
│  │  ├─ WindowsBatteryReader.cs               ← 现 BatteryService.cs 迁入
│  │  ├─ WindowsPowerNative.cs                 ← 现 PowerNative.cs 迁入（内容不变）
│  │  └─ WindowsAutoStart.cs                   ← 现 AutoStart.cs 迁入
│  └─ MacOS/
│     ├─ MacOSPlatformServices.cs
│     ├─ MacOSPowerMonitor.cs                  ← IOPS 通知 + 5s 轮询 + 400ms 去抖 + 加锁
│     ├─ MacOSBatteryReader.cs                 ← IORegistry AppleSmartBattery + pmset 兜底
│     ├─ MacOSPowerNative.cs                   ← IOKit / CoreFoundation P/Invoke
│     ├─ MacOSAppKit.cs                        ← libobjc.dylib 互操作（NSWindow/NSApp/NSScreen/NSBundle/NSProcessInfo）
│     ├─ MacOSAutoStart.cs                     ← LaunchAgent plist
│     └─ MacOSTrayMenu.cs                      ← NativeMenu 构建与本地化刷新
├─ Services/                                   ← 保留跨平台：Logger / UpdateChecker
├─ Views/TrayMenuWindow.*                      ← 仅 Windows 编译
├─ docs/macos-port-assessment.md               ← 本文档
├─ docs/macos-distribution.md                  ← 签名 / 公证 / 分发
├─ build/macos/Info.plist                      ← .app 模板（@@VERSION@@ 占位）
├─ build/macos/EndfieldCharge.entitlements     ← 最小权限
├─ scripts/package-macos.sh                    ← .app 组装 + 可选签名 + 可选 dmg
└─ .github/workflows/build.yml                 ← 新增 build-macos job
```

---

## 7. 接口设计（公开 API 变化）

```csharp
// Platform/BatterySnapshot.cs —— 从 BatteryService.cs 抽出，字段与语义不变
public sealed record BatterySnapshot(double RemainingWh, double FullWh, int Percent,
    bool AcOnline, bool Charging)
{
    public double? RateWatts { get; init; }
    public double? DesignCapacityWh { get; init; }
    public double? HealthPercent { get; }          // 现有实现保留
    public TimeSpan? EstimatedRemaining { get; init; }
    public bool HasBattery => FullWh > 0;
}

// Platform/IPowerMonitor.cs —— 取代对具体类 PowerWatcher / 静态类 BatteryService 的直接依赖
public interface IPowerMonitor : IDisposable
{
    event EventHandler<bool>? PowerSourceChanged;   // true = 已接交流电
    event EventHandler<bool>? PowerSavingChanged;   // true = 省电/低电量模式开启
    BatterySnapshot? GetSnapshot();
    bool TryGetAcOnline(out bool acOnline);
    void Start();
    void Stop();
}

// Platform/IAutoStartManager.cs
public interface IAutoStartManager
{
    bool IsSupported { get; }                       // macOS 非 bundle 时为 false
    bool IsEnabled();
    void Enable();
    void Disable();
}

// Platform/ITrayMenuPresenter.cs
public interface ITrayMenuPresenter : IDisposable
{
    void Attach(TrayIcon tray);
    void RefreshLocalization();
}

// Platform/IPlatformServices.cs
public interface IPlatformServices
{
    string PlatformName { get; }                    // "Windows" / "macOS"
    IPowerMonitor CreatePowerMonitor();
    IAutoStartManager AutoStart { get; }
    ITrayMenuPresenter CreateTrayMenuPresenter(TrayMenuActions actions);
    void ApplyAppActivationPolicy();                // macOS: Accessory
    void OnHudWindowShown(Window window);           // macOS: NSWindow level / collectionBehavior
    void ActivateForDialog();                       // macOS: 设置窗/对话框到前台
}

public sealed record TrayMenuActions(
    Action OnPreview, Action OnSettings, Action OnCheckUpdate, Action OnExit);

public static class PlatformServices
{
    public static IPlatformServices Create()
        => OperatingSystem.IsWindows() ? WindowsPlatformServices.Create()
         : OperatingSystem.IsMacOS()   ? MacOSPlatformServices.Create()
         : throw new PlatformNotSupportedException();
}
```

### 7.1 调用侧改动清单

| 文件 | 改动 |
|---|---|
| `Program.cs` | 入口：`--selftest` 分支优先；单实例改为平台分支（Win=Mutex / mac=文件锁）；`ApplyAppActivationPolicy()` 时机 |
| `App.axaml.cs` | `new PowerWatcher()` → `_platform.CreatePowerMonitor()`；`BatteryService.GetSnapshot()` → `_monitor.GetSnapshot()`；`PowerNative.TryGetAcOnline` → `_monitor.TryGetAcOnline()`；`OnTrayClicked` 拆为 4 个动作方法并交给 `ITrayMenuPresenter`；`OnSettingsChanged` 增加 `RefreshLocalization()`；字体栈 1 处 |
| `Views/HudWindow.axaml.cs` | `Opened` 事件调 `IPlatformServices.OnHudWindowShown(this)`（经 `App.CurrentPlatform` 静态属性获取，避免构造签名扩散） |
| `Views/HudWindow.axaml.cs` | macOS 出生前导：`WithBirthGeometry()` 把 `NotchBirthGeometry` 写入 `AnimationOptions`；`ResetToInitial()` 复位 `BirthHost` |
| `Animations/HudAnimations.cs` | 新增 `BirthOffset` / `BirthPillScale` 两条前导轨道；`MapCue` 支持前导时长并加 `MaxIntroFraction` 上限 |
| `Views/HudWindow.axaml` | 新增 `BirthHost`（仅位移）承接出生下移；`Pill.RenderTransformOrigin` 改为顶边中点 |
| `Settings/SettingsWindow.axaml.cs` | `Services.AutoStart.*` → `_platform.AutoStart.*`；`MessageBox.Show` 前调 `ActivateForDialog()`；字体栈 1 处 |
| `Settings/SettingsWindow.axaml` | 字体栈 1 处 |
| `Styles/HudTheme.axaml` | 字体栈 2 处 |
| `Localization.cs` | `DescAutoStart` 平台分支；省电/低电量模式文案平台分支；`FontInstalled` 平台分支 |

### 7.2 schema / 数据流变化

- `settings.json`：**无变化**（`AppSettings` 是 record，无平台专属字段）。
- 新增运行时产物：`~/Library/Logs/EndfieldCharge/*.log`、`~/Library/LaunchAgents/com.lenkmat.endfieldcharge.plist`、`~/Library/Application Support/EndfieldCharge/{settings.json,.lock}`。
- 新增只读诊断输出：`--selftest` 的 JSON（stdout，仅此一次）。

---

## 8. 实施步骤（每步可独立验证）

### Step 0 — 落文档
`docs/macos-port-assessment.md`（本文）+ `docs/macos-distribution.md`。

### Step 1 — 工程多目标化（不改任何行为）
`EndfieldCharge.csproj` 增加：条件 TFM、条件编译符号、条件 RID、条件 `System.Management`、条件 `ApplicationManifest`/`ApplicationIcon`；macOS 目标下 `UseAppHost=true`、`SelfContained=true`、`PublishSingleFile=false`。
**验证**：本机 `dotnet build -c Debug`（→ net10.0）通过；`dotnet publish -c Release -r osx-arm64` 产出可执行文件；Windows CI job 仍绿。

### Step 2 — 平台抽象层骨架
新增 `Platform/*.cs` 接口与 `PlatformServices.Create()`；抽出 `BatterySnapshot`；把 `PowerNative` / `PowerWatcher` / `BatteryService` / `AutoStart` **原样搬进** `Platform/Windows/` 并实现接口；改造 §7.1 的调用点。
**验证**：Windows 行为完全不变（CI + 代码走查 diff 仅"移动 + 替换调用"）。

### Step 3 — macOS 电池读取
`MacOSPowerNative.cs` + `MacOSBatteryReader.cs`。
**验证**：`--selftest` 输出逐字段对比 `ioreg -rn AppleSmartBattery` 与 `pmset -g batt`。

### Step 4 — macOS 电源监听
`MacOSPowerMonitor.cs`（IOPS 通知源 + 5s 轮询 + 400ms 双向去抖 + 加锁 + GCHandle 钉住回调）。
**验证**：真机插拔各 5 次，`--power-log` 记录无重复触发、无启动误弹；响应延迟实测 ≤1.5s。

### Step 5 — macOS 窗口互操作
`MacOSAppKit.cs` + 接入 `HudWindow.Opened`。
**验证**：HUD 不遮刘海；切 Space / 全屏应用上仍可见；点击消失；设置窗能到前台。

### Step 6 — 托盘原生菜单
`MacOSTrayMenu.cs` + `ITrayMenuPresenter`；拆分 `App.axaml.cs` 动作方法。
**验证**：四项均可用；切语言后文案更新；重启后常驻。

### Step 7 — 自启 / 低电量模式 / 路径
`MacOSAutoStart.cs`；`isLowPowerModeEnabled`（含 `respondsToSelector` 守卫与 `pmset` 兜底）；`SettingsManager` / `Logger` 的 macOS 路径分支。

### Step 8 — 字体与本地化
4 处字体栈替换；`Localization` 平台分支。

### Step 9 — 打包与 CI
`build/macos/Info.plist`（`CFBundleIdentifier=com.lenkmat.endfieldcharge`、`LSUIElement=true`、`NSHighResolutionCapable=true`、`LSApplicationCategoryType=public.app-category.utilities`、`CFBundleShortVersionString=@@VERSION@@`）、`build/macos/EndfieldCharge.entitlements`、`scripts/package-macos.sh`（publish → 组装 `.app/Contents/{MacOS,Resources}` → 写 Info.plist → `chmod +x` → 可选 `codesign` → 可选 `hdiutil` → 可选 `notarytool`）；`.github/workflows/build.yml` 增加 `build-macos` job（`macos-latest` + `setup-dotnet 10.0.x` + `--selftest` + 打包 + 上传 artifact）。
**验证**：本地脚本跑通、双击 `.app` 启动成功；CI 绿。

### Step 10 — 回归与文档
更新 `README.md`（macOS 章节、运行要求、`--selftest`、构建与打包命令）；全量跑 Windows job 确认无回归。

---

## 9. 边界情况与失败模式

| 场景 | 处理 |
|---|---|
| 无电池 Mac（mini/iMac/Studio） | `AppleSmartBattery` 匹配失败 → `GetSnapshot()` 返回 null → HUD 显示 `--`；`CheckAlerts` 因 `!HasBattery` 直接返回，不误弹提醒 |
| 健康度取整差异 | `system_profiler` 报 89%，mAh 比值算出 86.7% → 以原始 mAh 为准，不做二次对齐，文档说明 |
| `TimeRemaining = 65535`（充电中/充满） | 视为未知 → `EstimatedRemaining = null` |
| `Amperage = 0`（充满后） | 与 Windows 的 `Rate == 0 → null` 语义统一为 null，避免平台间差异 |
| 插拔瞬间 IOKit 返回缺省字段 | 400ms 双向去抖 + 复读校验；不符则回滚 `_lastAcOnline`（照搬现逻辑） |
| GC 回收 IOKit/CF 回调委托 | `GCHandle.Alloc(Normal)` 钉住，`Dispose` 释放；回调体全 `try/catch` |
| CF 对象泄漏 | 每个 `Create`/`Copy` 配对 `CFRelease`，`try/finally` 包裹 |
| macOS 27 无 `lowPowerModeEnabled` | `respondsToSelector:` 守卫 + `pmset -g` 的 `powermode` 兜底；**绝不裸调** |
| runloop 线程 vs timer 线程竞态 | 所有共享状态 `lock` 保护（Windows 版无此需要，显式标注为平台差异） |
| 非 bundle 运行（`dotnet run`） | `NSApp.setActivationPolicy:Accessory` 保证无 Dock 图标；自启项显示不可用 |
| LaunchAgent 写入/`launchctl` 失败 | 静默失败并用 `IsEnabled()` 复核回滚 UI 开关（与 Windows 侧注册表写失败静默一致） |
| 无签名分发的 Gatekeeper 拦截 | 文档给出右键「打开」与 `xattr -dr com.apple.quarantine` 两条路径，正式方案为 Developer ID + 公证 |
| macOS 未来移除某选择子 | 所有 AppKit 调用前置 `respondsToSelector:` 守卫，缺失即降级而非崩溃 |
| 多显示器 / HUD 位置 | 复用现 `ResolveScreen`；`WorkingArea` 已排除菜单栏 |
| 语言切换后菜单不更新 | `OnSettingsChanged` 显式重建 `NativeMenu` 并重新赋值 |
| 通知源完全不可用 | 退化为纯 5s 轮询（Plan B），功能等价、延迟变差 |

---

## 10. 验收与测试

### 10.1 自动化
- `dotnet build -c Release /p:TreatWarningsAsErrors=true`（本机 + Windows CI 双跑）
- `dotnet publish -c Release -r osx-arm64` 产物断言：`libAvaloniaNative.dylib`、`libSkiaSharp.dylib`、`.app/Contents/MacOS/EndfieldCharge` 均存在且 `file` 报 `arm64`
- `./EndfieldCharge.app/Contents/MacOS/EndfieldCharge --selftest` 在 CI macOS runner 返回 0 且 JSON 含 `acOnline`
- 新增 `tests/EndfieldCharge.Tests`（xUnit），对电池映射逻辑做**纯函数单测**：注入 `voltageMv` / `fullChargeMah` / `remainingMah` / `designMah` / `timeRemaining` 等原始值，断言 mWh 换算、百分比边界、健康度、`HasBattery`、`EstimatedRemaining` 的哨兵值（0 / 65535 / 0x80000000）处理

### 10.2 手工清单（真机逐条勾）
1. 插电 → 完整三态、文案「超充模式」、数值与 `ioreg` 一致
2. 拔电 → 简化胶囊
3. `<20%` 电量圈转红
4. 低电量提醒（阈值 5–40%）与充满提醒（≥99%）
5. 低电量模式开关 → 对应 HUD（文案「低电量模式」）
6. 点击 HUD 立即消失；`--debug-ring` 静态态正确
7. 设置页 4 个 Tab 全项可保存、重启后保留；动画页「播放」实时预览
8. 菜单栏菜单 4 项可用；切语言后文案更新
9. 登录时自启：开启后 `launchctl list | grep endfield` 可见，重启后自动运行
10. 多显示器 + 顶部居中/靠左/靠右三种位置
11. 全屏应用上 HUD 仍可见；切 Space 后出现
12. 检查更新（断网时显示「检查更新失败」，不崩）
13. 退出后无残留进程、无残留菜单栏图标
14. 连续插拔 20 次无重复 HUD、无窗口漂移

### 10.3 Windows 回归
现有 CI job 全绿 + 上述清单在 Windows 抽测 1 / 2 / 4 / 7 / 9 / 13。

---

## 11. 工作量与风险

### 11.1 工作量估算（按提交切分）

| Step | 内容 | 估算代码量 |
|---|---|---|
| 1 | csproj 多目标化 | ~40 行 |
| 2 | 抽象层 + Windows 搬迁 | ~150 新增 + ~700 迁移 |
| 3 | macOS 电池读取 | ~300 行 |
| 4 | macOS 电源监听 | ~250 行 |
| 5 | macOS 窗口互操作 | ~150 行 |
| 6 | macOS 托盘菜单 | ~120 行 |
| 7 | 自启 / 低电量 / 路径 | ~180 行 |
| 8 | 字体与本地化 | ~20 行 |
| 9 | 打包脚本 + Info.plist + CI | ~200 行 |
| 10 | README + 回归 | ~80 行 |
| — | 单测 | ~150 行 |
| **合计** | | **约 1200–1500 行新增、约 700 行迁移** |

### 11.2 风险清单

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| 通知源不可用 | 低 | 响应延迟从 ≤1.5s 变 ≤5.5s | Plan B 纯轮询；功能等价 |
| AppKit 选择子在 macOS 26/27 变动 | 中 | 崩溃 | **所有**调用前置 `respondsToSelector:` 守卫；已在 `lowPowerModeEnabled` 上踩坑并验证该守卫有效 |
| CF/IOKit 内存或 GC 问题 | 中 | 偶发崩溃、泄漏 | 互操作集中单文件；`GCHandle` 钉住；`CFRelease` 配对；`--selftest` 反复跑做泄漏冒烟 |
| LaunchAgent 在未公证应用上的行为 | 低 | 自启失效 | `IsEnabled()` 复核 + UI 明确提示；文档给出手动添加方法 |
| Avalonia 11.2.1 在 macOS 更旧版本（14/15）表现差异 | 低 | 兼容性 | CI 只跑 `macos-latest`；文档标注最低支持 macOS 13+，社区可反馈 |
| 无 Developer ID 导致分发摩擦 | 高 | 用户首次打开受阻 | 文档给出绕过路径；正式分发依赖后续证书采购（非本次范围） |
| Windows 侧回归 | 低 | 高 | 抽象层搬迁保持逻辑零改动 + CI Windows job 必须全绿 |

---

## 12. 假设与开放问题

### 12.1 假设
1. 主目标机为 Apple Silicon (arm64) + macOS 26/27；同时提供 `osx-x64` RID（Intel Mac 为次要目标，不单独验收）。
2. 用户接受 macOS 上以**原生菜单**取代 Windows 的自绘深色菜单（技术前提见 §5.5）。
3. 首版按 ad-hoc 签名分发；Developer ID + 公证作为可选后续步骤（本机仅 Apple Development 证书）。
4. 不引入 `net10.0-macos` 工作负载（理由见 ADR-4）。

### 12.2 开放问题（需用户确认）

| # | 问题 | 默认取值（未确认时按此执行） |
|---|---|---|
| Q1 | 插拔响应目标：若通知源不可用，是否接受 ≤5.5s？ | 接受，退化为纯轮询（实测本机通知源可用，正常为 ≤0.5s） |
| Q2 | macOS HUD 高度是否需要避开刘海？现在放在菜单栏**下方**（`WorkingArea.Y + 4`）而非覆盖菜单栏 | 放在菜单栏下方（与 Windows 视觉最接近，且不遮刘海与菜单栏图标） |
| Q3 | 打包产物：只要 `.app`，还是同时要 `.dmg`？ | 同时产出（dmg 便于分发） |
| Q4 | 是否本次顺带把 Windows TFM 迁到 net10？ | 否，仅预留 `/p:PortToNet10Windows=true` 开关 |
| Q5 | 菜单栏图标是否改用 `IsTemplateIcon`（随菜单栏亮暗自动反色）？ | 是（避免暗色菜单栏下看不清） |
| Q6 | 是否需要「开机自启」在 macOS 上默认开启？ | 否，默认关闭（与 Windows 现状一致） |
