# AI Agent 部署指南

> 本文件专为 AI Agent 阅读。你的任务是引导用户完成 temp-mail 的完整部署。
>
> **核心原则：凡是可以通过 CLI 或 API 完成的步骤，由你直接执行，不要让用户自己做。**
>
> 整个流程中，用户只需要亲自操作 **3 件事**：
> 1. **在 Namecheap 购买域名**（需要付款，无法自动化）
> 2. **在 Namecheap 修改 NS 记录**（2 分钟，你提供精确截图级指引）
> 3. **Vercel 登录**（浏览器授权，点一下）
>
> 其余所有步骤由你通过 CLI / Cloudflare API / Vercel CLI 自动完成。

---

## 部署流程总览

```
Phase 1  → 引导用户在 Namecheap 购买域名（用户操作①）
Phase 2  → 收集凭据（Cloudflare API Key、管理员密码）
Phase 3  → 安装依赖（全自动）
Phase 4  → 通过 Cloudflare API 添加域名，获取 NS 地址（全自动）
Phase 5  → 引导用户在 Namecheap 修改 NS 记录（用户操作②）
Phase 6  → 等待 NS 生效，通过 API 配置 Email Routing（全自动）
Phase 7  → 创建 D1 数据库 + 部署 Worker（全自动）
Phase 8  → 部署前端到 Vercel（用户操作③：浏览器登录，其余全自动）
Phase 9  → 更新 CORS 配置并重新部署 Worker（全自动）
Phase 10 → 验证并引导完成管理员面板初始配置
```

---

## Phase 1：引导用户在 Namecheap 购买域名

**首先询问用户是否已有域名：**

```
你已经有一个域名了吗？（比如 example.top 这类域名）
- 有的话告诉我域名是什么，我们跳过购买直接配置
- 没有的话我来引导你买一个，大约 $1–3 美元/年
```

**如果用户已有域名：** 将域名存入 `$DOMAIN`，询问注册商是哪家，跳到 Phase 2。

**如果用户没有域名：** 给出以下指引，等用户完成购买后再继续。

---

### Namecheap 购买域名步骤

```
好的，我们先去 Namecheap 买一个域名。步骤很简单：

第一步：打开 https://www.namecheap.com
        在首页大搜索框里输入你想要的名字，比如你的名字、项目名都可以
        （域名名字不重要，只是个地址，随便起）

第二步：在搜索结果页，从价格便宜的后缀里选一个：
        ✅ 推荐：.top（通常 $1–2/年）、.site、.online、.store（均 $1–5/年）
        ❌ 不推荐：.com（第一年便宜，但续费 $15+/年，没必要）

第三步：点击你选中的域名旁边的 "Add to cart"

第四步：点击右上角购物车图标 → 点击 "Checkout"
        如果没有 Namecheap 账号，这里会提示注册，注册一个即可（免费）

第五步：填写付款信息，完成购买

买完后告诉我你买的域名是什么（例如：mymail.top），我们继续下一步。
```

收到域名后存入 `$DOMAIN`，注册商记为 `namecheap`。

---

## Phase 2：收集凭据

域名已确认，现在一次性收集剩余必要信息：

```
很好！域名 $DOMAIN 已确认。现在需要两样东西：

1. Cloudflare 账号和 API Key
   （Cloudflare 负责接收你域名的邮件，免费账号就够）

   如果还没有 Cloudflare 账号：
   → 打开 https://dash.cloudflare.com/sign-up，注册一个免费账号

   有账号后获取 API Key：
   → 登录后打开：https://dash.cloudflare.com/profile/api-tokens
   → 页面拉到最底部，找到 "Global API Key"
   → 点击 "View"，输入你的 Cloudflare 登录密码
   → 复制显示出来的那串字符（40位左右）

   请告诉我：
   - Global API Key：（粘贴那串字符）
   - Cloudflare 账号邮箱：（你注册 Cloudflare 用的邮箱）

2. 管理员密码
   这是登录本系统管理后台的密码。
   输入"自动生成"让我帮你生成，或直接告诉我你想用的密码。
```

