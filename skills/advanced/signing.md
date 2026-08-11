---
name: signing
description: APK/AAB 签名与多包体校验（Manual：用户提到"签名"、"校验签名"、"v2/v3 签名"时引用）
activation: manual
---

# Purpose

签名配置（V1/V2/V3/SourceStamp）、keystore 管理、签名校验、应用升级覆盖安装失败排查。

# MUST

1. **当前签名配置**（app/build.gradle signingConfigs）：
   ```
   storeFile     : ./sharexm.jks
   storePassword : sharexm!123456
   keyAlias      : sharexm
   keyPassword   : sharexm!123456
   ```
   - 证书指纹（Manifest 注释中已标注）：
     - MD5:     `34:B4:A0:1C:6E:2F:72:8C:0B:13:D3:F4:93:FF:3D:9D`
     - SHA1:    `B3:65:3F:93:EF:3D:CD:A5:B9:F1:6D:37:0E:9C:69:01:18:CD:19:54`
     - SHA-256: `53:06:24:E2:FD:E3:3C:9A:A8:4E:AB:26:03:35:F0:62:18:3E:A7:4E:41:2F:49:C0:0E:80:13:77:78:72:23:57`
   - 注意：此 keystore 仅内部 beta/official 构建共用；如果后续 Google Play 上架走 App Signing，Play 会重新生成分发证书，指纹会变。
2. **签名校验（发版前必跑）**：
   ```bash
   # 方法 1：Android SDK 自带 apksigner
   apksigner verify -v --print-certs app/build/outputs/apk/official/release/*.apk
   # 方法 2：Java jarsigner（仅 V1 准确，V2+ 建议 apksigner）
   jarsigner -verify -verbose -certs app.apk
   ```
   预期输出 `Verified using v1 scheme (JAR signing): true`、`v2 scheme (APK Signature Scheme v2): true`、`v3 scheme (APK Signature Scheme v3): true`。
3. **签名不兼容升级失败排查**（用户反馈"无法覆盖安装"）：
   - 取出新旧两个 APK 的签名 CERT.RSA 比对：
     ```bash
     unzip -p old.apk META-INF/CERT.RSA | keytool -printcert
     unzip -p new.apk META-INF/CERT.RSA | keytool -printcert
     ```
     两个 SHA-256 必须完全一致，否则就是签名证书变了，Android 系统禁止覆盖安装（只能卸载重装）。
4. **多渠道 VasDolly 不破坏签名**：VasDolly 是写入 APK Signing Block，不重签，不丢失 V2/V3 签名——但必须保证「父包」本身已完整 V2 签名再打渠道。

# MUST NOT

1. 不要把 `sharexm.jks` 上传到公开 Git 仓库（本项目历史仓库已内置仅限内网；若需要迁移到外部，务必删除并替换新 keystore）
2. 不要丢失 keyPassword：一旦丢失，**没有任何办法恢复**，意味着不能覆盖更新，只能换包名重新上架
3. 不要用 debug keystore (`~/.android/debug.keystore`) 打 release 包（所有应用商店都拒绝 "Android Debug" 签名证书）
4. 不要在不同 CI 机器用不同 keystore（统一用一个 sharexm.jks，机器通过 `$KEYSTORE_FILE` 环境变量挂载）

# SHOULD

1. **定期轮换**：建议 1~2 年生成新 key（但要保留旧 key，存量用户升级要兼容）
2. **Google Play App Signing 迁移**：若走 Play Console → Setup → App signing，导出 `pepk.jar` 加密上传现有证书，Play 会给你「上传证书」+「分发证书」两套指纹，`google-services.json` 中的 SHA-1 要换分发证书指纹。
3. **密钥保存在独立 KeyPass / 1Password**，不要靠口头传递邮件明文密码
4. **OpenSDK / 微信 / 地图配置**：分享、登录、支付 SDK（微信开放平台、Google Maps、华为 HMS、支付宝）要求在开放平台配置包名 + SHA-1，每次升级签名都要同步更新。

# Decision

| 问题 | 处理 |
|---|---|
| 安装失败 "INSTALL_FAILED_UPDATE_INCOMPATIBLE" | 新老 APK 签名不一致 → 用上面 CERT.RSA 比对确认；若 beta 与 official 用不同包名属正常 |
| 微信分享失败 "invalid signature" | 微信开放平台后台配置的 SHA-1 与当前 APK 指纹不一致 → 更新后台 / 换回正确签名 |
| apksigner verify 报 "Missing META-INF/XXX.SF" | 签名只有 V1 → 启用 v2SigningEnabled true（AGP 8 默认开启，确认没在 build.gradle 强制关） |
| Google Play 拒绝 "APK 未签名" 或 "不安全加密" | 必须同时 V2/V3；AAB 用 jarsigner 签名（Studio/AGP bundle 任务会自动做） |
| 需要 V3 轮转支持（Android 9+） | AGP 8 默认支持；检查 `signingConfig.enableV3Signing = true`，保留旧证书能跨版本升级 |
