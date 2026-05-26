# 招财豆 (OpenClaw) 运行时待打的补丁

> 这些补丁要在**妙搭 Web 终端**改 (`/home/gem/workspace/agent/` 下的文件), 不在 hub repo 里。
> 进入: https://miaoda.feishu.cn/app/app_4jyb648smdxwm/ → 找 Terminal → 操作。
> 改完执行 `sh /home/gem/workspace/agent/scripts/restart.sh`。

## v0.3 补丁清单 (2026-05-25)

### 补丁 1: `unsuitable` 按钮字段映射 + 用 hub updatedCard 替换原卡

**问题**: 销售点"❌ 不合适"按钮后, 招财豆把 followupStatus 写成废枚举 (`无意向` 或类似), hub /follow-up 白名单 skip 了它, 跟进状态没登记, 也没触发入公海池。

**文件**: `/home/gem/workspace/agent/event-handlers.js`

**找**: `handleHuokeQuickAction` 函数。它处理飞书 `card.action.trigger` 事件, 收到 `value: { kind: "ai-huoke-quick", recordId, action }` 后调 hub /follow-up。

**改 — action 到 body 的映射** (现有可能是):
```js
// 老的 (大概):
const body = {
  recordId,
  followupStatus:
    action === "contacted" ? "已联系" :
    action === "followed-up" ? "已跟进" :
    action === "unsuitable" ? "无意向" :  // ← 废枚举, 飞书表 schema 没这值
    null,
};
```

**改成 (跟 setup_lark_base.py 真实 schema 对齐, 见根 README v0.2 清单)**:
```js
let body;
if (action === "contacted") {
  body = { recordId, followupStatus: "已联系" };
} else if (action === "followed-up") {
  body = { recordId, followupStatus: "已跟进" };
} else if (action === "unsuitable") {
  // v0.3: 已拒绝 + 不合适 → hub 会自动联动入公海池 (写 分配状态=公海池 +
  // 公海原因=跟进后拒绝 + 公海流入时间). 不必再调 /assign.
  body = {
    recordId,
    followupStatus: "已拒绝",
    feedbackQuality: "不合适",
    communicationResult: "跟进后拒绝",
  };
} else {
  // 兜底, 不动飞书表
  return;
}
```

**改 — 收 hub 响应后用 updatedCard 替换原卡** (这是 v0.3 新的):
```js
const response = await fetch(`${HUB_BASE_URL}/api/event-marketing/ai-leads/follow-up`, {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${process.env.HUB_WEBHOOK_SECRET}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(body),
});
const data = await response.json();

// v0.3: hub 响应里 updatedCard 是飞书互动卡片 JSON, 直接 return 给飞书
// card.action.trigger 回包, 飞书自动替换原派单卡 (按钮区消失变"已登记 ✓").
// 依赖派单卡 config.update_multi=true (hub v0.3 已配置).
if (data.ok && data.updatedCard) {
  return {
    toast: {
      type: "success",
      content: data.updated["分配状态"] === "公海池"
        ? "已标记 · 单子已入公海池"
        : "已登记跟进进展",
    },
    card: data.updatedCard,
  };
}
return { toast: { type: "info", content: "已收到, 请稍后" } };
```

### 补丁 2: 二次催卡 3 选项按钮也用 updatedCard 替换

**问题**: 二次催卡 (`buildFollowupNudgeCard`) 3 按钮 (`还在推进中 / 拿到决策时间 / 没下文了`) 点完按钮也应该消失, 不让销售反复点。

**文件**: 同上 `event-handlers.js`

**找**: 处理 `value: { kind: "ai-huoke-followup-result", recordId, result }` 的函数 (估计叫 `handleHuokeFollowupResult` 或类似)。

