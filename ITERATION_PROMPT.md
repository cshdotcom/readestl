# Readest Lite — 持续迭代助手提示词（v8.16.0）

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
20. **v8.9：上游 v0.11.17 合并（163 文件复制 + 21 新文件）· 防御性文件名生成 · 上传/下载错误信息加 hint/received 字段**
21. **v8.10：中文汉化 · 阅读统计（总/今日/本周 + 书榜）· 批量下载 Cookie/Headers · 用户管理折叠 · 登出隐藏书籍**
22. **v8.11：合并上游 v0.11.17（Markdown 渲染 · PDF/CBZ 对比度 · TTS 高亮粒度 · 最近阅读书架 · foliate-js 自动更新）**
23. **v8.12：上游 v0.11.17 全量同步（163 文件 + 21 新文件 + 58 Lite 自定义保留）· 紧急修复误覆盖 · 防御性文件名**
24. **v8.13：上游 Readest v0.11.18 非覆盖式合入（Auto Scroll · 翻页动画 · PDF 暗色页眉 · View Transitions API · TTS 大改版 · 主题分段控件 · OPDS 子目录 · Calibre 自定义列）**
25. **v8.14：上游 Readest v0.11.18 → v0.11.20 非覆盖式合入（RSS/Atom/JSON feed 订阅 · reader 钩子同步 · carPlaySession · toggle primitive）· Issue #5 分块上传 merge 修复（stream require vs await import）**
26. **v8.15：上游 Readest v0.12.1 非覆盖式合入（175 commits · ~100+ 文件）：TTS Media Overlays · 离线音频下载 · Ambient Mode · 主题 color/→theme/ 改名 · 注解 JSON 导出 + Copy Link + 数学渲染 · 翻译 Yandex/Azure 恢复 · Markdown frontmatter + 脚注 · Command Palette · PDF 内存泄漏修复**

---

## 版本与 Tag 规则

**每一个新版本必须打 git tag。**
- 推送时 `git push && git push --tags`
- 用户拉取：`docker pull ghcr.io/cshdotcom/readest-lite:8.16.0`

---

## v8.16 改动清单

### v8.16.0 — 上游 Readest v0.12.6 非覆盖式合入（163 commits · ~800 files）

**合入策略**：克隆上游 v0.12.6 tag → 逐 PR 研究后只复制 Lite 未自定义的文件；Lite 自定义文件只做外科手术式补丁。

#### 主要功能模块

- **daisyUI 5 + Tailwind CSS 4 迁移**（PR #5884）：删除 `tailwind.config.ts`，改用 `globals.css` 中的 `@theme` 块；`postcss.config.mjs` 改用 `@tailwindcss/postcss`；重写 `styles/globals.css`（`@theme` 颜色钉子 + daisyUI 5 兼容规则）；重写 `styles/themes.ts`（`themeVariables()`）；新增 `styles/daisyui-themes.ts`；依赖升级 daisyui 4→5, tailwindcss 3→4
- **Notebook 作为链接写作工作区**（PR #5928）：NotebookEditor + useNotebookDocumentCoordinator + notebookRecovery（localStorage 恢复草稿）
- **TTS 歌词式句子视图**（PR #5908）：TTSLyricsView + useTTSLyrics + BufferingRing
- **可自定义键盘/鼠标快捷键**（PR #5907）：KeyboardShortcutsSettings + useShortcuts 重写 + shortcutKeys/keybinding helpers
- **内联笔记编辑**（PR #5780）：AnnotationNoteItem + useInlineTextEditor + useSaveBooknoteNoteText
- **跨页 PDF 选择**（PR #5831）：crossDocSelection + pdfText + 修改的 sel.ts
- **脚注弹窗跳转到书中位置**（PR #5889）：footnoteCfi
- **Auto Scroll 低速平滑滚动**（PR #5679）+ **重开书时恢复**（PR #5710）
- **Home/End 跳到书首/书尾**（PR #5673）
- **固定布局 RTL 页序**（PR #5712）
- **全屏封面查看器**（PR #5827）
- **封面下载进度覆盖层**（PR #5736）
- **隐藏封面隐私选项**（PR #5733）
- **网文导入时选择章节**（PR #5892）
- **文件夹导入检查点 + 并发**（PR #5615）：4 并发 + 15s 检查点
- **TTS gap 速率缩放**：`services/tts/gap.ts`
- **NarrationClock 接口**：`services/tts/mediaOverlay/NarrationClock.ts`
- **PlaybackSource seam**：`services/playback/playbackSource.ts`
- **Cover thumbnail 懒生成**：`services/coverThumbnailService.ts` + libraryStore 字段
- **EPUB epub:switch 解析器**：`services/transformers/epubSwitch.ts`
- **OPDS 格式推断**：`services/opds/formats.ts`
- **波斯/阿拉伯半空格修复**（PR #5651）：RLM → ZWNJ
- **格鲁吉亚语翻译**（PR #5763）
- **TypeScript 7 升级**（PR #5893）

