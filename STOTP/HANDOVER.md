# STOTP Auth Center 交接文档

> 最后更新：2026-11-22  
> 源码仓库：<https://github.com/Ender-Drate/STOTP>  
> 当前 `origin/main`：`277b030`（2026-11-20）  
> 技术栈：Next.js 16 App Router + React 19 + Node 24 + 内置 `node:sqlite`  
> 本文档不包含管理员密码、SMTP 密码、OIDC RSA 私钥、`config-secret`、TOTP secret、Passkey 私钥。

---

## 1. 项目是什么

STOTP Auth Center 是一个可被第三方系统接入的 **OIDC / OAuth2 统一登录站 + SSO 管理台**。

- 标准 OIDC：授权码 + PKCE(S256)、RS256 id_token/access_token、refresh_token、JWKS、发现文档。
- Scope：`openid`、`profile`、`email`、`head`（头像）、`permission`、`offline_access`。
- 登录方式：密码、Passkey、TOTP 两步验证、外部 OAuth2 登录。
- 管理台：概览、OAuth2 应用、我的授权、用户、权限等级（RBAC）、登录入口、邮件、统计、风险控制、CSP 策略、接入文档。
- 安全能力：服务端会话、RSA 密码信封、scrypt 密码、TOTP 防重放、Passkey、CSP nonce、同源校验、IP 风险/主动防护、3D 场景点击验证码。
- 集成：`integrations/zblog-oauth2/` 是 Z-BlogPHP 插件，支持多 Provider、Passkey 入口和强制本地账号绑定。

它不是单纯的“登录页”，还负责 OAuth2 客户端管理、令牌签发、授权同意、审计和风控。

---

## 2. 代码与文档索引

| 内容 | 位置 |
|---|---|
| OIDC/OAuth2 端点 | `app/api/oauth/` |
| 登录/注册/找回/重置 | `app/login/`、`app/register/`、`app/forgot-password/`、`app/reset-password/` |
| 授权同意页 | `app/consent/` |
| 管理台页面 | `app/dashboard/` |
| 后台组件 | `app/components/` |
| 数据库与用户/令牌 | `lib/db.ts`、`lib/users.ts`、`lib/session.ts` |
| OIDC 校验与 scope | `lib/oidc.ts` |
| CSP | `proxy.ts`、`lib/csp.ts`、`app/dashboard/developer/csp/` |
| Passkey | `app/api/auth/passkey/`、`app/components/Passkey*.tsx`、`lib/webauthn.ts` |
| TOTP | `lib/totp.ts`、`app/components/TotpPanel.tsx` |
| 风控/验证码 | `lib/risk-signals.ts`、`lib/active-defense.ts`、`lib/captcha.ts`、`app/components/ShapeCaptcha.tsx` |
| 主题/语言 | `app/liquid-glass.css`、`app/components/PreferencesDock.tsx`、`lib/ui-i18n.ts` |
| Z-Blog 插件 | `integrations/zblog-oauth2/STOTPOAuth/` |
| Z-Blog 说明 | `integrations/zblog-oauth2/README.md` |
| 截图 | `docs/screenshots/` |
| 自检脚本 | `scripts/`、`package.json` |

关键入口：

```text
proxy.ts                       Node runtime CSP / 安全头
app/layout.tsx                 根布局、主题首屏脚本、偏好 Dock
app/api/oauth/authorize       OAuth2 授权
app/api/oauth/token           code/refresh 换令牌
app/api/oauth/userinfo        OIDC UserInfo
app/api/oauth/jwks            RS256 公钥
app/api/oauth/discovery       OIDC 发现
app/api/auth/passkey/*        Passkey options/verify
app/dashboard/*               管理台
lib/csp.ts                    CSP 默认值、清洗、保存、img-src 头像校验
lib/risk-signals.ts           风险信号与评分
lib/active-defense.ts         IP 临时限流/封禁
```

---

## 3. 运行与部署

### 3.1 本地启动

```powershell
npm install
$env:AUTH_ADMIN_PASSWORD = "change-me"
$env:AUTH_PUBLIC_URL = "http://127.0.0.1:3100"
npm run dev
```

Node 必须 **>= 22.13**，推荐 **Node 24 LTS**。`node:sqlite` 在旧版 Node 上会在请求阶段报
`ERR_UNKNOWN_BUILTIN_MODULE`。

首次启动在用户表为空时创建 `admin`；已有用户时不会自动新增管理员。需要重置管理员时：

