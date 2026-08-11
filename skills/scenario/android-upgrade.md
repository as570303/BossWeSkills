---
name: android-upgrade
description: Android SDK 升级适配（Auto：用户提到升级 compileSdk/targetSdk、适配 Android 15/16 时自动触发）
activation: auto
triggers:
  - "升级 SDK"
  - "compileSdk"
  - "targetSdk"
  - "Android 15"
  - "Android 16"
  - "适配 Android"
  - "API 35"
  - "API 36"
---

# Purpose

当 `compileSdk / targetSdk` 升级（如 34→35→36…）时，对照本项目已踩坑记录做系统化适配，避免漏改回归。

# MUST（已知升级项清单）

### 至 compileSdk 35 / targetSdk 35（Android 15）当前已在进行
1. **原生库 16KB 对齐（Google Play 审核要求）**
   - 所有 `.so` 的 LOAD 段 `p_align ≥ 0x4000`
   - 检查命令：`readelf -l <so> | grep LOAD`
   - 影响目录：`app/src/main/jniLibs/{arm64-v8a,armeabi-v7a}/` 与 `livebusinesslibrary/src/main/jniLibs/`
2. **监听器签名非空化**
   - `CompoundButton.OnCheckedChangeListener.onCheckedChanged(CompoundButton, …)` → 首参非空
   - `RadioGroup.OnCheckedChangeListener.onCheckedChanged(RadioGroup, …)` → 首参非空
   - 历史已修复示例位置：LiveWindowActivity.kt / RtcContactSettingFragment.kt / ShopBackgroundActivity.kt
3. **NEARBY_WIFI_DEVICES 权限**
   - Manifest 必须声明 `<uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES" />`
   - 运行时在 WebRTC 音视频发现前动态申请
4. **前台服务类型显式声明**
   - Service 标签添加 `android:foregroundServiceType="microphone|remoteMessaging|dataSync"`（与实际功能匹配）
5. **OnBackInvokedCallback（预测式返回）**
   - Manifest Application：`android:enableOnBackInvokedCallback="true"`（当前已设置）
   - 代码：迁移所有 `override onBackPressed()` → `OnBackPressedDispatcher`

### 后续升级到 36（Android 16）必做
6. **预测式返回全面强制**：所有 Activity/Fragment 必须走 OnBackPressedDispatcher，重写 `onBackPressed` 会被 lint 标红/崩溃
7. **受限存储 / Full Intent 权限收紧**：`PendingIntent` 必须声明 `FLAG_IMMUTABLE`/`FLAG_MUTABLE`（当前 minSdk 24 已有要求，但 36 惩罚更严）
8. **通知 trampoline 限制收紧**：禁止从 BroadcastReceiver/Service 中转启动 Activity（直接 Activity pendingIntent）

# MUST NOT

1. 不要只升 `compileSdk` 不跑 `assembleOfficialRelease` 全量打包（release + R8 + 资源压缩 才会暴露问题）
2. 不要忽略三方 SDK 版本，升级前查每个 SDK 是否支持目标 API（尤其 WebRTC / LiveKit / 友盟 / Bugly / Firebase BOM）
3. 不要漏了 `build.gradle` 中 NDK 版本：ndkVersion 在 `libs.versions.toml` 集中维护，随 SDK 同步推荐版本

# SHOULD

1. **升级 checklist 化**：每次 SDK 升级创建单独分支，按本 Skill MUST 列表逐项打勾
2. **Flavor 双验证**：`assembleBetaDebug` + `assembleOfficialRelease` 各跑一次
3. **AAB 验证**：官方包 `bundleOfficialRelease` 产出 aab → 用 `bundletool` 装到测试机验证 so 对齐与动态下发
4. **回归冒烟**：登录 → 发消息 → 音视频通话 → 社群动态 → 钱包充值/提现 → 推送

# Decision

| 升级步骤 | 顺序 |
|---|---|
| 1 | 修改 `gradle/libs.versions.toml` 的 compileSdk / targetSdk / ndkVersion / AGP |
| 2 | 同步升级 Firebase BOM / Bugly / 友盟 / WebRTC / LiveKit 等关键 SDK 到支持新版本号 |
| 3 | 全量编译 `./gradlew assembleOfficialRelease`，处理 lint / compile 错误 |
| 4 | 按本清单检查 Manifest 权限 / 前台服务类型 / enableOnBackInvokedCallback |
| 5 | 检查 so 16KB 对齐（readelf 批量脚本） |
| 6 | 真机冒烟 4 场景：启动/登录、聊天、音视频、支付 |
| 7 | LeakCanary / Profiler 回归一遍（内存/卡顿基线对比） |
