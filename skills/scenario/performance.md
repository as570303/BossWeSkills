---
name: performance
description: 性能优化知识（Auto：用户提到卡顿、慢、滑动掉帧时自动触发）
activation: auto
triggers:
  - "卡顿"
  - "掉帧"
  - "性能"
  - "慢"
  - "优化"
  - "ANR"
  - "RecyclerView"
---

# Purpose

IM 聊天列表、社群动态、会议等长列表场景下的流畅度优化，及应用冷/热启动速度优化。

# MUST

1. **先定位瓶颈再优化**：用 Android Studio Profiler（CPU / Memory / Network）确认热点函数，不拍脑袋优化
2. **主线程不做耗时**：
   - I/O（DB/SP/文件/网络）→ 必须切 IO 线程
   - JSON / Protobuf 解析 → 子线程，严禁在 onBindViewHolder 解析
   - 格式转换（日期、金额、表情解码）→ 提前缓存，不要 onBind 时计算
3. **RecyclerView 必做**：
   - 所有 Adapter 开 `setHasStableIds(true)`，`getItemId()` 返回稳定 id
   - `onCreateViewHolder` / `onBindViewHolder` 内无任何分配对象操作（new / substring / format）
   - 图片用 Glide 预加载（`preload` / `ListPreloader`），不要 onBind 时才发起
   - 聊天多类型 ItemProvider 布局相同的合并 ViewType，减少类型数量
4. **图片**：Glide 加载必须指定 `override(width, height)` 或 `centerCrop()`/`fitCenter()`，不要加载原图到列表
5. **避免过度绘制**：XML 根布局若父背景已覆盖，不重复写 `android:background`；聊天气泡用 Layer 合并

# MUST NOT

1. 不要在 `onBindViewHolder` 里 `new SimpleDateFormat`、`new Gson()`、`DateFormat.getDateInstance()` 等（这些初始化极慢）
2. 不要在 Activity/Fragment `onCreate/onResume` 串行执行大量初始化，走 AppInitManager 异步/懒加载
3. 不要为每个点击事件 `setOnClickListener` new 一个匿名对象（复用同一个 Listener 或用绑定扩展）
4. 不要在聊天列表滚动时 `notifyDataSetChanged()` 全量刷新，用 `notifyItemChanged` / DiffUtil

# SHOULD

1. **布局优化**：
   - 优先 ConstraintLayout 2.2.1，嵌套不超过 3~4 层
   - 用 `<include>` / `<merge>` / ViewStub 减少重复层级
   - 复杂聊天气泡考虑 `ConstraintLayout` + 百分比，不用多层 LinearLayout 嵌套
2. **对象池**：聊天消息解析频繁创建的对象（`SpannableStringBuilder`、表情解码容器）用 Pool 复用
3. **启动优化**：
   - `AppInitManager` 分阶段：`MAIN_THREAD`（必须首屏） / `BACKGROUND`（异步） / `LAZY`（首次使用）
   - `AndroidX SplashScreen` 库已接入（`libs.splashScreen`），SplashActivity 配合使用
4. **卡顿检测**：debug 构建可启用 Logan / xlog 打点，结合 Firebase Performance（如有）

# Decision

| 卡顿场景 | 优先排查 |
|---|---|
| 聊天列表滑动掉帧 | onBindViewHolder 耗时 / Glide 原图加载 / 表情解码未缓存 / ItemAnimator 过度动画 |
| 社群动态列表图片 | 图片尺寸过大（>1MB）/ 缺少占位/预加载 / GIF 解码阻塞 |
| 进入会话首帧慢 | 首屏消息 SQL 无索引 / 头像批量请求排队 / Room 主线程读 |
| 输入法弹起后跳动 | adjustNothing 配合聊天列表手动 scrollToPosition / 键盘高度监听错 |
| 红包/弹幕场景 | DanmakuFlameMaster 弹幕刷新频率 / 每帧对象分配过多 |
