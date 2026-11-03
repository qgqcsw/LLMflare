# 手动部署说明（LLMflare on Cloudflare Workers）

本文件只描述部署步骤。仓库原有代码保持与上游完全一致，未做任何修改。

> 本仓库从 [Jiao77/LLMflare](https://github.com/Jiao77/LLMflare) fork 而来。上游仓库中只有 `wrangler.example.jsonc`（内含占位符 ID），**没有**可直接部署的 `wrangler.jsonc`。因此无论采用哪种方案，都需要在部署前把真实资源 ID 提供给 Wrangler。

---

## 目录

- [开始之前：先理解 4 个概念](#开始之前先理解-4-个概念)
- [必须遵守的顺序（附原因）](#必须遵守的顺序附原因)
- [第 1 步：创建 KV 命名空间](#第-1-步创建-kv-命名空间)
- [第 2 步：创建 D1 数据库](#第-2-步创建-d1-数据库)
- [第 3 步：连接仓库并导入项目](#第-3-步连接仓库并导入项目)
- [第 4 步：选择一种配置方案](#第-4-步选择一种配置方案)
  - [方案 A：构建命令内联 ID（推荐）](#方案-a构建命令内联-id推荐)
  - [方案 B：提交 wrangler.jsonc 到仓库（最稳）](#方案-b提交-wranglerjsonc-到仓库最稳)
- [第 5 步：首次部署](#第-5-步首次部署)
- [第 6 步：添加 ADMIN_TOKEN（运行时密钥）](#第-6-步添加-admin_token运行时密钥)
- [第 7 步：验收](#第-7-步验收)
- [坑位清单（按易踩程度排序）](#坑位清单按易踩程度排序)
- [附录 A：为什么不推荐用「构建变量」](#附录-a为什么不推荐用构建变量)
- [附录 B：本地命令行部署](#附录-b本地命令行部署)
- [附录 C：日常维护](#附录-c日常维护)
- [附录 D：安全与限制](#附录-d安全与限制)
- [附录 E：常见报错对照表](#附录-e常见报错对照表)

---

## 开始之前：先理解 4 个概念

这四个东西很容易混，混了就会出现「填了值却不生效」「保存了却丢失」这类现象。

**1. 资源名 vs 绑定名 —— 两者无关**

云端的资源叫什么是你的自由（KV 可以叫 `llmflare-config`、`my-kv`、`abc` 都行），但代码里读取时用的是**绑定名**。本项目的两个绑定名固定为：

| 绑定名 | 类型 | 作用 | 能否改名 |
| --- | --- | --- | --- |
| `CONFIG` | KV Namespace | 保存 Provider 配置、上游 API Key、客户端密钥 | **不能改**，代码里写死 |
| `USAGE` | D1 Database | 用量统计 | **不能改**，代码与迁移命令都用它 |

所以「数据库叫 `llmflare-usage`」只是方便你识别，真正参与绑定的是 `database_id` 这个 UUID。

**2. KV ID 与 D1 ID 格式不同，可用来互相校验**

- KV Namespace ID：**32 位十六进制**，如 `0f2c8a1d9b7e4c3f8a5d6e7b1c2d3e4f`
- D1 Database ID：**36 位 UUID**，带四段连字符，如 `8f3b2c1a-4d5e-4f6a-9b8c-1d2e3f4a5b6c`

复制完对一眼长度，能挡掉一半的填错。

**3. 构建变量 ≠ 运行时变量**

Cloudflare 有**两套完全独立**的变量体系，位置不同、互不共享：

| | 位置 | 生效阶段 | 用途 |
| --- | --- | --- | --- |
| 构建变量 | Settings → **Build** → Build variables and secrets | 仅在构建容器内 | 给 sed / 构建脚本用 |
| 运行时变量 | Settings → **Variables & Secrets** | Worker 运行时 | 给代码 `env.XXX` 用 |

官方原文：Build variables "will not be accessible at runtime"。把 `ADMIN_TOKEN` 加进构建区，代码读不到；把 `KV_NAMESPACE_ID` 加进运行时区，构建读不到（会展开成空串）。

**4. 为什么不能用「自动预置资源」省掉填 ID**

Wrangler 4.45+ 支持省略 `id` 让它在部署时自动创建资源。但本项目的部署命令是：

```bash
npm run sync:cf-models && npx wrangler d1 migrations apply USAGE --remote && npx wrangler deploy
```

`d1 migrations apply` 排在 `wrangler deploy` **之前**。自动预置发生在 deploy 那一刻，所以跑迁移时数据库还不存在，会直接失败。**结论：KV 和 D1 必须提前手动创建。**

---

## 必须遵守的顺序（附原因）

```
① 建 KV（拿 ID）
       ↓
② 建 D1（拿 ID）        ← 必须在部署前，因为迁移命令先于 deploy 执行
       ↓
③ 连接仓库、填项目名与生产分支
       ↓
④ 配置构建（方案 A 或 B）
       ↓
⑤ 首次部署（这一步才真正创建 Worker）
       ↓
⑥ 加 ADMIN_TOKEN 运行时 Secret → 重新部署使其生效
       ↓
⑦ 验收（页面 / 登录 / 配置持久化）
```

**顺序错了会怎样：**

- 先配构建、后建资源 → 构建报 `"id": ""` 或 `Invalid namespace ID`，白等一次构建
- 先部署、后建 D1 → 迁移报找不到数据库，或部署成功但统计功能报错
- `ADMIN_TOKEN` 在首次部署前就加 → 部分界面版本下 Worker 脚本尚不存在，密钥添加报错；稳妥做法是部署后立刻加、再部署一次

---

## 第 1 步：创建 KV 命名空间

进入 Cloudflare Dashboard，左侧导航找 **Storage & Databases**（旧版界面路径是 **Workers & Pages → KV**）。

点 **KV** 标签页 → **Create a namespace** → 名称填 `llmflare-config`（可自定义，不影响绑定）→ 创建。

创建完成后取 ID：

- 列表页若有 **ID** 列，直接复制；
- 或点名称进入详情页，复制 **Namespace ID**。

⚠️ **不要从浏览器地址栏复制任何片段**。URL 里的标识与绑定用的 Namespace ID 不是同一个东西，这是最高频的抄错来源。

✅ 记下：**32 位十六进制**，粘贴到临时文本里备用。

---

## 第 2 步：创建 D1 数据库

同一区域点 **D1 SQL Database** 标签页 → **Create database** → 名称填 `llmflare-usage` → 创建。

进入数据库详情页，在 **Overview** 或 **Settings** 中找到 **Database ID**（36 位 UUID），复制。

⚠️ 该页面上同时存在 **Database name**、**Database ID**、**binding**、以及某些版本会露出内部 `uuid` 字段。**只取 Database ID** 那一个。

✅ 记下：**36 位带连字符的 UUID**，备用。

---

## 第 3 步：连接仓库并导入项目

**Workers & Pages** → **Create** → **Import a repository**（或 Workers 页签下的 **Connect to Git**）→ 授权 GitHub → 选择你 fork 的仓库 → 分支选 `main`。

填项目名称，例如 `llmflare`。这个名称会成为 Worker 名与默认域名前缀（`llmflare.<你的子域>.workers.dev`），一旦部署后改名需要重新分配域名，所以此刻就想好。

**Root directory 留空**（仓库根目录就是项目根目录）。

此时先不要点部署，直接进入下一步配置构建。

---

## 第 4 步：选择一种配置方案

两种方案都能成功，差异在于「ID 放在哪」：

| | 方案 A 内联 ID | 方案 B 提交 wrangler.jsonc |
| --- | --- | --- |
| 改动位置 | Dashboard 构建命令 | 仓库新增 1 个文件 |
| 仓库改动 | 无 | 新增 `wrangler.jsonc` |
| 依赖环境变量 | 不依赖 | 不依赖 |
| 排错难度 | 中（命令里能看到） | 低（配置文件直读） |
| 上手成本 | 一次性粘贴 | 需能编辑仓库文件 |
| 上游同步冲突 | 无 | 无（新增文件，路径不同） |
| **推荐场景** | 不想动仓库 | 想一次配好、长期稳定 |

两者都推荐；如果你仍受 GitHub 网页编辑器打不开的困扰，直接用方案 A。

---

### 方案 A：构建命令内联 ID（推荐）

在 **Settings → Build** 页面：

**Build command** 填入下面整条（把两个示例 ID 换成第 1、2 步拿到的真实值）：

```bash
sed -e 's/REPLACE_WITH_KV_NAMESPACE_ID/0f2c8a1d9b7e4c3f8a5d6e7b1c2d3e4f/' -e 's/REPLACE_WITH_D1_DATABASE_ID/8f3b2c1a-4d5e-4f6a-9b8c-1d2e3f4a5b6c/' wrangler.example.jsonc > wrangler.jsonc && echo "--- generated ---" && grep -A2 'kv_namespaces\|d1_databases' wrangler.jsonc && if grep -qE '"id"[[:space:]]*:[[:space:]]*""|"database_id"[[:space:]]*:[[:space:]]*""' wrangler.jsonc; then echo "ERROR: 生成的配置里存在空 id"; exit 1; fi
```

**Deploy command** 填：

```bash
npm run sync:cf-models && npx wrangler d1 migrations apply USAGE --remote && npx wrangler deploy
```

这条命令做了 5 件事，逐段解释：

1. `sed -e 's/.../.../'` —— 用真实 ID 替换示例文件中的两处占位符，生成 `wrangler.jsonc`。用**单引号**包裹是刻意的：单引号内 bash 不做变量展开，从而彻底消除「变量没注入 → 展开成空串」这类故障。
2. `&& echo "--- generated ---"` —— 在日志里打一条分隔线，方便定位。
3. `&& grep -A2 'kv_namespaces\|d1_databases'` —— 把生成结果的相关片段打进日志，出问题时可肉眼核对 ID 是否完整。
4. `&& if grep -qE '"id"...: ""' ...; then ... exit 1; fi` —— **空值自检**。注意这里匹配的是 `"id": ""` 这种**空字符串**形态，而不是匹配 `REPLACE_WITH`。原因见[坑位清单第 2 条](#坑位清单按易踩程度排序)。
5. 整个链条用 `&&` 串联，任一环节失败即中止，不会带着坏配置继续部署。

填完点页面底部 **Save**。

---

### 方案 B：提交 wrangler.jsonc 到仓库（最稳）

这个方案让构建阶段不再需要生成配置，Wrangler 直接读仓库里的真实文件。

**B-1 拿到模板内容**

打开仓库里的 `wrangler.example.jsonc`（在 GitHub 网页上点开即可），点击右上角 **Raw** 或 **Copy raw file** 复制全文。

**B-2 替换两处占位符**

把 `REPLACE_WITH_KV_NAMESPACE_ID` 换成你的 32 位 KV ID，把 `REPLACE_WITH_D1_DATABASE_ID` 换成你的 36 位 UUID。其余字段（`main`、`assets`、`compatibility_flags`、`observability` 等）**一律不要动**。

⚠️ 不要凭记忆手打整个配置。以仓库里的 `wrangler.example.jsonc` 为蓝本复制修改，是唯一能保证 `main`、`assets.directory`、`run_worker_first` 等字段与代码结构匹配的做法。

**B-3 在仓库根目录新增 wrangler.jsonc**

GitHub 仓库页 → **Add file → Create new file** → 文件名填 `wrangler.jsonc` → 粘贴 B-2 的内容 → 底部 **Commit changes**。

**B-4 调整构建配置**

回到 **Settings → Build**：

- **Build command**：留空，或填 `echo skip`
- **Deploy command** 保持不变：

```bash
npm run sync:cf-models && npx wrangler d1 migrations apply USAGE --remote && npx wrangler deploy
```

点 **Save**。

**B-5 验证仓库里确实有这个文件**

回到仓库首页，确认根目录出现 `wrangler.jsonc`，且内容里两个 ID 都非空。这一步别跳过——如果文件建在了子目录或者名字拼错（如 `wrangler.json`），构建会当成「没有配置文件」而触发自动配置流程，症状和没配一样。

---

## 第 5 步：首次部署

两种触发方式，效果相同：

- 在导入页面直接点 **Deploy**；
- 或去 **Deployments** 页面点 **Retry deployment**（适用于之前失败过的项目）。

官方说明：重试会使用「重试时刻已保存的构建配置」，所以改完配置点 Retry 即可，**不需要推新提交**。

首次构建约 2–4 分钟。构建日志里按顺序核对：

1. `Executing user build command:` 后是你填的命令（方案 B 是空/skip，跳过此项）
2. `--- generated ---` 与几行 JSON 片段（方案 A），检查 `id` 是否 32 位、`database_id` 是否 UUID
3. `Success: Build command completed`
4. `npx wrangler deploy` 输出 `Uploaded` 与 `Deployed`，并给出 `*.workers.dev` 地址

看到 `Deployed` 才算成功。构建成功 ≠ 部署成功，两者是分开的阶段。

---

## 第 6 步：添加 ADMIN_TOKEN（运行时密钥）

部署成功后，进入 **Settings → Variables & Secrets**（注意：是顶层这个分区，**不是** Build 分区）。

点 **Add** → 类型选 **Secret** → 名称 `ADMIN_TOKEN` → 值填 32 位以上随机串。可用任意方式生成，例如 `openssl rand -hex 32`，或自己敲一串 40 位以上混合字符。

保存后，Cloudflare 通常会提示需要重新部署一次才能让密钥生效——按提示操作，或再点一次 **Retry**。

⚠️ **必须是 Secret 类型，不能是明文变量。** 明文运行时变量会在下一次 `wrangler deploy` 时被自动删除，除非在配置文件里设 `"keep_vars": true`。Secret 不受此影响。

如果此时后台已经能打开但登录失败，先检查这一步是否漏做或值是否与输入不一致。

---

## 第 7 步：验收

按这个顺序做四项检查，每项验证的是不同链路：

**① 页面能否加载**

打开 `https://<项目名>.<子域>.workers.dev`。能看到登录页，说明 `assets` 静态资源绑定与 `run_worker_first` 路由规则都正常。

**② 能否登录**

用 `ADMIN_TOKEN` 登录后台，成功即验证 Secret 链路。失败则回第 6 步。

**③ 配置是否持久化**

添加一个 Provider（建议先用带免费额度的，如 Google AI Studio、Groq、OpenRouter），保存后**强制刷新页面**，看配置是否还在。

⚠️ 这一步专门验证 KV 绑定。如果保存提示成功但刷新后丢失，几乎一定是 `KV_NAMESPACE_ID` 填成了别的东西的 ID，而不是网关逻辑问题。

**④ 接口是否可用**

后台生成一把客户端密钥（形如 `llmf_...`，只展示一次，务必保存），然后：

```bash
curl https://<你的域名>/v1/chat/completions \
  -H "Authorization: Bearer llmf_你的客户端密钥" \
  -H "Content-Type: application/json" \
  -d '{"model":"Groq/openai/gpt-oss-20b","messages":[{"role":"user","content":"你好"}]}'
```

模型名格式为 `供应商名称/上游模型ID`。返回正常即全链路打通。

---

## 坑位清单（按易踩程度排序）

以下每一条都对应真实踩过的故障，按踩中概率从高到低排列。

### 坑 1：「构建变量」加在了运行时变量区

**症状**：构建日志出现 `KV_ID length=0 D1_ID length=0`，随后报 `"kv_namespaces[0]" bindings should have a string "id" field but got {"binding":"CONFIG","id":""}`。

**原因**：`Settings → Variables & Secrets` 是运行时变量，构建容器读不到。构建变量必须加在 `Settings → Build → Build variables and secrets`。

**判断方法**：在构建命令最前面加一句 `echo "KV_ID length=${#KV_NAMESPACE_ID}"`。输出 `length=0` 就是没注入，和 sed 无关。

**处理**：直接改用方案 A（内联 ID）或方案 B（提交文件）。这两条路都不依赖构建变量，能一次性绕开该故障面。

### 坑 2：自检写成了 grep 占位符，抓不到空值

**症状**：明明 ID 是空的，自检却一路绿灯，构建「成功」跑完 sed，直到 wrangler 才报错。

**原因**：`grep -q REPLACE_WITH` 只能发现「占位符没被替换」。而变量为空时，占位符**已经被替换成了空字符串**，文件里一个 `REPLACE_WITH` 都不剩，grep 必然放过。

**处理**：自检要匹配空值形态：

```bash
grep -qE '"id"[[:space:]]*:[[:space:]]*""' wrangler.jsonc
```

**这是本项目最容易误判的一点**——失败点被推迟到 wrangler 的配置校验，报错信息指向「格式不对」而不是「值缺失」，会让人误以为是 JSON 语法问题。

### 坑 3：让 Wrangler 自动预置资源

**症状**：迁移阶段报数据库不存在，或部署报找不到 D1。

**原因**：自动预置（省略 `id` 字段）发生在 `wrangler deploy` 那一刻，而本项目部署命令里 `d1 migrations apply` 排在它前面。

**注意区分**：自动预置要求**彻底省略** `id` 字段，写成 `{"binding":"CONFIG"}`。写成 `"id": ""` 是**非法值**，Wrangler 会拒绝，两者不是一回事。

**处理**：老老实实先建 KV 和 D1，再填 ID。

### 坑 4：ID 复制时混入空格或换行

**症状**：`Invalid namespace ID`、`Invalid database ID`，或报格式校验失败。

**原因**：剪贴板首尾带了空格、换行，或从多行文本里连带粘了下一行。值里多一个空格，生成出来就是 `"id": " 0f2c..."`。

**处理**：粘贴后把光标移到值的开头和结尾各按一次 Delete/Backspace。用长度校验也能挡住：KV 必须 32 位、D1 必须 36 位。

### 坑 5：抄错 ID 来源

- KV：从 URL 或页面内部标识复制 → 不是绑定用的 Namespace ID
- D1：复制了 **Database name**（`llmflare-usage`）而不是 **Database ID**（UUID）
- D1：复制了内部 `uuid` 字段而非 Database ID

**处理**：只用「详情页里明确标着 Database ID / Namespace ID 且旁边有复制按钮」的那个值。格式自检：KV 32 位十六进制、D1 36 位带连字符。

### 坑 6：改了绑定名

**症状**：部署成功，但后台保存配置后刷新丢失，或统计页报错。

**原因**：把 `CONFIG` 或 `USAGE` 改成了自己的命名（如 `KV`、`DB`）。代码里是按原绑定名读取的。

**处理**：绑定名一个字都不要改。资源名可以随便叫，绑定名固定。

### 坑 7：改完配置没 Save 就重试

**症状**：重试后日志里 `Executing user build command:` 还是旧命令。

**原因**：Workers Builds 只在 Save 后把新配置写入；Retry 使用「重试时刻已保存」的配置。

**处理**：先 Save，再 Retry。如果日志里命令没变，说明保存没生效，或改的是另一个项目（检查 Settings 页顶部的项目名与失败日志是否一致）。

### 坑 8：方案 B 的文件位置或文件名错

**症状**：构建时触发「自动项目配置」并生成一个 PR，或报找不到配置。

**原因**：文件名写成了 `wrangler.json`（少了 c）、或建在了子目录、或 `Root directory` 被填了其他值。

**处理**：文件名必须是 `wrangler.jsonc`，位置在仓库根目录，`Root directory` 留空。

### 坑 9：把 Workers AI 目录警告当成失败原因

**症状**：日志出现 `[WARN] Workers AI model catalog sync failed; using cached catalog (38 models, generated at 2026-10-19...)`。

**原因**：`npm run sync:cf-models` 内部会调 `wrangler ai models list`，这一步同样要解析 `wrangler.jsonc`。配置非法时它一起挂掉，脚本于是降级使用仓库里缓存的 38 个模型目录。

**处理**：不用管。配置修好后通常自动恢复；即便仍出现，也只是模型清单停留在旧版本，不影响 Workers AI 调用。真正让构建中止的是后面的 `Failed: error occurred while running deploy command`。

### 坑 10：构建成功就当部署成功

**症状**：Build 阶段全绿，访问域名却 404 或 500。

**原因**：`Build command` 只负责生成配置/编译；真正上传是 `Deploy command`。两者任一失败都会中断。

**处理**：日志必须出现 `Deployed` 字样。若只有 `Success: Build command completed` 而没有 `Deployed`，继续往下翻日志找 deploy 阶段的错误。

### 坑 11：ADMIN_TOKEN 用明文变量

**症状**：首次登录正常，重新部署一次后登录失败。

**原因**：明文运行时变量会在下一次 `wrangler deploy` 时被清除（除非 `"keep_vars": true`）。

**处理**：一律用 **Secret** 类型。

### 坑 12：在本地用双引号写 sed，又被 shell 抢先展开

**症状**：本地命令行执行时生成的配置里 `id` 为空。

**原因**：`sed -e "s/A/${VAR}/"` 中如果 `VAR` 未定义，双引号内会先展开成空串；而单引号不会。

**处理**：内联 ID 时用单引号；需要引用变量时，先 `echo` 确认变量非空。

---

## 附录 A：为什么不推荐用「构建变量」

这条路线在理论上完全可行（把 `KV_NAMESPACE_ID` / `D1_DATABASE_ID` 加进 `Settings → Build → Build variables and secrets`，构建命令继续用 `${KV_NAMESPACE_ID}` 引用），但在本项目的实际排障中失败了两次，原因有三：

1. **位置极易混淆**。构建变量与运行时变量分属不同分区，且菜单在不同 Dashboard 版本中位置会变，很容易加到运行时那一侧，症状就是 `length=0`。
2. **失败信息误导性强**。变量缺失 → 展开成空串 → Wrangler 报的是 binding 字段格式错误，而不是「变量未定义」，会把人引向 JSON 语法排查。
3. **收益仅是可读性**。构建命令里用变量引用唯一的好处是「看起来干净」，但它带来 3 个新故障面（位置、拼写、作用域），而内联 ID 只有 0 个。

如果你仍想用，唯一可靠的验证方式是在构建命令最前面打印长度：

```bash
echo "KV_ID length=${#KV_NAMESPACE_ID} D1_ID length=${#D1_DATABASE_ID}"
```

输出必须是 `32` 和 `36`。是 `0` 就立刻停止在变量上折腾，改用方案 A 或 B。

---

## 附录 B：本地命令行部署

需要 Node.js 20+ 与 Cloudflare 账号。

```bash
git clone https://github.com/qgqcsw/LLMflare.git
cd LLMflare
npm install
npx wrangler login
```

创建资源并取 ID（若已在控制台创建过，跳过创建，用 `npx wrangler kv namespace list` / `npx wrangler d1 list` 查看已有 ID）：

```bash
npx wrangler kv namespace create CONFIG
npx wrangler d1 create llmflare-usage
```

从示例生成配置（`wrangler.jsonc` 已在 `.gitignore` 中，不要提交）：

```bash
cp wrangler.example.jsonc wrangler.jsonc
```

Windows PowerShell：

```powershell
Copy-Item wrangler.example.jsonc wrangler.jsonc
```

编辑 `wrangler.jsonc`，替换两处占位符为真实 ID，然后：

```bash
npx wrangler d1 migrations apply USAGE --remote
npx wrangler secret put ADMIN_TOKEN
npm run deploy
```

顺序同样关键：**迁移 → 密钥 → 部署**。`npm run deploy` 会先同步 Workers AI 模型列表再部署。

---

## 附录 C：日常维护

### 同步上游更新

上游 `Jiao77/LLMflare` 有新提交时，你的 fork 不会自动更新。在 GitHub 仓库首页点分支栏右侧的 **Sync fork → Update branch**。

本仓库与上游的差异只有一个新增的 `DEPLOY.md`（方案 B 还会多一个 `wrangler.jsonc`），两者都是新增文件，与上游任何文件不重叠，因此不会产生合并冲突。这也是当初「不改原有代码」这个决定带来的直接好处。

同步产生的推送会自动触发 Cloudflare 重新构建部署，无需其他操作。

⚠️ 但要注意：如果你在仓库里**修改**过 `README.md` 或其他上游文件，冲突风险立刻上升，因为作者会持续更新这些文件。保持「只新增、不修改」是零摩擦同步的前提。

### 数据库迁移

仓库新增迁移文件后需要执行一次：

```bash
npx wrangler d1 migrations apply USAGE --remote
```

方案 A/B 的部署命令已包含这一步，每次构建自动执行，无需手动干预。

### 修改后台配置

后台配置（Provider、Key、模型白名单、客户端密钥）保存在 KV 中，修改代码或重新部署**不会**清空这些数据。

但如果删除了 KV 命名空间，配置会全部丢失，需要重新添加。所以不要为了「重来一次」去删 KV。

---

## 附录 D：安全与限制

- **Provider Key 只用免费额度或独立低额度 Key**。网关会代你向上游发起请求，Key 泄露即意味着额度被消耗。切勿放入绑定高额余额或无限付款方式的主 Key。
- **限制后台访问**：公开部署时建议给 `/admin/*` 配置 Cloudflare Access 或 WAF 规则，并定期轮换 `ADMIN_TOKEN`。
- **客户端密钥可细分权限**：后台支持为每把客户端密钥单独设置 RPM、24 小时请求数、本月 Token 上限、单次最大输出 Token。
- **免费额度仍受上游限制**：网关不会绕过上游的每分钟请求数、并发与每日额度。
- **Token 统计可能是估算值**：上游未返回 `usage` 时，部分流式请求的 Token 由旁路估算，适合观察趋势，不宜作为账单依据。
- **免费版 Workers 配额**：每日请求数与 CPU 时间有上限。个人使用足够，供多人长期使用建议升级付费版。
- **隐私**：Prompt 与模型回复不写入数据库，用量表只记录模型、Token、耗时、状态码等元数据。
- **两个资源 ID 不是凭据**：被写进构建命令或提交到仓库都不构成安全风险，真正能操作你账号的是 API Token。

---

## 附录 E：常见报错对照表

| 报错 / 现象 | 根因 | 处理 |
| --- | --- | --- |
| `bindings should have a string "id" field but got {"binding":"CONFIG","id":""}` | ID 被替换成了空串（变量未注入），或配置里本来就是空值 | 改用方案 A 内联 ID 或方案 B 提交文件；自检改为匹配空值 |
| `Invalid namespace ID` / `Invalid database ID` | 值带空格、格式错、抄错了来源 | 校验长度：KV 32 位、D1 36 位 UUID；清理首尾空白 |
| 占位符仍在配置中 | sed 目标字符串与文件里的实际占位符不一致 | 打开 `wrangler.example.jsonc` 逐字符比对 |
| `Failed: error occurred while running deploy command` | deploy 阶段失败，通常是配置非法或迁移失败 | 往上翻日志找第一条 `✘ [ERROR]` |
| `[WARN] Workers AI model catalog sync failed; using cached catalog` | `wrangler ai models list` 因配置非法而失败，脚本降级用缓存 | 可忽略；配置修好后通常自动恢复 |
| 部署成功但后台统计报错 | D1 迁移未执行 | 确认部署命令包含 `d1 migrations apply USAGE --remote` |
| 后台无法登录 | `ADMIN_TOKEN` 未设置、类型错（明文）或值不一致 | 用 Secret 类型重新设置并重新部署 |
| 保存 Provider 后刷新丢失 | KV 绑定指向了错误的命名空间 | 核对 `KV_NAMESPACE_ID` |
| 接口返回 404 | 路径缺 `/v1` 前缀，或鉴权头格式不对 | 用 `Bearer llmf_...`，路径带 `/v1` |
| 构建触发「自动项目配置」并生成 PR | 仓库根目录没有可识别的 `wrangler.jsonc` | 确认文件名与位置；方案 B 检查是否拼成 `wrangler.json` |
| 重试后日志里命令没变 | 改了配置但没 Save，或改的是另一个项目 | 先 Save 再 Retry；核对项目名 |