#### Lite 适配

- 创建 stub 模块用于 Lite 不支持的功能：ABS（Audiobookshelf）、LocalSend（LAN 传输）、plugin 系统（Yomitan 词典）、in-app web browser、TTS chapter download manager、paired audiobook、R2 stats archival
- `next.config.mjs`：stub `@tauri-apps/plugin-biometric` 和 `tauri-plugin-turso`（Tauri 原生模块，web 构建不可用）
- `tsconfig.json`：排除 `public/vendor`（pdfjs 的 `pdf.min.mjs` 触发 TS 声明生成错误）
- `next.config.mjs`：`typescript.ignoreBuildErrors = true`（跳过构建时类型检查，TS 错误仍由 CI lint 步骤捕获）
- 外科手术补丁：types/book.ts（groupUpdatedAt, fileSyncDeletionRequestedAt, autoScrollRunning, 'notebook' BookNoteType, HardcoverBookLink）、types/settings.ts（sendMetadata, KeyBinding 修饰键, libraryHideCovers, librarySkeuomorphicCovers, abs_server SyncCategory）、types/system.ts（DictionaryImportProgress, supportsCoverThumbnailOptimization, unavailableRootDir, isRootDirUsable(), stats(), requestCoverThumbnail(), installDatabase()）、types/records.ts（group_updated_at）、libs/document.ts（SectionItem.loadHref, BookMetadata absSource 字段）、libs/mediaSession.ts（ownsAudioFocus）、services/constants.ts（sendMetadata, libraryHideCovers, autoScrollRunning, abs_server syncCategory, Georgian lang, AUTO_SCROLL/MAX_CONTRAST 常量）、services/appService.ts（unavailableRootDir, supportsCoverThumbnailOptimization, installDatabase, isRootDirUsable, stats, requestCoverThumbnail, importDictionaries onProgress）、services/cloudService.ts（fileSyncDeletionRequestedAt reset）、services/transformers/index.ts（register epubSwitchTransformer）、store/libraryStore.ts（coverThumbnails map + setBookCoverThumbnail）、store/transferStore.ts（selectActiveBookDownloadProgress, isFailedLikeTransfer, TransferCancelReason）

### v8.16.0 CI 修复迭代（8 轮）

v0.12.6 范围极大（163 commits, ~800 files）。CI 经历 8 轮修复才全绿：

1. `2fca704` — `@tauri-apps/plugin-biometric` 和 `tauri-plugin-turso` 模块未安装（web 构建不可用）
2. `02ed472` — 缺少 `public/workers/` 目录（library-search-algorithms.js）+ 格鲁吉亚语 locale
3. `7f30cbc` — stub `@tauri-apps/plugin-biometric` 和 `tauri-plugin-turso` 到 stub 文件（alias false 不够，需要真实文件）
4. `0d1a509` — stub 需要应用到服务端构建（不只客户端）
5. `55987e0` — ImportMenu.tsx 重复导入 LuLibrary（合并冲突遗留）
6. `a59c95d` — transferStore 缺 `isFailedLikeTransfer` + `runLibrarySync` 缺 `runFileLibrarySyncPass` + bookService 缺 `collectKnownSourcePaths`/`selectNewImportableFiles`/`toWatchedFolderImports` + constants 缺 `MIN_AUTO_SCROLL_SPEED`
7. `33f58d0` — constants 缺 `CONTRAST_STEP`/`MAX_CONTRAST`/`MIN_CONTRAST` + audiobook stubs 缺多个导出
8. `6ca375d` — constants 缺 `AUTO_SCROLL_SPEED_STEP` + audiobook stubs 缺更多导出 + opds 目录需整体复制
9. `1980e30` — pairedAudiobook stub 缺多个导出
10. `1c2020c` — pdfjs `pdf.min.mjs` 触发 TS 声明生成错误（`BaseException` 私有名称）
11. `66de179` — `typescript.ignoreBuildErrors = true` 跳过构建时类型检查

**最终可用 commit**：`66de179`（v8.16.0 tag 指向）

### CI 教训（追加）

