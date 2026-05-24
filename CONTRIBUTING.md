# Contributing to stock-research-skill

Thanks for considering a contribution! This is a personal-scale **skill
pack** (a pure-markdown documentation repo) that orchestrates three
companion MCP servers
([`schwab-marketdata-mcp`](https://github.com/kevinkda/schwab-marketdata-mcp),
[`sec-edgar-mcp`](https://github.com/kevinkda/sec-edgar-mcp),
[`polygon-news-mcp`](https://github.com/kevinkda/polygon-news-mcp))
into multi-step equity-research playbooks.

## Before you start

1. Read the top-level [`README.md`](README.md) and decide which of the
   two skill variants your contribution lives in:
   `stock-research` (zh-CN, primary) or `stock-research-en` (English
   mirror).
2. Skim the matching `SKILL.md` and an existing playbook
   (`playbooks/shakeout-with-news.md` or `playbooks/insider-alert.md`)
   to match the tone, table layout, and 6-step + AC + rollback +
   failure-modes structure.
3. Check open issues / discussions to avoid duplicate work.

## Quality gate (must pass before PR)

- `npx markdownlint-cli2 "**/*.md"` must exit 0.
- Manual review of the bilingual mirror (zh + en) — every change to a
  Chinese source **must** ship with a matching English mirror edit (or
  an explicit follow-up tracked in `docs/RELEASE.md`).
- Inclusive-language self-audit (see below).

## CN ↔ EN mirror parity

The English `stock-research-en` directory is a structural mirror of the
Chinese primary `stock-research`. **Every** edit to a Chinese source
must land with one of:

- A matching edit to the English mirror (preferred).
- A placeholder English page (H1 + 1-paragraph abstract + link back to
  the Chinese source) plus a checkbox in
  [`docs/RELEASE.md`](docs/RELEASE.md) tracking the upgrade.

Do not let the two trees drift silently.

## Playbook authoring conventions

Every new playbook **must** include:

1. A header table with `target_repo`, `target_files`, `*_tools_used`,
   `max tool calls`, and a model-source / data-compliance note.
2. A `Pre-flight` block that exercises the 3 `health_check()` calls plus
   the `git remote` + `gh repo view --json isPrivate` private-repo gate.
3. Numbered `Steps` with explicit MCP tool calls (use the **live tool
   names**; reconcile against `health_check()` if uncertain — never
   guess from memory).
4. An `Acceptance criteria` checklist with at least 7 items, each
   attached to a runnable command.
5. A `Rollback` block that uses `git reset --soft` and explicitly warns
   against force-pushing `main`.
6. A `Failure modes` table covering at least 8 distinct symptoms.

## Commit message style

Follow [Conventional Commits](https://www.conventionalcommits.org/).
Examples:

- `feat(playbook): add earnings-watch playbook (zh + en)`
- `docs(skill): clarify activation handshake order`
- `chore(release): bump compatible MCP version ranges`

Subject ≤ 72 chars. Use English. Body explains *why*, not *what*.

## Branching

- `main` is the integration branch. PRs target `main`.
- For multi-PR work (e.g. translating a whole new playbook), use a topic
  branch.
- **Never force-push `main`** — the same Git Safety Protocol the
  companion MCP repos enforce.

## What contributions are welcome

- Upgrading any English-mirror placeholder page to a complete
  translation.
- New playbooks that **must** orchestrate ≥ 2 of the 3 MCP servers
  (single-MCP playbooks belong in the per-server skill packs).
- Documentation polish, broken-link fixes, table-formatting nits.
- Skill activation / governance changes — please open a discussion
  first.

## Inclusive language

Replace `master` / `blacklist` / `whitelist` etc. with `main` /
`deny list` / `allow list`. Self-audit before submitting.

## Questions?

Open a [discussion](https://github.com/kevinkda/stock-research-skill/discussions) —
issues are for bugs.

## License

By submitting a PR, you agree your contribution will be licensed under
MIT (see [LICENSE](LICENSE)).
