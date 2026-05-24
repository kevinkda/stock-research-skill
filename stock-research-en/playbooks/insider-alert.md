# Playbook — Insider alert (insider-trading anomaly scan over watchlist)

| Field | Value |
| --- | --- |
| target_repo | `/opt/workspace/code/kevinkda/stock-personal` |
| target_files | `research/insider-alert-YYYY-MM-DD.md` (new) |
| schwab tools used | `health_check`, `get_quotes` |
| sec-edgar tools used | `health_check`, `get_form4_filings` (or whatever Form 4 / insider tool the live server exposes) |
| polygon tools used | `health_check`, `ticker_news` |
| max tool calls | ≤ 20 (sec-edgar ≤ 10 + schwab ≤ 5 + polygon ≤ 5) |
| Lookback window | Default last 14 days of Form 4 filings; user may switch to 7d / 30d |
| Compliance | Uses only public SEC EDGAR Form 4 data; polygon and schwab data remain non-redistributable |

> **This playbook only runs inside the stock-personal repo.** If `cwd` is
> not under `${target_repo}`, switch to read-only mode: emit analysis to
> chat, **never write to any other repo**.

## Pre-flight (mandatory)

```text
1. schwab-marketdata-mcp.health_check()   → overall_status == "healthy" and
                                            server_version ∈ ">=0.3,<0.4"
2. sec-edgar-mcp.health_check()           → user_agent_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
3. polygon-news-mcp.health_check()        → api_key_configured == true and
                                            server_version ∈ ">=0.2,<0.3"
4. git -C ${target_repo} config --get remote.origin.url
   → must end with kevinkda/stock-personal(.git)?
5. gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate
   → must be true (else STOP; do not bypass)
6. cwd ∈ ${target_repo} subtree? If not → read-only mode (chat output only)
7. Confirm a watchlist source exists (default reads
   `${target_repo}/watchlist.md` or `${target_repo}/trackers/watchlist.md`).
   If missing → ask the user for an explicit ticker list.
```

## Steps

### Step 1 — Parse the watchlist

Read and parse the watchlist:

- Prefer `${target_repo}/watchlist.md`; fall back to
  `${target_repo}/trackers/watchlist.md`.
- Extract the ticker list (dedupe, strip whitespace, cap at 25).
- If the watchlist exceeds 25 entries, take the first 25 and note
  "truncated N entries" in the report.

### Step 2 — Fetch recent Form 4 filings per ticker

For each ticker call:

```text
sec-edgar-mcp.get_form4_filings(
    ticker=...,
    days_back=14,           # user may override 7 / 30
    transaction_codes=["P", "S", "A", "M"],
    # P = open-market purchase
    # S = open-market sale
    # A = grant / award
    # M = option exercise
)
```

Expected fields: `accession_number`, `filer_name`, `relationship`,
`transaction_date`, `transaction_code`, `transaction_shares`,
`transaction_price_per_share`, `shares_owned_after`, `is_director`,
`is_officer`, `is_ten_percent_owner`, `xbrl_url`.

> If sec-edgar-mcp exposes a different tool name (e.g. `list_form4`,
> `query_insider_transactions`), reconcile against the live tool list
> returned by `health_check()` / `get_server_info()` first; **never guess
> a tool name from memory**.

## Step 3 — Compute the anomaly score

Per ticker, compute locally (no extra SEC calls):

```text
anomaly_score = w1 * cluster_score
              + w2 * size_score
              + w3 * direction_score
              + w4 * insider_rank_score

Where:
- cluster_score: 1.0 if ≥ 3 distinct insiders trade the same direction
                 in 14 days; otherwise prorated.
- size_score:    transaction_value / 12-month median Form 4 transaction
                 value for that issuer.
- direction_score: net buy (P − S) direction (+1 buy / −1 sell / 0 mixed).
- insider_rank_score: CEO/CFO/Chairman = 1.0; other officers 0.7;
                      directors 0.5; 10% holders 0.3.
- Default weights w1..w4 = (0.35, 0.25, 0.25, 0.15); user may override.
```

A score ≥ **0.6** triggers an alert; scores < 0.6 land in the summary
table without an expanded block.

### Step 4 — Schwab quote cross-check on triggered tickers

Only for tickers that triggered in Step 3:

```text
schwab-marketdata-mcp.get_quotes(
    symbols=[<triggered_tickers>],
    fields=["QUOTE", "REGULAR"],
)
```

Record: `lastPrice`, `netChangeInDouble`, `totalVolume`, `52WeekHigh/Low`.
Compute "insider price vs current price" delta (implied unrealised P/L).

### Step 5 — Pull recent news for triggered tickers

Only for triggered tickers:

```text
polygon-news-mcp.ticker_news(
    ticker=...,
    limit=5,
    order="desc",
    sort="published_utc",
    published_utc_gte=<14d ago>,
)
```

Record the same fields as in shakeout-with-news Step 4: `title`,
`published_utc`, `publisher.name`, `article_url`, `insights[].sentiment`,
`insights[].sentiment_reasoning`.

### Step 6 — Cross-source classification

For each triggered ticker emit a single-line verdict picking one of:

- **Class A**: insider net-buy + price below recent support + un-priced
  negative news → "potential bottom" signal
- **Class B**: insider net-buy + price at new highs + bullish news →
  "management endorses trend" signal
- **Class C**: insider net-sell + price at new highs + bullish news →
  "management taking profit" signal (**not necessarily bearish** — watch
  the size)
- **Class D**: insider net-sell + price already declining + negative news
  → "high-risk distribution" signal

### Step 7 — Write the alert report

Write the following 8 sections to
`${target_repo}/research/insider-alert-YYYY-MM-DD.md`:

1. **Frontmatter**: `generated_at` (UTC), `watchlist_size`,
   `triggered_count`, `mcp_versions` (all 3), `window_days` (default 14).
2. **TL;DR**: one-liner, e.g. "3 insider alerts this week: AAPL Class B /
   GOOG Class C / TSLA Class A".
3. **Watchlist summary table**: one row per ticker + anomaly score +
   triggered y/n; sort descending by score.
4. **Per-triggered-ticker detail block**:
   - Form 4 transaction table (from Step 2).
   - Anomaly score breakdown (from Step 3).
   - Schwab quote vs insider price (from Step 4).
   - Class verdict (from Step 6).
5. **Top 5 news citations per triggered ticker** (title, url, publisher,
   timestamp only).
6. **Limitations notice**: 14-day window covers only filed Form 4
   transactions; Form 4 has a 2-business-day filing delay; this scan
   **does not replace** Section 16 compliance audit.
7. **Suggested follow-ups**: a generic action checklist per class
   (HOLD / trim / raise monitoring frequency); **does not constitute
   investment advice**.
8. **Data provenance & limits**: links to SEC EDGAR, polygon API tier,
   Schwab non-redistribution clause.

### Step 8 — Write & commit

```bash
cd ${target_repo}
if [ "$(git branch --show-current)" = "main" ]; then
    git switch -c research/cross-mcp-$(date +%Y%m%d)
fi
mkdir -p research
# Step 7 already wrote the file
git add research/insider-alert-$(date +%Y-%m-%d).md
git commit -m "research(cross-mcp): insider-alert $(date -u +%Y-%m-%d)"
# DO NOT push --force.  Plain push to the research branch is enough.
```

## Acceptance criteria

After completion verify each item (run the command and confirm output,
then tick):

- [ ] **Commit landed**: `git -C ${target_repo} log -1 --format="%H %s"`
      shows the new commit hash + a `research(cross-mcp):` prefixed
      message.
- [ ] **Today's new file under research/**:
      `ls ${target_repo}/research/insider-alert-$(date +%Y-%m-%d).md`.
- [ ] **Only research/ touched**: every path in
      `git -C ${target_repo} diff --stat HEAD~1` lives under `research/`.
- [ ] **All 3 health_checks were valid**: pre-flight transcript captured
      in chat context.
- [ ] **Report contains all 8 sections**:
      `grep -c '^##' ${target_repo}/research/insider-alert-$(date +%Y-%m-%d).md`
      is ≥ 8.
- [ ] **Each triggered ticker has all 4 sub-blocks**: Form 4 table /
      score breakdown / quote comparison / class verdict.
- [ ] **At least one Form 4 accession_number cited**:
      `grep -E 'accession[_ ]?number' ...md | wc -l ≥ 1`.
- [ ] **Limitations notice mentions the 2-business-day filing delay**:
      avoids misleading users into thinking the data is real-time.

## Rollback

```bash
cd ${target_repo}
git reset --soft HEAD~1   # undo the commit, keep working tree
# Inspect working tree, then git restore or git stash as needed
git restore research/insider-alert-$(date +%Y-%m-%d).md
# Never force-push the main branch.
```

## Failure modes

| Symptom | Action |
| --- | --- |
| Watchlist file missing | Ask the user for an explicit ticker list; do not auto-scan the entire market. |
| `sec-edgar-mcp` returns 429 (SEC fair-use throttling) | Wait 1s, retry once; STOP after a second failure; check whether the SEC user-agent is compliant. |
| `sec-edgar-mcp` user_agent unset | STOP; ask the user to set `SEC_EDGAR_USER_AGENT="<name> <email>"` in `.env`. |
| Tool names diverge from this playbook (server renamed) | Reconcile against the live `health_check()` tool list and call the actual name; **never guess from memory**. |
| `polygon-news-mcp` returns 401/403 | STOP, ask the user to check `POLYGON_API_KEY`. |
| `SchwabAuthError(reason="refresh_token_expired")` | STOP, ask the user to run `auth login_flow`. |
| `gh repo view` fails / repo is not private | **STOP and refuse to write**; do not bypass. |
| `research/insider-alert-YYYY-MM-DD.md` already exists this week | Ask whether to overwrite; default skip and tell the user (weekly idempotency). |
| A specific Form 4 row fails to parse (missing XBRL fields) | Mark that row `(parse error)`; do not block other rows; **never pad with zeros / forward-fill**. |
| All anomaly scores < 0.6 (no triggers) | Still write the report; mark TL;DR "no insider anomalies this period"; skip Step 4-6 to save quota. |