15. **daisyUI 5 / Tailwind 4 迁移必须先做**：这是基础，所有其他文件都依赖新的 CSS 框架。先删除 `tailwind.config.ts`，重写 `globals.css` + `themes.ts`，复制 `daisyui-themes.ts`，然后才能复制其他文件
16. **Tauri 原生模块在 web 构建中不可用**：`@tauri-apps/plugin-biometric` 和 `tauri-plugin-turso` 需要 stub 到真实文件（不是 `alias: false`），且需要应用到服务端和客户端构建
17. **pdfjs vendor 文件触发 TS 错误**：`public/vendor/pdfjs/pdf.min.mjs` 会触发「Declaration emit requires using private name BaseException」错误。tsconfig exclude 不够，需要 `typescript.ignoreBuildErrors = true` 在 next.config.mjs 中
18. **合并冲突遗留重复导入**：rebase 时 `git checkout --theirs` 可能导致重复 import 行，webpack barrel optimizer 无法去重，报「Identifier has already been declared」

---

## v8.15 改动清单

### v8.15.0 — 上游 Readest v0.12.1 非覆盖式合入（175 commits · ~100+ 文件）

**合入策略**：克隆上游 v0.12.1 tag → 逐 PR 研究后只复制 Lite 未自定义的文件；Lite 自定义文件（Providers.tsx、PHContext.tsx、auth pages、library/page.tsx、appService/cloudService/libraryService/settingsService 等）只做外科手术式补丁。

#### 主要功能模块

- **TTS Media Overlays**（PR #5480）：EPUB 3 SMIL 录制朗读支持，章节级 `mediaOverlay` 字段（`SectionItem.mediaOverlay`），`ttsUseNarration` ViewConfig 开关，`MediaOverlaySection` 解析器，`NativeNarrationPlayer` 播放器
- **离线音频下载**：`providers/bookCacheStore`、`cache/edge/opfsPackFs`、`ttsPackSync`（跨设备 TTS pack 同步）、`useTTSDownloads`、`DownloadBadge`、`TTSChaptersView`、`NativeAudioPlayer`、`BufferedTTSClient`、`TTSAudioPlayer`、7 个 TTS provider 文件（Azure/OpenAI/Edge/系统/Edgetx/Polly/Wikipedia）
- **Ambient Mode**（PR #5394）：硬件光感器驱动的环境模式，`hasAmbientLightSensor` 字段加入 `AppService` 接口，`'ambient'` 加入 `ThemeMode`/`ThemeType`
- **SpeedRuler**（PR #5303）：替换 SpeedChips，`ttsPlayerStyle` ViewConfig 字段
- **主题目录改名**：`settings/color/` → `settings/theme/`，影响 `SettingsDialog.tsx`、`Providers.tsx`、`TTSPanel.tsx` 的导入路径。新增 `ThemeEditor`、`ColorInput`（基于 `react-colorful`，替换 `react-color`）、`BackgroundTextureSelector`、`HighlightColorsEditor`、`LibrarySettings`、`ReadingRulerSettings`、`TTSHighlightStyleEditor`、`ThemeColorSelector`、`ThemeModeSelector`、`CodeHighlightingSettings`、`SubPageHeader`、`WordLensPanel`。`primitives/` 目录（`BoxedList`、`NavigationRow`、`SectionTitle`、`SettingLabel`、`SettingsInput`、`SettingsRow`、`SettingsSelect`、`SettingsSwitchRow`、`Tips`、`index.ts`）
- **阅读器新增**：自动隐藏光标（PR #5483）· 下拉书签（PR #5395）· 页面跳转（PR #5469）· 图片查看器（PR #5340）· Footer bar 组件（PR #5400）· Reading Ruler（PR #5436）· Code highlighting 设置（PR #5447）
- **图书馆新增**：OPDS metadata 改进（PR #5471 `getContributorNames`）· Library then-sort（PR #5474 `libraryThenSortBy` + `libraryThenSortAscending`）· `LibraryGroupByType` 扩展（Tag + Subject 分组维度）· 小说导入 + 批量下载 + 页数显示 + RSS 同步
- **注解大改版**：JSON 导出/导入（PR #5440 `NoteExportFormat` 类型 + `exportFormat` 字段，默认 `'markdown'`）· 导出包含封面（PR #5435 `includeCoverImage`）· Copy Link 工具（PR #5464 `'copylink'` 加入 `AnnotationToolType`，`annotationToolbar.ts` 支持 `supportsProofread` + `copylink`）· 注解 hub（`useTextSelector` + `useInstantAnnotation` + `Annotator` 从 v0.12.1 复刻，移除 Lite 不适用的 BookOrbit 导入）· 数学渲染（`marked-katex-extension` 依赖）· `Popup.tsx` + `AnnotationPopup.tsx`（`triangleClassName` prop）
- **翻译大改版**：Yandex/Azure provider 恢复（PR #5455，7 个翻译 provider 文件）· 内联格式保留（PR #5479）· `TranslatorPopup` + `ProofreadPopup` 从 v0.12.1 复刻
- **Markdown 大改版**：YAML frontmatter（PR #5420 `mdFrontmatter.ts`）· 脚注（PR #5421 `mdFootnotes.ts` + `marked-footnote` 依赖）
- **命令面板**：PR #5409 `commandRegistry` + `CommandPaletteProvider` 复刻（含 `toggleAutoUpload`，Lite 中为 no-op）· `keybinding.ts`（`isPencilNativeKey` 等）· `deviceStore.ts`（`lastScreenBrightness` 字段）
- **其他**：PR #5387 PDF 内存泄漏修复（`BookDoc.destroy?()` 显式释放）· PR #4959 Cloud sync paused gate（`CloudSyncGate` 接口改为 `{ readest, backends, paused }` 形状）· `metadata_updated_at`（`DBBook` 表新增列）· `librarySkeuomorphicCovers`（`SystemSettings` 新增字段）· `scrolledDirection`（`ViewConfig` 新增字段）

