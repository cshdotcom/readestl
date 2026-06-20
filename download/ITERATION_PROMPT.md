# Readest Lite — 持续迭代助手提示词（v8.5.0）

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
6. 同步/分享/阅读器：**1:1 复刻原版**
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

---

## 版本与 Tag 规则（v8.1.0 起强制）

**每一个新版本必须打 git tag。**

- 版本号：`v<major>.<minor>.<patch>`
- 推送时 `git push && git push --tags`
- Docker Image workflow 自动 push 3 个 image tag：`<version>`、`<major>.<minor>`、`latest`
- 用户拉取：`docker pull ghcr.io/cshdotcom/readest-lite:8.5.0`（不带 v 前缀）

---

## v8.5.0 改动清单

### 1. 配额 Enforce（后端真正生效）

**之前**：100TB 是写死的展示值，不强制。**v8.5 起真正 enforce。**

| 文件 | 改动 |
|---|---|
| `pages/api/storage/upload.ts` | enforce `storageQuotaMB`：上传前查 `getActualStorageUsage`，超额返 403 |
| `pages/api/storage/stats.ts` | 返回用户实际配额（`storageQuotaMB > 0` ? 实际 : 100TB 兼容） |
| `app/api/translate/google/route.ts` | enforce `translationQuotaKB`（1 KB = 1024 字符）+ 写 `UsageStat` 记账 |
| `pages/api/deepl/translate.ts` | enforce `translationQuotaKB` |
| **新建** `pages/api/usage/index.ts` | GET 返回 storage + translation 用量 + 配额 |

**配额单位约定**：
- `storageQuotaMB`：MB（1 MB = 1024 × 1024 字节）
- `translationQuotaKB`：KB（1 KB = 1024 字符，与 DeepL 计费一致）
- 0 = 无限

### 2. 配额 UI 显示真实数据

| 文件 | 改动 |
|---|---|
| `hooks/useQuotaStats.ts` | fetch `/api/usage`，60s 轮询刷新，显示真实用量 |
| `components/Quota.tsx` | `formatValue` 格式化 bytes/chars，`total=0` 显示 ∞ / Unlimited |

**⚠️ 关键教训**：`useQuotaStats` 用 `user?.id`（Supabase User 有这字段），**绝对不用** `user?.storageQuotaMB`（Supabase User 没这字段，TS 严格模式必报错，CI 必挂）。这是 v8.1 第一次提交失败的根因。

### 3. 代理路由移除白名单 + SSRF 黑名单

| 文件 | 改动 |
|---|---|
| `app/api/proxy/resource/route.ts` | 移除 `ALLOWED_HOSTS` 白名单 → `isPrivateHost` 黑名单 |
| `app/api/proxy/wiki/route.ts` | 同上 |

**SSRF 黑名单**：拒绝 `127.x` / `10.x` / `192.168.x` / `169.254.x` / `172.16-31.x` / 云元数据端点。登录用户可访问任意公网 host。

### 4. RemoteDownloadDialog fire-and-forget

| 文件 | 改动 |
|---|---|
| `app/library/components/RemoteDownloadDialog.tsx` | 立即关闭弹窗 + 后台异步下载 + 完成后 `onDownloadComplete` 回调 |
| `app/library/page.tsx` | 传 `onDownloadComplete={() => pullLibrary(true, true)}` |

不再 `window.location.reload()` 阻塞 UI。Cancel 按钮始终可点。

---

## v8.4.0 回顾：Per-user 加密 vault

### 架构

```
K  = 随机 256-bit AES-GCM 密钥（加密本地 library/settings）
KE = PBKDF2(密码, salt) 派生密钥（加密 K）
K_enc = encryptToEnvelope(K, KE) → 存服务端 User.encryptedVaultKey

登录：密码 → KE → 解密 K_enc → K → 解密本地 .enc 文件
登出：清 K（内存），加密数据保留磁盘，K_enc 在服务端
切换：不同 userId → 不同 .enc 文件 → 天然隔离
改密码：admin 清 K_enc → 用户下次登录生成新 K
```

### 关键文件

| 文件 | 用途 |
|---|---|
| `prisma/schema.prisma` | `User.encryptedVaultKey String?` |
| `libs/crypto/vaultKey.ts` | `generateVaultKey` / `exportVaultKeyToBase64` / `importVaultKeyFromBase64` / `getVaultSalt` |
| `context/VaultContext.tsx` | `vaultKey` state + `setVaultKey` / `clearVault` / `isVaultReady` |
| `utils/vaultState.ts` | 全局模块级 holder（让 libraryService/settingsService 不改签名自动加密） |
| `app/api/auth/v1/vault-key/route.ts` | GET/PUT/DELETE vault 密钥 |
| `services/libraryService.ts` | vault 激活时读写 `library-<userId>.enc` |
| `services/settingsService.ts` | vault 激活时读写 `settings-<userId>.enc` |
| `hooks/useUserActions.ts` | 登出清 VaultContext + vaultState |
| `app/auth/page.tsx` | 登录解密 K_enc → K + 上传 K_enc |
| `app/library/hooks/useBooksSync.ts` | 用户切换只清内存（不清磁盘） |

