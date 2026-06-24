# Playbook — Factor screen deep-dive（全市场因子筛选 + 多源深度研究）

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/factor-screen-YYYY-MM-DD.md`（新建） |
| clickhouse tools used | `health_check`, `screen_stocks`, `get_indicators`（可选复核） |
| schwab tools used | `health_check`, `get_quote` |
| sec-edgar tools used | `health_check`, `get_institutional_holders`（13F 机构持仓反查） |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 30（clickhouse ≤ 2 + 候选数 N≤6 × 3 源 + 4 health） |
| 数据合规 | ClickHouse 历史指标为派生计算；SEC 13F 公开；polygon/schwab 不可二次分发 |
| 用例 | 用 14.9 亿行历史做全市场横截面因子扫描选出 N 个候选，再逐一多源深度研究 |
| 触发关键词 | "因子筛选" / "全市场扫描" / "factor screen" / "选股 + 深度研究" / "横截面因子" |

> **本 playbook 仅在 stock-personal 仓库内运行**。如果 cwd 不在
> `${target_repo}` 子树下，必须切换为只读模式：分析输出到聊天上下文，
> **禁止落盘到任何其他仓库**。
>
> **clickhouse-mcp 是本 playbook 的核心编排对象**，但**非硬依赖**：CH 不可用
> 时本 playbook **降级**——跳过全市场扫描（Step 2），改由用户**手动提供候选
> ticker 列表**，仍可跑 Step 3-6 的多源深度研究（schwab + sec-edgar + polygon）。
> 降级时报告 frontmatter 标 `clickhouse: unavailable`，并在 TL;DR 注明
> "全市场因子扫描跳过（需配 CH 只读账户）"。

## Pre-flight（mandatory）

```text
1. clickhouse-mcp.health_check()          → overall_status == "ok" 且
                                            connection_configured == true 且
                                            clickhouse_reachable == true 且
                                            read_only == true 且
                                            server_version ∈ ">=0.1,<0.2"
   ↳ 若 overall_status == "unhealthy"（CH 未配只读账户或不可达）→ **降级**：
     不 STOP；标记 clickhouse_unavailable=true，跳过 Step 2，进入"手动候选"分支。
2. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" 且
                                            server_version ∈ ">=0.4,<0.5"
3. sec-edgar-mcp.health_check()           → user_agent_configured == true 且
                                            server_version ∈ ">=0.4,<0.5"
                                            （需 v0.4.0+ 才有 get_institutional_holders）
3.5. 验证 sec-edgar 服务端 UA 真实可达性（与 SKILL.md §Activation handshake
     step 2.5 完全一致）：读 health_check 返回的 sec_ua_reachable.status：
   - ACCEPTED       → ✅ 继续
   - REJECTED_HTML_403 → ❌ STOP，让用户改 SEC_EDGAR_USER_AGENT 为真实邮箱
   - UNCONFIGURED   → ❌ STOP，让用户配置 SEC_EDGAR_USER_AGENT
   - TIMEOUT / NETWORK_ERROR → ⚠️ WARN，继续但报告标 "SEC 探针不可用"
