# NFC 跳转助手（NfcLauncher）

HarmonyOS NEXT 平台的 NFC 标签 → 应用/深链 自动跳转工具。

绑定 NFC 标签 UID 与目标应用包名或直达链接，贴卡即可自动拉起目标；内置 NDEF 读取识别、空白卡写入、Embedded 嵌入模式等辅助功能。

---

## 功能特性

| 功能 | 说明 |
|---|---|
| **标签绑定** | 将 NFC 标签 UID 与应用包名/Ability 名 或直达链接（`alipays://`、`https://` 等）关联 |
| **自动跳转** | 开启开关后，贴近已绑定标签即自动拉起目标应用 |
| **直达链接** | 支持深链跳转（如支付宝小程序 `alipays://platformapi/startapp?appId=…`），优先于包名匹配 |
| **识别模式** | 贴近任意 NFC 卡，解析并展示 NDEF 记录内容（URI、TEXT、AAR 等），自动回填到绑定表单 |
| **写卡模式** | 将直达链接作为 NDEF URI 记录写入空白卡（NTAG213/215 等），复制出行为一致的标签 |
| **模拟触发** | 无需实体卡，直接按输入的标签 ID 执行完整跳转流程 |
| **Embedded 嵌入** | 可选 `windowMode: EMBEDDED` 参数，将目标应用嵌入本应用窗口，绕过应用切换拦截 |
| **持久化存储** | 绑定关系使用 `@kit.ArkData.preferences` 本地保存，列表可删除、点击回填编辑 |

---

## 技术栈

| 层级 | 依赖 |
|---|---|
| UI | ArkTS + `@kit.ArkUI` |
| NFC | `@kit.ConnectivityKit.tag` — 前台派发、NDEF 读写、URI 前缀压缩 |
| 跳转 | `@kit.AbilityKit.common.UIAbilityContext.startAbility` — 隐式 Want 分发 |
| 存储 | `@kit.ArkData.preferences` |
| 日志 | `@kit.PerformanceAnalysisKit.hilog` |
| 平台 | HarmonyOS NEXT API 12+（5.0.0） |

---

## 环境要求

- **DevEco Studio** 5.0.0 Release 及以上
- **HarmonyOS SDK** API 12 及以上
- **真机测试**：需要搭载 HarmonyOS NEXT 且支持 NFC 读取的设备（模拟器无 NFC 硬件）
- **NFC 卡**：NTAG213 / NTAG215 / NTAG216 / Mifare Classic 1K 等常见类型

---

## 快速开始

### 1. 克隆项目

```bash
git clone <repo-url>
cd NfcLauncher
```

### 2. 在 DevEco Studio 中打开

```
File → Open → 选择 NfcLauncher 目录
```

### 3. 配置签名

```
File → Project Structure → Signing Configs
  → 勾选 "Automatically generate signature"
  → 选择自动生成的 profile
```

### 4. 连接真机并运行

手机开启开发者模式 → 允许 USB 调试 → 点 DevEco 绿色 Run 按钮。

### 5. 首次使用流程

```
① 打开应用，状态栏显示「NFC 已就绪」
② 贴近一张 NFC 卡 → 标签 ID 自动填入
③ 填写目标应用的包名 + Ability 名，或直接填直达链接
④ 点「保存绑定」
⑤ 开启「贴近已绑定标签时自动跳转」开关
⑥ 再次贴同一张卡 → 自动拉起目标应用
```

---

## 项目结构

```
NfcLauncher/
├── AppScope/
│   ├── app.json5                 # 应用级配置
│   └── resources/                # 应用图标、字符串
├── entry/
│   ├── src/main/
│   │   ├── ets/
│   │   │   ├── common/
│   │   │   │   ├── BindStore.ets       # 绑定关系 Preferences 存取 + UID→HEX
│   │   │   │   └── NdefUtil.ets        # NDEF 记录解析器（URI/TEXT/AAR/MIME）
│   │   │   ├── pages/
│   │   │   │   └── Index.ets           # 主页面（全部功能入口）
│   │   │   └── entryability/
│   │   │       └── EntryAbility.ets    # 应用入口
│   │   ├── module.json5                 # 模块声明 + ohos.permission.NFC_TAG
│   │   └── resources/                   # 页面资源
│   └── build-profile.json5
├── hvigor/
├── hvigorfile.ts
├── build-profile.json5
└── oh-package.json5
```

---

## 核心实现说明

### 前台派发注册（生命周期绑定）

HarmonyOS NFC 前台派发在应用退后台后会被系统**自动失效**，页面回到前台时不会重建。因此注册/注销必须绑定到 `onPageShow()` / `onPageHide()` 而非 `aboutToAppear()`：

