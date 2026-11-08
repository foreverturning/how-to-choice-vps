# Lab 2 全栈小工具：0 元跑通前端、函数与数据库

> 做完你会得到：一个带 PostgreSQL 数据库的书签小工具，前端页面、Serverless 函数、数据库三层都在线上；一条可复现的“建库 → 建表 → 配环境变量 → 写接口 → 部署 → 验证”流程；一份写清免费额度、升级价格与备份位置的记录。
> 对应总纲：Lab 2 ｜ 对应文章：[02 算账篇](../chapters/02-算账篇.md) 的 2.3、[03 白嫖篇](../chapters/03-白嫖篇.md) 的 3.4 ｜ 预计耗时：90–150 分钟（不含平台排队与冷启动等待）

---

## 目标

用一个真实的小工具，把“带后端”的项目完整跑一遍。工具本身只做两件事：**列书签**、**加书签**。功能越少越好，因为本 Lab 要验证的不是业务，而是三层能不能接上、钱会不会超、密钥会不会漏。

技术选型如下，全部用通用组件，换成同类产品也成立：

| 层 | 本 Lab 的选择 | 作用 | 对应免费额度 |
|---|---|---|---|
| 前端 | Next.js（App Router）+ React | 页面与构建产物 | 前端托管 Hobby 类：100GB/月、100 万次函数调用/月（快照 P-023，2026-11） |
| 函数 | 同仓库的 `app/api/bookmarks/route.ts` | 按请求运行、不常驻的接口（Serverless 函数） | 计入上面的函数调用次数 |
| 数据库 | 托管 PostgreSQL，标准 SQL | 存书签，验证建表、连接、权限 | 免费档 2 个项目 / 500MB（快照 P-024，2026-11）；另一类免费档 0.5GB/项目（快照 P-025，2026-11） |

完成后你要能回答三个问题：

1. **为什么这条链路是 0 元？** 因为前端是静态文件、函数按请求计费、数据库在免费额度内，三者都没有“整月开着一台机器”的成本。
2. **哪里会突然花钱？** 商用要升级（Hobby 禁止商用，快照 P-023）；数据库超 500MB 或要更高并发要升级（快照 P-024）；数据库按量档约 $0.35/GB·月（快照 P-025，2026-11，待核对），对象存储与出网费用另计（待核对）。
3. **免费层和真服务器的差别在哪？** 休眠、冷启动、调用次数上限、连接数上限。这四条决定了它适合验证期，不适合直接当生产环境。

> 本 Lab 的定位是“自用、内测、演示”。一旦你要向用户收费、放广告、给客户交付，先读 03 篇的 3.6 与 02 篇的 2.3，再决定升级还是换轻量服务器。

## 前置条件

| 项目 | 要求 | 说明 |
|---|---|---|
| Node.js | 20 及以上，附 npm | Next.js 与构建命令都依赖它 |
| Git | 任意近期版本 | 用来触发平台自动部署 |
| 代码托管账号 | 一个（GitHub 类） | 建议单独注册，不复用主账号 |
| 前端托管账号 | 一个，Hobby 类免费档 | 要支持 Serverless 函数，不只是静态托管 |
| 数据库账号 | 一个，托管 PostgreSQL 免费档 | 能拿到连接串、能在控制台执行 SQL |
| `psql` | PostgreSQL 客户端 15 及以上 | 建表、插数据、验证权限都要用 |
| 终端 | bash / zsh / PowerShell 任一 | 下文以 bash 为主，PowerShell 差异处已注明 |
| 密码管理器 | 一个 | 数据库密码、连接串只放这里，不进仓库 |

开始前先做一件事：把项目目录里所有 `.env`、连接串、密钥找出来，确认它们不会被提交。这一步比后面任何一步都重要。

### 先看清免费额度与限制

这张表回答“我到底能免费用多少、什么时候会断”。读法：先看额度，再看限制，最后看升级价；四个编号都来自统一快照，价格会变，下单前以控制台为准。

