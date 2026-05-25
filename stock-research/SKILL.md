---
name: stock-research
language_directive: "Always respond to the user in Simplified Chinese (简体中文)."
required_workspace: "/opt/workspace/code/kevinkda/stock-personal"
mcp_dependencies:
  - name: schwab-marketdata-mcp
    version_range: ">=0.3,<0.4"
  - name: sec-edgar-mcp
    version_range: ">=0.2,<0.3"
  - name: polygon-news-mcp
    version_range: ">=0.2,<0.3"
description: |
  Cross-MCP equity research skill that orchestrates schwab-marketdata-mcp +
  sec-edgar-mcp + polygon-news-mcp into multi-step playbooks for the
  kevinkda/stock-personal investment workflow.

  Triggers on "shakeout with news", "insider alert", "shakeout 配新闻",
  "内部人交易告警", "shakeout-with-news", "insider-alert".

  对于以上场景使用本 skill；面向用户的所有回答必须使用简体中文。
---

# stock-research（中文版）

跨 MCP server 的股票研究 skill，编排 schwab-marketdata-mcp + sec-edgar-mcp +
polygon-news-mcp 三库形成多步 playbook，服务 kevinkda/stock-personal 投研流程。

## 何时使用本 skill

当用户请求涉及**多个数据源协同**的研究流程时使用：

- shakeout 信号 + 新闻情绪叠加 → `shakeout-with-news` playbook
- 内部人交易异常告警 → `insider-alert` playbook

如果只调用单一 MCP server，请用对应仓的 skill：

- `schwab-marketdata-ops` / `schwab-marketdata-workflows`
- 未来可能：`sec-edgar-ops` / `polygon-news-ops`

## Activation handshake（激活时必跑）

1. 调用 `schwab-marketdata-mcp.health_check()`，验证 `overall_status == "healthy"`
   且 server_version 在 `>=0.3,<0.4` 范围
2. 调用 `sec-edgar-mcp.health_check()`，验证 user_agent_configured + server_version
   在 `>=0.2,<0.3`
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
4. 跑 `git -C $required_workspace remote get-url origin` 确认在 `kevinkda/stock-personal`
5. 跑 `gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate` 确认 `true`
6. 任一失败 → 立即停下并告知用户

## Playbook 选择表

| 用户意图 | playbook | 工具链 |
| --- | --- | --- |
| "shakeout 配新闻" / "shakeout 触发后看新闻" | `playbooks/shakeout-with-news.md` | schwab(price_history) + polygon(sentiment_aggregate, ticker_news) |
| "内部人交易告警" / "扫 watchlist 内部人" | `playbooks/insider-alert.md` | sec-edgar(form4) + polygon(news) + schwab(quote) |

## Idempotency

| Playbook | 重复运行 | 副作用 |
| --- | --- | --- |
| shakeout-with-news | 同日重跑 ≤ 1 次（cache hit_rate ≥ 30% gate） | 写 `research/shakeout-news-YYYY-MM-DD.md`，每日新文件 |
| insider-alert | 同周重跑 ≤ 1 次 | 写 `research/insider-alert-YYYY-MM-DD.md` |

## 通用约束

- **commit 前缀**：`research(cross-mcp):`，便于事后审计与单库 skill 区分
- **不直接 commit 到 main**：在 `research/cross-mcp-YYYYMMDD` 分支上 commit
- **永不 push --force**：尤其不允许对 main / mainline 强推
- **私有仓校验**：每次写盘前 `gh repo view kevinkda/stock-personal --json isPrivate`
  必须为 `true`，否则停下不写

## See also

- 配套 MCP server：[schwab-marketdata-mcp](https://github.com/kevinkda/schwab-marketdata-mcp) +
  [sec-edgar-mcp](https://github.com/kevinkda/sec-edgar-mcp) +
  [polygon-news-mcp](https://github.com/kevinkda/polygon-news-mcp)
- 单库 skill：[schwab-marketdata-skill](https://github.com/kevinkda/schwab-marketdata-skill)
  （含 shakeout-analysis-v2 / voo-qqq-tracker / watchlist-snapshot /
  summary-md-refresh 等单库 playbook）
- 项目战略：`stock-personal/docs/STRATEGY.md`
