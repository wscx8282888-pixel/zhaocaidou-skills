# 销售名册快照

> ⚠️ 这是 hub `src/data/sales-roster.ts` 的快照, 可能滞后于代码。最新版以 hub repo 为准。
>
> 派单 `assignedTo` 必须用这里的"中文名", hub 端会查 openId 推 IM 卡。

## 当前 (2026-05-25)

| 中文名 | 部门 | openId | 备注 |
|---|---|---|---|
| Leo(自测) | 测试 | `ou_387fccc0ea601ae61869c37d93f8f03a` | owner 兼测试销售 |

## 加新销售流程

不是 skill 这边改, 是 **dowsure-hub 仓** 改:

1. 在飞书后台拿该销售 openId (`ou_...`):
   - 开放平台 → 应用调试 → 通讯录 → 按手机/邮箱搜 → 取 openId
   - 或 通讯录管理 → 成员详情 → URL 里看
2. 编辑 `dowsure-hub/src/data/sales-roster.ts`, `SALES_ROSTER` 数组追加:
   ```ts
   { name: "张三", openId: "ou_xxxx", dept: "南区" },
   ```
3. push 到 origin → Vercel/lighthouse 自动部署
4. 同步更新本文件上面的表 (push 这个 skill 仓)

## 派单时 assignedTo 写啥?

写**中文名**, 不是 openId。比如:

```json
{ "recordId": "recXXX", "assignmentStatus": "已分配", "assignedTo": "Leo(自测)" }
```

hub 端 `/assign` 路由会查 SALES_ROSTER 找 openId 推 IM 卡。如果 `assignedTo` 不在名册, 表会写但卡不会推 (响应 `notifySkipped` 计数+1)。