| 资源 | 免费额度 | 关键限制 | 超限或升级后 | 快照 |
|---|---|---|---|---|
| 前端托管（Hobby 类） | 流量 100GB/月；函数调用 100 万次/月 | **禁止商用**；超量限流，不自动扣费 | Pro 约 $20/月，按 7.2 折算约 144 元/月 | P-023，2026-11 |
| Serverless 数据库 A | 2 个项目；每库 500MB | 闲置 7 天休眠，唤醒有等待 | Pro 约 $25/月，按 7.2 折算约 180 元/月 | P-024，2026-11 |
| Serverless 数据库 B | 0.5GB/项目 | 冷启动明显，适合 Lab 不适合生产 | 按量约 $0.35/GB·月 | P-025，2026-11，待核对 |
| 应用托管（Free Web Service 类） | 750 小时/月 | 15 分钟无请求休眠 | 约 $7/月起，按 7.2 折算约 50 元/月 | P-026，2026-11 |

两个容易算错的点：

- **750 小时不是“无限常驻”。** 31 天 × 24 小时 = 744 小时，只够**一个**实例整月常驻；开两个服务，或部署时新旧实例短暂并存，就会超出。
- **两种数据库免费档不要混着算。** 2 个项目 / 500MB 与 0.5GB/项目是两类产品的口径，按你实际选的那一款记录，不要相加。

## 步骤

### 步骤 1｜把需求砍到最小

- **做什么**：只保留“列出书签”和“新增书签”。不做注册、不做上传、不做支付、不做分享。数据库只设计一张表。
- **在哪做**：纸上或编辑器里的 `README.md`。
- **成功标志**：你能用一句话说清：“一个页面、两个接口、一张四字段的表。”

### 步骤 2｜初始化项目并锁定依赖

- **做什么**：用脚手架创建 Next.js 项目，安装 PostgreSQL 驱动 `pg` 与类型声明，确认 `package-lock.json` 已生成。
- **在哪做**：本机工作目录。
- **成功标志**：`npm run dev` 能启动，浏览器打开 `http://localhost:3000` 有默认页面；`package-lock.json` 在项目根目录。

### 步骤 3｜创建数据库项目（会产生费用）

> ⚠️ 会产生费用（仅当你超出免费额度或主动选择付费档时）：免费档 2 个项目 / 500MB 为 0 元（快照 P-024，2026-11）；Pro 约 $25/月，约 180 元/月。选区域、选规格时看清是否标了“付费”，不要顺手点升级。

- **做什么**：登录数据库控制台，新建一个 PostgreSQL 项目。区域选离主要用户近的；项目名写 `bookmark-lab`；建库账号的密码用密码管理器生成 24 位以上随机串。
- **在哪做**：数据库平台控制台。
- **成功标志**：项目状态变成“运行中 / Healthy”；你能看到主机名、端口、库名、用户名；密码只存在密码管理器里。

### 步骤 4｜取两种连接串，先分清用途

- **做什么**：在控制台分别复制**连接池串**与**直连串**。前者给线上函数用，后者只给本地建表和迁移用。两者通常只差端口：连接池常为 `6543`，直连常为 `5432`，以控制台标注为准。
- **在哪做**：数据库控制台的“连接 / Connect”页。
- **成功标志**：两条串都带 `sslmode=require`；你能说清哪条进线上环境变量、哪条只留在本地。

连接串就是“用户名 + 密码 + 主机 + 端口 + 库名”，等于数据库的钥匙。不要截图发群、不要贴进 issue、不要写进代码注释。

### 步骤 5｜建表、建最小权限角色、开启 RLS

- **做什么**：写两个 SQL 文件：`db/schema.sql` 负责建表，`db/security.sql` 负责建 API 专用角色、授权、开启行级安全（RLS，Row Level Security）。先用直连串执行，再插两条示例数据。
- **在哪做**：本机项目根目录 + `psql`。
- **成功标志**：`\d+ public.bookmarks` 能看到 4 个字段；`select count(*)` 返回 2；`relrowsecurity` 为 `t`。

建表只保留 4 个字段，够用且不容易写错：

| 字段 | 类型 | 约束 | 说明 |
|---|---|---|---|
| `id` | `bigint` | 主键，自增 | 接口返回的稳定标识 |
| `title` | `text` | 非空，1–80 字 | 书签标题 |
| `url` | `text` | 非空，`http(s)://` 开头 | 目标地址 |
| `created_at` | `timestamptz` | 默认 `now()` | 排序与排查用 |

