# Playbook — Earnings preview（财报前 IV-rank-aware 定位简报）

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/earnings-preview-{TICKER}-YYYY-MM-DD.md`（新建） |
| schwab tools used | `health_check`, `get_cache_stats`, `get_iv_percentile`, `get_price_history` |
| sec-edgar tools used | `health_check`, `get_8k_with_items` |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 10（schwab ≤ 4 + sec-edgar ≤ 2 + polygon ≤ 1 + 3 health_check） |
| 数据时效窗口 | 8-K item 2.02 默认 since_days=90（覆盖最近 1 季度 + 1 个 historical earnings reference）；新闻情绪 7 天滚动；价格走势 30 天 |
| 数据合规 | SEC EDGAR Form 8-K 公开；polygon 新闻情绪与 schwab IV/价格均不可二次分发 |
| 用例 | 财报前 1-3 天对**单个 ticker** 给出 1 页 IV-rank-aware 定位简报（≤ 300 行） |
| 触发关键词 | "财报前瞻" / "earnings preview" / "财报要关注什么" / "财报即将发布" |

> **本 playbook 仅在 stock-personal 仓库内运行**。如果 cwd 不在
> `${target_repo}` 子树下，必须切换为只读模式：分析输出到聊天上下文，
> **禁止落盘到任何其他仓库**。

## Pre-flight（mandatory）

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" 且
                                            server_version ∈ ">=0.4,<0.5"
                                            （需 v0.4.0+ 才有 get_iv_percentile）
2. sec-edgar-mcp.health_check()           → user_agent_configured == true 且
                                            server_version ∈ ">=0.2.1,<0.3"
2.5. 验证 sec-edgar 服务端 UA 真实可达性（与 SKILL.md §Activation handshake
     step 2.5 完全一致）：读 health_check 返回的 sec_ua_reachable.status：
   - ACCEPTED       → ✅ 继续
   - REJECTED_HTML_403 → ❌ STOP，让用户改 SEC_EDGAR_USER_AGENT 为真实邮箱
   - UNCONFIGURED   → ❌ STOP，让用户配置 SEC_EDGAR_USER_AGENT
   - TIMEOUT / NETWORK_ERROR → ⚠️ WARN，继续但报告标 "SEC 探针不可用"
3. polygon-news-mcp.health_check()        → api_key_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true（else STOP；不要绕过）
6. cwd ∈ ${target_repo} subtree？若否 → 只读模式（仅输出到聊天）
```

## Steps

### Step 1 — 预读 watchlist 并选取 ticker

只读访问 `${target_repo}/portfolio/watchlist.md`：

- 解析 ticker 列表（去重、去空白）
- 用户传入 ticker → 直接用；未传 → 询问；不要默认扫描全 watchlist
  （earnings-preview 是**单标的** playbook，避免 quota 超额）
- 若 ticker **不在** watchlist：**接受**，但报告 frontmatter 标
  `coverage: ad-hoc`（区别于 `coverage: watchlist`）

### Step 2 — 拉取最近 8-K item 2.02 历史 filings

调用 sec-edgar 拉历史 earnings filings（用于推断**下一个**财报日 +
计算历史 earnings move 统计）：

```text
sec-edgar-mcp.get_8k_with_items(
    cik_or_ticker=<ticker>,
    item_codes=["2.02"],   # SEC item 2.02 = Results of Operations and Financial Condition
    since_days=90,         # 覆盖 1 季度 + 上一季度 reference
)
```

预期字段：`accession_number`、`filing_date`、`reported_date`、
`items`、`primary_document_url`、`form`（"8-K" / "8-K/A"）。

**下一财报日推断逻辑**（本地，不再调 SEC）：

- 取最近 3 期 item 2.02 8-K 的 `filing_date`，计算相邻间隔的中位数
  （US 大盘股通常 ≈ 91 天）
- `next_earnings_estimated = max(filing_date) + median_interval`
- 如间隔标准差 > 14 天 → 标 `low_confidence` 并在报告里告知
  "下一财报日推断置信度低，建议交叉验证 IR 官网"

### Step 3 — 获取 IV percentile（v0.4 P1/C 数据）

