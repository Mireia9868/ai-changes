# AI Change Log — 最小验证版

一个"AI 工具变更流"：停运 / 涨价 / 降额 / 弃用 / 政策变更，每条带日期、状态（已确认 vs 仅报道）、截止日、迁移动作、原始来源。

目的不是做产品，是**用一天时间验证"有没有人要这个"**。

---

## 一、工作量评估（先看这个）

| 项 | 状态 | 人力当量 |
|---|---|---|
| 事件数据模型（8 个字段 + 来源 + 验证日期） | ✅ 已完成 | 0.5 h |
| 采集器（HN 公开 API，7 组关键词，240 天窗口） | ✅ 已完成 | 1.5 h |
| 静态生成器（首页 + 8 个事件页 + JSON-LD + feed + sitemap + llms.txt + robots.txt） | ✅ 已完成 | 1.5 h |
| 8 条种子事件（逐条核对来源后手写） | ✅ 已完成 | 1 h |
| **本次合计** | | **≈ 4.5 h** |
| 域名 + 部署（Vercel/Netlify 拖 `dist/`） | ⬜ 待你做 | 0.5–1 h |
| GSC 验证 + 提交 sitemap + IndexNow | ⬜ 待你做 | 0.5 h |
| 3 处分发（subreddit / X / 一个 Discord） | ⬜ 待你做 | 1 h |
| **上线总成本** | | **≈ 1 天** |
| 每周维护（跑采集 → 审 15 条 → 提升 2–3 条） | 持续 | **1–2 h/周** |

> 硬约束：**如果每周 1–2 小时的维护你不愿意付，这个方向就不成立。** 变更流的价值与新鲜度成正比，停更即归零。

---

## 二、目录结构

```
labs/ai-changes/
├── data/
│   ├── events.json        # 人工确认的事件（唯一会发布的数据）
│   └── candidates.json    # 采集器产出的原始候选（不发布，供审阅）
├── scripts/
│   ├── collect.mjs        # 拉 HN 公开 API，分类 + 打分，输出候选
│   └── build.mjs          # 渲染静态站到 dist/
└── dist/                  # 构建产物，可直接部署（file:// 也能打开预览）
```

无框架、无依赖、无构建缓存。Node 20+ 直接跑。

---

## 三、怎么用

```bash
NODE=$(command -v node || echo "$HOME/.dsh/dsh-runtimes/dsh-primary-runtime/dependencies/node/bin/node")

# 1. 采集候选（默认 180 天窗口）
$NODE scripts/collect.mjs 240

# 2. 人工审阅：看 data/candidates.json，挑出真实的，按 events.json 的结构补一条
#    —— 这一步不要自动化。自动化会生产 slop，slop 会杀死信任。

# 3. 构建
SITE_URL="https://your-domain.com" $NODE scripts/build.mjs

# 4. 预览
open dist/index.html
```

想加订阅框：`SUBSCRIBE_URL="https://buttondown.com/xxx" ... build.mjs`，不设就完全不渲染（不放假表单）。

---

## 四、设计上刻意做的几个决定

| 决定 | 原因 |
|---|---|
| 每条必须标 `confirmed` / `reported` | 唯一可持续的差异化是"敢说自己没确认"。这也让 AI 引用时不会替你撒谎 |
| 每条必须有 `verifiedAt` 并渲染出来 | GEO 的四大引用门槛之一是"较新的日期"，且这是你能兑现的 |
| 页面纯静态，`file://` 可开 | 停更也不烂；不会被框架升级绑架 |
| 采集器只产出候选，不发布 | 避免"批量 AI 生成"→ 薄页 → 政策打击（你在 AdSense 那次已经踩过） |
| 同时输出 `feed.json` / `llms.txt` / `sitemap.xml` | 从第一天就当"数据源"，而不是"网页"。这是 codex-reset 那类产品的真正护城河 |
| 不写"Top 10 AI tools" | 正面战场已被 AI 概览吃掉 |

---

## 五、14 天怎么判生死

| 天 | 动作 | 通过线 |
|---|---|---|
| 1 | 部署 + GSC + sitemap | 首页可索引 |
| 2 | 3 处分发，标题就用"X is shutting down — what to do" | ≥ 5 条真实互动 |
| 3–7 | 看 GSC impressions | ≥ 20 |
| 8–14 | 看是否有人问"能不能订阅/有没有 API" | ≥ 1 个 |
| 14 | 决策 | 见下 |

**Kill：**
- 7 天 0 impressions → 需求不存在或词太新
- 有 impressions 但 CTR < 1% → 改一次标题，再 7 天不行 → kill
- 有流量但 0 订阅、0 主动询问 → 一次性好奇，转静态页不再投入

**加注（任一）：** 有人要 alert/API · 7 天自然回访 > 20 · 有厂商愿意换量或付钱。

---

## 六、下一步的三个分支

- **加注**：把 `collect.mjs` 扩到 OpenAI/Anthropic/Google 的 changelog RSS + GitHub release feed，加邮件订阅，出 `/api/changes.json`。
- **换题材**：同一套骨架换数据源即可复用（模型弃用、免费额度变更、区域可用性）。`events.json` 的 schema 不用改。
- **停**：`dist/` 留着当静态页，长尾会自己涨，零维护成本。

---

## 七、已知缺口（别假装没有）

- 种子只有 8 条，覆盖 8 个月，密度不足以让人"每周回来"。要靠采集器补到 20–30 条才算像样。
- 采集器只覆盖 HN 一个源，会漏掉大量只在 X/Discord 宣布的变更。
- `reported` 的条目还没有回归确认机制（90 天后应自动标记为"待复核"）。
- 没有订阅后端。`SUBSCRIBE_URL` 得你自己接一个。
- 变现路径未接：停运工具的替代品可以做联盟，但需要先有流量。
