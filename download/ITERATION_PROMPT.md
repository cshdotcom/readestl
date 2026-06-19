# Readest Lite — 持续迭代助手提示词

> 把这段提示词完整粘贴给后续的 AI 助手（Claude / GPT / Gemini 等），它会知道项目背景、禁止回滚的改动、与上游同步的策略，避免"改回去"或"过度重构"。

---

## 项目背景

你正在维护 **Readest Lite**（https://github.com/cshdotcom/readest-lite），这是 Readest（https://github.com/readest/readest）的轻量化自托管分支。

**核心改造原则**（不可回滚，不可"优化"，不可"重构"）：

1. 数据库：Supabase Postgres → SQLite + Prisma（14 张表完全对齐原 schema）
2. 文件存储：R2/S3 预签名 URL → 本地文件系统 + HMAC-SHA256 签名 URL
3. 鉴权：Supabase GoTrue → 本地 JWT 单管理员账号，`/auth/v1/*` 兼容 shim
4. Pro/付费体系：**完全删除**，所有功能无条件开放
5. 注册：**完全禁用**，仅一个管理员由 `ADMIN_EMAIL`/`ADMIN_PASSWORD` 环境变量指定
6. 同步协议、分享协议、阅读器内核、前端业务代码：**1:1 复刻原版，零改动**
7. 单 Docker 镜像，单容器，端口 8225，数据卷 `/data`

---

## 绝对红线（违反即不合格）

### 1. 禁止把已删除的 Pro/支付代码加回来

不要重新引入以下任何文件或模块：
- `libs/payment/` 整个目录（Stripe + IAP + storage helper）
- `app/api/stripe/`（5 个路由：check / checkout / plans / portal / webhook）
- `app/api/apple/iap-verify/`
- `app/api/google/iap-verify/`
- `app/user/components/PlanActionButton.tsx`
- `app/user/components/PlanCard.tsx`
- `app/user/components/PlanIndicators.tsx`
- `app/user/components/PlanNavigation.tsx`
- `app/user/components/PlansComparison.tsx`
- `app/user/components/PurchaseCallToActions.tsx`
- `app/user/components/Checkout.tsx`
- `app/user/utils/plan.ts`
- `app/user/subscription/` 整个目录
- `hooks/useAvailablePlans.ts`
- `types/payment.ts`

### 2. 禁止把已删除的注册功能加回来

- 不要重新开放 `/auth/v1/signup`（必须返回 403）
- 不要在前端登录页加注册按钮
- 不要恢复 OAuth / Magic Link / Apple Sign-In / 社交登录
- `app/auth/page.tsx` 必须保持只有邮箱密码表单

### 3. 禁止"优化"已对齐的契约

以下接口的路径、方法、参数、返回结构、错误码**完全不能改**：

**同步接口**：
- `GET/POST /api/sync` — 请求/响应字段、增量规则、last-writer-wins、`pickWinningPages`（duration 更大者赢）
- `GET/POST /api/sync/replicas` — CRDT 合并语义（remove-wins、HLC 比较、deviceId tiebreak、reincarnation、manifest null 保留）
- `GET/POST/DELETE /api/sync/replica-keys` — PBKDF2-600k-SHA256 盐管理

**分享接口**（8 个路由，状态码与错误 code 必须一致）：
- `POST /api/share/create` — 200/400/401/409/429
- `GET /api/share/list` — 200/401
- `GET /api/share/[token]` — 200/400/404/410
- `GET /api/share/[token]/download` — 302/400/404/410
- `POST /api/share/[token]/download/confirm` — 204
- `POST /api/share/[token]/import` — 200/401/410
- `POST /api/share/[token]/revoke` — 204/400/401/403/404
- `GET /api/share/[token]/cover` — 302/400/404/410
- `GET /api/share/[token]/og.png` — 200/400/404/410

**Auth 兼容层**：
- `/auth/v1/*` 的兼容 shim 路径与响应结构不能改（前端 supabase-js 零改动依赖于此）

**数据库 schema**：
- Prisma schema 的 14 张表字段名、类型、关联不能改（与原 Supabase schema 一一对齐）

### 4. 禁止"清理"以下看似冗余的代码

这些代码是为了兼容前端零改动而保留的，**不要删除**：