4. polygon-news-mcp.health_check()        → api_key_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
5. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
6. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true（else STOP；不要绕过）
7. cwd ∈ ${target_repo} subtree？若否 → 只读模式（仅输出到聊天）
```

> **clickhouse-mcp 只读账户**：clickhouse-mcp 连接需配 `CLICKHOUSE_MCP_HOST` /
> `CLICKHOUSE_MCP_USER`（建议指向 USA 侧 `readonly=1` + `GRANT SELECT` 专用账户）/
> `CLICKHOUSE_MCP_PASSWORD`。凭证只从 env 读，**从不写日志/repr**。未配时
> `health_check()` 返回 `connection_configured == false`，本 playbook 走降级分支。

## Steps

### Step 1 — 选取 universe 与因子条件

- 默认 universe：CH 已沉淀的 **1388 symbols**（L0/L1 全覆盖）。用户可缩到子集
  （如 sp500 / ndx100，由 watchlist 或用户指定 ticker 列表近似）。
- 默认因子条件（用户可改）：从下表选 1-3 个组合（`screen_stocks` 上限 10 个 filter）：

  | 因子主题 | indicator（CH 允许集） | operator | 示例阈值 | 含义 |
  | --- | --- | --- | --- | --- |
  | 超卖反转 | `rsi14` | `lt` | 30 | RSI 超卖榜 |
  | 动量 | `macd_hist` | `gt` | 0 | MACD 柱转正（动量回暖） |
  | 趋势确认 | `adx14` | `gt` | 25 | 强趋势（ADX>25） |
  | 均线多头 | `ma20` / `ma60` | — | 本地比对 | ma20>ma60 多头排列（拉两个指标本地比） |
  | 量能 | `obv` / `vwap` | — | 本地比对 | 价在 VWAP 上方 |

> indicator 名必须落在 clickhouse-mcp 的 `ALLOWED_INDICATORS` 白名单内
> （`ma5/ma10/ma20/ma50/ma60/ma120/ma200/ma250/ema12/ema26/macd_dif/macd_dea/`
> `macd_hist/atr14/boll_mid/boll_up/boll_low/rsi14/stoch_rsi14/mfi14/kdj_k/`
> `kdj_d/kdj_j/adx14/obv/vwap`）。不在白名单的指标会被 server 拒绝；
> **不要凭记忆猜指标名**，必要时先用 `get_server_info()` 校对。

### Step 2 — clickhouse-mcp 全市场横截面扫描

```text
clickhouse-mcp.screen_stocks(
    filters=[
        {"indicator": "rsi14",     "operator": "lt", "value": 30},
        {"indicator": "macd_hist", "operator": "gt", "value": 0},
    ],
    as_of=None,           # None = CH 视图中该 freq 最近一个交易日；可传 YYYY-MM-DD
    frequency="1d",       # 1m/5m/15m/1h/1d/1w；横截面因子默认 1d
    limit=100,            # 返回上限（1..2000）
)
```

预期返回字段：`frequency`、`as_of`、`filters[]`、`indicators[]`、`count`、
`matches[]`（每行 `symbol` + 命中的 `ind_*` 指标值）。

**本地收敛候选**（不再调 CH）：

- 按因子组合规则从 `matches` 选出 **top N（默认 N=5，上限 6**，受 max tool calls 约束）。
- 可选复核：对 top N 各调一次 `get_indicators(symbol, indicator=..., frequency="1d",
  start=<as_of-60d>, end=<as_of>)` 看指标近 60 日轨迹，剔除"单日毛刺"假信号。
- 记录每个候选的 `as_of` 与命中指标值，写入报告 §3。

> **降级分支（clickhouse_unavailable=true）**：跳过本 Step。改为：要求用户
> 提供候选 ticker 列表（≤ 6 个），报告标注"全市场扫描跳过；候选来自人工指定"。
> Step 3-6 照常跑。

### Step 3 — 候选实时报价（schwab）

仅对 Step 2 选出的 top N 候选，逐一调：

```text
schwab-marketdata-mcp.get_quote(
    symbol=<candidate>,
    fields=["QUOTE", "REGULAR"],
)
```

记录字段：`lastPrice`、`netChangeInDouble`、`netPercentChangeInDouble`、
`totalVolume`、`52WeekHigh`、`52WeekLow`、（如可得）`30DayAverageVolume`。
计算"当前价 vs CH 扫描时收盘"差异，标注是否已脱离扫描时点。

> 工具签名以 `get_server_info()` 实际暴露为准（部分版本暴露的是批量
> `get_quotes(symbols=[...])`）；按 server 实际签名调，不要凭记忆猜。

### Step 4 — 候选 13F 机构持仓（sec-edgar）

仅对 top N 候选，逐一调（反查哪些 13F 机构报告了持仓）：

```text
sec-edgar-mcp.get_institutional_holders(
    ticker=<candidate>,
    since_days=120,        # 覆盖最近 1 个 13F 申报季（季度 + 45 天申报窗口）
)
```

记录字段：`ticker`、`as_of_quarter`、`holder_count`、`total_shares_reported`、
`top_holders[]`（每条含 `filer_name` / `shares` / `value_usd` / `accession_number`）。
本地判定"机构关注度"：`holder_count` 高 + 头部机构集中 → 机构背书强。

> 13F 有 45 天申报延迟（季度结束后）；报告须标注"机构持仓为滞后数据，非实时"。
> 若该 ticker 无 13F 记录（小盘/新上市）→ 标 `13f: none`，不补 0、不前向填充。

### Step 5 — 候选新闻情绪（polygon）

仅对 top N 候选，逐一调：

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<candidate>,
    window="7d",           # 最近 7 天滚动情绪
)
```

