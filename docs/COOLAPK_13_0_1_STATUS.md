# 酷安 13.0.1（2301171）适配状态

本文记录把 `coolapk-purifier` 适配到酷安 **13.0.1** 的分析结论与证据。

**结论先行：13.0.1 无法用本模块现有的钩子集适配。** 详情见下方证据。本仓库已为此 fork 补齐了 GitHub 云端构建/发布与上游同步（见 `../.github/workflows/`），但 13.0.1 的功能性去广告钩子不可移植，原因如下。

## 版本事实

| 项目 | 值 |
| --- | --- |
| 包名 | `com.coolapk.market` |
| versionName | `13.0.1` |
| versionCode | `2301171` |
| 来源 APK | `Xposed-Modules-Repo/io.github.fentia.coolapkreviver` 发布的 `CoolApk_13.0.1.apk` |

- `coolapkreviver` **只伪造 OkHttp 请求头**（`X-App-Code` 由 `2301171` 改成 `2609291`）以对接新版接口，**不修改 APK 的 `versionCode`**。因此真实 `versionCode` 就是 `2301171`，本模块的 `versionCode` 命中逻辑无需改动。
- 13.0.1 是**明文 dex**（无 360 壳/运行时解密），`classes*.dex` 共 7 个、约 60184 个类，可直接离线分析。上游 `ADAPTATION_WORKFLOW.md` 提到的“壳 APK 需运行时 dump”只适用于 16.6.x，不适用于 13.0.1。

## 14 个 manifest target 的逐项目分析

分析工具：仓库内 `tools/manifest-generator`（dexlib2 离线解析）+ 针对性 dex 探针，直接扫描 13.0.1 的 7 个 `classes*.dex`。

| Target | 16.6.x 类型 | 13.0.1 结论 | 证据 |
| --- | --- | --- | --- |
| `detailSponsor` | getter `$$AutoValue_Feed.getDetailSponsorCard()` | **存在但无法落地** | getter 精确命中；但其 UI 终端 `detailSponsorUi` 模板缺失，D1 安装要求 getter+ui 同时存在（见下） |
| `detailSponsorUi` | ViewHolder binder | **缺失** | 模板字符串 `sponsorForFeedDetail` 在 13.0.1 dex 中 **0 次出现** |
| `replySponsor` | ViewHolder binder (D2) | **不可映射** | 模板 `feedDetailReplySponsorCard` 存在，但只出现在 `EntityAdHelper`、`FeedReplyListFragmentV8` 等，没有满足 D2 契约的 final ViewHolder（构造器 `(View, EntityAdHelper, DataBindingComponent)` + `void(Object)` 覆写抽象父类 + 静态 `R.layout` 字段 + `itemView`） |
| `sameTopic` | 事件流 (D3) | **缺失** | 模板字符串 `feedRecommendListCard` 在 13.0.1 dex 中 **0 次出现** |
| `topicDeviceRecommend` | 静态装配 (D5) | **缺失** | Compose 签名 `(Feed, owner, Composer, int)→Unit`；13.0.1 仅 1 个类引用 `androidx.compose.runtime.Composer`，宿主类不存在 |
| `topicDeviceRecommendUi` | Compose composable (D5) | **缺失** | Compose 组合函数 `(Modifier, FeedTarget, FeedDetailV13ViewModel, Composer, int)→void` 不存在 |
| `autoComment` | 静态装配 (D6) | **缺失** | 类 `RecyclerViewItemFullVisibleControllerKt` 在 13.0.1 dex 中 **0 次出现** |
| `autoCommentPrompt` | 静态装配 (D6) | **缺失** | `RecyclerViewItemFullVisibleControllerKt$addAutoShowFeedCommentView$1` 不存在 |
| `relatedData` | getter（仅观察） | **存在但无效果** | `$$AutoValue_Feed.getRelatedData()` 精确命中；但该钩子 `onGetter` 仅 `proceed()` 返回原值（observe-only），必须配合下方的 UI binder 才生效，而 13.0.1 没有这些 binder |
| `relatedIconListUi` | ViewHolder binder | **不可映射** | 模板 `iconListCard` 仅被 `EntityListFragment` 引用，没有满足契约的 final ViewHolder |
| `relatedContentUi` | ViewHolder binder | **不可映射** | `FeedDetailV13ViewModel` 类存在，但没有满足 `(View, DataBindingComponent, ViewModel)` 构造器 + `void(HolderItem)` 覆写的最终 ViewHolder |
| `searchHotCapsuleUi` | ViewHolder binder (D7) | **不可映射** | 模板 `capsuleListCard` 仅被 `EntityListFragment` 引用，没有满足 D7 契约的 final ViewHolder |
| `searchHotWordsUi` | ViewHolder binder (D7) | **不可映射** | 模板 `hotSearch` 被 `SearchMainListFragment` 及其内部类引用，没有满足 D7 契约的 final ViewHolder |
| `searchHotRankingsUi` | ViewHolder binder (D7) | **不可映射** | 模板 `searchHotListCard` 被 `SearchMainListFragment` 内部类引用，没有满足 D7 契约的 final ViewHolder |

