# Playbook — Factor screen deep-dive (full-market factor screen + multi-source research)

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/factor-screen-YYYY-MM-DD.md` (new) |
| clickhouse tools used | `health_check`, `screen_stocks`, `get_indicators` (optional recheck) |
| schwab tools used | `health_check`, `get_quote` |
| sec-edgar tools used | `health_check`, `get_institutional_holders` (13F holder reverse-lookup) |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 30 (clickhouse ≤ 2 + candidate count N≤6 × 3 sources + 4 health checks) |
| Data compliance | ClickHouse historical indicators are derived; SEC 13F is public; polygon/schwab are non-redistributable |
| Use case | Use the 1.49B-row history for a full-market cross-sectional factor scan, pick N candidates, then deep-research each across sources |
| Trigger keywords | "factor screen" / "full-market scan" / "因子筛选" / "screen + deep research" / "cross-sectional factor" |

> **This playbook only runs inside the stock-personal repo.** If `cwd` is
> not under `${target_repo}`, switch to read-only mode: emit analysis to
> chat, **never write to any other repo**.
>
> **clickhouse-mcp is the core orchestration target of this playbook**, but
> it is **not a hard dependency**: when CH is unavailable the playbook
> **degrades** — it skips the full-market scan (Step 2) and instead asks the
> user to **supply a candidate ticker list manually**, still running the
> Step 3-6 multi-source deep research (schwab + sec-edgar + polygon). In
> degraded mode the report frontmatter is marked `clickhouse: unavailable`
> and the TL;DR notes "full-market factor scan skipped (requires a CH
> read-only account)".

## Pre-flight (mandatory)

```text
1. clickhouse-mcp.health_check()          → overall_status == "ok" and
                                            connection_configured == true and
                                            clickhouse_reachable == true and
                                            read_only == true and
                                            server_version ∈ ">=0.1,<0.2"
   ↳ If overall_status == "unhealthy" (CH has no read-only account or is
     unreachable) → **degrade**: do NOT stop; set clickhouse_unavailable=true,
     skip Step 2, take the "manual candidates" branch.
2. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" and
                                            server_version ∈ ">=0.4,<0.5"
3. sec-edgar-mcp.health_check()           → user_agent_configured == true and
                                            server_version ∈ ">=0.4,<0.5"
                                            (needs v0.4.0+ for get_institutional_holders)
3.5. Verify sec-edgar server-side UA reachability (identical to SKILL.md
     §Activation handshake step 2.5): read health_check's sec_ua_reachable.status:
   - ACCEPTED       → ✅ continue
   - REJECTED_HTML_403 → ❌ STOP, tell the user to set SEC_EDGAR_USER_AGENT to a real email
   - UNCONFIGURED   → ❌ STOP, tell the user to configure SEC_EDGAR_USER_AGENT
   - TIMEOUT / NETWORK_ERROR → ⚠️ WARN, continue but flag "SEC probe unavailable"
