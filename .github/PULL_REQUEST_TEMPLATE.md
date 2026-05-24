# Pull request

## Summary

<!-- 1–3 sentences describing what changed and why. Link the issue
this closes. -->

## Type of change

- [ ] New / updated cross-MCP playbook
- [ ] CN ↔ EN mirror parity (translation upgrade or placeholder fill)
- [ ] Documentation polish only
- [ ] Tooling / CI / lint config

## Affected skill

- [ ] `stock-research` (zh-CN primary)
- [ ] `stock-research-en` (English mirror)

## Affected playbook

- [ ] `shakeout-with-news.md` (PB-1)
- [ ] `insider-alert.md` (PB-3)
- [ ] (other / new — describe below)

## Checklist

- [ ] `npx markdownlint-cli2 "**/*.md"` exits 0.
- [ ] `pre-commit run --all-files` passes.
- [ ] Conventional commit message — `docs(...)`, `chore(...)`,
      `feat(...)`, etc.
- [ ] CN ↔ EN mirror updated together (or placeholder + tracked
      follow-up in `CHANGELOG.md`).
- [ ] [`CHANGELOG.md`](../CHANGELOG.md) updated under
      `## [Unreleased]` (if user-visible).
- [ ] All `mcp_dependencies` `version_range` entries in `SKILL.md`
      checked — bumped only when one of the three companion MCP
      releases requires it.
- [ ] Inclusive-language audit — no `master` / `blacklist` /
      `whitelist` / `kill` / `abort`. Use `main` / `deny list` /
      `allow list` / `stop` instead.
- [ ] No secrets, `.env`, `token.json`, or Bearer tokens committed.
- [ ] No real Schwab / Polygon / SEC API responses committed
      (Schwab Market Data is non-redistributable).
- [ ] Playbook still verifies `gh repo view --json isPrivate` before
      writing (private-repo precheck).

## Test plan

<!-- How did you verify this change? -->

- [ ] Rendered the changed markdown on GitHub web / locally.
- [ ] If a playbook was changed, dry-ran it in a non-production
      directory (no real MCP calls required).

## Screenshots (if applicable)

<!-- Optional — preferred for rendering / table fixes. -->