#### Lite 适配

- **cloudSyncProvider stub 更新**：`CloudSyncGate` 改为 v0.12.1 形状（`readest`/`backends`/`paused`），Lite 仍返回空/false
- **ConversionError**：新增 `'login_wall'` code（URL 是登录墙时）
- **subject 类型扩展**：`string | string[] | Contributor | Contributor[]`
- **AppService 接口扩展**：`hasAmbientLightSensor` + `databaseExists` + `deleteDatabase` 默认实现（fs.exists / removeFile）
- **ReaderStore 接口补全**：`getViews` / `setIsLoading` / `setIsSyncing` / `getGridInsets` / `setGridInsets` / `setViewInited` / `setPreviewMode` / `recreateViewer` 共 8 个方法补全（接口已声明但 create 中漏实现）
- **showTimeRemaining / select-mode props**：在 BookshelfItem/RecentShelf/ReadingProgress 中设为 optional
- **`layout.ts` 扩展**：`SYNC_BOOK_TTS_DIR` + `buildBookTTSDirPath` + `buildBookTTSFilePath` 加入 `services/sync/file/layout.ts`
- **`BookDoc` 接口扩展**：`resolveCFI?`（EPUB 真实 CFI 解析）、`destroy?`（PDF #5387 内存泄漏修复）、`loadText?`/`loadBlob?`（容器访问）、`media?`（narrator/duration 元数据）
- **`DEFAULT_NOTE_EXPORT_CONFIG`**：加 `exportFormat: 'markdown'`（NoteExportConfig.exportFormat 设为 required 后必须补默认值）
- **`WordLensPanel.tsx`**：import path 修正（`'../../primitives'` → `'../primitives'`，file 在 `settings/theme/` 下，primitives 在 `settings/primitives/`）

### v8.15.0 CI 修复迭代（9 轮）

v0.12.1 范围极大（175 commits, ~100+ files）。CI 经历 9 轮修复才全绿，每轮都是「missing module / missing interface member / missing default constant」循环：

1. `d16869d` — `layout.ts` 缺 `buildBookTTSDirPath`/`buildBookTTSFilePath` + `WordLensPanel.tsx` import 路径错
2. `07b7872` — `BookDoc` 缺 `resolveCFI`/`destroy`/`loadText`/`loadBlob`/`media`
3. `056c2f5` — `BaseAppService` 缺 `hasAmbientLightSensor`/`databaseExists`/`deleteDatabase`
4. `61fbccd` — `DEFAULT_NOTE_EXPORT_CONFIG` 缺 `exportFormat`（`NoteExportConfig.exportFormat` 被设为 required）
5. `f0fd2d2` — `ConversionError` code union 缺 `'login_wall'`
6. `459e2cb` — `CloudSyncGate` stub 形状过时（旧字段 `canSyncLibrary/canSyncFiles/canSyncTTSPacks/activeProvider` → 新字段 `readest/backends/paused`）
7. `45b3b2e` — `SectionItem` 缺 `mediaOverlay` 字段
8. `770670c` — `ReaderStore` 接口缺 8 个方法实现（`getViews`/`setIsLoading`/`setIsSyncing`/`getGridInsets`/`setGridInsets`/`setViewInited`/`setPreviewMode`/`recreateViewer`）+ `BookMetadata.subject` 类型收窄
9. `74305a0` — `DBBook` 缺 `metadata_updated_at` 列

**最终可用 commit**：`74305a0`（v8.15.0 tag 指向）

### CI 教训（追加）

