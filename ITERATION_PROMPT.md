# Readest Lite — 持续迭代助手提示词（v8.13.1）

> 把这段提示词完整粘贴给后续的 AI 助手。

---

## 项目背景

你正在维护 **Readest Lite**（https://github.com/cshdotcom/readest-lite），这是 Readest 的轻量化自托管分支。

**核心改造原则**（不可回滚）：

1. 数据库：Supabase Postgres → SQLite + Prisma 5.22
2. 文件存储：R2/S3 → 本地文件系统 + HMAC-SHA256 签名 URL
3. 鉴权：Supabase GoTrue → 本地 JWT **多用户**系统
4. Pro/付费体系：**完全删除**
5. 注册：**完全禁用**
6. 同步/分享/阅读器：**1:1 复刻原版** + 持续合并上游修复
7. 单 Docker 镜像，端口 8225，数据卷 `/data`
8. 字体本地化
9. DeepL 默认关闭
10. **v8.0：所有翻译/词典代理强制要求登录**
11. COEP 全站 credentialless
12. 远程书籍下载：`/api/books/download-url`
13. **v8.2：代理开关 proxyEnabled**
14. **v8.3：账号切换数据隔离**
15. **v8.4：Per-user 加密 vault — 服务端托管密钥 + AES-GCM 加密**
16. **v8.5：配额 enforce + 真实配额 UI + SSRF 黑名单 + fire-and-forget 下载**
17. **v8.6：合并上游 0.11.12 + 图片保存 + 分享永久/自定义有效期 + 下载任务队列**
18. **v8.7：跨设备下载任务队列（DownloadTask 表 · 暂停/恢复/重试 · 5s 轮询 · 异步后台下载）**
19. **v8.8：分块上传规避 Cloudflare 524 超时（大文件自动切 5MB · 服务端流式合并 · 小文件零变化）**
20. **v8.9：下载任务增强（进度条/速度/ETA/用时 · 点击看完整日志 · 批量下载 · 自动重命名（base64/中文/Content-Disposition）· 高级选项（Cookie + Custom Headers））**
21. **v8.10：中文汉化 + 笔记链接手机默认走 web reader + 登出清空残留 library.json + 阅读统计（总/今日/本周 + 书榜） + 下载记录折叠**
22. **v8.10.1：批量下载 per-URL Cookie/Headers 语法（URL | cookie:VALUE | header:Key: VALUE）**
23. **v8.10.2：笔记导出链接绝对 URL 修复（站外可点击）+ Reader 书不在库里时自动重试加载 + 用户管理折叠**
24. **v8.10.3：登出后隐藏全部书籍（修复 demo books 残留）+ 移出分组不再踢出用户（修复 group-empty auto-navigate）**
25. **v8.10.4：恢复跨设备视图设置同步（可选）— 用户中心 → Manage Sync → View Settings toggle**
26. **v8.11.0：合并上游 v0.11.17（Markdown 渲染 + PDF 对比度 + TTS 粒度 + 最近阅读书架 + foliate-js 自动更新）**
27. **v8.12.0：逐文件对比同步上游 v0.11.17（163 文件复制 + 21 新文件 + 58 Lite 自定义保留 + Google Drive/payment 排除）**
28. **v8.12.1：紧急修复 v8.12.0 误覆盖 Lite 自定义（getRemoteBookFilename 在 'local' 返回空串 → 下载报 File not found + AboutWindow 被覆盖成上游版本 → 关于弹官方页面）**
29. **v8.12.2：防御性文件名生成（即使 getStorageType/book.format/book.title 异常也永不返回空串）+ 上传/下载失败响应体加 hint/received 字段方便诊断 + 放宽 download fallback parts.length 检查**
30. **v8.13.0：上游 Readest v0.11.18 非覆盖式合入（Auto Scroll 阅读模式 + 中键自动滚动 + PDF 暗色模式页眉页脚 + 两指滚动 vs 捏合 + View Transitions API gating + TTS 大改版：无缝 Web Audio + 关书继续播放 + mini player + 词典朗读按钮 + 主题分段控件 + 校对规则开关 + OPDS 子目录爬取 + Calibre 自定义列 + 按阅读进度排序）**
31. **v8.13.1：翻页动画（幻灯片/卷页，PR #4940 JS-only port）+ 修复「关于」页面版本号（package.json 0.11.4 → 8.13.1）+ bookService 并发导入崩溃修复（createDir 幂等，PR #4962）**

