---
name: release
description: 应用发布与打渠道包流程（Manual：仅当用户明确要求"发布"、"打渠道包"、"提审"时引用）
activation: manual
---

# Purpose

指导从 assemble 构建 → 渠道打包 → 签名校验 → 上传 → 提审的完整发布流程。

# MUST

1. **发布前 Checklist**：
   - `config.gradle` 中 `versionCode` / `versionName` 已递增（official 与 beta 逻辑见 app/build.gradle）
   - `debug_versionCode` / `debug_versionName` 按规则更新
   - Jenkins 参数 `VERSION_CODE` / `VERSION_NAME` 与本地一致（若走 CI 构建）
2. **构建命令**（本地调试时）：
   ```
   # 测试包
   ./gradlew assembleBetaRelease
   # 正式包
   ./gradlew assembleOfficialRelease
   # 正式 aab（Google Play）
   ./gradlew bundleOfficialRelease
   ```
3. **签名配置**：
   - `signingConfigs.release` 使用 `./sharexm.jks`
   - 密码、别名、密钥密码在 `config.gradle` signingconfigs 中（密码 = sharexm!123456）
   - release 构建需 V2+V3 签名（AGP 8.x 默认）
   - 注意：Official AAB 构建时 `useLegacyPackaging = false`（so 不压缩），以满足 Google Play 16KB 对齐
4. **渠道打包**：使用 `pack/VasDolly.jar` 多渠道
   - 渠道列表：`pack/channel.txt`（一行一渠道，如 `googleplay`、`huawei`、`xiaomi`）
   - 脚本：`pack/channel.sh` 或 `java -jar pack/VasDolly.jar put -c channel.txt base.apk output_dir/`
5. **16KB 对齐校验（官方包必做）**：
   ```bash
   # 检查所有 so 的 LOAD 段 Align
   for so in app/src/main/jniLibs/*/*.so; do
     echo "=== $so ==="; readelf -l "$so" | grep LOAD
   done
   ```
   要求每行 Align ≥ `0x4000`，否则上架 Google Play 失败。
6. **Jenkins CI 构建**：
   - Gradle 参数：`IS_JENKINS=true`
   - 输出：`xm_<build_type>_<version>_<yyyymmddHHmm>_release.apk`（见 app/build.gradle variant 自定义输出）
   - 上传逻辑：`jenkins_upload.gradle` 已配置对应 upload 目标

# MUST NOT

1. 不要在本地开发者机器 sharexm.jks 做 release 构建（除非本地调试；正式包走 Jenkins）
2. 不要手动修改 build 目录下任何产物（渠道号、Manifest 等），所有修改走 build.gradle / 打包脚本
3. 不要跳过 `assembleOfficialRelease` 直接提 AAB（release + minify + shrinkResources 能暴露 beta 没遇到的 R8/资源移除崩溃）
4. 不要忘了 `COPY_APK=true` 参数把 APK 复制到 `app/outputApk/` 方便分发

# SHOULD

1. **构建前 clean**：`./gradlew clean` 避免增量残留
2. **R8 mapping 保留**：Firebase Crashlytics 默认 mapping 不上传（`mappingFileUploadEnabled false`），发版前可临时开启或手动上传 mapping.txt
3. **回归冒烟清单**（渠道包发布前必过）：
   - 冷启动 → 登录（手机号+密码、助记词）
   - 单聊（文本/表情/图片/语音/视频/文件/红包/转账/位置）
   - 群聊/社群消息
   - 音视频一对一、群会议
   - 钱包：余额页、充币、提币、转账（至少走到校验密码页）
   - 推送收消息 / 前后台切换 / 账号互踢
4. **版本管理脚本**：`versionManage.sh` 可辅助修改版本号

# Decision

| 发布目标 | 产物类型 | 打包命令 |
|---|---|---|
| QA 内测包 | APK (beta) | assembleBetaRelease → 打渠道包 |
| 客户/运营测试包 | APK (official beta 风格 versionName) | assembleOfficialRelease + flavor 区分 |
| 应用商店（国内） | APK (official) + 多渠道 | assembleOfficialRelease → VasDolly |
| Google Play | AAB (official) | bundleOfficialRelease → Play Console |
| 热更新补丁 | Tinker patch | tinkerPatchOfficialRelease（tinker-support.gradle 已注释，按需启用） |