记录字段：`avg_sentiment ∈ [-1, 1]`、`positive_count`、`negative_count`、
`neutral_count`、`article_count`、`window_start`、`window_end`、
`top_articles[]`（如可得：title / url / publisher / published_utc）。
情绪判定：`avg_sentiment ≥ +0.3` → bullish；`≤ -0.3` → bearish；其它 neutral。
**不转载新闻正文**（版权安全）。

### Step 6 — 生成候选清单 + 多源研究 brief

把以下 8 段 markdown 写入 `${target_repo}/research/factor-screen-YYYY-MM-DD.md`：

1. **Frontmatter**：`generated_at` (UTC)、`universe`、`factor_filters`、`as_of`、
   `candidate_count`、`mcp_versions`（4 个：clickhouse/schwab/sec-edgar/polygon）、
   `clickhouse`（ok | unavailable）。
2. **TL;DR**：单行结论，例如
   "全市场 RSI<30 ∩ MACD柱>0 扫出 5 个候选；AAPL 机构背书最强 + 情绪 +0.41"。
3. **因子扫描结果表**（来自 Step 2）：每行候选 + 命中指标值 + `as_of`；
   降级时标"扫描跳过，候选人工指定"。
4. **每个候选的多源研究子块**（top N，每个含 4 子段）：
   - schwab 实时报价（来自 Step 3）：last / 涨跌 / 量 / 距 52w 高低
   - sec-edgar 13F 机构持仓（来自 Step 4）：holder_count / top 3 机构 + accession
   - polygon 新闻情绪（来自 Step 5）：avg_sentiment / 三类计数 / 情绪判定
   - **综合研究结论**：因子信号 × 机构 × 情绪 的一行定性判断（草案信号）
5. **候选横向对比表**：N 行 × 列（因子分 / 机构关注 / 情绪 / 综合），按综合排序。
6. **风险提示**：① CH 指标可能滞后（USA L2 增量未自动触发）；② 13F 45 天延迟；
   ③ 因子信号是横截面相对，不构成择时；列举 1-2 个未消化风险点。
7. **后续动作建议**：每个候选给**通用**动作（加入 watchlist / 深研财报 /
   监控频率），**不构成投资建议**。
8. **数据出处与限制**：链接 clickhouse-mcp（CH 派生指标）、SEC EDGAR 13F、
   polygon API tier、Schwab Market Data 不可二次分发声明。

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 已写入文件
git add research/factor-screen-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): factor-screen-deep-dive $(date -u +%Y-%m-%d)"
# DO NOT push --force.  普通 push 到 research 分支即可。
```

## Acceptance criteria

完成后逐项验证（每项跑命令并确认输出，再勾选）：

- [ ] **Activation handshake 7 步留底**：pre-flight 输出在聊天上下文；
      clickhouse 不可用时必须走降级分支（标 `clickhouse: unavailable`）而非 STOP
- [ ] **commit 已创建**：`git -C ${target_repo} log -1 --format="%H %s"`
      输出最新 commit hash + `research(cross-mcp):` 前缀的 message
- [ ] **research/ 下有当日新文件**：
      `ls ${target_repo}/research/factor-screen-$(date +%Y-%m-%d).md`
- [ ] **仅 research/ 被改动**：
      `git -C ${target_repo} diff --stat HEAD~1` 列出的所有文件路径都在
      `research/` 目录下
- [ ] **报告含全部 8 段**：
      `grep -c '^##' ${target_repo}/research/factor-screen-$(date +%Y-%m-%d).md` ≥ 8
- [ ] **每个候选含 4 子段**：每个 top N 候选必须含 schwab 报价 / 13F 持仓 /
      新闻情绪 / 综合结论四块
- [ ] **CH 数据出处含 as_of**：报告 §3 / §8 必须标注扫描 `as_of` 日期，
      避免误导用户以为是实时全市场快照
- [ ] **降级路径可验证**：若 clickhouse_unavailable，§3 必须明确标注
      "全市场扫描跳过，候选人工指定"，且 Step 3-6 仍完整产出

## Rollback

```bash
cd ${target_repo}
# 已 commit 但还没 push → 用 reset --soft 改 commit 内容后再 push
git reset --soft HEAD~1   # 撤销 commit，保留 working tree
git restore research/factor-screen-$(date +%Y-%m-%d).md

