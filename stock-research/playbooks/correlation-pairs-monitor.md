# Playbook — Correlation pairs monitor（相关性配对监控）

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/correlation-pairs-YYYY-MM-DD.md`（新建） |
| clickhouse tools used | `health_check`, `get_correlation_matrix`, `get_ohlcv`（价差计算） |
| schwab tools used | `health_check`, `get_quote` |
| sec-edgar tools used | （本 playbook 不调用 sec-edgar） |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 24（clickhouse ≤ 4 + 高相关对 P≤5 × 2 源 + 3 health） |
| 数据合规 | ClickHouse 历史相关性为派生计算；polygon/schwab 不可二次分发 |
| 用例 | 用 watchlist 多标的历史相关性矩阵找高相关对 → 实时价差 → 配对交易候选 |
| 触发关键词 | "相关性配对" / "配对交易" / "correlation pairs" / "pairs trading" / "相关性矩阵" |

> **本 playbook 仅在 stock-personal 仓库内运行**。如果 cwd 不在
> `${target_repo}` 子树下，必须切换为只读模式：分析输出到聊天上下文，
> **禁止落盘到任何其他仓库**。
>
> **clickhouse-mcp 是本 playbook 的核心编排对象**，且**为硬性前置**：相关性
> 矩阵依赖 CH 的多标的历史 OHLCV（MCP 单 symbol 逐个拉无法批量算）。CH 不可
> 用时本 playbook **降级为只读建议**：不落盘相关性结果，仅输出
> "需配 CH 只读账户后重跑"，并可选地用 schwab 实时报价给出**当前价位快照**
> （无历史相关性，不构成配对信号）。降级时 frontmatter 标 `clickhouse: unavailable`。

## Pre-flight（mandatory）

```text
1. clickhouse-mcp.health_check()          → overall_status == "ok" 且
                                            connection_configured == true 且
                                            clickhouse_reachable == true 且
                                            read_only == true 且
                                            server_version ∈ ">=0.1,<0.2"
   ↳ 若 overall_status == "unhealthy"（CH 未配只读账户或不可达）→ **降级**：
     不 STOP；标记 clickhouse_unavailable=true，跳过 Step 2-3，
     进入"只读建议"分支（仅输出当前价位快照 + 提示配 CH）。
2. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" 且
                                            server_version ∈ ">=0.4,<0.5"
3. polygon-news-mcp.health_check()        → api_key_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true（else STOP；不要绕过）
6. cwd ∈ ${target_repo} subtree？若否 → 只读模式（仅输出到聊天）
```

> **clickhouse-mcp 只读账户**：连接需配 `CLICKHOUSE_MCP_HOST` /
> `CLICKHOUSE_MCP_USER`（建议指向 USA 侧 `readonly=1` + `GRANT SELECT` 专用账户）/
> `CLICKHOUSE_MCP_PASSWORD`。凭证只从 env 读，**从不写日志/repr**。未配时
> `health_check()` 返回 `connection_configured == false`，本 playbook 走降级分支。
>
> **本 playbook 不调用 sec-edgar**（配对交易看的是价差/相关性 + 新闻背离，
> 不直接用基本面 filings），故 handshake 只跑 3 个 health_check。

## Steps

### Step 1 — 解析 watchlist 与配对池

- 读取 watchlist：优先 `${target_repo}/portfolio/watchlist.md`，回退
  `${target_repo}/watchlist.md` 或 `${target_repo}/trackers/watchlist.md`。
- 提取 ticker 列表（去重、去空白）。`get_correlation_matrix` 要求
  **2 ≤ 标的数 ≤ 50**；超 50 时取前 50 并在报告标注"截断 N 个"。
- 用户可显式传一组标的（如同行业 ETF / 同板块龙头）替代 watchlist。

### Step 2 — clickhouse-mcp 历史相关性矩阵

```text
clickhouse-mcp.get_correlation_matrix(
    symbols=["XLE", "XOM", "CVX", "COP", "SLB"],   # 2..50 个，去重
    start="2025-06-01",     # 默认回看 ≈ 252 个交易日（1 年）
    end="2026-06-01",
    frequency="1d",         # 配对相关默认日线
    method="pearson",       # pearson | spearman（spearman 更抗异常值）
)
```

预期返回字段：`symbols[]`、`frequency`、`method`、`start`、`end`、
`matrix`（嵌套 dict：`matrix[a][b]` = a,b 的相关系数 ∈ [-1,1] | null）。
**注意**：相关性基于**日简单收益率**（不是价格电平），对齐两标的共同交易日。

**本地选高相关对**（不再调 CH）：

- 取 `matrix` 上三角（去对角线 1.0 与对称重复），按 |corr| 降序。
- 选 **|corr| ≥ 0.8** 的对作为配对候选（用户可改阈值）；上限 **P=5 对**
  （受 max tool calls 约束）。
- 标注每对的相关方向（正相关 = 同向，可做价差均值回归；强负相关 = 对冲候选）。

> **降级分支（clickhouse_unavailable=true）**：跳过本 Step 与 Step 3。
> 报告仅输出"需配 CH 只读账户后重跑相关性矩阵"，可选地用 schwab 报价给
> watchlist 当前价位快照（明确标注"无历史相关性，非配对信号"）。

### Step 3 — 配对价差历史（clickhouse-mcp get_ohlcv）

对 Step 2 选出的高相关对，逐对拉两腿历史收盘算价差 Z-score：

```text
clickhouse-mcp.get_ohlcv(
    symbol=<leg_A>,
    start="2025-06-01",
    end="2026-06-01",
    frequency="1d",
    limit=300,
)
# 对 leg_B 同样调一次
```

预期返回字段：`symbol`、`frequency`、`start`、`end`、`table`、`count`、
`bars[]`（每条 `ts` / `open` / `high` / `low` / `close` / `volume`）。

**本地计算价差统计**（不再调 CH）：

- 对齐两腿共同交易日的 `close`，算价差 `spread = close_A - β·close_B`
  （β 用 OLS 斜率或简单 ratio，本地估）。
- 计算 spread 的均值 / 标准差 → 当前 **Z-score = (spread_now - mean) / std**。
- 估半衰期（spread 的 AR(1) 系数 → `halflife = -ln2 / ln(φ)`），衡量回归速度。
- |Z| ≥ 2 标记"价差偏离"（潜在配对入场区）；|Z| < 1 标记"价差收敛"。

### Step 4 — 配对实时价差（schwab）

仅对 Step 2 高相关对的两腿，逐一调实时报价校准当前价差：

```text
schwab-marketdata-mcp.get_quote(
    symbol=<leg>,
    fields=["QUOTE", "REGULAR"],
)
```

记录 `lastPrice` / `netPercentChangeInDouble`，用实时价重算当前 spread 与 Z-score
（覆盖 Step 3 用 CH 历史末日收盘算的旧 Z），标注"实时 Z vs 历史末日 Z"差异。

> 工具签名以 `get_server_info()` 实际暴露为准；按 server 实际签名调，不要猜。

### Step 5 — 新闻背离验证（polygon）

仅对 Step 2 高相关对，对**两腿各**调一次新闻情绪，验证是否有基本面背离
（高相关对突然价差走阔，常因一腿出了独立利好/利空）：

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<leg>,
    window="7d",
)
```