---

## 版本与 Tag 规则

**每一个新版本必须打 git tag。**
- 推送时 `git push && git push --tags`
- 用户拉取：`docker pull ghcr.io/cshdotcom/readest-lite:8.13.1`

---

## v8.13.1 改动清单

### v8.13.1 — 翻页动画 + 关于页面版本号修复 + bookService 崩溃修复

**PR #4940 翻页动画（幻灯片/卷页）— JS-only port**：
- 新增文件：`utils/pageCurl.ts`、`utils/pageSlide.ts`、`app/reader/hooks/useCapturedTurn.ts`、`app/reader/utils/capturedTurn.ts`
- `types/book.ts`：新增 `PageTurnStyle` 类型 + `pageTurnStyle` 字段
- `services/constants.ts`：`DEFAULT_VIEW_CONFIG` 加 `pageTurnStyle: 'push'`
- `utils/bridge.ts`：加 `captureWebviewRegion`（web 端 reject，回退 CSS curl）
- `FoliateViewer.tsx`：调用 `useCapturedTurn` + `applyPageTurnAttributes`，移除 inline no-swipe 块
- `BooksGrid.tsx`：加 `data-view-transition-root` 属性
- `ControlPanel.tsx`：加 Animation Style SettingsSelect（Push/Slide/Page Curl，按 `supportsViewTransitionGroup` gating）
- 跳过原生代码（swift-rs/Rust/Kotlin/Swift）— Lite 是 web-only

**PR #5000 gate captured slide/curl on scrollLocked**：已包含在复制的 `useCapturedTurn.ts` 中

**PR #4962 bookService 崩溃修复**：`bookService.ts` 的 `createDir` 改为幂等递归（`createDir(path, base, true)`），修复并发导入竞争

**关于页面版本号修复**：`apps/readest-app/package.json` version 从上游 `0.11.4` 改为 Lite 真实版本 `8.13.1`。`getAppVersion()` 读 `package.json.version`，AboutWindow 显示 `Version 8.13.1`

**最终可用 commit**：`d584c8e`（v8.13.1 tag 指向）

---

## v8.13.0 改动清单

### v8.13.0 — 上游 Readest v0.11.18 非覆盖式合入

**方法**：逐 PR 研究、逐文件改、不覆盖 Lite 自定义代码。合入 19 个 PR，排除 15 个不适用 PR。

**第一批 阅读器核心修复（7 PRs）**：
- PR #4910 注解深链不同书切换（useOpenAnnotationLink + useBooksManager.openBookInReader + goToCfiWhenReady）
- PR #4912 两指滚动 vs 捏合缩放（useIframeEvents 延迟决策 + 24px 死区）
- PR #4911 PDF 暗色模式页眉页脚（mix-blend-difference）
- PR #5001 页边距收缩到安全区（fixed-layout 用 text-base-content）
- PR #4989 View Transitions API gating（新增 viewTransition.ts + supportsViewTransitionsAPI/Group）
- PR #4955 中键自动滚动（新增 useMiddleClickAutoscroll + AutoscrollIndicator + autoscroller.ts）
- PR #4998/#4999 Auto Scroll 阅读模式（新增 useAutoScroll + AutoScrollControl + PacedScroller + Shift+A 快捷键）

**第二批 TTS 大改版（5 PRs）**：
- PR #4931 Edge TTS 无缝 Web Audio（新增 WebAudioPlayer/pcm/timeStretch/ttsDuration/SectionTimeline）
- PR #4941 关书继续播放（新增 TTSSessionManager + ttsMediaBridge + NowPlayingBar）
- PR #4957 词典弹窗朗读按钮（新增 wordPronouncer）
- PR #4994 Android TTS 媒体控制（TS 侧 only，移除 alwaysInForeground）
- PR #4996 TTS mini player（新增 TTSMiniPlayer/TTSPlayerSheet/TTSScrubber，删除 TTSBar/TTSIcon/TTSPanel）

