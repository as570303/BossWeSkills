---
name: database-coding
description: 数据库与本地存储规范（FileMatch：**/service/db/**, **/modellibrary/**, **/provider/**）
activation: fileMatch
filePattern:
  - "**/service/db/**/*.kt"
  - "**/service/db/**/*.java"
  - "**/modellibrary/**/*.kt"
  - "**/modellibrary/**/*.java"
  - "**/provider/**/*.kt"
  - "**/provider/**/*.java"
  - "**/dao/**/*.kt"
  - "**/dao/**/*.java"
---

# Purpose

Room、SQLite、MMKV、ContentProvider 等本地存储使用规范。

# MUST

1. **KV 存储优先 MMKV**：轻量配置、登录态、用户偏好使用 MMKV（`mmkv-static` 1.2.10），仅老代码维持 SP
2. **结构化数据走 Room**：Entity / DAO / Database 放在 modellibrary，编译期校验 SQL
3. **大批量聊天记录**：通过 `service/db/` 下的 `ChatManager` / `DBManager` 统一访问，不要直接开 SQLiteDatabase
4. **DB 升级**：Room `Migration` 必须覆盖每个版本的增量迁移，严禁 `fallbackToDestructiveMigration`
5. **ContentProvider**：`${applicationId}.provider.domainprovider.read` 权限已在 Manifest 声明，跨进程读域名按此权限

# MUST NOT

1. 不要在主线程执行 Room/SQLite 写操作（必须 IO 协程调度或 Rx IO 线程）
2. 不要存明文密码、Token、密钥到 SharedPreferences / 数据库（加密或走 native）
3. 不要在 DB 里缓存大量网络请求结果（过期不清理容易 OOM/膨胀）
4. 不要把数据库对象（Cursor、SQLiteDatabase）传到 Activity/Fragment UI 层

# SHOULD

1. **DB 版本管理**：sysdbmanager.dbVersion 在 config.gradle 集中维护，升级版本号 + 写 Migration
2. **分页加载**：聊天历史、长列表用 Room `PagingSource` 或 `limit/offset` 分页，不取全表
3. **读写分离**：频繁读的查询加索引（`@Index`），批量写用 `@Transaction`
4. **备份与加密**：敏感表考虑 SQLCipher（当前项目未接入，需讨论），备份走 WorkManager 调度

# Decision

| 场景 | 选择 |
|---|---|
| 登录态/开关/简单配置 | MMKV（add / encode / decode） |
| 用户/好友/群组结构化列表 | Room Entity + DAO |
| 聊天消息记录（海量） | DBManager + ChatManager 封装 |
| 内置省市区/emoji 等只读数据 | resourceslibrary assets → JSON/SQLite 首次导入 |
| 跨进程数据共享（域名配置） | ContentProvider + 读权限声明 |