```bash
AUTH_ADMIN_PASSWORD='强密码' npm run seed
```

### 3.2 生产

```powershell
npm run build
$env:NODE_ENV = "production"
npm run start
```

`npm run start` 默认只监听 `127.0.0.1:3100`，由同机 nginx 反代；局域网直连用 `npm run start:lan`。
需要自包含包时：

```bash
AUTH_STANDALONE=1 npm run build
node .next/standalone/server.js
```

自包含模式需要把 `.next/static` 复制到 `.next/standalone/.next/static`。

### 3.3 关键环境变量

| 变量 | 说明 |
|---|---|
| `AUTH_PUBLIC_URL` | 对外地址；issuer、发现文档、Passkey RP/Origin、登录跳转都依赖它，必须与浏览器地址一致 |
| `AUTH_DATA_DIR` | SQLite、RSA 密钥目录，默认 `./data` |
| `AUTH_DB_PATH` | 覆盖数据库路径 |
| `AUTH_ADMIN_USERNAME` / `AUTH_ADMIN_EMAIL` / `AUTH_ADMIN_PASSWORD` | 首次启动管理员 |
| `AUTH_TRUST_PROXY` | 默认 `1`，只信任反代覆盖后的 `X-Real-IP` |
| `AUTH_REQUIRE_ENCRYPTED_PASSWORD` | 生产 HTTPS 默认要求 RSA 信封密码 |
| `AUTH_REQUIRE_EMAIL_VERIFICATION` | 默认 `1`；生产不要关闭 |
| `AUTH_SMTP_*` / `AUTH_MAIL_FROM` | SMTP 初始/兜底配置；后台「邮件服务」优先 |
| `AUTH_CONFIG_SECRET` | 后台敏感配置加密密钥；多实例共享 |
| `AUTH_DISPLAY_TZ` | 页面显示时区，默认 `Asia/Hong_Kong` |
| `AUTH_E2E` | 仅本地/CI；生产 HTTPS 下设置会拒绝启动 |

数据目录：

```text
data/auth.db             用户、OAuth 应用、令牌、会话、审计、app_settings
data/oidc-keys.json      RS256 私钥（0600）
data/password-key.json   RSA 密码信封私钥（0600）
data/avatar-secret       头像签名密钥（0600）
data/config-secret       SMTP 等敏感配置加密密钥（0600）
```

**这些文件不要进仓库，不要通过聊天/工单传递；交接时线下转移并确认权限为 600。**

### 3.4 nginx

