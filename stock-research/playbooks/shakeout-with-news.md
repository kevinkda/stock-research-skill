# Playbook — Shakeout with news（shakeout 信号叠加新闻情绪）

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/shakeout-news-YYYY-MM-DD.md`（新建） |
| schwab tools used | `health_check`, `get_cache_stats`, `get_price_history`, `get_quotes` |
| sec-edgar tools used | （本 playbook 不调用 sec-edgar） |
| polygon tools used | `health_check`, `sentiment_aggregate`, `ticker_news` |
| max tool calls | ≤ 18（schwab ≤ 8 + polygon ≤ 8 + 2 health） |
| 模型来源 | `trackers/voo-qqq-tracker.md §10`（Tang Keyin 私有方法论；本 playbook **不复述**模型，只引用） |

> **本 playbook 仅在 stock-personal 仓库内运行**。如果 cwd 不在
> `${target_repo}` 子树下，必须切换为只读模式：分析输出到聊天上下文，
> **禁止落盘到任何其他仓库**。

## Pre-flight（mandatory）

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" 且
                                            server_version ∈ ">=0.3,<0.4"
2. polygon-news-mcp.health_check()        → api_key_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
3. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
4. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true（else STOP；不要绕过）
5. cwd ∈ ${target_repo} subtree？若否 → 只读模式（仅输出到聊天）
6. ls ${target_repo}/trackers/voo-qqq-tracker.md 必须存在并可读；
   否则 STOP 并报告 "shakeout 模型来源缺失"。
```

## Steps

### Step 1 — 选取分析标的

默认扫描 `["VOO", "QQQ", "SPY"]`（与 voo-qqq-tracker 对齐）。
用户可以指定自定义 watchlist（≤ 5 个标的，受 max tool calls 约束）。

### Step 2 — 触发 schwab shakeout 检测

对每个 symbol 调用：

```text
schwab-marketdata-mcp.get_price_history(
    symbol=...,
    period_type="MONTH",
    period="THREE_DAYS",
    frequency_type="DAILY",
)
schwab-marketdata-mcp.get_quotes(
    symbols=["VIX", *symbols],
    fields=["QUOTE", "REGULAR"],
)
```

按 `trackers/voo-qqq-tracker.md §10` 的 8 项信号扫描在**本地**计算（不再
触发任何 schwab 调用）。**模型口径以 §10 文件当前内容为准**——
本 playbook 不复制 §10 的阈值或决策矩阵。

### Step 3 — 拉取新闻情绪聚合

仅对 Step 2 中**触发 shakeout 信号**（即决策矩阵命中 HOLD/REVIEW/TRIM）
的 symbol 调用：

```text
polygon-news-mcp.sentiment_aggregate(
    ticker=...,
    window="7d",          # 最近 7 天滚动情绪
)
```

期望返回字段：`avg_sentiment ∈ [-1, 1]`、`positive_count`、
`negative_count`、`neutral_count`、`article_count`、`window_start`、
`window_end`。

### Step 4 — 拉取头部新闻原文（仅 top 3）

为每个命中标的拉 top 3 篇新闻供报告引用：

```text
polygon-news-mcp.ticker_news(
    ticker=...,
    limit=3,
    order="desc",
    sort="published_utc",
)
```

仅记录字段：`title`、`published_utc`、`publisher.name`、`article_url`、
`insights[].sentiment`（如果可得）、`insights[].sentiment_reasoning`。
**不在报告里转载新闻正文**（避免版权问题）。

### Step 5 — 生成交叉报告

把以下 8 段 markdown 写入
`${target_repo}/research/shakeout-news-YYYY-MM-DD.md`：

1. **Frontmatter**：`generated_at` (UTC), `symbols`, `mcp_versions`
   (3 个)，`cache_hit_rate`（schwab）。
2. **TL;DR**：单行结论，例如
   "QQQ 命中 shakeout（7/8）+ 新闻情绪 +0.42 → 维持 HOLD，外部叙事支持"。
