# macOS 移植验证指南

配套文档：[macos-port-assessment.md](./macos-port-assessment.md)（方案）、[macos-distribution.md](./macos-distribution.md)（分发）

---

## 1. 快速验证（3 分钟）

```bash
# 1) 构建
dotnet build -c Debug

# 2) 诊断（不需要 UI，最快确认互操作是否可用）
dotnet bin/Debug/net10.0/EndfieldCharge.dll --selftest

# 3) 打包成 .app（内含 --selftest 冒烟门禁）
./scripts/package-macos.sh

# 4) 启动
open build/macos/dist/EndfieldCharge.app
```

`--selftest` 期望输出（本机实测样例，Apple Silicon / macOS 27 / 电池放电中）：

```json
{
  "platform": "macOS",
  "bundle": "true",
  "powerSaveModeText": "低电量模式",
  "dataDir": "~/Library/Application Support/EndfieldCharge",
  "logDir": "~/Library/Logs/EndfieldCharge",
  "autoStartSupported": "true",
  "autoStartEnabled": "false",
  "acReadable": "true",
  "acOnline": "false",
  "batteryPercent": "97",
  "remainingWh": "65.809",
  "fullWh": "68.037",
  "designWh": "79.100",
  "healthPercent": "86.0",
  "charging": "false",
  "rateWatts": "-16.177",
  "estimatedRemainingMinutes": "281",
  "hasBattery": "true",
  "inAppBundle": "true",
  "bundlePath": ".../EndfieldCharge.app",
  "bundleIdentifier": "com.lenkmat.endfieldcharge",
  "bundleExecutable": ".../EndfieldCharge.app/Contents/MacOS/EndfieldCharge",
  "lowPowerModeSource": "iokit",
  "lowPowerMode": "true"
}
```

**判读要点**

| 字段 | 期望 | 异常含义 |
|---|---|---|
| `fatalError` | 不存在 | 互操作层初始化失败，看错误链 |
| `acOnline` | 与 `pmset -g batt` 一致 | 与系统不符 = ExternalConnected 读取有问题 |
| `batteryPercent` / `remainingWh` / `fullWh` | 与 `ioreg -rn AppleSmartBattery` 换算一致 | 差异 >2% 说明电压/容量键取错 |
| `healthPercent` | 约等于 `fullWh/designWh` | `system_profiler` 报的是标称取整值，允许 ±3 |
| `rateWatts` | 放电为负、充电为正 | 符号反了 = Amperage 补码未还原 |
| `estimatedRemainingMinutes` | 与 `pmset` 的 `H:MM remaining` 接近 | 若为 65535 说明哨兵值未过滤 |
| `lowPowerModeSource` | `iokit` | `pmset-or-unsupported` = `isLowPowerModeEnabled` 不存在 |
| `autoStartSupported` | bundle 内为 `true` | 非 bundle 运行为 `false` 属正常 |
| `bundleIdentifier` | `com.lenkmat.endfieldcharge` | `null` = 未在 .app 内运行 |

### 与系统真值交叉核对

```bash
pmset -g batt
# Now drawing from 'Battery Power'
#  -InternalBattery-0 (id=...)  100%; discharging; 5:19 remaining present: true

ioreg -rn AppleSmartBattery | grep -E '"(Voltage|Amperage|TimeRemaining|ExternalConnected)"|BatteryData'
#   "Voltage" = 12645
#   "Amperage" = 18446744073709550026      ← 无符号补码，实为 -2092 mA
#   "TimeRemaining" = 256
#   "ExternalConnected" = No
#   "BatteryData" = {..."FullChargeCapacity"=5374,"RemainingCapacity"=5236,...
```

验算：`5236 mAh × 12.645 V / 1e6 = 66.2 Wh`、`5374 × 12.645 / 1e6 = 68.0 Wh`、
`5367/6249 = 86%`、`-2092 mA × 12.645 V / 1e6 = -26.5 W`。

---

## 2. 日志观察点

应用日志：`~/Library/Logs/EndfieldCharge/log-YYYYMMDD.txt`

**正常启动应看到：**