```typescript
onPageShow(): void  { this.registerNfc(); }
onPageHide(): void  { this.unregisterNfc(); }
```

### 跳转链路

```
真实贴卡：
  系统 NFC 服务 (system_server)
    → registerForegroundDispatch 回调
    → handleTag() 查绑定表
    → launchApp()
        ├─ 有直达链接 → startAbility({ action: 'viewData', uri })
        └─ 有包名     → startAbility({ bundleName, abilityName })

模拟触发：
  按钮点击 → simulateTag() → handleTag()
  （同 launchApp，forceLaunch=true 忽略自动跳转开关）
```

### 写卡流程

```
写卡模式开启 → 贴近空白卡
  → tag.getNdef(tagInfo)
  → isNdefWritable() 校验
  → tag.ndef.makeUriRecord(uri)    // 自动处理 URI 前缀压缩
  → tag.ndef.createNdefMessage()
  → ndefTag.connect() + writeNdef()
```

### NDEF 解析

[NdefUtil.ets](entry/src/main/ets/common/NdefUtil.ets) 手动实现了 URI 前缀表（RTD_URI）、TEXT 记录语言码解码、AAR 包名提取等逻辑，不依赖额外库。

---

## 常见目标应用包名

| 应用 | bundleName | abilityName |
|---|---|---|
| 支付宝（鸿蒙版） | `com.alipay.mobile.client` | `EntryAbility` |
| 微信 | `com.tencent.wechat` | `EntryAbility` |
| QQ | `com.tencent.mqq` | `EntryAbility` |
| QQ 音乐 | `com.tencent.hm.qqmusic` | `EntryAbility` |
| QQ 邮箱 | `com.tencent.qqmail.hmos` | `EntryAbility` |
| 淘宝 | `com.taobao.taobao4hmos` | `EntryAbility` |
| 闲鱼 | `com.taobao.idlefish4ohos` | `EntryAbility` |

> 精确查询：`hdc shell bm dump -a | findstr /i <应用名>`

---

## 直达链接示例

| 目标 | 链接 |
|---|---|
| 支付宝小程序 | `alipays://platformapi/startapp?appId=小程序ID` |
| 通用 https | `https://render.alipay.com/p/s/ulink/axdk?s=dc&scheme=...` |
| 应用内页面 | `intent://#Intent;scheme=自定义scheme;...` |

小程序 ID 获取：支付宝内打开目标小程序 → 右上角「···」→ 复制分享链接 → 提取 `appId=` 后的数字。

---

## 已知限制

### 三方应用调用身份限制

`startAbility` / `openLink` 由三方应用发起时，Want 会携带调用方 bundleName 和 uid。部分应用（如支付宝 NFC 碰一碰路由）会据此判断来源，只信任 `system_server` 或白名单调用方。

**绕过方案**：用户真实贴卡（系统 NFC 服务分发）→ 调用方为 system_server → 可信通过。应用内模拟跳转无法在代码层面伪造调用方身份。

### hdc shell 辅助

```bash
# 手动触发与真机贴卡等价的系统级分发
hdc shell aa start -A ohos.want.action.viewData -U <链接>
```

这条命令跑在 shell 用户（uid=2000）下，AMS 分发时仍会标记调用方身份——但 shell 有 `START_ABILITIES_FROM_BACKGROUND` 等特权，某些场景下可能绕过三方拦截，适用于开发调试。

### Embedded 嵌入模式

部分应用不支持被嵌入窗口（`WINDOW_MODE_EMBEDDED`），会忽略该参数回落到全屏显示。能否嵌入取决于目标应用窗口组件配置。

### API 版本兼容

- `promptAction.openToast` 从 API 18 开始支持，当前项目最低 API 12，已加 `deviceInfo.sdkApiVersion` 版本检测静默降级
- 真机测试需 HarmonyOS NEXT 5.0+，旧 Android 手机无法运行 ArkTS 应用

---

## 权限声明

[module.json5](entry/src/main/module.json5) 中申请了 `ohos.permission.NFC_TAG`（normal 级权限，安装时自动授予，无需运行时弹窗）。

---

## 写卡注意事项

- **NTAG213/215/216**：完全兼容，容量分别 144/504/888 字节
- **Mifare Classic**：旧款卡片需先格式化为 NDEF 格式
- **已锁定卡**：`isNdefWritable()` 返回 false，无法写入
- **写入后不可撤销**：操作前确认链接正确；如需恢复空白，可写入一条空 TEXT 记录

---

## License

MIT
