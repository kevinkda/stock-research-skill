# stock-research-skill

[English](./README.md) | [简体中文](./README_zh.md)

![License](https://img.shields.io/github/license/kevinkda/stock-research-skill)
![Translation](https://img.shields.io/badge/i18n-EN%20%2B%20zh--CN-blue)
![Skills](https://img.shields.io/badge/skills-2-blue)
![Release](https://img.shields.io/github/v/release/kevinkda/stock-research-skill)
![Releases](https://img.shields.io/github/release-date/kevinkda/stock-research-skill?label=last%20release)

A Cursor / Claude Code **skill pack** that orchestrates **three MCP servers**
— [`schwab-marketdata-mcp`](https://github.com/kevinkda/schwab-marketdata-mcp),
[`sec-edgar-mcp`](https://github.com/kevinkda/sec-edgar-mcp), and
[`polygon-news-mcp`](https://github.com/kevinkda/polygon-news-mcp) — into
multi-step equity-research playbooks for the
[`kevinkda/stock-personal`](https://github.com/kevinkda/stock-personal)
investment workflow.

This skill is **read-only documentation**; the actual API traffic is owned
by the three MCP servers above.

---

## Overview

This repository ships **one cross-MCP skill** in two language variants
(简体中文 primary + English mirror), so an agent can be steered to whichever
prose language fits the user's request:

- **`stock-research`** (Chinese primary) — multi-step playbooks that join
  shakeout signals (schwab) with news sentiment (polygon) and insider
  filings (sec-edgar).
- **`stock-research-en`** (English mirror) — same scope, English prose.

Both skills:

- Share the same activation handshake (3 × `health_check` + git/gh
  private-repo gate).
- Share the same governance rules (private-repo-only writes; never push
  to `main`; no force-push; per-MCP version range gating).
- Differ only in prose language and the `language_directive` frontmatter
  field.

---

## Skill variants

| Skill | Language | When to use |
| --- | --- | --- |
| [`stock-research`](stock-research/SKILL.md) | 简体中文 (primary) | Cross-MCP equity research playbooks (`shakeout-with-news`, `insider-alert`). |
| [`stock-research-en`](stock-research-en/SKILL.md) | English (mirror) | Same scope as `stock-research`, English prose. |

> Use **stock-research** for "shakeout 配新闻", "扫 watchlist 内部人",
> "shakeout-with-news", "insider-alert".
> If the user only needs a single MCP server, prefer the per-server skill
> packs ([`schwab-marketdata-skill`](https://github.com/kevinkda/schwab-marketdata-skill),
> future `sec-edgar-skill` / `polygon-news-skill`).

---

## Compatibility with the MCP servers

| This skill repo | Compatible MCP servers |
| --- | --- |
| `v0.1.x` | `schwab-marketdata-mcp >=0.3,<0.4` + `sec-edgar-mcp >=0.2,<0.3` + `polygon-news-mcp >=0.2,<0.3` |

The version ranges are encoded in each `SKILL.md`'s `mcp_dependencies`
frontmatter. The activation handshake calls `health_check()` on every
server and refuses to continue if any `server_version` falls outside its
range.

---

## Installation

The exact mechanism depends on your client version. Typical layouts:

### Cursor

Cursor discovers user-level skills under `~/.cursor/skills/` plus any
directory the user adds via Settings → Skills. Symlink (or copy) the
folders you want — install the Chinese primary, the English mirror, or
both:

```bash
# Chinese primary (default for this repo)
ln -s "$(pwd)/stock-research"     ~/.cursor/skills/stock-research

# English mirror (optional; install in addition or instead)
ln -s "$(pwd)/stock-research-en"  ~/.cursor/skills/stock-research-en
```

### Claude Code

Claude Code picks up `~/.claude/skills/<name>/SKILL.md`. The same symlink
approach works:

```bash
ln -s "$(pwd)/stock-research"     ~/.claude/skills/stock-research
ln -s "$(pwd)/stock-research-en"  ~/.claude/skills/stock-research-en
```

> **Prerequisite**: all three companion MCP servers must already be
> registered. See each server's `docs/REGISTER.md`:
>
> - [schwab-marketdata-mcp/docs/REGISTER.md](https://github.com/kevinkda/schwab-marketdata-mcp/blob/main/docs/REGISTER.md)
> - [sec-edgar-mcp/docs/REGISTER.md](https://github.com/kevinkda/sec-edgar-mcp/blob/main/docs/REGISTER.md)
> - [polygon-news-mcp/docs/REGISTER.md](https://github.com/kevinkda/polygon-news-mcp/blob/main/docs/REGISTER.md)

See also [`docs/REGISTER.md`](docs/REGISTER.md) in this repo for the
combined activation checklist.

---

## License

MIT License — see [LICENSE](./LICENSE).

---

## Acknowledgements

This skill pack is the cross-MCP companion to:

- **[schwab-marketdata-mcp](https://github.com/kevinkda/schwab-marketdata-mcp)** —
  read-only Schwab Market Data MCP server (12 tools, OAuth, DuckDB
  cache).
- **[sec-edgar-mcp](https://github.com/kevinkda/sec-edgar-mcp)** —
  read-only SEC EDGAR MCP server (filings, Form 4 / Form 13F XBRL).
- **[polygon-news-mcp](https://github.com/kevinkda/polygon-news-mcp)** —
  read-only Polygon news MCP server (ticker news, sentiment aggregate).

The skill markdown design pattern is inspired by the
[`schwab-marketdata-skill`](https://github.com/kevinkda/schwab-marketdata-skill)
companion repo and Anthropic's published skill packs.

This project is **not affiliated with, endorsed by, or sponsored by**
Charles Schwab Corporation, the U.S. Securities and Exchange Commission,
or Polygon.io. Each MCP server independently complies with its upstream
provider's Terms of Service. Schwab Market Data is non-redistributable;
all derived markdown stays in the private `kevinkda/stock-personal` repo.

---

## See also

- [stock-personal](https://github.com/kevinkda/stock-personal) — private
  investment journal target repo (this skill writes to its `research/`
  directory only).
- [schwab-marketdata-skill](https://github.com/kevinkda/schwab-marketdata-skill)
  — single-MCP playbooks (`shakeout-analysis-v2`, `voo-qqq-tracker`,
  `watchlist-snapshot`, `summary-md-refresh`, `option-chain-research`).