4. polygon-news-mcp.health_check()        → api_key_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
5. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
6. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true (else STOP; do not bypass)
7. cwd ∈ ${target_repo} subtree? If not → read-only mode (chat output only)
```

> **clickhouse-mcp read-only account**: clickhouse-mcp needs
> `CLICKHOUSE_MCP_HOST` / `CLICKHOUSE_MCP_USER` (point it at a dedicated
> `readonly=1` + `GRANT SELECT` account on the USA side) /
> `CLICKHOUSE_MCP_PASSWORD`. Credentials are read from env only and are
> **never logged or repr'd**. When unset, `health_check()` returns
> `connection_configured == false` and this playbook takes the degraded branch.

## Steps

### Step 1 — Pick the universe and factor filters

- Default universe: the **1388 symbols** already in CH (L0/L1 full coverage).
  The user may narrow to a subset (e.g. sp500 / ndx100, approximated via the
  watchlist or a user-supplied ticker list).
- Default factor filters (user-overridable): pick 1-3 from the table below
  (`screen_stocks` allows up to 10 filters):

  | Factor theme | indicator (CH allow-list) | operator | example threshold | meaning |
  | --- | --- | --- | --- | --- |
  | oversold reversal | `rsi14` | `lt` | 30 | RSI oversold leaderboard |
  | momentum | `macd_hist` | `gt` | 0 | MACD histogram turned positive |
  | trend confirm | `adx14` | `gt` | 25 | strong trend (ADX>25) |
  | MA bullish | `ma20` / `ma60` | — | local compare | ma20>ma60 bullish stack (pull both, compare locally) |
  | volume | `obv` / `vwap` | — | local compare | price above VWAP |

> indicator names must be in clickhouse-mcp's `ALLOWED_INDICATORS` allow-list
> (`ma5/ma10/ma20/ma50/ma60/ma120/ma200/ma250/ema12/ema26/macd_dif/macd_dea/`
> `macd_hist/atr14/boll_mid/boll_up/boll_low/rsi14/stoch_rsi14/mfi14/kdj_k/`
> `kdj_d/kdj_j/adx14/obv/vwap`). Indicators outside the allow-list are
> rejected server-side; **do not guess indicator names** — verify with
> `get_server_info()` when in doubt.

### Step 2 — clickhouse-mcp full-market cross-sectional scan

```text
clickhouse-mcp.screen_stocks(
    filters=[
        {"indicator": "rsi14",     "operator": "lt", "value": 30},
        {"indicator": "macd_hist", "operator": "gt", "value": 0},
    ],
    as_of=None,           # None = most recent trading day for that freq in the CH view; or pass YYYY-MM-DD
    frequency="1d",       # 1m/5m/15m/1h/1d/1w; cross-sectional factors default to 1d
    limit=100,            # return cap (1..2000)
)
```

Expected fields: `frequency`, `as_of`, `filters[]`, `indicators[]`, `count`,
`matches[]` (each row `symbol` + the matched `ind_*` indicator values).

**Converge candidates locally** (no extra CH calls):

- Pick **top N (default N=5, max 6**, bounded by max tool calls) from `matches`
  per the factor-combination rule.
- Optional recheck: for each top-N call `get_indicators(symbol, indicator=...,
  frequency="1d", start=<as_of-60d>, end=<as_of>)` to inspect the indicator's
  last-60-day trajectory and drop "single-day spike" false signals.
- Record each candidate's `as_of` and matched indicator values for report §3.

> **Degraded branch (clickhouse_unavailable=true)**: skip this Step. Instead
> ask the user for a candidate ticker list (≤ 6); mark "full-market scan
> skipped; candidates user-specified". Steps 3-6 run as normal.

### Step 3 — Candidate live quote (schwab)

Only for the top-N candidates from Step 2, call one by one:

```text
schwab-marketdata-mcp.get_quote(
    symbol=<candidate>,
    fields=["QUOTE", "REGULAR"],
)
```

Record: `lastPrice`, `netChangeInDouble`, `netPercentChangeInDouble`,
`totalVolume`, `52WeekHigh`, `52WeekLow`, (if available) `30DayAverageVolume`.
Compute "current price vs CH scan close" delta and note whether it has drifted
from the scan timestamp.

> The tool signature follows what `get_server_info()` actually exposes (some
> versions expose the batch `get_quotes(symbols=[...])`); call per the actual
> server signature, do not guess.

### Step 4 — Candidate 13F institutional holders (sec-edgar)

Only for top-N candidates, call one by one (reverse-lookup which 13F managers
report a position):

```text
sec-edgar-mcp.get_institutional_holders(
    ticker=<candidate>,
    since_days=120,        # covers the latest 13F filing quarter (quarter + 45-day window)
)
```

Record: `ticker`, `as_of_quarter`, `holder_count`, `total_shares_reported`,
`top_holders[]` (each with `filer_name` / `shares` / `value_usd` /
`accession_number`). Locally judge "institutional interest": a high
`holder_count` with top-holder concentration → strong institutional endorsement.

> 13F has a 45-day filing lag (after quarter end); the report must flag
> "institutional holdings are lagged data, not real time". If the ticker has
> no 13F record (small-cap / newly listed) → mark `13f: none`; do not pad with
> zeros or forward-fill.

### Step 5 — Candidate news sentiment (polygon)

Only for top-N candidates, call one by one:

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<candidate>,
    window="7d",           # rolling 7-day sentiment
)
```

