# Playbook — Insider alert（内部人交易告警扫描）

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/insider-alert-YYYY-MM-DD.md`（新建） |
| schwab tools used | `health_check`, `get_quotes` |
| sec-edgar tools used | `health_check`, `get_form4_filings`（或当前 server 暴露的等价 form4 / insider 工具） |
| polygon tools used | `health_check`, `ticker_news` |
| max tool calls | ≤ 20（sec-edgar ≤ 10 + schwab ≤ 5 + polygon ≤ 5） |
| 数据时效窗口 | 默认最近 14 天 Form 4 申报；用户可调到 7d / 30d |
| 数据合规 | 仅使用 SEC EDGAR 公开 Form 4；polygon 与 schwab 数据均不可二次分发 |

> **本 playbook 仅在 stock-personal 仓库内运行**。如果 cwd 不在
> `${target_repo}` 子树下，必须切换为只读模式：分析输出到聊天上下文，
> **禁止落盘到任何其他仓库**。

## Pre-flight（mandatory）

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" 且
                                            server_version ∈ ">=0.3,<0.4"
2. sec-edgar-mcp.health_check()           → user_agent_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
3. polygon-news-mcp.health_check()        → api_key_configured == true 且
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true（else STOP；不要绕过）
6. cwd ∈ ${target_repo} subtree？若否 → 只读模式（仅输出到聊天）
7. 确认存在当前 watchlist 来源（默认读 `${target_repo}/watchlist.md` 或
   `${target_repo}/trackers/watchlist.md`）；找不到 → 询问用户提供 ticker 列表
```

## Steps

### Step 1 — 解析 watchlist

读取并解析 watchlist：

- 优先 `${target_repo}/watchlist.md`，回退 `${target_repo}/trackers/watchlist.md`
- 提取 ticker 列表（去重、去空白、上限 25 个）
- 若 watchlist > 25：仅取前 25 个，并在报告里标注"截断 N 个"

### Step 2 — 拉取每个 ticker 的近期 Form 4

对每个 ticker 调：

```text
sec-edgar-mcp.get_form4_filings(
    ticker=...,
    days_back=14,           # 用户可改 7 / 30
    transaction_codes=["P", "S", "A", "M"],
    # P = open-market purchase
    # S = open-market sale
    # A = grant / award
    # M = option exercise
)
```

预期字段：`accession_number`、`filer_name`、`relationship`、
`transaction_date`、`transaction_code`、`transaction_shares`、
`transaction_price_per_share`、`shares_owned_after`、`is_director`、
`is_officer`、`is_ten_percent_owner`、`xbrl_url`。

> 若 sec-edgar-mcp 暴露的工具名不同（例如 `list_form4`、
> `query_insider_transactions`），调用前用 `health_check()` /
> `get_server_info()` 返回的 tool list 校对一次工具名；以**当前 server
> 实际暴露**为准，**不要凭记忆猜工具名**。

### Step 3 — 计算"异常"打分

对每个 ticker 在本地计算（不再触发 SEC 调用）：

```text
异常打分 = w1 * cluster_score
        + w2 * size_score
        + w3 * direction_score
        + w4 * insider_rank_score

其中：
- cluster_score：14 天内 ≥ 3 位独立内部人同向交易 → 1.0；否则按比例
- size_score：单笔交易金额 / 该公司过去 1 年 Form 4 中位数
- direction_score：净买入（P - S）的方向分（买入 +1 / 卖出 -1 / 混合 0）
- insider_rank_score：CEO/CFO/Chairman = 1.0；其他高管 0.7；
                     director 0.5；10% holder 0.3
- 权重 w1..w4 默认 (0.35, 0.25, 0.25, 0.15)，用户可改
```

异常打分 ≥ **0.6** 触发告警；< 0.6 仅纳入摘要表，不展开。

### Step 4 — 触发标的的当日 quote 交叉验证

仅对 Step 3 触发告警的 ticker：

```text
schwab-marketdata-mcp.get_quotes(
    symbols=[<triggered_tickers>],
    fields=["QUOTE", "REGULAR"],
)
```

记录字段：`lastPrice`、`netChangeInDouble`、`totalVolume`、`52WeekHigh/Low`。
计算"insider price vs current price"差值（隐含浮亏/浮盈）。

### Step 5 — 拉取触发标的的近期新闻

仅对触发告警的 ticker：

```text
polygon-news-mcp.ticker_news(
    ticker=...,
    limit=5,
    order="desc",
    sort="published_utc",
    published_utc_gte=<14d ago>,
)
```

记录字段同 shakeout-with-news Step 4：`title`、`published_utc`、
`publisher.name`、`article_url`、`insights[].sentiment`、
`insights[].sentiment_reasoning`。

### Step 6 — 交叉判定

对每个触发标的输出一行**判定结论**，落入 4 类之一：

- **Class A**：内部人净买入 + 价格已跌破近期支撑 + 新闻面有未消化负面 →
  "可能见底"信号
