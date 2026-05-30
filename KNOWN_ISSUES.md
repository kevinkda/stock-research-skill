# Known Issues

Tracked known issues and limitations for `stock-research-skill`. For
resolved issues see [CHANGELOG.md](./CHANGELOG.md).

## Open

### Depends on multiple MCP server version ranges (cross-MCP orchestrator)

This skill orchestrates `schwab-marketdata-mcp` + `sec-edgar-mcp` +
`polygon-news-mcp` into multi-step playbooks (`shakeout-with-news` PB-1,
`insider-alert` PB-3). Each playbook declares compatible MCP server
version ranges; if any installed server falls outside its range, the
Activation handshake (3 × `health_check` per server + git/gh repo gate)
is expected to fail rather than run against an incompatible surface.

### `insider-alert` (PB-3) data quality depends on `sec-edgar-mcp` health

PB-3 relies on `sec-edgar-mcp`'s Form 4 parser and a real, reachable
`SEC_EDGAR_USER_AGENT`. A placeholder UA causes SEC fair-use 403 denials
(the R7 incident); the playbook exits gracefully (`triggered_count=0`)
rather than blocking, but the signal is hollowed out. Ensure the
downstream `sec-edgar-mcp` is at ≥ v0.2.2 (R8 parser fix) and configured
with a real UA before relying on PB-3 output.

### Handshake failure needs a manual MCP host reload

`.env` edits on any orchestrated MCP server do not take effect until the
MCP host window is reloaded (`Cmd+Shift+P → Developer: Reload Window`).

## Upstream / Deferred

- **No code dependencies** — this repo ships Markdown skills only; the
  only dependabot ecosystem is `github-actions`, a no-op until workflows
  land under `.github/workflows/`.

## Resolved

See [CHANGELOG.md](./CHANGELOG.md) for the full history.
