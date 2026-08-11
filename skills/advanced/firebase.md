---
name: firebase
description: Firebase 接入与 Crashlytics 排查（Manual：用户要求"配置 Firebase"、"看崩溃后台"时引用）
activation: manual
---

# Purpose

Firebase 全家桶（BOM、Crashlytics、Messaging、Analytics、AppDistribution）在本项目的接入方式与常见问题。

# MUST

1. **接入现状**：
   - Firebase BOM：`platform(libs.firebase.bom)`（32.7.0）已在 app/build.gradle 引入
   - 已启用模块：
     - `firebase-analytics` → 应用埋点
     - `firebase-messaging:23.4.0` → 推送（与 `wdpushlibrary` 双通道）
     - `firebase-crashlytics:18.6.0` → 崩溃采集
   - Gradle 插件：根目录 build.gradle 中 `com.google.gms:google-services:4.4.4`、`com.google.firebase:firebase-crashlytics-gradle:3.0.6`
   - 配置文件：`app/google-services.json`（主 module）+ flavor 专属 `app/src/beta/agconnect-services.json` / `app/src/official/agconnect-services.json`（华为 HMS 配置）
2. **google-services.json 多环境**：
   - beta / official 两个 package name 不同（beta 加 `.debug` 后缀）
   - `google-services.json` 中必须同时包含两个 client（或对应 flavor 目录放置独立 JSON）
3. **Crashlytics mapping 上传**：
   - 当前 `firebaseCrashlytics { mappingFileUploadEnabled false }`（debug/release 均关）
   - 要开启在对应 buildType 里设 `true`，或 CI 环境变量 `FIREBASE_APP_ID` + upload crashlytics 任务
4. **FCM 推送通道**：
   - FirebaseMessagingService 子类（wdpushlibrary 内）处理 token 与消息到达
   - Android 13+ 必须动态申请 `POST_NOTIFICATIONS` 权限，否则 push 收不到
5. **Analytics 埋点事件**：
   - 事件名/参数名统一全大写下划线（`login_success`、`chat_send_msg`）
   - 不允许带用户手机号、UID 明文到参数（匿名化 user_id = hash）

# MUST NOT

1. 不要把 firebase-crashlytics:3.x Gradle 插件和老版本 `io.fabric` 插件混开（Fabric 已废弃）
2. 不要在 `google-services.json` 只放一个 client（会导致 beta/official 其中一个 flavor GMS 初始化失败）
3. 不要在 release 构建漏了 `apply plugin: 'com.google.gms.google-services'`（放在 app/build.gradle 顶部）
4. 不要把 Crashlytics 自定义键（CustomKey）里塞完整聊天内容、用户手机号等 PII 信息（只带聊天 ID / 会话 ID 关联）

# SHOULD

1. **Crash-free 监控**：Firebase Console → Release & Crash → Crash-free users 目标 ≥ 99.5%
2. **自定义键（CustomKeySamples）**：当前项目 `crash/CustomKeySamples.kt` 已示例打点方式，崩溃时附带上 UID / 版本号 / 最后路由页 / 网络状态，加快定位
3. **App Distribution（可选）**：beta 包通过 Firebase App Distribution 分发内测，比手传群文件更合规（但未接入，接入需加 `firebase-appdistribution-gradle` 插件）
4. **Performance Monitoring（可选）**：需要冷启动/网络耗时监控时引入 `firebase-perf`，与 buildTrace.gradle 不冲突

# Decision

| 问题 | 排查 |
|---|---|
| FCM 收不到通知 | 1) token 回调是否拿到；2) POST_NOTIFICATIONS 权限是否开启；3) 前台/后台 channel 是否匹配；4) VPN/网络是否拦截 Firebase 长连接 |
| Crashlytics 没有数据 | 1) mappingFileUploadEnabled 是否开启；2) google-services.json 对应当前包名；3) 看 Logcat tag `FirebaseCrashlytics` 初始化日志 |
| Analytics 事件无上报 | 1) DebugView `adb shell setprop debug.firebase.analytics.app <package>` 开启；2) 事件名是否超过 40 字符 / 非法字符 |
| Google Login 失败 | 1) SHA-1/SHA-256 是否在 Firebase Console 配置对应 release/debug keystore；2) 支持 Web 客户端 ID |
