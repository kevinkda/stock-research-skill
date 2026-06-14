# Playbook — Earnings preview (IV-rank-aware pre-earnings positioning brief)

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/earnings-preview-{TICKER}-YYYY-MM-DD.md` (new) |
| schwab tools used | `health_check`, `get_cache_stats`, `get_iv_percentile`, `get_price_history` |
| sec-edgar tools used | `health_check`, `get_8k_with_items` |
| polygon tools used | `health_check`, `get_news_sentiment_aggregate` |
| max tool calls | ≤ 10 (schwab ≤ 4 + sec-edgar ≤ 2 + polygon ≤ 1 + 3 health_check) |
| Data freshness window | 8-K item 2.02 default `since_days=90` (covers most-recent quarter + one historical reference); news sentiment 7-day rolling; price history 30 days |
| Compliance | SEC EDGAR Form 8-K is public; polygon news sentiment and schwab IV/price are non-redistributable |
| Use case | Single-ticker, 1-page IV-rank-aware positioning brief 1–3 days before earnings (≤ 300 lines) |
| Trigger keywords | "earnings preview" / "earnings positioning" / "what to watch this earnings" / "财报前瞻" |

> **This playbook only runs inside the stock-personal repo.** If `cwd`
> is not under `${target_repo}`, switch to read-only mode: emit
> analysis to chat, **never write to any other repo**.

## Pre-flight (mandatory)

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" and
                                            server_version ∈ ">=0.4,<0.5"
                                            (need v0.4.0+ for get_iv_percentile)
2. sec-edgar-mcp.health_check()           → user_agent_configured == true and
                                            server_version ∈ ">=0.2.1,<0.3"
2.5. Verify the sec-edgar server-side UA reachability (identical to
     SKILL.md §Activation handshake step 2.5): read the
     sec_ua_reachable.status field returned by health_check():
   - ACCEPTED       → ✅ continue
   - REJECTED_HTML_403 → ❌ STOP, ask the user to change
                         SEC_EDGAR_USER_AGENT to a real reachable email
   - UNCONFIGURED   → ❌ STOP, ask the user to configure
                       SEC_EDGAR_USER_AGENT
   - TIMEOUT / NETWORK_ERROR → ⚠️ WARN, continue but flag the report:
                                "SEC probe transiently unavailable"
3. polygon-news-mcp.health_check()        → api_key_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true (else STOP; do not bypass)
6. cwd ∈ ${target_repo} subtree? If not → read-only mode (chat output only)
```

> **schwab cache is now opt-in (disabled by default)**: when not explicitly
> enabled `get_cache_stats()` returns `enabled == false` and `hit_rate_24h`
> stays 0. To route this run through the cache and relieve Schwab quota,
> `export SCHWAB_CACHE_ENABLED=true` (also accepts `1` / `yes` / `on`) before
> launching the MCP. Without it the playbook still completes normally
> (IV/price all live); the cache gate in idempotency below is **only active
> when `SCHWAB_CACHE_ENABLED=true`**.

## Steps

### Step 1 — Preview watchlist and pick the ticker

Read-only access to `${target_repo}/portfolio/watchlist.md`:

- Parse the ticker list (dedupe, trim).
- If the user passes a ticker → use it directly. If not → ask; do
  not default to scanning the entire watchlist (earnings-preview is
  a **single-ticker** playbook to avoid quota blowout).
- If the ticker is **not** in the watchlist: **accept it**, but tag
  the frontmatter with `coverage: ad-hoc` (vs `coverage: watchlist`).

### Step 2 — Fetch recent 8-K item 2.02 historical filings

Call sec-edgar to pull historical earnings filings (used to
**infer the next earnings date** + compute historical earnings move
statistics):

```text
sec-edgar-mcp.get_8k_with_items(
    cik_or_ticker=<ticker>,
    item_codes=["2.02"],   # SEC item 2.02 = Results of Operations and Financial Condition
    since_days=90,         # covers latest quarter + one prior reference
)
```

Expected fields: `accession_number`, `filing_date`, `reported_date`,
`items`, `primary_document_url`, `form` ("8-K" / "8-K/A").

**Next-earnings-date inference logic** (local, no extra SEC call):

