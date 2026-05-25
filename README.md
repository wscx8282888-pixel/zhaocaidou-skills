# 招财豆 Skills

> dowsure 招财豆 AI 获客系统的 Claude Code 技能包。
> 两个独立 plugin, 一个给运营管理员, 一个给一线销售, 各自只装自己那把。

---

## 这是什么

[**招财豆**](https://github.com/wscx8282888-pixel/dowsure-hub) 是 dowsure 的 AI 获客 agent (跑在飞书妙搭云电脑的 OpenClaw 文件型 agent)。每天 08:00 自动从小蓝本拉新线索 → 写入飞书"AI 获客线索池"表 → 管理员审一遍 → 派给销售 → 销售在飞书 IM 收到富信息卡 → 点按钮或自然语言回复 → 招财豆 LLM 抽取信息回写表 → 形成闭环。

这套技能包让招财豆的能力**从飞书 IM 延伸到 Claude Code**:

- **管理员**: 不用打开 hub `/tools/event-marketing` 网页, 直接在 Claude Code 里问 "今天新单多少", "把 XX 派给 YY", "哪些超 24h 没动"
- **销售**: 不用记字段名, 直接说 "我跟张总加上微信了, 下周二线下见", Claude 自动抽出结构化字段写回飞书表

整套数据**最终落在飞书表**, 这里只是操作入口。底下还是同一个 dowsure-hub。

---

## 两个 skill 的视角对比

| | `zhaocaidou-admin` (管理员) | `zhaocaidou-sales` (销售) |
|---|---|---|
| **角色** | 运营经理 / 项目 owner | 一线 BD / 销售 |
| **看什么** | AI 获客线索池**全表** (含 stats) | **只看自己名下**的线索 |
| **能改谁** | 任何一条 lead | 只能改自己名下的 |
| **能派单吗** | ✅ 派给任何销售 (含 leadSnapshot, 招财豆推 IM 富卡) | ❌ 派单是管理员的事 |
| **能催跟进吗** | ✅ 手动触发 `/nudge` | ❌ 催单是管理员/cron 的事 |
| **能标跟进吗** | ✅ 任意 record (代销售操作) | ✅ 只能自己名下的 |
| **能写反馈吗** | ✅ 表单式直填字段 | ✅ **自然语言抽取**, 提交前必须给销售看草稿确认 |
| **典型对话** | "今天有几条新单, 把 ⭐ 的派给 ZZ" | "我跟 XX 聊了, 他说下周再说" |
| **依赖 env** | `ZHAOCAIDOU_HUB_BASE_URL` + `ZHAOCAIDOU_WEBHOOK_SECRET` | 上面两个 + `ZHAOCAIDOU_SALES_NAME` (你在销售名册里的中文名) |

---

## 装机

### 0. 装前置 (两把 skill 都需要)

问 Leo 拿 `OPENCLAW_WEBHOOK_SECRET` (64 位 hex)。在 `~/.zshrc` 或 `~/.bashrc` 末尾加:

```bash
# 招财豆 skills (两把都需要)
export ZHAOCAIDOU_HUB_BASE_URL="http://101.33.198.192"
export ZHAOCAIDOU_WEBHOOK_SECRET="<问 Leo 拿>"

# 只销售装这个: 你在 sales-roster 里的中文名 (一字不差, 含括号和半角符号)
export ZHAOCAIDOU_SALES_NAME="张三"
```

加完 `source ~/.zshrc`, 然后**重启 Claude Code** (env 只在进程启动时读, 不重启拿不到)。

> ⚠️ **secret 三不**: 不进 git, 不进 Slack/飞书消息, 不要发给 Claude 让它"帮你看"。chmod 600 ~/.zshrc 是好习惯。

### 1. 加 marketplace

```
/plugin marketplace add wscx8282888-pixel/zhaocaidou-skills
```

### 2. 装对应身份的 plugin

```
# 管理员 (Leo 等运营):
/plugin install zhaocaidou-admin@zhaocaidou-skills

# 销售 (BD):
/plugin install zhaocaidou-sales@zhaocaidou-skills
```

**两个都装也行** (比如管理员想体验销售视角)。skill 会根据你自然语言里说的话决定触发哪把。

### 3. 验证

新开一个 Claude Code session, 输入:

```
看下 zhaocaidou-admin / zhaocaidou-sales 装好没, env 三个都对没
```

skill 会自检 env + 打通 hub 一次, 5 秒内告诉你装好没。

---

## 升级

招财豆和 hub 还在迭代, 这把 skill 跟着 hub 升级。**装好以后跟随 git push 自动更新**:

```
/plugin marketplace update zhaocaidou-skills
/plugin update zhaocaidou-admin     # 或 zhaocaidou-sales
```

每次 hub /follow-up 加字段、销售名册加人、新加 endpoint, 这把 skill 会在本仓 push, 你们 update 一下就同步。

---

## 仓库结构

```
zhaocaidou-skills/
├── .claude-plugin/marketplace.json         ← marketplace 入口, 列两个 plugin
├── README.md                                ← 你正在读
│
├── admin/                                   ← 管理员 plugin (zhaocaidou-admin)
│   ├── README.md                            ← 管理员视角详解
│   ├── .claude-plugin/plugin.json
│   ├── skills/zhaocaidou-admin/SKILL.md     ← 主 skill 文件 (Claude 读这个)
│   └── references/
│       ├── sales-roster.md                  ← 销售名册快照 (含 openId)
│       ├── endpoint-schema.md               ← 4 个 endpoint 完整 schema
│       └── troubleshooting.md               ← 8 种常见错误排查
│
└── sales/                                   ← 销售 plugin (zhaocaidou-sales)
    ├── README.md                            ← 销售视角详解
    ├── .claude-plugin/plugin.json
    ├── skills/zhaocaidou-sales/SKILL.md     ← 主 skill 文件
    └── references/
        ├── endpoint-schema.md               ← 销售能调的 2 个 endpoint
        ├── llm-extraction.md                ← 销售口语 → 字段 8 个示例
        └── troubleshooting.md               ← 销售特有 8 种问题
```

详细视角说明各自见 [admin/README.md](admin/README.md) 和 [sales/README.md](sales/README.md)。

---

## 设计原则 (为啥这样切)

1. **薄壳**: skill 不存任何业务状态, 每次调用都打 hub。这样所有人看到的数据跟 hub UI 一致, 永不漂移。
2. **不绕过 hub 写飞书**: 直接调飞书 OpenAPI 也能写, 但会绕开 hub 的 revalidateTag, 导致 hub UI 缓存陈旧。**永远走 hub**。
3. **env 锁身份**: 销售名册的中文名是 hub 强 contract, sales skill 用 `$ZHAOCAIDOU_SALES_NAME` env 硬绑。对话里临时说"我是 X" 不可信。
4. **secret 走 env 不进 git**: 所有 curl 用 shell 变量引用, 永不写明文。
5. **自然语言提交前必须确认**: LLM 抽错的成本 (改飞书表) 比销售点一下"嗯" 高 10 倍。永远先草稿。

---

## hub 侧依赖

这把 skill 依赖 hub 的两个 endpoint 双轨鉴权 (Bearer 或 cookie):

- `GET /api/event-marketing/ai-leads` — 看线索 (hub commit [`6a165d1`](https://github.com/wscx8282888-pixel/dowsure-hub/commit/6a165d1) 之后才支持 Bearer)
- `POST /api/event-marketing/ai-leads/follow-up` — 写跟进 (Bearer 一直支持)

管理员还用 `POST /assign` (无 caller auth, hub 内部 agent token) 和 `POST /nudge` (Bearer)。

如果 hub 没升级到 6a165d1 之后, skill GET 会被 302 到 /login。这是底线。

---

## 相关

- [dowsure-hub](https://github.com/wscx8282888-pixel/dowsure-hub) — hub 本体, AI 获客 endpoint 在 `src/app/api/event-marketing/ai-leads/`
- 招财豆运行时 — 飞书妙搭 OpenClaw fork, 管理后台 `https://miaoda.feishu.cn/app/app_4jyb648smdxwm/`
- AI 获客全链路设计 + 15 个踩坑笔记 — Leo 的 memory `project_ai_huoke_integration.md`
