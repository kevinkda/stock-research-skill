---
name: stock-research
language_directive: "Always respond to the user in Simplified Chinese (简体中文)."
required_workspace: "/opt/workspace/code/kevinkda/stock-personal"
mcp_dependencies:
  - name: schwab-marketdata-mcp
    version_range: ">=0.4,<0.5"
  - name: sec-edgar-mcp
    version_range: ">=0.4,<0.5"
  - name: polygon-news-mcp
    version_range: ">=0.2,<0.3"
  - name: clickhouse-mcp
    version_range: ">=0.1,<0.2"
description: |
  Cross-MCP equity research skill that orchestrates schwab-marketdata-mcp +
  sec-edgar-mcp + polygon-news-mcp + clickhouse-mcp into multi-step playbooks
  for the kevinkda/stock-personal investment workflow.

  Triggers on "shakeout with news", "insider alert", "shakeout 配新闻",
  "内部人交易告警", "shakeout-with-news", "insider-alert",
  "earnings preview", "财报前瞻", "factor screen", "因子筛选",
  "全市场扫描", "correlation pairs", "相关性配对", "配对交易".

  对于以上场景使用本 skill；面向用户的所有回答必须使用简体中文。
---

# stock-research（中文版）

跨 MCP server 的股票研究 skill，编排 schwab-marketdata-mcp + sec-edgar-mcp +
polygon-news-mcp + clickhouse-mcp 四库形成多步 playbook，服务
kevinkda/stock-personal 投研流程。

## 何时使用本 skill

当用户请求涉及**多个数据源协同**的研究流程时使用：

- shakeout 信号 + 新闻情绪叠加 → `shakeout-with-news` playbook
- 内部人交易异常告警 → `insider-alert` playbook
- 财报前瞻（IV-rank-aware 定位简报）→ `earnings-preview` playbook
- 全市场因子筛选 + 多源深度研究 → `factor-screen-deep-dive` playbook
- 相关性配对监控（配对交易候选）→ `correlation-pairs-monitor` playbook

如果只调用单一 MCP server，请用对应仓的 skill：

- `schwab-marketdata-ops` / `schwab-marketdata-workflows`
- 未来可能：`sec-edgar-ops` / `polygon-news-ops`

## Activation handshake（激活时必跑）

1. 调用 `schwab-marketdata-mcp.health_check()`，验证 `overall_status == "healthy"`
   且 server_version 在 `>=0.4,<0.5` 范围
2. 调用 `sec-edgar-mcp.health_check()`，验证 user_agent_configured + server_version
   在 `>=0.4,<0.5`（factor-screen-deep-dive 用 `get_institutional_holders`，
   需 sec-edgar v0.4.0+）
2.5. 验证 sec-edgar 服务端 UA 真实可达性。读
   `health_check()` 返回的 `sec_ua_reachable.status` 字段（v0.2.0+，
   R7 三层防御第三层）：
   - `ACCEPTED` → ✅ SEC 实际接受当前 UA，可继续
   - `REJECTED_HTML_403` → ❌ STOP，让用户把 `sec-edgar-mcp/.env` 里的
     `SEC_EDGAR_USER_AGENT` 改为真实可达邮箱并 reload Cursor
     （SEC fair-use 政策已 deny-list 当前 UA）
   - `UNCONFIGURED` → ❌ STOP，让用户配置 `SEC_EDGAR_USER_AGENT`
     （UA 缺失、格式错误、或包含 `noreply`/`example.com`/`set-your-email`
     等占位符）
   - `TIMEOUT` / `NETWORK_ERROR` → ⚠️ WARN（继续执行，但在最终报告中
     标注 "SEC 探针暂时不可用，可能影响数据新鲜度"）
   注意：`user_agent_configured=true` 只验本地 env 格式，不验 SEC 端
   是否实际接受。本 step 检查的是 sec-edgar-mcp 服务端发出的真实 HEAD
   探针结果（5 min 缓存），可识别"邮箱合法但 SEC IP deny-list"等
   仅靠字符串黑名单识别不出的深层问题（2026-05-25 PB-3 实测发现）。
3. 调用 `polygon-news-mcp.health_check()`，验证 api_key_configured + server_version
   在 `>=0.2,<0.3`
