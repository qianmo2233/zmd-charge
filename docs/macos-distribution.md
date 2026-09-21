# macOS 分发说明：签名、公证与打包

> 配套文档：[macos-port-assessment.md](./macos-port-assessment.md)（§ADR-3 / §ADR-4 / §Step 9）

---

## 1. 为什么 macOS 不能像 Windows 那样"丢一个 exe"

| Windows 做法 | macOS 对应机制 | 后果 |
|---|---|---|
| 单文件 `EndfieldCharge.exe` | 无此概念；应用必须是 `.app` 目录包 | 裸可执行文件无法被 LaunchAgent / Finder / 通知系统识别 |
| 注册表 `Run` 键自启 | 需要 bundle 路径 | 裸可执行文件路径在升级后失效 |
| 无签名 | Gatekeeper + 隔离属性（quarantine） | 从网络下载的未签名应用首次打开被拦截 |
| exe 图标 + 文件名 | `Info.plist` 的 `CFBundleIconFile` / `CFBundleName` | 菜单栏、Dock、关于面板的展示名都来自 plist |

结论：macOS 必须产出 `.app` bundle，并由 `Info.plist` 描述应用身份。

---

## 2. Bundle 结构

```
EndfieldCharge.app/
└─ Contents/
   ├─ Info.plist                      ← 应用身份与行为声明
   ├─ MacOS/
   │  └─ EndfieldCharge               ← dotnet publish 的 apphost（arm64 Mach-O）
   ├─ Resources/
   │  ├─ tray_bolt.png                ← 菜单栏图标（1x/2x/3x 命名变体可选）
   │  └─ AppIcon.icns                 ← 应用图标（可选）
   └─ _CodeSignature/                 ← codesign 后生成
```

`dotnet publish` 的全部输出（`*.dll`、`libAvaloniaNative.dylib`、`libSkiaSharp.dylib`、
`libHarfBuzzSharp.dylib`、`libcoreclr.dylib`、`libhostfxr.dylib` 等）都放进 `Contents/MacOS/`，
与 apphost 同目录 —— 这一点与 Windows 版"native dll 必须和 exe 同目录"的约束同源。

> ⚠️ 单文件发布（`PublishSingleFile`）在 macOS 上**不启用**：原生 dylib 无法打进单文件，
> 且 bundle 本身就是最好的"目录封装"，没有单文件的收益。

---

## 3. `Info.plist` 关键键

| 键 | 值 | 作用 |
|---|---|---|
| `CFBundleIdentifier` | `com.lenkmat.endfieldcharge` | 身份标识；LaunchAgent 的 `Label` 与之一致 |
| `CFBundleName` | `EndfieldCharge` | 菜单栏 / 关于面板显示名 |
| `CFBundleExecutable` | `EndfieldCharge` | `Contents/MacOS/` 下的可执行文件名 |
| `CFBundlePackageType` | `APPL` | 固定值 |
| `CFBundleShortVersionString` | `@@VERSION@@` | 用户可见版本；由打包脚本注入 |
| `CFBundleVersion` | `@@VERSION@@` | 构建号；由打包脚本注入 |
| `LSMinimumSystemVersion` | `13.0` | 最低系统版本 |
| **`LSUIElement`** | `true` | **不占 Dock、不进 Cmd-Tab**（菜单栏应用的关键键） |
| `NSHighResolutionCapable` | `true` | Retina 渲染 |
| `LSApplicationCategoryType` | `public.app-category.utilities` | 分类 |
| `NSHumanReadableCopyright` | `Copyright © 2026 Lenkmat` | — |

`LSUIElement` 与运行时的 `NSApplication.setActivationPolicy:Accessory` **同时使用**：
前者覆盖 bundle 启动场景，后者覆盖 `dotnet run` / 裸可执行文件的开发场景。

> 注意：`LSUIElement` 只在**真正的 bundle 启动**时生效。用 `open -a` 打开 `.app` 生效；
> 直接执行 `Contents/MacOS/EndfieldCharge` 时不会生效 —— 这正是运行时 `setActivationPolicy` 的价值。

---

## 4. 签名

### 4.1 三级签名方案

| 方案 | 命令 | 适用场景 | 局限 |
|---|---|---|---|
| ad-hoc | `codesign --force --deep --sign - EndfieldCharge.app` | 本机自用、CI 产物 | 其他机器下载仍被 Gatekeeper 拦 |
| Apple Development | `codesign --force --deep --options runtime --sign "Apple Development: ..." --entitlements ...` | 团队内测 | 不满足公证要求 |
| Developer ID Application | `codesign --force --deep --options runtime --sign "Developer ID Application: ..." --entitlements ...` | **正式分发** | 需开发者账号与证书 |

本机已有 `Apple Development: 烨 刘 (793F7H68S8)`，因此当前可自动完成前两级；
Developer ID 需后续在开发者账号中申请（不在本次范围）。

### 4.2 entitlements（`build/macos/EndfieldCharge.entitlements`）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <!-- .NET JIT 在 hardened runtime 下必需 -->
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
  <key>com.apple.security.cs.disable-library-validation</key><true/>
</dict>
</plist>
```

`.NET` 应用启用 hardened runtime 时不做上述声明会直接启动失败（JIT 被拒）。
本项目**不使用 App Sandbox**（读 IOKit 电池信息、写 `~/Library/LaunchAgents` 在沙盒内需额外 entitlement，
而本应用是工具类软件，用户自选分发渠道）。

### 4.3 公证（Notarization，仅 Developer ID 需要）

```bash
# 一次性：把 app-specific password 存进钥匙串
xcrun notarytool store-credentials "AC_PASSWORD" \
  --apple-id "<apple-id>" --team-id "<TEAMID>" --password "<app-specific-password>"