```
[..] [INFO] MacOSPlatformServices: activation policy = Accessory
[..] [INFO] App: platform=macOS, bundle=True
[..] [INFO] MacOSPowerMonitor: IOPS 电源变化通知源已挂载
[..] [INFO] MacOSPowerMonitor: runloop start, initialAc=False, initialSaver=True, pollInterval=5s
[..] [INFO] MacOSPlatformServices: NSWindow level=101 collectionBehavior=273 transparent=True
```

| 日志 | 含义 |
|---|---|
| `activation policy = Accessory` | 不占 Dock 成功 |
| `IOPS 电源变化通知源已挂载` | 事件主路径可用；若为 `WARN ... 仅使用 5s 轮询兜底` 则退化为轮询（功能仍可用，延迟变差） |
| `runloop start, initialAc=...` | 记录了初值；启动时不弹 HUD（已插电/已开低电量模式都静默） |
| `NSWindow level=101 collectionBehavior=273` | HUD 已提升到弹窗层级并跨 Space 可见 |
| `transparent=True` | **窗口背景为 `[NSColor clearColor]`**，胶囊之外透出桌面 |
| `WARN ... 窗口背景不透明（会显示为黑底）` | ⚠️ 退化成黑底 —— 检查 `HudWindow.axaml` 的 `TransparencyLevelHint` 是否被删掉 |

**插拔电源时应看到：**

```
[..] [INFO] MacOSPowerMonitor: AC disconnected → 上报 HUD
[..] [INFO] MacOSPowerMonitor: AC connected → 上报 HUD
```

若插拔后**没有**这两行：
1. 确认 `IOPS 电源变化通知源已挂载` 是否出现（没有则等 5s 轮询兜底）；
2. 确认 `initialAc` 与实际状态是否一致（不一致说明初值读取时机有问题）；
3. 400ms 去抖会滤掉瞬时抖动 —— 若电源在 400ms 内抖动回来，会被判定为抖动并回滚（**这是预期行为**）。

---

## 3. 手工验收清单

### 3.1 基础

- [ ] 启动后 Dock **无**图标，Cmd-Tab **不**出现 EndfieldCharge
- [ ] 菜单栏出现图标
- [ ] 无多余窗口弹出（HUD 只在事件/预览时出现）
- [ ] 退出后进程结束、菜单栏图标消失

### 3.2 电源事件（核心）

- [x] **拔电** → ≤1.5s 内弹出简化电量胶囊 —— 实测 233–403 ms，已通过
- [x] **插电** → ≤1.5s 内弹出完整三态动画（电标 → 撑高显示「超充模式」→ 收成胶囊显示电量）—— 实测 244–338 ms，已通过
- [x] 连续插拔 7 次，无重复 HUD —— 已通过（每次恰好一次上报）
- [ ] 电池充满时插着不动，不出现反复弹窗（需长时间静置观察）

### 3.3 数值

- [ ] HUD 的 `剩余 mWh / 满充 mWh` 与 `ioreg` 换算值一致（±50 mWh）
- [ ] 百分比与 `pmset -g batt` 一致
- [ ] 电量 < 20% 时电量圈变红（`#FF4D4F`）

### 3.4 窗口行为

- [x] **胶囊之外透出桌面，无黑色矩形底** —— 已修复并验证（见 §6 第 1 条）
- [ ] HUD 停留在菜单栏下方；左/右落点距屏幕边缘 20 DIP
- [ ] **出生动画**：插电时窄条从刘海内部向下离开，再展开到选定落点，接入三态
- [ ] 出生窄条位于刘海内部，返回终点与出生点一致
- [ ] 拔电的简化胶囊同样从刘海处落下
- [ ] 无刘海外接屏：不播放刘海动画，左/右落点按胶囊边缘对齐
- [ ] 设置页「动画」页的预览也带出生前导（预览用的是滑块参数 + 当前屏幕几何）
- [ ] 点击 HUD 立即淡出消失
- [ ] 切到另一个 Space 后再触发，HUD 仍可见
- [ ] 全屏应用（如全屏 Safari/视频）上方触发，HUD 仍可见
- [ ] HUD 出现时**不抢焦点**（当前输入的应用不失去键盘焦点）

