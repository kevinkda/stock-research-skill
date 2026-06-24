# Playbook — Correlation pairs monitor (correlation-driven pairs monitoring)

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/correlation-pairs-YYYY-MM-DD.md` (new) |
| clickhouse tools used | `health_check`, `get_correlation_matrix`, `get_ohlcv` (spread calc) |
| schwab tools used | `health_check`, `get_quote` |
| sec-edgar tools used | (this playbook does not call sec-edgar) |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 24 (clickhouse ≤ 4 + high-corr pairs P≤5 × 2 sources + 3 health checks) |
| Data compliance | ClickHouse historical correlation is derived; polygon/schwab are non-redistributable |
| Use case | Use the historical correlation matrix of a watchlist to find high-corr pairs → live spread → pairs-trading candidates |
| Trigger keywords | "correlation pairs" / "pairs trading" / "相关性配对" / "配对交易" / "correlation matrix" |

> **This playbook only runs inside the stock-personal repo.** If `cwd` is
> not under `${target_repo}`, switch to read-only mode: emit analysis to
> chat, **never write to any other repo**.
>
> **clickhouse-mcp is the core orchestration target of this playbook** and is
> a **hard prerequisite**: the correlation matrix depends on CH's multi-symbol
> historical OHLCV (single-symbol MCPs cannot batch-compute it). When CH is
> unavailable the playbook **degrades to read-only advice**: it does not write
> correlation results, only emits "configure a CH read-only account and re-run",
> and optionally uses schwab live quotes to give a **current price snapshot**
> (no historical correlation, not a pairs signal). In degraded mode the
> frontmatter is marked `clickhouse: unavailable`.

## Pre-flight (mandatory)

```text
1. clickhouse-mcp.health_check()          → overall_status == "ok" and
                                            connection_configured == true and
                                            clickhouse_reachable == true and
                                            read_only == true and
                                            server_version ∈ ">=0.1,<0.2"
   ↳ If overall_status == "unhealthy" (CH has no read-only account or is
     unreachable) → **degrade**: do NOT stop; set clickhouse_unavailable=true,
     skip Steps 2-3, take the "read-only advice" branch (only emit a current
     price snapshot + a "configure CH" hint).
2. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" and
                                            server_version ∈ ">=0.4,<0.5"
3. polygon-news-mcp.health_check()        → api_key_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true (else STOP; do not bypass)
6. cwd ∈ ${target_repo} subtree? If not → read-only mode (chat output only)
```

> **clickhouse-mcp read-only account**: the connection needs
> `CLICKHOUSE_MCP_HOST` / `CLICKHOUSE_MCP_USER` (point it at a dedicated
> `readonly=1` + `GRANT SELECT` account on the USA side) /
> `CLICKHOUSE_MCP_PASSWORD`. Credentials are read from env only and are
> **never logged or repr'd**. When unset, `health_check()` returns
> `connection_configured == false` and this playbook takes the degraded branch.
>
> **This playbook does not call sec-edgar** (pairs trading looks at
> spread/correlation + news divergence, not fundamental filings directly), so
> the handshake only runs 3 health_checks.

## Steps

### Step 1 — Parse the watchlist and the pair pool

- Read the watchlist: prefer `${target_repo}/portfolio/watchlist.md`, fall back
  to `${target_repo}/watchlist.md` or `${target_repo}/trackers/watchlist.md`.
- Extract the ticker list (dedup, trim). `get_correlation_matrix` requires
  **2 ≤ symbols ≤ 50**; above 50, take the first 50 and note "truncated N" in
  the report.
- The user may explicitly pass a set of symbols (e.g. same-sector ETFs / sector
  leaders) instead of the watchlist.

### Step 2 — clickhouse-mcp historical correlation matrix

```text
clickhouse-mcp.get_correlation_matrix(
    symbols=["XLE", "XOM", "CVX", "COP", "SLB"],   # 2..50, deduped
    start="2025-06-01",     # default lookback ≈ 252 trading days (1 year)
    end="2026-06-01",
    frequency="1d",         # pairs correlation defaults to daily
    method="pearson",       # pearson | spearman (spearman is more outlier-robust)
)
```

Expected fields: `symbols[]`, `frequency`, `method`, `start`, `end`,
`matrix` (nested dict: `matrix[a][b]` = correlation of a,b ∈ [-1,1] | null).
**Note**: correlation is over **daily simple returns** (not price levels),
aligned on the two symbols' shared trading days.

**Pick high-corr pairs locally** (no extra CH calls):

- Take the upper triangle of `matrix` (drop the 1.0 diagonal and symmetric
  duplicates), sort by |corr| descending.
- Pick pairs with **|corr| ≥ 0.8** as pair candidates (user-overridable
  threshold); cap at **P=5 pairs** (bounded by max tool calls).
- Note each pair's correlation sign (positive = co-moving, spread mean-reversion
  candidate; strong negative = hedge candidate).

> **Degraded branch (clickhouse_unavailable=true)**: skip this Step and Step 3.
> The report only emits "configure a CH read-only account and re-run the
> correlation matrix", optionally a schwab current price snapshot of the
> watchlist (clearly marked "no historical correlation, not a pairs signal").

### Step 3 — Pair spread history (clickhouse-mcp get_ohlcv)

For the high-corr pairs from Step 2, pull both legs' historical closes per pair
and compute the spread Z-score:

```text
clickhouse-mcp.get_ohlcv(
    symbol=<leg_A>,
    start="2025-06-01",
    end="2026-06-01",
    frequency="1d",
    limit=300,
)
# call once more for leg_B
```

Expected fields: `symbol`, `frequency`, `start`, `end`, `table`, `count`,
`bars[]` (each `ts` / `open` / `high` / `low` / `close` / `volume`).

**Compute spread stats locally** (no extra CH calls):

- Align the two legs' `close` on shared trading days, compute the spread
  `spread = close_A - β·close_B` (β via OLS slope or a simple ratio, estimated
  locally).
- Compute the spread mean / std → current **Z-score = (spread_now - mean) / std**.
- Estimate the half-life (spread AR(1) coefficient → `halflife = -ln2 / ln(φ)`),
  measuring the reversion speed.
- |Z| ≥ 2 marks "spread divergence" (potential pair entry zone); |Z| < 1 marks
  "spread convergence".

### Step 4 — Live pair spread (schwab)

Only for the two legs of the Step 2 high-corr pairs, call live quotes one by one
to recalibrate the current spread:

```text
schwab-marketdata-mcp.get_quote(
    symbol=<leg>,
    fields=["QUOTE", "REGULAR"],
)
```

Record `lastPrice` / `netPercentChangeInDouble`, recompute the current spread
and Z-score from live prices (overriding the Step 3 Z computed from the CH
last-day close), and note the "live Z vs historical last-day Z" delta.

> The tool signature follows what `get_server_info()` actually exposes; call
> per the actual server signature, do not guess.

### Step 5 — News divergence check (polygon)

Only for the Step 2 high-corr pairs, call news sentiment **once per leg** to
verify whether there is a fundamental divergence (a high-corr pair whose spread
suddenly widens is often because one leg got an idiosyncratic catalyst):

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<leg>,
    window="7d",
)
```

