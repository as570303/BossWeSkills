---
name: project-rules
description: Android IM 项目核心规则与红线（Always 自动加载）
activation: always
---

# Project Rules

## Project

- **项目定位**: IM 即时通讯 + 社群 + 音视频会议 + 钱包支付 的综合社交应用
- **主语言**: Kotlin（Java 仅存于历史模块，新增代码一律用 Kotlin）
- **构建**: Gradle 8.7.3 + AGP 8.7.3 + Kotlin 2.1.0，Java 17
- **版本目录**: `gradle/libs.versions.toml` 统一管理依赖版本
- **SDK 版本**: compileSdk = 35（Android 15），minSdk = 24，targetSdk = 35
- **多 Flavor**: official（正式包）/ beta（测试包）/ gamedev
- **多语言**: zh-rCN（简中）、zh-rTW（繁中）、ja-rJP（日语）

## Architecture

### 模块划分
- **主应用**: `app/` — Product Flavor、Application 入口、主 Activity/Fragment
- **基础能力模块**: `*library/`（如 netlibrary、socketlibrary、widgetlibrary、utilslibrary）
- **业务模块**: `module_*`（如 module_shop）
- **跨模块路由**: GoRouter（arouterlibrary），**禁止直接 startActivity 引用其他模块 Activity**

### 架构模式
- 混合架构：传统 MVP（`mvp/` 包）+ ViewModel + LiveData + Coroutines
- 数据层：Repository → DataSource（`net/` 网络 / Room 数据库 / Socket 长连接）
- 初始化统一走 `init/` 包下 AppInitManager 分模块初始化

### 核心能力模块
| 模块 | 职责 |
|---|---|
| `netlibrary` / `app/src/main/java/.../net/` | HTTP 请求（Retrofit + OkHttp + RxJava/Coroutines） |
| `socketlibrary` / `service/SocketService.kt` | IM 长连接（Netty + Protobuf） |
| `wdpushlibrary` | 推送通道 |
| `rtccalluilibrary` | 音视频通话 UI（WebRTC / LiveKit） |
| `translationlibrary` | 翻译与语音转文字 |
| `modellibrary` | 数据模型（DB Entity / DTO） |
| `resourceslibrary` | 内置资源（emoji 数据库、省市区 JSON、扫描配置） |
| `nativekeys` | 原生密钥（CMake 构建 libxm_keys.so） |
| `widgetlibrary` | 自定义 View 组件 |
| `opensdklibrary` | 第三方开放平台集成（微信等） |

## 关键目录

```
app/src/main/java/com/sharexm/xm/
├── bean/              # 数据模型（Model/Bean/Dto/Extra）
├── event/             # EventBus 事件
├── init/              # 应用初始化分模块
├── manager/           # 业务管理类
├── mvp/               # MVP 结构（contract/model/presenter）
├── net/               # 网络层（Retrofit Api、Repository、Converter）
├── repository/        # 数据仓库
├── service/           # Service（Socket 服务、DB 服务）
├── service/db/        # 数据库 Manager（Room 封装 + SQLite 历史）
├── ui/                # UI 层
│   ├── activity/      # Activity
│   ├── fragment/      # Fragment
│   └── adapter/       # Adapter / ItemProvider
└── BaseActivity.java  # 基类
```

## 全局命名规范

### 类后缀（强约束）
| 类型 | 后缀 | 示例 |
|---|---|---|
| Activity | `Activity` | `ChatActivity` |
| Fragment | `Fragment` | `ChatFragment` |
| Dialog | `Dialog` | `LoadingDialog` |
| Adapter | `Adapter` | `EmotionsAdapter` |
| ViewHolder | `Holder` | `MessageHolder` |
| ItemProvider | `ItemProvider` | `TextMessageItemProvider` |
| ViewModel | `ViewModel` | `SessionViewModel` |
| PopupWindow | `PopupWindow` | `MeetingMorePopupWindow` |
| 自定义 View | `View` / 具体控件名 | `IndicatorView` |
| 工具类 | `Util` / `Helper` | `DeviceUtil` |
| 管理类 | `Manager` | `WxShareManager` |
| 常量容器 | `Constants` | `ChatConstants` |

- **基类前缀**: `Base`（`BaseActivity`、`BaseViewModel`），新增不用 `Abs`
- **Kotlin 接口**: 不加 `I` 前缀；Java 接口若模块已有 `I` 前缀则延续

