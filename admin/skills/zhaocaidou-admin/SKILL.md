---
name: zhaocaidou-admin
description: 招财豆 AI 获客管理员能力 — 派单 / 审单 / 催单 / 看全表 / 调销售名册. 用户说 "派单"/"派给 XX"/"分配给 XX"/"看今日新单"/"AI 获客线索"/"催跟进"/"招财豆怎么样了"/"哪些客户没人跟"/"退回这单"/"标记归档" 时务必触发, 即使用户没显式说 "招财豆". 这把 skill 调 dowsure-hub 的 ai-leads API, 是 hub /tools/event-marketing 操作台的 CLI 等价物.
---

# 招财豆管理员

招财豆 = dowsure 的 AI 获客 agent。每天 08:00 从小蓝本拉新单写入飞书"AI 获客线索池"表, 由你 (管理员) 审一遍 → 派给销售 → 销售在飞书 IM 富卡上点 [已联系/已跟进/不合适] 或自然语言回复 → 招财豆 LLM 抽取 → 回写表 → 你看到推进情况。

这把 skill 让你**在自己的 Claude Code 里直接做派单决策**, 不必每次都去 hub /tools/event-marketing 页面点。

## 什么时候触发我

- "今天 AI 获客有多少新单"、"看今日线索"、"AI 获客这周如何"
- "把 XXX 公司派给 YYY"、"分配 XX 给销售 ZZ"、"派单"
- "标 XX 归档"、"退回 XX"、"XX 这单算了"
- "催一下没动的"、"哪些超 24h 没跟进"
- "改 XX 的跟进状态为已联系"

不该触发我的: hub 自身的代码 / UI / 部署问题。那是 dowsure-hub 仓库的事。

## 装机前置 (一次性)

需要两个 env. 没有就告诉用户没装好, 不要 mock 数据:

```bash
echo $ZHAOCAIDOU_HUB_BASE_URL    # 比如 http://101.33.198.192
echo $ZHAOCAIDOU_WEBHOOK_SECRET   # 64 位 hex 共享 secret
```

两个都为空 → 用户没装好。让他参考根 README, 在 `~/.zshrc` 加 export 行后 `source ~/.zshrc`。

> **绝对不打印 secret 给用户看**。所有 curl 用 `"Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET"` shell 引用形式, 永远不展开。

---

## 核心操作

### 1. 看今日 AI 获客新单 (列表 / 分析)

```bash
curl -fsS "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads" | jq '.leads | length'
```

返回 `{ leads, stats, cache }`。**全表**, 不带 auth, 不带过滤。

字段 (摘): `recordId / company / mobile / mainCategory / mainMarket / priorityLevel / recommendedEntryPoint / recommendedSales / assignmentStatus / assignedTo / followupStatus / gmvBucket / generatedAt`

常见用法:
- 今日新增: `jq '.leads[] | select(.generatedAt > (now*1000 - 86400000))'`
- 待审 (Karen 没派的): `jq '.leads[] | select(.assignmentStatus == "待 Karen 审核")'`
- 重点推荐: `jq '.leads[] | select(.priorityLevel | test("🔥|⭐"))'`
- 全表 stats 直接看响应里的 `.stats`

**强制刷缓存**: `?refresh=1` 加上 (用户问"最新"或"刚才看的不对"时用)。

### 2. 派单 (POST /assign — 这是招财豆 IM 富卡的来源)

```bash
curl -fsS -X POST "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads/assign" \
  -H "Content-Type: application/json" \
  -d '{
    "updates": [
      {
        "recordId": "recXXXXXX",
        "assignmentStatus": "已分配",
        "assignedTo": "<销售中文名>",
        "leadSnapshot": { /* 见下面 */ }
      }
    ]
  }' | jq
```

`assignmentStatus` 白名单: `已分配 / 已归档 / 公海池 / 销售退回`

**派单 (`已分配`) 时必须带 `assignedTo` + 完整 `leadSnapshot`**, 否则推给销售的飞书 IM 卡是空的。`leadSnapshot` 直接从上一步 `/ai-leads` 响应里那条 lead 对象抄字段 (company / mobile / priorityLevel / mainCategory / mainMarket / recommendedEntryPoint / recommendedReason / recommendedSales / legalRep / whyWorthOutreach / city / foundedYears / teamSize / sellerType / devStage / gmvBucket / financingScenario / growthSignalCount / cashPressureSignalCount)。

派单后副作用:
1. 写飞书表 `分配状态=已分配 / 已分配给=<sales> / 分配时间=now`
2. 通过销售名册 (见 references/sales-roster.md) 查 `assignedTo` → `openId`, 推飞书 IM 富信息卡 (header=评级+月销, 含 3 按钮 [已联系/已跟进/不合适])
3. 洗 hub 缓存

**销售名册不在表里**, 在 hub 代码 `src/data/sales-roster.ts`。当前快照见 references/sales-roster.md。`assignedTo` 必须是这里的中文名 (而非 openId), hub 会去查 openId。如果 `assignedTo` 不在名册, 表会写但 IM 不推 (会进 `notifySkipped`)。

**派"已归档 / 公海池 / 销售退回"时**不需要 `assignedTo` 和 `leadSnapshot`, 只 PATCH 状态。

**批量派单**: `updates` 是数组, 一次能派多条。

### 3. 标跟进 (代销售操作 — 销售掉链子时管理员可以代抄)

```bash
curl -fsS -X POST "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads/follow-up" \
  -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "recordId": "recXXX",
    "followupStatus": "已联系",
    "feedbackQuality": "质量好",
    "communicationResult": "线上推进中",
    "nextFollowupDate": "2026-05-30"
  }' | jq
```

