---
name: kotlin-coding
description: Kotlin 代码规范（FileMatch：**.kt, **.kts）
activation: fileMatch
filePattern:
  - "**/*.kt"
  - "**/*.kts"
---

# Purpose

Kotlin 代码编写规范，适用于所有新增/修改的 .kt / .kts 文件。

# MUST

1. **不可变优先**：变量默认 `val`，仅必须重新赋值时用 `var`
2. **空安全**：可空类型显式标注 `?`，使用前必须判空，严禁滥用 `!!`
   - 优先 `?.` / `?:` / `let` 空安全调用
   - 仅在 100% 确定非空（如刚通过 if 判空）时才用 `!!`
3. **类后缀**：Activity/Fragment/Dialog/Adapter/ViewModel 等必须加固定后缀（见 core project-rules）
4. **小驼峰统一**：变量/方法/字段不加 `m`/`s` 匈牙利前缀
5. **协程规范**：
   - `suspend` 函数不加 `suspend` 后缀，按业务语义命名
   - 主线程调用用 `viewModelScope.launch` / `lifecycleScope.launch`
   - 禁止 `GlobalScope.launch`
6. **函数默认参数**：优先用默认参数代替 Builder 模式 / 重载
7. **顶层函数**：扩展函数/工具方法放 `xxxExtensions.kt` / `xxxUtil.kt`，不散落在业务类里
8. **when 表达式**：枚举/密封类分支必须穷举，或加 `else` 兜底

# MUST NOT

1. 不要在 `.kt` 新代码里混用 Java getter/setter 风格（直接用属性）
2. 不要用 `companion object` 存大量可变静态状态（用 Manager/单例 object）
3. 不要用 `lateinit` 修饰可空类型（直接用 `? = null`）
4. 不要写巨型函数，单函数超过 80 行考虑拆分
5. 不要在一个文件里放多个无关联的公开类

# SHOULD

1. 优先使用 `apply` / `run` / `with` / `also` / `let` 标准函数提升可读性
2. 实体数据类用 `data class`，并定义在 `bean/` 对应包
3. 优先使用 `sealed class` / `sealed interface` 表达有限状态集
4. 循环优先 `forEach` / 序列操作，嵌套过深时用普通 `for`
5. 字符串模板优先 `${obj.name}`，不要手动拼接 `+`
6. 长参数列表考虑 `@JvmOverloads` + 默认参数，或拆成配置 data class

# Decision

| 场景 | 选择 |
|---|---|
| 简单 DTO | `data class`（自动 equals/hashCode/toString） |
| 有限状态（如消息类型、UI 状态） | `sealed class` / `sealed interface` |
| 工具方法集合（纯静态） | 顶层函数 → `xxxExtensions.kt` |
| 有状态单例 | `object` 关键字 |
| 持有 Context 的管理器 | `Manager` 类，在 Application 初始化时注册 |