### 3.5 托盘菜单

- [ ] 左键菜单栏图标 → 弹出菜单（预览电量 HUD / 设置 / 检查更新 / 退出）
- [ ] 点击「预览电量 HUD」→ 立即播放真实电池数据的完整动画
- [ ] 点击「设置」→ 设置窗口出现并位于前台
- [ ] 点击「检查更新」→ 有更新弹确认框 / 无更新弹「已是最新版本」；断网时「检查更新失败」且不崩溃
- [ ] 点击「退出」→ 进程结束
- [ ] 设置里切换语言为「英文」→ 保存 → 菜单文案变英文

### 3.6 设置窗口

- [ ] 四个 Tab（通用 / 动画 / 通知 / 关于）均可切换
- [ ] 全局缩放、显示时长、HUD 位置（顶部居中/靠右/靠左）保存后立即生效
- [ ] 显示器下拉能列出所有显示器；选择后 HUD 出现在指定显示器
- [ ] 动画页滑块 + 「播放」能实时预览（无需保存）
- [ ] 语言切换为中文/英文/自动，文案立即更新
- [ ] 保存后重启应用，全部设置保留（`~/Library/Application Support/EndfieldCharge/settings.json`）

### 3.7 提醒

- [ ] 低电量提醒：阈值设为 90%，拔电后应弹「电量不足」卡牌，4s 自动消失
- [ ] 充满提醒：电量 ≥99% 时插入电源应弹「已充满」

### 3.8 低电量模式

- [ ] 系统设置 → 电池 → 开启「低电量模式」→ 弹出完整三态动画，标题为「低电量模式」
- [ ] 关闭低电量模式 → 弹出简化胶囊
- [ ] 在设置「通知」页关闭该提示后，开关低电量模式不再弹 HUD

### 3.9 开机自启

- [ ] 设置「通用」页勾选开机自启 → 保存 → `launchctl list | grep endfield` 能看到条目
- [ ] 检查 `~/Library/LaunchAgents/com.lenkmat.endfieldcharge.plist` 内容正确（`ProgramArguments` 指向 .app 内的可执行文件）
- [ ] 取消勾选 → 保存 → plist 被删除
- [ ] （可选）重启后自动运行

### 3.10 无电池机型（如有 Mac mini / iMac）

- [ ] 启动不崩溃
- [ ] `--selftest` 输出 `"battery": "null"`，退出码仍为 0
- [ ] 触发预览时 HUD 显示 `--`，不弹低电量提醒

---

## 4. 已完成的自动化验证

| 项 | 状态 |
|---|---|
| `dotnet build -c Debug`（net10.0 / macOS） | ✅ 0 warning 0 error |
| `dotnet build -c Debug -p:TargetFramework=net8.0-windows`（交叉编译 Windows） | ✅ 通过 |
| 两平台 `TreatWarningsAsErrors` 严格构建 | ✅ 无编译器/分析器警告 |
| `dotnet publish -c Release -r osx-arm64` | ✅ 产出 arm64 apphost + 全部原生 dylib |
| `--selftest` 数值 vs `pmset` / `ioreg` 逐字段交叉核对 | ✅ 全部一致 |
| `dotnet test tests/EndfieldCharge.Tests` | ✅ 34/34 通过（HUD 事件合并 10 + 出生几何 24） |
| 出生几何 vs 真机 `safeAreaInsets` 交叉核对 | ✅ 出生顶点闭式校验为 33.0 = `WorkingArea.Y`；最终顶点 67.4（见 §6 第 3 条） |
| 真机连续 7 次插拔的事件合并 | ✅ 每次物理动作恰好一次 HUD（真机日志见 §6 第 2 条） |
| HUD 窗口透明（`transparent=True`） | ✅ 实测窗口背景为 `[NSColor clearColor]` |
| `./scripts/package-macos.sh` | ✅ 产出 `.app` + `.dmg`，codesign 校验通过，`--selftest` 冒烟通过 |
| HUD `NSWindow` 增强 | ✅ 实测 `level=101`、`collectionBehavior=273` |
| IOPS 电源变化通知源挂载 | ✅ 实测挂载成功 |