# 每次发布
ditto -c -k --keepParent EndfieldCharge.app EndfieldCharge.zip
xcrun notarytool submit EndfieldCharge.zip --keychain-profile "AC_PASSWORD" --wait
xcrun stapler staple EndfieldCharge.app
```

CI 中通过 Secrets（`APPLE_ID` / `TEAM_ID` / `APPLE_APP_PASSWORD` / `SIGNING_IDENTITY`）驱动；
secrets 缺失时脚本自动降级为 ad-hoc 签名并跳过公证，不阻塞构建。

---

## 5. 打包脚本行为（`scripts/package-macos.sh`）

```
1. 解析版本号：$VERSION 环境变量 → git tag (v*) → 0.0.<run_number> → 0.1.0 兜底
2. dotnet publish -c Release -f net10.0 -r <osx-arm64|osx-x64> -o build/macos/publish \
     -p:Version=$VERSION -p:UseAppHost=true -p:SelfContained=true -p:PublishSingleFile=false
3. 组装 EndfieldCharge.app/Contents/{MacOS,Resources}
4. 用 sed 把 @@VERSION@@ 替换为真实版本，写出 Contents/Info.plist
5. chmod +x Contents/MacOS/EndfieldCharge
6. 可选：codesign id -u 拿 UID → launchctl 兼容
7. 签名：有 SIGNING_IDENTITY 用之，否则 ad-hoc（-s -）
8. 校验：codesign --verify --verbose=2；plutil -lint Info.plist
9. 可选：hdiutil create -volname ... -srcfolder ... -ov -format UDZO EndfieldCharge-$VERSION.dmg
10. 可选：notarytool submit + stapler（仅 Developer ID）
11. 冒烟：Contents/MacOS/EndfieldCharge --selftest（必须在脚本内跑通才算打包成功）
```

第 11 步是本脚本的关键设计：**打包即冒烟**。若 IOKit 互操作在 CI runner 上因环境差异失败，
打包阶段就会红，而不是等用户下载后才发现。

---

## 6. 用户安装与首次打开

### 6.1 正常路径（已签名+公证）
1. 下载 `EndfieldCharge-x.y.z.dmg`
2. 打开 dmg，把 `EndfieldCharge.app` 拖入「应用程序」
3. 双击启动；菜单栏出现闪电图标

### 6.2 ad-hoc / 未公证路径（当前实际分发方式）
首次打开会被 Gatekeeper 拦截（提示"无法验证开发者"）。三种绕过方式：

```bash
# 方式 A：命令行去隔离（推荐，一次即可）
xattr -dr com.apple.quarantine /Applications/EndfieldCharge.app

# 方式 B：GUI —— 在「访达」中右键点击 App → 选择「打开」→ 在弹窗中再次点「打开」
# 方式 C：系统设置 → 隐私与安全性 → 底部「仍要打开」
```

必须在 README 与 Release 说明中**显式提供**这段说明，否则用户会认为应用损坏。

### 6.3 卸载
```bash
rm -rf /Applications/EndfieldCharge.app
rm -f  ~/Library/LaunchAgents/com.lenkmat.endfieldcharge.plist   # 若开过自启
rm -rf ~/Library/Application\ Support/EndfieldCharge
rm -rf ~/Library/Logs/EndfieldCharge
```

---

## 7. 自动更新与版本比较

`Services/UpdateChecker.cs` 用 `Assembly.GetName().Version` 与 GitHub Release 的 tag 比较。
因此 macOS 打包链路**必须**同时注入版本号到两处，否则会出现"应用内说自己没更新，但已发布新版"：

| 位置 | 注入方式 |
|---|---|
| 程序集版本 | `dotnet publish -p:Version=$VERSION` |
| `CFBundleShortVersionString` / `CFBundleVersion` | `package-macos.sh` 里 sed 替换 `Info.plist` 的 `@@VERSION@@` |

版本号来源与 Windows CI 保持一致：`v*` tag 用 tag 名，否则 `0.0.<run_number>`。

---

## 8. CI 要点（`.github/workflows/build.yml` 的 `build-macos` job）

```yaml
build-macos:
  runs-on: macos-latest        # Apple Silicon runner，与目标机型一致
  steps:
    - checkout
    - setup-dotnet 10.0.x      # macOS 侧依赖 net10 SDK
    - Resolve version          # 与 Windows job 相同的 tag 规则
    - dotnet restore
    - dotnet build -c Release  # 本机 TFM 自动为 net10.0
    - ./scripts/package-macos.sh   # 内含 publish + 组装 + 签名 + dmg + --selftest 冒烟
    - upload-artifact: EndfieldCharge.app, EndfieldCharge-*.dmg
    - Create GitHub Release（仅 v* tag，与 Windows 产物一起发布）
```

注意事项：
- `macos-latest` 为 arm64 runner，`--selftest` 能读到真实 IOKit 路径（runner 无电池，
  `AppleSmartBattery` 匹配会失败 → 应断言 `batteryPercent == null` 而非断言非空，这样才算真正覆盖无电池分支）。
- Windows job 不受影响（`runs-on: windows-latest` → `$(OS)` 为 `Windows_NT` → TFM 仍为 `net8.0-windows`）。
- 两个 job 的 artifact 名称必须区分（`EndfieldCharge-<ver>` / `EndfieldCharge-<ver>-macos`），
  否则 `upload-artifact` 会冲突。
