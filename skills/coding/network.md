---
name: network-coding
description: 网络层与 IM 长连接规范（FileMatch：**/net/**, **/socketlibrary/**, **/netlibrary/**）
activation: fileMatch
filePattern:
  - "**/net/**/*.kt"
  - "**/net/**/*.java"
  - "**/socketlibrary/**/*.kt"
  - "**/socketlibrary/**/*.java"
  - "**/netlibrary/**/*.kt"
  - "**/netlibrary/**/*.java"
---

# Purpose

HTTP（Retrofit/OkHttp）、IM 长连接（Netty/Socket）、推送（wdpushlibrary）三类通道的使用规范。

# MUST

1. **HTTP 统一出口**：所有 REST 请求走 `app/src/main/java/.../net/` 下的 RetrofitManager / ChatRepository / `*Api.kt`
   - 严禁在业务层自行 `OkHttpClient().newCall()`
2. **错误码统一处理**：自定义 `ApiException` + `ErrorCode`，UI 层不直接拿 HTTP code 判断
3. **WebSocket / IM 消息**：通过 `SocketService` / `SyncSocketService` / `SendMsgService2` 发送接收
   - 收到消息用 EventBus / MessageEvent 分发到上层
4. **序列化**：Protobuf 统一走 `javalite`（`protobuf-javalite` 依赖），聊天协议在 `app/src/main/proto/imchat.proto`
5. **上传文件**：统一走 toslibrary（火山引擎 TOS）或阿里云 OSS 封装，不直传业务服务器
6. **域名配置**：业务域名从 `configlibrary` 或 `assets/http_wedo_config.json` 读取，不硬编码在代码里

# MUST NOT

1. 不要在主线程发起同步网络请求（Retrofit `execute()` 等）
2. 不要忽略失败回调，至少 Toast / Error 状态提示用户
3. 不要把 Socket 的 Netty `Channel` / `ByteBuf` 直接暴露给 UI / Presenter 层
4. 不要为单次请求单独创建 OkHttpClient 实例（复用连接池）

# SHOULD

1. **HTTPS 优先**：`usesCleartextTraffic` 在 official 包为 `false`，beta 包允许 `true` 用于内网调试
2. **日志**：release 包关闭 HTTP 日志拦截器，debug 包使用 OkHttp LoggingInterceptor（BASIC/HEADERS）
3. **重试策略**：请求失败走项目已有重试机制（指数退避），不要死循环 while 重试
4. **证书校验**：Release 包严禁关闭 SSL 校验；beta 调试抓包仅在 debug 构建中临时放开
5. **超时配置**：连接/读/写超时使用 netlibrary 默认值，特殊场景单独指定并注释原因

# Decision

| 场景 | 选择 |
|---|---|
| 登录/用户/配置类 HTTP | netlibrary → Retrofit Api |
| 聊天消息（文本/图片/文件） | SocketService（IM 长连接） |
| 信令/音视频协商 | Socket + WebRTC 数据通道 |
| 推送消息到达 | wdpushlibrary → 广播 / EventBus |
| 图片/视频/文件上传 | toslibrary（TOS） + 回调业务服务器 |
| SSE 流式订阅（如 AI 回复） | OkHttpSseManager（EventSource） |