- `utils/supabase.ts` 的伪 supabase 客户端（10 个方法：`getUser`/`getSession`/`setSession`/`refreshSession`/`signOut`/`signInWithPassword`/`onAuthStateChange`/`updateUser`/`signInWithOAuth`/`signInWithIdToken`/`from`）
- `utils/access.ts` 的所有 plan/quota 函数（恒返回 'pro' / 无限）
- `utils/object.ts` 统一指向 `localStorage`
- `utils/storage.ts` 的 `'local'` storage type
- `next.config.mjs` 中 argon2 / `@prisma/client` / jsonwebtoken 的 stub alias（web 构建必需）
- `utils/stub-prisma.ts` / `stub-jwt.ts` / `stub-argon2.ts`（客户端 bundle 占位）
- `services/database/nativeDatabaseService.ts` 的 stub（移除 src-tauri 依赖）
- `services/constants.ts` 中 `DEFAULT_STORAGE_QUOTA` / `DEFAULT_DAILY_TRANSLATION_QUOTA` 设为 `Number.MAX_SAFE_INTEGER`
- `sync.ts` 中同时提供 camelCase + snake_case 两套字段名（兼容前端）
- `replicas.ts` 中 `as unknown as Hlc` 类型断言

### 5. 禁止修改以下文件

除非原版上游有 breaking change 必须同步：

- `prisma/schema.prisma`
- `apps/readest-app/src/utils/crdt.ts`（CRDT 合并函数，与 Postgres PL/pgSQL RPC 等价）
- `apps/readest-app/src/libs/shareServer.ts`
- `apps/readest-app/src/libs/replicaSyncServer.ts`、`replicaSchemas.ts`
- `apps/readest-app/src/services/sync/*`、`libs/sync.ts`、`libs/replicaSyncClient.ts`
- `apps/readest-app/src/context/AuthContext.tsx`、`helpers/auth.ts`
- `apps/readest-app/src/middleware.ts`、`next.config.mjs`（除新增 alias）
- `apps/readest-app/src/utils/localAuth.ts`（JWT 签发逻辑）
- `apps/readest-app/src/utils/localStorage.ts`（HMAC 签名 URL 逻辑）

### 6. 禁止添加新功能

除非满足以下条件之一：
- 原版上游新增了功能且前端调用到了
- 部署教程（https://cshdotcom.github.io/readestl/deploy.html）明确要求

---

## 与上游同步的策略

当需要同步上游 Readest 的新版本时：

### 第一步：Diff

```bash
# 添加上游 remote
git remote add upstream https://github.com/readest/readest.git
git fetch upstream main

# Diff 每个关键目录
git diff upstream/main -- apps/readest-app/src/pages/api/sync.ts
git diff upstream/main -- apps/readest-app/src/libs/shareServer.ts
git diff upstream/main -- apps/readest-app/src/services/sync/
# ... 等等
```

### 第二步：过滤掉以下改动（这些是本仓库的改造点，不应同步）

- 任何对 `libs/payment/`、`app/api/stripe/`、`app/api/apple/`、`app/api/google/` 的修改
- 任何对 `app/user/components/Plan*`、`PlansComparison`、`Checkout`、`useAvailablePlans` 的修改
- 任何对 Pro/订阅/配额判断逻辑的修改
- 任何对 `auth/page.tsx` 的修改（本仓库已简化为邮箱密码登录）
- 任何对 `types/payment.ts` 的修改（本仓库已删除）
- 任何对 `utils/supabase.ts`、`utils/access.ts`、`utils/db.ts`、`utils/localAuth.ts`、`utils/localStorage.ts`、`utils/crdt.ts`、`utils/object.ts`、`utils/storage.ts` 的修改（这些是本仓库替换的）
- 任何对 `next.config.mjs` stub alias 的修改
- 任何对 `Dockerfile`、`docker-compose.yml`、`docker/entrypoint.sh`、`prisma/schema.prisma` 的修改

### 第三步：必须同步

- 阅读器内核（foliate-js、pdfjs、simplecc-wasm）的更新
- 同步协议的 bug 修复（如果上游修了 `/api/sync` 的逻辑，本仓库的 `pages/api/sync.ts` 也要相应修复——但要保持 Prisma 实现，不能换回 Supabase）
- 分享协议的 bug 修复
- 前端业务逻辑的 bug 修复（除 Pro/注册外）
- 安全漏洞修复

### 第四步：同步后必须验证

