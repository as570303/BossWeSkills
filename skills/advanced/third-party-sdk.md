---
name: third-party-sdk
description: 第三方 SDK 接入规范与本项目高频库使用指南（Manual：用户要求"接入某 SDK"、"升级某依赖"时引用）
activation: manual
---

# Purpose

当需要接入新第三方 SDK 或升级已有库时的选型、版本兼容、混淆规则、依赖冲突排查指南。

# MUST

1. **依赖版本源**：所有三方库版本集中在 `gradle/libs.versions.toml` 的 `[versions]` 与 `[libraries]` 块。新增/升级三方库必须先改 TOML，不要 `build.gradle` 中写死版本号字符串。
2. **依赖仓库顺序**：build.gradle `allprojects.repositories` 已配置 aliyun / google / mavenCentral / jitpack / 华为 / nexus（私仓），顺序决定解析优先级。私有 .aar 优先走 flatDir（`utilslibrary/libs/`、`wdlivelibrary/libs/`）。
3. **版本兼容矩阵（高频冲突区）**：
   | 家族 | 建议版本对齐 | 说明 |
   |---|---|---|
   | Kotlin / Coroutines / Compose | Kotlin 2.1.0 ↔ coroutines 1.5~1.8（当前 1.5.2，升级需谨慎测试） | Compose 未引入，若引入必须对应 Kotlin 版本（2.1.0 → compose-compiler 1.5.x） |
   | OkHttp / Retrofit / Okio | Retrofit 2.9.0 ↔ OkHttp 4.8.1 ↔ Okio 2.x（同系列同小版本） | 禁止 OkHttp3 + 4 混用，统一 4.x |
   | Lifecycle / ViewModel / LiveData | 全局统一 `lifecycleVersion = "2.8.0"`（libs.versions.toml） | 不要混 2.5 / 2.6 / 2.8 |
   | Room / SQLite | Room 2.6.1 ↔ sqlite 2.4.0（已配置） | Room 升级必须写 Migration |
   | Media3 / ExoPlayer | media3 = "1.4.1"（全局统一） | ExoPlayer 旧包 `com.google.android.exoplayer` 已迁到 Media3 |
   | LiveKit / WebRTC | livekit-android 2.23.1.3 ↔ webrtc-sdk 125.6422.07（项目已排除 `io.livekit:livekit-android` 转用 com.wjxls 定制） | 不要直接升级 io.livekit 官方包，先过 wjxls 定制版测试 |
4. **新增 SDK 四件套**（接入完整才上线）：
   - [ ] `consumer-rules.pro`（模块独立 proguard 规则，release minify 不崩）
   - [ ] Manifest 权限 & `<queries>`（targetSdk 30+ 包可见性）
   - [ ] ProGuard/R8 keep 规则写入模块 `proguard-rules.pro`（开源 SDK 一般 README 给）
   - [ ] README / Wiki 记录接入版本、回调入口、前台服务类型（如果用到）
5. **排除冲突**：`configurations.all { exclude group: 'xxx', module: 'yyy' }` 统一写在 app/build.gradle（项目已有多组 exclude：markwon core/image-glide/ext-tables、livekit-android 等），不要散落在各模块。

# MUST NOT

1. 不要为了一个小功能引入大型 SDK（例如"要一个圆角 drawable"就拉 com.github.getActivity 全家桶）→ 先查 utilslibrary / widgetlibrary 是否已有实现
2. 不要同时接入两套功能重叠 SDK（Glide + Fresco + Picasso / Retrofit + Ktor / Room + GreenDAO），保留主方案，另一个逐步移除
3. 不要用 `implementation 'com.xxx:sdk:+'` 动态版本号（每次构建结果不可复现，CI 必挂或引入未知 CVE）
4. 不要忽略 `minSdk` 兼容：接入 SDK 前查它 `minSdk` 是否 ≤ 24（本项目 minSdk=24），否则会 `uses-sdk` merger 失败
5. 不要引入纯 Java 过时库（如 `android.support.v4` 老包），必须 AndroidX 迁移版

# SHOULD

1. **升级前查 CHANGELOG**：跨大版本（如 Glide 4.x→5.x、Room 2.4→2.7、Kotlin 1.9→2.1）先看官方 Migration Guide
2. **依赖冲突排查命令**：
   ```bash
   # 看某个依赖的完整引入链
   ./gradlew :app:dependencyInsight --dependency okhttp --configuration officialReleaseRuntimeClasspath
   # 导出全部依赖树（冲突发群里给同事排查用）
   ./gradlew :app:dependencies > deps.txt
   ```
3. **.aar / .jar 本地依赖**：优先用 maven 依赖，不建议本地 libs/*.aar（打渠道包体积 / 升级都麻烦）。实在需要放 flatDir：
   - 放对应模块 `libs/`，不要堆在 `app/libs/`
   - jar/aar 文件名带版本号（`toastlibrary-1.1.7.aar` 优于 `toast.aar`）
4. **License 合规**：商业 SDK（FaceUnity、万得 livekit、WebRTC 商业版）确认合同范围 & 到期日期，到期前 1 个月提醒升级。

# Decision

| 场景 | 选择库 |
|---|---|
| 网络请求 (HTTP) | Retrofit 2.9 + OkHttp 4（不换 Ktor） |
| IM 长连接 (TCP) | Netty 4.1 + socketlibrary（不换 OkHttp WebSocket） |
| 图片加载 | Glide 4.12 + Transformations（不换 Fresco/Coil） |
| 列表刷新 | SmartRefreshLayout 2.x（不换 SwipeRefreshLayout） |
| 通用弹窗 | XPopup 2.x（不重复造轮子） |
| 路由 / 深链 | GoRouter 2.5.2（不换 ARouter/Navigation） |
| 数据选择器（日期/地址） | AndroidPicker（gzu-liyujiang）+ PickerView（contrarywind） |
| 视频播放（本地/网络非通话） | GSYVideoPlayer / jiaozivideoplayerlibrary |
| 直播弹幕 | DanmakuFlameMaster 0.9.25（依赖 libndkbitmap.so，注意 16KB 对齐） |
| 录制 / 剪辑 | FFmpegKit 5.1.LTS（gpl） / ffmpeg-kit-full-gpl |
| 推送通道 | 本项目双通：Firebase FCM + wdpushlibrary（自研长连） |