收到回答后：
- 如果选自动生成密码：生成一个 16 位强密码（含大小写字母、数字、符号），**醒目地展示给用户，明确要求他保存好，后面登录管理后台要用**。
- 将信息存入变量：`$CF_API_KEY`、`$CF_EMAIL`、`$ADMIN_PASSWORD`

---

## Phase 3：安装依赖（全自动）

```bash
# 前端依赖
npm install

# Worker 依赖
cd worker && npm install && cd ..
```

检查 wrangler：
```bash
npx wrangler --version
```

如果命令不存在：
```bash
npm install -g wrangler
```

---

## Phase 4：通过 Cloudflare API 添加域名（全自动）

有了 API Key，所有 Cloudflare 配置通过 API 完成，不需要用户登录 Cloudflare 网页。

### 4.1 获取 Account ID

```bash
curl -s "https://api.cloudflare.com/client/v4/accounts" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY"
```

解析响应中第一个账户的 `id`，存入 `$CF_ACCOUNT_ID`。

### 4.2 检查域名是否已在 Cloudflare

```bash
curl -s "https://api.cloudflare.com/client/v4/zones?name=$DOMAIN" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY"
```

- `result` 数组非空 → 域名已存在，取 `result[0].id` 存入 `$ZONE_ID`，跳到 4.4
- `result` 为空数组 → 执行 4.3

### 4.3 将域名添加到 Cloudflare

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/zones" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY" \
  -H "Content-Type: application/json" \
  --data "{\"name\":\"$DOMAIN\",\"account\":{\"id\":\"$CF_ACCOUNT_ID\"},\"jump_start\":false}"
```

从响应中提取：
- `result.id` → 存入 `$ZONE_ID`
- `result.name_servers[0]` → 存入 `$NS1`
- `result.name_servers[1]` → 存入 `$NS2`

### 4.4 确认 Nameserver 地址

如果是从已存在的 zone 拿到的 `$ZONE_ID`，再查一次确认 NS：

```bash
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_ID" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY"
```

从 `name_servers` 字段确认 `$NS1`、`$NS2`。

---

## Phase 5：引导用户在 Namecheap 修改 NS 记录

这是整个部署中唯一需要用户自己动手的配置操作，给出截图级精确指引：

```
现在需要你在 Namecheap 做一个设置，把域名交给 Cloudflare 管理。
操作大约 2 分钟，我会一步一步告诉你点哪里。

你的域名：$DOMAIN
需要填入的地址（Cloudflare 分配给你的）：
  第一个：$NS1
  第二个：$NS2

步骤如下：

① 打开这个链接（登录 Namecheap 后的域名管理页）：
   https://ap.www.namecheap.com/domains/list/

② 在列表里找到 $DOMAIN，点击它右边的橙色 "Manage" 按钮

③ 进入管理页后，找页面中间的 "NAMESERVERS" 那一行
   左边有一个下拉框，默认显示 "Namecheap BasicDNS"

④ 点开那个下拉框，选择最下面的 "Custom DNS"

⑤ 下拉框变成两个输入框：
   第一个输入框填：$NS1
   第二个输入框填：$NS2
   （如果只出现一个输入框，填完后点旁边的绿色 "+" 号加第二个）

⑥ 点击输入框右边的绿色对勾 ✓ 保存

保存成功后告诉我"完成了"，我来自动检测是否生效。
你不需要等待，告诉我后我会在后台持续检测。
```

**如果用户的域名不在 Namecheap（其他注册商）：**

- **Porkbun**：登录 → 点域名 Details → Nameservers → Edit → 填入 $NS1 和 $NS2 → 保存
- **GoDaddy**：My Products → 域名旁 DNS → Nameservers → Change → Enter my own → 填入 $NS1 和 $NS2
- **Dynadot**：My Domains → 点域名 → DNS Settings → Name Servers → Custom → 填入 $NS1 和 $NS2
- **其他**：找到"Nameservers"或"自定义 DNS"设置，填入上述两个地址

---

## Phase 6：等待 NS 生效，配置 Email Routing（全自动）

用户说"完成了"后立即开始轮询，不要让用户等待。

### 6.1 轮询 NS 是否切换到 Cloudflare

每隔 60 秒执行一次，直到成功：

```bash
dig NS $DOMAIN +short
```

输出包含 `cloudflare.com` → NS 已切换，继续。

同步通过 API 确认 zone 激活：

```bash
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_ID" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY" \
  | grep -o '"status":"[^"]*"'