3. **每个 symbol 的 §10 8 项信号表**（来自 Step 2，引用而非复述模型）。
4. **每个命中标的的新闻情绪聚合表**（avg_sentiment / counts / window）。
5. **新闻 × 信号叠加判定**：4 种组合的标准化结论
   （shakeout-命中 × 情绪-正、shakeout-命中 × 情绪-负、
   shakeout-反转 × 情绪-正、shakeout-反转 × 情绪-负）。
6. **Top 3 新闻引用**（每标的，仅 title+url+publisher+发布时间）。
7. **风险提示**：列举本期 §10.6 失效场景是否触发；列举新闻面 1-2 个
   未消化风险点。
8. **数据出处与限制**：链接 voo-qqq-tracker.md §10、schwab cache 命中率、
   polygon API tier、Schwab Market Data 不可二次分发声明。

### Step 6 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 5 已写入文件
git add research/shakeout-news-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): shakeout-with-news $(date -u +%Y-%m-%d)"
# DO NOT push --force.  普通 push 到 research 分支即可。
```

## Acceptance criteria

完成后逐项验证（每项跑命令并确认输出，再勾选）：

- [ ] **commit 已创建**：`git -C ${target_repo} log -1 --format="%H %s"`
      输出最新 commit hash + `research(cross-mcp):` 前缀的 message
- [ ] **research/ 下有当日新文件**：
      `ls ${target_repo}/research/shakeout-news-$(date +%Y-%m-%d).md`
- [ ] **仅 research/ 被改动**：
      `git -C ${target_repo} diff --stat HEAD~1` 列出的所有文件路径都在
      `research/` 目录下
- [ ] **3 个 health_check 当时均 valid**：pre-flight 输出留底（聊天上下文）
- [ ] **schwab 缓存命中率 ≥ 30%**：playbook 末尾再调一次
      `schwab-marketdata-mcp.get_cache_stats()`，`hit_rate_24h ≥ 0.3`
- [ ] **报告含全部 8 段**：
      `grep -c '^##' ${target_repo}/research/shakeout-news-$(date +%Y-%m-%d).md` ≥ 8
- [ ] **无来自 §10 的逐字复制**：随机抽 1 段 §10 原文，
      `grep -F` 在新报告里**不应**出现完整复制
- [ ] **新闻 url 可点击**：`grep -E '^- \[.*\]\(https?://' ...md | wc -l ≥ 3`

## Rollback

```bash
cd ${target_repo}
git reset --soft HEAD~1   # 撤销 commit，保留 working tree
# 检查 working tree 后决定 git restore 或 git stash
git restore research/shakeout-news-$(date +%Y-%m-%d).md
# 绝不 force-push 主分支。
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `trackers/voo-qqq-tracker.md` §10 不存在或被裁掉 | **STOP**。报告 "shakeout 模型来源缺失"；**绝不编造模型**。 |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP，要求用户先跑 `auth login_flow` |
| `SchwabRateLimitError` | 等 `retry_after_seconds`，重试 1 次；连续两次 STOP |
| `polygon-news-mcp` 返回 401/403 | STOP，要求用户检查 `POLYGON_API_KEY` |
| `polygon-news-mcp` 返回 429 | 等 60s 重试 1 次；继续失败则 STOP |
| `gh repo view` 失败 / 仓库不是 private | **STOP and refuse to write**；不绕过 |
| 同一日已存在 `research/shakeout-news-YYYY-MM-DD.md` | 询问用户是否覆盖；默认 skip 并告知 |
| VIX quote 不可得 | 信号 #7 标 `N/A`，决策矩阵改用 7 项加权；**不要伪造 VIX** |
| `sentiment_aggregate` 返回空（article_count == 0） | 在报告里如实标"7 日内无相关新闻"；不补 0 |
| schwab cache lock 冲突（另一进程在用 `cache.duckdb`） | 等 5s 重试 1 次；仍冲突 → 设 `SCHWAB_CACHE_BYPASS=1` 跑实时路径，并在 frontmatter 注明 "cache locked, bypassed" |
