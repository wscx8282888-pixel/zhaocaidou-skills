# 故障排查 (sales 视角)

## 装机自检

```bash
echo "HUB:    $ZHAOCAIDOU_HUB_BASE_URL"
echo "NAME:   $ZHAOCAIDOU_SALES_NAME"
echo "SECRET 长度: $(echo -n "$ZHAOCAIDOU_WEBHOOK_SECRET" | wc -c)"
# HUB 应该是 http://101.33.198.192 之类的 URL
# NAME 应该是你在销售名册里的中文名 (一字不差)
# SECRET 长度应该是 64
```

任意一项空 / 0 → 找 Leo 拿 + 在 ~/.zshrc 加 export + source 重启 Claude Code。

## "我手上的线索" 是空

最常见 3 种原因:

1. **`$ZHAOCAIDOU_SALES_NAME` 跟名册不一字不差**  
   名册里是 "Leo(自测)" 但你写成 "Leo 自测" 或 "Leo（自测）" (中文全角括号), 都会过滤不出来。让用户找 Leo 确认名册原文。

2. **你确实手上没单**  
   先看全表里你的状态: 撇开身份过滤, jq 数 `.leads[] | .assignedTo` 的频次:
   ```bash
   curl -fsS "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads" \
     | jq '[.leads[] | .assignedTo] | group_by(.) | map({sales: .[0], count: length}) | sort_by(-.count)'
   ```
   看里头有没有跟 `$ZHAOCAIDOU_SALES_NAME` 接近的名字 — 也许是录入时拼错。

3. **hub 缓存陈旧** (5min)  
   加 `?refresh=1` 强制刷一次, 再查:
   ```bash
   curl -fsS "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads?refresh=1" | jq '.cache'
   ```

## Bearer 401 (写跟进时)

```json
{"error": "unauthorized"}
```

- `$ZHAOCAIDOU_WEBHOOK_SECRET` 没设、值错、含换行
- lighthouse hub 端的 `OPENCLAW_WEBHOOK_SECRET` 跟你本地不一致 (Leo 那边换过 secret 没同步给你)
- 修法: 让 Leo 比对两端 secret 的长度 + 首尾 4 位 (不打全文)

## hub 不通 (timeout / 5xx)

```bash
curl -fsS -o /dev/null -w "HTTP %{http_code}\n" "$ZHAOCAIDOU_HUB_BASE_URL/api/event-marketing/ai-leads"
# 200 = 通; 其它一般是 lighthouse 上 pm2 挂了
```

不在你 skill 修复范围, 让 Leo 看 pm2 / lighthouse 服务器。

## 写完了飞书表没变

1. 响应里 `ok: true` 但 `updated: {}` 几乎空 → 字段全被 skip (不在白名单), 看 `skipped[]` 原因
2. `ok: true` 且 `updated` 非空 → 写成功了。但飞书表你刷新还是旧的:
   - hub 缓存被你的写入 invalidate 了, 但飞书表的 web view 自己有缓存, 等 30s 或刷页面
   - 表是不是你看错了? hub 写的是"AI 获客线索池"表 (`tblPviQZ92U30tUT`)

## 报"那不是分配给你的"但用户坚持是他的

可能销售名册里那条 lead 的 assignedTo 没填或者填错了。让用户:
1. 去飞书 IM 看招财豆有没有给他推过这条 lead 的卡 — 如果推了说明 hub 是认他的
2. 没推 → 让 Leo 在 hub UI /tools/event-marketing 检查那条 lead 的 `已分配给` 字段, 或者重新派一次

## LLM 抽错了字段

`response.skipped[]` 里看 hub 报的 "xxx 不在白名单" 提示。最常见:

- `followupStatus="跟进中"` (不是合法值, 应该是 `已跟进`)
- `feedbackQuality="好"` (应该是 `质量好`)
- `nextFollowupDate="下周一"` (LLM 应该算出 `2026-06-02` 这种 ISO)

让 Claude (你自己) 改抽取再发一次。如果反复抽错同一类, 反馈给 Leo 让他改 references/llm-extraction.md 加约束。

## "我成单了, 但金额不对"

如果飞书表写的金额跟你说的差一个零 → LLM 单位换算错 ("5万" 抽成 5 而不是 50000)。**永远在草稿里把金额写成 `¥128,000 (12.8 万)` 让销售看**, 不要只写 `128000`。