**第三批 图书馆/UI/OPDS（7 PRs）**：
- PR #4947 传输队列持久化（transferManager 加 clearCompleted/clearFailed/clearAll）
- PR #4831/#4913 主题分段控件（ThemeModeSelector 重写）
- PR #4859 校对规则开关（ProofreadRules 重写）
- PR #4948 OPDS 子目录爬取（feedChecker + MAX_CRAWL_DEPTH）
- PR #5002 OPDS 自托管认证（isPrivateHostAllowed + 400 重试 + skipSslVerification）
- PR #4939 Calibre 自定义列（CalibreCustomColumn + formatCalibreColumnValue + getCalibreColumnsText）
- PR #4427 按阅读进度排序（LibrarySortByType.Progress + getBookReadRatio）

**foliate-js 类型声明**：新增 `src/foliate-js.d.ts`，声明 `getSentences`/`SentenceEntry`/`TTS`（TTS 声明为 `any` 类型匹配上游 allowJs 宽松推断）。解决 Lite Docker 构建（git clone）vs 上游（git submodule）的类型推断差异。

**排除的 PR**：Sentry（#4914/#4929/#4952/#4958）、iOS/macOS 原生（#4917/#4891/#4896/#4890/#4909）、Nix（#4883/#4932）、Calibre 插件（#4918）、turso（#4927）、Android e2e（#4921）、koplugin（#4954）、PR #4949（被 #4989 取代）、S3 sync 系列（#4990/#4976/#4975/#4982/#4973/#4971/#4981/#4946/#4944/#4928）、watched folder（#4902，留待 v8.14）

**最终可用 commit**：`11d195b`

---

## v8.12.0 改动清单

### v8.12.0 — 逐文件对比同步上游 v0.11.17

**方法**：逐文件对比 Lite 和上游 v0.11.17，分类处理：
- 163 个安全文件：直接从上游复制
- 21 个新文件：从上游添加（排除 Google Drive/payment/tests）
- 58 个 Lite 自定义文件：**绝不覆盖**，只手动添加缺失字段