### 安全保障

- **密码不存客户端**：只在登录瞬间用于 KE 派生
- **K 只在内存**：VaultContext，登出清除
- **K_enc 只在服务端**：User.encryptedVaultKey（密文）
- **浏览器存储是密文**：library-<userId>.enc / settings-<userId>.enc
- **跨账号隔离**：不同 userId → 不同 .enc 文件 + 不同 K_enc

---

## v8.3.0 回顾：账号切换数据隔离

- `useUserActions.handleLogout`：清 library state + settings state + cursor + transferQueue + VaultContext
- `useBooksSync`：`prevUserIdRef` / `replaceModeRef` / `didInitialPushRef` 检测用户切换
- `updateLibrary` 加 replace 分支（不 merge，直接 setLibrary(syncedBooks)）
- 不删 Books/ 目录下的 `<hash>/` 文件

---

## 多用户系统（v7.0，仍适用）

- **admin**：`ADMIN_EMAIL`/`ADMIN_PASSWORD` 创建
- **user**：管理员通过用户管理面板创建
- Admin API：`GET/POST /api/admin/users`，`PUT/DELETE /api/admin/users/[id]`（仅 admin）
- `validateAdmin()` 用于 admin API；`validateUserAndToken` 从 DB 刷新 role/quotas
- **v8.5：配额真正 enforce**——`storageQuotaMB > 0` 时上传检查，`translationQuotaKB > 0` 时翻译检查

---

## 绝对红线

- Pro/支付/注册不能加回来
- 固定路由不能改回 catch-all
- `READEST_WEB_BASE_URL = ''`
- WebSearchPopup 不能加回来
- sync/replicas/replica-key/share/auth 路径参数返回结构不能改
- Prisma schema 不能改已有字段（可以加新字段）
- 代理路由用强制 auth
- **不要动 useQuotaStats.ts 用 Supabase User 上不存在的字段**（CI 必挂）
- **handleLogout 是 async**——调用方必须 await
- **useBooksSync 的 prevUserIdRef/replaceModeRef/didInitialPushRef 不能删**
- **vaultState.ts 是 libraryService/settingsService 自动加密的桥梁，不能删**
- **VaultContext 必须在 AuthProvider 内（依赖 useAuth）**
- **useUserActions 不再 import useEnv/envConfig**（登出不调 appService）
- **代理路由已移除白名单**——不要加回 ALLOWED_HOSTS
- **RemoteDownloadDialog 是 fire-and-forget**——不要改回同步阻塞 + window.location.reload()

---

## 已知"奇怪"代码（50 项）

1-30: 与 v7.0 相同

31. **v8.0：代理路由用强制 auth**
32. 远程书籍下载 `/api/books/download-url` 需要强制 auth
33. ~~WebSearchPopup~~ → **v8.1.0 已删除**
34. User 模型有 role/displayName/storageQuotaMB/translationQuotaKB/encryptedVaultKey 字段
35. `validateAdmin()` 用于 admin API
36. **v8.1.0：远程下载同时写 File 表 + Book 表**
37. **v8.1.0：`ADMIN_USERNAME` env**
38. **v8.1.0：每个版本必须打 git tag**
39. **v8.2.0：`utils/proxy.ts` 提供 `isProxyEnabled` / `fetchViaWikiProxy`**
40. **v8.2.0：AboutWindow 删除检查更新 + 版权 AGPL**
41. **v8.2.0：SupportLinks 删除 Discord/Reddit，加 SVG globe 按钮**
42. **v8.2.0：代理路由加 GET health check**
43. **v8.3.0：登出时彻底清空 library + settings + cursor + transferQueue**
44. **v8.3.0：useBooksSync 加 prevUserIdRef/replaceModeRef/didInitialPushRef**
45. **v8.3.0：updateLibrary 加 replace 分支**
46. **v8.3.0：不删 Books/ 目录下的 `<hash>/` 文件**
47. **v8.4.0：Per-user 加密 vault——K_enc 在服务端，K 在内存**
48. **v8.4.0：vaultState.ts 全局 holder 让 services 自动加密**
49. **v8.4.0：登录时上传 K_enc，登出不需要密码**
50. **v8.5.0：配额真正 enforce——storageQuotaMB/translationQuotaKB > 0 时超额返 403**
51. **v8.5.0：代理路由移除白名单 + SSRF 黑名单（isPrivateHost）**
52. **v8.5.0：RemoteDownloadDialog fire-and-forget + onDownloadComplete prop**

