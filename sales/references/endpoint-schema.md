# Endpoint Schema (sales 视角)

销售只能调 2 个 endpoint。其它 (派单 / 催单 / 写表 / 立项) 一律拒绝, 即使用户问也不调。

---

## GET /api/event-marketing/ai-leads

**Auth**: 无 (V1)  
**Query**: `?refresh=1` 可强制刷缓存, 但销售场景一般不用 — hub 5min 自动刷, 没必要每次都强制

**Response**:
```json
{
  "leads": [AiHuokeLead],
  "stats": { /* 不用看, 那是管理员视角全表 stats */ },
  "cache": { "state": "fresh|refresh|stale", "updatedAt": "...", "ttlSeconds": 300 }
}
```

> ⚠️ 这个 endpoint **返回全表**, 销售 skill 必须客户端 jq 过滤 `assignedTo == $ZHAOCAIDOU_SALES_NAME`, **不能把全表丢给用户**。

**AiHuokeLead 字段** (销售关心的):
```
recordId         (必)  — 写 /follow-up 用
company          (必)  — 公司名
mobile                 — 客户手机号
mainCategory           — 主营品类
mainMarket             — 主要市场
priorityLevel          — 🔥 重点推荐 / ⭐ 优质美国站 / ⚡ 正常触达 / ⚙️ 待补全 / 💤 待验证
recommendedEntryPoint  — 切入点 (给销售看的一句话)
recommendedReason      — 为什么派给你 (招财豆解释)
gmvBucket              — 月 GMV 段位
legalRep               — 法定代表人
city                   — 城市
assignmentStatus       — 应该都是 "已分配" (能看到的)
assignedTo             — 应该 = 当前销售 (过滤后)
followupStatus         — 当前跟进状态 (你已经标过的)
generatedAt            — 派单时间 ms 时间戳
```

---

## POST /api/event-marketing/ai-leads/follow-up

**Auth**: `Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET`

**Body** (`recordId` 必填; 其它选填):
```json
{
  "recordId": "recXXX",
  "followupStatus": "未联系|已联系|已跟进|已报价|已成单|已退回|已拒绝",
  "feedbackQuality": "质量好|一般|不合适",
  "communicationResult": "线上推进中|计划线下拜访|放弃跟进|跟进后拒绝",
  "unsuitableReason": "free text",
  "additionalRequest": "free text",
  "dealAmount": 12345.67,
  "nextFollowupDate": "2026-05-30",
  "rawFeedback": "销售跟我说的原始口语, LLM 抽前的整段 (强烈建议带, 抽错时管理员能看上下文)",
  "followupNudgeResult": "还在推进中|拿到决策时间|没下文了"
}
```

**字段写入语义**:
- 不传的字段飞书侧不动 (PATCH 语义)
- 不在白名单的值 hub 会 skip 不报错, 但 skip 信息在 `response.skipped[]` 里, **必须报给用户看**
- `nextFollowupDate` 必须 ISO `YYYY-MM-DD`, "下周一" 这种 LLM 算了再传

**Response**:
```json
{
  "ok": true,
  "recordId": "recXXX",
  "updated": { "跟进状态": "已联系", "最近反馈时间": 1716624000000 },
  "skipped": ["followupStatus=\"嗯\" 不在白名单"]
}
```

`updated` 字段是飞书侧实际写的 (中文 key)。给用户回执时可以说 "已写: 跟进状态=已联系", 不必报"最近反馈时间" — 那是 hub 内部用的标记。

**销售特有的身份护栏**:

调 /follow-up 前必须验证:
1. `recordId` 来自上一步 GET + 客户端过滤的结果 (不能直接吃用户给的陌生 recordId)
2. 该 lead 的 `assignedTo == $ZHAOCAIDOU_SALES_NAME`

否则拒绝并提示"那不是分配给你的"。这一层 hub 端 V1 不校验, 全靠 skill 端约束。

---

## 销售绝对不能调的 endpoint (即使用户要求)

| Endpoint | 谁能调 | 用户问起怎么回 |
|---|---|---|
| POST /ai-leads/assign | 管理员 skill | "派单是管理员的事, 让 Leo 来" |
| POST /ai-leads/nudge | 管理员 skill / cron | "手动催单是管理员的事" |
| GET /ai-leads 但不过滤 | 没人 | "你只能看分配给你的, 看全表得管理员" |
| 任何 /event-marketing/* 其他路径 | 不是这把 skill 的范围 | "这把 skill 不管那个" |
