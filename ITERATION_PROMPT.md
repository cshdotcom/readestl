# Readest Lite — 持续迭代助手提示词（v8.3.0）

> 把这段提示词完整粘贴给后续的 AI 助手。

---

## 项目背景

你正在维护 **Readest Lite**（https://github.com/cshdotcom/readest-lite），这是 Readest 的轻量化自托管分支。

**核心改造原则**（不可回滚）：

1. 数据库：Supabase Postgres → SQLite + Prisma 5.22（14 张表 + User 扩展字段）
2. 文件存储：R2/S3 → 本地文件系统 + HMAC-SHA256 签名 URL
3. 鉴权：Supabase GoTrue → 本地 JWT **多用户**系统，`/api/auth/v1/*` 固定路由
4. Pro/付费体系：**完全删除**
5. 注册：**完全禁用**，管理员通过用户管理面板创建用户
6. 同步/分享/阅读器：**1:1 复刻原版**
7. 单 Docker 镜像，端口 8225，数据卷 `/data`
8. 字体本地化（`public/font/dist/`）
9. DeepL 默认关闭，需 `DEEPL_ENABLED=true`
10. **v8.0：所有翻译/词典代理强制要求登录**
11. COEP 全站 credentialless
12. 远程书籍下载：`/api/books/download-url`
13. **v8.2：代理开关 proxyEnabled**——用户可在设置里关代理走客户端直连
14. **v8.3：账号切换数据隔离**——登出时彻底清空当前账号数据，防止跨账号泄露

---

## 版本与 Tag 规则（v8.1.0 起强制）

**每一个新版本必须打 git tag，防止源码混乱。**

- 版本号格式：`v<major>.<minor>.<patch>`（如 `v8.3.0`、`v8.3.1`、`v8.4.0`）
- 每个 tag 对应一个 commit，commit message 必须含完整改动清单
- 推送时 `git push && git push --tags`
- 主页 / 部署教程 / ITERATION_PROMPT.md 必须同步更新到对应版本号
- Docker Image workflow 会用 `type=semver` 自动 push 3 个 image tag：`<version>`、`<major>.<minor>`、`latest`
- 用户拉取特定版本：`docker pull ghcr.io/cshdotcom/readest-lite:8.3.0`（注意：不带 v 前缀）

---

## v8.3.0 改动清单

### 1. 账号切换数据隔离（v8.3 核心）

**问题**：v8.2 及之前，登出账号 A → 登录账号 B 后，A 的书库、阅读进度、批注、第三方服务凭据（WebDAV/KOSync/Readwise/Hardcover/AI）、App Lock PIN 等全部残留，B 能看到 A 的数据。更严重的是 `handleAutoSync` 会把 A 的本地书 push 到 B 的云端账号，造成数据泄露。

**修复**：

#### 1.1 登出时彻底清理（`hooks/useUserActions.ts`）

`handleLogout` 改为 async，执行以下清理：
- **清空 library**：`libraryStore.setLibrary([])` + `appService.saveLibraryBooks([], { replace: true })`
- **重置 sync cursor**：`lastSyncedAtBooks`/`Configs`/`Notes` = 0（下次登录走全量 pull）
- **清账号绑定的 settings 字段**：
  - `kosync` / `readwise` / `hardcover` / `webdav` / `aiSettings` → 重置为 DEFAULT_*_SETTINGS
  - `pinCodeEnabled` / `pinCodeHash` / `pinCodeSalt` → undefined（清 App Lock PIN）
  - `lastSyncedAtReplicas` → {}（清 replica 同步状态）
- **清 transfer queue**：`transferStore.clearAll()`（避免上个账号的上传/下载任务继续跑给下个账号）
- **最后**：`await logout()`（修了原来没 await 的 bug）+ `navigateToLibrary`

**不删**：Books/ 目录下的 `<hash>/` 文件夹（文件本体保留，重新登录同账号可复用）

#### 1.2 user.id 变化检测 + 全量 pull replace（`app/library/hooks/useBooksSync.ts`）

新增 3 个 ref：
- `prevUserIdRef`：记录上次的 user.id
- `replaceModeRef`：标记下次 `updateLibrary` 走 replace 模式（不 merge）
- `didInitialPushRef`：标记登录后是否已 push 过一次未同步的书