Record: `avg_sentiment ∈ [-1, 1]`, `positive_count`, `negative_count`,
`neutral_count`, `article_count`, `window_start`, `window_end`,
`top_articles[]` (if available: title / url / publisher / published_utc).
Sentiment verdict: `avg_sentiment ≥ +0.3` → bullish; `≤ -0.3` → bearish;
else neutral. **Do not reproduce article body text** (copyright safety).

### Step 6 — Write the candidate list + multi-source research brief

Write the following 8 sections to
`${target_repo}/research/factor-screen-YYYY-MM-DD.md`:

1. **Frontmatter**: `generated_at` (UTC), `universe`, `factor_filters`, `as_of`,
   `candidate_count`, `mcp_versions` (all 4: clickhouse/schwab/sec-edgar/polygon),
   `clickhouse` (ok | unavailable).
2. **TL;DR**: one-liner verdict, e.g. "Full-market RSI<30 ∩ MACD-hist>0 scanned
   5 candidates; AAPL has the strongest institutional backing + sentiment +0.41".
3. **Factor scan result table** (from Step 2): each candidate + matched indicator
   values + `as_of`; in degraded mode mark "scan skipped, candidates user-specified".
4. **Per-candidate multi-source research sub-block** (top N, each with 4 sub-sections):
   - schwab live quote (from Step 3): last / change / volume / distance to 52w high-low
   - sec-edgar 13F institutional holders (from Step 4): holder_count / top 3 holders + accession
   - polygon news sentiment (from Step 5): avg_sentiment / three counts / verdict
   - **synthesised research conclusion**: a one-line qualitative judgment of
     factor signal × institutions × sentiment (draft signal)
5. **Candidate cross-comparison table**: N rows × columns (factor score / inst.
   interest / sentiment / synthesis), sorted by synthesis.
6. **Risk callouts**: ① CH indicators may lag (USA L2 increment not auto-triggered);
   ② 13F 45-day lag; ③ factor signals are cross-sectional relative, not timing;
   list 1-2 un-priced risks.
7. **Suggested next actions**: per candidate give a **generic** action (add to
   watchlist / deep-dive filings / monitoring cadence); **not investment advice**.
8. **Data provenance & limits**: links to clickhouse-mcp (CH derived indicators),
   SEC EDGAR 13F, polygon API tier, Schwab non-redistribution clause.

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 already wrote the file
git add research/factor-screen-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): factor-screen-deep-dive $(date -u +%Y-%m-%d)"
# DO NOT push --force.  Plain push to the research branch is enough.
```

## Acceptance criteria

After completion verify each item (run the command and confirm output, then tick):

- [ ] **Activation handshake 7 steps captured**: pre-flight transcript in chat
      context; when clickhouse is unavailable it MUST take the degraded branch
      (mark `clickhouse: unavailable`) rather than STOP.
- [ ] **Commit landed**: `git -C ${target_repo} log -1 --format="%H %s"` shows
      the new commit hash + a `research(cross-mcp):` prefixed message.
- [ ] **Today's new file under research/**:
      `ls ${target_repo}/research/factor-screen-$(date +%Y-%m-%d).md`.
- [ ] **Only research/ touched**: every path in
      `git -C ${target_repo} diff --stat HEAD~1` lives under `research/`.
- [ ] **Report contains all 8 sections**:
      `grep -c '^##' ${target_repo}/research/factor-screen-$(date +%Y-%m-%d).md` ≥ 8.
- [ ] **Each candidate has 4 sub-sections**: every top-N candidate must contain
      schwab quote / 13F holders / news sentiment / synthesis.
- [ ] **CH provenance includes as_of**: report §3 / §8 must state the scan
      `as_of` date so the user is not misled into thinking it is a real-time
      full-market snapshot.
- [ ] **Degraded path verifiable**: if clickhouse_unavailable, §3 must clearly
      mark "full-market scan skipped, candidates user-specified", and Steps 3-6
      still produce full output.

## Rollback

```bash
cd ${target_repo}
# Committed but not pushed → reset --soft, amend content, then push
git reset --soft HEAD~1   # undo the commit, keep working tree
git restore research/factor-screen-$(date +%Y-%m-%d).md

