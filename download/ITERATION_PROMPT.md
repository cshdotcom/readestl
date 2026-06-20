# Readest Lite — 持续迭代助手提示词（v8.4.0）

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

---

## 版本与 Tag 规则（v8.1.0 起强制）

**每一个新版本必须打 git tag。**

- 版本号：`v<major>.<minor>.<patch>`
- 推送时 `git push && git push --tags`
- Docker Image workflow 自动 push 3 个 image tag：`<version>`、`<major>.<minor>`、`latest`
- 用户拉取：`docker pull ghcr.io/cshdotcom/readest-lite:8.4.0`（不带 v 前缀）

---

## v8.4.0 改动清单

### 架构：服务端托管密钥 + Per-user 加密数据

```
K  = 随机 256-bit AES-GCM 密钥（加密本地 library/settings）
KE = PBKDF2(密码, salt) 派生密钥（加密 K）
K_enc = encryptToEnvelope(K, KE) → 存服务端 User.encryptedVaultKey

登录：密码 → KE → 解密 K_enc → K → 解密本地 library-<userId>.enc / settings-<userId>.enc
登出：清 K（内存）→ 加密数据保留磁盘 → K_enc 在服务端
切换：不同 userId → 不同 .enc 文件 → 天然隔离
改密码：admin 清 K_enc → 用户下次登录生成新 K
```

### 1. 后端基础设施（alpha.1）

- **Prisma schema**：`User.encryptedVaultKey String?`
- **`libs/crypto/vaultKey.ts`**（新文件）：
  - `generateVaultKey()` — 随机 256-bit AES-GCM CryptoKey
  - `exportVaultKeyToBase64(key)` / `importVaultKeyFromBase64(b64)`
  - `getVaultSalt(userId)` — 从 userId 派生 PBKDF2 salt
- **`localAuth.ts`**：`AuthSession` 加 `encryptedVaultKey` 字段；`signInWithPassword` / `refreshSession` 返回 `user.encryptedVaultKey`
- **`/api/auth/v1/vault-key`**（新路由）：GET/PUT/DELETE
- **admin 改密码**：清空 `encryptedVaultKey`

### 2. 前端 VaultContext + 登录解密（alpha.2）

- **`context/VaultContext.tsx`**（新文件）：`vaultKey` state + `setVaultKey` / `clearVault` / `isVaultReady`
- **`Providers.tsx`**：包 `<VaultProvider>`
- **`app/auth/page.tsx`**：登录后用密码派生 KE → 解密 K_enc → K → `setVaultKey(K)`；首次登录生成新 K

### 3. 加密读写（alpha.3）

- **`utils/vaultState.ts`**（新文件）：全局模块级 holder，让 libraryService/settingsService 不改签名自动加密
- **`VaultContext.tsx`**：同步 vaultKey + userId 到 vaultState
- **`libraryService.ts`**：vault 激活时读写 `library-<userId>.enc`（加密 CipherEnvelope）；未激活时原逻辑
- **`settingsService.ts`**：vault 激活时读写 `settings-<userId>.enc`；未激活时原逻辑
- **迁移**：首次登录读旧明文 → 立即加密写入新文件

### 4. 登出加密 + 用户切换（v8.4.0）

- **`app/auth/page.tsx`**：登录时用 KE 加密 K → PUT `/api/auth/v1/vault-key` 上传 K_enc
- **`useUserActions.handleLogout`**：
  - 清 library/settings 内存 state（磁盘加密文件保留）
  - 清 transfer queue
  - 清 VaultContext + vaultState（K 从内存清除）
  - **不需要密码**（K_enc 已在服务端）
- **`useBooksSync.ts`**：用户切换只清内存 state（不清磁盘，per-user .enc 文件天然隔离）

### 安全保障

