# zhaocaidou-sales — 销售 plugin

招财豆 AI 获客系统的**一线销售**视角技能包。装这把的销售能在 Claude Code 里看自己手上的线索、用自然语言更新跟进状态。

## 谁应该装

- 一线 BD / 销售 (招财豆给你派过单, 或将来会派)
- 想用一句话替代飞书表填字段的销售

**不该装这把的人**: 运营管理员 (那把是 `zhaocaidou-admin`, 视角不同, 但管理员也可以两把都装体验销售视角)。

## 视角

> "招财豆每天给我推几条客户线索, 我跟进, 我标进度。我只关心我手上的, 别人的不是我的事"

这把 skill 让你**不用打开飞书 app 也不用记字段名**。你用大白话告诉 Claude 跟客户聊了啥, Claude 帮你转成飞书表能接受的结构化数据, 写完飞书表里那一行就更新了。

## 能做的 3 件事

### 1. 看我的线索 (永远只看自己的, 不看全表)

> "我手上还有几单没跟"
> "看我的"
> "我的线索按评级排"

返回**只过滤了你身份的线索** — `assignedTo == $ZHAOCAIDOU_SALES_NAME`。看不到别人名下的。

### 2. 标跟进 (按钮式快速)

> "标杭州越禾已联系"
> "标 XX 已加微信"
> "深圳那家是无意向, 他做 Temu"

直接写飞书表对应字段。状态白名单: 未联系 / 已联系 / 已跟进 / 已加微信 / 已绑店 / 已成单 / 无意向。

身份护栏: 调写之前 skill 会先验你给的公司是不是真的在你名下 (从 GET /ai-leads 过滤后的结果里找), 防止越权改别人的单。

### 3. 自然语言反馈 (核心场景, 最常用)

这才是这把 skill 的重头。销售跟你 (Claude) 说大白话:

> "我加上越禾张总微信了, 他说下周一聊一下补货融资"

skill 抽出:
- 跟进状态: `已加微信`
- 沟通结果: `线上推进中`
- 下次跟进日期: `2026-06-02` (Claude 算出"下周一"具体哪天)
- 补充信息: `客户想了解补货融资`

**关键设计**: skill **必须先把抽到的字段给你看草稿**, 让你确认 "对吗?" → 你说"嗯"才 POST。LLM 抽错的成本 (去飞书手工改) 比销售点一下"嗯"高 10 倍, 这一秒不能省。

抽取规则 + 8 个真实场景示例见 [references/llm-extraction.md](references/llm-extraction.md)。

## 不能做的事 (硬边界)

- ❌ 看全表 / 看别人名下的 — `assignedTo != $ZHAOCAIDOU_SALES_NAME` 永远过滤掉
- ❌ 改别人的单 (即使用户给 recordId, skill 也会拒)
- ❌ 派单 — 那是管理员的事 (`zhaocaidou-admin`)
- ❌ 触发催跟进 — 那是管理员/cron 的事
- ❌ 跳过草稿确认直接 POST — 永远先草稿
- ❌ 直接读写飞书表 (绕过 hub)

遇到用户问超出范围 ("派一条给我"、"看 XX 的单"), skill 会回 "这把 skill 只管你自己的, 找 Leo 用管理员 skill"。

## 装机

见根 [../README.md](../README.md) 装机段。**销售比管理员多一个 env**:

1. `~/.zshrc` 加 3 个 env:
   ```bash
   export ZHAOCAIDOU_HUB_BASE_URL="http://101.33.198.192"
   export ZHAOCAIDOU_WEBHOOK_SECRET="<问 Leo 拿>"
   export ZHAOCAIDOU_SALES_NAME="张三"   # ★ 必须跟销售名册一字不差
   ```
2. `/plugin marketplace add wscx8282888-pixel/zhaocaidou-skills`
3. `/plugin install zhaocaidou-sales@zhaocaidou-skills`

**`ZHAOCAIDOU_SALES_NAME` 必须跟 hub `src/data/sales-roster.ts` 里的 `name` 字段完全一致** — 带括号空格的都不能差。不知道自己叫啥就找 Leo 要名册行。

重启 Claude Code 让 env 生效。

## 典型对话流

**张三**: 我手上还几单没动?

**Claude (触发 zhaocaidou-sales)**:  
你手上还有 3 条没标跟进:
1. 杭州越禾 (🔥 重点推荐, 派给你 2 天了)
2. 上海艾博 (⭐ 优质美国站, 派给你 1 天)
3. 兴时达 (⚡ 正常触达, 今天派的)

要哪条先动?

**张三**: 越禾今天加上微信了, 客户说下周一聊补货融资的事

**Claude**:  
我准备这样写:
- 公司: **杭州越禾** (recXXX)
- 跟进状态: **已加微信**
- 沟通结果: **线上推进中**
- 下次跟进日期: **2026-06-02** (下周一)
- 补充信息: **客户希望了解补货融资**

对吗?

**张三**: 对

**Claude**: 写完了。飞书表那一行已经更新, 招财豆 5 分钟内会反映在管理员看板上。

## 参考文档

- [skills/zhaocaidou-sales/SKILL.md](skills/zhaocaidou-sales/SKILL.md) — 主 skill (Claude 实际读这个)
- [references/endpoint-schema.md](references/endpoint-schema.md) — 销售能调的 2 个 endpoint 完整 schema + 不能调的 endpoint 列表
- [references/llm-extraction.md](references/llm-extraction.md) — 销售口语 → 字段 8 个示例 (加微信 / 没人接 / 不合适 / 约线下 / 要资料 / 成单 / 复杂场景 / 模糊语气追问)
- [references/troubleshooting.md](references/troubleshooting.md) — 销售特有 8 种问题 (我的是空 / 401 / 写后表没变 / LLM 抽错 等)