调用 schwab v0.4.0+ 新工具（**默认 refresh=False**，从 iv_history
缓存读，避免触发 option chain 拉取）：

```text
schwab-marketdata-mcp.get_iv_percentile(
    underlying=<ticker>,
    expiry_bucket="30d",   # 30d ATM IV 最贴近财报跨式定位
    lookback_days=252,     # 1 个交易年
    refresh=False,         # 缓存优先；批量/调度场景默认值
)
```

预期字段：`underlying`、`expiry_bucket`、`current_atm_iv`、
`percentile_rank` ∈ [0, 100] | None、`sample_count`、
`lookback_start`、`lookback_end`、`warning?`。

**IV rank 阈值告警表**（本地判定，写入报告 §6）：

| current_atm_iv 在 lookback 中的 percentile_rank | 期权定价含义 | 推荐定位 |
| --- | --- | --- |
| < 30 | IV 偏低 → 期权偏便宜 → 市场低估 earnings move | **long_iv_straddle**（买跨式）候选 |
| 30 ≤ rank ≤ 70 | IV 正常 | **no_action**（不主动建期权头寸） |
| > 70（特别 > 90） | IV 偏高 → 期权偏贵 → 市场已 priced-in earnings move | **sell_iv_condor**（卖宽跨式 / iron condor）候选 |
| `null` + `sample_count_below_30` 警告 | 数据稀疏 | **insufficient_data**，标 "low confidence"，**不**给推荐 |

> **重要免责**：以上是定位**信号草案**，不是交易指令。User
> 必须自行评估方向性观点、IV crush 风险、bid-ask spread、流动性。

### Step 4 — 获取近期价格走势

调用 schwab 拉 30 天日 K：

```text
schwab-marketdata-mcp.get_price_history(
    symbol=<ticker>,
    period_type="MONTH",
    period="ONE_DAY",       # 30 个 trading days，对齐 IV 30d bucket
    frequency_type="DAILY",
)
```

> 工具签名以 `get_server_info()` 实际暴露为准；如 `period` 字段名
> 在你跑的版本上不同（如 `period_count` / `lookback`），按 server
> 实际签名调用，不要凭记忆猜。

本地计算（不再调 schwab）：

- 30 天最高 / 最低 / 收盘
- ATR(14) 估算（衡量"非财报日"波动）
- 距 52-week 高 / 低距离（schwab quote 返回值，**复用 watchlist
  缓存**而非额外调 quote）
- 关键支撑 / 阻力（用最近 30 天的 swing high/low + EMA21）

