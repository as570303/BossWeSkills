---
name: android-ui-coding
description: Android UI 开发规范（FileMatch：**/ui/**/*.kt, **/layout/*.xml, **/res/**/*.xml）
activation: fileMatch
filePattern:
  - "**/ui/**/*.kt"
  - "**/ui/**/*.java"
  - "**/layout/*.xml"
  - "**/res/drawable/*.xml"
  - "**/res/values/*.xml"
---

# Purpose

Android XML 布局、Activity/Fragment/自定义 View 等 UI 开发规范。

# MUST

1. **返回键适配**：Activity 必须使用 `OnBackPressedDispatcher.addCallback`，**不得重写 `onBackPressed()`**（Android 16+ 强约束）
2. **ViewBinding 优先**：新增 UI 代码一律用 ViewBinding（`buildFeatures.viewBinding = true`），老代码 ButterKnife 维持不动
3. **布局前缀严格匹配**：
   - Activity 布局：`activity_xxx.xml`
   - Fragment 布局：`fragment_xxx.xml`
   - Dialog 布局：`dialog_xxx.xml`
   - RecyclerView item：`item_xxx.xml`
   - 自定义组合布局：`view_xxx.xml` / `layout_xxx.xml`
   - PopupWindow：`pop_xxx.xml`
   - 头部/尾部：`header_xxx.xml` / `footer_xxx.xml`
   - 空状态：`empty_xxx.xml`
4. **ID 命名**：推荐 `控件缩写_业务` 下划线风格（`tv_title`、`rv_list`、`btn_send`、`iv_avatar`），同一文件内风格一致
5. **聊天气泡操作图标**命名固定为 `icon_operate_xxx`（drawable）
6. **多语言资源**：新增 `strings.xml` key 必须同步到 `values-zh-rCN`、`values-zh-rTW`、`values-ja-rJP` 三个目录
7. **RecyclerView 适配器**：优先用 BRVAH（BaseRecyclerViewAdapterHelper），聊天消息多类型用 `ItemProvider` 模式
8. **屏幕方向**：聊天/支付/登录等核心 Activity 固定 `portrait`，Manifest 中显式声明

# MUST NOT

1. 不要在 Activity/Fragment 中直接写网络请求逻辑（移到 Repository/Manager）
2. 不要在 XML 中硬编码颜色值/尺寸值，统一抽 `colors.xml` / `dimens.xml`
3. 不要用 `wrap_content` + `scrollbars` 嵌套多个层级，注意过度绘制
4. 不要在非 UI 线程操作 View，必须切主线程

# SHOULD

1. **沉浸式状态栏**：优先用 ImmersionBar（3.2.2），`fitsSystemWindows` 配合主题设置
2. **加载状态**：用 StatusLayoutManager 或统一的 `view_empty.xml` 管理 Loading/Empty/Error
3. **刷新**：列表刷新用 SmartRefreshLayout 2.x（`refresh-header-classics` / `refresh-footer-classics`）
4. **弹窗**：优先 XPopup 2.x（弹窗、底部弹窗、输入弹窗），复杂自定义走 dialoglibrary
5. **图片加载**：统一 Glide 4.12 + Glide 变换库（模糊/圆角等），不要混用 Fresco/Picasso
6. **圆角/边框背景**：优先 ShapeView / ShapeDrawable（getActivity ShapeView:10.0），少写大量 shape xml
7. **点击防抖**：使用项目已有 `View.click` 扩展或 RxBinding 防抖，避免快速点击重复触发

# Decision

| 场景 | 选择 |
|---|---|
| 通用弹窗（确认/列表/输入） | XPopup → dialoglibrary 封装 |
| 加载中 Loading | loadingDialogLibrary / StatusLayoutManager |
| Toast | toastlibrary（统一 UI 风格） |
| 选择文件/图片/拍照 | selector 库（PictureSelector 3.x 封装） |
| 视频播放（非通话） | GSYVideoPlayer / jiaozivideoplayerlibrary |
| PDF 预览 | pdf-viewer（barteksc） |
| 日期/地址选择器 | AndroidPicker（gzu-liyujiang） / Android-PickerView |
| 列表侧滑菜单 | EasySwipeMenuLayout / recyclerview-x |
