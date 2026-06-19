# Readest Lite — 单容器轻量化版

基于 readest 官方仓库的轻量化改造：
- 数据库：Supabase Postgres → SQLite + Prisma
- 文件存储：R2/S3 → 本地文件系统（`/data/books`）
- 鉴权：Supabase Auth → 本地单账号 JWT（兼容前端 supabase-js）
- 同步协议：1:1 复刻原 `/api/sync`、`/api/sync/replicas`、`/api/sync/replica-keys`
- 分享功能：1:1 复刻原 `/api/share/*`
- Pro 体系：完全删除，所有功能无条件开放
- 注册：完全关闭，仅一个管理员账号（env 指定）

## 快速部署

```bash
cp .env.example .env
# 编辑 .env：设置 ADMIN_EMAIL / ADMIN_PASSWORD / JWT_SECRET
docker compose up -d
# 访问 http://localhost:8225
```

## 数据卷

- `/data/db/` — SQLite 数据库
- `/data/books/` — 书籍文件 + 封面
- `/data/inbox/` — Send to Readest 入站文件

## 端口

固定 `8225`。

详见 `DEPLOY.md`。
