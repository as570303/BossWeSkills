---
name: test-coding
description: 单元测试与 UI 测试规范（FileMatch：**/test/**, **/androidTest/**）
activation: fileMatch
filePattern:
  - "**/src/test/**/*.kt"
  - "**/src/test/**/*.java"
  - "**/src/androidTest/**/*.kt"
  - "**/src/androidTest/**/*.java"
---

# Purpose

测试编写规范，确保新增代码的可测性与回归效率。

# MUST

1. **测试与源码同包**：`src/test/java/<同包路径>` 以便访问 package-private 成员
2. **JUnit 4 依赖统一**：`testImplementation libs.junit`（4.13.2），Kotlin 测试用 `kotlin-test-junit`
3. **本地单元测试不依赖 Context**：必须 Context 的逻辑抽到 androidTest，或用 Robolectric
4. **命名**：测试类 `被测类名Test`；方法 `行为_场景_期望结果`（如 `sendMessage_emptyText_returnsError`）

# MUST NOT

1. 不要写依赖网络实时返回结果的测试（Mock WebServer 或 Mock Repository）
2. 不要在测试里 `Thread.sleep()` 等固定时间，用 CountDownLatch / 超时机制
3. 不要让测试用例互相依赖（按方法独立可重复执行）

# SHOULD

1. **Repository 层**：优先写单元测试，Mock Api/DAO
2. **ViewModel**：用 `InstantTaskExecutorRule` + Mock Repository 验证状态流
3. **复杂工具类**（如加密、消息解析、分页计算）：100% 覆盖边界用例
4. **数据模型 Protobuf**：序列化/反序列化 round-trip 测试

# Decision

| 场景 | 测试层级 |
|---|---|
| 工具方法、纯函数、扩展函数 | Unit Test（test/） |
| ViewModel 状态机、UseCase 业务 | Unit Test（Robolectric 可选） |
| Room DAO / Migration | Instrumented Test（androidTest/） |
| Activity/Fragment 端到端流程 | 暂不强制（当前项目无 Espresso 依赖，按需引入） |
