# Playbook — Shakeout with news (overlay news sentiment on shakeout signals)

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/shakeout-news-YYYY-MM-DD.md` (new) |
| schwab tools used | `health_check`, `get_cache_stats`, `get_price_history`, `get_quotes` |
| sec-edgar tools used | (this playbook does not call sec-edgar) |
| polygon tools used | `health_check`, `sentiment_aggregate`, `ticker_news` |
| max tool calls | ≤ 18 (schwab ≤ 8 + polygon ≤ 8 + 2 health checks) |
| Model source | `trackers/voo-qqq-tracker.md §10` (Tang Keyin private methodology; this playbook **does not restate** the model — it references it) |

> **This playbook only runs inside the stock-personal repo.** If `cwd` is
> not under `${target_repo}`, switch to read-only mode: emit analysis to
> chat, **never write to any other repo**.

## Pre-flight (mandatory)

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" and
                                            server_version ∈ ">=0.3,<0.4"
2. polygon-news-mcp.health_check()        → api_key_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
3. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
4. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true (else STOP; do not bypass)
5. cwd ∈ ${target_repo} subtree? If not → read-only mode (chat output only)
6. ls ${target_repo}/trackers/voo-qqq-tracker.md must exist and be readable;
   otherwise STOP and report "shakeout model source missing".
```

## Steps

### Step 1 — Pick the universe

Default scan is `["VOO", "QQQ", "SPY"]` (aligned with voo-qqq-tracker).
The user may override with a custom watchlist (≤ 5 tickers, bounded by
the max tool calls budget).

### Step 2 — Trigger the schwab shakeout detection

For each symbol call:

```text
schwab-marketdata-mcp.get_price_history(
    symbol=...,
    period_type="MONTH",
    period="THREE_DAYS",
    frequency_type="DAILY",
)
schwab-marketdata-mcp.get_quotes(
    symbols=["VIX", *symbols],
    fields=["QUOTE", "REGULAR"],
)
```

Run the §10 8-signal scan **locally** (no extra schwab calls). **Source
of truth is the live `trackers/voo-qqq-tracker.md §10` content** — this
playbook does not duplicate the thresholds or the decision matrix.

### Step 3 — Pull aggregate news sentiment

Only for symbols that **trigger a shakeout signal** in Step 2 (decision
matrix lands on HOLD/REVIEW/TRIM):

```text
polygon-news-mcp.sentiment_aggregate(
    ticker=...,
    window="7d",          # rolling 7-day sentiment
)
```

Expected fields: `avg_sentiment ∈ [-1, 1]`, `positive_count`,
`negative_count`, `neutral_count`, `article_count`, `window_start`,
`window_end`.

### Step 4 — Pull top 3 ticker news (citation only)

For each triggered symbol fetch top 3 articles for citation:

```text
polygon-news-mcp.ticker_news(
    ticker=...,
    limit=3,
    order="desc",
    sort="published_utc",
)
```

Record only: `title`, `published_utc`, `publisher.name`, `article_url`,
`insights[].sentiment` (when available), `insights[].sentiment_reasoning`.
**Do not reproduce article body text** in the report (copyright safety).

### Step 5 — Write the cross-source report

Write the following 8 sections to
`${target_repo}/research/shakeout-news-YYYY-MM-DD.md`:

1. **Frontmatter**: `generated_at` (UTC), `symbols`, `mcp_versions` (all 3),
   `cache_hit_rate` (schwab).
2. **TL;DR**: one-liner verdict, e.g. "QQQ shakeout match (7/8) + sentiment
   +0.42 → HOLD; external narrative is supportive."
3. **Per-symbol §10 8-signal table** (from Step 2; reference the model, do
   not restate it).
4. **Per-symbol news sentiment aggregate table** (avg_sentiment / counts /
   window).
5. **Sentiment × signal cross-tab judgment**: standardised conclusions for
   the 4 combinations (shakeout-hit × pos, shakeout-hit × neg, shakeout-
   reversal × pos, shakeout-reversal × neg).
6. **Top 3 news citations per symbol** (title + url + publisher + ts only).
7. **Risk callouts**: which §10.6 failure modes are active this run; 1-2
   un-priced news risks.
8. **Data provenance & limits**: links to voo-qqq-tracker.md §10, schwab
   cache hit rate, polygon API tier, Schwab non-redistribution clause.

### Step 6 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 5 already wrote the file
git add research/shakeout-news-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): shakeout-with-news $(date -u +%Y-%m-%d)"
# DO NOT push --force.  Plain push to the research branch is enough.
```

## Acceptance criteria

After completion verify each item (run the command and confirm output,
then tick):

- [ ] **Commit landed**: `git -C ${target_repo} log -1 --format="%H %s"`
      shows the new commit hash + a `research(cross-mcp):` prefixed
      message.
- [ ] **Today's new file under research/**:
      `ls ${target_repo}/research/shakeout-news-$(date +%Y-%m-%d).md`.
- [ ] **Only research/ touched**: every path in
      `git -C ${target_repo} diff --stat HEAD~1` lives under `research/`.
- [ ] **All 3 health_checks were valid**: pre-flight transcript captured
      in chat context.
- [ ] **Schwab cache hit rate ≥ 30%**: end-of-playbook
      `schwab-marketdata-mcp.get_cache_stats()` returns
      `hit_rate_24h ≥ 0.3`.
- [ ] **Report contains all 8 sections**:
      `grep -c '^##' ${target_repo}/research/shakeout-news-$(date +%Y-%m-%d).md`
      is ≥ 8.
- [ ] **No verbatim §10 copy-paste**: pick a random sentence from §10;
      `grep -F` should **not** find it in the new report.
- [ ] **News URLs are clickable**:
      `grep -E '^- \[.*\]\(https?://' ...md | wc -l ≥ 3`.

## Rollback

```bash
cd ${target_repo}
git reset --soft HEAD~1   # undo the commit, keep working tree
# Inspect working tree, then git restore or git stash as needed
git restore research/shakeout-news-$(date +%Y-%m-%d).md
# Never force-push the main branch.
```

## Failure modes

| Symptom | Action |
| --- | --- |
| `trackers/voo-qqq-tracker.md` §10 missing or truncated | **STOP**. Report "shakeout model source missing"; **never fabricate the model**. |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP, ask the user to run `auth login_flow`. |
| `SchwabRateLimitError` | Wait `retry_after_seconds`, retry once; STOP after a second failure. |
| `polygon-news-mcp` returns 401/403 | STOP, ask the user to check `POLYGON_API_KEY`. |
| `polygon-news-mcp` returns 429 | Wait 60s, retry once; STOP if it happens again. |
| `gh repo view` fails / repo is not private | **STOP and refuse to write**; do not bypass. |
| `research/shakeout-news-YYYY-MM-DD.md` already exists today | Ask whether to overwrite; default skip and tell the user. |
| VIX quote unavailable | Mark signal #7 as `N/A`, weight the decision matrix on 7 signals; **do not fabricate VIX**. |
| `sentiment_aggregate` empty (article_count == 0) | Report "no relevant news in the past 7 days"; do not pad with zeros. |
| Schwab cache lock contention (another process holds `cache.duckdb`) | Wait 5s, retry once; if still locked, set `SCHWAB_CACHE_BYPASS=1` to take the live path and note "cache locked, bypassed" in the frontmatter. |