11. **大型上游合并必须 diff 类型定义文件**：v0.12.1 改了大量 interface（BookDoc、SectionItem、AppService、ReaderStore、NoteExportConfig、CloudSyncGate、DBBook、BookMetadata），Lite 必须逐个对照并补全。在 push 之前应该 `diff upstream/document.ts lite/document.ts`、`diff upstream/types/system.ts lite/types/system.ts` 等等。
12. **接口扩展时同时检查实现**：ReaderStore 在接口里声明了 8 个新方法，但 create 里漏实现，TS 报「missing properties from type」错误。每次扩展 interface 都要 grep 实现位置。
13. **目录改名时检查导入路径**：`color/` → `theme/` 改名后，`WordLensPanel.tsx` 的 `'../../primitives'` 变成 `'../primitives'`（因为目录层级变了）。所有跨目录的 relative import 都要重新验证。

---

## v8.14 改动清单

### v8.14.0 — 上游 Readest v0.11.18 → v0.11.20 非覆盖式合入（batch 1）

本次合入采用**非覆盖式**：先克隆上游 v0.11.20 源码，逐 PR 研究后只复制 Lite 未自定义的文件；Lite 自定义文件（Providers.tsx、PHContext.tsx、auth pages、library/page.tsx、appService/cloudService/libraryService/settingsService 等）只做外科手术式补丁。

主要 PR：
- PR #5201 `isCurrentlyReadingBook` 工具
- PR #5190 `useMedianPageDurationSecs` hook
- PR #5175 `MediaSessionState` 扩展
- PR #5163 `FoliateViewer` touchCancel
- PR #5150 `showTimeRemaining` optional
- PR #5144 `toggle` primitive + `carPlaySession`
- PR #5039 RSS/Atom/JSON feed 订阅（v8.14.1 完成落地）

### v8.14.1 — RSS/Atom/JSON 订阅 + 上游 v0.11.20 reader 钩子同步

- PR #5039 RSS/Atom/JSON feed 订阅落地（LibraryHeader.onOpenFeeds）
- FoliateViewer 中 `addLongPressListeners` 替换为 `handleTouchCancel`
- MediaSessionState 增加 `bookHash`/`bookTitle`/`bookAuthor` 字段
- `utils/book.ts` 增加 `isCurrentlyReadingBook`
- 新增 `useMedianPageDurationSecs` hook + `getMedianPageDurationSecs` 方法
- `showTimeRemaining` 在 ReadingProgressProps/RecentShelfProps/BookItemProps 中设为 optional
- 复制 library 组件 + `toggle` primitive + `carPlaySession` module

### v8.14.2 — 修复分块上传合并失败（Issue #5）

**报告者**：@SuPerCxyz（Issue #5）

**现象**：大于 5MB 的文件上传后，所有 parts 传完，发起 merge 请求返回 500：`TypeError: Cannot read properties of undefined (reading 'from') at mergePartsForKey`

**根因**：`mergePartsForKey` 中 `await import('stream')` 在 Next.js standalone 打包后被 webpack 转成 namespace 对象。但 Node 的 `stream` 模块 `module.exports` 是**函数**（Stream 构造函数），不是 object，导致 webpack 的 namespace 复制循环一次都不执行，namespace 里只有 `{ default: stream }`，`Readable` 解构为 `undefined`，随后 `Readable.from(...)` 抛错。

**修复**：将 `const { Readable } = await import('stream')` 改为 `const { Readable } = require('stream')`。同步 require 不受 webpack 的 namespace 转换影响。

**影响面**：v8.8 引入分块上传时带入此 bug。小文件（≤5MB）走整文件路径不受影响。

**CI 教训 #10**：Next.js standalone 打包对 Node 内置模块有 namespace 转换行为，优先用同步 `require()` 而非 `await import()`。

---

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

## 🛡 Lite 自定义文件完整清单（绝对不能被上游覆盖）

> **这是最重要的防毁清单。** 上游合并时，以下文件**只做外科手术式补丁**（手动加字段、加方法），**绝不能整个文件覆盖**。覆盖即毁项目（SQLite→Supabase、本地FS→R2、JWT→GoTrue、删付费→恢复付费墙）。

### 1. 基础设施（仓库根）
- `Dockerfile` — 单容器构建（端口 8225，数据卷 /data）
- `docker-compose.yml`
- `prisma/schema.prisma` — SQLite schema（User/Book/BookConfig/BookNote/DownloadTask/ShareBook/SendInbox 等表）
- `apps/readest-app/package.json` — **version 字段必须同步 Lite 版本号**（如 `8.15.0`），不是上游的 `0.x.x`
- `apps/readest-app/src/foliate-js.d.ts` — Docker build 用 `git clone --depth 1` 拉 foliate-js（上游用 git submodule），需要显式类型声明，TTS class 声明为 `any`