**改 — 收 hub /follow-up 响应 (带 followupNudgeResult 字段) 后, 同样用 updatedCard 替换**:
```js
const body = {
  recordId,
  followupNudgeResult: result,  // result ∈ {"还在推进中", "拿到决策时间", "没下文了"}
};
// 如果 result === "没下文了", 可以加 followupStatus="已退回" 让 hub 自动入公海池
if (result === "没下文了") {
  body.followupStatus = "已退回";  // 触发 hub 联动入公海池
  body.communicationResult = "放弃跟进";
}

const response = await fetch(`${HUB_BASE_URL}/api/event-marketing/ai-leads/follow-up`, { ... 同上 ... });
const data = await response.json();

if (data.ok && data.updatedCard) {
  return { toast: { type: "success", content: "已记录" }, card: data.updatedCard };
}
```

### 补丁 3: 自然语言路径也带 rawFeedback + 触发联动入公海池

**问题**: 销售在 DM 自然语言说"这家不合适, 做的不是亚马逊", 招财豆 LLM 抽取后调 hub /follow-up 时如果 followupStatus 是新枚举对的, hub 已经自动入公海 (v0.3 联动)。但如果 LLM 抽错了 (比如还在用老的"无意向" 或者没传 followupStatus), 就不会入公海。

**文件**: `/home/gem/workspace/agent/skills/ai-huoke-followup/SKILL.md`

**确认**: 该 SKILL.md 的 LLM 抽取规范跟 [zhaocaidou-skills sales/references/llm-extraction.md](../../sales/references/llm-extraction.md) 一致 (用新枚举 `已联系/已跟进/已报价/已成单/已退回/已拒绝`, 没有 `无意向/已加微信/已绑店`)。

**确认**: SKILL.md 要求 LLM 抽完字段时**永远带 `rawFeedback` 字段** (销售原话), 让管理员能 catch LLM 抽错丢的上下文。

---

## 完整改完后的验证 (5 步端到端)

```bash
# Step 1: 改完 event-handlers.js + SKILL.md, 重启招财豆
sh /home/gem/workspace/agent/scripts/restart.sh

# Step 2: 看 招财豆 真起来了
ps aux | grep openclaw | grep -v grep
tail -20 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# Step 3: 在自己 Mac 上派一条给"团结" (用 admin skill 或 curl)
curl -X POST "http://101.33.198.192/api/event-marketing/ai-leads/assign" \
  -H "Content-Type: application/json" \
  -d '{"updates":[{"recordId":"<选一条>","assignmentStatus":"已分配","assignedTo":"团结","leadSnapshot":{"company":"<那家>"}}]}'

# Step 4: 让团结点"❌ 不合适" 按钮 (在他飞书招财豆 DM 里)

# Step 5: 验证
# - 飞书 IM 上那张派单卡 buttons 应该消失, 变成"招财豆 · 跟进已登记 ✓" 灰色卡, 显示"📤 已自动入公海池 (跟进后拒绝)"
# - 飞书 AI 获客线索池表那一行: 跟进状态=已拒绝, 反馈质量=不合适, 沟通结果=跟进后拒绝,
#   分配状态=公海池, 公海原因=跟进后拒绝, 公海流入时间=点击时间
curl -fsS -H "Authorization: Bearer $ZHAOCAIDOU_WEBHOOK_SECRET" \
  "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads?refresh=1" \
  | jq '.leads[] | select(.recordId=="<上面那条>")'
```

## 改不动了怎么办

- 招财豆挂了 → 妙搭"AI 运维"按钮自愈, 或 `ps + restart.sh`
- 改坏了 → `cd /home/gem/workspace/agent && git status; git diff` 看本地 diff; `git checkout -- event-handlers.js` 回退
- 妙搭 Web 终端进不去 → 让 owner (Karen?) 在妙搭后台拉 Leo / 你进去

## 相关 hub 改动 (背景)

- hub commit `cd14a75` (v0.3): /follow-up 加联动 + updatedCard
- hub commit `608e5a3` (v0.2): nudge-2 路径 + followupNudgeResult 字段
- hub commit `530775c` (v0.2): /follow-up 白名单跟飞书表对齐