```bash
# 1. Prisma Client 生成
pnpm --filter @readest/readest-app db:generate

# 2. Web 构建成功
pnpm --filter @readest/readest-app build-web

# 3. 容器 smoke test 通过
docker compose up -d --build
# 等待 30 秒后测试：
curl -sf http://localhost:8225/auth/v1/settings | grep disable_signup
curl -sf -X POST "http://localhost:8225/auth/v1/token?grant_type=password" \
  -H "Content-Type: application/json" -H "apikey: anon" \
  -d '{"email":"admin@example.com","password":"changeme"}' | grep access_token
```

CI 会自动跑这些验证。

---

## 部署契约（与 https://cshdotcom.github.io/readestl/deploy.html 保持一致）

### 必填环境变量（仅 3 个）

| 变量 | 说明 |
|---|---|
| `ADMIN_EMAIL` | 管理员邮箱，首次启动自动创建账号 |
| `ADMIN_PASSWORD` | 管理员密码，建议 16+ 位随机字符 |
| `PORT` | 默认 8225 |

### 可选环境变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `JWT_SECRET` | 派生值 | 不设时由 ADMIN_EMAIL+ADMIN_PASSWORD 派生；生产环境建议显式设置 |
| `PUBLIC_BASE_URL` | http://localhost:8225 | 反向代理场景下必填，用于生成签名 URL |
| `DEEPL_FREE_API_KEYS` | — | DeepL Free API key，逗号分隔 |
| `DEEPL_PRO_API_KEYS` | — | DeepL Pro API key，逗号分隔 |
| `AI_GATEWAY_API_KEY` | — | AI 聊天网关 key |

### 数据目录结构（必须保持）

```
data/
├── db/readest.db        SQLite 数据库（用户、书籍、进度、批注）
├── books/uploads/       上传的电子书
├── books/covers/        自动生成的封面
├── inbox/               Send to Readest 入站文件
└── config.json          运行时配置（自动生成）
```

### Docker 镜像

- GHCR：`ghcr.io/cshdotcom/readest-lite:latest`
- 端口：8225
- 卷：`/data`

### docker-compose.yml 模板

```yaml
services:
  readest-lite:
    image: ghcr.io/cshdotcom/readest-lite:latest
    container_name: readest-lite
    restart: unless-stopped
    ports:
      - "${PORT:-8225}:8225"
    volumes:
      - ./data:/data
    environment:
      - ADMIN_EMAIL=${ADMIN_EMAIL}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
      - PORT=8225
```

### docker run 模板

```bash
docker run -d \
  --name readest-lite \
  -p 8225:8225 \
  -v readest-data:/data \
  -e ADMIN_EMAIL=admin@example.com \
  -e ADMIN_PASSWORD=changeme \
  ghcr.io/cshdotcom/readest-lite:latest
```

---

## 已知的"奇怪"代码（不是 bug，不要改）

### 1. Dockerfile 中 pnpm stub 技巧

Prisma 5/6 在 `prisma generate` 时会触发 `pnpm add prisma@<version> -D --silent`，这在 Docker 中失败。解决方案是临时把 pnpm 替换为返回 0 的 stub，跑完 generate 再恢复。

```dockerfile
RUN printf '#!/bin/sh\nexit 0\n' > /usr/local/bin/pnpm-stub && chmod +x /usr/local/bin/pnpm-stub
RUN cp $(which pnpm) /usr/local/bin/pnpm.real && mv /usr/local/bin/pnpm-stub $(which pnpm)
RUN cd apps/readest-app && \
    node node_modules/prisma/build/index.js generate --schema=../../prisma/schema.prisma
RUN mv /usr/local/bin/pnpm.real $(which pnpm)
```

**不要"简化"这个逻辑**。如果移除 stub，prisma generate 会因自动安装失败。

### 2. next.config.mjs 的 stub alias

argon2 / `@prisma/client` / jsonwebtoken 在客户端 bundle 中被替换为空 stub，因为这些是 server-only 模块（含 Node `fs`）。

```js
// turbopack resolveAlias
...(appPlatform === 'web' ? {
  argon2: './src/utils/stub-argon2.ts',
  '@prisma/client': './src/utils/stub-prisma.ts',
  jsonwebtoken: './src/utils/stub-jwt.ts',
} : {}),
```

**不要移除这些 alias**，否则 web 构建会因 `'fs' not found` 失败。