新增 2 个 effect：
- **用户切换检测**：`useEffect([user?.id])`
  - `prevId !== null && currId !== null && prevId !== currId`（账号切换）→ 清 library + 设 replaceMode + 重置 didInitialPush
  - `prevId === null`（未登录→登录或首次安装）→ 不清 library（保留未登录时导入的书）
- **登录后显式 push**：`useEffect([user, useSyncInited, libraryLoaded, lastSyncedAtBooks])`
  - 等 `lastSyncedAtBooks > 0`（pull 完成）后，主动 `pushLibrary()` 一次
  - 解决"未登录时导入书 → 登录后自动同步"的时序问题

#### 1.3 updateLibrary 加 replace 模式（`app/library/hooks/useBooksSync.ts`）

`updateLibrary` 函数开头加 `replaceModeRef.current` 检查：
- `true` → 直接 `setLibrary(syncedBooks)` + `saveLibraryBooks(syncedBooks, { replace: true })`，不走 merge 逻辑
- `false` → 原有 merge 逻辑（processOldBook + 新书追加）

### 2. 数据安全说明

**v8.3 的"清空"是逻辑清空，不是加密清空**：
- `saveLibraryBooks([], { replace: true })` 覆盖 library.json 为空数组 `[]`
- settings 敏感字段重置为默认值（空字符串/undefined）
- 浏览器存储里能找到文件，但内容已清空（不是密文）
- **v8.4 计划**：per-user 数据目录 + AES-GCM 加密，登出时数据加密保留而非清空

### 3. 不做的事（v8.3 范围外）

- ❌ 不按 userId 分文件（`library_<userId>.json`）——工作量大，v8.4 做
- ❌ 不加密 library.json ——v8.4 用 AES-GCM 加密
- ❌ 不删 Books/ 目录下的书籍文件——文件本体保留
- ❌ 不动 `useQuotaStats.ts`——CI 必挂（Supabase User 类型问题）
- ❌ 不动后端 API、Prisma schema、代理路由

---

## v8.4 待办（下次，按账号隔离 + 加密）

- per-user library 文件：`library_<userId>.json`
- per-user settings 文件：`settings_<userId>.json`（账号绑定的字段）
- 登出时数据 AES-GCM 加密保留（不删，用 user.id 派生密钥加密）
- 登录时按 userId 加载对应文件
- 全局共享：UI 偏好（themeMode/customThemes/themeColor 等）

## v8.5+ 待办（写入笔记）

- 配额 enforce（`storageQuotaMB` / `translationQuotaKB` 真正生效）
- 配额 UI 显示真实数据（替换写死的 100TB）——注意不要用 `user?.storageQuotaMB`，因为 useAuth 返回的是 Supabase User 类型没有这字段
- 代理路由移除白名单 + SSRF 黑名单（登录用户全域放开）
- RemoteDownloadDialog fire-and-forget + 任务队列 + 同步 + 重试

---

## 多用户系统（v7.0，仍适用）

### 用户角色
- **admin**：管理员，由 `ADMIN_EMAIL`/`ADMIN_PASSWORD` 环境变量创建
- **user**：普通用户，由管理员通过用户管理面板创建

### User 模型扩展字段（Prisma schema）
```
role             String   @default("user") // "admin" | "user"
displayName      String?
storageQuotaMB   Int      @default(0) // 0=无限（v8.5 计划 enforce）
translationQuotaKB Int    @default(0) // 0=无限（v8.5 计划 enforce）
```

### Admin API 路由
- `GET /api/admin/users` — 列出所有用户（仅 admin）
- `POST /api/admin/users` — 创建用户（仅 admin）
- `PUT /api/admin/users/[id]` — 更新用户（仅 admin）
- `DELETE /api/admin/users/[id]` — 删除用户（仅 admin，不能删自己/最后管理员）

### 前端
- `UserManagement.tsx` 组件在 `app/user/components/` 下
- 仅 `user.userRole === 'admin'` 时显示在用户中心
- 普通用户看不到用户管理面板
- `validateAdmin()` 函数在 `access.ts` 和 `localAuth.ts` 中导出