- Take the most recent 3 item-2.02 8-K `filing_date` values; compute
  the median pairwise interval (US large caps typically ≈ 91 days).
- `next_earnings_estimated = max(filing_date) + median_interval`
- If the interval std-dev > 14 days → tag `low_confidence` and tell
  the reader "next-earnings-date inference is low confidence; cross-
  check with the IR site".

### Step 3 — Fetch IV percentile (v0.4 P1/C data)

Call the new schwab v0.4.0+ tool (**default `refresh=False`** —
serve from the `iv_history` cache and avoid triggering an option-
chain pull):

```text
schwab-marketdata-mcp.get_iv_percentile(
    underlying=<ticker>,
    expiry_bucket="30d",   # 30d ATM IV is the closest match for an earnings straddle
    lookback_days=252,     # one trading year
    refresh=False,         # cache-first; the default for batch / scheduled use
)
```

Expected fields: `underlying`, `expiry_bucket`, `current_atm_iv`,
`percentile_rank` ∈ [0, 100] | None, `sample_count`,
`lookback_start`, `lookback_end`, `warning?`.

**IV rank threshold table** (computed locally, written to report §6):

| `percentile_rank` of `current_atm_iv` in lookback | Option pricing implication | Recommended posture |
| --- | --- | --- |
| < 30 | IV cheap → market under-pricing the earnings move | **long_iv_straddle** candidate |
| 30 ≤ rank ≤ 70 | IV normal | **no_action** (do not initiate option-driven positions) |
| > 70 (especially > 90) | IV rich → market has priced-in the earnings move | **sell_iv_condor** candidate |
| `null` + `sample_count_below_30` warning | Sparse data | **insufficient_data**, label "low confidence", **no recommendation** |

> **Important disclaimer**: the table above produces a **draft
> positioning signal**, not a trade order. The user must form their
> own directional view, IV-crush risk assessment, bid-ask
> spread / liquidity check, and risk budgeting.

### Step 4 — Fetch the recent price trajectory

Call schwab to pull 30-day daily candles:

```text
schwab-marketdata-mcp.get_price_history(
    symbol=<ticker>,
    period_type="MONTH",
    period="ONE_DAY",       # ~30 trading days, matching the 30d IV bucket
    frequency_type="DAILY",
)
```

> Tool signature: trust whatever `get_server_info()` exposes. If a
> field name differs in your installed version (e.g. `period_count`
> instead of `period`), follow the live signature — **never guess
> from memory**.

Local computation only (no extra schwab call):

- 30-day high / low / close
- ATR(14) estimate (gauges "non-earnings-day" volatility)
- Distance to 52-week high / low (already on the schwab quote
  return; **reuse the watchlist cache** rather than calling quote
  again)
- Key support / resistance (recent 30-day swing-high / low + EMA21)

### Step 5 — Fetch polygon news sentiment (7-day pre-earnings)

```text
polygon-news-mcp.get_news_sentiment_aggregate(
    ticker=<ticker>,
    window="7d",   # rolling 7-day pre-earnings sentiment
)
```

Expected fields: `avg_sentiment ∈ [-1, 1]`, `positive_count`,
`negative_count`, `neutral_count`, `article_count`, `window_start`,
`window_end`, `top_articles[]` (when available — title / url /
publisher / published_utc).

**Sentiment classification**:

- `avg_sentiment ≥ +0.3` and `positive_count > negative_count` → **bullish_setup**
- `avg_sentiment ≤ -0.3` and `negative_count > positive_count` → **bearish_setup**
- Otherwise → **mixed_or_neutral**

If `article_count == 0` → flag the report "no relevant news in the
past 7 days"; **do not pad with zeros**; still produce the brief.

### Step 6 — Generate the 1-page brief

Write the following 8 sections to
`${target_repo}/research/earnings-preview-{TICKER}-YYYY-MM-DD.md`,
**total ≤ 300 lines**:

1. **Frontmatter**: `generated_at` (UTC), `ticker`, `coverage`
   (watchlist | ad-hoc), `mcp_versions` (all 3), `cache_hit_rate`
   (schwab), `next_earnings_estimated` + inference confidence,
   `iv_rank` + `iv_recommendation`, `sentiment_class`.
