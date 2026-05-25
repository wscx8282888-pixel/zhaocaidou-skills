# 销售名册快照

> ⚠️ 这是 hub `src/data/sales-roster.ts` 的快照, 可能滞后于代码。最新版以 hub repo 为准。
>
> 派单 `assignedTo` 必须用这里的"中文名", hub 端会自动选 receiver 类型 (open_id > chat_id > email) 推 IM 卡。

## 当前 (2026-05-25, v0.2)

| 中文名 | 部门 | receiver 类型 | id (脱敏) | 备注 |
|---|---|---|---|---|
| Leo(自测) | 测试 | open_id | `ou_387f...8f03a` | owner 兼测试销售 |
| 团结 | 测试 | chat_id | `oc_938b...9718` | v0.2 加, 招财豆↔团结 DM |
| 东东 | 测试 | chat_id | `oc_3db9...88a1` | v0.2 加, 招财豆↔东东 DM |
| Karen | 测试 | chat_id | `oc_517a...4f80` | v0.2 加, 招财豆↔Karen DM |

## v0.2 — 加新销售的 3 种方式 (任选其一, 不必都填)

hub `SalesEntry` 接口现在支持 3 种 receiver, 按优先级 open_id > chat_id > email 派 IM 卡:

```ts
// 方式 A: open_id (最严谨, 但要查飞书后台)
{ name: "张三", openId: "ou_xxx", dept: "南区" }

// 方式 B: chat_id (招财豆跟该销售已有 DM 时最快 — 直接复制 DM 会话 id)
{ name: "张三", chatId: "oc_xxx", dept: "南区" }

// 方式 C: email (飞书账号注册邮箱, HR 通讯录就有)
{ name: "张三", email: "zhang.san@dowsure.com", dept: "南区" }

// 混填也行 — hub 自动按优先级选
{ name: "张三", openId: "ou_xxx", chatId: "oc_xxx", email: "..." }
```

加完跑流程:

1. 编辑 `dowsure-hub/src/data/sales-roster.ts`, `SALES_ROSTER` 数组追加 entry
2. push 到 origin → GH Actions 自动 deploy 到 lighthouse
3. 同步更新本文件上面的表 + push zhaocaidou-skills

## 怎么拿这 3 种 id

| receiver | 怎么拿 |
|---|---|
| `openId` (ou_xxx) | 飞书开放平台 → 应用调试 → 通讯录 → 按手机/邮箱搜 → 取 open_id; 或在通讯录管理 URL 里看 |
| `chatId` (oc_xxx) | 招财豆已经跟该销售有 DM → DM 链接里就有, 或者在妙搭终端跑招财豆相关 script 查 |
| `email` | HR 通讯录, 或销售个人飞书账号 → 设置 → 邮箱 |

## 派单时 assignedTo 写啥?

写**中文名**, 不是 id。比如:

```json
{ "recordId": "recXXX", "assignmentStatus": "已分配", "assignedTo": "团结" }
```

hub 端 `/assign` 路由会查 SALES_ROSTER 找到 entry, 用 `salesToRecipient()` 解析成 `{idType, id}` 推 IM 卡。如果 `assignedTo` 不在名册或 entry 三个 id 全空, 表会写但卡不会推 (响应 `notifySkipped` 计数+1)。
