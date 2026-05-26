---
name: zhaocaidou-sales
description: 招财豆 AI 获客销售助手 — 看分配给我的线索 / 标跟进状态 / 自然语言写反馈自动回写飞书表. 用户说 "我手上有哪些线索"/"我的单"/"看我的"/"标已联系"/"我跟 XX 聊了"/"加上微信了"/"对方说..."/"这单不合适"/"明天约线下"/"下周二再聊" 时务必触发, 即使没显式说 "招财豆" 或 "线索". 这把 skill 只动当前销售自己的单, 看不到也派不动别人的.
---

# 招财豆销售

招财豆是 dowsure AI 获客 agent, 每天自动派线索给你, 通常你在飞书 IM 上点 [已联系/已跟进/不合适] 按钮回应。

这把 skill 让你**在自己的 Claude Code 里直接处理线索**, 用自然语言告诉 Claude 你跟客户聊了啥, Claude 帮你提取信息然后写回飞书表 — 不用打开飞书 app, 也不用记字段叫啥。

**关键边界**: 这把 skill 只动**分配给你**的线索。看不到别人的, 派不了单, 触发不了催跟进。那些是管理员的事。

## 什么时候触发我

- "我手上有多少单还没跟"、"我的线索"、"看我的"
- "我跟 XX 公司聊了, 他说..."、"加上微信了"、"打了没人接"
- "标 XX 已联系"、"XX 已报价"
- "这单不合适, 对方做 Temu"、"已拒绝"
- "下周二再聊"、"约了线下"

不该触发我的: 派单 / 看全表 / 改别人的单 — 这些是管理员的事。需要让用户找 Leo。

## 装机前置 (一次性)

```bash
echo $ZHAOCAIDOU_HUB_BASE_URL      # 比如 http://101.33.198.192
echo $ZHAOCAIDOU_WEBHOOK_SECRET     # 64 位 hex, 跟 Leo 拿
echo $ZHAOCAIDOU_SALES_NAME         # 你在销售名册里的中文名, 比如 "张三" 或 "Leo(自测)"
```

任意一个为空 → 装机没走完。让用户参考根 README 在 `~/.zshrc` 加 export, `source ~/.zshrc` 后重启 Claude Code。

> `$ZHAOCAIDOU_SALES_NAME` **必须跟 hub `src/data/sales-roster.ts` 里的 `name` 字段一字不差**。带括号空格的也要一致 (比如 `"Leo(自测)"` 不能写成 `"Leo 自测"`)。差一个字过滤就拿不到线索。
>
> 不知道自己叫啥 → 让用户找 Leo 要名册行。

> `$ZHAOCAIDOU_WEBHOOK_SECRET` **绝对不打印不展示**, 所有 curl 用 shell 变量 `"Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET"` 引用。

---

## 核心操作

### 1. 看分配给我的线索

```bash
curl -fsS "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads" \
  | jq --arg me "$ZHAOCAIDOU_SALES_NAME" '
      .leads
      | map(select(.assignedTo == $me))
      | sort_by(.generatedAt) | reverse
    '
```

返回这名销售名下的所有 lead。**这把 skill 必须永远带 `assignedTo == $ZHAOCAIDOU_SALES_NAME` 过滤**, 不能把全表丢给用户 — 那是数据越权。

常见子集 (在过滤后基础上加):
- 没动过 (新派给我的): `select(.followupStatus == null or .followupStatus == "未联系")`
- 推进中: `select(.followupStatus | IN("已联系", "已跟进", "已报价"))`
- 已结束 (终态): `select(.followupStatus | IN("已成单", "已退回", "已拒绝"))`
- 高优先级先看: `sort_by(.priorityLevel | test("🔥|⭐") | not) | sort_by(.generatedAt) | reverse`
- 超 24h 没标进展的 (要被催的): `select((now*1000 - .generatedAt) > 86400000 and (.followupStatus == null or .followupStatus == "未联系"))`

每条 lead 用户关心的字段: `company / mobile / priorityLevel / recommendedEntryPoint / mainCategory / mainMarket / gmvBucket / followupStatus / generatedAt`