### Step 5 — 获取 polygon news sentiment（财报前 7 天情绪）

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<ticker>,
    window="7d",   # 财报前 7 天滚动情绪
)
```

预期字段：`avg_sentiment ∈ [-1, 1]`、`positive_count`、
`negative_count`、`neutral_count`、`article_count`、`window_start`、
`window_end`、`top_articles[]`（如可得，含 title / url / publisher /
published_utc）。

**情绪判定**：

- `avg_sentiment ≥ +0.3` 且 `positive_count > negative_count` → **bullish_setup**
- `avg_sentiment ≤ -0.3` 且 `negative_count > positive_count` → **bearish_setup**
- 其它 → **mixed_or_neutral**

如 `article_count == 0` → 报告里标 "财报前 7 日内无相关新闻"，
**不补 0**，仍生成 brief。

### Step 6 — 生成 1-page brief

把以下 8 段 markdown 写入
`${target_repo}/research/earnings-preview-{TICKER}-YYYY-MM-DD.md`，
**总行数 ≤ 300**：

1. **Frontmatter**：`generated_at` (UTC)、`ticker`、`coverage`
   (watchlist | ad-hoc)、`mcp_versions`（3 个）、`cache_hit_rate`
   （schwab）、`next_earnings_estimated` + 推断置信度、`iv_rank` +
   `iv_recommendation`、`sentiment_class`。
2. **Header**：ticker / 公司名 / 下一财报日推断（来自 Step 2）+
   推断方法说明（"基于最近 3 期 8-K item 2.02 中位数间隔"）+
   置信度。
3. **IV percentile 卡**（来自 Step 3）：current_atm_iv / percentile_rank
   / sample_count / lookback 窗口；附 §3 阈值表的当前命中行 +
   推荐定位。
4. **价格走势 30 天**（来自 Step 4）：30 天最高 / 最低 / 收盘 / ATR(14)
   / 距 52w 高低 / 关键支撑阻力。**不画 ASCII K 线**（保持 ≤ 300 行）。
5. **新闻情绪 7 天**（来自 Step 5）：avg_sentiment / 三类计数 /
   情绪判定；top 5 articles 引用（title + url + publisher +
   published_utc，**不转载正文**）。如 article_count = 0 → 标 "财报前
   7 日内无相关新闻"。
6. **历史 earnings move 统计**（来自 Step 2 + Step 4 交叉）：
   - 从 8-K item 2.02 提取最近 3 期 filing_date
   - 用 price_history 计算每期 filing_date ± 1 trading day 的收盘
     变化（绝对值 + 方向）
   - 输出表：`| 期次 | filing_date | T-1 收盘 | T+1 收盘 | move % |`
   - 计算 3 期 move % 的中位数 → 与当前 IV implied move 对比
     （implied_move ≈ current_atm_iv × sqrt(1/365) × 100%）
7. **推荐定位**（综合 §3 IV rank + §5 sentiment）：从下表选 1 行：

   | IV rank \ Sentiment | bullish_setup | bearish_setup | mixed_or_neutral |
   | --- | --- | --- | --- |
   | < 30 | long_iv_straddle + 偏多倾向 | long_iv_straddle + 偏空倾向 | long_iv_straddle |
   | 30–70 | directional_long_only | directional_short_only | no_action |
   | > 70 | sell_iv_condor + 偏多腿宽 | sell_iv_condor + 偏空腿宽 | sell_iv_condor |
   | null（稀疏） | insufficient_data |  |  |

   **再次强调**：以上为**草案信号**，不是交易指令；用户需自行
   决定方向性观点 + 仓位规模 + 风险预算。

8. **数据出处与限制**：每段一句脚注 →
   - IV rank 数据：schwab `get_iv_percentile`（cache hit / miss）
   - 历史 8-K：sec-edgar `get_8k_with_items`（公开 EDGAR）
   - 新闻情绪：polygon `get_news_sentiment_aggregate`（不可二次分发）
   - 价格：schwab `get_price_history`（不可二次分发）
   - 下一财报日：本地推断，**非** IR 官方公告，置信度 = ...
   - Schwab Market Data 不可再分发声明（保持与 shakeout-with-news 一致）

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 已写入文件
TICKER_LOWER=$(echo <ticker> | tr '[:upper:]' '[:lower:]')
# 文件名实际用大写 TICKER 保持与 SEC ticker 一致
git add research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): earnings-preview <TICKER> $(date -u +%Y-%m-%d)"
# DO NOT push --force.  普通 push 到 research 分支即可。
```

## Acceptance criteria

完成后逐项验证（每项跑命令并确认输出，再勾选）：

- [ ] **Activation handshake 6/6 PASS**：pre-flight 全部 6 步留底
      （聊天上下文）；任一失败必须 STOP 而非降级
- [ ] **IV percentile 数据成功获取**：`get_iv_percentile` 返回非空
      response；`percentile_rank` 为 None 时**必须**带 `sample_count_below_30`
      警告并在报告里标 "low confidence"，不能静默
- [ ] **8-K item 2.02 fallback ≥ 1 条**：`get_8k_with_items` `count ≥ 1`；
      `count == 0` 视为该 ticker 上市未满 1 季度，**STOP** 不写报告
- [ ] **polygon sentiment ≥ 5 articles**：`get_news_sentiment_aggregate`
      返回 `article_count ≥ 5`；< 5 时报告标 "low news coverage" 但**不**STOP
- [ ] **价格走势 30 天数据完整**：`get_price_history` 返回的
      `candles` 数量 ≥ 20（30 trading days 容许 ≤ 10 个 holiday gap）
- [ ] **输出报告 ≤ 300 行**：
      `wc -l ${target_repo}/research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md` ≤ 300；
      超出时**自动截断 §4 价格走势的细节文字**并在底部加 WARN
