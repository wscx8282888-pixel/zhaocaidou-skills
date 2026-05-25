# Endpoint Schema (admin 用得到的)

完整定义见 hub repo `src/app/api/event-marketing/ai-leads/`。本文件是 skill 自包含速查。

---

## GET /api/event-marketing/ai-leads

**Auth**: 无  
**Query**: `?refresh=1` (强制刷缓存)

**Response**:
```json
{
  "leads": [AiHuokeLead],
  "stats": { /* 今日新增 / 待审 / 已分配 / 评级分布 */ },
  "cache": { "state": "fresh|refresh|stale", "updatedAt": "2026-05-25T...", "ttlSeconds": 300 }
}
```

**AiHuokeLead 关键字段**:
```
recordId, company, mobile, companyPhone, mainCategory, mainMarket,
priorityLevel ("🔥 重点推荐" | "⭐ 优质美国站" | "⚡ 正常触达" | "⚙️ 待补全信息" | "💤 待验证标签"),
recommendedEntryPoint, recommendedSales, recommendedReason, alternativeSales,
assignmentStatus ("待 Karen 审核" | "已分配" | "销售退回" | "公海池" | "已归档"),
assignedTo, followupStatus, gmvBucket, asinCount, generatedAt (ms),
legalRep, whyWorthOutreach, city, foundedYears, teamSize, sellerType,
devStage, financingScenario, estimatedCreditLimit, growthSignalCount, cashPressureSignalCount
```

---

## POST /api/event-marketing/ai-leads/assign

**Auth**: 无 (V1; hub 内部用 agent token 写飞书)

**Body**:
```json
{
  "updates": [
    {
      "recordId": "recXXX",
      "assignmentStatus": "已分配" | "已归档" | "公海池" | "销售退回",
      "assignedTo": "<中文名, 仅 已分配 时填>",
      "leadSnapshot": { /* 复制对应 lead 的字段, 给 IM 卡渲染用 */ }
    }
  ]
}
```

**leadSnapshot 字段** (全可选, 给越多 IM 卡越丰富):
```
company, mobile, priorityLevel, mainCategory, mainMarket,
recommendedEntryPoint, recommendedReason, recommendedSales,
legalRep, whyWorthOutreach, city, foundedYears, teamSize,
sellerType, devStage, gmvBucket, financingScenario,
growthSignalCount, cashPressureSignalCount
```

**Response**:
```json
{
  "ok": true,
  "updated": 3,         // 写表成功条数
  "failed": 0,
  "notified": 2,        // IM 卡推送成功条数
  "notifySkipped": 1,   // assignedTo 不在名册等原因
  "details": [...]      // 仅有失败时返回
}
```

---

## POST /api/event-marketing/ai-leads/follow-up

**Auth**: `Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET`

**Body** (recordId 和 company 至少给一个):
```json
{
  "recordId": "recXXX",
  "company": "亚马逊 (用于反查, recordId 缺时)",
  "followupStatus": "未联系|已联系|已跟进|已报价|已成单|已退回|已拒绝",
  "feedbackQuality": "质量好|一般|不合适",
  "communicationResult": "线上推进中|计划线下拜访|放弃跟进|跟进后拒绝",
  "unsuitableReason": "free text",
  "additionalRequest": "free text",
  "dealAmount": 12345.67,
  "nextFollowupDate": "2026-05-30"
}
```

不在白名单的值会被 skip 写进响应 `skipped[]`, 不报错。

**Response**:
```json
{
  "ok": true,
  "recordId": "recXXX",
  "updated": { /* 实际写飞书的字段 */ },
  "skipped": ["followupStatus=\"xxx\" 不在白名单"]
}
```

副作用: 写飞书的同时把"最近反馈时间"=now 标记一下 (LLM 抽过这次)。

---

## POST /api/event-marketing/ai-leads/nudge

**Auth**: `Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET`  
**Body**: 无

**Response**:
```json
{
  "ok": true,
  "scanned": 932,
  "candidates": 12,
  "nudged": 7,
  "skipped": [
    {"recordId": "recXXX", "reason": "状态非已分配"},
    {"recordId": "recYYY", "reason": "派单<24h"},
    {"recordId": "recZZZ", "reason": "20h 内已催过"}
  ]
}
```

筛选规则:
1. `assignmentStatus="已分配"`
2. `followupStatus` ∈ {null, "未联系"}
3. `assignedTo` 在 sales-roster
4. `now - 分配时间 >= 24h`
5. `now - 水信最近推送时间 >= 20h`

被催 → 推飞书 IM 橙色 header 卡 + 写"水信最近推送时间 + 推送次数+1"。