2. **Header**: ticker / company name / next-earnings inference (from
   Step 2) + inference method note ("based on the median interval of
   the last 3 item-2.02 8-K filings") + confidence tag.
3. **IV percentile card** (from Step 3): `current_atm_iv` /
   `percentile_rank` / `sample_count` / lookback window; show which
   row of the §3 threshold table fires + recommended posture.
4. **30-day price trajectory** (from Step 4): 30-day high / low /
   close / ATR(14) / distance-to-52w / key support-resistance.
   **Do not draw an ASCII candlestick chart** (keeps the report ≤
   300 lines).
5. **7-day news sentiment** (from Step 5): `avg_sentiment` / 3-class
   counts / sentiment label; up to 5 top articles cited (title +
   url + publisher + published_utc; **do not reproduce article
   bodies**). If `article_count = 0` → flag "no relevant news in
   the past 7 days".
6. **Historical earnings-move statistics** (cross from Step 2 +
   Step 4):
   - From the item-2.02 8-K list, take the most recent 3
     `filing_date` values.
   - Use price_history to compute close-to-close change at
     `filing_date ± 1 trading day` (absolute % + sign).
   - Render the table:
     `| period | filing_date | T-1 close | T+1 close | move % |`
   - Compute the median of the 3 move % values → compare with
     today's IV-implied move
     (`implied_move ≈ current_atm_iv × sqrt(1/365) × 100%`).
7. **Recommended posture** (combining §3 IV rank + §5 sentiment) —
   pick exactly one row from:

   | IV rank \ Sentiment | bullish_setup | bearish_setup | mixed_or_neutral |
   | --- | --- | --- | --- |
   | < 30 | long_iv_straddle + bullish lean | long_iv_straddle + bearish lean | long_iv_straddle |
   | 30–70 | directional_long_only | directional_short_only | no_action |
   | > 70 | sell_iv_condor + wider call wing | sell_iv_condor + wider put wing | sell_iv_condor |
   | null (sparse) | insufficient_data |  |  |

   **Repeat the disclaimer**: the cell above is a **draft signal**,
   not a trade order; the user owns directional view + position
   size + risk budget.

8. **Data provenance & limits**: one-line footnote per source →
   - IV rank: schwab `get_iv_percentile` (cache hit / miss).
   - Historical 8-Ks: sec-edgar `get_8k_with_items` (public EDGAR).
   - News sentiment: polygon `get_news_sentiment_aggregate` (non-
     redistributable).
   - Prices: schwab `get_price_history` (non-redistributable).
   - Next earnings date: locally inferred — **not** an IR official
     announcement; confidence = ...
   - Schwab Market Data non-redistribution clause (mirror the
     shakeout-with-news wording).

### Step 7 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 6 already wrote the file
# File name uses uppercase TICKER to match SEC ticker casing
git add research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): earnings-preview <TICKER> $(date -u +%Y-%m-%d)"
# DO NOT push --force.  Plain push to the research branch is enough.
```

## Acceptance criteria

After completion verify each item (run the command and confirm
output, then tick):

- [ ] **Activation handshake 6/6 PASS**: all 6 pre-flight steps
      captured (chat context); any failure must STOP, not degrade.
- [ ] **IV percentile data fetched successfully**: `get_iv_percentile`
      returns a non-empty response; if `percentile_rank` is None
      it **must** carry `sample_count_below_30` and the report
      must label "low confidence" — never silent.
- [ ] **8-K item 2.02 fallback returns ≥ 1 row**: `get_8k_with_items`
      `count ≥ 1`. `count == 0` means the ticker has been listed
      < 1 quarter — **STOP** without writing the report.
- [ ] **polygon sentiment ≥ 5 articles**:
      `get_news_sentiment_aggregate` `article_count ≥ 5`. < 5
      flags "low news coverage" but does **not** STOP.
- [ ] **30-day price trajectory complete**: `get_price_history`
      returns `candles` count ≥ 20 (30 trading days minus up to
      10 holiday gaps).
- [ ] **Report ≤ 300 lines**:
      `wc -l ${target_repo}/research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md`
      is ≤ 300; if exceeded, **auto-truncate the §4 prose** and
      append a WARN at the bottom.
- [ ] **Commit message starts with `research(cross-mcp):`**:
      `git -C ${target_repo} log -1 --format="%s"` matches.
- [ ] **Provenance section footnotes every figure**: §8 must list
      4 tools (`get_iv_percentile` / `get_8k_with_items` /
      `get_news_sentiment_aggregate` / `get_price_history`), and
      every quoted figure (IV rank / sentiment / 8-K count /
      30d high-low) must footnote its source tool at least once.

## Rollback

```bash
cd ${target_repo}
# Committed but not yet pushed → reset --soft to amend then push
git reset --soft HEAD~1   # undo the commit, keep working tree
git restore research/earnings-preview-<TICKER>-$(date +%Y-%m-%d).md

