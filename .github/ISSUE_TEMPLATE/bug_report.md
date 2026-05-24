---
name: Bug report
about: A cross-MCP playbook misbehaves, frontmatter is rejected, or markdown renders wrong.
title: "bug: <short summary>"
labels: ["bug"]
---

## Category

- [ ] **Playbook failure** — a `stock-research` playbook's steps no
      longer produce the expected markdown output.
- [ ] **SKILL frontmatter not honored** — the agent ignored
      `language_directive`, `mcp_dependencies`, `required_workspace`,
      or another frontmatter field.
- [ ] **MCP version mismatch** — one of the three companion MCP servers
      (`schwab-marketdata-mcp`, `sec-edgar-mcp`, `polygon-news-mcp`) is
      outside the declared `version_range` and the activation handshake
      did not catch it.
- [ ] **Markdown rendering issue** — a page renders incorrectly on
      GitHub or inside the agent's UI (broken table, escaped backticks,
      wrong heading nesting).
- [ ] **Stale link / reference** — a playbook links to a file that no
      longer exists or has moved (e.g. the upstream
      `voo-qqq-tracker.md §10` reference).
- [ ] **Other** — describe below.

## Affected skill

- [ ] `stock-research` (zh-CN primary)
- [ ] `stock-research-en` (English mirror)

## Affected playbook

- [ ] `shakeout-with-news.md` (PB-1)
- [ ] `insider-alert.md` (PB-3)
- [ ] (other / future)

## Environment

| Field | Value |
| ----- | ----- |
| Agent host (Cursor / Claude Code / etc.) | host name + version |
| Skill repo commit | `git rev-parse HEAD` |
| `schwab-marketdata-mcp` version | `get_server_info().server_version` |
| `sec-edgar-mcp` version | `get_server_info().server_version` |
| `polygon-news-mcp` version | `get_server_info().server_version` |
| Renderer (if rendering issue) | GitHub web / agent UI / VS Code preview |

## What you did

Quote the user prompt that activated the skill, plus the file path of
the playbook that misbehaved.

```text
<paste user prompt here>
```

## Expected behavior

What the SKILL.md / playbook says should happen.

## Actual behavior

What actually happened — a screenshot, the agent transcript, or a
quoted markdown excerpt.

## Additional context

- Did this start after a specific commit?
- Does it reproduce in the other language mirror (CN ↔ EN)?
- Which MCP server returned the unexpected payload (if applicable)?
