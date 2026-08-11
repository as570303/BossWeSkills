---
name: crash-analysis
description: Crash / 崩溃分析定位知识（Auto：用户提到崩溃、Fatal Exception 时自动触发）
activation: auto
triggers:
  - "崩溃"
  - "crash"
  - "fatal exception"
  - "闪退"
  - "ANR"
---

# Purpose

当用户提供 Crash 堆栈或描述闪退问题时，指导 AI 系统化地定位与修复本项目中的崩溃。

# MUST

1. **先看堆栈**：确认崩溃类型（NPE / OOM / IllegalStateException / UnsatisfiedLinkError / SecurityException）
2. **核对工程红线**：
   - 若为 `CompoundButton?` / `RadioGroup?` 类型不匹配 → Android 15 @NonNull 监听器签名问题，按 core project-rules §MUST NOT.2 修复
   - 若为 `UnsatisfiedLinkError` / `.so` load 失败 → 检查 16KB 页对齐（readelf -l 看 LOAD Align ≥ 0x4000）
   - 若为 `SecurityException`（WiFi/蓝牙/定位）→ 检查 targetSdk 35 权限声明与运行时请求
   - 若为 `SuperNotCalledException` / Fragment 状态丢失 → 检查 `onBackPressed` 是否被错误重写（必须 OnBackPressedDispatcher）
3. **确认 Flavor**：official（release, minify）还是 beta（debug, no-minify），混淆后的堆栈需用 mapping 还原
4. **日志工具**：本项目日志加密存储，使用 `log_decrypt/` 目录下工具解密：
   - Python：`log_decrypt/decrypt_log_file.py` / `decrypt_more_type.py`
   - Jar：`log_decrypt/decrypt.jar` / `decrypt_new.jar`

# MUST NOT

1. 不要只看首行堆栈，要追踪 Cause 链（Caused by: 通常是真正原因）
2. 不要忽略设备/系统版本信息（Android 15+ 特有崩溃占比高）
3. 不要在主线程加耗时操作「临时规避」，要根治原因

# SHOULD

1. **Firebase Crashlytics**：release 包崩溃优先查 Firebase（Crashlytics Gradle 插件已接入 3.0.6）
2. **Bugly**：同时接入腾讯 Bugly 4.1.9，可做二次交叉确认
3. **LeakCanary 2.14**：debug 构建已启用，内存相关崩溃先看 leakcanary 泄露日志
4. **复现路径**：要求用户/QA 提供复现步骤、Android 版本、机型、Flavor（official/beta）

# Decision

| 崩溃类型 | 排查优先级 |
|---|---|
| NPE (KotlinNullPointerException) | 数据层 null 未判空 / ViewBinding 视图生命周期错配 |
| OutOfMemoryError | Glide 大图 / Activity 泄露 / RecyclerView 未回收 / 长连接消息缓存膨胀 |
| UnsatisfiedLinkError (libxm_keys.so / libndkbitmap.so) | ABI 缺失 / 16KB 对齐失败 / so 压缩策略 |
| SecurityException (NEARBY_WIFI_DEVICES 等) | Manifest 权限缺失 / 运行时未动态申请 / targetSdk 升级 |
| NetworkOnMainThreadException | HTTP / Socket 被放到主线程（lint 已禁止但仍可写错） |
| Resources$NotFoundException | flavor 资源缺失 / 多语言 key 缺译 / drawable 仅在一个 density |

# Examples

**例 1：CompoundButton 监听器报错（API 35）**
```
Caused by: java.lang.IncompatibleClassChangeError:
  abstract method 'void android.widget.CompoundButton$OnCheckedChangeListener.onCheckedChanged(android.widget.CompoundButton, boolean)'
→ 修复：把第一个参数从 CompoundButton? 改为 CompoundButton（非空）
```

**例 2：.so 加载失败**
```
java.lang.UnsatisfiedLinkError: dlopen failed: .../libxm_keys.so has load segment NOT page-aligned
→ 修复：readelf -l libxm_keys.so 检查 LOAD Align，重编 so 使 Align=0x4000
```