# 已 push 但发现错误 → 用 git revert（保留 audit trail，绝不 force push）
git revert <hash>
git push origin <branch>   # 不 --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `clickhouse-mcp.health_check()` `overall_status == "unhealthy"`（CH 未配/不可达） | **降级不 STOP**：标 `clickhouse: unavailable`，跳过 Step 2，要求用户提供候选 ticker 列表；Step 3-6 照常 |
| `clickhouse-mcp` `connection_configured == false`（缺只读账户 env） | 同上降级；提示用户配 `CLICKHOUSE_MCP_HOST/_USER/_PASSWORD`（建议 `readonly=1` 专用账户） |
| `screen_stocks` 大查询超时（CH `max_execution_time` 触发，返回 query failed） | 缩小 universe（用 watchlist 子集）或收紧 filter 阈值重试 1 次；仍超时 → 降级走人工候选 |
| `screen_stocks` 返回 `count == 0`（无标的命中因子） | 不 STOP；放宽阈值重试 1 次；仍 0 → 报告标"本期无因子命中"，TL;DR 注明 |
| indicator 名不在 `ALLOWED_INDICATORS` 白名单（server 拒绝） | 用 `get_server_info()` 校对白名单，换合法 indicator；**不要猜指标名** |
| `sec-edgar-mcp` 返回 429（SEC fair-use 限流） | 等 1s 重试 1 次；连续两次 STOP；提示用户 sec-edgar UA 是否合规 |
| `sec-edgar-mcp` 403 / `sec_ua_reachable.status == REJECTED_HTML_403` | **STOP**，提示用户检查 `SEC_EDGAR_USER_AGENT`（参考 SKILL.md handshake step 2.5） |
| 候选无 13F 记录（小盘/新上市） | 标 `13f: none`；不补 0、不前向填充；综合结论降级为"机构数据缺失" |
| `polygon-news-mcp` 返回 401/403 | STOP，要求用户检查 `POLYGON_API_KEY` |
| `polygon-news-mcp` 返回 429 | 等 60s 重试 1 次；继续失败则 STOP |
| `get_news_sentiment_aggregate` `article_count == 0` | 报告标"7 日内无相关新闻"；不补 0；综合结论按 neutral 处理 |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP，要求用户先跑 `auth login_flow` |
| 任一 token / 凭证过期（schwab / polygon / sec-edgar / CH） | STOP 报告具体哪一源失败 + 修复方法；**不继续编造数据** |
| `gh repo view` 失败 / 仓库不是 private | **STOP and refuse to write**；不绕过 |
| 同一日已存在 `research/factor-screen-YYYY-MM-DD.md` | 询问用户是否覆盖；默认 skip 并告知 |

## Idempotency

| 重复运行 | 副作用 |
| --- | --- |
| 同日重跑 ≤ 1 次 | 写 `research/factor-screen-YYYY-MM-DD.md`，每日新文件；同名存在时询问覆盖（默认 skip） |
| 不同日 | 每日新文件；文件名带 date，天然隔离 |
| CH 不可用降级跑 | 仍写当日文件（候选人工指定）；frontmatter 标 `clickhouse: unavailable` 区分 |

## See also

- 同仓 playbooks：
  - `playbooks/correlation-pairs-monitor.md`（相关性配对监控，同样编排 CH + MCP）
  - `playbooks/shakeout-with-news.md`（shakeout 信号 + 新闻情绪）
  - `playbooks/earnings-preview.md`（财报前 IV-rank-aware 定位简报）
- clickhouse-mcp：[kevinkda/clickhouse-mcp](https://github.com/kevinkda/clickhouse-mcp)
  （7 只读工具，14.9 亿行历史 + L2 物化指标）
- `stock-personal/docs/sprints/usa-clickhouse-quant-integration-plan.md §3`：
  本 playbook 的量化用例来源（全市场扫描 / 横截面因子 P0）。