记录 `avg_sentiment` / `article_count` / `top_articles[]`（如可得）。
**背离判定**：两腿情绪差 `|sent_A - sent_B| ≥ 0.4` → "基本面背离"（价差走阔可能
有真实驱动，**慎做均值回归**）；情绪接近 → "无明显背离"（价差偏离更可能是
技术性，均值回归逻辑更成立）。**不转载新闻正文**。

### Step 6 — 生成配对监控报告

把以下 8 段 markdown 写入 `${target_repo}/research/correlation-pairs-YYYY-MM-DD.md`：

1. **Frontmatter**：`generated_at` (UTC)、`symbols`（输入池）、`window`
   (start..end)、`method`、`corr_threshold`、`pair_count`、`mcp_versions`
   （3 个：clickhouse/schwab/polygon）、`clickhouse`（ok | unavailable）。
2. **TL;DR**：单行结论，例如
   "XLE-XOM 相关 0.93、当前 Z +2.3 偏离 + 无新闻背离 → 均值回归候选"。
3. **相关性矩阵摘要**（来自 Step 2）：上三角 |corr| 降序 top 表（pair / corr /
   方向）；降级时标"矩阵跳过，需配 CH"。
4. **每个高相关对的明细块**（top P，每个含 4 子段）：
   - 历史价差统计（来自 Step 3）：β / mean / std / 历史末日 Z / 半衰期
   - 实时价差（来自 Step 4）：两腿 last / 实时 Z / vs 历史 Z 差异
   - 新闻背离（来自 Step 5）：两腿 avg_sentiment / 背离判定
   - **配对结论**：相关 × Z 偏离 × 背离 的一行定性判断（草案信号）
5. **配对候选排序表**：P 行 × 列（corr / 当前 Z / 半衰期 / 背离 / 综合），
   按 |Z| × 无背离 排序。
6. **风险提示**：① 相关性是历史的，未来可能 regime 切换断裂；② 配对交易需做
   多空两腿，本 playbook **不下单**（仅信号草案）；③ 半衰期长 = 资金占用久；
   列举 1-2 个未消化风险。
7. **后续动作建议**：每对给**通用**动作（加入配对监控 / 等 Z 回归 / 关注背离
   新闻），**不构成投资建议**。
