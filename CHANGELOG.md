# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.1] - 2026-06-15

### Changed

- **Playbook cache-gate is now conditional on `SCHWAB_CACHE_ENABLED`**
  (`shakeout-with-news`, `earnings-preview`, zh + en). Following the
  `schwab-marketdata-mcp` change that makes the DuckDB cache opt-in
  (default-disabled), the previously hard "cache hit_rate ≥ 30%"
  acceptance criteria would have failed every run. Each affected
  playbook now: (1) documents the `export SCHWAB_CACHE_ENABLED=true`
  opt-in in its pre-flight prelude, and (2) only verifies the hit-rate
  gate when the cache is actually enabled — otherwise the item is
  skipped and the report frontmatter is marked `cache: disabled`.
  `SKILL.md` idempotency table wording synced accordingly. Docs-only,
  user-facing behavior clarification.

## [0.3.0] - 2026-05-29

### Added

- **`earnings-preview` playbook** (zh + en, 6-step + 8 AC + rollback +
  10 failure modes, `playbooks/earnings-preview.md`): IV-rank-aware
  pre-earnings positioning brief. Combines schwab
  `get_iv_percentile` (v0.4 P1/C ATM-IV percentile rank), sec-edgar
  `get_8k_with_items(item_codes=["2.02"])` (historical earnings filings
  → next-earnings-date inference + per-quarter realized move), polygon
  `get_news_sentiment_aggregate(window="7d")` (pre-earnings news
  flow), and schwab `get_price_history` (30 trading days OHLCV → 30d
  high/low + ATR(14)). Generates a 1-page brief at
  `research/earnings-preview-{TICKER}-YYYY-MM-DD.md` (≤ 300 lines)
  with an IV-rank × sentiment cross-table that maps to one of
  `long_iv_straddle` / `directional_long_only` / `directional_short_only`
  / `sell_iv_condor` / `no_action` / `insufficient_data` recommendations.
  All recommendations are **draft signals**, not trade orders;
  brief tagged `low confidence` when `sample_count_below_30` warning
  is present on IV percentile.
- **SKILL.md (zh + en)**: `playbooks` selection table now includes
  `earnings-preview` row, and the trigger keywords list covers
  `"earnings preview"` / `"财报前瞻"` / `"earnings positioning"` /
  `"财报要关注什么"`. Idempotency table adds `earnings-preview` row
  (at most once per ticker per day, isolated by ticker + date).

### Changed

- `schwab-marketdata-mcp` `version_range` bumped from `>=0.3,<0.4` to
  `>=0.4,<0.5`. The `earnings-preview` playbook depends on the
  `get_iv_percentile` tool (P1/C, shipped in `schwab-marketdata-mcp
  v0.4.0`), which is not present in the `0.3.x` line. Activation
  handshake step 1 now requires the `0.4.x` server.
- SKILL.md (zh + en) `description.Triggers on` line extended with
  `"earnings preview"` / `"财报前瞻"`.

### Compatibility

- Requires `schwab-marketdata-mcp >=0.4,<0.5` +
  `sec-edgar-mcp >=0.2.1,<0.3` +
  `polygon-news-mcp >=0.2,<0.3`.
- Older `schwab-marketdata-mcp 0.3.x` users will fail the Activation
  handshake — `get_iv_percentile` is unavailable, so the
  `earnings-preview` playbook cannot run.

## [0.2.0] - 2026-05-25

### Changed

- SKILL.md handshake step 2.5 upgraded from string-blacklist to
  server-side probe (R7 third layer). Step 2.5 now reads the
  `sec_ua_reachable.status` field exposed by `sec-edgar-mcp v0.2.1+`
  (currently HEAD on main pre-release) and gates on `ACCEPTED` /
  `REJECTED_HTML_403` / `UNCONFIGURED` / `TIMEOUT` / `NETWORK_ERROR`,
  rather than substring-matching `users.noreply.github.com` on the
  raw UA. Server-side probe is authoritative — it catches "email is
  well-formed but SEC IP deny-listed it" cases that no client-side
  string check can detect. Updated in both zh and en SKILL.md.
- Bumped `sec-edgar-mcp` `version_range` to `>=0.2.1,<0.3` to enforce
  the server-side probe field availability at handshake time.

### Fixed

- SKILL.md handshake step 2.5: block noreply UA (R7). SEC EDGAR
  fair-use policy returns 403 for User-Agent emails ending in
  `users.noreply.github.com` (treated as undeclared automated tool).
  Activation handshake now STOPs early instead of letting PB-3 fail
  mid-execution. Discovered during 2026-05-25 PB-3 validation; root
  cause: `health_check` `user_agent_configured=true` only checks
  local env presence, not SEC-side acceptance.

## [0.1.0] - 2026-05-24

### Added

- Initial public release of `stock-research-skill`, a Cursor / Claude
  Code skill orchestrating 3 MCP servers (schwab-marketdata-mcp +
  sec-edgar-mcp + polygon-news-mcp) into multi-step equity research
  playbooks.
- **`shakeout-with-news` playbook** (zh + en, 6-step + 8 AC + rollback +
  10 failure modes): triggers schwab shakeout v2 detection, augments
  with polygon news sentiment aggregate, writes
  `research/shakeout-news-YYYY-MM-DD.md` to stock-personal.
- **`insider-alert` playbook** (zh + en, 8-step + 8 AC + rollback): scans
  watchlist via sec-edgar Form 4 XBRL, joins polygon news + schwab
  quote for cross-validation, writes
  `research/insider-alert-YYYY-MM-DD.md`.
- Bilingual SKILL.md (`stock-research` / `stock-research-en`) with
  Activation handshake (3 × `health_check` + git/gh repo private gate)
  and playbook selection table.
- Bilingual README + CONTRIBUTING + docs/REGISTER + docs/RELEASE.
- Dependabot config for GitHub Actions weekly updates.
- Issue + PR templates with bilingual update + acceptance criteria
  reminders.

### Compatibility

- Requires `schwab-marketdata-mcp >=0.3,<0.4` +
  `sec-edgar-mcp >=0.2,<0.3` +
  `polygon-news-mcp >=0.2,<0.3`.
- Falling back to lower minor versions will fail the Activation
  handshake.

[Unreleased]: https://github.com/kevinkda/stock-research-skill/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/kevinkda/stock-research-skill/releases/tag/v0.3.0
[0.2.0]: https://github.com/kevinkda/stock-research-skill/releases/tag/v0.2.0
[0.1.0]: https://github.com/kevinkda/stock-research-skill/releases/tag/v0.1.0