不要用建库的超级用户跑线上 API。单独建一个只能 `select` / `insert` 的角色，即使连接串泄露，对方也删不了表、改不了结构。

### 步骤 6｜配置本地环境变量

- **做什么**：创建 `.env.local`，写入运行时连接串（最小权限角色 + 连接池端口）与本地站点地址；把 `.env*` 加进 `.gitignore`；另建一份不含真实值的 `.env.example` 提交到仓库。
- **在哪做**：本机项目根目录。
- **成功标志**：`git status` 里看不到 `.env.local`；重启 `npm run dev` 后，接口能读到变量；仓库里只有占位符。

变量名不要带 `NEXT_PUBLIC_` 或 `VITE_` 前缀。带这两个前缀的变量会被打进前端产物，等于公开。数据库连接串、Service Role Key 这类值，只放服务端变量。

### 步骤 7｜写一个 API 路由

- **做什么**：新建 `app/api/bookmarks/route.ts`，实现 `GET`（列最近 50 条）与 `POST`（新增一条）。连接池放在模块作用域，`max` 设为 3；校验 `title` 长度与 `url` 协议；出错只返回 `{ "error": "db_unavailable" }`，不返回堆栈。
- **在哪做**：本机编辑器。
- **成功标志**：保存后开发服务器自动编译，终端没有红色报错；浏览器访问 `/api/bookmarks` 能看到 JSON 数组。

`export const dynamic = "force-dynamic"` 这一行不要省。它告诉构建器“这个路由每次请求都执行”，避免构建时把数据库结果静态烤进产物。

### 步骤 8｜本地联调：先 GET，再 POST，最后打一次错误输入

- **做什么**：用 `curl` 依次请求列表、新增、非法输入。新增成功后回数据库查一次 `count(*)`，确认真的落库。
- **在哪做**：本机终端 + `psql`。
- **成功标志**：GET 返回 200 与数组；POST 返回 201 与新记录；空标题或非 `http` 地址返回 400；数据库条数从 2 变成 3。

### 步骤 9｜构建并检查“构建阶段不碰数据库”

- **做什么**：执行 `npm run build`，再用 `npm run start` 起生产模式，重复一次 GET。观察构建日志里有没有数据库连接报错。
- **在哪做**：本机项目根目录。
- **成功标志**：构建成功；生产模式 GET 仍返回 200；构建日志里没有“连接超时 / 环境变量缺失”。如果构建阶段就报数据库错误，说明某个页面在构建时查询了数据库，改成动态渲染或移到客户端。

### 步骤 10｜部署到前端托管（会产生费用）

> ⚠️ 会产生费用（仅当你超出免费额度或升级 Pro 时）：Hobby 类免费档为 0 元，含 100GB/月流量与 100 万次函数调用/月（快照 P-023，2026-11）；Pro 约 $20/月，约 144 元/月。免费档禁止商用。

- **做什么**：把代码推到代码托管平台；在托管平台“从 Git 仓库导入”；构建命令填 `npm run build`；Node 版本选 20；在环境变量页添加 `DATABASE_URL`（连接池串）与 `APP_ORIGIN`（线上域名），Production 与 Preview 都加。
- **在哪做**：代码托管平台 + 前端托管控制台。
- **成功标志**：部署日志最后一行是成功；拿到 `https://<你的项目域名>`；`curl -I` 返回 200。

不要把 `DATABASE_URL_DIRECT` 配到线上运行时。迁移用的高权限连接串只留在本地，线上函数只需要最小权限的 `DATABASE_URL`。

### 步骤 11｜线上初始化数据库

- **做什么**：对线上库执行一次 `db/schema.sql` 与 `db/security.sql`，再插入两条示例数据。若本地无法直连线上库，就用平台控制台的 SQL 编辑器按同样顺序执行。
- **在哪做**：本机终端（直连串）或数据库控制台 SQL 页。
- **成功标志**：线上 `select count(*)` 返回 2；`relrowsecurity` 为 `t`；`app_writer` 角色存在且只有 `select` / `insert` 权限。

`schema.sql` 用 `create table if not exists`，重复执行不会删数据。但 `security.sql` 里的 `create role` 只能跑一次；重跑前先把那一行注释掉，否则会报“角色已存在”。

