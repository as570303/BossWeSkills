---
name: gitlab-ci
description: GitLab CI / Jenkins 构建流水线知识（Manual：仅当用户要求"配 CI"、"改流水线"、"Jenkins"时引用）
activation: manual
---

# Purpose

指导 `.gitlab-ci.yml` / Jenkins Gradle 构建脚本的配置、调优、常见问题排查。

# MUST

1. **CI 根配置**：项目根已存在 `.gitlab-ci.yml`（GitLab 管道）和 `jenkins_upload.gradle`（Jenkins 集成上传）
2. **CI 环境变量**（必须在 GitLab 项目 Settings → CI/CD Variables 配置，禁止写死在 yml）：
   - `ANDROID_HOME` / `ANDROID_SDK_ROOT`：Android SDK 根路径
   - `JAVA_HOME`：JDK 17 路径
   - `KEYSTORE_PASSWORD` / `KEY_PASSWORD` / `KEY_ALIAS`：签名凭证
   - 内部 Nexus 账号：`NEXUS_USER` / `NEXUS_PWD`（build.gradle 中已有 xm-deploy 等凭证占位）
3. **构建缓存**（必须配置，否则每次 20+ min）：
   - Gradle User Home：`~/.gradle/caches` → cache key = `checksum["gradle/libs.versions.toml", "build.gradle"]`
   - Android SDK：持久化不清理（`$ANDROID_HOME/platforms;build-tools;ndk`）
4. **任务阶段建议**：
   - stage 1 `assemble`：`assembleOfficialRelease` + `assembleBetaRelease`（产出 APK/AAB）
   - stage 2 `test`：`testOfficialReleaseUnitTest`（本地单测，无 Android device 依赖）
   - stage 3 `lint`：`lintVitalOfficialRelease`（Google Play 必过 lint）
   - stage 4 `upload`：调用 `jenkins_upload.gradle` / 自建脚本传分发平台
5. **Flavor 矩阵组合**：`official` / `beta` × `debug` / `release` 一般只跑 betaDebug + officialRelease 两档节省时间

# MUST NOT

1. 不要把密钥密码（Nexus / signingConfig）直接写进 `.gitlab-ci.yml`、`config.gradle`
2. 不要在 CI 并行跑多个 Gradle daemon（`--no-daemon` 避免 CI 机器内存爆）
3. 不要让 `clean` 任务每次自动执行（除非构建机器磁盘紧张；clean 会清空 build/，拖慢）
4. 不要在一个 job 里同时做 assemble + test + lint（拆 job 并行加速，失败能重试对应阶段）

# SHOULD

1. **Gradle 参数 CI 友好**：
   ```bash
   ./gradlew assembleOfficialRelease \
     --no-daemon \
     --stacktrace \
     -PIS_JENKINS=true \
     -PVERSION_CODE=${CI_COMMIT_SHORT_SHA:0:8} \
     -PCOPY_APK=true
   ```
2. **产物 Artifacts 保留**：
   - `app/outputApk/*.apk` 保留 30 天
   - `app/build/outputs/mapping/**/mapping.txt` 保留 180 天（Crashlytics 还原堆栈用）
3. **通知失败**：失败 job 推飞书/钉钉群 @相关开发，带上 job url + log tail 50 行
4. **包体积监控**：CI 记录每次 APK/AAB 大小，超阈值（如 APK 增加 > 2MB）告警并附 top N 大文件清单

# Decision

| CI 常见问题 | 排查思路 |
|---|---|
| R8 报错 `Missing class`（release only） | 检查 consumer-rules.pro / proguard-rules.pro 是否缺少反射/序列化 keep |
| NDK 找不到 `ndkVersion` | CI SDK Manager 装对应 NDK：`sdkmanager "ndk;27.2.12479018"` |
| 依赖下载失败（Nexus 401） | 检查 CI 环境变量 NEXUS_USER / NEXUS_PWD 是否正确 + 没有特殊字符 bash 转义 |
| build 时间 40min+ | 开 Gradle remote cache + `org.gradle.parallel=true` / `org.gradle.caching=true`（gradle.properties） |
| lintVital 失败（Google Play 上架阻塞） | 先本地跑 `lintVitalOfficialRelease`，修 lint-results 报告里的 fatal Issue |
