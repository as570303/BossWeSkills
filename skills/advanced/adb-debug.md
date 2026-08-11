---
name: adb-debug
description: ADB 调试命令速查与本项目专属用法（Manual：用户要求"用 adb 调试"、"装包"、"看日志"时引用）
activation: manual
---

# Purpose

常用 adb / gradle 命令 + 本项目特有调试技巧（安装包、清数据、抓日志、启动指定页、权限授予等）。

# MUST

1. **构建与安装**：
   ```bash
   # 构建 beta debug 并安装到已连接设备
   ./gradlew assembleBetaDebug && adb install -r app/build/outputs/apk/beta/debug/app-beta-debug.apk

   # 快速重装 + 清数据（等价卸载重装，debug 调试完清状态）
   adb uninstall com.bosswe.bw.debug && adb install -r app-beta-debug.apk

   # 给所有 flavor 包同时授权运行时权限（省得手点）
   adb shell pm grant com.bosswe.bw.debug android.permission.CAMERA
   adb shell pm grant com.bosswe.bw.debug android.permission.RECORD_AUDIO
   adb shell pm grant com.bosswe.bw.debug android.permission.POST_NOTIFICATIONS
   ```
2. **启动指定 Activity（跳过登录直测页面，提效神器）**：
   ```bash
   # 聊天页（需要传 chatId 等 extra，按实际补）
   adb shell am start -n com.bosswe.bw.debug/com.sharexm.xm.ui.activity.ChatActivity --ei chat_id 12345

   # 启动 Splash 后清栈
   adb shell am start -S -n com.bosswe.bw.debug/com.sharexm.xm.ui.activity.SplashActivity
   ```
3. **日志抓取**（重点！本项目 release 日志是加密的，debug 可读）：
   ```bash
   # 实时日志 + 写入文件，tag 过滤（XM_ 开头）
   adb logcat -v time | grep -E "XM_|AndroidRuntime|System.err|Fatal" > app.log

   # 读取加密日志文件（release 用户反馈日志）
   # 先 pull：adb pull /sdcard/Android/data/com.bosswe.bw/files/log/ ./logs/
   # 再解密：
   python log_decrypt/decrypt_log_file.py ./logs/xm_20250811.log
   ```
4. **ANR traces**：
   ```bash
   adb pull /data/anr/traces.txt ./anr_traces.txt
   # 无 root 权限时用 bugreport
   adb bugreport ./bugreport.zip
   unzip ./bugreport.zip && grep -A 40 "\"main\"" bugreport-*/FS/data/anr/traces.txt
   ```
5. **GoRouter 深链路由测试**（项目已配置 scheme）：
   ```bash
   adb shell am start -a android.intent.action.VIEW -d "debugxm://chat?chat_id=123"
   ```

# MUST NOT

1. 不要在 release 包用 `am start` 强行起 `exported=false` 的 Activity（Manifest 已 exported=false 起了会报 SecurityException）
2. 不要用 `root` 后的设备乱删 `/data/data/com.bosswe.bw/databases/`（MMKV 文件损坏后整个 KV 丢失）
3. 不要把 `adb shell input tap` 自动化脚本写进 CI（屏幕分辨率/ROM 差异大，不可靠）

# SHOULD

1. **adb 多设备选择**：同时插多台机/模拟器时用 `adb -s <serial>` 指定设备（`adb devices` 查看序列号）
2. **录屏（演示 Bug）**：
   ```bash
   adb shell screenrecord /sdcard/bug.mp4 && adb pull /sdcard/bug.mp4 ./bug.mp4
   ```
3. **截图**：
   ```bash
   adb exec-out screencap -p > bug.png
   ```
4. **LeakCanary 泄露堆转储导出**：debug 包检测到泄露，在通知栏点 "Dump heap" → `adb pull /sdcard/Download/leakcanary-*.hprof ./`

# Decision

| 场景 | 推荐命令组合 |
|---|---|
| 开发迭代快速装包 | `./gradlew assembleBetaDebug && adb install -r -t app/build/outputs/apk/beta/debug/app-beta-debug.apk` |
| QA 反馈崩溃日志 | pull 加密日志 → python decrypt_log_file.py → 搜 Fatal / Caused by |
| 深链跳转 GoRouter 测试 | adb shell am -V -d "debugxm://..." |
| 性能测启动时间 | `adb shell am start -S -W com.bosswe.bw.debug/.ui.activity.SplashActivity` 看 WaitTime |
| 输入法/键盘冲突问题 | `adb shell ime list -s` / `adb shell settings put secure show_ime_with_hard_keyboard 0` |