**已完成的真机事件验证**（Apple Silicon / macOS 27，通过 100ms 采样 `ioreg ExternalConnected` 作为系统真值对照应用日志）：

| 轮次 | 系统真值变化时刻 | 应用上报时刻 | 端到端延迟 |
|---|---|---|---|
| 插电 | 23:40:02.701 | 23:40:02.984 | **283 ms** |
| 拔电 | 23:40:12.392 | 23:40:12.625 | **233 ms** |
| 插电 | 23:40:24.290 | 23:40:24.574 | **284 ms** |
| 拔电 | 23:40:29.801 | 23:40:30.204 | **403 ms** |
| 插电 | 23:40:33.880 | 23:40:34.124 | **244 ms** |
| 拔电 | 23:40:51.623 | 23:40:51.864 | **241 ms** |
| 插电 | 23:40:54.156 | 23:40:54.494 | **338 ms** |

- **7/7 次插拔全部检出，每次恰好一次上报**（无重复 HUD、无遗漏）。
- 延迟区间 233–403 ms，与设计的 `ChangeConfirmDelay = 400ms` 一致 —— **符合预期，且远低于 1.5s 目标**。
- 重点验证了去抖逻辑：有一次插电同时触发了「低电量模式 OFF」，两者相隔仅 250 ms，
  应用**没有**因此多弹第二次 HUD（`PowerSourceChanged` 与 `PowerSavingChanged` 各自独立且都做了"与上次不同"判定）。
- 主路径确认为 push 通知（延迟 <400ms 而非 5s 轮询周期），说明 `IOPSNotificationCreateRunLoopSource` 生效。

**尚未完成（需人工在真机上操作）**：§3 中除「插拔电源事件」以外的 UI 条目 ——
菜单栏菜单点击、设置窗口交互、HUD 视觉位置与全屏/Space 可见性、提醒弹窗、
开机自启开关、语言切换等。`--selftest` 与日志只能覆盖数据与事件链路，
UI 交互需人工过一遍清单。

---

## 5. 已知限制

| 限制 | 说明 | 后续 |
|---|---|---|
| 菜单栏图标不做 template 化 | `Assets/tray_bolt.png` 是 1538×1539 的**深底白闪电圆角方块**，不是透明底单色图形。直接标为 template 会把整个深色方块做成不透明蒙版，视觉错误。需要新设计一版 19×19pt（@1x/@2x/@3x）透明底白色闪电，再开启 `IsTemplateIcon` | 需设计资源 |
| 菜单栏鼠标移上去不显示 tooltip | Avalonia 的 macOS 后端通过 `[[statusItem button] setToolTip:]` 设置，但状态栏按钮通常不展示 tooltip，属系统行为 | 无解，非 bug |
| macOS 上左键点菜单栏图标**不**同时弹 HUD | macOS 后端不派发 `TrayIcon.Clicked`，无法区分"点了图标"。用菜单里的「预览电量 HUD」替代 | Windows 行为保留不变 |
| 菜单栏图标不做亮度自适应 | 同上（未 template 化）；当前图标在浅色/深色菜单栏下都是深底白闪电方块，可辨识 | 同第 1 条 |
| pmset 兜底路径下 mWh 为名义值 | `pmset` 不提供容量，只能用百分比 + 名义 100Wh 换算；该路径仅在 IOKit 完全不可用时走到 | 可改为解析 `ioreg` 文本兜底 |
| 轮询间隔 5s | macOS 用 push 通知承担主路径，轮询只是兜底，故从 Windows 的 2s 放宽到 5s | 如需更快兜底可调 `MacOSPowerMonitor.PollInterval` |
| 分发需绕过 Gatekeeper | 当前只有 Apple Development 证书，无 Developer ID → 无法公证。见 [macos-distribution.md](./macos-distribution.md) §6.2 | 采购 Developer ID |

---

## 6. 踩坑与修复记录

### 1. 无边框透明窗口在 macOS 上显示整块黑底（已修复）

**现象**：HUD 弹出时背后有一块 1200×160 的黑色矩形，胶囊看起来浮在黑框里。