# Already pushed but the report is wrong → use git revert (preserves
# audit trail; never force-push)
git revert <hash>
git push origin <branch>   # NOT --force
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `get_iv_percentile` `sample_count < 30` | WARN, do not STOP; `percentile_rank=None`; report labels "low confidence (sample_count=N)"; recommendation degrades to `insufficient_data`. |
| `get_8k_with_items` `count == 0` (no item 2.02 in 90 d) | **STOP**: ticker has likely been listed < 1 quarter or SEC index is lagging; do not back-derive the earnings date from polygon news. |
| `get_news_sentiment_aggregate` all `sentiment=neutral` with `article_count >= 5` | Do not STOP; §5 labels `mixed_or_neutral`; §7 still picks the mixed-column signal. |
| `get_price_history` 401 / `SchwabAuthError(reason="refresh_token_expired")` | **STOP**, ask the user to run `uv run python -m schwab_marketdata_mcp.auth login_flow`. |
| `sec-edgar-mcp` 403 / `sec_ua_reachable.status == REJECTED_HTML_403` | **STOP**, ask the user to fix `SEC_EDGAR_USER_AGENT` (refer to SKILL.md handshake step 2.5). |
| Network timeout (any MCP) | Retry once; if it fails again → STOP; **never fabricate data**. |
| Activation handshake fails any step | **STOP**, do not retry; tell the user which step failed and how to fix it. |
| After generation `wc -l > 300` | Auto-truncate §4 prose paragraphs (keep the table); append `> ⚠️ Report truncated to fit 300-line limit; full data in commit metadata.` at the bottom. |
| Ticker is not in the watchlist | **Accept**, frontmatter `coverage: ad-hoc`; do not STOP (ad-hoc lookup is a legitimate use case). |
| schwab cache `hit_rate < 0.3` (stale ≥ 24 h, only when `SCHWAB_CACHE_ENABLED=true`) | WARN, do not STOP; frontmatter `cache_freshness: stale`; suggest re-running with `refresh=True` next time (mind quota). Not applicable when cache is disabled (the default). |

## Idempotency

| Repeat run | Side effects |
| --- | --- |
| Same ticker, same day, ≤ 1 run (cache hit_rate ≥ 30 % gate applies only when cache enabled; no gate when cache disabled) | Writes `research/earnings-preview-{TICKER}-YYYY-MM-DD.md`; if the same-name file exists, ask whether to overwrite (default skip). |
| Same ticker, different day | One new file per day; the filename's ticker + date give natural isolation. |

## See also

- Sister playbooks in this skill:
  - `playbooks/shakeout-with-news.md` (shakeout signal + news sentiment overlay)
  - `playbooks/insider-alert.md` (Form 4 anomaly alert)
- `stock-personal/docs/STRATEGY.md §1 Q1`: where the option-chain
  edge / IV rank fits in the strategy thesis.
- `schwab-marketdata-mcp/CHANGELOG.md §0.4.0`: the v0.4 P1/C
  `get_iv_percentile` spec (refresh path, `iv_history` materialised
  table, sample-count threshold of 30).
- `sec-edgar-mcp` README §`get_8k_with_items`: SEC item-code
  reference (1.01 / 2.02 / 5.02 / 7.01 / 9.01 etc.).
- `stock-personal/docs/sprints/v0.5-roadmap.md §2 V5-C`: source of
  this playbook's sprint acceptance criteria.