3.5. 调用 `clickhouse-mcp.health_check()`（量化 playbook 必跑；非量化 playbook
   可跳过），验证：
   - `overall_status == "ok"` 且 `connection_configured == true` 且
     `clickhouse_reachable == true` 且 `read_only == true` 且 server_version
     在 `>=0.1,<0.2`
   - **clickhouse 降级约定**：若 `overall_status == "unhealthy"` 或
     `connection_configured == false`（CH 未配只读账户或不可达）→ **不阻塞**：
     量化 playbook 进入降级分支（factor-screen 改人工候选；correlation-pairs
     仅出只读建议），报告 frontmatter 标 `clickhouse: unavailable`。
     **clickhouse 不可用绝不阻塞用现有 3 个 MCP 的部分。**
   - **clickhouse-mcp 只读账户**：需配 `CLICKHOUSE_MCP_HOST` /
     `CLICKHOUSE_MCP_USER`（建议 USA 侧 `readonly=1` + `GRANT SELECT` 专用账户）/
     `CLICKHOUSE_MCP_PASSWORD`；凭证只从 env 读，从不写日志/repr。
4. 跑 `git -C $required_workspace remote get-url origin` 确认在 `kevinkda/stock-personal`
5. 跑 `gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate` 确认 `true`
6. 任一失败 → 立即停下并告知用户（clickhouse 不可用除外：量化 playbook 降级，不停）

## Playbook 选择表

| 用户意图 | playbook | 工具链 |
| --- | --- | --- |
| "shakeout 配新闻" / "shakeout 触发后看新闻" | `playbooks/shakeout-with-news.md` | schwab(price_history) + polygon(sentiment_aggregate, ticker_news) |
| "内部人交易告警" / "扫 watchlist 内部人" | `playbooks/insider-alert.md` | sec-edgar(form4) + polygon(news) + schwab(quote) |
| "财报前瞻" / "earnings preview" / "财报要关注什么" | `playbooks/earnings-preview.md` | schwab(get_iv_percentile, get_price_history) + sec-edgar(get_8k_with_items) + polygon(get_news_sentiment_aggregate) |
| "因子筛选" / "全市场扫描" / "横截面因子选股 + 深度研究" | `playbooks/factor-screen-deep-dive.md` | clickhouse(screen_stocks) + schwab(get_quote) + sec-edgar(get_institutional_holders) + polygon(get_news_sentiment_aggregate) |
| "相关性配对" / "配对交易" / "相关性矩阵监控" | `playbooks/correlation-pairs-monitor.md` | clickhouse(get_correlation_matrix, get_ohlcv) + schwab(get_quote) + polygon(get_news_sentiment_aggregate) |

## Idempotency

| Playbook | 重复运行 | 副作用 |
| --- | --- | --- |
| shakeout-with-news | 同日重跑 ≤ 1 次（启用缓存时受 hit_rate ≥ 30% gate；缓存默认禁用，需 `export SCHWAB_CACHE_ENABLED=true` 才校验） | 写 `research/shakeout-news-YYYY-MM-DD.md`，每日新文件 |
| insider-alert | 同周重跑 ≤ 1 次 | 写 `research/insider-alert-YYYY-MM-DD.md` |
| earnings-preview | 同 ticker 同日重跑 ≤ 1 次（启用缓存时受 hit_rate ≥ 30% gate；缓存默认禁用，需 `export SCHWAB_CACHE_ENABLED=true` 才校验） | 写 `research/earnings-preview-{TICKER}-YYYY-MM-DD.md`，按 ticker+date 隔离 |
| factor-screen-deep-dive | 同日重跑 ≤ 1 次 | 写 `research/factor-screen-YYYY-MM-DD.md`，每日新文件；CH 不可用降级时仍写（候选人工指定），frontmatter 标 `clickhouse: unavailable` |
| correlation-pairs-monitor | 同日重跑 ≤ 1 次 | 写 `research/correlation-pairs-YYYY-MM-DD.md`，每日新文件；CH 不可用时仅在用户接受"无相关性矩阵"时落盘 |

## 通用约束

- **commit 前缀**：`research(cross-mcp):`，便于事后审计与单库 skill 区分
- **不直接 commit 到 main**：在 `research/cross-mcp-YYYYMMDD` 分支上 commit
- **永不 push --force**：尤其不允许对 main / mainline 强推
- **私有仓校验**：每次写盘前 `gh repo view kevinkda/stock-personal --json isPrivate`
  必须为 `true`，否则停下不写

## See also

- 配套 MCP server：[schwab-marketdata-mcp](https://github.com/kevinkda/schwab-marketdata-mcp) +
  [sec-edgar-mcp](https://github.com/kevinkda/sec-edgar-mcp) +
  [polygon-news-mcp](https://github.com/kevinkda/polygon-news-mcp) +
  [clickhouse-mcp](https://github.com/kevinkda/clickhouse-mcp)（量化：14.9 亿行历史 + L2 物化指标）
- 单库 skill：[schwab-marketdata-skill](https://github.com/kevinkda/schwab-marketdata-skill)
  （含 shakeout-analysis-v2 / voo-qqq-tracker / watchlist-snapshot /
  summary-md-refresh 等单库 playbook）
- 项目战略：`stock-personal/docs/STRATEGY.md`
