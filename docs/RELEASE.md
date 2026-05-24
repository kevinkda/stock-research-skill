# Release Process — stock-research-skill

This document describes the end-to-end release process for the
`stock-research-skill` repository, which ships **Skill packages** consumed
by Cursor, Claude Code, Kiro CLI, Cline, Roo Code, and other
skills-compatible agents.

A "release" of this repo is a tagged, immutable snapshot that pairs with
matching versions of three companion MCP servers
([`schwab-marketdata-mcp`](https://github.com/kevinkda/schwab-marketdata-mcp),
[`sec-edgar-mcp`](https://github.com/kevinkda/sec-edgar-mcp),
[`polygon-news-mcp`](https://github.com/kevinkda/polygon-news-mcp)).

> **Scope:** GitHub repository releases (tags + GitHub Releases UI).
> Skills are consumed directly from the GitHub tag URL by the agent
> runtime; there is no separate package registry.

---

## 1. Prerequisites

Before cutting a release, confirm **all** of the following:

| Item | Verification command |
| --- | --- |
| `gh` CLI installed (>= 2.50) | `gh --version` |
| `gh` authenticated to `github.com` | `gh auth status` |
| Working tree clean on `main` | `git status` shows nothing to commit |
| Local `main` is up-to-date with `origin/main` | `git fetch && git status -sb` |
| `markdownlint-cli2` exits 0 | `npx markdownlint-cli2 "**/*.md"` |
| All `SKILL.md` files have valid YAML frontmatter | manual check |
| EN mirror is in sync with the CN primary | manual diff |
| Each companion MCP server has a matching version released on GitHub | `gh release view vX.Y.Z --repo kevinkda/<mcp-server>` |

If any of these fail, **stop** and fix before proceeding.

---

## 2. Versioning Policy (SemVer)

The repository follows [Semantic Versioning 2.0.0](https://semver.org/)
at the **repository level** (single tag covers all skills inside).

### 2.1 When to bump

| Change type | Bump | Example |
| --- | --- | --- |
| Doc-only edit, typo, language polish, EN mirror catch-up | **patch** | `0.1.0 → 0.1.1` |
| New playbook added, new section in an existing skill | **minor** | `0.1.0 → 0.2.0` |
| Removed playbook, renamed skill folder, breaking handshake change | **major** | `0.1.0 → 1.0.0` |
| Pre-1.0 breaking change | **minor** (allowed under SemVer 0.x) | `0.1.0 → 0.2.0` |

### 2.2 Compatibility with the MCP servers

Each `SKILL.md` declares MCP-version constraints in its frontmatter:

```yaml
mcp_dependencies:
  - name: schwab-marketdata-mcp
    version_range: ">=0.3,<0.4"
  - name: sec-edgar-mcp
    version_range: ">=0.2,<0.3"
  - name: polygon-news-mcp
    version_range: ">=0.2,<0.3"
```

When releasing, **verify** each constraint still matches the live
release of that MCP server. If you publish a skill release that
requires a newer server, bump that constraint **before** tagging.

**Tag format:** `vX.Y.Z` (e.g. `v0.1.0`). Always prefixed with `v`.

---

## 3. Release Checklist

The full sequence to release **`v0.1.0`** (substitute the actual version):

### 3.1 Pre-flight

```bash
cd /opt/workspace/code/kevinkda/stock-research-skill
git checkout main
git pull --ff-only origin main
git status                           # must be clean

npx markdownlint-cli2 "**/*.md"      # must exit 0

head -25 stock-research/SKILL.md
head -25 stock-research-en/SKILL.md
```

### 3.2 Update CHANGELOG.md

Move the contents of `[Unreleased]` into a fresh
`## [X.Y.Z] - YYYY-MM-DD` section at the top, leaving `[Unreleased]`
empty for the next cycle. Append the compare/tag link footer.

### 3.3 Commit + tag

```bash
git add CHANGELOG.md
git commit -m "chore(release): vX.Y.Z"
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin main
git push origin vX.Y.Z
```

### 3.4 Create the GitHub Release

Extract the `[X.Y.Z]` section of `CHANGELOG.md` to a temporary file:

```bash
awk '/^## \[X.Y.Z\]/{flag=1; next} /^## \[/{flag=0} flag' \
    CHANGELOG.md > /tmp/stock-research-vX.Y.Z-notes.md

gh release create vX.Y.Z \
  --repo kevinkda/stock-research-skill \
  --title "vX.Y.Z — <release headline>" \
  --notes-file /tmp/stock-research-vX.Y.Z-notes.md \
  --verify-tag

gh release view vX.Y.Z --repo kevinkda/stock-research-skill --web
```

> Add `--draft` to review the page before publishing, or `--prerelease`
> for non-stable lines (e.g. `v0.2.0-rc.1`).

### 3.5 Verify

```bash
gh release list --repo kevinkda/stock-research-skill
```

Manually confirm on GitHub:

- Tag `vX.Y.Z` is present.
- Release notes render correctly.
- Source tarballs (`.tar.gz`, `.zip`) are auto-attached.
- Tarball, when extracted, contains both skill folders
  (`stock-research`, `stock-research-en`).

---

## 4. Release Notes Template

```markdown
## What's Changed

### Added
- <user-visible new playbooks or sections, in past tense>

### Changed
- <wording / structural changes>

### Fixed
- <typo, broken link, lint fixes>

### Compatibility
- Requires `schwab-marketdata-mcp` `>=0.3,<0.4`,
  `sec-edgar-mcp` `>=0.2,<0.3`,
  `polygon-news-mcp` `>=0.2,<0.3`.

## Migration

<For minor/major releases: explicit before/after snippets if a skill
name, folder, or handshake changed. Omit for pure-patch releases.>

## Acknowledgements

Thanks to <contributor handles> for issues, reviews, and language polish.

## Full Changelog

https://github.com/kevinkda/stock-research-skill/compare/<prev-tag>...vX.Y.Z
```

---

## 5. Rollback / Recovery

If a release was published in error (wrong notes, wrong commit, broken
markdown, missing skill files):

```bash
gh release delete vX.Y.Z --yes --repo kevinkda/stock-research-skill

git tag -d vX.Y.Z
git push --delete origin vX.Y.Z
```

After rollback, fix the underlying issue, **bump the patch version**
(`vX.Y.Z → vX.Y.(Z+1)`), and run the full release checklist again.
Never re-use a tag name that has been published, even if deleted.

---

## 6. Repository metadata

After the **first** release, set the GitHub repo description and topics:

```bash
gh repo edit kevinkda/stock-research-skill \
  --description "Cross-MCP equity research skill orchestrating schwab + sec-edgar + polygon for shakeout/insider/earnings playbooks." \
  --add-topic mcp \
  --add-topic cursor-skill \
  --add-topic claude-skill \
  --add-topic finance \
  --add-topic equity-research \
  --add-topic shakeout-analysis \
  --add-topic insider-trading
```

Topics persist across future releases.

---

## 7. Notes for Future Releases

- Keep `CHANGELOG.md` updated **as part of each PR**, not only at
  release time.
- When any companion MCP server bumps a minor version, update every
  `SKILL.md`'s `mcp_dependencies` constraint in the **same** PR that
  bumps this repo's version.
- Keep CN ↔ EN mirror parity: any change to a CN skill must land with
  the matching EN edit (or a clearly tracked follow-up).
- Consider automating Steps 3.3-3.4 with a `scripts/release.sh` once
  the process has run cleanly two or three times manually.