### 关键观察

1. **连“存在”的 target 也无法落地**：`detailSponsor` 的 getter 在 13.0.1 中精确存在，但 `D1DetailSponsorDelta.install` 要求 `getterSpec` 与 `uiSpec` **同时非空**，否则返回 `TARGET_MISSING`；而 `detailSponsorUi` 的模板在 13.0.1 中缺失。`relatedData` 的 getter 是 observe-only，自身不产生任何去广告效果。
2. **UI 终端契约整体不匹配**：16.6.x 的 D1/D2/D3/D5/D6/D7 钩子都依赖“某一特定 final ViewHolder + `void(Object)` binder + 静态 `R.layout` 字段 + `itemView`”的结构，并在运行时用 `isExactTarget` 做严格校验。13.0.1 的对应广告（回复区赞助、相关推荐图标、热门搜索等）模板字符串只出现在 **Fragment 层级**（`EntityListFragment`、`SearchMainListFragment` 及其内部类），而非独立的最终 ViewHolder，因此契约无法匹配。
3. **Compose 时代特性整体缺失**：`topicDevice*` / `autoComment*` 四类属于 16.6.x 的 Compose/全屏控制器路径，13.0.1 中相关类（`RecyclerViewItemFullVisibleControllerKt`、Compose 组合函数）根本不存在。

## 为什么不能“硬塞”一个 13.0.1 profile

上游 `ARCHITECTURE_INVARIANTS.md` / `ADAPTATION_WORKFLOW.md` 明确：

- manifest 解析 **fail-closed**：`status=validated` 的 profile 必须包含完整的**核心 target**（D1/D2/D3/D5/D6 与 `relatedData` observe-only getter），缺失任何一个都会让整个 manifest 解析失败；UI 子类 target（`relatedIconListUi`/`relatedContentUi`/三个搜索 UI target）才是 feature 级可选。
- `status=draft` 的 profile **永远不会进入生产安装路径**（`validatedProfileFor` 只返回 validated）。
- 未知宿主版本 → `UNSUPPORTED`，不做 nearest fallback、不做模糊猜测。

由于上面第 1–3 点，13.0.1 **无法凑齐一个完整的核心 target 集合**，因此：
- 若写成 `validated`：解析会因核心 target 缺失而整体失败（整个模块不可用，比“不支持”更糟）；
- 若写成 `draft`：永远不会被加载，13.0.1 仍表现为 `UNSUPPORTED`（诚实但无功能）。

两种方式都不能让 13.0.1 真正去除广告。

## 已交付内容（本 fork）

- `.github/workflows/build.yml`：AGP 8.12 / JDK 17 / compileSdk 37 云端构建 `assembleCompatible`（自带的 debug 签名可直接安装），打 `v*` tag 时自动发 GitHub Release。
- `.github/workflows/sync-upstream.yml`：每周 + 手动把 `yylsping/coolapk-purifier` 上游合并进 fork 并开 PR，fork 本地改动不会被静默覆盖。
- 本文档：记录 13.0.1 的分析结论与证据。

## 若仍需 13.0.1 去广告：下一步

要在 13.0.1 上真正去除广告，需要**针对 13.0.1 的架构另写一套钩子**（而不是移植现有钩子），工作量与一次完整适配相当，且需要在真机/模拟器上验证。大致范围：

- 13.0.1 的赞助卡由 `com.coolapk.market.view.ad.SponsorSelfDrawViewHolder` 系列（布局 `item_sponsor_self_draw*`、`item_grid_sponsor_self_draw_*`、`item_sponsor_self_draw_news_*`）承载；
- 需要为 13.0.1 重新定位回复区赞助、详情页赞助、相关推荐、热门搜索等广告的注入点，并新增对应的 `D*` 钩子与 `TargetSpec`，再写一个 13.0.1 的 validated profile。

这属于独立的较大工程，不在“适配现有模块”的范围内，建议作为单独任务评估。