```

`status` 为 `active` 时继续。

> NS 生效通常需要 10 分钟到 2 小时。期间可以继续和用户聊，或者告诉他去做别的事，生效后通知他。

### 6.2 开启 Email Routing

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/enable" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY" \
  -H "Content-Type: application/json"
```

Cloudflare 自动添加所需的 MX 和 SPF 记录，无需手动操作。

### 6.3 添加 Catch-all 规则（所有来信 → Worker）

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Catch-all to temp-mail-worker",
    "enabled": true,
    "matchers": [{"type": "all"}],
    "actions": [{"type": "worker", "value": ["temp-mail-worker"]}]
  }'
```

确认响应中 `success` 为 `true`。

---

## Phase 7：创建 D1 数据库并部署 Worker（全自动）

设置环境变量，让 wrangler 使用 API Key 认证，跳过浏览器登录：

```bash
export CLOUDFLARE_API_KEY="$CF_API_KEY"
export CLOUDFLARE_EMAIL="$CF_EMAIL"
export CLOUDFLARE_ACCOUNT_ID="$CF_ACCOUNT_ID"
```

### 7.1 创建 D1 数据库

```bash
cd worker && npx wrangler d1 create temp-mail-db
```

从输出解析 `database_id`（UUID 格式），存入 `$DATABASE_ID`。

如果报错"已存在"：
```bash
npx wrangler d1 list
```
找到 `temp-mail-db` 对应的 ID。

### 7.2 写入 Worker 配置

修改 `worker/wrangler.toml`（用实际值替换占位符）：

```toml
name = "temp-mail-worker"
main = "src/index.ts"
compatibility_date = "2024-12-01"
workers_dev = true
account_id = "$CF_ACCOUNT_ID"

[[d1_databases]]
binding = "DB"
database_name = "temp-mail-db"
database_id = "$DATABASE_ID"

[triggers]
crons = ["0 3 * * *"]

[vars]
ALLOWED_ORIGINS = "http://localhost:3000"
ADMIN_PASSWORD = "$ADMIN_PASSWORD"
```

### 7.3 初始化数据库表结构

```bash
npm run db:init
```

### 7.4 部署 Worker

```bash
npm run deploy
```

解析输出中的 Worker URL（格式：`https://temp-mail-worker.账号名.workers.dev`），存入 `$WORKER_URL`。

告知用户：`✅ 后端已部署完成。`

---

## Phase 8：部署前端到 Vercel

### 8.1 更新前端环境变量

修改项目根目录的 `.env.local`：

```env
NEXT_PUBLIC_WORKER_URL=$WORKER_URL
```

### 8.2 登录 Vercel

```bash
npm install -g vercel
vercel login
```

这会打开浏览器。告诉用户：

```
🌐 浏览器会打开 Vercel 登录页面，推荐用 GitHub 账号登录（点 "Continue with GitHub"），
授权完成后回到这里，我自动继续。
```

等用户确认登录完成。

### 8.3 部署到 Vercel

```bash
vercel --yes -e NEXT_PUBLIC_WORKER_URL="$WORKER_URL"
```

首次部署时 vercel 会提问，按以下方式回答：
- `Set up and deploy?` → Y
- `Which scope?` → 选择个人账号
- `Link to existing project?` → N
- `What's your project's name?` → temp-mail（或任意名称）
- `In which directory is your code located?` → ./（直接回车）

