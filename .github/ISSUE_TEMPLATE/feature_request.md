---
name: Feature request
about: Suggest a new cross-MCP playbook or a language mirror.
title: "feat: <short summary>"
labels: ["enhancement"]
---

## Category

- [ ] **New cross-MCP playbook** — a multi-step workflow that
      orchestrates two or more of the companion MCP servers
      (`schwab-marketdata-mcp`, `sec-edgar-mcp`, `polygon-news-mcp`).
- [ ] **Extend an existing playbook** — add a step, a failure mode, or
      an acceptance criterion to PB-1 (`shakeout-with-news`) or PB-3
      (`insider-alert`).
- [ ] **New language mirror** beyond the existing zh-CN + English
      pair.
- [ ] **Other** — describe below.

> Out of scope here: per-MCP single-tool references. Those live in the
> dedicated companion skill repos
> (`schwab-marketdata-skill`, future `sec-edgar-skill`, future
> `polygon-news-skill`). This repo is a **routing / orchestration layer
> only**.

## Motivation

What user problem does this solve? Quote a representative user prompt
that the existing playbook content cannot answer well.

## Proposed change

- Affected files / new file paths.
- For a new playbook: list the steps the agent should perform, the
  MCP tools called per step, the output markdown it produces, and the
  target repository it writes into (must be private).

## Plan / scope alignment

- [ ] Does this respect Schwab's non-redistributable Market Data
      constraint? Any output must land in a private repo.
- [ ] Does this stay within the read-only API surface of all three
      MCP servers? (No Trader API, no SEC submission writes, no
      Polygon trade execution.)
- [ ] Does the playbook declare its `gh repo view --json isPrivate`
      precheck and refuse to write on `false`?
- [ ] Does the playbook have a documented rollback path that does
      **not** force-push `main`?

## Additional context

Link to relevant agent transcripts, related issues, or upstream MCP
server tool specifications.