**根因**：`Background="Transparent"` 只影响 Avalonia 的绘制层，**不改变 NSWindow 自身**。
Avalonia 的 macOS 后端把窗口背景色绑定在 `TransparencyLevelHint` 上：

```objc
// native/Avalonia.Native/src/OSX/WindowBaseImpl.mm
HRESULT WindowBaseImpl::SetTransparencyMode(AvnWindowTransparencyMode mode) {
    [Window setBackgroundColor: (mode != Transparent ? [NSColor windowBackgroundColor] : [NSColor clearColor])];
}
```

`TransparencyLevelHint` 默认是**空列表** → `TransparencyLevel` 保持 `None` →
`[NSColor windowBackgroundColor]`（不透明系统底色，深色模式下就是黑色）。

另外 `TopLevel.HandleTransparencyLevelChanged` 会在 `ActualTransparencyLevel == None` 时
把模板里的 `PART_TransparencyFallback` 边框填成 `TransparencyBackgroundFallback`，
默认是 `Brushes.White` —— 这也是必须把它一起设为 `Transparent` 的原因。

**实测对照**（同一份代码，仅改 `TransparencyLevelHint`）：

| 设置 | `ActualTransparencyLevel` | `NSWindow.backgroundColor == [NSColor clearColor]` |
|---|---|---|
| 不设（默认空列表） | `None` | **`False`** → 不透明黑底 |
| `TransparencyLevelHint="Transparent"` | `Transparent` | **`True`** → 透出桌面 |

**修复**（`Views/HudWindow.axaml`）：

```xml
SystemDecorations="None"
Background="Transparent"
TransparencyLevelHint="Transparent"
TransparencyBackgroundFallback="Transparent"
```

**回归保护**：`MacOSPlatformServices.OnHudWindowShown` 每次显示 HUD 都会断言窗口背景，
不透明时写 `WARN ... 窗口背景不透明（会显示为黑底）`，日志里的 `transparent=True` 即为通过。

> Windows 侧不受影响：`TransparencyLevelHint` 在 Windows 上映射为分层窗口，行为等价；
> 该属性是跨平台写法，两端可共用。

### 2. 同一物理动作连弹两次 HUD，后一次把前一次动画掐断（已修复）

**现象**：插拔电源时"大概率先显示 Simple Capsule 一下，再播 Full Sequence（甚至不播）"。

**根因**：macOS 在插拔电源时，除了上报 AC 变化，还会**紧接着自动切换低电量模式**，
于是同一个物理动作触发两次 HUD：

```
23:58:56.331  low power mode OFF → 通知订阅方     ← 副作用事件
23:58:56.737  AC connected       → 通知订阅方     ← 真实动作（滞后 406ms）
```

第二次进入 `HudWindow.ShowAndPlayAsync` 时执行的 `_cts.Cancel()` 会把第一次的动画**中途掐断** ——
插电时看到的是"简化胶囊闪一下 → 完整三态"，拔电时看到的可能是"简化胶囊闪一下 → 又一遍简化胶囊"。

实测滞后时长：**402 / 674 / 674 / 2128 / 2863 ms**（跨度很大，单纯加去抖窗口吸收不完）。

**修复**：新增 `Platform/HudDisplayCoordinator.cs`，把同一动作的多条事件合并为一次播放。

| 机制 | 说明 |
|------|------|
| 优先级 | 交流电变化（高） &gt; 低电量模式变化（低）。两者都到时只播交流电那次 |
| 自适应稳定窗 | 交流电 120ms（保证插拔手感）；低电量模式 1100ms（吸收滞后副作用） |
| 副作用抑制窗 | 交流电派发后 4s 内到达的低电量模式事件判定为副作用直接丢弃，**仅 macOS 启用**（Windows 的省电模式不会被电源切换改写，手动开关必须照常提示） |
| 同类合并 | 同类事件连发只顺延稳定窗，不新增待播项 |

**真机验证结果**（连续 7 次插拔，日志见 `~/Library/Logs/EndfieldCharge/`）：