---

## 绝对红线

### 禁止恢复已删除功能
- Pro/支付/注册代码不能加回来
- 固定路由不能改回 catch-all
- `READEST_WEB_BASE_URL = ''`
- **WebSearchPopup 不能加回来**（X-Frame-Options 拒绝，不可用）

### 禁止修改已对齐的契约
- sync/replicas/replica-key/share/auth 的路径、参数、返回结构不能改
- Prisma schema 不能改（包括 User 扩展字段）

### 禁止清理"奇怪代码"
- 所有 v7.0/v8.0/v8.1/v8.2 的条目仍然适用
- **v8.0：代理路由用强制 auth**（不是可选 auth）
- `validateAdmin` 用于 admin API
- `validateUserAndToken` 从 DB 刷新 role/quotas
- `init-admin.mjs` 设置 `role='admin'`，v8.1.0 起也设 `displayName`

---

## 已知"奇怪"代码（46 项）

1-30: 与 v7.0 相同

31. **v8.0：代理路由用强制 auth**（不是可选 auth；翻译/词典代理均要求登录）
32. 远程书籍下载 `/api/books/download-url` 需要强制 auth
33. ~~WebSearchPopup 用 iframe srcDoc + sandbox~~ → **v8.1.0 已删除**
34. User 模型有 role/displayName/storageQuotaMB/translationQuotaKB 字段
35. `validateAdmin()` 用于 admin API 路由（不是 `validateUserAndToken`）
36. **v8.1.0：远程下载同时写 File 表 + Book 表**（之前只写 File 表，书架看不到）
37. **v8.1.0：`ADMIN_USERNAME` env 设置管理员 displayName**
38. **v8.1.0：每个版本必须打 git tag**
39. **v8.2.0：`utils/proxy.ts` 提供 `isProxyEnabled` / `fetchViaWikiProxy`**——词典/翻译根据 `proxyEnabled` 决定走代理还是直连
40. **v8.2.0：AboutWindow 删除检查更新 + 版权 AGPL**，标题 "Readest Lite" 含 Lite 渐变高亮
41. **v8.2.0：SupportLinks 删除 Discord/Reddit**，加 SVG globe 按钮指向 cshdotcom.github.io/readestl/
42. **v8.2.0：代理路由加 GET health check**（带 hint 字段告诉用户如何 curl 测试）
43. **v8.3.0：登出时彻底清空 library + settings + cursor + transferQueue**——`useUserActions.handleLogout` 改为 async
44. **v8.3.0：useBooksSync 加 prevUserIdRef/replaceModeRef/didInitialPushRef**——检测账号切换 + replace 模式 + 登录后 push
45. **v8.3.0：updateLibrary 加 replace 分支**——用户切换时直接 setLibrary(syncedBooks) 不 merge
46. **v8.3.0：不删 Books/ 目录下的 `<hash>/` 文件**——文件本体保留，重新登录同账号可复用

---

## API 路由清单