### 步骤 12｜验证权限边界

- **做什么**：用最小权限连接串执行一次 `select`（应成功）、一次 `create table`（应失败）；再查 `pg_class.relrowsecurity`（应为 `t`）。如果平台提供 `anon` / `authenticated` 角色，确认它们即使有表权限，也被 RLS 挡住：用该角色直查应返回空或报错。
- **在哪做**：本机终端 + `psql`。
- **成功标志**：查询成功；建表被拒绝并提示 `permission denied`；RLS 标志为真。失败是预期结果，不是配置坏了。

注意：RLS 对表的所有者（owner）不生效。用建库账号跑 API，RLS 等于没开；必须用 `app_writer` 这类独立角色。

### 步骤 13｜线上 API 验证

- **做什么**：用 `curl -i` 请求线上 GET，再用 POST 新增一条，最后再 GET 一次确认能看到。检查状态码、`content-type`、`cache-control`，并确认错误响应里没有堆栈和连接串。
- **在哪做**：本机终端。
- **成功标志**：GET 200、POST 201、JSON 可解析；响应头含 `cache-control: no-store`；故意传坏数据返回 400，且错误信息只有错误码。

第一次请求可能明显慢，这是函数冷启动或数据库唤醒，不是接口写错。连续请求第二次再判断性能。

### 步骤 14｜登记额度、设告警、导出备份（会产生费用）

> ⚠️ 会产生费用（当你把备份放到对象存储、或跨区下载时）：数据库超出免费 0.5GB 后按量约 $0.35/GB·月（快照 P-025，2026-11，待核对）；对象存储的存储与出网费用另计（待核对）。免费额度内、备份留在本地时不产生费用。

- **做什么**：把套餐、月额度、重置周期、休眠规则、升级价格写进一张额度表；在平台账单页设 0 元、1 元/天、30 元/月三道告警；用 `pg_dump` 导出一份备份，放到本地加密目录，并写进到期日历。
- **在哪做**：平台控制台 + 本机终端 + 日历工具。
- **成功标志**：额度表每一格都有值；告警测试消息能收到；备份文件里能搜到 `CREATE TABLE public.bookmarks`；日历里有“每月 1 日看用量、每季度恢复演练”的提醒。

## 命令

### 1. 初始化项目

```bash
npx create-next-app@latest bookmark-lab --typescript --app --eslint --no-tailwind --use-npm
cd bookmark-lab
npm i pg
npm i -D @types/pg
```

### 2. 最小表结构与权限脚本

`db/schema.sql`：

```sql
create table if not exists public.bookmarks (
  id bigint generated by default as identity primary key,
  title text not null check (char_length(title) between 1 and 80),
  url text not null check (url ~ '^https?://'),
  created_at timestamptz not null default now()
);
```

`db/security.sql`：

```sql
-- 只跑一次；重跑前注释掉 create role 这一行
create role app_writer login password :'app_password';

grant usage on schema public to app_writer;
grant select, insert on public.bookmarks to app_writer;

alter table public.bookmarks enable row level security;

create policy bookmarks_read on public.bookmarks
  for select to app_writer using (true);

create policy bookmarks_insert on public.bookmarks
  for insert to app_writer with check (char_length(title) between 1 and 80);

revoke all on public.bookmarks from public;

-- 若平台自带 anon / authenticated 角色，取消下一行注释
-- revoke all on public.bookmarks from anon, authenticated;
```

### 3. 建表、建角色、插数据

```bash
export DATABASE_URL_DIRECT='postgresql://<建库账号>:<密码>@<主机>:5432/<库名>?sslmode=require'

psql "$DATABASE_URL_DIRECT" -v ON_ERROR_STOP=1 -f db/schema.sql
psql "$DATABASE_URL_DIRECT" -v ON_ERROR_STOP=1 \
  -v app_password='<强密码>' -f db/security.sql

psql "$DATABASE_URL_DIRECT" -v ON_ERROR_STOP=1 -c "
insert into public.bookmarks (title, url) values
  ('价格快照（示例）', 'https://example.com/pricing'),
  ('写作规范（示例）', 'https://example.com/spec');
"
```

PowerShell 下环境变量写法不同：