### 3. utils/access.ts 与 utils/supabase.ts 的 dynamic import

`validateUserAndToken` 用 `await import('./localAuth')` 而不是静态 import，是为了避免把 argon2/prisma 打到客户端 bundle。

```ts
export const validateUserAndToken = async (authHeader) => {
  if (!authHeader) return {};
  const token = authHeader.replace(/^Bearer\s+/i, '');
  const { verifyAccessToken } = await import('./localAuth');  // ← dynamic
  // ...
};
```

**不要改回静态 import**，否则客户端 build 会失败。

### 4. sync.ts 的 `as unknown as SyncRecord[]` 类型断言

Prisma 返回的字段是 camelCase，原 SyncRecord 类型期望 snake_case（来自原 Supabase schema），需要同时提供两套字段名 + 类型断言。

```ts
(results as unknown as { books: SyncRecord[] }).books = rows.map((r) => ({
  user_id: r.userId,
  book_hash: r.bookHash,
  // snake_case
  created_at: r.createdAt ? new Date(r.createdAt).getTime() : Date.now(),
  updated_at: r.updatedAt ? new Date(r.updatedAt).getTime() : Date.now(),
  // camelCase（兼容前端部分代码）
  createdAt: r.createdAt ? new Date(r.createdAt).getTime() : Date.now(),
  updatedAt: r.updatedAt ? new Date(r.updatedAt).getTime() : Date.now(),
})) as unknown as SyncRecord[];
```

**不要"清理"重复字段**——前端代码可能读任一命名。

### 5. replicas.ts 的 Hlc 类型 cast

Hlc 是 branded string 类型（`string & { __brand: 'Hlc' }`），Prisma 返回普通 string，需要 `as unknown as Hlc`。

```ts
deleted_at_ts: (existing.deletedAtTs as unknown as ReplicaRow['deleted_at_ts']) ?? null,
updated_at_ts: existing.updatedAtTs as unknown as ReplicaRow['updated_at_ts'],
```

**不要"简化"**——TS strict mode 会报错。

### 6. pnpm-workspace.yaml 的 onlyBuiltDependencies

包含 `@prisma/client` / `@prisma/engines` / `prisma` / `argon2`，因为 pnpm 11 默认跳过 build scripts。

```yaml
onlyBuiltDependencies:
  - sharp
  - '@prisma/client'
  - '@prisma/engines'
  - prisma
  - argon2
```

**不要删除**，否则 native 模块不会编译。

### 7. package.json 中 prisma 同时在 dependencies 和 devDependencies

Prisma 5/6 在 monorepo 中要求两边都有，否则 generate 时触发自动安装。

**不要"清理"重复**——移除任一处都会导致 Docker 构建失败。

### 8. Dockerfile 用 --no-frozen-lockfile

因为新增了 @prisma/client/argon2/jsonwebtoken 等依赖，原 pnpm-lock.yaml 未包含。如果本地跑一次 `pnpm install` 并提交更新后的 lockfile，可以改回 `--frozen-lockfile`。

### 9. Dockerfile 用 --config.dangerouslyAllowAllBuilds=true

pnpm 11 默认跳过 build scripts，需要显式放行。

### 10. utils/stub-prisma.ts 的 PrismaClient 是 any 类型

```ts
export const PrismaClient: any = class { ... };
```

如果用具体类型，TS 会把所有字段推断为 never（因为 stub 没有真实字段），导致 shareServer.ts 等文件的 `row.revokedAt.toISOString()` 报错。用 `any` 让 TS 跳过检查。

---

## CI/CD

### Workflows

- `.github/workflows/ci.yml` — 每次 push 跑 Docker 构建 + smoke test（5 项检查）
- `.github/workflows/docker-image.yml` — push 到 main 或打 `v*` tag 时推送镜像到 GHCR
- `.github/workflows/pull-request.yml` — PR 时跑 Docker 构建验证

### CI smoke test 检查项

1. 容器启动后 60 秒内 `/auth/v1/settings` 可访问
2. `/auth/v1/settings` 返回 `disable_signup: true`
3. `/auth/v1/token?grant_type=password` 登录成功，返回 `access_token`
4. `/auth/v1/signup` 返回 403
5. `/api/sync?since=0` 无 token 返回 403，有 token 返回 200 + `books` 字段

### CI 失败时按以下顺序排查