- **密码不存客户端**：只在登录瞬间用于 KE 派生
- **K 只在内存**：VaultContext，登出清除
- **K_enc 只在服务端**：User.encryptedVaultKey（密文）
- **浏览器存储是密文**：library-<userId>.enc / settings-<userId>.enc
- **跨账号隔离**：不同 userId → 不同 .enc 文件 + 不同 K_enc

---

## v8.5+ 待办

- 配额 enforce（`storageQuotaMB` / `translationQuotaKB` 真正生效）
- 配额 UI 显示真实数据（⚠️ 不要用 `user?.storageQuotaMB`，useAuth 返回的是 Supabase User 类型没有这字段，CI 必挂）
- 代理路由移除白名单 + SSRF 黑名单
- RemoteDownloadDialog fire-and-forget + 任务队列 + 同步 + 重试

---

## 多用户系统（v7.0，仍适用）

- **admin**：`ADMIN_EMAIL`/`ADMIN_PASSWORD` 创建
- **user**：管理员通过用户管理面板创建
- Admin API：`GET/POST /api/admin/users`，`PUT/DELETE /api/admin/users/[id]`（仅 admin）
- `validateAdmin()` 用于 admin API；`validateUserAndToken` 从 DB 刷新 role/quotas

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

---

## CI 失败排查（18 步）

1-16: 与 v8.3 相同
17. **v8.3：handleLogout 是 async**——必须 await
18. **v8.4：useUserActions 不再 import useEnv/envConfig**（登出不调 appService）
19. **v8.4：VaultContext 必须在 AuthProvider 内**（依赖 useAuth 的 user）

---

## 禁止行为清单（42 条）

v7.0-v8.3 的所有禁止行为仍然适用，加上：

- ❌ 不把 vaultState.ts 删掉（libraryService/settingsServer 靠它自动加密）
- ❌ 不把 VaultContext 从 AuthProvider 内移出（依赖 useAuth）
- ❌ 不在 handleLogout 里调 saveSettings（vaultState 还没清，会加密写盘覆盖）
- ❌ 不把 useBooksSync 用户切换的"不清磁盘"改回"清磁盘"（per-user .enc 文件天然隔离）
- ❌ 不在 admin 改密码时忘记清空 encryptedVaultKey（否则旧密码 KE 解不开新 K_enc）

---

## API 路由清单

| 路由 | 方法 | Auth | 说明 |
|---|---|---|---|
| `/api/auth/v1/token` | POST | 无 | 登录/刷新（返回 encryptedVaultKey） |
| `/api/auth/v1/vault-key` | GET/PUT/DELETE | Bearer | **v8.4 新增**：vault 密钥管理 |
| `/api/sync` | GET/POST | Bearer | 同步（按 userId 过滤） |
| `/api/admin/users` | GET/POST | Admin | 用户管理 |
| `/api/admin/users/[id]` | PUT/DELETE | Admin | 改密码时清 encryptedVaultKey |
| `/api/translate/google` | GET/POST | Bearer | Google 翻译代理 |
| `/api/proxy/wiki` | GET | Bearer | Wikipedia 代理 |
| `/api/proxy/resource` | GET | Bearer | 通用资源代理 |
| `/api/books/download-url` | POST | Bearer | 远程书籍下载 |
| `/api/deepl/translate` | POST | Bearer | DeepL 翻译 |
| `/api/tts/edge` | POST/GET | Bearer | Edge TTS |
| `/api/storage/*` | * | Bearer | 文件存储 |
| `/api/share/*` | * | 混合 | 分享 |
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

## 主页与部署教程

- 主页：`https://cshdotcom.github.io/readestl/`
- 部署教程：`https://cshdotcom.github.io/readestl/deploy.html`
- **迭代提示词（隐藏）**：`https://cshdotcom.github.io/readestl/aph.html`

---

**版本**：v8.4.0
**最后更新**：2026-06-20
**适用 commit**：`48b2d79` 及之后
**CI 状态**：✅ Docker Image + CI smoke test success
**镜像**：`ghcr.io/cshdotcom/readest-lite:8.4.0` / `8.4` / `latest`
