# 故障排查 (admin 视角)

## 装机检查

```bash
echo "HUB: $ZHAOCAIDOU_HUB_BASE_URL"
echo "SECRET (长度): $(echo -n "$ZHAOCAIDOU_WEBHOOK_SECRET" | wc -c)"
# 64 才对; 0 = 没装
```

## hub 通不通?

```bash
curl -fsS -o /dev/null -w "HTTP %{http_code}\n" "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads"
# 200 = 通
```

不通 (timeout / 5xx) 一般是 lighthouse 上 pm2 挂了:
- 让 Leo SSH `ssh ubuntu@101.33.198.192` 跑 `pm2 status` / `pm2 restart dowsure-hub`
- 或看 `~/dowsure-hub/.next-build.log`

## Bearer 鉴权报 401

```json
{"error": "unauthorized"}
```

- `$ZHAOCAIDOU_WEBHOOK_SECRET` 没设、值错、含换行
- lighthouse 上 hub 的 `.env` 里 `OPENCLAW_WEBHOOK_SECRET` 跟你本地不一致
- 修法: 让 Leo 比对两端 secret 长度 + 首尾 8 位 (不打全文)

## 派单后 IM 卡没推 (notifySkipped > 0)

响应里看 `details[*].notifySkipped=true`, 原因:
- `assignedTo` 不在 sales-roster (写飞书成功但找不到 openId)
- 飞书 agent token 拿失败 (lighthouse env 没配 FEISHU_AGENT_APP_ID/SECRET)
- 招财豆 app 在飞书后台权限被收回

## 飞书表写失败 (failed > 0)

- 招财豆 app 在 "AI 获客线索池"表 (`tblPviQZ92U30tUT`) 的权限被收回
- recordId 已删 / 字段名飞书侧改了

## 缓存陈旧 (UI 跟 jq 不一致)

```bash
curl -fsS "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads?refresh=1" | jq '.cache'
```

`refresh=1` 强制洗两层缓存 (SwrCache + unstable_cache)。

## /nudge 报"nudged=0"但你明明知道有超时的

`skipped[]` 看原因:
- "20h 内已催过" → cron 已经跑过了, 你再手动跑就被去重
- "状态非已分配" → 该 lead 已被改成"销售退回"/"无效"等
- "assignedTo 不在 sales-roster" → 名册过期, 去 hub 加

## 招财豆挂了 (派单后卡完全没推, /assign 响应里 notified=0)

这把 skill 管不了招财豆运行时。让 Leo:
1. 去妙搭 https://miaoda.feishu.cn/app/app_4jyb648smdxwm/ 终端 tab
2. `ps aux | grep openclaw | grep -v grep` 看进程
3. `sh /home/gem/workspace/agent/scripts/restart.sh`
4. 或点妙搭的"AI 运维"自愈