必须保留/覆盖以下头：

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $remote_addr;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
```

不要再重复添加 CSP、`X-Frame-Options`、`Referrer-Policy`，应用已经下发。
完整配置见 `docs/nginx-auth-center.conf`。

### 3.5 部署注意

1. 备份 `data/`（尤其 `auth.db` 和 RSA 私钥）。
2. 执行 `npm ci && npm run build`。
3. `rm -rf .next` 后重启，避免旧 CSS/路由缓存。
4. `AUTH_PUBLIC_URL` 不要临时改；换域名会让现有 Passkey 失效。
5. 升级后先跑 `npm run check:i18n` 和 `npm run e2e` 冒烟。

---

## 4. 认证与数据模型

### 4.1 OAuth2 / OIDC

端点：

```text
GET  /api/oauth/authorize
POST /api/oauth/token
GET  /api/oauth/userinfo
GET  /api/oauth/head?uid=
POST /api/oauth/revoke
GET  /api/oauth/jwks
GET  /.well-known/openid-configuration
```

要求：

- 只支持 `response_type=code`。
- 公共客户端必须 PKCE(S256) + state。
- 授权码 10 分钟、一次性，绑定 client/redirect_uri/PKCE。
- Access Token 15 分钟；Refresh Token 30 天并轮换。
- `email_verified` 只在邮箱验证通过后为 true；下游不要仅凭邮箱关联账号。

### 4.2 会话

- 服务端 session 表 + HttpOnly/SameSite=Lax Cookie；HTTPS 加 Secure。
- 访问时更新 `last_seen_at`；设置页可撤销单设备/其他设备。
- 改密、重置密码、禁用用户会撤销会话/令牌/Passkey/第三方绑定。

### 4.3 用户状态

- `pending`：注册后待邮箱验证，不签发会话。
- `active`：可登录。
- `disabled`：拒绝登录，已有会话被撤销。
- 权限通过 `roles` + 权限节点，不依赖固定 admin/user。

### 4.4 安全边界

- 密码：scrypt + 随机盐；浏览器提交前 RSA-OAEP 信封加密。
- TOTP：RFC 6238，6 位/30 秒/±1 步；同 timestep 防重放。
- Passkey：只存公钥；RP ID/Origin 从 `AUTH_PUBLIC_URL` 推导；换域名失效。
- 外部 OAuth2：state + PKCE；未绑定本地账号必须显式绑定；不要按邮箱自动接管。
- CSP：`proxy.ts` 按请求生成 nonce；后台 CSP 编辑需要 TOTP。
- 头像：由当前 `img-src` 动态校验；错误文案是「该地址未被允许作为头像」。
- 同源：写请求校验 Origin；反代别名加入 `AUTH_ALLOWED_ORIGINS`。

---

## 5. 管理台与日常操作

### 5.1 权限

- `superadmin`：全部权限，唯一可管理权限等级/CSP。
- `admin`：日常管理。
- `user`：普通用户。
- 权限节点：`admin.access`、`users.view/manage/disable`、`oauth.apps.manage`、`login.providers.manage`、`stats.view`、`audit.view`、`risk.manage`。

### 5.2 后台页面

- OAuth2 应用：创建/编辑/轮换 secret/启停/删除；secret 只显示一次。
- 登录入口：外部 OAuth2 Provider 配置。
- 用户：角色、状态、风险、会话；删除仅最高管理员。
- 权限：自定义权限等级。
- 邮件服务：SMTP 配置/测试，密码加密且不回显。
- 统计：CPU/内存/磁盘/网络/QPS/负载。
- 风险控制：验证码阈值、72h IP、风险观测、IP 观察/封禁、主动防护、审计导出。
- CSP 策略：增删指令/来源；`{{nonce}}` 自动替换；保存/恢复默认需要当前 TOTP。
- 设置：个人资料、邮箱、密码、TOTP、Passkey、第三方绑定、会话。

### 5.3 风控/验证码

- 3D 场景点击验证码：服务端生成 8 个几何体，客户端回传点击坐标；3 分钟、一次性、绑定 IP/用途。
- 主动防护：高风险 IP 临时 `throttle`/`block`，最长 900 秒；不是永久封禁。
- 观测信号：webdriver、UA/WebGL、CDP 痕迹、语言/时区/平台一致性等；只能提高攻击成本。

---

## 6. 前端、主题与 i18n

- 主题：`app/liquid-glass.css` 的 Apple 材质风格；玻璃只用于侧栏/浮层/登录卡/偏好胶囊，内容卡片低反光。
- 偏好：`localStorage` 的 `stotp-theme`（`system/light/dark`）和 `stotp-lang`（`zh/en`）。
- 首屏主题由 `app/layout.tsx` 内联 nonce 脚本提前设置，避免暗色闪白。
- 语言：`lib/ui-i18n.ts` 是客户端 UI 词典 + MutationObserver；不是服务端 locale。
- `npm run check:i18n` 会扫描页面/组件/API 错误文案，CI 已接入。
- 当前边界：OAuth 文档长段落、部分动态 API 错误仍可能保留中文；后续可迁服务端 locale。
- 截图和视觉回归在 `docs/screenshots/`、本地 `.tmp/`。

---

## 7. Z-BlogPHP 插件

位置：

```text
integrations/zblog-oauth2/STOTPOAuth/
integrations/zblog-oauth2/STOTPOAuth.zip
STOTPOAuth.zip（仓库根目录同名副本）
```

### 7.1 能力

- 多个任意 OAuth2/OIDC Provider。
- 授权码 + PKCE、字段映射（Subject/Email/Name）。
- 登录页注入普通 Provider 按钮和「使用 Passkey 登录」按钮。
- 绑定表：`%pre%oauth2_links`（ProviderID、MemberID、SubjectHash、Email）。

### 7.2 强制绑定（重要）

- **绝不调用 `GetMemberByEmail` 判断登录用户**。
- 回调只按 `(Provider, Subject)` 查 link。
- 没有 link：写入一次性 pending，跳转 `oauth2.php?action=bind`。
- 绑定页必须输入本地 Z-Blog **用户名 + 密码**，使用 `Verify_Original` 校验。
- 绑定 token 10 分钟、最多 8 次尝试；成功后写入 link 并登录。
- 已绑定其他本地账号的 subject 拒绝改绑。
- 自动注册和邮箱自动绑定逻辑已移除；邮箱只作绑定记录。
- 没有本地账号时先登录/注册 Z-Blog，再重新发起第三方登录。

### 7.3 Passkey

- 后台全局设置：Passkey 开关、Passkey Provider、Passkey 登录入口。
- 入口留空时默认 `Provider 域名 + /login`。
- 流程：Z-Blog → `oauth2.php?action=start&passkey=1` → STOTP `/login?method=passkey&next=<authorize 相对路径>` → STOTP 自动 WebAuthn → OAuth 回调 → 绑定/登录。
- STOTP 登录页读取 `method=passkey`，自动请求 `/api/auth/passkey/options`。
- 硬件安全密钥若不是 discoverable credential，可能需要在 STOTP 页面填写用户名后再点一次。
- 这是 **STOTP 登录页的 Passkey**，不是 Z-Blog 本地 WebAuthn。

### 7.4 插件验证边界

已做：

```text
PHP 8.3 -l（common/include/main/oauth2）
PHP helper smoke（origin/login URL/callback URL）
npm run build / check:i18n / e2e
浏览器验证 /login?method=passkey 会发起 passkey options
```

未做：完整 Z-Blog + MySQL 实例的绑定页/Passkey 端到端。上线前应在测试站手工走一次
“未绑定 → 绑定页 → 登录 → 再次 Passkey 登录”。

---

## 8. 云端近期变更（2026-11-13 ~ 2026-11-20）

从 `a394980` 到 `277b030`：

- `9ad15ab` 补全后台/表单 i18n（#102）
- `4ba9327` 英文模式浏览器标题回退修复（#102）
- `5d1c129` 几何图形点击验证码（#106）
- `042c510` 3D 场景验证码、前端零凭据、主动防护（#108/#109/#110）
- `252697e` 无中文字体服务器验证码提示乱码修复（#111）
- `277b030` 硬件安全密钥 Passkey 与手机端风控误判修复（#112）

当前分支状态：

```text
origin/main: 277b030
开放 PR: #114  feat/zblog-passkey-binding
  Z-Blog Passkey + 强制绑定
  CI 通过，待合并