Record `avg_sentiment` / `article_count` / `top_articles[]` (if available).
**Divergence verdict**: leg sentiment gap `|sent_A - sent_B| ≥ 0.4` →
"fundamental divergence" (spread widening may have a real driver — **be cautious
with mean reversion**); close sentiment → "no clear divergence" (spread
divergence more likely technical, mean-reversion logic holds better). **Do not
reproduce article body text.**

### Step 6 — Write the pairs monitor report

Write the following 8 sections to
`${target_repo}/research/correlation-pairs-YYYY-MM-DD.md`:

1. **Frontmatter**: `generated_at` (UTC), `symbols` (input pool), `window`
   (start..end), `method`, `corr_threshold`, `pair_count`, `mcp_versions`
   (all 3: clickhouse/schwab/polygon), `clickhouse` (ok | unavailable).
2. **TL;DR**: one-liner verdict, e.g. "XLE-XOM corr 0.93, current Z +2.3 diverged,
   no news divergence → mean-reversion candidate".
3. **Correlation matrix summary** (from Step 2): upper-triangle |corr|-descending
   top table (pair / corr / sign); in degraded mode mark "matrix skipped, requires CH".
4. **Per high-corr pair detail block** (top P, each with 4 sub-sections):
   - historical spread stats (from Step 3): β / mean / std / last-day Z / half-life
   - live spread (from Step 4): both legs last / live Z / vs historical Z delta
   - news divergence (from Step 5): both legs avg_sentiment / divergence verdict
   - **pair conclusion**: a one-line qualitative judgment of correlation × Z
     divergence × news divergence (draft signal)
5. **Pair candidate ranking table**: P rows × columns (corr / current Z /
   half-life / divergence / synthesis), sorted by |Z| × no-divergence.
6. **Risk callouts**: ① correlation is historical and may break on a regime
   switch; ② pairs trading needs a long+short two-leg book — this playbook
   **does not place orders** (signals only); ③ a long half-life = capital tied
   up longer; list 1-2 un-priced risks.
7. **Suggested next actions**: per pair give a **generic** action (add to pair
   monitoring / wait for Z to revert / watch divergence news); **not investment
   advice**.