---

## API 路由清单

| 路由 | 方法 | Auth | 说明 |
|---|---|---|---|
| `/api/auth/v1/token` | POST | 无 | 登录/刷新（返回 encryptedVaultKey） |
| `/api/auth/v1/vault-key` | GET/PUT/DELETE | Bearer | v8.4：vault 密钥管理 |
| `/api/auth/v1/user` | GET | Bearer | 获取用户 |
| `/api/auth/v1/logout` | POST | 无 | 204 |
| `/api/auth/v1/settings` | GET | 无 | 配置 |
| `/api/sync` | GET/POST | Bearer | 同步（按 userId 过滤） |
| `/api/sync/replicas` | GET/POST | Bearer | CRDT 副本 |
| `/api/sync/replica-keys` | GET/POST/DELETE | Bearer | 盐管理 |
| `/api/storage/*` | * | Bearer | 文件存储（v8.5 enforce 配额） |
| `/api/storage/_put` | PUT | 签名 | 内部上传 |
| `/api/storage/_get` | GET/HEAD | 签名 | 内部下载 |
| `/api/share/*` | * | 混合 | 分享 |
| `/api/admin/users` | GET/POST | Admin | 用户管理 |
| `/api/admin/users/[id]` | PUT/DELETE | Admin | 改密码时清 encryptedVaultKey |
| `/api/translate/google` | GET/POST | Bearer | Google 翻译代理（v8.5 enforce + 记账） |
| `/api/proxy/wiki` | GET | Bearer | Wikipedia 代理（v8.5 SSRF 黑名单） |
| `/api/proxy/resource` | GET | Bearer | 通用资源代理（v8.5 SSRF 黑名单） |
| `/api/books/download-url` | POST | Bearer | 远程书籍下载（v8.1 写 Book 表） |
| `/api/usage` | GET | Bearer | **v8.5 新增**：返回 storage + translation 用量 |
| `/api/deepl/translate` | POST | Bearer+DEEPL_ENABLED | DeepL 翻译（v8.5 enforce） |
| `/api/tts/edge` | POST/GET | Bearer | Edge TTS |
| `/api/kosync` | POST | Bearer | KOReader 同步代理 |
| `/api/metadata/search` | POST | Bearer | 元数据搜索 |
| `/api/opds/proxy` | POST | Bearer | OPDS 代理 |
| `/api/hardcover/graphql` | POST | 透传 | Hardcover 代理 |
| `/api/ai/chat` | POST | Bearer | AI 聊天 |
| `/api/ai/embed` | POST | Bearer | AI 嵌入 |
| `/api/user/delete` | DELETE | Bearer | 普通用户可删自己 |
| `/api/send/*` | * | Bearer | Send to Readest |

---

## 部署契约

### 必填
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
| `ADMIN_USERNAME` | — | v8.1：管理员显示名 |
| `DEEPL_ENABLED` | `false` | |
| `DEEPL_FREE_API_KEYS` | — | |
| `DEEPL_PRO_API_KEYS` | — | |
| `AI_GATEWAY_API_KEY` | — | |

---

## CI 失败排查（20 步）

1-16: 与 v8.3 相同
17. **v8.3：handleLogout 是 async**——必须 await
18. **v8.4：useUserActions 不再 import useEnv/envConfig**
19. **v8.4：VaultContext 必须在 AuthProvider 内**（依赖 useAuth）
20. **v8.5：useQuotaStats 用 user?.id 不用 user?.storageQuotaMB**（Supabase User 类型问题）

---

## 禁止行为清单（48 条）

v7.0-v8.4 的所有禁止行为仍然适用，加上：

- ❌ 不把配额 enforce 改回写死 100TB（v8.5 起真正 enforce）
- ❌ 不把代理路由白名单加回 ALLOWED_HOSTS（v8.5 改为 SSRF 黑名单）
- ❌ 不把 RemoteDownloadDialog 改回同步阻塞 + window.location.reload()
- ❌ 不把 useQuotaStats 改回写死 100TB / 0 usage（v8.5 fetch /api/usage）
- ❌ 不在 useQuotaStats 用 user?.storageQuotaMB（CI 必挂）

---

## 主页与部署教程

- 主页：`https://cshdotcom.github.io/readestl/`
- 部署教程：`https://cshdotcom.github.io/readestl/deploy.html`
- **迭代提示词（隐藏）**：`https://cshdotcom.github.io/readestl/aph.html`

---

**版本**：v8.5.0
**最后更新**：2026-06-20
**适用 commit**：`a130093` 及之后
**CI 状态**：✅ Docker Image + CI smoke test success
**镜像**：`ghcr.io/cshdotcom/readest-lite:8.5.0` / `8.5` / `latest`
