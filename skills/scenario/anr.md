---
name: anr
description: ANR（应用无响应）分析知识（Auto：用户提到 ANR、无响应、弹窗卡死时自动触发）
activation: auto
triggers:
  - "ANR"
  - "无响应"
  - "卡死"
  - "应用不响应"
  - "trace"
---

# Purpose

ANR（Application Not Responding）定位：输入事件 5s / BroadcastReceiver 10s / Service 20s 超时，如何从 traces.txt 找到主线程阻塞点。

# MUST

1. **先读 traces.txt**：
   - 位置：`/data/anr/traces.txt`（或 Bugly / Firebase ANR 面板导出）
   - 看 main 线程堆栈顶部（"main" prio=5 tid=1），第一行阻塞函数就是元凶
2. **分类排查**：
   - **Binder 调用**：AIDL / ContentProvider 远端调用阻塞 → 改为异步或缓存
   - **SharedPreferences apply/commit**：`commit()` 写卡主线程 → 改用 MMKV
   - **主线程 I/O**：读 DB / 文件 / network 在 onCreate / onResume → 移到 IO 线程
   - **锁竞争**：`synchronized` / `Lock.lock()` 拿不到锁 → 缩小锁范围或用并发集合
   - **View 测量死循环**：自定义 View onMeasure 互相依赖 → 检查 measure 逻辑
3. **核对 targetSdk 35 特有 ANR 源**：
   - 前台 Service 未正确声明 `foregroundServiceType` → 系统不允许启动导致超时
   - `onBackPressed` 重写导致预测式返回手势死锁 → 改用 OnBackPressedDispatcher

# MUST NOT

1. 不要用 `SystemClock.sleep()` / `Thread.sleep()` 在主线程等异步结果（用 CountDownLatch 或 LiveData observe）
2. 不要在 `onReceive(Context, Intent)` 里执行网络/DB（BroadcastReceiver 仅 10s 超时）
3. 不要忽略 StrictMode：debug 构建启用 StrictMode 检测主线程 I/O 与未关闭资源

# SHOULD

1. **ANR-WatchDog（可选）**：若线上 ANR 捕获率不足，可引入 ANR-WatchDog 机制补采
2. **Firebase / Bugly 面板**：对比多个 ANR 堆栈 top 方法，找共性瓶颈
3. **启动 ANR**：ContentProvider 与 Application onCreate 初始化太重 → AppInitManager 异步/懒加载
4. **关键路径打点**：`init/*Init.kt` 每步打耗时日志（Logan / xlog），启动优化后验证

# Decision

| ANR 类型 | 典型堆栈关键词 | 处理 |
|---|---|---|
| SP 写卡住 | `at android.app.SharedPreferencesImpl.commit` | 全部替换 MMKV |
| 主线程 DB | `at android.database.sqlite.SQLiteQuery.nativeExecute` | Room 强制 withContext(Dispatchers.IO) |
| 锁阻塞 | `at java.lang.Thread.State: BLOCKED (on object monitor)` | 打印持锁线程堆栈，缩小 synchronized 范围 |
| Binder 卡住 | `at android.os.BinderProxy.transactNative` | 缓存远端结果 / 用 oneway 异步 |
| Service 启动超时 | `at android.app.ActivityManagerProxy.startService` | 前台服务先 startForeground + 缩短 onCreate |
