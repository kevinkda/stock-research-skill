# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

[Unreleased]: https://github.com/kevinkda/stock-research-skill/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/kevinkda/stock-research-skill/releases/tag/v0.1.0
