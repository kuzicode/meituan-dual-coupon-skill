# 美团双渠道每日领券

让任意 AI agent 每天自动领取两个渠道的美团外卖/生活优惠券，合并成一条报告发给你。

| 渠道 | 来源 | 登录方式 | 依赖 | 每日券量 |
| --- | --- | --- | --- | --- |
| 一 | 美团开放平台 ClawHub 官方 skill | 手机号 + 短信 | Python 3 + httpx | 约 10 张 |
| 二 | WorkBuddy 美团生活助手领券技能 | 美团 App 扫码 | Node ≥ 18 + Python 3 | 约 27–38 张 |

## 快速开始

1. 克隆/下载本仓库到本地任意位置。
2. 把下面这句话发给你的 agent（Codex / Hermes / Grok / Muse / WorkBuddy / Claude Code …）：

   > 阅读本仓库的 `SKILL.md`，帮我完成环境准备和两个渠道的登录，然后注册每天 08:00（北京时间）的美团领券定时任务。

3. 按 agent 提示输入手机号、短信验证码，并用美团 App 扫码。之后每天自动执行。

## 定时任务怎么挂

| agent 类型 | 做法 |
| --- | --- |
| 有内置定时任务（Muse、WorkBuddy、Hermes、OpenClaw 等） | 让 agent 用自带定时功能注册，执行体 = `SKILL.md`「每日任务」 |
| 无内置调度（Codex CLI、Grok CLI 等） | 用系统 cron / launchd 调 agent 的非交互模式，见下例 |

```cron
# crontab -e（机器时区非北京时间时用 CRON_TZ 或自行换算）
CRON_TZ=Asia/Shanghai
0 8 * * * cd /path/to/meituan-dual-coupon-skill && codex exec "按 SKILL.md 的「每日任务」执行，输出合并报告" >> ~/meituan-coupon.log 2>&1
```

无内置调度时报告只会写进日志；需要推送请让 agent 接入你自己的通知渠道（邮件 / IM 机器人等）。

## 注意

- 登录凭证保存在 `channel*/credentials/`，已被 `.gitignore` 排除，**不要提交或外传**。
- 所有命令必须带 `TZ=Asia/Shanghai`，否则 UTC 环境早上会拿到昨天的券。
- 登录过期时 agent 不会自动重登，会在报告里提示你重新短信/扫码。
- 仅供个人自用；渠道一使用前需同意美团《Skill 服务使用规则》。