字段白名单 (不在白名单的值被 skip 但不报错, 响应有 `skipped[]`):
- `followupStatus`: 未联系 / 已联系 / 已跟进 / 已报价 / 已成单 / 已退回 / 已拒绝
- `feedbackQuality`: 质量好 / 一般 / 不合适
- `communicationResult`: 线上推进中 / 计划线下拜访 / 放弃跟进 / 跟进后拒绝

free text 字段: `unsuitableReason / additionalRequest`  
其它: `dealAmount` (成单金额数字), `nextFollowupDate` (ISO YYYY-MM-DD)

`recordId` 缺时可用 `company` 反查 (按"公司名"精确匹配, 取情报生成时间最新一条)。

### 4. 触发催单 (v0.2: 两种模式, lighthouse cron 已自动跑)

```bash
# 模式 A: 一次催 (未联系 24h+) — 北京 09:00 自动跑
curl -fsS -X POST "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads/nudge" \
  -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" | jq

# 模式 B: 二次催 (已跟进 24h+, v0.2 新) — 北京 09:15 自动跑
curl -fsS -X POST "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads/nudge?type=followup-check" \
  -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" | jq
```

**模式 A (橙色卡)**: 筛 `未联系/null + 24h+`, 推"派给你 24h 了还没动" 给销售。
**模式 B (蓝色卡, v0.2)**: 筛 `已联系/已跟进/已报价 + 反馈 24h+`, 推 3 按钮 `[还在推进中 / 拿到决策时间 / 没下文了]`, 销售点完招财豆按钮路径调 `/follow-up` 写"跟进二次结果"字段。

返回 `{ ok, mode, scanned, candidates, nudged, skipped }`。`mode="followup-check"` 标识走的是二次催 (默认无 mode 字段)。

### 5. v0.2: 看销售反馈原文 (LLM 抽错 catch)

每次销售自然语言反馈, 招财豆 LLM 抽完结构化字段时, 也会把整段原话写到飞书表"销售反馈原文"列。如果 hub 看板上某条 lead 字段看起来怪 (LLM 抽错可能), 可以让 Claude:

```bash
# 拿这条 lead 的 rawFeedback 字段对照看
curl -fsS -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" \
  "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads" \
  | jq '.leads[] | select(.recordId == "recXXX") | {followupStatus, communicationResult, rawFeedback}'
```

对比"销售原话"和"hub 写的字段", 决定要不要让销售重新说一遍 (你帮他改) 或直接调 `/follow-up` 修正字段。

---

## 常见工作流

### 工作流 A: "把今天的新单都过一遍, 该派的派"

```
1. GET /ai-leads — 取全表
2. jq 过滤: 今日生成 + 未派单 (assignmentStatus="待 Karen 审核")
3. 对每条 lead, 看 AI 推荐销售 (recommendedSales) 和评级 (priorityLevel)
4. 跟用户确认每条派给谁 (或批量按 recommendedSales 派)
5. POST /assign 批量 updates
6. 汇报: X 条已派 / Y 条标归档 / Z 条留待审核
```

### 工作流 B: "今天哪些超 24h 没动?"

```
1. GET /ai-leads
2. jq 过滤: assignmentStatus="已分配" + followupStatus∈{null, "未联系"}
3. 算每条的派单年龄 (now - 分配时间)
4. 列出 >24h 的, 按销售分组报给用户
5. 用户决定是 POST /nudge 全量催, 还是手动催某几条
```

### 工作流 C: "XX 公司销售退回了, 重新派给 YY"

```
1. 先 POST /assign 把 XX 标 "销售退回" (清除原 assignedTo)
2. 再 POST /assign 用相同 recordId 标 "已分配" + assignedTo="YY" + leadSnapshot
   (会重新推一张 IM 卡给 YY)
```

---

## 边界 / 安全

| 这把 skill 能干 | 这把 skill 不该干 |
|---|---|
| 读 /ai-leads 全表 | 改 dowsure-hub 代码 |
| POST /assign 任意销售 | 调 hub 其他无关 endpoint (write-leads / approval/*) |
| POST /follow-up 任意 record | SSH 到招财豆妙搭机 |
| POST /nudge | 改 sales-roster.ts (那是 hub 仓的事) |
| 看 stats / cache 状态 | 直接读写飞书表 (绕过 hub) |

**如果用户问的事超出范围** (比如"招财豆挂了怎么修"、"hub 上线"、"加新销售到名册"), 老实说: "这把 skill 不管这个, 请走 dowsure-hub 仓或妙搭运维"。

**销售名册的当前快照**: 见 `references/sales-roster.md`。如果用户要派给的销售不在快照里, 提示用户先在 `dowsure-hub/src/data/sales-roster.ts` 加上, 然后上线 hub。skill 这边的快照可以一起 PR 更新。

---

## 进阶参考

- `references/sales-roster.md` — 销售名册快照 (含 openId)
- `references/endpoint-schema.md` — 全部 endpoint 的 body/response schema (用作 jq filter 参考)
- `references/troubleshooting.md` — 常见错误 + lighthouse 端 log 查法

## 为什么这样设计

- **薄壳**: 这把 skill 只负责"包装 curl + 让 Claude 理解 schema", 业务逻辑全在 hub。所以 hub 升级 (加字段/换 schema) 后这把 skill 也要跟着改。
- **不存 secret**: secret 走 env, 不进 git/对话/SKILL.md。
- **不本地缓存**: 每次都打 hub, 用 hub 的 5min 缓存层。这样你和别的销售看到的数据是同一个。
- **不绕过 hub**: 直接调飞书 OpenAPI 也能写, 但会绕开 hub 的 revalidateTag, 导致 hub UI 缓存陈旧。**永远走 hub**。