```powershell
$env:DATABASE_URL_DIRECT = "postgresql://<建库账号>:<密码>@<主机>:5432/<库名>?sslmode=require"
psql $env:DATABASE_URL_DIRECT -v ON_ERROR_STOP=1 -f db/schema.sql
```

### 4. 验证表结构与数据

```bash
psql "$DATABASE_URL_DIRECT" -c "\d+ public.bookmarks"
psql "$DATABASE_URL_DIRECT" -c "select id, title, url, created_at from public.bookmarks order by id;"
psql "$DATABASE_URL_DIRECT" -c "select relrowsecurity from pg_class where oid = 'public.bookmarks'::regclass;"
```

### 5. 本地环境变量

`.env.local`（不要提交）：

```ini
# 运行时：最小权限角色 + 连接池端口
DATABASE_URL=postgresql://app_writer:<强密码>@<主机>:6543/<库名>?sslmode=require
# 仅本地迁移用：建库账号直连，不要配到线上运行时
DATABASE_URL_DIRECT=postgresql://<建库账号>:<密码>@<主机>:5432/<库名>?sslmode=require
APP_ORIGIN=http://localhost:3000
```

`.env.example`（可以提交，只放占位符）：

```ini
DATABASE_URL=postgresql://app_writer:<强密码>@<主机>:6543/<库名>?sslmode=require
APP_ORIGIN=http://localhost:3000
```

```bash
printf 'node_modules/\n.next/\n.env\n.env.*\n!.env.example\n' >> .gitignore
```

PowerShell 下改用：

```powershell
"node_modules/`n.next/`n.env`n.env.*`n!.env.example" | Out-File -Encoding utf8 -Append .gitignore
```

### 6. API 路由

`app/api/bookmarks/route.ts`：

```ts
import { NextResponse } from "next/server";
import { Pool } from "pg";

export const runtime = "nodejs";
export const dynamic = "force-dynamic";

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 3,
  idleTimeoutMillis: 10_000,
  connectionTimeoutMillis: 5_000,
});

export async function GET() {
  try {
    const { rows } = await pool.query(
      "select id, title, url, created_at from public.bookmarks order by id desc limit 50"
    );
    return NextResponse.json(rows, {
      headers: { "cache-control": "no-store" },
    });
  } catch (err) {
    console.error("GET /api/bookmarks failed", err);
    return NextResponse.json({ error: "db_unavailable" }, { status: 503 });
  }
}

export async function POST(req: Request) {
  const body = await req.json().catch(() => null);
  const title = typeof body?.title === "string" ? body.title.trim() : "";
  const url = typeof body?.url === "string" ? body.url.trim() : "";

  if (title.length < 1 || title.length > 80 || !/^https?:\/\//.test(url)) {
    return NextResponse.json({ error: "invalid_input" }, { status: 400 });
  }

  try {
    const { rows } = await pool.query(
      "insert into public.bookmarks (title, url) values ($1, $2) returning id, title, url, created_at",
      [title, url]
    );
    return NextResponse.json(rows[0], { status: 201 });
  } catch (err) {
    console.error("POST /api/bookmarks failed", err);
    return NextResponse.json({ error: "db_unavailable" }, { status: 503 });
  }
}
```

### 7. 最小前端页面

`app/page.tsx`：

```tsx
"use client";

import { useEffect, useState } from "react";

// pg 默认把 bigint 读成字符串，避免大整数精度丢失
type Bookmark = { id: string; title: string; url: string; created_at: string };

export default function Page() {
  const [items, setItems] = useState<Bookmark[]>([]);

  useEffect(() => {
    fetch("/api/bookmarks")
      .then((r) => (r.ok ? r.json() : Promise.reject(r.status)))
      .then(setItems)
      .catch(() => setItems([]));
  }, []);

  return (
    <main style={{ padding: 24 }}>
      <h1>书签</h1>
      <ul>
        {items.map((b) => (
          <li key={b.id}>
            <a href={b.url}>{b.title}</a>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

### 8. 本地开发与构建

```bash
npm run dev
npm run build
npm run start
```

### 9. 本地验证 API

```bash
curl -s http://localhost:3000/api/bookmarks
curl -s -X POST http://localhost:3000/api/bookmarks \
  -H "content-type: application/json" \
  -d '{"title":"本地测试","url":"https://example.com/local"}'
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3000/api/bookmarks \
  -H "content-type: application/json" \
  -d '{"title":"","url":"not-a-url"}'
