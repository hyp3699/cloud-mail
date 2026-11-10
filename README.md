## 安装教程

> 本节内容来自官方部署文档 [快速开始](https://doc.skymail.ink/guide/quick-start.html)，按本仓库当前代码（`wrangler.toml`、`wrangler-action.toml`、`.github/workflows/deploy-cloudflare.yml`）核对过，个别与文档不一致的地方已在文中注明。
>
> 三种部署方式任选其一，效果相同，区别只在操作位置和后续更新方式：
>
> | 方式 | 适合人群 | 后续更新 |
> | --- | --- | --- |
> | [方法一：界面部署](#方法一界面部署cloudflare-dashboard) | 不想装 Node / 不想用命令行 | 同步代码后 Worker 自动更新，需重新绑定 D1、KV |
> | [方法二：GitHub Action 部署](#方法二github-action-部署) | 想一键部署、以后自动更新 | 同步仓库代码到最新，Action 自动重新部署 |
> | [方法三：命令部署](#方法三命令部署本地命令行) | 习惯本地开发、想自己改代码 | 本地改完再跑一次 `pnpm run deploy` |

### 部署前准备（三种方式都要）

1. **注册 Cloudflare 账号，并把域名接入 Cloudflare**
   - 注册地址：<https://dash.cloudflare.com>
   - 域名必须已托管在 Cloudflare（DNS 由 Cloudflare 解析），否则后面收不到邮件、也绑不了自定义域。不会接入可参考腾讯云教程：<https://cloud.tencent.cn/developer/news/1457060>
2. **准备好三个值，后面每一步都要用**
   - **邮箱域名**：你用来收发邮件的域名，例如 `example.com`，可配多个
   - **管理员邮箱**：例如 `admin@example.com`，必须是上面邮箱域名下的地址
   - **JWT 密钥（`jwt_secret`）**：自己随便输入一串字符串，**不要含特殊字符 `? % # / \`**
3. **（可选）先想好发件方式**：只收邮件可以先不配；要发邮件，见 [邮件发送](#邮件发送) 一节，二选一：Resend，或 Cloudflare Email（需要 Workers Paid 套餐）。

### 方法一：界面部署（Cloudflare Dashboard）

1. **Fork / 克隆仓库到自己的 GitHub 账号**
   - 本仓库：<https://github.com/hyp3699/cloud-mail>
   - 上游：<https://github.com/maillab/cloud-mail>
2. **在 Cloudflare Dashboard 创建 Worker 项目**
   - Workers & Pages → Create → 选择 **Import a repository / 从 GitHub 导入**。
3. **设置项目目录并部署**
   - 导入时把目录设为 `mail-worker`，然后点 Deploy 部署。
   - 注意：`mail-worker/wrangler.toml` 里的 `[build]` 会自动先构建前端：`pnpm --prefix ../mail-vue install && pnpm --prefix ../mail-vue run build`，打包产物放在 `mail-worker/dist`。
4. **设置环境变量**：Worker → Settings → Variables，添加下面三项（缺一不可）

   | 变量名 | 必需 | 用途 / 格式 |
   | --- | --- | --- |
   | `domain` | ✅ | 邮箱域名，多域名用 JSON 数组，例如 `["example.com","example2.com"]` |
   | `admin` | ✅ | 管理员邮箱地址，例如 `admin@example.com` |
   | `jwt_secret` | ✅ | JWT 密钥，随便输入一串字符串，不要输入特殊字符 `? % # / \` |

5. **设置自定义域**：Worker → Settings → Domains，添加你要访问网站的域名（例如 `mail.example.com` 或 `skymail.ink`）。
6. **绑定数据库**：先在 Cloudflare 控制台创建 KV 和 D1 数据库，再到 Worker → Settings → Bindings 添加绑定，**变量名必须 exactly 是 `kv` 和 `db`**，不能改名。
   - KV：创建一个 KV Namespace，绑定变量名 `kv`
   - D1：创建一个 D1 Database，绑定变量名 `db`
   - （可选）R2：见 [对象存储](#对象存储r2)，绑定变量名 `r2`
7. **设置邮件转发（收件）**：Cloudflare → 账户主页 → 你的域名 → 电子邮件 → 电子邮件路由 → 路由规则 → Catch-all 地址，编辑为“发送到 Worker”，选择刚部署的 Worker。**不做这一步收不到邮件。**
8. **初始化数据库**：浏览器打开
   ```
   https://你的 Worker 自定义域/api/init/你的 jwt_secret
   ```
   看到成功提示即完成。只会创建 / 更新表结构，不会覆盖已有数据。
9. **登录网站**：浏览器打开自定义域名，注册管理员账号（用上面 `admin` 那个邮箱），登录即可使用。

### 方法二：GitHub Action 部署

1. **创建 Cloudflare API 令牌**
   - 打开 <https://dash.cloudflare.com/profile/api-tokens> → Create Token → 选择 **Workers 模板**（Edit Cloudflare Workers），创建后复制令牌。
   - 同时记下 **账户 ID（Account ID）**：在 Cloudflare Dashboard 右侧边栏可见。
2. **Fork 本仓库到自己的 GitHub 账号**：<https://github.com/hyp3699/cloud-mail>
3. **配置 Action Secrets / Variables**：仓库 → Settings → Secrets and variables → Actions，按下表添加。`NAME` 以外均为部署必需；`D1_DATABASE_ID` / `KV_NAMESPACE_ID` 不填时，工作流会自动创建同名数据库并取其 ID。

   | 变量名 | 必需 | 用途 |
   | --- | --- | --- |
   | `CLOUDFLARE_API_TOKEN` | ✅ | 上一步创建的 Cloudflare API 令牌 |
   | `CLOUDFLARE_ACCOUNT_ID` | ✅ | Cloudflare 账户 ID |
   | `CUSTOM_DOMAIN` | ✅ | 自定义域名，用来访问网站，例如 `skymail.ink` |
   | `DOMAIN` | ✅ | 邮箱域名，多域名用 JSON 数组，例如 `["example.com","example2.com"]` |
   | `ADMIN` | ✅ | 管理员邮箱地址，例如 `admin@example.com` |
   | `JWT_SECRET` | ✅ | JWT 密钥，随便输入一串字符串，不要输入特殊字符 `? % # / \` |
   | `D1_DATABASE_ID` | ✅* | D1 数据库 ID；不填则自动创建（参考界面部署的绑定数据库一节，先在控制台创建也可） |
   | `KV_NAMESPACE_ID` | ✅* | KV 命名空间 ID；不填则自动创建 |
   | `R2_BUCKET_NAME` | ❌ | R2 桶名；不填则不绑定 R2，附件默认走 KV 存储 |
   | `EMAIL` | ❌ | 用 Cloudflare Email 发件时填固定值 `email`，见 [Cloudflare 邮件发送](#cloudflare-邮件发送) |
   | `NAME` | ❌ | Worker 项目名，默认 `cloud-mail` |
   | `AI_MODEL` | ❌ | Workers AI 模型，默认 `@cf/meta/llama-3.1-8b-instruct-fast` |
   | `ANALYSIS_CACHE` | ❌ | 开启分析页数据缓存，`true` 开启 / `false` 关闭，默认 `false` |
   | `PROJECT_LINK` | ❌ | 登录界面是否显示 GitHub 图标，`true` 显示 / `false` 隐藏，默认 `true` |

   > 工作流实际读取的是 `mail-worker/wrangler-action.toml`：`CUSTOM_DOMAIN` 为空时不绑定自定义路由；`R2_BUCKET_NAME` 为空时删除 R2 绑定；`EMAIL` 非空时才追加 `[[send_email]] name = "email"`。
4. **开始部署**：仓库 → Actions → 选择 `Deploy cloud-mail to Cloudflare Workers` → Run workflow，等待运行完成（绿色 ✅ 即成功）。
5. **设置邮件转发**：同方法一第 7 步，域名 → 电子邮件路由 → Catch-all 地址 → 发送到 Worker。
6. **登录网站**：浏览器打开自定义域名，注册管理员账号，登录即可使用。
   - Action 部署已在部署时完成数据库初始化，一般不用再手动访问 `/api/init/...`；更新后如需更新表结构，再访问一次即可。

### 方法三：命令部署（本地命令行）

1. **环境准备**
   - Node.js **v20.19.6**（文档要求；本仓库 GitHub Action 用的是 Node 24 + pnpm 11，也能跑）
   - Cloudflare 账号（需要已绑定域名）
   - 安装 pnpm（若未安装）：`npm install -g pnpm`
2. **克隆项目到本地**
   ```bash
   git clone https://github.com/hyp3699/cloud-mail  # 拉取代码
   cd cloud-mail/mail-worker                          # 进入 worker 目录
   ```
3. **安装依赖**
   ```bash
   pnpm i
   ```
4. **项目配置**：编辑 `mail-worker/wrangler.toml`，取消注释并填好数据库绑定和环境变量（以下为本仓库当前文件的结构，文档里的旧示例定时任务是 `0 16 * * *`，当前仓库是每小时 `0 * * * *`）：

   ```toml
   [[d1_databases]]
   binding = "db"              # D1 数据库绑定名，默认不可修改
   database_name = ""          # D1 数据库名字
   database_id = ""            # D1 数据库 id

   [[kv_namespaces]]
   binding = "kv"              # KV 绑定名，默认不可修改
   id = ""                     # KV 数据库 id

   [[r2_buckets]]
   binding = "r2"              # R2 对象存储绑定名，默认不可修改
   bucket_name = ""            # R2 对象存储桶的名字

   [assets]
   binding = "assets"          # 静态资源绑定名，默认不可修改
   directory = "./dist"        # 前端 Vue 项目打包的静态资源存放位置，默认 dist

   [triggers]
   crons = ["0 * * * *"]       # 定时任务，每小时执行；resetDaySendCount 仅在 UTC 0 点生效

   [vars]
   orm_log = false
   domain = []                 # 邮件域名，可配置多个，示例：["example1.com","example2.com"]
   admin = ""                  # 管理员的邮箱，示例："admin@example.com"
   jwt_secret = "你的密钥"      # JWT 密钥，不要含特殊字符 ? % # / \
   ```

   > 本仓库 `wrangler.toml` 里 D1 / KV / R2 三段默认是注释掉的，部署前记得取消注释并填值；`[[send_email]] name = "forward"`（站内转发到已验证的其他邮箱）和 `[ai] binding = "ai"` 已默认开启。
   >
   > 也可以先在 Cloudflare 控制台创建好 KV、D1 数据库，再用 `wrangler` 命令行绑定，或直接在 Dashboard 上改绑定，效果一样。
5. **远程部署**
   ```bash
   pnpm run deploy
   ```
   该命令即 `wrangler deploy`，会先按 `[build]` 构建前端（`mail-vue`），再部署 Worker。
6. **设置邮件转发**：Cloudflare → 账户主页 → 你的域名 → 电子邮件 → 电子邮件路由 → 路由规则 → Catch-all 地址，编辑为“发送到 Worker”。
7. **初始化 / 更新数据库**：浏览器打开
   ```
   https://你的 Worker 自定义域/api/init/你的 jwt_secret
   ```
8. **登录网站**：浏览器打开自定义域名，注册管理员账号，登录即可使用。

### 环境配置：更多变量

除上面三项必填变量外，还可以在 Worker Variables / Action Secrets 里配这些可选变量：

| Worker 变量名 | Action Secret | 默认值 | 必需 | 用途 |
| --- | --- | --- | --- | --- |
| `ai_model` | `AI_MODEL` | `@cf/meta/llama-3.1-8b-instruct` | ❌ | Workers AI 模型（验证码识别用；Action 默认是 `@cf/meta/llama-3.1-8b-instruct-fast`） |
| `analysis_cache` | `ANALYSIS_CACHE` | `false` | ❌ | 开启分析页数据缓存（`true` 开启 / `false` 关闭） |
| `project_link` | `PROJECT_LINK` | `true` | ❌ | 隐藏登录界面 GitHub 图标（`true` 显示 / `false` 隐藏） |
| `orm_log` | — | `false` | ❌ | 是否打印 SQL 日志，仅本地 `wrangler.toml` / `wrangler-dev.toml` 里配 |

### 邮件发送

发件二选一，配好后在系统设置里开启对应方式。

#### Resend 邮件发送

1. 注册 [Resend](https://resend.com)，并添加域名，完成 DNS 验证。
2. 创建 API Key 并复制。
3. 在 Resend 控制台设置发送状态回调：`https://你的 Worker 自定义域/api/webhooks`。
4. 在 Resend 里选择对应选项（域名 / 回调）。
5. 到 Cloud Mail 的系统设置里填入 Resend API Key 并开启。

#### Cloudflare 邮件发送

> ⚠️ 该方式发送邮件需要 **Workers Paid**（付费）套餐。

1. 在 Cloudflare 里添加要发送邮件的域名。
2. 绑定电子邮件服务到 Worker 绑定或 Action Secret，变量值固定为 `email`：

   | Worker 绑定名称 | Action Secret | 必需 | 用途 |
   | --- | --- | --- | --- |
   | `email` | `EMAIL` | ✅ | 固定填 `email` |

   - 界面部署：在 Worker → Settings → Bindings 添加 Email Sending 绑定，名称 `email`。
   - Action 部署：`EMAIL` 填 `email`，工作流会自动追加 `[[send_email]] name = "email"`。
   - 命令部署：在 `wrangler.toml` 里加一段：
     ```toml
     [[send_email]]
     name = "email"
     ```
3. 到 Cloud Mail 的系统设置里开启 Cloudflare Email。

### 系统设置

以下都在部署完成、登录网站后，在站内「系统设置」里操作。

#### 对象存储（R2）

> 邮件附件默认使用 KV 存储，可以切换为 R2 或其他 S3 协议存储。

1. 在 Cloudflare 创建 R2 对象存储桶。
2. 添加到 Action Secret 或 Workers 绑定：

   | Worker 绑定名称 | Action Secret | 必需 | 用途 |
   | --- | --- | --- | --- |
   | `r2` | `R2_BUCKET_NAME` | ✅ | R2 桶名称 |

#### 人机验证（Turnstile）

1. 在 Cloudflare Dashboard 创建 Turnstile 组件。
2. 创建 API Key 并复制（Site Key + Secret Key）。
3. 到 Cloud Mail 的系统设置里填入并开启，用于登录 / 注册 / 添加邮箱时的人机验证。

#### 邮件转发

**转发到 Telegram：**

1. 在 TG 里找到 `@BotFather`，创建机器人，复制 Token。
2. 给机器人发一条消息。
3. 浏览器打开 `https://api.telegram.org/bot<你的机器人 Token>/getUpdates` 获取 `chat_id`；如果获取不到，就再给机器人发几条消息。
4. 到 Cloud Mail 的系统设置里填入 Bot Token 和 `chat_id` 并开启。

**转发到其他邮箱：**

1. 在 Cloudflare 验证目标邮箱（Email Routing → Destination addresses 里验证）。
2. 到 Cloud Mail 的系统设置里填入目标邮箱并开启。

#### OAuth2 登录（以 GitHub 为例）

1. 在 <https://github.com/settings/applications/new> 创建 OAuth App。
2. 回调地址（Authorization callback URL）填：
   ```
   https://你的 Worker 自定义域名/login
   ```
   > 文档原文写的是 `https//你的worker自定义域名/login`，少了一个冒号，以上面这行为准。
3. 创建密钥（Client ID / Client Secret）后，到 Cloud Mail 的系统设置里添加。

### 项目更新

- **GitHub Action 部署**：同步仓库代码到最新，GitHub Action 会自动重新部署更新。
- **界面部署**：同步代码后 Worker 会自动更新；但绑定的数据库会掉，需要重新绑定 `d1`、`kv` 数据库；然后执行一次 `https://你的 Worker 自定义域/api/init/你的 jwt_secret` 更新数据库（只会更新，不会覆盖已有数据）。
- **命令部署**：本地 `git pull` 拉取最新代码，再跑一次 `pnpm run deploy`，然后访问一次 `/api/init/你的 jwt_secret`。

### API 接口

接口文档较长（登录、邮件列表 / 发送 / 删除、邮箱地址管理、用户管理等），未全部收入本 README，完整版见：

- [接口文档](https://doc.skymail.ink/api/api-doc.html)

调用要点：除「登录」外，接口需在请求头 `Authorization` 中填入登录返回的身份令牌（**不要加 Bearer 前缀**）；管理类接口需要对应权限，管理员账号不受限制。

示例：登录

```bash
curl -X POST "https://skymail.ink/api/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "password"
  }'
```

### 常见问题

- **收不到邮件**：先检查域名 → 电子邮件路由 → Catch-all 是否已设为“发送到 Worker”，以及 `domain` 变量是不是 JSON 数组、域名有没有写错。
- **初始化失败 / 打不开 `/api/init/...`**：检查自定义域是否已绑定、URL 里的 `jwt_secret` 是否与部署时填的一致（且不含 `? % # / \`）。
- **部署时提示 `JWT_SECRET` 为空或含非法字符**：Action 工作流会直接报错退出，重新填一个只含字母数字的密钥。
- **`DOMAIN` 格式报错**：Action 要求 `DOMAIN` 必须是 JSON 数组，例如 `["example.com"]`，不是裸域名 `example.com`。
- **界面部署更新后打不开**：同步代码后 D1 / KV 绑定会掉，重新绑定 `db` / `kv`，再访问一次 `/api/init/你的 jwt_secret`。
- **Cloudflare 发件不可用**：确认已开通 Workers Paid，再检查 `email` 绑定 / `EMAIL` 变量是否已配。
