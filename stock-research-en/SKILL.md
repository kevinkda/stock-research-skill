---
name: stock-research-en
language_directive: "Always respond to the user in English."
required_workspace: "/opt/workspace/code/kevinkda/stock-personal"
mcp_dependencies:
  - name: schwab-marketdata-mcp
    version_range: ">=0.3,<0.4"
  - name: sec-edgar-mcp
    version_range: ">=0.2.1,<0.3"
  - name: polygon-news-mcp
    version_range: ">=0.2,<0.3"
description: |
  Cross-MCP equity research skill that orchestrates schwab-marketdata-mcp +
  sec-edgar-mcp + polygon-news-mcp into multi-step playbooks for the
  kevinkda/stock-personal investment workflow.

  Triggers on "shakeout with news", "insider alert", "shakeout 配新闻",
  "内部人交易告警", "shakeout-with-news", "insider-alert".

  Use this skill for the scenarios above; respond to the user in English.
---

# stock-research-en (English mirror)

Cross-MCP equity research skill that chains schwab-marketdata-mcp +
sec-edgar-mcp + polygon-news-mcp into multi-step playbooks for the
kevinkda/stock-personal investment workflow.

## When to use this skill

Use this skill when the user requests a research workflow that **spans
multiple data sources**:

- Shakeout signal + news sentiment overlay → `shakeout-with-news` playbook
- Insider trading anomaly alert → `insider-alert` playbook

If only a single MCP server is needed, prefer the per-server skill:

- `schwab-marketdata-ops` / `schwab-marketdata-workflows`
- (future) `sec-edgar-ops` / `polygon-news-ops`

## Activation handshake (mandatory)

1. Call `schwab-marketdata-mcp.health_check()`; verify `overall_status == "healthy"`
   and `server_version` satisfies `>=0.3,<0.4`.
2. Call `sec-edgar-mcp.health_check()`; verify `user_agent_configured` and
   `server_version` satisfies `>=0.2,<0.3`.
2.5. Verify the sec-edgar server-side UA reachability. Read the
   `sec_ua_reachable.status` field returned by `health_check()` (v0.2.0+,
   third layer of the R7 three-layer defence):
   - `ACCEPTED` → ✅ SEC actually accepts the configured UA; continue.
   - `REJECTED_HTML_403` → ❌ STOP and tell the user to change
     `SEC_EDGAR_USER_AGENT` in `sec-edgar-mcp/.env` to a real reachable
     email and reload Cursor (SEC fair-use policy has deny-listed the
     current UA).
   - `UNCONFIGURED` → ❌ STOP and tell the user to configure
     `SEC_EDGAR_USER_AGENT` (UA missing, malformed, or contains a
     known placeholder such as `noreply`, `example.com`, or
     `set-your-email`).
   - `TIMEOUT` / `NETWORK_ERROR` → ⚠️ WARN (continue execution but flag
     in the final report: "SEC probe transiently unavailable; data
     freshness may be affected").
   Note: `user_agent_configured=true` only validates the local env-var
   format, not whether SEC's edge actually accepts the UA. This step
   inspects the result of a real HEAD probe issued by sec-edgar-mcp
   server-side (cached for 5 min), which catches deep issues like
   "email is well-formed but SEC has IP deny-listed it" that pure
   string-blacklist checks cannot detect (discovered 2026-05-25 during
   PB-3 validation).
3. Call `polygon-news-mcp.health_check()`; verify `api_key_configured` and
   `server_version` satisfies `>=0.2,<0.3`.
4. Run `git -C $required_workspace remote get-url origin` and confirm it
   points to `kevinkda/stock-personal`.
5. Run `gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate`
   and confirm the result is `true`.
6. If any check fails — STOP immediately and report to the user.

## Playbook selection table

| User intent | Playbook | Tool chain |
| --- | --- | --- |
| "shakeout with news" / "news after a shakeout trigger" | `playbooks/shakeout-with-news.md` | schwab(price_history) + polygon(sentiment_aggregate, ticker_news) |
| "insider alert" / "scan watchlist for insider trades" | `playbooks/insider-alert.md` | sec-edgar(form4) + polygon(news) + schwab(quote) |

## Idempotency

| Playbook | Repeat run | Side effects |
| --- | --- | --- |
| shakeout-with-news | At most once per day (cache hit_rate ≥ 30% gate) | Writes `research/shakeout-news-YYYY-MM-DD.md`; one new file per day |
| insider-alert | At most once per week | Writes `research/insider-alert-YYYY-MM-DD.md` |

## Universal constraints

- **Commit prefix**: `research(cross-mcp):` — distinguishes this skill from
  per-server skills during audit.
- **Never commit to main**: work on a `research/cross-mcp-YYYYMMDD` branch.
- **Never force-push**: especially never to `main` / `mainline`.
- **Private-repo gate**: every write must be preceded by
  `gh repo view kevinkda/stock-personal --json isPrivate` returning `true`,
  or the playbook stops without writing.

## See also

- Companion MCP servers:
  [schwab-marketdata-mcp](https://github.com/kevinkda/schwab-marketdata-mcp) +
  [sec-edgar-mcp](https://github.com/kevinkda/sec-edgar-mcp) +
  [polygon-news-mcp](https://github.com/kevinkda/polygon-news-mcp).
- Per-server skills:
  [schwab-marketdata-skill](https://github.com/kevinkda/schwab-marketdata-skill)
  (ships shakeout-analysis-v2 / voo-qqq-tracker / watchlist-snapshot /
  summary-md-refresh and other single-server playbooks).
- Project strategy: `stock-personal/docs/STRATEGY.md`.