给用户展示时把 `generatedAt` 转人话 ("3 天前派给你"), 不要 raw ms。

> ⚠️ hub 当前没有按销售过滤的 endpoint, 我们走"客户端过滤"路。如果某天 hub 加了 `/api/event-marketing/ai-leads/my?sales=...`, 优先用那个 (减传输量)。看 references/endpoint-schema.md 检查最新状态。

### 2. 标跟进 (按钮式 — 用户明确说啥状态)

```bash
curl -fsS -X POST "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads/follow-up" \
  -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "recordId": "recXXX",
    "followupStatus": "已联系"
  }' | jq
```

字段白名单 (跟飞书表 schema 对齐, 不在白名单的值 hub 会 skip):
- `followupStatus`: `未联系` / `已联系` / `已跟进` / `已报价` / `已成单` / `已退回` / `已拒绝`
- `feedbackQuality`: `质量好` / `一般` / `不合适`
- `communicationResult`: `线上推进中` / `计划线下拜访` / `放弃跟进` / `跟进后拒绝`
- `unsuitableReason`: free text
- `additionalRequest`: free text
- `dealAmount`: number (成单后)
- `nextFollowupDate`: ISO `YYYY-MM-DD`

**v0.3 业务联动**: 销售写 `followupStatus=已拒绝` (客户不合适) 或 `已退回` (自己跟不动) → hub **自动**把这条单入公海池, 管理员看板能看到, 等他重派或归档。销售不用再做任何事。
- **`rawFeedback`** (v0.2 推荐带): 销售原始口语, LLM 抽前的整段。这字段帮管理员
  在 LLM 抽错时能看到原始上下文, 是保险绳。除非销售只是说 "标已联系" 这种
  按钮式短指令, 否则**永远带这个字段**, 把销售说的原话填进去。
- **`followupNudgeResult`** (v0.2): 销售在二次催卡上点 3 选项之一时用 (招财豆按钮路径
  自动调, sales skill 一般不直接用这字段)

**用户没说的字段一律不传** (而不是猜)。空写比错写好 — 错写会污染数据。

**身份护栏**: POST follow-up 前要先确认 `recordId` 对应的 lead 的 `assignedTo == $ZHAOCAIDOU_SALES_NAME` (从上一步 jq 过滤过的结果里挑)。**绝不能允许用户用 recordId 改别人的单**。如果用户提了一个不在他名下的 recordId, 拒绝并提示"那不是分配给你的"。

### 3. 自然语言反馈 (核心场景 — 主要用法)

销售更常说的是 "我跟张总聊了, 他说下周线下见", 不会说 "把跟进状态设成已跟进, 沟通结果设成计划线下拜访"。这把 skill 的灵魂是**让 Claude 把口语转成结构化字段, 用户确认后再回写**。

工作流:

```
1. 销售口语描述 (一两句话, 含公司名 + 沟通情况)
   ↓
2. 你 (Claude) 先确认是哪条 lead:
   - 用户没给公司名 → 让他说
   - 给了公司名 → 用 GET /ai-leads 拿"我的"过滤后, 找 .company == "<那家>" 的 recordId
   - 找不到 → "这家不在你名下, 你确认下名字, 或者让 Leo 检查派单了没"
   - 多家 fuzzy match → 列出来让用户挑
   ↓
3. 按 references/llm-extraction.md 的规则, 你 (Claude) 自己抽出字段
   (followupStatus / feedbackQuality / communicationResult / nextFollowupDate / unsuitableReason / additionalRequest)
   ↓
4. ★ 给销售看草稿, 让他确认 ★
   "我准备这样写:
     公司: XX
     跟进状态: 已跟进
     沟通结果: 线上推进中
     下次跟进日期: 2026-06-02
     补充信息: 客户希望了解备货融资具体额度
   对吗?"
   ↓
5. 用户确认 → POST /follow-up
   用户改 → 按改后的字段重新构造 → 再确认 → POST
```

**永远不要跳过第 4 步**。LLM 抽取偶尔会错 (尤其日期 / 状态边界), 销售一句"嗯"花 1 秒, 错写要去飞书手工改要 3 分钟。

