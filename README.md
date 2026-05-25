# 招财豆 Skills (zhaocaidou-skills)

招财豆 AI 获客系统的 Claude Code 技能包。两个独立 plugin:

| Plugin | 给谁装 | 能做啥 |
|---|---|---|
| `zhaocaidou-admin` | 运营管理员 (Leo) | 派单 / 审单 / 催单 / 看全表 / 调销售名册 |
| `zhaocaidou-sales` | 一线销售 | 看自己手上的线索 / 标已联系/已跟进 / 自然语言写反馈 |

两个 plugin 都通过 Claude Code 调 `dowsure-hub` 的 AI 获客 API (lighthouse 101.33.198.192)。
不直接读写飞书表 — hub 端统一处理鉴权、写表、洗缓存。

---

## 装机 (一次性)

```bash
# 1. 加 marketplace
/plugin marketplace add wscx8282888-pixel/zhaocaidou-skills

# 2. 装对应身份的 plugin
/plugin install zhaocaidou-admin@zhaocaidou-skills   # 管理员
# 或
/plugin install zhaocaidou-sales@zhaocaidou-skills   # 销售
```

## 配本地 env (装完一次, 不要进 git!)

两个 plugin 都需要 hub 鉴权 secret。问 Leo 拿后, 写进你的 shell rc:

```bash
# ~/.zshrc 或 ~/.bashrc
export ZHAOCAIDOU_HUB_BASE_URL="http://101.33.198.192"
export ZHAOCAIDOU_WEBHOOK_SECRET="<问 Leo 拿>"
```

`source ~/.zshrc` 后, Claude Code 启动会自动读到。

> ⚠️ 这把 secret **不进 git / 不进对话**。
> Skill 里所有 curl 都用 `$ZHAOCAIDOU_WEBHOOK_SECRET` 引用, 永远不写明文。

## 升级

```bash
/plugin marketplace update zhaocaidou-skills
/plugin update zhaocaidou-admin     # 或 zhaocaidou-sales
```

repo 跟着招财豆开发同步更新, 装完一次以后会自动跟。

---

## 仓库结构

```
zhaocaidou-skills/
├── .claude-plugin/marketplace.json        ← marketplace 入口
├── README.md                              ← 你正在读
├── admin/                                 ← 管理员 plugin (zhaocaidou-admin)
│   ├── .claude-plugin/plugin.json
│   ├── skills/zhaocaidou-admin/SKILL.md   ← 主 skill
│   └── references/                        ← endpoint schema / 派单 SOP
└── sales/                                 ← 销售 plugin (zhaocaidou-sales)
    ├── .claude-plugin/plugin.json
    ├── skills/zhaocaidou-sales/SKILL.md   ← 主 skill
    └── references/                        ← LLM 抽取模板 / 字段白名单
```

## 相关文档

- 招财豆全链路: [project_ai_huoke_integration](https://github.com/wscx8282888-pixel/dowsure-hub) memory
- 招财豆运行时: 文件型 OpenClaw agent on 飞书妙搭
- hub API: `dowsure-hub/src/app/api/event-marketing/ai-leads/`