| 路由 | 方法 | Auth | 说明 |
|---|---|---|---|
| `/api/auth/v1/token` | POST | 无 | 登录/刷新 |
| `/api/auth/v1/signup` | POST | 无 | 403 禁用 |
| `/api/auth/v1/user` | GET | Bearer | 获取用户 |
| `/api/auth/v1/logout` | POST | 无 | 204 |
| `/api/auth/v1/settings` | GET | 无 | 配置 |
| `/api/sync` | GET/POST | Bearer | 同步（按 userId 过滤） |
| `/api/sync/replicas` | GET/POST | Bearer | CRDT 副本 |
| `/api/sync/replica-keys` | GET/POST/DELETE | Bearer | 盐管理 |
| `/api/storage/*` | * | Bearer | 文件存储 |
| `/api/storage/_put` | PUT | 签名 | 内部上传 |
| `/api/storage/_get` | GET/HEAD | 签名 | 内部下载 |
| `/api/share/*` | * | 混合 | 分享 |
| `/api/admin/users` | GET/POST | **Admin** | 用户管理 |
| `/api/admin/users/[id]` | PUT/DELETE | **Admin** | 用户操作 |
| `/api/translate/google` | GET/POST | **Bearer（v8.0 强制）** | Google 翻译代理（v8.2 加 GET health check） |
| `/api/proxy/wiki` | GET | **Bearer（v8.0 强制）** | Wikipedia 代理 |
| `/api/proxy/resource` | GET | **Bearer（v8.0 强制）** | 通用资源代理 |
| `/api/books/download-url` | POST | Bearer | 远程书籍下载（v8.1 写 Book 表） |
| `/api/deepl/translate` | POST | Bearer+DEEPL_ENABLED | DeepL 翻译 |
| `/api/tts/edge` | POST/GET | Bearer | Edge TTS |
| `/api/kosync` | POST | Bearer | KOReader 同步代理 |
| `/api/metadata/search` | POST | Bearer | 元数据搜索 |
| `/api/opds/proxy` | POST | Bearer | OPDS 代理 |
| `/api/hardcover/graphql` | POST | 透传 | Hardcover 代理 |
| `/api/ai/chat` | POST | Bearer | AI 聊天 |
| `/api/ai/embed` | POST | Bearer | AI 嵌入 |
| `/api/user/delete` | DELETE | Bearer | 普通用户可删自己，管理员不可删 |
| `/api/send/*` | * | Bearer | Send to Readest |

---

## 部署契约

### 必填（2 个）
`ADMIN_EMAIL`、`ADMIN_PASSWORD`

### 可选
| 变量 | 默认 | 说明 |
|---|---|---|
| `PORT` | 8225 | |
| `JWT_SECRET` | 派生 | |
| `DATABASE_URL` | `file:/data/db/readest.db` | |
| `BOOKS_DIR` | `/data/books` | |
| `INBOX_DIR` | `/data/inbox` | |
| `PUBLIC_BASE_URL` | — | 反向代理/分享必填 |
| `ADMIN_USERNAME` | — | v8.1.0 新增：管理员显示名 |
| `DEEPL_ENABLED` | `false` | 设 `true` 启用翻译 |
| `DEEPL_FREE_API_KEYS` | — | |
| `DEEPL_PRO_API_KEYS` | — | |
| `AI_GATEWAY_API_KEY` | — | |

---

## CI 失败排查（17 步）
1. TS 未使用变量 → 删除
2. 类型不匹配 → `as unknown as`
3. 客户端 'fs' → webpack isServer alias + `--webpack`
4. PrismaClient undefined → serverExternalPackages
5. prisma generate → pnpm stub + DATABASE_URL
6. init-admin → createRequire
7. smoke test 超时 → curl + DATABASE_URL
8. "[object Object]" → sync GET tags/progress/metadata 原始字符串
9. 上传 500 → mkdirSync
10. 分享封面 500 → getStorageBase() 绝对 URL
11. TTS 500 → isomorphic-ws/ws 在 serverExternalPackages
12. SharedArrayBuffer 错误 → COEP credentialless
13. Record<string,unknown> index signature → 用 `['key']` 而非 `.key`
14. **v8.0：getAccessToken 是 async** → 必须 `await`，不能当同步函数调
15. **v8.1.0：移除 WebSearchPopup 后，AnnotationToolType 不要再加 `'websearch'`**
16. **v8.2.0：不要动 `useQuotaStats.ts` 用 `user?.storageQuotaMB`**——useAuth 返回的 user 是 Supabase User 类型没有这字段，会 TS 编译失败
17. **v8.3.0：`handleLogout` 是 async**——调用方必须 `await handleLogout()`，不要同步调用

### v8.3 CI 教训

**v8.1 第一次提交失败根因**：`useQuotaStats.ts` 改为 `fetch /api/usage` + 用 `user?.storageQuotaMB`，但 `useAuth()` 返回的是 Supabase User 类型，没有 `storageQuotaMB` 字段，TS 严格模式必报错。

**v8.2 对策**：严格避免动 useQuotaStats，只做 proxyEnabled 这条线（不涉及 Supabase User 类型）+ 安全的 UI 改动（字符串替换、删除代码、新增 SVG）。