抽取规则 + 8 个示例见 [references/llm-extraction.md](../../references/llm-extraction.md)。

---

## 常见工作流

### 工作流 A: "我手上还有几单没动的"

```
1. GET /ai-leads → jq 过滤 assignedTo == 我
2. 进一步过滤 followupStatus ∈ {null, "未联系"}
3. 按派单时间排序 (新的在上)
4. 列给用户: 公司名 / 评级 / 切入点 / 派给我多久了
5. (可选) 用户点哪条 → 进入工作流 C (自然语言写反馈)
```

### 工作流 B: "标 XX 公司已联系" (按钮式快速)

```
1. GET /ai-leads → 过滤 assignedTo == 我 → 找 .company 匹配
2. 找到 1 条 → POST /follow-up { recordId, followupStatus: "已联系" }
3. 找到 0 条 → "XX 不在你名下, 你确认下"
4. 找到多条 → 列给用户挑
```

### 工作流 C: "我跟 XX 聊了..." (自然语言, 主要用法)

按上面第 3 节"自然语言反馈"5 步走。

### 工作流 D: "我成单了" (打款金额)

```
1. 确认是哪家 + 在你名下
2. 抽出 dealAmount (从用户说的 "签了 5 万" / "首单 ¥120k"), 必须是数字
3. 默认 followupStatus="已成单", communicationResult 留空 (成单已经是终态)
4. 给用户看草稿 (强调金额单位), 确认
5. POST
```

---

## 边界 / 安全

| 这把 skill 能干 | 这把 skill 绝对不该干 |
|---|---|
| 看分配给我的线索 (`assignedTo == $ZHAOCAIDOU_SALES_NAME`) | 看全表 / 看别人的 |
| 标自己的单的跟进状态 | 改别人的单 (即使用户给 recordId) |
| 自然语言抽取后回写自己的单 | 派单 (`/assign`) — 那是管理员 |
| 给用户看草稿等他确认 | 跳过确认直接 POST |
| | 触发催跟进 (`/nudge`) — 管理员/cron 独占 |
| | 直接读写飞书表 (绕过 hub) |
| | 调 hub 其他无关 endpoint |

**遇到用户问超出范围的事** (派单 / 加销售 / 看全表 / hub 部署), 直接说: "这把 skill 只管你自己的线索, 你要找 Leo 用管理员 skill 处理"。

**身份混乱时** (用户说"我是张三, 帮我看"但 `$ZHAOCAIDOU_SALES_NAME=李四`): 不要用对话里的"我是 X"覆盖 env 身份。env 是装机硬绑定的, 对话身份不可信。提示用户改 env 然后 source 重启 Claude。

---

## 进阶参考

- [references/endpoint-schema.md](../../references/endpoint-schema.md) — 销售能调的 2 个 endpoint 完整 body/response
- [references/llm-extraction.md](../../references/llm-extraction.md) — 销售口语 → 结构化字段的 8 个示例 + 抽取规则
- [references/troubleshooting.md](../../references/troubleshooting.md) — env 没配 / Bearer 401 / "我的"是空 / 写后表里没变 等

## 为什么这样设计

- **薄壳**: skill 不存任何业务状态, 每次都打 hub。这样 hub 5min 缓存内你看到的跟 IM 卡里的一致。
- **客户端过滤而不是新 endpoint**: V1 hub 没按销售过滤的 GET, 先用客户端 jq 过滤跑通流程。流量小 (一天几百条), 不优化。
- **必须确认再写**: LLM 偶尔抽错, 销售一句"嗯"花 1 秒就能 catch。错写飞书事后手改非常贵。
- **env 锁身份**: 销售名册的中文名是 hub 强 contract, 这把 skill 强约束 env 必须一字不差。这样不会发生"客户 A 改成客户 B 单"的越权。
- **不重复造轮子**: 不本地缓存名册、不本地维护字段白名单 — 全部以 hub 为准。hub 升级 (加字段/换枚举) 时这把 skill 跟着改 references/, 装的销售拉一下就同步。
