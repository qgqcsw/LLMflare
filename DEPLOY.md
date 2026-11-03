# 手动部署说明（LLMflare on Cloudflare Workers）

本文件只描述部署步骤，仓库代码保持与上游完全一致，未做任何修改。

> 前提：本仓库是从 [Jiao77/LLMflare](https://github.com/Jiao77/LLMflare) fork 而来，上游仓库中只有 `wrangler.example.jsonc`（含占位符 ID），没有可直接使用的 `wrangler.jsonc`。因此下面两种方式都会在**构建阶段**由示例文件生成配置文件，不向仓库提交任何新代码。

---

## 准备工作（两种方式都需要）

| 资源 | 名称 | 绑定名 | 说明 |
| --- | --- | --- | --- |
| KV Namespace | 任意，建议 `llmflare-config` | `CONFIG` | 保存 Provider 配置与 API Key |
| D1 Database | 任意，建议 `llmflare-usage` | `USAGE` | 用量统计，需执行迁移 |
| Secret | `ADMIN_TOKEN` | — | 后台管理员登录凭据，必须足够长且随机 |

单个 Worker 的请求地址为 `https://<worker名>.<子域>.workers.dev`，也可以后续绑定自有域名。

---

## 方式一：Cloudflare 控制台导入仓库（全程不用命令行）

适合不希望在本地安装任何工具的情况。Cloudflare 会在你推送代码时自动构建并部署，之后同步上游更新只需在 GitHub 上点一次 `Sync fork`。

### 1. 创建 KV Namespace

控制台左侧 **Storage & Databases → KV** → **Create instance**，名称填 `llmflare-config`。
创建完成后进入该 namespace，在详情页复制 **Namespace ID**（一串 32 位十六进制字符）。

### 2. 创建 D1 Database

左侧 **Storage & Databases → D1 SQL Database** → **Create database**，名称填 `llmflare-usage`。
进入数据库详情页，复制 **Database ID**（UUID 格式）。

> 注意：数据库名称可以自定义，但上表中的**绑定名** `USAGE` 与 `CONFIG` 不要改，仓库里的代码就是按这两个名字读取的。

### 3. 导入仓库

**Workers & Pages** → **Create** → **Import a repository**（或 Workers 页签下的 **Connect to Git**）→ 授权 GitHub 并选择你 fork 的仓库 `qgqcsw/LLMflare`。

### 4. 填写构建配置

| 字段 | 值 |
| --- | --- |
| 项目名称 | `llmflare`（自定义，会作为 Worker 名称） |
| 生产分支 | `main` |
| 根目录 | `/`（留空） |
| 构建命令 | 见下方 |
| 部署命令 | 见下方 |

构建命令（一行，作用是把示例配置转成真实配置）：

```bash
sed -e "s/REPLACE_WITH_KV_NAMESPACE_ID/${KV_NAMESPACE_ID}/" -e "s/REPLACE_WITH_D1_DATABASE_ID/${D1_DATABASE_ID}/" wrangler.example.jsonc > wrangler.jsonc
```

部署命令（先跑数据库迁移，再部署）：

```bash
npm run sync:cf-models && npx wrangler d1 migrations apply USAGE --remote && npx wrangler deploy
```

### 5. 设置环境变量与密钥

进入项目的 **Settings → Variables and Secrets**（构建配置页也可填写），新增三项：

| 名称 | 类型 | 值 |
| --- | --- | --- |
| `KV_NAMESPACE_ID` | 文本 | 第 1 步复制的 Namespace ID |
| `D1_DATABASE_ID` | 文本 | 第 2 步复制的 Database ID |
| `ADMIN_TOKEN` | **Secret** | 自己生成的长随机串 |

`ADMIN_TOKEN` 务必选择 **Secret** 类型，不要用明文变量，否则会出现在构建日志里。

生成随机串可用任意方式，例如 `openssl rand -hex 32`，或自己敲一串 40 位以上的混合字符。

### 6. 部署与首次验证

点击 **Deploy**，等待构建完成（首次约 2–4 分钟）。完成后访问分配的 `*.workers.dev` 地址：

1. 用 `ADMIN_TOKEN` 登录后台；
2. 添加至少一个 Provider（建议先用带免费额度的，如 Google AI Studio、Groq、OpenRouter）；
3. 在模型列表里勾选要对外开放的模型；
4. 生成一把客户端访问密钥（形如 `llmf_...`），保存好，后台不会再次完整展示。

### 7. 验证接口

```bash
curl https://<你的域名>/v1/chat/completions \
  -H "Authorization: Bearer llmf_你的客户端密钥" \
  -H "Content-Type: application/json" \
  -d '{"model":"Groq/openai/gpt-oss-20b","messages":[{"role":"user","content":"你好"}]}'
```

模型名格式为 `供应商名称/上游模型ID`，可在后台 `/v1/models` 或模型列表中查到。

---

## 方式二：本地命令行部署

需要 Node.js 20+ 与 Cloudflare 账号。

### 1. 克隆并安装

```bash
git clone https://github.com/qgqcsw/LLMflare.git
cd LLMflare
npm install
npx wrangler login
```

### 2. 创建资源并记录 ID

```bash
npx wrangler kv namespace create CONFIG
npx wrangler d1 create llmflare-usage
```

命令会输出各自的 ID，复制备用。

### 3. 生成配置文件（不提交到仓库）

```bash
cp wrangler.example.jsonc wrangler.jsonc
```

Windows PowerShell 用：

```powershell
Copy-Item wrangler.example.jsonc wrangler.jsonc
```

编辑 `wrangler.jsonc`，把两处占位符替换为第 2 步得到的 ID：

```jsonc
{
  "kv_namespaces": [
    { "binding": "CONFIG", "id": "替换为你的 KV Namespace ID" }
  ],
  "d1_databases": [
    {
      "binding": "USAGE",
      "database_name": "llmflare-usage",
      "database_id": "替换为你的 D1 Database ID",
      "migrations_dir": "./migrations"
    }
  ]
}
```

> `wrangler.jsonc` 已在 `.gitignore` 中，不要提交。这样本地与上游代码依然保持完全一致。

### 4. 迁移数据库、设置密钥、部署

```bash
npx wrangler d1 migrations apply USAGE --remote
npx wrangler secret put ADMIN_TOKEN
npm run deploy
```

`npm run deploy` 会先同步 Workers AI 模型列表再部署。

### 5. 可选：绑定自有域名

控制台 → 你的 Worker → **Settings → Domains & Routes** → **Add → Custom domain**。

---

## 日常维护

### 同步上游更新

上游 `Jiao77/LLMflare` 有新提交时，你的 fork 不会自动更新。在 GitHub 仓库首页点击分支栏右侧的 **Sync fork → Update branch** 即可。本仓库仅新增了 `DEPLOY.md` 这一个文件，与上游任何文件都不重叠，因此几乎不会出现合并冲突。

使用方式一部署时，同步产生的推送会自动触发 Cloudflare 重新构建部署，无需其他操作。

### 数据库迁移

仓库新增迁移文件后需要重新执行一次：

```bash
npx wrangler d1 migrations apply USAGE --remote
```

方式一部署时，这一步已包含在部署命令里，每次构建会自动执行。

### 修改配置

后台配置（Provider、Key、模型白名单、客户端密钥）保存在 KV 中，修改代码或重新部署**不会**清空这些数据。但如果删除了 KV namespace，配置会全部丢失，需要重新添加。

---

## 安全与限制

- **Provider Key 建议只用免费额度或独立低额度 Key**。网关会代你向上游发起请求，Key 泄露意味着额度被消耗。切勿放入绑定高额余额或无限付款方式的主 Key。
- **限制后台访问**：公开部署时建议给 `/admin/*` 路径配置 Cloudflare Access 或 WAF 规则，并定期轮换 `ADMIN_TOKEN`。
- **客户端密钥可细分权限**：后台支持为每把客户端密钥单独设置 RPM、24 小时请求数、本月 Token 上限与单次最大输出 Token。
- **免费额度仍受上游限制**：网关不会绕过上游的每分钟请求数、并发与每日额度限制。
- **Token 统计为估算值的情形**：上游未返回 `usage` 时，部分流式请求的 Token 是旁路估算的，适合观察趋势，不宜作为账单依据。
- **免费版 Workers 配额**：每日请求数与 CPU 时间有上限。个人使用足够，若供多人长期使用建议升级付费版。
- **隐私**：Prompt 与模型回复不写入数据库，用量表只记录模型、Token、耗时、状态码等元数据。

---

## 常见问题

**构建时报错找不到 `wrangler.jsonc`**
说明构建命令没有执行或执行失败。检查构建命令是否完整复制，以及 `KV_NAMESPACE_ID`、`D1_DATABASE_ID` 两个变量是否已在构建环境中配置。

**部署成功但后台统计报错**
数据库迁移未执行。重新运行 `npx wrangler d1 migrations apply USAGE --remote`，或检查部署命令里是否包含这一步。

**后台无法登录**
`ADMIN_TOKEN` 未设置或与输入值不一致。可在控制台 Variables and Secrets 中重新设置后重新部署。

**访问接口返回 404**
确认请求路径带 `/v1` 前缀，且 `Authorization` 头为 `Bearer llmf_...` 形式。自定义域名场景下路径规则相同。

**同步上游后构建失败**
先看构建日志。若失败由上游代码改动引起，可暂时回退到上一个可用的提交，或等待上游修复。