- [ ] **commit message 以 `research(cross-mcp):` 开头**：
      `git -C ${target_repo} log -1 --format="%s"` 输出匹配
- [ ] **数据出处段每个数字有 MCP tool 脚注**：§8 数据出处段必须列出
      4 个 tool（`get_iv_percentile` / `get_8k_with_items` /
      `get_news_sentiment_aggregate` / `get_price_history`），且每个
      数字（IV rank / sentiment / 8-K count / 30d 高低）至少有 1 处
      脚注引用对应 tool

## Rollback

```bash
cd ${target_repo}
# 已 commit 但还没 push → 用 reset --soft 改 commit 内容后再 push
git reset --soft HEAD~1   # 撤销 commit，保留 working tree
# 检查 working tree 后决定 git restore 或 git stash
git restore research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md

# 已 push 但发现错误 → 用 git revert（保留 audit trail，绝不 force push）
git revert <hash>
git push origin <branch>   # 不 --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `get_iv_percentile` `sample_count < 30` | WARN 不 STOP；`percentile_rank=None`；报告标 "low confidence (sample_count=N)"；推荐定位降级为 `insufficient_data` |
| `get_8k_with_items` `count == 0`（90 天内无 item 2.02） | **STOP**：该 ticker 可能上市 < 1 季度或 SEC 索引滞后；不要凭 polygon 新闻倒推财报日 |
| `get_news_sentiment_aggregate` 全部 `sentiment=neutral` 且 `article_count >= 5` | 不 STOP；§5 标 `mixed_or_neutral`；§7 推荐表的 mixed 列照常给信号 |
| `get_price_history` 401 / `SchwabAuthError(reason="refresh_token_expired")` | **STOP**，提示用户先跑 `uv run python -m schwab_marketdata_mcp.auth login_flow` |
| `sec-edgar-mcp` 403 / `sec_ua_reachable.status == REJECTED_HTML_403` | **STOP**，提示用户检查 `SEC_EDGAR_USER_AGENT`（参考 SKILL.md handshake step 2.5） |
| 网络 timeout（任一 MCP）| 重试 1 次；仍失败 → STOP；不**继续编造数据** |
| Activation handshake 任一步失败 | **STOP**，不重试；提示用户具体哪一步失败 + 修复方法 |
| 报告生成后 `wc -l > 300` | 自动截断 §4 文字段落（保留表格），底部加 `> ⚠️ Report truncated to fit 300-line limit; full data in commit metadata.` |
| ticker 不在 watchlist | **接受**，frontmatter 标 `coverage: ad-hoc`；不 STOP（人工 ad-hoc 查询合法用例） |
| schwab cache hit_rate < 0.3（陈旧 ≥ 24h） | WARN 不 STOP；frontmatter 标 `cache_freshness: stale`；建议下次跑 `refresh=True`（注意 quota） |

## Idempotency

| 重复运行 | 副作用 |
| --- | --- |
| 同 ticker 同日重跑 ≤ 1 次（cache hit_rate ≥ 30% gate） | 写 `research/earnings-preview-{TICKER}-YYYY-MM-DD.md`；同名文件存在时询问是否覆盖（默认 skip） |
| 同 ticker 不同日 | 每日新文件；文件名带 ticker + date，天然隔离 |

## See also

- 同仓 playbooks：
  - `playbooks/shakeout-with-news.md`（shakeout 信号 + 新闻情绪叠加）
  - `playbooks/insider-alert.md`（Form 4 异常告警）
- `stock-personal/docs/STRATEGY.md §1 Q1`：option-chain edge / IV
  rank 在策略 thesis 中的位置。
- `schwab-marketdata-mcp/CHANGELOG.md §0.4.0`：`get_iv_percentile`
  v0.4 P1/C 规格（refresh path / iv_history 物化表 / sample_count
  阈值 30）。
- `sec-edgar-mcp` README §`get_8k_with_items`：item code 索引参考
  （1.01 / 2.02 / 5.02 / 7.01 / 9.01 等）。
- `stock-personal/docs/sprints/v0.5-roadmap.md §2 V5-C`：本 playbook
  的 sprint 验收标准来源。
