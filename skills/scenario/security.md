---
name: security
description: 安全合规与隐私规范（Auto：用户提到安全、隐私、合规、加密、权限时自动触发）
activation: auto
triggers:
  - "安全"
  - "隐私"
  - "加密"
  - "权限"
  - "合规"
  - "permission"
---

# Purpose

涉及用户隐私、密钥管理、Manifest 权限、数据加密等安全相关代码修改时的红线。

# MUST

1. **密钥绝不硬编码**：
   - 核心密钥放 `nativekeys` 模块（CMake + JNI 读取 libxm_keys.so），通过 `NativeAppKeys.kt` 获取
   - 三方 SDK Key（Google Maps / Firebase / 微信）：
     - Google Maps：`secrets-gradle-plugin` 从 `local.properties` 读取 → Manifest `MAPS_API_KEY`
     - 微信 AppID：configlibrary 配置 + buildConfig
2. **HTTPS / 网络安全**：
   - official release 包 `usesCleartextTraffic=false`，严禁明文 HTTP
   - `network_security_config.xml` 中 debug 仅放开测试环境域名白名单，production 只允许系统 CA
3. **Manifest 权限最小化**：
   - 新增权限必须说明用途（为什么用 / 和哪个功能模块对应），Comment 加在 Manifest 旁
   - 已存在 REQUEST_INSTALL_PACKAGES 用 gradle 脚本在打包时移除（见 app/build.gradle variant.outputs 清理逻辑）
4. **本地数据加密**：
   - 登录 Token / Session：MMKV 默认加密 + Android Keystore 绑定（如已接入）
   - 支付密码 / 助记词：绝不明文存 SP/DB，走 AndroidX Security Crypto 或 native 加密
5. **WebView 安全**：
   - `setAllowFileAccess(false)`、`setAllowContentAccess(false)`
   - `WebView.EnableSafeBrowsing=false`（项目当前已关闭 meta-data，若后续打开记得更新）
   - JSBridge 接口做白名单校验，不开放 eval 任意代码执行

# MUST NOT

1. 不要把 `sharexm.jks` 密钥密码（sharexm!123456）提交到公开仓库（本项目 config.gradle 中已内置，仅限内测 CI 环境）
2. 不要在日志打印身份证号 / 手机号 / 支付密码 / Token 完整值（至少脱敏中间 4 位 `138****1234`）
3. 不要使用全局自签名 `TrustManager`（"accept all SSL"）——会绕过证书校验
4. 不要给 beta 包随意放开权限（`WRITE_SETTINGS`、`REQUEST_INSTALL_PACKAGES` 等），上线后被扫描到会驳回

# SHOULD

1. **隐私合规 Checklist**（上架商店前）：
   - 首次启动弹《隐私协议》Dialog，未同意不初始化网络/推送 SDK（当前项目已有 PrivacyActivity）
   - 权限按需申请：定位/相机/存储/通知在实际使用场景弹出，不启动时一次性全要
2. **ProGuard / R8**：`proguard-rules.pro` + 模块 `consumer-rules.pro` 保留 keep 规则避免反射/序列化崩溃
3. **Apk 签名**：release 用 V2+V3 签名（AGP 8.x 默认启用），不要仅 V1
4. **依赖安全**：`libs.versions.toml` 中三方 SDK 定期升小版本修 CVE，OkHttp/Retrofit/Glide/Firebase 保持活跃维护版本

# Decision

| 敏感资产 | 存放位置 |
|---|---|
| App 签名密钥 (jks) | 仅 Jenkins CI，本地开发者不用 sharexm.jks（除非本地打 release 测试） |
| 业务核心加密密钥 | nativekeys → libxm_keys.so |
| HTTP 域名配置（多环境切换） | assets/http_wedo_config.json + configlibrary |
| Google Maps / Firebase API Key | secrets-gradle-plugin + google-services.json |
| 微信/支付/推送 SDK Key | configlibrary + flavor buildConfigField |
| 登录态 / 用户设置 | MMKV (加密) |
| 聊天历史 DB | service/db（DBManager 加密可选接入 SQLCipher） |
