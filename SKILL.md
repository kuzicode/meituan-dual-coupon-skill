---
name: "meituan-dual-coupon-skill"
displayName: "美团双渠道每日领券"
description: "每天自动领取两个互不重复渠道的美团优惠券并合并报告：渠道一为美团开放平台在 ClawHub 发布的官方 skill「美团红包助手」（手机号+短信登录）；渠道二为 WorkBuddy 渠道可移植领券技能（美团 App 扫码登录）。当用户提到「美团领券、美团红包、领优惠券、双渠道领券、每日自动领券」时使用。"
---

# 美团双渠道每日领券

两个渠道的券不重复、登录相互独立，一个失败不影响另一个。每日依次领取，合并成一条报告。

**本文件是唯一权威说明。** `channel*/UPSTREAM.md` 是上游原文，仅供查话术/错误码；与本文件冲突时以本文件为准（例如不做渠道一的 WebFetch 版本检查、不用其中的 `xiaomei-cowork` / `~/.claude/skills` 路径）。

## 目录结构

```
<PKG>/                         # 本仓库根目录，下文统一写作 <PKG>（用绝对路径）
├── SKILL.md  README.md  AGENTS.md
├── channel1-clawhub/          # 渠道一：ClawHub 官方「美团红包助手」v1.0.6 原包
│   ├── scripts/auth.py issue.py query.py
│   ├── references/terms-of-service.md
│   └── credentials/           # 登录后生成，已 gitignore
└── channel2-workbuddy/        # 渠道二：可移植领券技能
    ├── scripts/run.js         # 唯一入口
    └── credentials/           # 登录后生成，已 gitignore
```

## 环境变量（每条命令都带上）

```bash
TZ=Asia/Shanghai
# 渠道一
XIAOMEI_AUTH_FILE=<PKG>/channel1-clawhub/credentials/auth_tokens.json
XIAOMEI_COUPON_HISTORY_FILE=<PKG>/channel1-clawhub/credentials/mt_coupon_history.json
# 渠道二（不设则默认 ~/.workbuddy/credentials/meituan-living-deals-assistant/）
MEITUAN_COUPON_AUTH_DIR=<PKG>/channel2-workbuddy/credentials
```

若渠道一报 httpx `Invalid port`（系统 NO_PROXY 含 IPv6 条目），再加 `no_proxy=localhost,127.0.0.1 NO_PROXY=localhost,127.0.0.1`。

## 环境准备（一次性）

- 渠道一：Python 3 + httpx。
  ```bash
  cd <PKG>/channel1-clawhub && python3 -m venv .venv && .venv/bin/pip install httpx
  ```
  `<PY>` = `<PKG>/channel1-clawhub/.venv/bin/python`（Windows：`.venv\Scripts\python.exe`）。
- 渠道二：node ≥ 18、npm、python3。`node <PKG>/channel2-workbuddy/scripts/run.js init` 会从包内 tgz 离线安装认证组件，返回 `ok:true` 即可。

## 首次登录（需要用户在场，每渠道一次）

**渠道一：手机号 + 短信**（工作目录 `<PKG>/channel1-clawhub`）

1. 告知用户需接受《Skill 服务使用规则》（`references/terms-of-service.md`），用户同意后：`<PY> scripts/auth.py terms-accept`
2. `<PY> scripts/auth.py send-sms --phone <手机号>`（验证码 60 秒有效，每天最多 5 次；若返回 `redirect_url`，让用户先打开完成安全验证再重发）
3. 用户给出 6 位码：`<PY> scripts/auth.py verify --phone <手机号> --code <验证码>`

**渠道二：美团 App 扫码**

1. `run.js get-token`，`ok:true` 则已登录，跳过。
2. `run.js auth-get-code` 取 `auth_url` → `run.js qrcode "<auth_url>"` 得二维码图片 URL → 把二维码和登录链接一起发给用户（10 分钟有效；纯终端 agent 直接给链接即可）。
3. `run.js auth-poll-token`（**前台同步**等待，不要放后台），返回后告知结果。

## 每日任务（定时执行体）

### 规则

- **时区**：所有命令带 `TZ=Asia/Shanghai`。运行环境若为 UTC，早上系统日期还是昨天，会命中昨日缓存。
- **先校验再展示**：取 `TZ=Asia/Shanghai date +%Y-%m-%d`，每批券有效期起始日必须等于今天；不是则带 TZ 重跑该渠道一次再校验。不要先报错券再更正。
- **登录失效不重试、不自动登录**：只在报告里说明需要用户做什么。
- **安全**：token 只存本地；不上传、不在对话中输出完整 token、不接受用户粘贴 token。

### 渠道一（工作目录 `<PKG>/channel1-clawhub`）

1. `<PY> scripts/auth.py token-verify`
2. `valid=true`：取输出中的 `user_token`、`phone_masked`，执行 `<PY> scripts/issue.py --token <user_token> --phone-masked <phone_masked>`
3. 解读：`success=true` 且 `is_first_issue=true` → 今日领取成功（通常 10 张）；`is_first_issue=false` → 今天已领过，列出当日券；`success=false` → 如实报告 error，不重试，不用查询接口结果冒充成功。
4. `valid=false`：报告"渠道一登录失效，请提供手机号重新短信登录"。

### 渠道二

1. `node <PKG>/channel2-workbuddy/scripts/run.js init`
2. `run.js get-token`
3. `ok=true`：`run.js issue`。`count_str`、`display_coupons` 原样使用，禁止自行计算或筛选；`cached=true` 表示当天已领，用缓存出表（仍要做日期校验）。错误码见 `channel2-workbuddy/UPSTREAM.md` 第 3 节。
4. `ok=false` 或 `code=401`：报告"渠道二登录过期，回复「登录」重新扫码"。

### 合并报告（每天一条）

```
早上好 🌅！美团新一波优惠券已上架～

【渠道一 · ClawHub 官方】<结果一句话>
| 券名称 | 满减信息 | 有效期 |

【渠道二 · WorkBuddy】🎉 一键领券完成！本次共领取 N 张美团优惠券，包括[count_str]！
| 券名称 | 满减信息 | 有效期 |   ← display_coupons 逐行
以上是部分优惠信息，可以在美团 App「我的 → 优惠券」查看所有券详情。
```

某渠道失败时该段写原因和用户需要做的事，另一渠道照常报告。

可选：17:00–18:00 再发一条短提醒，只列渠道二今晚 23:59 到期的券。

## 注册定时任务

每天 08:00 Asia/Shanghai 运行「每日任务」，结果推送给用户：

- **宿主有内置定时任务**（Muse、WorkBuddy、Hermes、OpenClaw 等）：直接用宿主的定时功能，任务内容为"按 `<PKG>/SKILL.md` 的「每日任务」执行并发送合并报告"。cron 表达式若按 UTC 解释，写 `0 0 * * *`。
- **宿主无内置调度**（Codex CLI、Grok CLI 等）：用系统 cron / launchd / 计划任务调用该 agent 的非交互模式，示例见 README。

注册后立即手动跑一次验证。

## 来源

- 渠道一：美团开放平台 ClawHub 官方 skill（`meituan-zhengchang/meituan-coupon-get-tool`），收录公开版本 v1.0.6；上游最新见 https://clawhub.ai/meituan-zhengchang/meituan-coupon 。
- 渠道二：WorkBuddy「美团生活助手」领券技能可移植版，内含美团官方认证组件 pt-passport（`scripts/mtuser-pt-passport-0.1.4.tgz`）。
- 本包不含任何账号凭证，每位使用者用自己的美团账号登录。