### 变量与方法
- 变量/字段：小驼峰，**不加 `m`/`s` 匈牙利前缀**
- 默认 `val` 不可变，仅必要时 `var`
- 可空类型显式 `?`，不滥用 `!!`
- 方法：小驼峰，动词开头；布尔返回以 `is/has/can/should` 开头
- 回调方法以 `on` 开头：`onItemClick`

### 常量
- `const val` / `static final`：全大写下划线
- `const val MAX_MESSAGE_LENGTH = 5000`

### 资源命名
- 布局：`activity_*.xml` / `fragment_*.xml` / `dialog_*.xml` / `item_*.xml` / `view_*.xml` / `pop_*.xml`
- drawable：`bg_*`（背景）/ `ic_*`（图标）/ `shape_*`（形状）/ `selector_*`
- 聊天气泡操作图标：`icon_operate_xxx`（固定约定）
- 颜色：语义化 `color_*` / `bg_*` / `text_*`，不用 `color1`/`red2`
- ID：推荐 `tv_title` / `rv_list` / `btn_send` 下划线风格

## MUST NOT（工程红线）

### 编译红线（违反直接报错）
1. **原生库 16KB 页对齐**：所有 `.so` 的 ELF LOAD 段 `Align ≥ 0x4000`
   - 影响目录：`app/src/main/jniLibs/`、`livebusinesslibrary/src/main/jniLibs/`
   - 新增/替换 `.so` 后必须用 `readelf -l` 校验
2. **CompoundButton / RadioGroup 监听器非空**：
   - `CompoundButton.OnCheckedChangeListener` 第一个参数必须是 `CompoundButton`（非 `CompoundButton?`）
   - `RadioGroup.OnCheckedChangeListener` 第一个参数必须是 `RadioGroup`（非 `RadioGroup?`）
   - 原因：Android 15（API 35）给参数加了 `@NonNull`
3. **Activity 返回导航**：必须用 `OnBackPressedDispatcher`，**不得重写 `onBackPressed()`**
   - 适配 Android 16+ 预测式返回手势

### 功能红线
4. **NEARBY_WIFI_DEVICES 权限**：`AndroidManifest.xml` 必须声明此权限（WebRTC 本地网络发现需要）
5. **跨模块跳转**：一律走 GoRouter，**禁止直接 import 其他模块 Activity 类**
6. **前台服务类型**：`foregroundServiceType` 必须与使用场景匹配（targetSdk 35 强校验）

### 工程行为红线
7. **不要在 Activity/Fragment 中直接执行网络请求**，走 Repository / UseCase / Manager
8. **不要新增 deprecated API 的调用**
9. **不要为简单需求引入新的第三方依赖**，优先复用 utilslibrary / widgetlibrary 已有能力
10. **不要修改与当前任务无关的代码**，不做范围外的风格统一重构
11. **不要硬编码密钥/URL**，密钥放 nativekeys（libxm_keys.so），配置放 configlibrary 或 flavor assets

## AI Coding 基本行为

### 修改前
- 先读现有实现，理解上下文再动手
- 检查同目录是否已有类似能力可复用
- 确认目标模块是否需要跨模块路由（GoRouter path）

### 修改中
- **最小修改范围**：只改与任务直接相关的代码
- 新增代码用 Kotlin；修改 Java 历史代码不强行转 Kotlin
- 优先用项目已有工具库（Blankj utilcodex、dialoglibrary、toastlibrary 等）
- 不进行无关重构，不把老代码的命名风格强行统一
- 多语言资源缺译需同步添加到 `values-zh-rCN`、`values-zh-rTW`、`values-ja-rJP`

### 修改后
- 检查 Build（Kotlin 编译）是否通过
- 涉及 Manifest 权限变更时同步检查 targetSdk 匹配
- 涉及原生库变更时检查 16KB 对齐与 ABI 覆盖（arm64-v8a / armeabi-v7a）

## Decision（快速选择）

### 网络请求
```
简单同步/异步 → net/ 下 Retrofit Api + Repository
IM 消息推送  → socketlibrary / SocketService
上传文件     → 走 UploadInit / toslibrary（火山引擎 TOS）
```

### 数据存储
```
KV 轻量配置  → MMKV（优先） / SP
结构化数据   → Room（modellibrary DB Manager）
大量聊天记录 → DBManager（ChatManager 封装）
内置静态资源  → resourceslibrary assets
```

### UI 注入
```
老项目代码    → ButterKnife（维持不动）
新增代码      → ViewBinding（buildFeatures.viewBinding = true）
列表适配器    → BRVAH（BaseRecyclerViewAdapterHelper）
```