```

### 10. 部署

```bash
git add .
git commit -m "feat: bookmarks api"
git push
```

推送后在托管平台导入仓库，构建命令填 `npm run build`，Node 版本选 20，环境变量只加 `DATABASE_URL` 与 `APP_ORIGIN`。

### 11. 线上验证

```bash
curl -i https://<你的项目域名>/api/bookmarks
curl -s -X POST https://<你的项目域名>/api/bookmarks \
  -H "content-type: application/json" \
  -d '{"title":"线上第一条","url":"https://example.com/first"}'
curl -s https://<你的项目域名>/api/bookmarks | head -c 500
```

### 12. 权限检查

```bash
psql "$DATABASE_URL" -c "select count(*) from public.bookmarks;"
psql "$DATABASE_URL" -c "create table should_fail(id int);"
```

第二条应返回 `permission denied`。如果它成功了，说明 `DATABASE_URL` 用的是建库账号，不是 `app_writer`。

### 13. 备份

```bash
pg_dump "$DATABASE_URL_DIRECT" --no-owner --no-privileges -f backup-2026-11.sql
grep -c "CREATE TABLE" backup-2026-11.sql
```

## 产出

走完这 14 步，你手上应该有六样东西：

| 产出 | 验收标准 |
|---|---|
| 线上页面 | `https://<你的项目域名>` 能打开，列表能显示数据库里的书签 |
| API 地址 | `GET /api/bookmarks` 返回 200；`POST` 返回 201；坏输入返回 400 |
| 数据表 | `public.bookmarks` 有 4 个字段，至少有 2 条记录 |
| 权限边界 | RLS 为开启状态；`app_writer` 只能 `select` / `insert`；建表被拒绝 |
| 部署记录 | 写清构建命令、Node 版本、环境变量名（不写值）、部署触发方式 |
| 额度与备份 | 一张额度表 + 一份可读的 `.sql` 备份 + 日历提醒 |

额度表按这个格式填（示例行仅示意，数字按你选的产品抄）：

| 资源 | 免费额度 | 重置周期 | 休眠或限流 | 升级价格 | 备份位置 |
|---|---|---|---|---|---|
| 前端托管 | 100GB/月；100 万次函数调用/月 | 每月 | 超量限流 | Pro 约 $20/月，约 144 元/月 | Git 仓库 |
| 数据库 | 2 个项目 / 500MB | 不重置，按占用 | 闲置 7 天休眠 | Pro 约 $25/月，约 180 元/月 | 本地加密盘 + 对象存储 |
| 应用托管（如选用） | 750 小时/月 | 每月 | 15 分钟无请求休眠 | 约 $7/月起，约 50 元/月 | Git 仓库 |

## 常见问题

### 1. 报错 `too many clients already`，连接池耗尽

这是 Serverless + 数据库最典型的组合问题。函数按请求扩实例，每个实例各开一个连接池；并发 20 个实例、每个 `max: 3`，就是 60 条连接，免费档的数据库连接数很快见底。

按顺序改四件事：

1. **运行时一定用连接池串**（通常是 `6543` 端口），不要用直连串（`5432`）。连接池会把短连接复用到少量真实连接上。
2. **把 `max` 降到 1–3**。Serverless 里 `max: 10` 不是“性能更好”，是“更快把库打死”。
3. **`new Pool()` 放在模块顶层**，不要写在处理函数里。放在函数里等于每个请求新建一个池，连接只增不减。
4. **不要在事务里跨 `await` 做外部请求**。连接被占着等网络，池很快耗尽。

改完还报错，就去数据库控制台看当前连接数，确认有没有遗留的 `psql` 会话；免费档的连接上限很低，一条忘记关的会话也会占坑。

### 2. 第一次访问很慢，第二次就正常

这是冷启动，不是接口写错。两种情况会叠加：函数实例被回收后要重新启动（通常几百毫秒到几秒），数据库休眠被唤醒还要再等（几秒到几十秒）。

处理方式：

