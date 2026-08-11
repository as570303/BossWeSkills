---
name: memory
description: 内存优化与 OOM 处理（Auto：用户提到 OOM、内存泄露、内存溢出时自动触发）
activation: auto
triggers:
  - "OOM"
  - "内存泄露"
  - "内存溢出"
  - "OutOfMemoryError"
  - "leak"
---

# Purpose

Out Of Memory（OOM）、Activity/Fragment/ViewModel 泄露、长连接缓存膨胀等内存问题排查与修复。

# MUST

1. **先确认泄露路径**：LeakCanary 2.14 在 debug 构建已启用，先看 LeakCanary 生成的泄露链（Leak trace）
2. **常见泄露源**（本项目高频）：
   - `EventBus.register()` 未解注册 → 必须在对应生命周期 `onStop/onDestroy` 中 `unregister()`
   - `GoRouter` 回调持有 Activity/Fragment 引用 → 改用 ViewModel 监听或生命周期绑定
   - `SocketService` / `BroadcastReceiver` 注册未解绑
   - 静态单例（Manager/Util）持有 Context → 改为 `Application` Context 或 `WeakReference`
   - 内部类 Handler / Runnable 隐式持有 Activity 引用 → `WeakReference` + `removeCallbacksAndMessages(null)`
3. **Bitmap 内存**：
   - Glide 加载列表图片必须 `override()` 缩略图尺寸，禁止全尺寸解码
   - 大图预览（SubsamplingScaleImageView 3.10.0）按需分块加载，不整张进内存
   - FaceUnity / GPUImage 滤镜使用完释放 GL 纹理
4. **缓存上限**：聊天消息 / 头像 LruCache 必须设置 `maxSize`（建议按 `Runtime.maxMemory()` 比例），严禁无限 HashMap

# MUST NOT

1. 不要用 `static Context / static Activity / static View` 存任何 UI 引用
2. 不要在 Activity 被销毁后仍继续执行 Retrofit / Coroutine 请求（取消订阅 / viewModelScope 自动取消）
3. 不要在 Application 里缓存大型 byte[] / Bitmap，LruCache 控制上限
4. 不要忽略 `onTrimMemory(level)` 回调，需在 TRIM_MEMORY_BACKGROUND 时清空 Glide/自定义缓存

# SHOULD

1. **Profiler + MAT 结合**：Profiler 抓 .hprof → MAT 查 Histogram / Dominator Tree → 定位持有链
2. **Native 内存**：
   - WebRTC / LiveKit / FFmpeg 释放 codec / PeerConnection 资源（`dispose()`）
   - DanmakuFlameMaster（libndkbitmap.so）停止渲染后释放 native Bitmap
3. **WebView**：`WebView.destroy()` + 从父 ViewGroup `removeView()`，多进程方案避免主进程内存膨胀
4. **MMKV 缓存**：不要把大 JSON 存到 MMKV（> 10KB 走文件或 Room）

# Decision

| 内存问题类型 | 典型场景 | 处理方式 |
|---|---|---|
| Activity 泄露 | EventBus 未解注册 / Handler 延迟消息 | LeakCanary 看 GC Root → unregister / removeCallbacks |
| Fragment 泄露 | ViewPager/Adapter 保存旧 Fragment 引用 | Adapter.Factory + viewLifecycleOwner |
| ViewModel 泄露 | VM 持有 View/Activity 引用 | 把 UI 逻辑移回 Fragment/Activity，VM 只发状态 |
| OOM (pitch=2080KB 大图) | 聊天发送原图进内存 | Glide downsample + 限制图片尺寸 |
| 长连接缓存无限增长 | Socket 收到消息不清理 oldest | 环形队列 + 滑窗，超过上限存 DB 不存内存 |
| WebRTC 摄像头内存不释放 | 通话结束未 dispose PeerConnection | onDestroy 统一释放 audio/video track + peer |