**v8.3 对策**：继续不动 useQuotaStats。账号切换数据隔离只动 `useUserActions.ts`（登出清理）+ `useBooksSync.ts`（user.id 检测 + replace 模式 + 登录后 push），不碰 Supabase User 类型。

---

## 禁止行为清单（38 条）

- ❌ 不重构"不要改"的文件
- ❌ 不简化"奇怪代码"
- ❌ 不移除 `--webpack` flag
- ❌ 不改回 static import
- ❌ 不改回 catch-all 路由
- ❌ 不改回 TypeScript init-admin
- ❌ 不改回不带 /api 的 getSupabaseUrl
- ❌ 不改回 localhost NEXT_PUBLIC_*
- ❌ 不恢复 Pro/支付/注册
- ❌ 不预解析 sync GET tags/progress/metadata
- ❌ 不改回 storage.readest.com 字体路径
- ❌ 不改回相对路径 getStorageBase()
- ❌ 不改回 MAX_SAFE_INTEGER 配额
- ❌ 不移除 DEEPL_ENABLED 检查
- ❌ 不移除 try/catch
- ❌ 不移除 mkdirSync
- ❌ 不改回 require-corp COEP
- ❌ 不改回直连 Google/Wikipedia（词典/翻译必须支持代理开关，默认走代理）
- ❌ 不移除 isomorphic-ws/ws 从 serverExternalPackages
- ❌ 不改回 READEST_WEB_BASE_URL
- ❌ **v8.0：不把代理路由改回可选 auth**（必须是强制 auth）
- ❌ 不移除 RemoteDownloadDialog
- ❌ **v8.1.0：不加回 WebSearchPopup**（X-Frame-Options 拒绝，不可用）
- ❌ 不移除 onDownloadFromUrl 从 ImportMenu
- ❌ 不把 admin API 的 validateAdmin 改回 validateUserAndToken
- ❌ 不移除 UserManagement 组件
- ❌ 不让普通用户看到用户管理面板
- ❌ 不允许管理员删除自己或最后一个管理员
- ❌ **v8.1.0：不把远程下载改回只写 File 表**（必须同时写 Book 表）
- ❌ **v8.1.0：不把 AboutWindow/SupportLinks/LegalLinks 改回上游 readest.com 链接**
- ❌ **v8.1.0：不打 tag 不能说完成**（每个版本必须有 git tag）
- ❌ **v8.2.0：不把 AboutWindow 加回检查更新 / 版权 AGPL**（已删除）
- ❌ **v8.2.0：不把 SupportLinks 加回 Discord/Reddit**（已删除，只保留 GitHub + Website）
- ❌ **v8.2.0：不把 Edge TTS 挂钩 proxyEnabled**（保持 v8.0 行为）
- ❌ **v8.2.0：不把 DATA_SUBDIR 改名**（会破坏用户数据目录）
- ❌ **v8.2.0：不动 useQuotaStats.ts 用 Supabase User 上不存在的字段**（CI 必挂）
- ❌ **v8.3.0：不把 handleLogout 改回同步**（必须 async，含清理逻辑）
- ❌ **v8.3.0：不把 useBooksSync 的 prevUserIdRef/replaceModeRef/didInitialPushRef 删掉**（账号切换隔离靠它们）
- ❌ **v8.3.0：不把 updateLibrary 的 replace 分支删掉**（用户切换时必须 replace 不 merge）

---

## 主页与部署教程

- 主页：`https://cshdotcom.github.io/readestl/`
- 部署教程：`https://cshdotcom.github.io/readestl/deploy.html`
- **迭代提示词（隐藏）**：`https://cshdotcom.github.io/readestl/aph.html`
- 源码仓库：`cshdotcom/readestl`（GitHub Pages 仓库）
- 主页含「持续迭代」section，强调高频迭代 + CI 守护 + 社区驱动
- 部署教程 FAQ 含 v8.3 新功能说明（账号切换数据隔离、登出清理、代理开关、代理鉴权说明）

---

**版本**：v8.3.0
**最后更新**：2026-06-20
**适用 commit**：v8.3.0 tag 及之后
**CI 状态**：⏳ 待验证（v8.3.0 push 后必须确认 GitHub Actions Docker Image + CI smoke test 均 conclusion=success）