8. **Data provenance & limits**: links to clickhouse-mcp (CH historical
   correlation/spread), polygon API tier, Schwab non-redistribution clause.

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 already wrote the file
git add research/correlation-pairs-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): correlation-pairs-monitor $(date -u +%Y-%m-%d)"
# DO NOT push --force.  Plain push to the research branch is enough.
```

## Acceptance criteria

After completion verify each item (run the command and confirm output, then tick):

- [ ] **Activation handshake 6 steps captured**: pre-flight transcript in chat
      context; when clickhouse is unavailable it MUST take the degraded branch
      (mark `clickhouse: unavailable`) rather than STOP.
- [ ] **Commit landed**: `git -C ${target_repo} log -1 --format="%H %s"` shows
      the new commit hash + a `research(cross-mcp):` prefixed message.
- [ ] **Today's new file under research/**:
      `ls ${target_repo}/research/correlation-pairs-$(date +%Y-%m-%d).md`.
- [ ] **Only research/ touched**: every path in
      `git -C ${target_repo} diff --stat HEAD~1` lives under `research/`.
- [ ] **Report contains all 8 sections**:
      `grep -c '^##' ${target_repo}/research/correlation-pairs-$(date +%Y-%m-%d).md` ≥ 8.
- [ ] **Each high-corr pair has 4 sub-sections**: every pair must contain
      historical spread / live spread / news divergence / pair conclusion.
- [ ] **Correlation window fully labelled**: report §3 / §8 must state the
      `start..end` window + `method` so the user is not misled into reading it
      as an "any point in time" correlation.
- [ ] **Pair conclusion marked "not an order"**: §4 / §6 must clearly state
      "draft signal, not an order", and this playbook has zero order surface
      throughout.

## Rollback

```bash
cd ${target_repo}
# Committed but not pushed → reset --soft, amend content, then push
git reset --soft HEAD~1   # undo the commit, keep working tree
git restore research/correlation-pairs-$(date +%Y-%m-%d).md

# Pushed and then found wrong → git revert (keeps audit trail, never force push)
git revert <hash>
git push origin <branch>   # no --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `clickhouse-mcp.health_check()` `overall_status == "unhealthy"` (CH unset/unreachable) | **Degrade, do not STOP**: mark `clickhouse: unavailable`, skip Steps 2-3, only emit "configure a CH read-only account and re-run" + optional schwab current price snapshot (not a pairs signal). |
| `clickhouse-mcp` `connection_configured == false` (missing read-only account env) | Degrade as above; tell the user to set `CLICKHOUSE_MCP_HOST/_USER/_PASSWORD` (recommend a `readonly=1` dedicated account). |
| `get_correlation_matrix` symbols < 2 or > 50 (validation failure) | Resize the pool to [2,50]; > 50 take first 50 and mark "truncated"; < 2 ask the user to add symbols. |
| `get_correlation_matrix` / `get_ohlcv` large-query timeout (CH `max_execution_time`) | Shorten the lookback window (e.g. 252→120 trading days) or reduce symbol count, retry once; still timing out → degrade. |
| `matrix[a][b] == null` (insufficient shared trading days, correlation uncomputable) | Mark the pair `corr: null`; do not pad with zeros; drop from candidates and note "insufficient data". |
| No pair with abs(corr) ≥ threshold | Do not STOP; lower the threshold (e.g. 0.8→0.7), retry once; still none → report "no high-corr pair this run". |
| `get_ohlcv` `count == 0` (a leg has no historical bars) | Drop the pair; mark "leg history missing"; do not forward-fill. |
| `polygon-news-mcp` returns 401/403 | STOP, ask the user to check `POLYGON_API_KEY`. |
| `polygon-news-mcp` returns 429 | Wait 60s, retry once; STOP if it happens again. |
| `get_news_sentiment_aggregate` a leg `article_count == 0` | Mark that leg "no news in 7 days"; treat divergence conservatively as "no divergence". |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP, ask the user to run `auth login_flow`. |
| Any token / credential expired (schwab / polygon / CH) | STOP and report which source failed + how to fix; **do not fabricate data**. |
| `gh repo view` fails / repo is not private | **STOP and refuse to write**; do not bypass. |
| `research/correlation-pairs-YYYY-MM-DD.md` already exists today | Ask whether to overwrite; default skip and tell the user. |

## Idempotency

| Repeat run | Side effects |
| --- | --- |
| At most once per day | Writes `research/correlation-pairs-YYYY-MM-DD.md`; one new file per day; asks before overwrite (default skip). |
| Different day | One new file per day; filename carries date, naturally isolated. |
| CH-unavailable degraded run | Writes today's file only if the user accepts "no correlation matrix"; frontmatter marked `clickhouse: unavailable`; otherwise nothing is written. |

## See also

- Sibling playbooks:
  - `playbooks/factor-screen-deep-dive.md` (full-market factor screen + multi-source research)
  - `playbooks/shakeout-with-news.md` (shakeout signal + news sentiment)
  - `playbooks/earnings-preview.md` (pre-earnings IV-rank-aware positioning brief)
- clickhouse-mcp: [kevinkda/clickhouse-mcp](https://github.com/kevinkda/clickhouse-mcp)
  (7 read-only tools; `get_correlation_matrix` computes Pearson/Spearman in Python)
- `stock-personal/docs/sprints/usa-clickhouse-quant-integration-plan.md §3`:
  source of this playbook's quant use cases (multi-symbol correlation matrix P0 /
  pairs-trading screen P1).
