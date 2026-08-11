---
name: architecture-coding
description: 架构分层与模块协作规范（FileMatch：**/mvp/**, **/repository/**, **/manager/**, **/net/**, **/service/**）
activation: fileMatch
filePattern:
  - "**/mvp/**/*.kt"
  - "**/mvp/**/*.java"
  - "**/repository/**/*.kt"
  - "**/repository/**/*.java"
  - "**/manager/**/*.kt"
  - "**/manager/**/*.java"
  - "**/service/**/*.kt"
  - "**/service/**/*.java"
  - "**/net/**/*.kt"
  - "**/net/**/*.java"
---

# Purpose

架构分层规范，定义 UI / Presenter / ViewModel / Repository / DataSource 之间的协作边界。

# MUST

1. **UI 不直接访问 DataSource**：Activity/Fragment 不能直接 new `Retrofit Api`、不能直接读写 `DBManager`
   - 必须通过 Presenter / ViewModel → Repository（或 Manager）间接访问
2. **ViewModel 不持有 View 引用**：ViewModel 严禁持有 Activity/Fragment/View/Context（用 `AndroidViewModel` 仅限 Application）
3. **跨模块路由**：一律走 GoRouter（路径常量统一管理），禁止 `Intent(this, OtherModuleActivity::class.java)`
4. **Manager 单例初始化**：在 `init/` 包对应模块的 `*Init.kt` 中注册到 AppInitManager，不要在 Activity 里懒加载
5. **EventBus 事件命名**：继承自 `*Event`（`MessageEvent`、`AddFriendOkEvent`），发送后必须在生命周期解绑

# MUST NOT

1. 不要在 Activity/Fragment/Presenter 里嵌套复杂的业务 if-else，抽到 Manager/UseCase 函数
2. 不要跨模块直接依赖实现类（通过 GoRouter 路由 / 接口注册表 InterfaceRegistry）
3. 不要把数据库 Cursor / Netty Channel 之类底层对象传到 UI 层
4. 不要在 Application `onCreate()` 塞初始化逻辑，统一到 `init/` 分模块（支持懒加载/异步）

# SHOULD

1. **MVP 模块**：Contract 定义 View 与 Presenter 接口，Model 走 net/DB Repository
2. **新增模块优先 MVVM**：ViewModel + LiveData/Flow + Repository，不再写新的 MVP Contract
3. **复杂业务逻辑**：建议创建 UseCase（如 `SendMessageUseCase`），把 Presenter/ViewModel 中的长函数拆出来
4. **数据源协调**：Repository 负责「本地缓存 + 网络 + Socket」三路数据合并策略，UI 层只看单一入口
5. **事件通信**：优先 LiveData/Flow 观察（VM→UI），跨组件用 EventBus，大范围解耦用 Channel（`eventChannel/EventChannel.kt`）

# Decision

| 场景 | 选择 |
|---|---|
| 简单单页面（列表+刷新） | UI → ViewModel → Repository（直接，不必 UseCase） |
| 复杂业务（发消息、支付、建群） | UI → ViewModel → UseCase → Repository |
| 老模块维护 | 沿用既有 MVP 结构，不整页重构为 MVVM |
| 跨模块数据回调 | InterfaceRegistry 注册接口 / GoRouter 带 result 返回 |
| 组件生命周期事件（前后台） | AppForegroundInit + ActivityLifeSdkApplication |