# Pushed and then found wrong → git revert (keeps audit trail, never force push)
git revert <hash>
git push origin <branch>   # no --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `clickhouse-mcp.health_check()` `overall_status == "unhealthy"` (CH unset/unreachable) | **Degrade, do not STOP**: mark `clickhouse: unavailable`, skip Step 2, ask the user for a candidate ticker list; Steps 3-6 as normal. |
| `clickhouse-mcp` `connection_configured == false` (missing read-only account env) | Degrade as above; tell the user to set `CLICKHOUSE_MCP_HOST/_USER/_PASSWORD` (recommend a `readonly=1` dedicated account). |
| `screen_stocks` large-query timeout (CH `max_execution_time` hit, returns query failed) | Narrow the universe (watchlist subset) or tighten filter thresholds, retry once; still timing out → degrade to manual candidates. |
| `screen_stocks` returns `count == 0` (no symbol matches the factors) | Do not STOP; loosen thresholds, retry once; still 0 → report "no factor matches this run", note in TL;DR. |
| indicator name not in `ALLOWED_INDICATORS` allow-list (server rejects) | Verify the allow-list via `get_server_info()`, swap in a valid indicator; **do not guess indicator names**. |
| `sec-edgar-mcp` returns 429 (SEC fair-use rate limit) | Wait 1s, retry once; STOP after a second failure; remind the user to check the sec-edgar UA. |
| `sec-edgar-mcp` 403 / `sec_ua_reachable.status == REJECTED_HTML_403` | **STOP**, tell the user to check `SEC_EDGAR_USER_AGENT` (see SKILL.md handshake step 2.5). |
| candidate has no 13F record (small-cap / newly listed) | mark `13f: none`; do not pad with zeros or forward-fill; downgrade synthesis to "institutional data missing". |
| `polygon-news-mcp` returns 401/403 | STOP, ask the user to check `POLYGON_API_KEY`. |
| `polygon-news-mcp` returns 429 | Wait 60s, retry once; STOP if it happens again. |
| `get_news_sentiment_aggregate` `article_count == 0` | Report "no relevant news in the past 7 days"; do not pad with zeros; treat synthesis as neutral. |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP, ask the user to run `auth login_flow`. |
| Any token / credential expired (schwab / polygon / sec-edgar / CH) | STOP and report which source failed + how to fix; **do not fabricate data**. |
| `gh repo view` fails / repo is not private | **STOP and refuse to write**; do not bypass. |
| `research/factor-screen-YYYY-MM-DD.md` already exists today | Ask whether to overwrite; default skip and tell the user. |

## Idempotency

| Repeat run | Side effects |
| --- | --- |
| At most once per day | Writes `research/factor-screen-YYYY-MM-DD.md`; one new file per day; asks before overwrite (default skip). |
| Different day | One new file per day; filename carries date, naturally isolated. |
| CH-unavailable degraded run | Still writes today's file (candidates user-specified); frontmatter marked `clickhouse: unavailable` to distinguish. |

## See also

- Sibling playbooks:
  - `playbooks/correlation-pairs-monitor.md` (correlation pairs monitor, also
    orchestrates CH + MCP)
  - `playbooks/shakeout-with-news.md` (shakeout signal + news sentiment)
  - `playbooks/earnings-preview.md` (pre-earnings IV-rank-aware positioning brief)
- clickhouse-mcp: [kevinkda/clickhouse-mcp](https://github.com/kevinkda/clickhouse-mcp)
  (7 read-only tools, 1.49B-row history + L2 materialised indicators)
- `stock-personal/docs/sprints/usa-clickhouse-quant-integration-plan.md §3`:
  source of this playbook's quant use cases (full-market scan / cross-sectional
  factor P0).