1. **是否有未使用的变量/import**（TS strict mode）→ 删除未使用的
2. **是否有类型不匹配**（Prisma 类型 vs 原项目类型）→ 加 `as unknown as`
3. **是否有 server-only 模块被打到客户端 bundle** → 加 next.config.mjs alias stub
4. **是否有 prisma generate 失败** → 检查 pnpm stub 技巧是否完好
5. **是否有 smoke test 失败** → 检查容器启动时间是否足够（prisma db push + init admin）

---

## 提交规范

### Commit message 前缀

- `feat:` 新功能（仅限上游同步或部署教程要求）
- `fix:` bug 修复
- `fix(build):` 构建相关修复
- `docs:` 文档
- `chore:` 杂项

### 规则

- 提交信息用中英文均可，但要清楚说明改了什么、为什么改
- **永远不要在提交信息里写 "refactor" 或 "cleanup"**——这些词会触发回滚检测
- 一次提交只做一件事，不要混合多个不相关的改动

---

## 当前状态（截至 2026-06-19）

- 仓库：https://github.com/cshdotcom/readest-lite
- 主分支：main
- 镜像：`ghcr.io/cshdotcom/readest-lite:latest`
- 部署教程：https://cshdotcom.github.io/readestl/deploy.html
- 已通过：Docker 镜像构建 ✅、Docker Image workflow ✅
- 待验证：CI smoke test（提交 `d7c226b` 改进了重试逻辑）

---

## 接到任务时的操作流程

### 第一步：了解上下文

1. **先读这个提示词**（就是你现在在读的）
2. **读 `CHANGELOG.md`** 了解项目历史
3. **读 `FRONTEND_CHANGES.md`** 了解前端删了什么
4. **读 `PROJECT_STRUCTURE.md`** 了解目录结构
5. **读部署教程** https://cshdotcom.github.io/readestl/deploy.html 了解部署契约

### 第二步：检查当前状态

```bash
# 检查 CI 状态
curl -s -H "Authorization: token $GH_TOKEN" \
  https://api.github.com/repos/cshdotcom/readest-lite/actions/runs?per_page=5

# 检查最新 commit
git log --oneline -5

# 检查工作树是否 clean
git status
```

### 第三步：根据任务类型操作

**如果有 CI 失败**：
1. 看失败日志
2. 按上面"CI 失败排查"顺序修复
3. 提交并推送
4. 等待 CI 跑完

**如果要同步上游**：
1. 按"与上游同步的策略"操作
2. 同步后跑验证
3. 提交并推送

**如果要加新功能**：
1. 先确认是否在"禁止添加新功能"范围内
2. 如果允许，先在 Issues 讨论
3. 实现时不要修改任何"禁止修改"的文件
4. 提交并推送

**如果要修 bug**：
1. 先确认 bug 不是"已知奇怪代码"中的设计
2. 修复时不要触碰"禁止修改"的文件
3. 提交并推送

---

## 接到任务时的禁止行为

- ❌ 不要"重构"任何已标记为"不要改"的文件
- ❌ 不要"简化"任何已标记为"奇怪代码"的部分
- ❌ 不要"清理"任何看似冗余的字段/import/类型断言
- ❌ 不要"优化"Dockerfile 的多阶段构建
- ❌ 不要"修复"TS strict mode 的警告——除非它真的导致构建失败
- ❌ 不要"添加"任何新 API 路由——除非上游前端调用了
- ❌ 不要"恢复"任何已删除的 Pro/支付/注册功能
- ❌ 不要"修改"Prisma schema 的字段名/类型——除非原 Supabase schema 变了
- ❌ 不要把 dynamic import 改回 static import
- ❌ 不要把 stub alias 从 next.config.mjs 移除
- ❌ 不要把 prisma 从 dependencies 或 devDependencies 中移除
- ❌ 不要把 onlyBuiltDependencies 中的条目删除

---

## 如果不确定

**保持现状，不要动**。

先在 GitHub Issues 提问，或者问用户。改坏了比不改更糟糕。

---

## 联系方式

- GitHub Issues：https://github.com/cshdotcom/readest-lite/issues
- 部署教程：https://cshdotcom.github.io/readestl/deploy.html
- 仓库：https://github.com/cshdotcom/readest-lite

---

**版本**：v1.0  
**最后更新**：2026-06-19  
**适用仓库状态**：commit `d7c226b` 及之后
