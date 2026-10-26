# 酷安净化

基于 LSPosed 的酷安净化模块，可隐藏广告和多类冗余内容，并在酷安设置中提供独立开关。

## 功能与默认设置

打开酷安“设置”→“酷安净化”，可分别调整以下 10 项功能：

| 功能 | 全新配置默认值 |
| --- | --- |
| 去除启动/开屏广告和全屏广告 | 开启 |
| 去除首页信息流广告与赞助卡片 | 开启 |
| 去除帖子回复区及评论中的赞助内容 | 开启 |
| 去除自动评论提示 | 关闭 |
| 去除话题与机型推荐 | 关闭 |
| 去除帖子相关推荐 | 关闭 |
| 去除同话题动态 | 关闭 |
| 去除帖子内推广 | 关闭 |
| 去除热门搜索与热榜热词 | 关闭 |
| 去除搜索页热榜 | 关闭 |

设置由 LSPosed 保存，升级会保留已有开关选择，无需将模块加入自启动或后台白名单。更改选项后，请强制停止酷安并重新打开；若提示“设置未保存”，请重试。

使用广告过滤功能仍可能小概率触发账号风控。

## 兼容性

| 项目 | 要求 |
| --- | --- |
| 目标应用 | 酷安，包名 `com.coolapk.market` |
| 酷安版本 | 16.6.1（2608212）、16.6.2（2609151）、16.6.4（2609291）、16.6.5（2610162）支持全部 10 项功能 |
| Android | 9.0 及以上 |
| 框架 | 支持 libxposed Modern API 102 的 LSPosed |
| 模块版本 | 2.6.1 |

开屏和首页信息流会尝试自动适配；其他功能仅在上表列出的酷安版本中启用。无法适配时会保留酷安原有显示。

> **关于酷安 13.0.1（2301171）**：本模块**暂不支持**该版本。13.0.1 的明文 dex 与 16.6.x 的广告渲染架构差异过大，现有 `MANIFEST_EXACT` 钩子的 14 个 target 无法映射（详见 [docs/COOLAPK_13_0_1_STATUS.md](docs/COOLAPK_13_0_1_STATUS.md)）。要在 13.0.1 上真正去广告需另写一套针对该版本的钩子，属于独立工作。

## 安装与使用

1. 从 [GitHub Releases](https://github.com/yylsping/coolapk-purifier/releases) 下载并安装 APK。
2. 在 LSPosed 中启用模块，作用域只选择“酷安”。
3. 强制停止酷安后重新打开。
4. 在酷安“设置”→“酷安净化”中调整选项，按提示重新启动酷安。

模块没有独立桌面入口，配置页只能从酷安设置中打开。部分系统每次进入配置页时会显示跨应用打开确认，选择“仅本次允许”即可。首次适配时可能显示一次开屏广告。

## 构建

需要 JDK 17 和 Android SDK Platform 37；模块 targetSdk 为 35。

### 自动化构建（推荐）

本仓库已配置 GitHub Actions：推送到 `main` 会自动构建可直接安装的 `compatible` 包（自带 debug 签名），并对 `v*` tag 自动发布 GitHub Release。无需本地配置 Android SDK 即可获得构建产物。

### 本地构建

```powershell
.\gradlew.bat --no-daemon testDebugUnitTest testCompatibleUnitTest testReleaseUnitTest assembleDebug
```

调试包输出到 `app/build/outputs/apk/debug/`。生成 release 包可使用 `assembleRelease`，签名配置放在本地 `signing-private/` 中。

### 同步上游

本 fork 通过 `.github/workflows/sync-upstream.yml` 与上游 `yylsping/coolapk-purifier` 保持同步：每周一自动尝试把上游 `main` 合并进本 fork 并开 PR（也可在 Actions 页面手动触发）。PR 留待人工审阅合并，避免 fork 本地改动（CI、文档）被静默覆盖；若出现无法自动解决的冲突，会在分支上提示人工处理。

## 问题反馈

请在 [Issues](https://github.com/yylsping/coolapk-purifier/issues) 中提供模块、酷安、Android 与 LSPosed 版本，以及开关状态、复现步骤和必要的脱敏日志。

LSPosed 模块日志可用于排查问题。请勿上传账号凭据或未经脱敏的完整设备日志。

## 许可证

本项目采用 [MIT License](LICENSE)。

本项目用于学习、研究和个人设备使用，与酷安及 LSPosed 项目无隶属或认可关系。