- 演示或验收前，先手动请求一次 `/api/bookmarks` 预热，等它返回再打开页面。
- 缩小函数依赖，别在模块顶层做重活（读大文件、初始化 SDK、跑迁移）。
- 数据库连接用惰性初始化，避免冷启动时同时做“启动函数 + 建连接”两件事。
- 如果真实用户会碰到冷启动，说明该升级或换常驻方案了；这不是靠改代码能消除的。

### 3. 数据库休眠 / 应用休眠后被唤醒，第一次请求 500 或 502

先分清两个休眠：数据库闲置 7 天休眠（快照 P-024，2026-11）；应用托管 15 分钟无请求休眠（快照 P-026，2026-11）。前者唤醒慢，后者更频繁。

看到 500 / 502 / 504 时，不要立刻改代码。先重试一次，多数情况下第二次就成功；再看平台日志，确认是“数据库连接超时”还是“函数超时”。如果是演示场景，提前预热即可；如果是生产场景，升级到不休眠的档位，或改用常驻服务。

有人会用定时请求“保活”。可以，但要先读条款：保活本身消耗函数调用与流量额度，也可能被平台视为异常流量；它解决的是演示可用性，不是生产可靠性。

### 4. 环境变量明明配了，线上还是读不到

按这个顺序排查，基本一次命中：

1. **配错环境**：平台通常分 Production / Preview / Development，三套变量不互通。你部署的是 Production，却只配了 Preview，就会读不到。
2. **改完没重新部署**：多数平台的环境变量在部署时注入，改完要触发一次新部署才生效。
3. **变量名写错**：`DATABASE_URL` 不是 `DATABASE_URI`，大小写也要一致。
4. **用了公开前缀**：`NEXT_PUBLIC_` / `VITE_` 开头的变量会被打进前端，服务端密钥不能用这种名字；反过来，前端要用的公开配置才需要这个前缀。
5. **本地与线上混了**：`.env.local` 只对本地生效，线上必须在平台控制台单独配；本地改完要重启 `npm run dev`。

最快的确认方式是在服务端日志里打印 `Boolean(process.env.DATABASE_URL)`，只打印布尔值，不打印内容。返回 `false` 就是没注入，不用再怀疑代码。

### 5. 浏览器报 CORS，但 `curl` 是正常的

CORS 是浏览器行为，`curl` 不执行同源策略，所以“curl 正常、浏览器报错”很常见。

如果前端和 API 同域（本 Lab 就是），本不该有 CORS 问题；出现报错，先看请求地址是不是写成了另一个域名或 `http://`。如果前端确实在另一个域名，API 要返回：

```text
Access-Control-Allow-Origin: https://<你的前端域名>
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: content-type
Vary: Origin
```

并且处理 `OPTIONS` 预检，返回 204。不要图省事写 `*`：带 Cookie 的请求不允许 `*`，而且 `*` 等于允许任何站点调用你的写接口。上线后按域名精确白名单，别用通配。

### 6. 免费额度超了会怎样，会不会悄悄扣钱

多数免费 PaaS 的处理是**限流或要求升级**，不是直接扣费；但不同产品不一样，必须看你选的那一款条款。已知口径：

- 前端托管 Hobby：100GB/月、100 万次函数调用/月，超量限流；免费档禁止商用（快照 P-023，2026-11）。
- 数据库：2 个项目 / 500MB（快照 P-024）；另一类 0.5GB/项目（快照 P-025）。写满后表现是写入失败或报错，不会自动帮你扩容。
- 应用托管：750 小时/月（快照 P-026）。按 31 天算是 744 小时，只够一个实例常驻。

对策不是“省着点用”，而是三件事：**设 0 元与日预算告警、给列表接口分页、给响应加缓存**。一次被爬、一次把整库数据塞进一个 JSON，就能把月额度打穿。

### 7. 部署成功，但线上接口一直 500

按“日志 → 变量 → 连接 → 表 → 权限”的顺序查，不要跳步：