- **Class B**：内部人净买入 + 价格创新高 + 新闻面利好 →
  "管理层背书趋势"信号
- **Class C**：内部人净卖出 + 价格创新高 + 新闻面利好 →
  "管理层止盈"信号（**不一定看空**，但需关注规模）
- **Class D**：内部人净卖出 + 价格已下跌 + 新闻面负面 →
  "高风险派发"信号

### Step 7 — 生成告警报告

把以下 8 段 markdown 写入
`${target_repo}/research/insider-alert-YYYY-MM-DD.md`：

1. **Frontmatter**：`generated_at` (UTC), `watchlist_size`, `triggered_count`,
   `mcp_versions`（3 个）、`window_days`（默认 14）。
2. **TL;DR**：单行结论，例如
   "本周扫到 3 个内部人异常告警：AAPL Class B / GOOG Class C / TSLA Class A"。
3. **Watchlist 全表**：每行一个 ticker + 异常打分 + 是否触发；按打分降序。
4. **每个触发标的的明细块**：
   - Form 4 交易明细表（来自 Step 2）
   - 异常打分拆解（来自 Step 3）
   - schwab quote 与 insider price 对比（来自 Step 4）
   - 触发哪一 class（来自 Step 6）
5. **Top 5 新闻引用**（每触发标的，仅 title+url+publisher+发布时间）。
6. **限制声明**：14 天窗口仅覆盖 Form 4 中的"已申报"交易；
   Form 4 申报有 2 个工作日的延迟；本扫描**不替代**Section 16 合规审计。
7. **后续动作建议**：每 class 给一个**通用**动作清单（HOLD / 减仓 /
   增加监控频率），**不构成投资建议**。
8. **数据出处与限制**：链接 SEC EDGAR、polygon API tier、Schwab 不可二
   次分发声明。

### Step 8 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 7 已写入文件
git add research/insider-alert-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): insider-alert $(date -u +%Y-%m-%d)"
# DO NOT push --force.  普通 push 到 research 分支即可。
```

## Acceptance criteria

完成后逐项验证（每项跑命令并确认输出，再勾选）：

- [ ] **commit 已创建**：`git -C ${target_repo} log -1 --format="%H %s"`
      输出最新 commit hash + `research(cross-mcp):` 前缀的 message
- [ ] **research/ 下有当日新文件**：
      `ls ${target_repo}/research/insider-alert-$(date +%Y-%m-%d).md`
- [ ] **仅 research/ 被改动**：
      `git -C ${target_repo} diff --stat HEAD~1` 列出的所有文件路径都在
      `research/` 目录下
- [ ] **3 个 health_check 当时均 valid**：pre-flight 输出留底（聊天上下文）
- [ ] **报告含全部 8 段**：
      `grep -c '^##' ${target_repo}/research/insider-alert-$(date +%Y-%m-%d).md` ≥ 8
- [ ] **触发标的均含 4 子段**：每个触发 ticker 必须含 Form 4 表 / 打分拆解 /
      quote 对比 / class 判定
- [ ] **Form 4 引用了 accession_number**：
      `grep -E 'accession[_ ]?number' ...md | wc -l ≥ 1`（至少 1 处直引）
- [ ] **限制声明含 "Form 4 ... 2 .* delay"** 字样：避免误导用户以为是实时数据

## Rollback

```bash
cd ${target_repo}
git reset --soft HEAD~1   # 撤销 commit，保留 working tree
# 检查 working tree 后决定 git restore 或 git stash
git restore research/insider-alert-$(date +%Y-%m-%d).md
# 绝不 force-push 主分支。
```

## Failure modes

| Symptom | Action |
| --- | --- |
| watchlist 文件缺失 | 询问用户提供 ticker 列表；不要默认扫描全市场 |
| `sec-edgar-mcp` 返回 429（SEC fair-use 限流） | 等 1s 重试 1 次；连续两次 STOP；提示用户 sec-edgar SEC user-agent 是否合规 |
| `sec-edgar-mcp` user_agent 未配置 | STOP，要求用户先在 .env 设 `SEC_EDGAR_USER_AGENT="<name> <email>"` |
| 工具名与本 playbook 不一致（server 改名） | 用 `health_check()` 列出的实际 tool list 校对，按当前名调用；**不要凭记忆猜** |
| `polygon-news-mcp` 返回 401/403 | STOP，要求用户检查 `POLYGON_API_KEY` |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP，要求用户先跑 `auth login_flow` |
| `gh repo view` 失败 / 仓库不是 private | **STOP and refuse to write**；不绕过 |
| 同周已存在 `research/insider-alert-YYYY-MM-DD.md` | 询问用户是否覆盖；默认 skip 并告知（idempotency 周级别） |
| 触发标的某条 Form 4 解析失败（XBRL 缺字段） | 该条标 `(parse error)`，不影响其他条；**不补 0 / 不前向填充** |
| 异常打分全部 < 0.6（无触发） | 仍写报告，TL;DR 标"本期无内部人异常"，跳过 Step 4-6（节省 quota） |