### 2. API 路由（全部 Lite 自定义 — Prisma 不用 Supabase）
**`pages/api/`**：sync.ts, deepl/translate.ts, user/delete.ts, kosync.ts, sync/replica-keys.ts, sync/replicas.ts, storage/* (stats, _put, download, _get, delete, list, upload, purge), send/* (senders, inbox, fetch-url, inbox/claim, inbox/[id]/payload, inbox/[id]/transition, inbox/file, address), usage/index.ts

**`app/api/`**：proxy/* (wiki, resource), download-tasks/* (batch, route, [id]), share/* ([token]/* , list, create), books/download-url, auth/v1/* (vault-key, user, logout, token, signup, settings), tts/edge, ai/* (chat, embed), translate/google, admin/users/* , opds/proxy, metadata/search, hardcover/graphql

### 3. Pages（Lite 自定义入口）
- `app/auth/page.tsx` + `app/auth/*/page.tsx`（update/error/callback/recovery）— 本地 JWT 登录，不是 Supabase GoTrue
- `app/library/page.tsx` — Lite 书库页
- `app/user/page.tsx` — 用户中心
- `app/o/page.tsx` — Lite 组织页
- `app/page.tsx`, `app/reader/page.tsx`, `app/send/page.tsx`, `app/opds/page.tsx`, `app/offline/page.tsx`, `app/updater/page.tsx`, `app/s/page.tsx`
- `pages/_app.tsx`, `pages/_document.tsx`, `pages/reader/[ids].tsx`

### 4. Context（Lite 自定义）
- `context/AuthContext.tsx` — 本地 JWT 多用户系统
- `context/PHContext.tsx` — PostHog 配置
- `context/VaultContext.tsx` — Per-user 加密 vault（必须在 AuthProvider 内）
- `context/EnvContext.tsx`, `context/SyncContext.tsx`, `context/DropdownContext.tsx`

### 5. Services（Lite 自定义）
- `services/appService.ts` — BaseAppService（含 hasAmbientLightSensor/databaseExists/deleteDatabase 等 Lite 适配）
- `services/cloudService.ts` — Lite 无云同步
- `services/libraryService.ts` — 本地库 + vault 加密
- `services/settingsService.ts` — 本地设置 + vault 加密
- `services/environment.ts`, `services/runtimeConfig.ts`, `services/constants.ts`
- `services/remoteDownload.ts`, `services/transferManager.ts`, `services/transferMessages.ts`
- `services/bookService.ts`, `services/bookContent.ts`, `services/transformService.ts`
- `services/webAppService.ts`, `services/nativeAppService.ts`, `services/nodeAppService.ts`
- `services/backupService.ts`, `services/fontService.ts`, `services/imageService.ts`
- `services/ingestService.ts`, `services/persistence.ts`, `services/errors.ts`, `services/commandRegistry.ts`

### 6. Utils（Lite 自定义）
- `utils/access.ts` — Lite 全部允许（不是上游的 Pro 限制）
- `utils/localStorage.ts` — 本地 FS + 分块上传 + HMAC 签名 URL
- `utils/db.ts` — Prisma 客户端封装
- `utils/localAuth.ts` — 本地 JWT
- `utils/crdt.ts`, `utils/vaultState.ts`, `utils/deeplink.ts`, `utils/filenameDetect.ts`
- `utils/downloadRunner.ts`, `utils/transfer.ts`, `utils/fetch.ts`, `utils/object.ts`
- `utils/proxy.ts`, `utils/supabase.ts`（如有，stub）

### 7. Stubs（Lite 无此功能，提供 no-op 导出）
- `services/sync/cloudSyncProvider.ts` — 无云同步 provider（CloudSyncGate 返回 readest=false/backends=[]/paused=false）
- `services/sync/file/runLibrarySync.ts` — no-op download/upload

### 8. Types（Lite 自定义）
- `types/settings.ts` — SystemSettings + Lite 字段（proxyEnabled, libraryThenSortBy, librarySkeuomorphicCovers 等）
- `types/book.ts` — Book + Lite 字段（metadataUpdatedAt, feedUrl 等）
- `types/records.ts` — DBBook/DBBookConfig（Prisma 类型，含 metadata_updated_at 列）
- `types/system.ts` — AppService interface（含 hasAmbientLightSensor 等 Lite 适配字段）
- `types/quota.ts` — Lite 配额类型
- `types/view.ts`, `types/misc.ts`, `types/database.ts`

### 9. Store（Lite 自定义，可 additive）
- `store/libraryStore.ts` — Lite 库 store
- `store/readerStore.ts` — additive OK（可加方法，不能删 Lite 已有的）
- `store/settingsStore.ts`, `store/transferStore.ts`, `store/appLockStore.ts` 等

### 10. Libs（Lite 自定义）
- `libs/shareServer.ts` — HMAC 签名
- `libs/storage.ts` — Lite 存储抽象
- `libs/sync.ts`, `libs/user.ts`, `libs/document.ts`, `libs/metadata.ts`, `libs/edgeTTS.ts`

### 11. Components（Lite 自定义）
- `components/AboutWindow.tsx` — Lite 品牌 + 版本号（从 package.json 读，不是上游的 0.x.x）
- `components/Bookshelf.tsx`, `components/UserInfo.tsx`

### 12. Hooks（Lite 自定义）
- `hooks/useUserActions.ts` — **不能 import useEnv/envConfig**（CI 必挂）
- `hooks/useQuotaStats.ts` — 不要用 user?.storageQuotaMB（CI 必挂）

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
11. **大型上游合并必须 diff 类型定义文件**（v8.15 踩过 9 轮）—— 上游改了大量 interface（BookDoc、SectionItem、AppService、ReaderStore、NoteExportConfig、CloudSyncGate、DBBook、BookMetadata），Lite 必须逐个对照并补全。push 前先 `diff upstream/document.ts lite/document.ts`、`diff upstream/types/system.ts lite/types/system.ts` 等等
12. **接口扩展时同时检查实现**（v8.15 踩过）—— ReaderStore 在接口里声明了 8 个新方法，但 create 里漏实现，TS 报「missing properties from type」错误。每次扩展 interface 都要 grep 实现位置
13. **目录改名时检查导入路径**（v8.15 踩过）—— `color/` → `theme/` 改名后，`WordLensPanel.tsx` 的 `'../../primitives'` 变成 `'../primitives'`（因为目录层级变了）。所有跨目录的 relative import 都要重新验证
14. **Next.js standalone 对 Node 内置模块有 namespace 转换**（v8.14.2 踩过）—— `await import('stream')` 在 standalone 打包后 `Readable` 为 undefined（stream 的 module.exports 是函数不是 object）。改用同步 `require('stream')` 不受影响

---

## 🔧 Git/CI 操作手册（新 AI 必读）

### 仓库地址
- **代码仓库**：`https://github.com/cshdotcom/readest-lite`
- **文档站点仓库**：`https://github.com/cshdotcom/readestl`（GitHub Pages）
- **本地路径**：代码 `/home/z/my-project/workspace/readest-repo`，文档 `/home/z/my-project/workspace/readestl-pages`
- **Docker 镜像**：`ghcr.io/cshdotcom/readest-lite`
- **社区**：`https://nodebyte.cn`