1. **函数日志**：第一行报错通常就写明是 `ECONNREFUSED`、`timeout`、`password authentication failed` 还是 `relation "bookmarks" does not exist`。
2. **环境变量**：确认 Production 环境有 `DATABASE_URL`，且值是连接池串。
3. **连接**：确认串里带 `sslmode=require`；确认数据库没有 IP 白名单把你挡在外面。
4. **表**：线上库有没有跑过 `schema.sql`；`search_path` 下能不能找到 `public.bookmarks`。
5. **权限**：`app_writer` 有没有 `select` / `insert`；RLS 有没有策略；是不是误用了建库账号。

502 多半是函数崩溃或超时，503 多半是数据库不可用，500 多半是代码异常。先把状态码和日志对上，再改代码。

## 安全提示

1. **Service Role Key 绝不能进前端。** 它是绕过 RLS 的“总钥匙”，一旦出现在 `NEXT_PUBLIC_`、`VITE_`、HTML 或浏览器请求里，等于把数据库交给所有人。只放服务端环境变量；如果曾经提交过、截图过、发过群，按已泄露处理，立刻轮换。
2. **开启 RLS，并用最小权限角色跑 API。** RLS 默认拒绝，按需加策略；API 用 `app_writer` 这类只能读写一张表的角色，不要用建库账号。表 owner 会绕过 RLS，这一点最容易让“我开了 RLS”变成自我安慰。
3. **密钥轮换要写成动作，不是口号。** 数据库密码、Service Role Key、平台访问令牌，建议 90 天换一次；发生泄露、离职、设备丢失时立即换。换完更新平台环境变量并重新部署，再确认旧连接已断开。
4. **不要用主账号。** 数据库、托管、代码托管各用独立邮箱与独立项目；开 MFA；支付方式用限额卡；只授目标仓库或目标项目的权限，不给组织管理员。这个 Lab 的数据和密钥，不应该和你的主业务放在同一个账号下。
5. **备份导出要能恢复，不是只导出。** 上线前、改表前、每月固定时间各做一次 `pg_dump`；备份加密后放本地和另一处存储；不要提交到仓库。每季度拿备份恢复到临时库一次，确认不是空文件。
6. **输入校验与限流一起做。** 公开的 `POST` 接口会被人刷；校验 `title` 长度与 `url` 协议只是第一道，后面还要加请求体大小限制、按 IP 限流、必要时加人机验证。否则 100 万次调用额度可能在一夜之间被刷完。
7. **日志里不写敏感值。** 连接串、密码、完整请求头、用户 token 都不要进日志；错误响应只给错误码，堆栈留在服务端。你排查时想看的内容，攻击者也想看。

## 完成后你应该得到什么

- 一个 `https://` 开头的全栈小工具：页面能列出书签，接口能新增书签，数据真的落在 PostgreSQL 里。
- 一条可复现的链路：建库 → 建表 → 建最小权限角色 → 配环境变量 → 写函数 → 本地联调 → 构建 → 部署 → 线上验证。换一个“待办 / 记账 / 收藏”项目，照这 14 步再走一遍即可。
- 一张权限与额度表：写清了 RLS 状态、API 角色权限、月额度、休眠规则、升级价格。这张表就是 03 篇“到期日历”的第一行。
- 一份可用的备份：`pg_dump` 出来的 `.sql` 文件，能搜到建表语句，能在临时库恢复。
- 一个成本结论：在免费额度内，这个项目的月成本是 0 元；要商用、要去掉休眠、要更大数据库，再按快照 P-023～P-026 升级。把它和 02 篇场景 B 的轻量方案（99–408 元/年，快照 P-004、P-011、P-013）放在一起比，你会看到“免费 PaaS 省的是钱，换掉的是稳定性和商用授权”。

如果这一步走通了，下一步可以做两件事：把页面做得像样一点，回到 [Lab 1 静态作品集](./Lab1-静态作品集.md) 补部署与域名；或者认真算一遍“免费层 vs 轻量服务器”的账，读 [02 算账篇](../chapters/02-算账篇.md) 的 2.3 与 [05 成本解剖篇](../chapters/05-成本解剖篇.md) 的 5.6。

## 免责声明

> 本文价格为 2026-11 的公开信息快照与量级估算，会随厂商调价、汇率、地区与配置变化而不同；文中不对低价机、转卖机与中小商家点名，引用头部公有云公开定价或财报时仅作出处，不构成购买推荐或收益承诺。实际下单前请以厂商官网、合同与工单答复为准。