解析输出中的预览 URL，存入 `$VERCEL_PREVIEW_URL`。

### 8.4 发布到生产

```bash
vercel --prod --yes
```

解析生产 URL，存入 `$VERCEL_URL`。

---

## Phase 9：更新 CORS 并重新部署 Worker（全自动）

修改 `worker/wrangler.toml` 中的 `ALLOWED_ORIGINS`：

```toml
ALLOWED_ORIGINS = "$VERCEL_URL,$VERCEL_PREVIEW_URL,http://localhost:3000"
```

重新部署：

```bash
cd worker && npm run deploy
```

---

## Phase 10：验证并完成管理员面板配置

### 10.1 自动验证（全部通过才宣布成功）

```bash
# Worker API 正常返回
curl -s "$WORKER_URL/api/config"

# CORS 配置正确
curl -s -H "Origin: $VERCEL_URL" -I "$WORKER_URL/api/config" | grep "Access-Control-Allow-Origin"

# Email Routing MX 记录已生效
dig MX $DOMAIN +short
```

预期：
- 第一条返回包含 `domains` 字段的 JSON
- 第二条响应头包含 `Access-Control-Allow-Origin: $VERCEL_URL`
- 第三条输出包含 `cloudflare.net`

### 10.2 引导用户完成管理员面板配置

所有验证通过后：

```
🎉 部署全部完成！最后在管理后台做一个配置（2 分钟）：

1. 打开：$VERCEL_URL/admin
2. 输入管理员密码：$ADMIN_PASSWORD
3. 在"域名列表"里点击添加，输入：$DOMAIN，保存
4. 回到首页：$VERCEL_URL
5. 点"随机生成"得到一个临时邮箱地址
6. 用这个地址去任意网站注册（比如注册个论坛账号），看看邮件能不能收到

收到邮件 = 一切正常 ✅
没收到 = 告诉我，我来排查 🔍
```

---

## 故障排查

### 收不到邮件

```bash
# 1. Worker API 是否正常
curl "$WORKER_URL/api/config"

# 2. MX 记录是否指向 Cloudflare
dig MX $DOMAIN +short

# 3. 管理员面板域名是否已添加
curl -s "$WORKER_URL/api/config" | python3 -c "import sys,json; print(json.load(sys.stdin).get('domains'))"

# 4. Email Routing 规则是否已创建
curl -s "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "X-Auth-Email: $CF_EMAIL" \
  -H "X-Auth-Key: $CF_API_KEY"
```

### 前端报 CORS 错误

确认 `ALLOWED_ORIGINS` 包含 `$VERCEL_URL`，重新部署 Worker。

### Worker 部署失败

检查 `worker/wrangler.toml` 中 `account_id` 和 `database_id` 是否正确填写。

### Email Routing 报错 `zone not active`

NS 尚未生效，继续等待并轮询 zone 的 `status` 字段。

---

## 变量速查

| 变量 | 从哪里来 | 用于 |
|------|---------|------|
| `$DOMAIN` | Phase 1 用户提供 | zone 创建、管理员面板 |
| `$CF_API_KEY` | Phase 2 用户提供 | 所有 Cloudflare API 调用 |
| `$CF_EMAIL` | Phase 2 用户提供 | 所有 Cloudflare API 调用 |
| `$ADMIN_PASSWORD` | Phase 2 用户提供/生成 | wrangler.toml、告知用户 |
| `$CF_ACCOUNT_ID` | Phase 4.1 API 查询 | wrangler 认证、创建 zone |
| `$ZONE_ID` | Phase 4.2/4.3 API 返回 | Email Routing API |
| `$NS1` / `$NS2` | Phase 4.3/4.4 API 返回 | 告知用户填入 Namecheap |
| `$DATABASE_ID` | Phase 7.1 wrangler 输出 | wrangler.toml |
| `$WORKER_URL` | Phase 7.4 wrangler 输出 | .env.local、CORS 配置 |
| `$VERCEL_URL` | Phase 8.4 vercel 输出 | CORS 配置、告知用户 |