```
23:58:56.737  AC connected      → AcConnected 压制了 1 个低优先级待播项 → 派发 AcConnected
23:59:03.838  AC disconnected   → AcDisconnected 压制了 1 个低优先级待播项 → 派发 AcDisconnected
23:59:12.425  AC connected      → 派发 AcConnected
23:59:15.408  saver OFF（滞后 2863ms）→ 判定为电源切换副作用，丢弃（不播）
23:59:21.640  AC disconnected   → AcDisconnected 压制了 1 个 → 派发 AcDisconnected
23:59:41.596  AC connected      → AcConnected 压制了 1 个 → 派发 AcConnected
23:59:50.623  AC disconnected   → AcDisconnected 压制了 1 个 → 派发 AcDisconnected
23:59:59.674  AC connected      → 派发 AcConnected
```

**每次物理动作恰好一次派发**：4 次插电全部派发 `AcConnected`（完整三态），
3 次拔电全部派发 `AcDisconnected`（简化胶囊），无一次被后续事件打断。

**回归测试**：`tests/EndfieldCharge.Tests/HudDisplayCoordinatorTests.cs`，10 个用例全部通过：

| 用例 | 锁定行为 |
|------|----------|
| `PlugIn_WithLaggedSaverOff_PlaysOnlyFullSequence` | 插电 + 402ms 后自动关低电量模式 → 只播完整三态 |
| `Unplug_WithLaggedSaverOn_PlaysOnlySimpleCapsule` | 拔电 + 674ms 后自动开低电量模式 → 只播简化胶囊 |
| `SaverArrivesBeforeAc_AcWins` | 副作用事件反超到 AC 之前仍以 AC 为准 |
| `SaverFarAfterAc_StillPlays` | 超出 4s 相关窗的低电量模式事件必须照常播（防止手动开关静默失效） |
| `ManualSaverToggle_Plays` | 纯手动开关低电量模式照常播放 |
| `Windows_DoesNotSuppressSaverAfterAc` | Windows 不启用副作用抑制 |
| `RepeatedSameKind_CoalescesToSinglePlay` | 同类事件连发只播一次 |
| `AcDisplayLatency_StaysWithinBudget` | 插拔播放在 120ms 稳定窗后立即发生 |
| `AfterDispose_NoDispatch` | 退出后不再派发 |

```bash
dotnet test tests/EndfieldCharge.Tests/EndfieldCharge.Tests.csproj
```

### 3. 刘海出生与返回的坐标约定

旧版把嵌套 Center 容器当成额外缩放偏移、又给水平中心添加补偿，造成出生偏右。
现改用布局完成后的 `Pill.TranslatePoint` 测量顶边中心。BirthHost 在 GlobalScale 外，
位移直接等于目标点减测量点，不除以 UI 缩放；顶边中心缩放不会额外移动该点。

透明窗口覆盖目标屏幕顶部及运动路径，先提升 NSWindow 层级，再用
`setFrameTopLeftPoint:` 定位。只设置 Avalonia Position 会被工作区约束到菜单栏下方。
出生条完整位于刘海高度内，先下移离开刘海，再展开到左/中/右落点，返回时反向运动。
左右位置按实际胶囊宽度计算，边缘留 20 DIP。NSScreen 按屏幕左上角两轴匹配。

本机原生窗口运行检查（1512pt 屏宽，强制出生预览）通过：

| 配置 | 窗口原点 | 出生/返回中心 X | 展开中心 X（左/中/右） |
|---|---|---|---|
| 完整动画，缩放 0.8 | (0,0) | 756 | 244 / 756 / 1268 |
| 简化动画，缩放 1.2 | (0,0) | 756 | 356 / 756 / 1156 |

几何测试覆盖刘海内部边界、多种缩放、左右落点及无刘海禁用。
多显示器混合 DPI、全屏 Space 和视觉节奏仍需手工验收。

### 4. `TransparencyLevelHint` 无法写在 `Window` 标签的注释里

XAML 的属性区域内不能夹注释（`Background="Transparent" <!-- ... -->` 会导致 XML 解析失败）。
说明文字必须放在 `<Window ...>` 之后、内容元素之前。
