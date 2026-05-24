# Register `stock-research-skill` to Cursor / Claude Code

This document is the combined activation checklist for the
`stock-research-skill` repo. Because the skill orchestrates **three** MCP
servers, all three must be registered to your agent client **before** the
skill activates correctly.

## 1. Prerequisites

| Item | How to check |
| --- | --- |
| `gh` CLI ≥ 2.50 | `gh --version` |
| `gh` authenticated to `github.com` | `gh auth status` |
| 3 MCP servers registered | see each server's `docs/REGISTER.md` (links below) |
| `kevinkda/stock-personal` cloned locally at `/opt/workspace/code/kevinkda/stock-personal` | `git -C /opt/workspace/code/kevinkda/stock-personal status` |
| `gh repo view kevinkda/stock-personal --json isPrivate` returns `true` | `gh repo view kevinkda/stock-personal --json isPrivate -q .isPrivate` |

If any of these fail — fix it before activating the skill. Playbooks
will refuse to write if the private-repo gate fails.

## 2. Register the 3 companion MCP servers first

Each server has its own onboarding doc; complete them **in this order**
(later steps depend on earlier ones being healthy):

1. [`schwab-marketdata-mcp/docs/REGISTER.md`](https://github.com/kevinkda/schwab-marketdata-mcp/blob/main/docs/REGISTER.md)
   — OAuth flow + DuckDB cache initialisation.
2. [`sec-edgar-mcp/docs/REGISTER.md`](https://github.com/kevinkda/sec-edgar-mcp/blob/main/docs/REGISTER.md)
   — set `SEC_EDGAR_USER_AGENT="<name> <email>"` (SEC fair-use rule).
3. [`polygon-news-mcp/docs/REGISTER.md`](https://github.com/kevinkda/polygon-news-mcp/blob/main/docs/REGISTER.md)
   — set `POLYGON_API_KEY=<your_key>`.

After each server is registered, run:

```text
schwab-marketdata-mcp.health_check()   → overall_status == "healthy"
sec-edgar-mcp.health_check()           → user_agent_configured == true
polygon-news-mcp.health_check()        → api_key_configured == true
```

If any of the three fails, the cross-MCP playbooks will refuse to start.

## 3. Symlink this skill into the agent's skill directory

### Cursor

```bash
ln -s /opt/workspace/code/kevinkda/stock-research-skill/stock-research \
      ~/.cursor/skills/stock-research

ln -s /opt/workspace/code/kevinkda/stock-research-skill/stock-research-en \
      ~/.cursor/skills/stock-research-en

ls -la ~/.cursor/skills/ | grep stock-research
```

### Claude Code

```bash
ln -s /opt/workspace/code/kevinkda/stock-research-skill/stock-research \
      ~/.claude/skills/stock-research

ln -s /opt/workspace/code/kevinkda/stock-research-skill/stock-research-en \
      ~/.claude/skills/stock-research-en

ls -la ~/.claude/skills/ | grep stock-research
```

## 4. Restart the agent client

- **Cursor**: `Cmd/Ctrl-Shift-P → Developer: Reload Window`, or fully
  quit and relaunch.
- **Claude Code**: restart the CLI / restart the IDE.

## 5. Smoke test

Open a chat in `/opt/workspace/code/kevinkda/stock-personal/` and try:

```text
跑一次 shakeout-with-news playbook，仅扫 VOO/QQQ/SPY，写到 research/。
```

The agent should:

1. Activate `stock-research`.
2. Run all 3 `health_check()` calls.
3. Verify `gh repo view --json isPrivate` returns `true`.
4. Execute Steps 1-6 of `playbooks/shakeout-with-news.md`.
5. Commit a new file under `research/shakeout-news-YYYY-MM-DD.md` on a
   `research/cross-mcp-YYYYMMDD` branch with a `research(cross-mcp):`
   prefixed message.

If any handshake step fails, the agent must STOP and surface the
underlying error — **the playbook will not silently bypass guard
rails**.

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Skill never auto-activates | client did not pick up the symlink | Verify `~/.cursor/skills/stock-research/SKILL.md` resolves; restart the agent. |
| `health_check` fails for schwab | refresh-token expired | Re-run `auth login_flow` per `schwab-marketdata-mcp` docs. |
| `health_check` fails for sec-edgar | `SEC_EDGAR_USER_AGENT` unset | Set it in the MCP server's `.env`; restart the server. |
| `health_check` fails for polygon | `POLYGON_API_KEY` unset / 401 / 403 | Set it in the MCP server's `.env`; verify key on polygon.io. |
| `gh repo view` fails | `gh auth login` not run on this machine | Run `gh auth login --hostname github.com --git-protocol https --web` interactively. |
| Playbook STOPs with "shakeout 模型来源缺失" | `trackers/voo-qqq-tracker.md` missing in stock-personal | Sync stock-personal: `git -C /opt/workspace/code/kevinkda/stock-personal pull`. |