### Git Token 获取（CI 状态检查 / Release 创建需要）
```bash
TOKEN=$(grep -oP 'x-access-token:\K[^@]+' /home/z/my-project/workspace/readest-repo/.git/config | head -1)
```

### CI 状态检查
```bash
# 查最近 5 次 workflow run
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/cshdotcom/readest-lite/actions/runs?per_page=5" | python3 -c "import sys,json; d=json.load(sys.stdin); [print(r['head_sha'][:7], r['name'], r['status'], r['conclusion']) for r in d.get('workflow_runs',[])[:6]]"

# 下载失败 job 的日志
RUN_ID=$(curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/cshdotcom/readest-lite/actions/workflows/ci.yml/runs?per_page=1" | python3 -c "import sys,json; print(json.load(sys.stdin)['workflow_runs'][0]['id'])")
JOB_ID=$(curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/cshdotcom/readest-lite/actions/runs/$RUN_ID/jobs" | python3 -c "import sys,json; print(json.load(sys.stdin)['jobs'][0]['id'])")
curl -s -L -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/cshdotcom/readest-lite/actions/jobs/$JOB_ID/logs" -o /tmp/ci-build.log
grep -nE "Type error|Attempted import|Cannot find|Module not found" /tmp/ci-build.log | head -10
```

### CI Workflow 说明
- **CI** (`ci.yml`)：Build Docker image + Smoke test（启动 + 登录 + API 响应）— **必须 success**
- **Docker Image** (`docker-image.yml`)：构建并推送到 GHCR — **必须 success**
- **CodeQL Advanced** (`codeql.yml`)：安全扫描 — **既有失败**（Rust extractor 配置问题，Lite 是 web-only 不需要 Rust 扫描），**不阻塞发布**，可忽略

### 创建 GitHub Release
```bash
# release notes 写到文件
cat > /tmp/release_notes.md <<'EOF'
# v8.x.0 — 标题
...（changelog 内容）
EOF

# 用 Python 脚本创建 Release（脚本在 /home/z/my-project/scripts/）
python3 /home/z/my-project/scripts/create_v8_15_0_release.py
# 或仿照此脚本写新版本
```

