# zhaocaidou-admin — 管理员 plugin

招财豆 AI 获客系统的**运营管理员**视角技能包。装这把的人能动整个线索池, 包括派单给任何销售、催跟进、看全表 stats。

## 谁应该装

- 运营经理 / 项目 owner (Leo 本人就是这把)
- 想 review 招财豆派单效果的人
- 想替销售代填跟进状态的人

**不该装这把的人**: 一线销售。销售装 `zhaocaidou-sales` 那把。

## 视角

> "我是运营, 我要管 AI 获客整条线: 看池子状态、决定派给谁、催那些拖延的、复盘成单率"

这把 skill 让你**绕过 hub 网页, 用对话操作**。等价于 hub `/tools/event-marketing` 操作台的 CLI 版本, 但你可以一句话批量做事 (派 10 条 ⭐ 的给 ZZ), 网页上点 10 次。

## 能做的 4 件事

### 1. 看全表 + stats

> "今天 AI 获客有多少新单"
> "看待 Karen 审核的"
> "🔥 重点推荐的列出来"

返回**全表** (含所有销售名下的 lead), 包括今日新增 / 待审 / 已分配 / 评级分布的 stats。

### 2. 派单 (含 IM 卡)

> "把杭州越禾派给 Leo(自测)"
> "把今天 ⭐ 的 5 条都派给张三"
> "标 recXXX 归档"

派单后:
- 飞书表写 `分配状态=已分配 / 已分配给=<sales> / 分配时间=now`
- 招财豆推飞书 IM 富信息卡 (评级/月销/法人/手机/信号/切入点) + 3 按钮 [已联系/已跟进/不合适] 给销售
- hub 缓存洗掉

### 3. 标跟进 (代销售操作)

> "把杭州越禾的状态标成已报价"
> "标 recXXX 已成单, 金额 5 万"

跟销售自己用 sales skill 标的效果一样, 只是管理员能改任意 record (销售只能改自己的)。

### 4. 催跟进

> "手动跑一次催单"

触发 `/nudge`, 扫全表找:
- 已分配 + 未跟进 + 派单 > 24h + 上次催 > 20h 的 leads
- 给每条对应的销售推飞书 IM 橙色提醒卡

平时这事是腾讯云 cron 每天 09:00 自动跑, 手动触发用于"派单后想立刻催一波"或调试。

## 不能做的事 (硬边界)

- ❌ 改 hub 代码 / 改 sales-roster — 那是 dowsure-hub 仓库的事
- ❌ SSH 招财豆运行时 — 那是妙搭运维的事
- ❌ 直接读写飞书表 (绕过 hub) — 会让 hub UI 缓存陈旧
- ❌ 调 hub 其他无关 endpoint (write-leads / approval/*)

遇到用户问超出范围, skill 会老实说 "这把 skill 不管这个"。

## 装机

见根 [../README.md](../README.md) 装机段。3 步:

1. `~/.zshrc` 加 2 个 env (`ZHAOCAIDOU_HUB_BASE_URL` + `ZHAOCAIDOU_WEBHOOK_SECRET`)
2. `/plugin marketplace add wscx8282888-pixel/zhaocaidou-skills`
3. `/plugin install zhaocaidou-admin@zhaocaidou-skills`

重启 Claude Code 让 env 生效。

## 典型对话流

**Leo**: 今天 AI 获客咋样

**Claude (触发 zhaocaidou-admin)**:  
今日新增 13 条, 8 条待 Karen 审核, 5 条已派给 Leo(自测)。重点推荐 3 条:
| 公司 | 评级 | 月销 | 切入点 |
|---|---|---|---|
| 杭州越禾 | 🔥 重点推荐 | $2M-$5M | 类目扩张缺备货, 可推融资 |
| 上海艾博 | ⭐ 优质美国站 | $500k-$2M | 旺季备货可贷 |
| ... | ... | ... | ... |

**Leo**: 把 🔥 那条派给 Leo(自测), ⭐ 两条都派给张三

**Claude**: 派给 Leo(自测): 杭州越禾 (recXXX). 派给张三: 上海艾博, 兴时达。3 张 IM 卡已推飞书。1 分钟内对方能看到。

## 参考文档

- [skills/zhaocaidou-admin/SKILL.md](skills/zhaocaidou-admin/SKILL.md) — 主 skill (Claude 实际读这个)
- [references/sales-roster.md](references/sales-roster.md) — 销售名册快照 (派单 `assignedTo` 要用这里的中文名)
- [references/endpoint-schema.md](references/endpoint-schema.md) — 4 个 endpoint 完整 body/response
- [references/troubleshooting.md](references/troubleshooting.md) — 8 种常见错误排查
