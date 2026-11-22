# Vote System（投票系统）交接文档

> 最后更新：2026-11-22
> 源码仓库：<https://github.com/Ender-Drate/vote-system>
> 当前 `origin/main`：`3cc916a`（2026-11-09）
> 待合并：PR [#42](https://github.com/Ender-Drate/vote-system/pull/42) `fix/issue-41-admin-poll-limits`（`f386ee7`，2026-11-20，`Closes #41`）
> 技术栈：Go 1.22（后台：TCP RPC + 自带管理界面，`net/tcp` + `html/template` + `go:embed`）+ PHP 8.1（前台）+ MySQL 8.0 / 5.7
> 本文档**不包含** `rpc_secret`、`csrf_secret`、数据库密码、管理员密码、OAuth `client_secret`、生产域名与服务器地址。

---

## 1. 项目是什么

一个**前后台分离、匿名、单 IP 单票**的投票系统，面向社区/班级一类的小型公开投票。

- **前台（PHP）**：投票列表、投票页、结果页、验证码图片；不连数据库，所有数据经本机 TCP RPC 找后台要。
- **后台（Go）**：投票内核（校验、限流、风控）+ 自带管理界面（独立端口、独立会话）；唯一能连 MySQL 的进程。
- **题型**：单选 / 多选 / 简答 / 排序。
- **匿名**：没有账号体系，也没有登录态 Cookie；去重只靠 IP（IPv4/IPv6 都以 16 字节 IPv4-mapped 形式存储）。
- **反自动化**：图片验证码 + 工作量证明（PoW）双层。
- **管理端**：投票增删改、状态切换、结果统计（含按 IP 聚合）、IP 规则、审计日志、风险评分、管理员管理、危险操作二次授权（外部 OAuth2）。

它不是问卷系统：没有条件跳转、没有分页、没有多租户，也不做实时刷新。

---

## 2. 架构与信任边界

```text
        公网
         │ HTTPS
         ▼
┌──────────────────────┐        ┌────────────────────────┐
│ PHP 前台 (php-fpm)    │  TCP   │ Go 后台 voted           │        ┌─────────┐
│ public/ 为唯一根目录  │ ─────▶ │  · RPC  127.0.0.1:7788  │ ─────▶ │  MySQL  │
│ 不直连数据库          │  HMAC  │  · 管理 127.0.0.1:7789  │  SQL   │ 8.0/5.7 │
└──────────────────────┘        └────────────────────────┘        └─────────┘
         ▲                                  ▲
         │        内网 / VPN / IP 白名单      │
    （公众）                          管理员浏览器
```

三条硬规则：

1. **前台不碰数据库**。读写都走 RPC 方法白名单，没有通用 SQL 通道。
2. **RPC 只监听回环**，每帧 HMAC-SHA256 签名；密钥不符直接断开并记日志，不返回业务数据。
3. **管理界面是独立端口/进程**，与前台没有共享会话或 Cookie，只通过 MySQL 交换数据。

为什么用裸 TCP 而不是 HTTP：帧自带时间戳与随机 nonce，重放窗口在服务端关闭；只需要一个共享密钥，没有 JWT/Cookie 传递。

---

## 3. 代码与文档索引

| 内容 | 位置 |
|---|---|
| 程序入口（RPC + 管理界面 + 控制台） | `backend/cmd/voted/main.go` |
| 业务规则（校验、限流、验证码、统计） | `backend/internal/vote/` |
| TCP 帧格式 / HMAC / 防重放 | `backend/internal/protocol/` |
| RPC 方法分发 | `backend/internal/rpc/server.go` |
| 管理界面（handler / 模板 / 静态资源） | `backend/internal/admin/` |
| 存储接口与两种实现 | `backend/internal/store/`（`store.go` / `mysql/` / `memory/`） |
| 迁移 SQL | `backend/internal/migrations/0001~0007*.sql` |
| 配置加载与默认值 | `backend/internal/config/config.go` |
| 图片验证码 | `backend/internal/vote/service.go`（`renderCaptcha`） |
| 工作量证明 | `backend/internal/pow/` |
| 风控打分 | `backend/internal/risk/risk.go` |
| 运维控制台 | `backend/internal/console/console.go` |
| 演示数据（memory 驱动） | `backend/internal/demo/seed.go` |
| 前台路由与控制器 | `frontend/public/index.php`、`frontend/app/Router.php`、`frontend/app/Controllers/` |
| 前台 RPC 客户端 | `frontend/app/Rpc/Client.php` |
| 前台模板 | `frontend/views/` |
| 前台静态资源 | `frontend/public/assets/` |
| 前台文案（zh/en） | `frontend/app/lang/zh.php`、`en.php` |
| 部署样例 | `deploy/systemd/voted.service`、`deploy/nginx/vote-system.conf.example` |
| 文档 | `docs/ARCHITECTURE.md`、`PROTOCOL.md`、`SECURITY.md`、`RISK-CONTROL.md`、`DEPLOYMENT.md`、`DESIGN.md` |

关键入口：

```text
backend/cmd/voted/main.go              启动 RPC(7788)、管理界面(7789)、控制台、过期清理
backend/internal/vote/service.go       Submit 校验链、validateAnswers、aggregate 统计
backend/internal/admin/server.go       管理端路由、登录/会话/CSRF、投票保存
backend/internal/admin/view.go         表单解析、题目校验、回显
backend/internal/admin/static/admin.js 题目编辑器（切题型要保留选项）
frontend/app/Controllers/PollController.php  投票页渲染、提交、结果
```

---

## 4. 运行与部署

### 4.1 本地跑通（无需 MySQL）

```powershell
# 后台（终端 A）
cd backend
copy config\config.example.yaml config\config.yaml
# 改：env: development、database.driver: memory、demo.seed: true
go run ./cmd/voted -config ./config/config.yaml

# 前台（终端 B）
cd frontend
copy config\config.example.php config\config.php
# 两边 security.rpc_secret / backend.rpc_secret 必须完全一致
php -S 127.0.0.1:8080 -t public dev-router.php
```

`demo.seed=true` 会创建管理员（`demo.admin_username`，密码留空则随机打印到日志）和一个包含四种题型的示例投票。
内存驱动重启即丢数据，只用于本地。

### 4.2 生产部署

```bash
# 后台
cd backend
go build -o bin/voted ./cmd/voted        # 需要 Go >= 1.22（go.mod 里是 1.22.0）
./bin/voted -config /etc/vote-system/backend.yaml

# 前台：public/ 作为站点根，php-fpm 运行
```

- `deploy/systemd/voted.service`：`ExecStart=/srv/vote-system/backend/bin/voted -config /etc/vote-system/backend.yaml`，`WorkingDirectory=/srv/vote-system/backend`，`ProtectSystem=strict`。
- `deploy/nginx/vote-system.conf.example`：前台站点的反代/静态示例；管理端口建议再加 IP 白名单。
- 两份配置（`backend/config/config.yaml`、`frontend/config/config.php`）都在 `.gitignore` 里，**不进仓库**；交接时线下转移。
- 迁移在**后台启动时自动执行**（`internal/migrations` → `schema_migrations`）；DB 账号需要 `CREATE/ALTER`。

### 4.3 当前线上实况（交接时确认）

- 部署目录在 SSD 上（仓库根 `Vote_System/`，同一块盘有多个挂载点）；`backend/` 与 `frontend/` 同级。
- **没有用 systemd**：`backend/start.sh` 里是
  `<Go 安装目录>/bin/go run ./cmd/voted -config ./config/config.yaml`，以 root 运行。
  - 因为是 `go run`，**改完代码只要重启进程就会重新编译生效**，不需要单独 `go build`。
  - 但进程必须在 `backend/` 目录下启动（路径都是相对的）；重启方式取决于它现在挂在哪（tmux/screen/nohup），先 `ss -lptn 'sport = :7789'` 找到 PID 再决定。
  - 强烈建议改成 systemd + `go build -o bin/voted`：`go run` 每次都编译、进程树多一层、日志不好收。
- 端口：RPC `127.0.0.1:7788`、管理端 `127.0.0.1:7789`（默认值）。
- 管理界面的模板与静态资源是 `//go:embed` **编译进二进制的**：换成预编译二进制后，改模板必须重新 `go build` 并重启，否则页面不会变。

### 4.4 升级步骤

```bash
cd $VOTE_ROOT
git fetch --all && git log --oneline HEAD..origin/main     # 先看会升什么
git merge --ff-only origin/main                            # 或按分支部署
cd backend && go build -o bin/voted ./cmd/voted            # 仅预编译部署需要
sudo systemctl restart voted                               # 或停掉 go run 进程后重启
```

1. 先备份数据库（见 §14）。
2. 迁移只向前，不回滚；回退二进制前要确认旧版本能读新列。
3. 前后台要**一起**升级：新错误码的文案在前台 `lang/*.php`，只升后台会显示成通用错误。
4. 升级后跑 §12 的冒烟：建投票 → 投一票 → 看结果。

---

## 5. 数据模型与迁移

| 表 | 作用 |
|---|---|
| `admins` | 管理员：bcrypt 密码、失败计数/锁定、TOTP 遗留列、OAuth2 绑定 |
| `admin_sessions` | 会话令牌（只存 `sha256(token)`）、IP/UA、绝对与空闲过期 |
| `polls` | 投票：slug、状态、结果公开方式、`one_per_ip`、时间窗、`max_total_choices` |
| `questions` | 题目：类型、必答、`min_choices`/`max_choices`、字数上限 |
| `options` | 选项：按 `position` 排序 |
| `votes` | 票：`ip`（展示）、`ip_key`（唯一键去重）、`ua_hash`、回执 `receipt` |
| `vote_answers` | 答案：`choices JSON` / `rank JSON` / `text_value` |
| `captchas` | 图片验证码：`answer_hash`、尝试次数、一次性消费、过期 |
| `ip_rules` | IP/CIDR 的 block/allow 规则，可带过期时间 |
| `audit_logs` | 审计：登录、投票增删改、状态、IP 规则、管理员操作 |

迁移：

```text
0001_init.sql                     建全部表
0002_captchas_answer_hash.sql     补 captchas.answer_hash（否则验证码永远校验失败）
0003_votes_ip_key.sql             唯一键 (poll_id, ip) → (poll_id, ip_key)，支持关闭"一 IP 一票"
0004_votes_ip_index.sql           idx_votes_ip (ip, created_at)，按 IP 统计
0005_admins_totp.sql              admins.totp_*（历史，已停用）
0006_admins_oauth.sql             admins.oauth_subject/email/bound_at（危险操作二次授权）
0007_polls_max_total_choices.sql  polls.max_total_choices（整份投票最多票数，PR #42）
```

`votes` 的去重语义：`one_vote_per_ip=true` 时 `ip_key = ip`；关掉时每票随机 16 字节，唯一性仍由数据库兜底。

---

## 6. 投票规则与校验链

`vote.submit` 依次校验（顺序有意义，改前先读 `docs/ARCHITECTURE.md`）：

```text
1 帧签名 / 时间戳 / nonce（协议层，失败直接断开）
2 IP 规则：block 命中即拒；allow 命中放行；都没命中看 limits.ip_default_action
3 限流：limits.vote_per_ip_per_minute（默认 3/分/IP，跨投票）
4 投票状态与时间窗（status=open 且 starts_at <= now <= ends_at）
5 一 IP 一票（若开启）：查 votes(poll_id, ip)，命中 → already_voted
6 答案结构（见下）
7 图片验证码：存在、未过期、未消费、IP 一致、答案正确、尝试次数 < 上限
8 PoW：签名、有效期、难度、nonce 未用过、算力达标
9 事务写 votes + vote_answers；唯一键冲突 → already_voted；receipt 撞车重试 3 次
```

**答案结构**（`validateAnswers`）：

- 题目必须属于该投票，同一题不能重复提交；选项必须属于该题，多选题去重。
- 单选必须恰好 1 项；排序必须给**全部**选项排序且不重复。
- 多选：`min_choices` 是最少勾选数，`max_choices` 为 0 表示不限。
  - `min_choices = 0` 表示"可以一项都不选"：此时"必答"只要求**回答这道题**，空 `choices` 是合法答案，且**不写** `vote_answers`（不影响结果页"份数"）。
- 简答：去首尾空白、长度 ≤ `max_length`（且受 `limits.answer_text_max` 封顶）、拒绝控制字符。
- **整份投票上限**：`polls.max_total_choices > 0` 时，一次提交里所有单选/多选勾选项数之和不得超过它；超限返回 `too_many_choices`。排序题（全排序）和简答题不计数。

结果公开方式：`never` / `after_vote` / `always`。关掉"一 IP 一票"后没有"这个人投过没有"的概念，`after_vote` 会退化成"立即公开"。公开结果不含简答原文，管理端才看得到。

RPC 错误码（详见 `docs/PROTOCOL.md`）：`unknown_method`、`invalid_argument`、`poll_not_found`、`poll_closed`、`already_voted`、`captcha_invalid`、`captcha_expired`、`captcha_disabled`、`ip_blocked`、`rate_limited`、`too_many_choices`、`results_hidden`、`internal`。
`message` 只给运维看，前台不直接展示（开发模式除外）。

---

## 7. 人机验证（验证码 + PoW）

### 7.1 图片验证码

- 尺寸固定 320×100；数字位数 `captcha.length`（默认 4，范围 4~8，位数越多越糊）。
- 库里只存 `sha256(id ‖ 0 ‖ 答案 ‖ 0 ‖ rpc_secret)`，PNG **只在进程内存**，重启即失效。
- 有效期 `captcha.ttl`（默认 5 分钟）；同一 id 最多尝试 `max_attempts`（默认 5）次，用尽即作废。
- 消费是原子的（`UPDATE … WHERE consumed_at IS NULL`），"解一次、反复提交"不成立。
- 每个 IP 每分钟最多申请 `create_per_ip_per_minute`（默认 10）张。
- 验证码与 IP 绑定，换 IP 使用直接失败。

### 7.2 工作量证明

```text
digest  = sha256(salt + ":" + counter)      # counter 是十进制字符串
通过条件 = digest 的前导零比特数 >= bits
```

- `base_bits` 17 / `max_bits` 21 / 每 `escalate_every` 3 次申请 +1 位（窗口 `window` 10 分钟）。
- 挑战与解都一次性，有效期 `ttl` 5 分钟；重放返回 `captcha_expired`。
- 服务端签名覆盖 `nonce|salt|bits|expires|ip`，客户端改不了难度也换不了 IP 复用。
- 前端在 Web Worker 里求解（`frontend/public/assets/pow-worker.js`），17 bit 桌面约几十毫秒。

两层都关（`captcha.enabled`/`captcha.pow.enabled = false`）时 RPC 照常工作，只用于压测/本地。

---

## 8. 管理端

### 8.1 能力

- **投票**：新建/编辑（标题、简介、状态、结果公开方式、开始/结束时间、**整份投票最多票数**）、题目编辑器（单选/多选/简答/排序，选项增删、必答、最少/最多、字数上限）、改状态、查看结果。
  - 有票之后题目与选项变**只读**（服务端还会再比对一次差异，防止手工构造请求改坏历史答案）。
  - 校验失败会把已填内容回显，不会清空整张表单。
  - 题目编辑器切换题型**不会丢选项**（切前把题干/提示/必答/上下限/选项读回状态对象）。
- **结果**：每题选项计数/百分比、排序题平均名次与第一名次数、简答份数与原文（仅管理端）、按 IP 聚合的票数（识别刷票）。
- **IP 规则**：CIDR block/allow，可设过期时间；`ip_default_action` 可设白名单制。
- **审计**：最近 300 条操作 + 风险评分面板。
- **管理员**：新建/停用/删除（至少保留一个、不能删自己）、改自己的密码（改密会吊销其它会话）。
- **账号**：OAuth2 外部身份绑定/查看；解绑只能走服务器控制台。
- **危险操作**（`/super/danger`）：按 IP 删某投票的票、删除有票的投票；必须**重新走一次 OAuth2 授权**，pending 5 分钟一次性。

### 8.2 安全机制

- 密码 bcrypt（`admin_bcrypt_cost` 默认 12，允许 10~15）。
- 登录失败 5 次锁 15 分钟；同一 IP 10 分钟内最多 20 次尝试（进程内计数）。
- 会话：32 字节随机 token 放 Cookie，库里只存 `sha256(token)`；HttpOnly + `SameSite=Strict`，HTTPS 必须 `admin_cookie_secure: true`；绝对 8 小时、空闲 30 分钟。
- CSRF：每个 POST 校验 `HMAC(rpc_secret, "csrf|" + sha256(session_token))`；登录页用 IP + 当天日期派生的匿名令牌。
- 管理端 IP 白名单 `server.admin_allowed_cidrs`，名单外一律 403。
- 响应头：CSP（`default-src 'self'; img-src 'self' data:; style-src 'self'; script-src 'self'; form-action 'self'; base-uri 'none'; frame-ancestors 'none'`）、`X-Content-Type-Options`、`X-Frame-Options: DENY`、`Referrer-Policy`、`Cache-Control: no-store`。
- 删除确认用 `data-confirm` + 事件委托（CSP 禁内联脚本，`onsubmit` 会被拦掉）。

### 8.3 运维控制台

后台进程的标准输入是一个交互控制台（systemd 下可用 `StandardInput=tty` 或本地直接跑）：

```text
list admins                列出管理员
create admin <用户名>       创建管理员，打印一次性密码
reset password <用户名>      重置密码，打印一次性密码
unbind oauth <用户名>        解绑外部 OAuth2 身份（控制台不可交互时用 -unbind-oauth 参数）
help / quit
```

空库启动时不会自动建管理员：先 `create admin`，或用 `demo.seed`（仅 memory 驱动）。

---

## 9. 风控与审计

`risk.Evaluate` 只**打分不拦截**：按 IP 聚合窗口内的票，输出 `low/medium/high`，由人在审计页/危险操作页处理。

| 信号 | 阈值（示例配置） | 分值 |
|---|---|---|
| 同一 UA 出现在多个 IP（分布式刷票主信号） | ≥ 3 个 IP | +3 |
| 同 /24 网段票数 | ≥ 8 票 | +3 |
| 单 IP 票数 | ≥ 4 → +3；≥ 10 → +5 | +3 / +5 |
| 60 秒内峰值 | ≥ 3 票 | +4 |
| 横跨投票数 | ≥ 2 | +2 |
| 投票间隔过于规律（CV） | < 0.25 | +3 |
| 同一 IP 多个 UA | ≥ 2 | +2 |
| 没有 UA | — | +1 |

- 跨 IP 信号在 `min_votes`（默认 2）之前计算：分布式刷票每个 IP 只投一票，先按票数切掉就会漏掉最典型的攻击。
- 分档：≥ 6 `high`，≥ 3 `medium`，其余 `low`。
- 阈值全部在 `risk:` 配置段，可调。

---

## 10. 前台页面与 i18n

| 路由 | 页面 |
|---|---|
| `/` | 进行中的投票列表 |
| `/p/{slug}` | 投票页（四种题型 + 人机验证） |
| `/p/{slug}/submit` | 提交（POST：CSRF + 验证码 + PoW + 一 IP 一票） |
| `/p/{slug}/result` | 统计结果（按 `show_results` 决定是否公开） |
| `/captcha/{id}.png` | 验证码图片 |

- 无登录态、无 Cookie；CSRF 是 `HMAC(csrf_secret, ts|ip|purpose)` 的无状态令牌，默认 30 分钟。
- 语言：`zh-CN` / `en`，`?lang=en` 切换，`Accept-Language` 兜底；词典在 `frontend/app/lang/`。
- 视觉规范见 `docs/DESIGN.md`（配色对比度、玻璃层级、响应式与可访问性）。
- 投票页会显示「整份投票最多选 N 项 · 已选 M」，超限标红并拦下提交（服务端仍是权威）。
- 已知小瑕疵：`captcha.enabled=false` 时投票页仍会渲染"验证码加载失败"区块（功能不受影响）。

---

## 11. 测试与自检

```bash
cd backend
go vet ./...
go test ./...                 # 单元 + 管理端 HTTP + 协议 + PoW + 风控 + 存储

# MySQL 集成测试（默认跳过）
VOTE_TEST_MYSQL_DSN='vote:pw@tcp(127.0.0.1:3306)/vote_system_test?charset=utf8mb4&parseTime=true' \
  go test ./internal/store/mysql -run Integration -v

# PHP 语法
cd frontend && find . -name '*.php' -exec php -l {} \;
```

- 仓库**没有配置 CI**（没有 `.github/`），以上命令要手工跑。
- 管理端题目编辑器是前端 JS，跑不了 Go 测试；改动 `admin.js` 后至少手工验证：切题型选项还在、`min=0` 显示 0、保存后 `q{i}_min` 提交为 0。
- 冒烟：建一个投票 → 投一票 → 看结果页数字 → 后台审计页有记录。

---

## 12. 近期变更与当前状态

```text
origin/main : 3cc916a  修复代码审计发现的问题（管理端保存、CSP、结果可见性、IP 伪造）(#39)  2026-11-09
开放 PR     : #42      fix/issue-41-admin-poll-limits（f386ee7）                        2026-11-20
                        Closes #41，状态 MERGEABLE，未合并
```

PR #42 修的是 issue #41 的三个问题：

1. **多选「最少选择」填 0 被改回 1** —— `parseQuestions` 里 `MinChoices < 1` 强行改成 1；现在 0 原样保存，空答案合法。
2. **没有「整份投票最多投 N 票」** —— 新增 `polls.max_total_choices`（迁移 0007）+ 管理端输入框 + 前台计数提示 + 服务端 `too_many_choices` 校验。
3. **切换题型选项丢失** —— 题目编辑器切题型前先把选项读回状态对象再重建。

配套改动：校验失败回显表单、改状态保留上限、新增 `too_many_choices` 错误文案、文档（PROTOCOL/ARCHITECTURE/SECURITY）与回归测试。

注意：

- **bundle 只是中转产物**（`vote-system-issue41.bundle` 等），只代表打包时的那个 commit，**不要当最新 main 用**；需要时按 `origin/main` 重新打包。
- 合并 PR #42 后要重新构建后台并在测试环境走一遍：建「最少 0 项 + 整体 3 票」的投票，验证空答案能提交、选 4 项被拒。

---

## 13. 已知限制 / 待办

- **没有 CI**：迁移、Go 测试、PHP 语法、前端 JS 都靠手工。建议接 GitHub Actions（`go vet`/`go test` + `php -l`）。
- **结果页聚合在内存里做**（`vote.aggregate`）：票数上万仍可，十万级需要预聚合/物化统计表。
- **单实例内存态**：限流令牌桶、验证码 PNG、PoW nonce 缓存都在进程内；多实例部署会不一致（PNG 只有生成它的进程有）。要横向扩展得先把这些外置。
- **验证码依赖图片**：只能用 OCR 破解，PoW 提高成本；不是无感风控。
- **TOTP 已停用**：`0005` 的 `admins.totp_*` 列保留未清理，`internal/totp` 已无调用方（只剩自身测试）；危险操作改走外部 OAuth2 —— 未启用 `oauth.enabled` 时**无法**执行按 IP 删票/删有票投票。
- **管理端只有中文**；前台 zh/en 两套。
- **排序题要求全排序**，选项很多时体验一般（没有拖拽，只有上下按钮）。
- **部署现状是 `go run`**（见 §4.3），建议改 systemd + 预编译二进制；改模板必须重编。
- **`frontend/config/config.php` 与 `backend/config/config.yaml` 不在仓库**，换机器必须重新创建，且两边 `rpc_secret` 必须一致。
- **没有账号体系**：纯匿名、按 IP 去重，同一 NAT 后的人只能投一次（这是设计目标，不是缺陷）。
- 迁移只向前：回滚版本前先确认旧二进制能读新结构。

---

## 14. 交接检查清单

- [ ] 拿到生产域名、nginx 配置、TLS 证书位置、进程管理方式（现在是不是 `go run`？谁在拉起？）。
- [ ] 线下接收 `backend/config/config.yaml`、`frontend/config/config.php`（含 `rpc_secret` / `csrf_secret`），确认权限仅运行用户可读。
- [ ] 拿到 MySQL 地址、账号、备份方式；实际做一次 `mysqldump` 并恢复到临时库。
- [ ] 确认 `schema_migrations` 已到 `7`，`polls` 有 `max_total_choices` 列。
- [ ] 确认管理员账号可用、`oauth.enabled` 与绑定状态（危险操作依赖它）。
- [ ] 确认 `server.admin_allowed_cidrs` 与管理端监听地址（务必回环 + 白名单）。
- [ ] 跑 `go vet ./...`、`go test ./...`、`php -l`。
- [ ] 冒烟：建投票 → 前台投一票 → 结果页数字正确 → 审计页有记录。
- [ ] 确认 PR #42 是否已合并；未合并则按分支部署，合并后按 main 重新构建。
- [ ] 记录交接时的 commit、bundle hash（如有）、部署时间。

---

## 15. 相关链接

- 仓库：<https://github.com/Ender-Drate/vote-system>
- Issue #41：<https://github.com/Ender-Drate/vote-system/issues/41>
- PR #42：<https://github.com/Ender-Drate/vote-system/pull/42>
- 架构：<https://github.com/Ender-Drate/vote-system/blob/main/docs/ARCHITECTURE.md>
- 协议与错误码：<https://github.com/Ender-Drate/vote-system/blob/main/docs/PROTOCOL.md>
- 安全：<https://github.com/Ender-Drate/vote-system/blob/main/docs/SECURITY.md>
- 风控：<https://github.com/Ender-Drate/vote-system/blob/main/docs/RISK-CONTROL.md>
- 部署：<https://github.com/Ender-Drate/vote-system/blob/main/docs/DEPLOYMENT.md>
- 本交接仓库：<https://github.com/Ender-Drate/Introduction>
