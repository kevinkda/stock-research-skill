# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