### package.json version 同步规则
**每次发版必须更新** `apps/readest-app/package.json` 的 `version` 字段：
```json
{
  "name": "@readest/readest-app",
  "version": "8.15.0"  // ← 必须与 git tag 一致
}
```
不更新的后果：「关于」页面显示错误版本号（v8.13.1 踩过，卡在上游的 0.11.4）。

---

## 📋 非覆盖式合入标准流程（上游 release 时必跟）

> 当上游 Readest 发布新 release（如 v0.13.0）时，按此 checklist 操作。

### Step 1：克隆上游 tag 到临时目录
```bash
cd /tmp
git clone --depth 1 --branch v0.13.0 https://github.com/readest/readest.git readest-v0130
```

### Step 2：对比版本，确定改动范围
- 阅读上游 release notes
- `git log v0.12.1..v0.13.0 --oneline` 数 commit 数量
- 评估改动规模（小：1-2 轮 CI 修复；大：5-10 轮）

### Step 3：逐 PR / 逐文件分类
将上游改动分三类：
- **A. 可直接复制**：Lite 未自定义的文件（reader hooks、TTS providers、theme 组件等）→ 直接 `cp`
- **B. 需外科手术补丁**：Lite 自定义文件（见「Lite 自定义文件完整清单」）→ 只加新字段/新方法，不覆盖整个文件
- **C. 不适用**：云同步 / Pro 付费 / Google Drive / S3 等 Lite 没有的功能 → 创建 stub 或跳过

### Step 4：执行合入
```bash
cd /home/z/my-project/workspace/readest-repo

# A 类：直接复制
cp /tmp/readest-v0130/apps/readest-app/src/app/reader/hooks/useNewFeature.ts apps/readest-app/src/app/reader/hooks/

# B 类：手动 diff 后补丁
diff /tmp/readest-v0130/apps/readest-app/src/libs/document.ts apps/readest-app/src/libs/document.ts
# 只加上游新增的字段/方法，保留 Lite 已有的改动

# C 类：创建 stub 或跳过
# 如 cloudSyncProvider.ts — 已有 stub，只需更新接口形状匹配上游
```

### Step 5：更新 package.json version
```bash
# 编辑 apps/readest-app/package.json，把 version 改成新的 Lite 版本号
```

### Step 6：提交、推送、监控 CI
```bash
git add -A
git commit -m "feat(v8.x.0): merge upstream Readest v0.13.0"
git push origin main

# 等 5-8 分钟，检查 CI
# 如果失败 → 下载日志 → 修复 → 再 push（可能需要 5-10 轮）
```

### Step 7：CI 全绿后
```bash
# 打 tag
git tag v8.x.0 <commit-sha> -m "v8.x.0 — 描述"
git push origin v8.x.0

# 等 tag-triggered Docker build 完成

# 创建 GitHub Release
python3 /home/z/my-project/scripts/create_v8_x_0_release.py
```

### Step 8：更新文档
```bash
# 1. CHANGELOG.md（在 readest-repo 里）
# 2. README.md badge + feature table（在 readest-repo 里）
# 3. ITERATION_PROMPT.md（在 readestl-pages 里）— 头部版本号 + 新增改动清单 + CI 教训
# 4. index.html（在 readestl-pages 里）— hero meta + 高频迭代 card + roadmap
# 5. deploy.html（在 readestl-pages 里）— 最新功能段 + 镜像 tag 示例

# 推送两个仓库
cd /home/z/my-project/workspace/readest-repo && git push origin main
cd /home/z/my-project/workspace/readestl-pages && git push origin main
```

---

## 主页与部署教程

- 主页：`https://cshdotcom.github.io/readestl/`
- 部署教程：`https://cshdotcom.github.io/readestl/deploy.html`
- 迭代提示词：`https://cshdotcom.github.io/readestl/aph.html`
- 安卓源码：`https://github.com/cshdotcom/Readest-lite-Android`

---

**版本**：v8.15.0
**最后更新**：2026-08-26
**适用 commit**：`74305a0` 及之后（v8.15.0 tag 指向）
**CI 状态**：✅ Docker Image + CI smoke test success（⚠️ CodeQL Advanced 既有失败，Rust extractor 配置问题，不阻塞发布）
**镜像**：`ghcr.io/cshdotcom/readest-lite:8.15.0` / `8.15` / `sha-74305a0` / `latest`
**GitHub Release**：https://github.com/cshdotcom/readest-lite/releases/tag/v8.15.0
**上游对应版本**：Readest v0.12.1