**Lite 自定义保留**：VaultProvider、PHContext、本地 JWT 认证、Prisma+SQLite、本地文件存储、登出清空、用户/demo 守卫、syncViewSettings、Bookshelf group-empty 修复、deeplink resolveWebBaseUrl、proxyEnabled、WebDAVSyncLogEntry、下载任务、阅读统计、用户管理、RemoteDownloadDialog、所有 pages/api/*

**排除**：Google Drive、Stripe/Apple/payment、所有 __tests__

**手动添加的类型字段**：webtoonMode、showStickyProgressBar、wordLensGlossFontSize/Color、SearchMode、mode、abandoned、coverHash/coverUpdatedAt、fraction、deletedAt+updatedAt(ProofreadRule)、nearbyWords、segments+emphasized+text(SearchExcerpt)、refresh(HardwarePageTurner)、biometricUnlockEnabled、WebDAVBrowseSortByType、browseSortBy/browseSortAscending、synced_at/uploaded_at/downloaded_at(BookDataRecord)

**最终可用 commit**：`e85e8ce`

---

## v8.11.0 改动清单

### v8.11.0 — 合并上游 v0.11.17 功能

**合并的上游 PR**：
- #4816 Markdown (.md) 文件渲染
- #4800 PDF/CBZ 对比度选项
- #4807 TTS 高亮粒度（单词/句子）
- #4829 最近阅读书架
- #4820 自动翻页角落区域上限
- #4804 清理空高亮
- foliate-js 自动更新（pinch-zoom + scroll lag 修复）

**未合并**：
- #4846 双击选词 — 依赖链太深（5+ 文件 API 变更），已回滚

**最终可用 commit**：`aca02d3`

---

## v8.10.4 改动清单

### v8.10.4 — 恢复跨设备视图设置同步（可选）

**背景**：v8.6.0 合并上游 PR #4672 时把 view settings（字体/主题/排版）改成设备本地，只同步 CFI。用户反馈希望恢复跨设备同步。

**实现**：不再硬编码设备本地，改为可选开关：
- `settings.ts` 新增 `syncViewSettings?: boolean`（默认 false）
- `useProgressSync.ts` 检查 `settings.syncViewSettings`，true 时合并远端 config 到本地
- `SyncCategoriesSection.tsx` 加独立的 View Settings toggle

**向后兼容**：默认 false 保持 v8.6 行为，老用户升级不变。想要跨设备同步的用户去 用户中心 → Manage Sync → View Settings 开启。

---

## v8.10.3 改动清单

### v8.10.3 — 登出隐藏全部书籍 + 移出分组不再踢出用户

**1. 登出后书籍全部隐藏（修复 demo books 残留）**
- 问题：登出后个别书还显示
- 根因：demoBooks effect 在 libraryLoaded=true 时重新添加 demo books，登出后 libraryLoaded 被 initLibrary 重设为 true，demoBooks state 还在 → effect 触发 → demo books 加回来
- 修复：demoBooks effect 加 !token || !user 守卫

**2. 移出分组不再踢出用户（修复 group-empty auto-navigate）**
- 问题：用户在分组里把最后一本书移出 → currentBookshelfItems.length === 0 → 立即 updateUrlParams({ group: null }) → 被踢回根书库 → 再点进去又被踢出
- 修复：只在 getGroupName(groupId) 返回空（group 真被删了）才导航走；group 暂时为空时显示空状态提示

**最终可用 commit**：`e16f7b9`

---

## v8.10.2 改动清单

### v8.10.2 — 笔记链接修复 + 用户管理折叠

**1. 笔记导出链接打不开（核心 bug）**
- 问题：`buildAnnotationWebUrl` 用构建期 `READEST_WEB_BASE_URL`（Readest Lite 里为 `''`），导出链接是相对路径 `/o/book/...`，站外点不开
- 修复：新增 `resolveWebBaseUrl()` 运行时解析，优先用 `runtimeConfig.apiBaseUrl`（`PUBLIC_BASE_URL` 注入），回退 `window.location.origin`
- 导出链接现在永远是绝对 URL

**2. Reader 书不在库里时自动重试**
- 问题：用户从笔记链接进入 `/reader/{hash}`，内存 library 还没加载完，`getBookByHash` 返回 undefined，报「无法打开书籍」
- 修复：`initViewState` 失败时调 `appService.loadLibraryBooks()` 重读磁盘，找到继续，找不到才报错

**3. 用户管理折叠**
- `UserManagement.tsx` 默认只显 3 个用户，「查看全部」按钮打开 `AllUsersModal`

**最终可用 commit**：`6cf4ad7`

---

## v8.10.1 改动清单

### v8.10.1 — 批量下载 per-URL Cookie/Headers 语法

**问题**：v8.9 批量下载只支持一组全局 Cookie/Headers，不同网站要分多次提交。

**解决**：扩展 textarea 语法：
```
# 注释
https://site-a.com/book.epub | cookie:sessionid=abc123
https://site-b.com/book.epub | cookie:PHPSESSID=def | header:Referer: https://site-b.com
```

**合并逻辑**：per-URL 指令优先，没有指令的 URL 回退到全局 Advanced Options 里的 Cookie/Headers。

**向后兼容**：API 同时接受 `items: [{ url, cookies?, headers? }]` 和旧版 `urls: string[] + 全局 cookies/headers`。

---

## v8.10 改动清单

### v8.10.0 — 中文汉化 + 笔记链接手机修复 + 登出安全 + 阅读统计 + 下载折叠

**1. 中文汉化（zh-CN + zh-TW）**
- 补全 v8.7-v8.9 所有新增字符串的中文翻译（60 个 key）

**2. 笔记导出链接手机修复**
- `/o/page.tsx`：手机默认直接跳 web reader，不再尝试启动 App
- `useOpenAnnotationLink.ts`：书不在库里时导航到书库页

**3. 登出后残留书籍修复**
- `handleLogout`：清空 `library.json` + 重置 `libraryLoaded`
- `library/page.tsx`：加 `user/token` 守卫

**4. 阅读统计功能**
- `GET /api/stats/aggregate` 端点
- `ReadingStatsCard.tsx`：横向滚动卡片 + 点击打开 Modal
- `ReadingStatsModal.tsx`：Tab + 书榜排序 + 渐变进度条 + 搜索

**5. 下载记录折叠**
- `DownloadTasks.tsx` 默认只显 3 条 + 「查看全部」按钮
- `DownloadTasksModal.tsx` 完整列表

**最终可用 commit**：`fa35e84`

---

## v8.9 改动清单

### v8.9.0 — 下载任务增强

**问题背景**：用户要求：下载进度条、速度、ETA、用时、完整日志（点击任务弹出）、批量下载、自动重命名（base64/中文/查询参数）、Cookie + Custom Headers 高级选项。

**架构**：
- `filenameDetect.ts` — 智能文件名识别（优先级：Content-Disposition > URL path > URL query `file=` > base64 > Content-Type > fallback）
- `downloadRunner.ts` — 共享下载执行器（被 create/retry/batch 共用）
  - 流式读取 `response.body.getReader()`
  - 每秒 throttle 写库 progress/downloadedBytes/totalBytes/speedBps/etaSeconds
  - 每 2 秒独立检查暂停状态（不被进度更新干扰）
  - 完整日志写入 DownloadLog 表（info/warn/error）
- 新增 API `GET /api/download-tasks/[id]/logs` 返回任务日志
- POST `/api/download-tasks` 加 `cookies` / `headers` / `batch` 字段
- `RemoteDownloadDialog` 重写：Tab（Single/Batch）+ Advanced Options（Cookie + Headers）
- `DownloadTasks.tsx` 加 progress bar、速度、ETA、用时，点击行打开详情 Modal
- 新增 `DownloadTaskDetailModal.tsx` 显示完整日志（筛选 All/INFO/WARN/ERROR + Auto-scroll）

### v8.9.0 CI 修复

1. `94cc02c` — `filenameDetect.ts` `noUncheckedIndexedAccess` 守卫
   - `starMatch[1]` → 加 `starMatch && starMatch[1]` 守卫
   - `plainMatch[1]` → 加 `plainMatch && plainMatch[1]` 守卫
   - `split(';')[0]` → 用中间变量 + `|| ''` 兑底

**最终可用 commit**：`94cc02c`（v8.9.0 tag 指向此 commit）

---

## v8.8 改动清单

### v8.8.0 — 分块上传规避 Cloudflare 524 超时

**问题**：用户走 Cloudflare 反代访问时，大文件上传超 CF 100 秒硬性超时，返回 524。浏览器报：`File upload failed: Error: Upload failed with status 524`。

**修复**：客户端 `webUpload` 把 >5MB 文件切成 5MB 块，串行 PUT 每块。服务端 `_put.ts` 新增三个分支：
- `merge=1&total=M` → `mergePartsForKey` 流式合并 parts
- `index=N&total=M` → `createPartWriteStream` 写第 N 块到 `<fileKey>.parts/<NNNNN>`
- 无额外参数 → 原整传路径（小文件 + Tauri）

**关键实现**：
- `localStorage.ts::createPartWriteStream` — `index===0` 时先清空 parts 目录（避免重试上传残留旧 part）
- `localStorage.ts::mergePartsForKey` — 用 `Readable.from(async generator)` + `pipeline` 流式 concat，不 buffer 整文件到内存
- `transfer.ts::webUpload` — 进度回调跨块累计，URL 解析用 `window.location.href` 作 base

**向后兼容**：小文件、Tauri 端、旧客户端向新服务端整传 — 全部不受影响。

---

## v8.7 改动清单

### v8.7.0 — 跨设备下载任务队列

- Prisma schema 新增 `DownloadTask` 表（id, userId, url, filename, status, error, bookHash, fileSize, createdAt/startedAt/completedAt）
- API 路由：
  - `GET /api/download-tasks` — 列表
  - `POST /api/download-tasks` — 创建（异步后台 fetch → 写 File + Book 表 → 更新状态）
  - `DELETE /api/download-tasks/[id]` — 删除
  - `POST /api/download-tasks/[id]` — 重试/暂停/恢复
  - `POST /api/download-tasks/batch` — 批量：retry_failed/pause_all/resume_all/clear_completed/clear_failed/clear_all
- 前端 `DownloadTasks.tsx` 组件（用户中心）：5s 轮询、状态图标、批量按钮、单条操作、URL 复制
- `RemoteDownloadDialog` 简化：POST 创建后 toast 提示去用户中心查看

### v8.7.0 CI 修复（3 个 follow-up commit）

1. `78c0deb` — 移除 `[id]/route.ts` 未使用的 `ALLOWED_EXTENSIONS`（TS `noUnusedLocals`）
2. `78c0deb` — 移除 `DownloadTasks.tsx` 未使用的 `IoAlertCircleOutline` import（同上）
3. `e43a3a0` — `eventDispatcher.off('refresh-library', handleRefreshLibrary)` 传 2 个参数（API 签名要求 event + callback）

**最终可用 commit**：`e43a3a0`（v8.7.0 tag 重新指向此 commit）

---

## v8.6 改动清单

### v8.6.0 — 合并上游 Readest 0.11.12

- #4669 z-index scale 修复（SettingsDialog z-[110], ModalPortal z-[120], Alert z-[130], RSVP z-[100/101]）
- #4673 catalog card hover（hover:bg-base-300）
- #4677 sync readingStatusChanged（undefined/null 视为相等，防止无状态书被重新钉到顶部）
- #4672 view settings device-local（useProgressSync 不再合并远端 config，只同步 CFI）
- foliate-js #4670 + #4675 + #4679（Docker build 的 `git clone --depth 1` 自动获取最新，含 PDF OOM 修复、cover bg、cover fill）

### v8.6.1 — 图片保存 + 下载任务队列 + CI 修复

- #4680 image save/share button（ZoomControls + ImageViewer，Web 端用 saveFile 下载）
- RemoteDownloadDialog 改用 transferStore 任务队列（有进度/状态/重试）
- page.tsx 加 `refresh-library` 事件监听
- CI 修复：IoDownloadOutline → IoCloudDownloadOutline，blob → arrayBuffer，移除未使用 import

### v8.6.2 — 分享永久 + 自定义有效期

- ShareBookDialog：SegmentedControl 加 ∞（永久）选项，选中时隐藏日期选择器
- share/create API：`expirationDays=0` 表示永久（expiresAt = year 9999），1-365 天正常过期
- 日期选择器：点击直接弹出浏览器原生 date picker

---

## v8.5 回顾

- 配额 enforce：upload.ts（storageQuotaMB）、translate/google + deepl（translationQuotaKB）
- 新建 `/api/usage` 接口
- useQuotaStats fetch /api/usage，60s 轮询
- Quota.tsx formatValue（bytes/chars），total=0 显示 ∞
- 代理路由 SSRF 黑名单（isPrivateHost）
- RemoteDownloadDialog fire-and-forget（v8.6 改为任务队列）

---

## v8.4 回顾：Per-user 加密 vault

```
K  = 随机 256-bit AES-GCM 密钥（加密本地 library/settings）
KE = PBKDF2(密码, salt) 派生密钥（加密 K）
K_enc = encryptToEnvelope(K, KE) → 存服务端 User.encryptedVaultKey
```

关键文件：vaultKey.ts / VaultContext.tsx / vaultState.ts / libraryService.ts / settingsService.ts

---

## 绝对红线

- 不要动 useQuotaStats.ts 用 Supabase User 上不存在的字段（CI 必挂）
- handleLogout 是 async
- useBooksSync 的 prevUserIdRef/replaceModeRef/didInitialPushRef 不能删
- vaultState.ts 不能删
- VaultContext 必须在 AuthProvider 内
- 代理路由已移除白名单（SSRF 黑名单）
- RemoteDownloadDialog 用 POST 创建 task + eventDispatcher.dispatch('refresh-library')（v8.7 起，不用 transferStore）
- 分享有效期：0=永久，1-365=正常
- `git -C` 确保在正确目录操作，不要把 workspace/ 提交到 readest-lite
- husky hooks 用 `-c core.hooksPath=/dev/null` 跳过

---

## CI 失败排查

1. TS 未使用变量/import → 删除（v8.7.0 踩过 `ALLOWED_EXTENSIONS` / `IoAlertCircleOutline`）
2. `eventDispatcher.off(name, cb)` 必须传 2 个参数（v8.7.0 踩过）—— 如需 inline cb 请先抽成 `useCallback`
3. useQuotaStats 不要用 user?.storageQuotaMB
4. handleLogout async 必须 await
5. VaultContext 在 AuthProvider 内
6. useUserActions 不 import useEnv/envConfig
7. 代理路由不加回 ALLOWED_HOSTS
8. RemoteDownloadDialog 不调 useBooksSync（用 eventDispatcher）
9. **git -C 确保正确目录**，不要把根仓库文件提交到 readest-lite
10. **上传大文件必须分块**（v8.8）—— 不要走整文件 PUT，CF 100s 超时会返 524；块大小 5MB，index=0 时服务端自动清理上次失败残留的 parts
11. **`noUncheckedIndexedAccess`**（v8.9 踩过）—— `@sindresorhus/tsconfig` 启用此项，所有 `array[N]` 返回 `T | undefined`，必须加 `&& arr[N]` 守卫或用 `?.` 链。`regex.match(...)[N]` 和 `str.split(...)[N]` 都要守卫。
12. **React 组件父传函数 prop**（v8.9 踩过）—— 父组件每次重渲染会传入新函数引用，如果该函数在子组件 useEffect deps 里，会导致无限循环。解法：子组件用 `useRef` 存最新函数，effect 只依赖真正的状态变量。
13. **下载任务表格 schema 变更必须考虑 prisma db push**（v8.9）—— 新增字段默认值要能向后兼容现有 row（用 `@default(0)` 或 `?`）。`onDelete: Cascade` 确保删 task 自动删关联的 DownloadLog。
14. **foliate-js 类型声明**（v8.13 踩过）—— Lite 的 Docker 构建 `git clone` foliate-js 到 packages/，上游用 git submodule。TypeScript 在 `moduleResolution:bundler` 下无法推断 `foliate-js/tts.js` 的动态 import 类型。必须新增 `src/foliate-js.d.ts` 声明 `getSentences`/`SentenceEntry`/`TTS`。TTS 类声明为 `any` 类型匹配上游 allowJs 的宽松推断（显式声明会因 strict mode 严格检查导致 `getCFI` 参数类型不匹配）。
15. **package.json 版本号**（v8.13.1 踩过）—— `getAppVersion()` 读 `apps/readest-app/package.json` 的 `version` 字段。上游版本是 `0.11.x`，Lite 每次发版必须把 version 改成 Lite 真实版本（如 `8.13.1`），否则「关于」页面显示上游版本号。
16. **非覆盖式上游合入**（v8.13 总结）—— 合入上游 PR 时逐 PR 研究、逐文件改，不粗暴复制。关键：判断每个文件是否 Lite 自定义（对比 v0.11.17 字节一致 = 安全合入；Lite 有改动 = 仅增量修改）。foliate-js submodule 的改动在 Docker 构建时 `git clone` 自动获取，无需手动管理 submodule pointer。

---

## 主页与部署教程

- 主页：`https://cshdotcom.github.io/readestl/`
- 部署教程：`https://cshdotcom.github.io/readestl/deploy.html`
- 迭代提示词：`https://cshdotcom.github.io/readestl/aph.html`
- 安卓源码：`https://github.com/cshdotcom/Readest-lite-Android`

---

**版本**：v8.13.1
**最后更新**：2026-07-09
**适用 commit**：`d584c8e` 及之后（v8.13.1 翻页动画 + 关于版本号修复 + bookService 崩溃修复）
**CI 状态**：✅ Docker Image + CI smoke test success
**镜像**：`ghcr.io/cshdotcom/readest-lite:8.13.1` / `8.13` / `latest`
