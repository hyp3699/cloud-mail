## 安装教程

> 本节内容来自官方部署文档 [快速开始](https://doc.skymail.ink/guide/quick-start.html)，个别与文档不一致的地方已在文中注明。
>
> 本教程只讲界面部署：不用装 Node / 不用命令行，在 Cloudflare Dashboard 里操作即可。

### 部署前准备

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

### 环境配置：更多变量

除上面三项必填变量外，还可以在 Worker Variables 里配这些可选变量：

| Worker 变量名 | 默认值 | 必需 | 用途 |
| --- | --- | --- | --- |
| `ai_model` | `@cf/meta/llama-3.1-8b-instruct` | ❌ | Workers AI 模型（验证码识别用） |
| `analysis_cache` | `false` | ❌ | 开启分析页数据缓存（`true` 开启 / `false` 关闭） |
| `project_link` | `true` | ❌ | 隐藏登录界面 GitHub 图标（`true` 显示 / `false` 隐藏） |

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
2. 绑定电子邮件服务到 Worker 绑定，变量值固定为 `email`：在 Worker → Settings → Bindings 添加 Email Sending 绑定，名称 `email`。
3. 到 Cloud Mail 的系统设置里开启 Cloudflare Email。

### 系统设置

以下都在部署完成、登录网站后，在站内「系统设置」里操作。

#### 对象存储（R2）

> 邮件附件默认使用 KV 存储，可以切换为 R2 或其他 S3 协议存储。

1. 在 Cloudflare 创建 R2 对象存储桶。
2. 添加到 Workers 绑定，绑定变量名 `r2`，填 R2 桶名称。

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

同步代码后 Worker 会自动更新；但绑定的数据库会掉，需要重新绑定 `d1`、`kv` 数据库；然后执行一次 `https://你的 Worker 自定义域/api/init/你的 jwt_secret` 更新数据库（只会更新，不会覆盖已有数据）。

### 常见问题

- **收不到邮件**：先检查域名 → 电子邮件路由 → Catch-all 是否已设为“发送到 Worker”，以及 `domain` 变量是不是 JSON 数组、域名有没有写错。
- **初始化失败 / 打不开 `/api/init/...`**：检查自定义域是否已绑定、URL 里的 `jwt_secret` 是否与部署时填的一致（且不含 `? % # / \`）。
- **界面部署更新后打不开**：同步代码后 D1 / KV 绑定会掉，重新绑定 `db` / `kv`，再访问一次 `/api/init/你的 jwt_secret`。
- **Cloudflare 发件不可用**：确认已开通 Workers Paid，再检查 `email` 绑定是否已配。
