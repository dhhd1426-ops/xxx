# 项目交接说明（给下一个 Claude）

> 本文件记录这个仓库的部署进度、踩过的坑和下一步计划。仓库是**公开的**：不要在这里或任何提交里写入令牌、账户 ID、JWT 密钥等敏感值，它们只存在 GitHub Secrets 里。
>
> 用户使用中文交流，主要用手机操作（截图沟通），请用中文回复，步骤写细、一次一步。

## 这是什么

- 本仓库是 [maillab/cloud-mail](https://github.com/maillab/cloud-mail)（MIT 许可）的副本，未改动源码。
- cloud-mail：部署在 Cloudflare Workers 上的域名邮箱系统（Workers + D1 + KV + Durable Objects，前端 Vue 在 `mail-vue/`，后端在 `mail-worker/`）。
- 部署方式：GitHub Actions 工作流 `.github/workflows/deploy-cloudflare.yml`（只在 `mail-worker/**`、`mail-vue/**` 变更或手动 Run workflow 时触发）。

## 当前配置（2026-11-20）

| 项目 | 值 |
|---|---|
| 邮件域名 | `fabledream.de5.net`（DNSHE 免费二级域名，到期 2027-11-20，需在 DNSHE 续期） |
| DNS 托管 | Cloudflare（NS：`hadlee.ns.cloudflare.com` / `major.ns.cloudflare.com`，在 DNSHE 的“DNS服务器”里设置） |
| Cloudflare 账号 | 用户的 Gmail 账号，免费计划 |
| workers.dev 子域名 | `zexidream.workers.dev` |
| 网站地址 | https://cloud-mail.zexidream.workers.dev |
| Worker 名 | `cloud-mail` |
| D1 / KV | 均名为 `cloud-mail`，由工作流自动创建 |
| R2 | 未配置（避免绑卡），附件存储不可用 |
| 管理员（ADMIN） | `admin@fabledream.de5.net`（**用户尚未注册这个账号**） |
| 已注册用户 | `zexidream@fabledream.de5.net`（普通用户，不是管理员） |

GitHub Secrets（Settings → Secrets and variables → Actions）已设置：
`CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`、`DOMAIN`（`["fabledream.de5.net"]`）、`ADMIN`、`JWT_SECRET`。
未设置：`PASSWORD`、`CUSTOM_DOMAIN`、`R2_BUCKET_NAME`、`EMAIL`、`KV_NAMESPACE_ID`、`D1_DATABASE_ID`。

API 令牌 = “Edit Cloudflare Workers” 模板 + 额外账户策略 **D1 → Edit**。

Cloudflare Email Routing（fabledream.de5.net）：
- 已启用，MX（route1/2/3.mx.cloudflare.net）、SPF、DKIM 记录已自动添加。
- 路由规则：
  - Catch-all → Send to a Worker → `cloud-mail`（已设置并启用，但见下方“问题 5”）
  - `zexidream@fabledream.de5.net` → Send to a Worker → `cloud-mail`（单独规则，加上后收信成功）

**状态：部署成功，收信已打通（Gmail → zexidream@ 能收到）。发信尚未配置。**

## 遇到的问题和解决办法

1. **工作流第一次运行失败**：代码推送时自动触发，当时 Secrets 还没填。→ 填好 Secrets 后手动 Run workflow。
2. **Set up KV database 失败**：`Could not route to /client/v4/accounts/***/...` `[code: 7003]`。原因是 `CLOUDFLARE_ACCOUNT_ID` 抄错（从截图识别时漏了字符，应为 32 位十六进制）。→ 用 Cloudflare 页面的复制按钮重新粘贴。**以后不要从截图手抄 ID，让用户点复制按钮。**
3. **Start deployment 失败**：`You need a workers.dev subdomain in order to proceed [code: 10063]`。新账号没打开过 Workers 页面。→ 在 Cloudflare 打开“计算 → Workers & Pages”即自动创建子域名，然后重跑。
4. **Gmail 发来的邮件被退信 `550 5.1.1 Address does not exist`**，且 Email Routing 的 Activity log 为空。DNS（MX/SPF/DKIM）检查无误，Catch-all 也显示启用。→ 新建针对 `zexidream@` 的单独路由规则后收信成功。
5. **未确认的问题**：Catch-all 规则是否真正生效还不确定（可能只是当时生效延迟，也可能确实没生效）。下一步需测试发往一个**没有单独规则**的地址（如 `test123@fabledream.de5.net`）能否收到。
6. 浏览器自动翻译会把 Cloudflare 界面翻得很怪（如 “Disable”→“取消平”、“Catch-all”→“万能”、“Workers”→“工作人员”、“Onboard domain”→“机载域名”）。解释截图时注意这一点。
7. 工作流有两条可忽略的警告：`CUSTOM_DOMAIN variable is not set`、`Node.js 20 is deprecated`。

## 下一步

1. **验证 Catch-all**：给一个没有单独规则的新地址发邮件。若被 550 退信，说明 Catch-all 没生效：可以删掉重建，或者每新增一个邮箱就加一条单独规则。
2. **注册管理员**：在网站上注册 `admin@fabledream.de5.net`，登录后进入系统设置（注册开关、用户权限等）。
3. **配置发信**（如用户需要），二选一：
   - Cloudflare Email Sending：在 Secrets 里加 `EMAIL`（任意非空值），工作流会追加 `[[send_email]] name = "email"` 绑定，然后重跑。可能需要先在 Cloudflare 开通 Email Sending（Beta）。
   - Resend：在管理员后台的设置里填 Resend API Key（代码见 `mail-worker/src/service/resend-service.js`）。
4. 可选：设置 `CUSTOM_DOMAIN`（例如 `mail.fabledream.de5.net`）让网站用自己的域名；配置 R2 以支持附件（需要绑卡）。
5. 同步上游更新：从 maillab/cloud-mail 拉取更新后推送到 main，工作流会自动重新部署。

## 提醒用户的风险

- 谷歌等大平台可能不接受 de5.net 这类免费二级域名的邮箱注册，或者会要求手机验证。
- 用这个系统批量注册第三方平台账号，容易被平台风控并整批封号，也可能违反这些平台的使用条款。