```

历史 bundle 只代表旧 commit，不要当作当前 main；需要重新打包。

---

## 9. 已知限制 / 待办

- i18n 是客户端 DOM 词典，不是服务端 locale；API 动态错误、文档长段落仍可能有中文。
- SQLite 单节点；多实例需替换 `lib/db.ts` 为共享数据库。
- 主动防护是有时限流，不是 WAF；高水平和分布式攻击仍需 nginx/WAF。
- 3D 验证码不是无感风控，不能保证对高级打码平台免疫。
- Passkey 需要 HTTPS/`localhost`；改 `AUTH_PUBLIC_URL` 会使已注册 Passkey 失效。
- 外部登录 client_secret 在 `data/auth.db`，限制文件权限并备份。
- Z-Blog 插件完整实例测试未跑；Passkey 依赖 STOTP 登录页。
- 合并 #114 后需要更新 `README.md` 的 Z-Blog 功能描述、重新打 `STOTPOAuth.zip`/bundle。

---

## 10. 交接检查清单

- [ ] 拿到 `AUTH_PUBLIC_URL`、部署路径、nginx 配置、进程管理器（systemd/pm2）说明。
- [ ] 线下接收 `data/auth.db`、`data/oidc-keys.json`、`data/password-key.json`、`data/avatar-secret`、`data/config-secret`。
- [ ] 确认以上文件权限为 600，已做离线备份。
- [ ] 确认管理员账号、SMTP、TOTP/Passkey 绑定状态。
- [ ] 确认备份/恢复流程，并实际恢复一次到临时目录。
- [ ] 确认 Z-Blog 插件是否启用、Provider 配置、绑定表数据。
- [ ] 确认旧 bundle 不用于新部署；按当前 main 重新构建。
- [ ] 跑 `npm run check:i18n`、`npm run e2e`、`npm run e2e:security`。
- [ ] 在测试站验证 Z-Blog 首次绑定与 Passkey 登录。
- [ ] 记录本次交接的 commit、bundle hash、部署时间。

---

## 11. 相关链接

- STOTP：<https://github.com/Ender-Drate/STOTP>
- Z-Blog 插件说明：<https://github.com/Ender-Drate/STOTP/blob/main/integrations/zblog-oauth2/README.md>
- 接入文档（运行中）：`/docs`
- OIDC Discovery：`/api/oauth/discovery`
- 本交接仓库：<https://github.com/Ender-Drate/Introduction>