8. **数据出处与限制**：链接 clickhouse-mcp（CH 历史相关性/价差）、polygon
   API tier、Schwab Market Data 不可二次分发声明。

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 已写入文件
git add research/correlation-pairs-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): correlation-pairs-monitor $(date -u +%Y-%m-%d)"
# DO NOT push --force.  普通 push 到 research 分支即可。
```

## Acceptance criteria

完成后逐项验证（每项跑命令并确认输出，再勾选）：

- [ ] **Activation handshake 6 步留底**：pre-flight 输出在聊天上下文；
      clickhouse 不可用时必须走降级分支（标 `clickhouse: unavailable`）而非 STOP
- [ ] **commit 已创建**：`git -C ${target_repo} log -1 --format="%H %s"`
      输出最新 commit hash + `research(cross-mcp):` 前缀的 message
- [ ] **research/ 下有当日新文件**：
      `ls ${target_repo}/research/correlation-pairs-$(date +%Y-%m-%d).md`
- [ ] **仅 research/ 被改动**：
      `git -C ${target_repo} diff --stat HEAD~1` 列出的所有文件路径都在
      `research/` 目录下
- [ ] **报告含全部 8 段**：
      `grep -c '^##' ${target_repo}/research/correlation-pairs-$(date +%Y-%m-%d).md` ≥ 8
- [ ] **每个高相关对含 4 子段**：每对必须含 历史价差 / 实时价差 / 新闻背离 /
      配对结论四块
- [ ] **相关性窗口标注完整**：报告 §3 / §8 必须标注 `start..end` 窗口 + `method`，
      避免误导用户以为是"任意时点"的相关性
- [ ] **配对结论标注"非下单指令"**：§4 / §6 必须明确"信号草案，不构成下单"，
      且本 playbook 全程零下单 surface

## Rollback

```bash
cd ${target_repo}
# 已 commit 但还没 push → 用 reset --soft 改 commit 内容后再 push
git reset --soft HEAD~1   # 撤销 commit，保留 working tree
git restore research/correlation-pairs-$(date +%Y-%m-%d).md

# 已 push 但发现错误 → 用 git revert（保留 audit trail，绝不 force push）
git revert <hash>
git push origin <branch>   # 不 --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `clickhouse-mcp.health_check()` `overall_status == "unhealthy"`（CH 未配/不可达） | **降级不 STOP**：标 `clickhouse: unavailable`，跳过 Step 2-3，仅输出"需配 CH 只读账户后重跑"+ 可选 schwab 当前价位快照（非配对信号） |
| `clickhouse-mcp` `connection_configured == false`（缺只读账户 env） | 同上降级；提示用户配 `CLICKHOUSE_MCP_HOST/_USER/_PASSWORD`（建议 `readonly=1` 专用账户） |
| `get_correlation_matrix` 标的数 < 2 或 > 50（校验失败） | 调整池大小到 [2,50]；> 50 取前 50 并标"截断"；< 2 询问用户补标的 |
| `get_correlation_matrix` / `get_ohlcv` 大查询超时（CH `max_execution_time`） | 缩短回看窗口（如 252→120 交易日）或减少标的数重试 1 次；仍超时 → 降级 |
| `matrix[a][b] == null`（两腿共同交易日不足，无法算相关） | 该对标 `corr: null`；不补 0；从配对候选剔除并在报告注明"数据不足" |
| 无任何对相关系数绝对值 ≥ 阈值 | 不 STOP；降低阈值（如 0.8→0.7）重试 1 次；仍无 → 报告标"本期无高相关对" |
| `get_ohlcv` `count == 0`（某腿无历史 bars） | 该对剔除；标"leg 历史缺失"；不前向填充 |
| `polygon-news-mcp` 返回 401/403 | STOP，要求用户检查 `POLYGON_API_KEY` |
| `polygon-news-mcp` 返回 429 | 等 60s 重试 1 次；继续失败则 STOP |
| `get_news_sentiment_aggregate` 某腿 `article_count == 0` | 该腿标"7 日内无新闻"；背离判定按"无背离"保守处理 |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP，要求用户先跑 `auth login_flow` |
| 任一 token / 凭证过期（schwab / polygon / CH） | STOP 报告具体哪一源失败 + 修复方法；**不继续编造数据** |
| `gh repo view` 失败 / 仓库不是 private | **STOP and refuse to write**；不绕过 |
| 同一日已存在 `research/correlation-pairs-YYYY-MM-DD.md` | 询问用户是否覆盖；默认 skip 并告知 |

## Idempotency

| 重复运行 | 副作用 |
| --- | --- |
| 同日重跑 ≤ 1 次 | 写 `research/correlation-pairs-YYYY-MM-DD.md`，每日新文件；同名存在时询问覆盖（默认 skip） |
| 不同日 | 每日新文件；文件名带 date，天然隔离 |
| CH 不可用降级跑 | 仅当用户接受"无相关性矩阵"时写当日文件；frontmatter 标 `clickhouse: unavailable`；否则不落盘 |

## See also

- 同仓 playbooks：
  - `playbooks/factor-screen-deep-dive.md`（全市场因子筛选 + 多源深度研究）
  - `playbooks/shakeout-with-news.md`（shakeout 信号 + 新闻情绪）
  - `playbooks/earnings-preview.md`（财报前 IV-rank-aware 定位简报）
- clickhouse-mcp：[kevinkda/clickhouse-mcp](https://github.com/kevinkda/clickhouse-mcp)
  （7 只读工具；`get_correlation_matrix` 在 Python 算 Pearson/Spearman）
- `stock-personal/docs/sprints/usa-clickhouse-quant-integration-plan.md §3`：
  本 playbook 的量化用例来源（多标的相关性矩阵 P0 / 配对交易筛选 P1）。
