---
name: startup
description: 启动优化知识（Auto：用户提到启动慢、秒开、Splash 优化时自动触发）
activation: auto
triggers:
  - "启动慢"
  - "启动速度"
  - "冷启动"
  - "秒开"
  - "splash"
---

# Purpose

应用冷启动（进程从 0 开始）速度优化：从 Zap 启动、Splash、首帧渲染到首页对话列表可用的全链路。

# MUST

1. **Application.onCreate 严禁阻塞**：
   - 所有初始化必须通过 `init/AppInitManager.kt` 分发三档：
     - `MAIN_THREAD`：仅首屏必须（极少，如 MMKV、Log 初始化）
     - `BACKGROUND`：线程池异步（网络/推送/三方 SDK）
     - `LAZY`：首次调用时懒加载（RTC、直播、翻译、AI 数字人等低频模块）
   - 严禁直接在 MyApplication onCreate 串行写大量 init 代码
2. **Splash 机制**：
   - `AndroidX SplashScreen` (`libs.splashScreen`) 已接入
   - `SplashActivity` 用 `@style/SplashScreenTheme`，不要在 onCreate 里 setContentView 做大量 inflate
3. **主线程不做 I/O**：启动阶段（Application → Splash → MainActivity）严格禁止读 DB / SP / 资产文件到主线程
4. **ContentProvider 检查**：Firebase / Facebook / 第三方 SDK ContentProvider 自动初始化是否被滥用，不需要的通过 manifest `tools:node="remove"` 移除

# MUST NOT

1. 不要在 `MainActivity.onCreate` 里串行等待网络请求返回才渲染（首页骨架屏 / 占位数据先上）
2. 不要在启动阶段加载多 dex 阻塞：`multidex` 2.0.1 已配置 DEX 优化，注意方法数不要无限制增长
3. 不要在 `SplashActivity` 里做初始化，它只负责品牌跳转，初始化放到 Application + AppInitManager

# SHOULD

1. **启动打点**：
   - `AppInitManager` 每步 `*Init.kt` 前后打耗时日志（SystemClock.elapsedRealtime()），发布前看 Top N 耗时
   - 可用 `buildTrace.gradle` 看构建各 Task 耗时对 CI 的影响
2. **提前渲染**：
   - 首页 MainActivity 布局若复杂，用 `ViewStub` 懒加载次级面板（社群/发现/我的 tab）
   - 聊天列表用预加载 ViewHolder / Glide 预热缓存
3. **Baseline Profile / ProfileInstaller**：`profileinstaller` 1.3.0 已引入，关键启动+聊天路径可编写 Baseline Profile 提前编译
4. **DEX 布局优化**：release R8 开启 D8 优化 + `minifyEnabled true`（官方包已开）

# Decision

| 启动阶段 | 常见瓶颈 | 优化手段 |
|---|---|---|
| Zap (fork 到 Application) | 多 ContentProvider 自动初始化 | tools:node="remove" 不用的 / 禁用 App Startup 不需要的 |
| Application.onCreate | 串行 init 太多 | AppInitManager BACKGROUND / LAZY 拆分 |
| Splash → MainActivity | 布局 inflate 巨复杂 | ConstraintLayout 扁平化 / ViewStub / AsyncLayoutInflater |
| Main 首帧 | 会话列表数据 DB 读阻塞 | Room 后台预取 + 首屏占位（骨架屏） |
| 首次对话可用 | Glide 头像冷加载 | preload 会话列表前 30 个头像 |
