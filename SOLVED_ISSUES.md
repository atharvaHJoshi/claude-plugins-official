# claude-plugins-official — Audit, Issues & Fixes

A repo-wide audit of `anthropics/claude-plugins-official`: what was found, how each
issue was verified, what was fixed, and how the fix was validated.

- **Repository:** https://github.com/anthropics/claude-plugins-official
- **Audited revision:** `main` @ `ea10b213` (bumped plugin SHAs, Oct 2026)
- **Working branch:** `fix/marketplace-audit-and-ci-fixes`
- **Method:** upstream validator invariants I1–I11 replicated locally, the official
  `claude plugin validate` CLI (v2.1.302) run against the marketplace and plugins,
  the repo's own frontmatter/license checks run over the full tree, and the
  SchemaStore marketplace schema validated with `ajv`. Extra cross-checks
  (description drift, orphan dirs, `skills`/`strict` shape, CLI advice) round out
  the coverage; every flag they raised is accounted for under
  "Extra cross-checks" below.

## At a glance

- **Result:** 9 findings — 8 fixed, 1 documented as a deliberate non-change; all
  verification checks green (see Verification Record).
- **Patch:** 4 commits on
  [`fix/marketplace-audit-and-ci-fixes`](https://github.com/atharvaHJoshi/claude-plugins-official/tree/fix/marketplace-audit-and-ci-fixes)
  ([compare view](https://github.com/atharvaHJoshi/claude-plugins-official/compare/main...fix/marketplace-audit-and-ci-fixes)).
- **Upstream report:** [issue #6770](https://github.com/anthropics/claude-plugins-official/issues/6770)
  — the two frontmatter bugs, with repro and suggested patches.
- **Upstream PR:** [#6771](https://github.com/anthropics/claude-plugins-official/pull/6771)
  — opened for the reviewable diff, auto-closed by `close-external-prs.yml` as
  expected (non-member PRs are closed by policy; the issue tracker is the
  accepted non-member channel).
- **Headline numbers:** frontmatter full-tree **4 errors → 0**; replicated
  I1–I11 **0 violations**; changed plugin dirs **14/14 pass**; marketplace CLI
  **29 → 27 warnings** (all `version` advice).

---

## Summary of Issues

| # | Issue | Type | Severity | Location | Status |
|---|-------|------|----------|----------|--------|
| [1](#1-i8--14-vendored-plugins-ship-no-pluginjson) | 14 vendored plugins ship no `.claude-plugin/plugin.json` (invariant **I8**) | Manifest data | High | `plugins/*-lsp`, `plugins/receipts`, `plugins/session-report` + 12 marketplace entries | ✅ Fixed |
| [2](#2-i1--templatesplugins-array-is-out-of-alpha-order) | `plugins[]` out of alpha order — 10 entries misplaced (invariant **I1**) | Manifest data | Medium | `.claude-plugin/marketplace.json` | ✅ Fixed |
| [3](#3-frontmatter-ci-validates-nothing-gh-pr-diff---diff-filter-is-not-a-gh-flag) | Frontmatter CI validates nothing (`gh pr diff --diff-filter` is not a gh flag) | CI bug | High | `.github/workflows/validate-frontmatter.yml` | ✅ Fixed |
| [4](#4-frontmatter-validator-corrupts-yaml-block-scalars) | Frontmatter validator corrupts YAML block scalars (`description: \|`) | CI bug | High | `.github/scripts/validate-frontmatter.ts` | ✅ Fixed |
| [5](#5-revert-failed-bumps-workflow-can-never-fire) | `revert-failed-bumps.yml` can never fire since per-entry bump mode | Dead CI | Medium | `.github/workflows/revert-failed-bumps.yml`, `.github/workflows/scan-plugins.yml` | ✅ Fixed |
| [6](#6-discover_bumpspy-is-unreferenced-dead-code) | `.github/scripts/discover_bumps.py` unreferenced since #1762 | Dead code | Low | `.github/scripts/discover_bumps.py` | ✅ Fixed |
| [7](#7-marketplace-schema-url-404s) | Marketplace `$schema` URL returns 404 | Metadata | Low | `.claude-plugin/marketplace.json` | ✅ Fixed |
| [8](#8-cli-advice-messaging-plugins-have-no-author-in-pluginjson) | 4 messaging plugins have no `author` in `plugin.json` | Manifest data | Low | `external_plugins/{discord,fakechat,imessage,telegram}` | ✅ Fixed |
| [9](#9-cli-advice-25-pluginjson-lack-version--31-readmes-lack-install-line) | 25 `plugin.json` lack `version`; 31 READMEs lack the install line | Advice | Info | various | 📋 Documented — deliberately not changed (see rationale) |

---

## 1. (I8) — 14 vendored plugins ship no `plugin.json`

**Invariant:** I8 ("vendored source path exists and contains `.claude-plugin/plugin.json`").

**Affected entries (14):** `clangd-lsp`, `csharp-lsp`, `gopls-lsp`, `jdtls-lsp`,
`kotlin-lsp`, `lua-lsp`, `php-lsp`, `pyright-lsp`, `ruby-lsp`, `rust-analyzer-lsp`,
`swift-lsp`, `typescript-lsp` (all `./plugins/*-lsp`) plus `receipts` and
`session-report`.

**How it was discovered:** replicating the upstream `validate-plugins` action
invariants locally. Upstream has already acknowledged this as a known issue —
PR [#1762](https://github.com/anthropics/claude-plugins-official/pull/1762) lists
"14 vendored dirs contain no `.claude-plugin/plugin.json`" and says: *"Once these
are fixed, tighten `warn-invariants` in `validate-plugins.yml` to `\"I1 I3\"`."*

**Impact:**

- The official CLI **errors** when validating an LSP plugin folder directly
  (`claude plugin validate plugins/typescript-lsp` → `No manifest found in
  directory`). The action's step 40 validates *changed* in-repo folders, so any
  PR that touches one of these plugin directories fails CI.
- Upstream cannot turn I8 into a hard error while these exist.

**Root cause:** the 12 LSP plugins were authored as "marketplace entry is the
manifest" entries (`strict: false` + inline `lspServers`). When the I8 invariant
was adopted, the dirs never gained a real manifest. `receipts` and
`session-report` are skills-only plugins that also never gained one.

**Fix (subtle — the easy fix would silently break the LSP plugins):** the plugin
docs state that once `plugin.json` exists, **entry-level `lspServers` no longer
apply** ("Declare them in `plugin.json`"). Adding a bare manifest to the LSP
dirs would therefore silently disable every language server. The fix:

1. Add `.claude-plugin/plugin.json` to each of the 12 LSP dirs containing
   `name`, `description`, `version`, `author`, **and `lspServers` (moved from
   the entry)**.
2. Remove `lspServers` from the 12 marketplace entries (now dead) and drop the
   now-obsolete `strict: false` (it only existed to permit a manifest-less
   entry; with no entry component fields, `strict` is behaviourally irrelevant).
3. Add `.claude-plugin/plugin.json` (`name`, `description`, `author`) to
   `receipts` and `session-report`. No `version` is added on purpose — see #9.

**Verification:** `claude plugin validate` on each changed folder passes; the
replicated I8 check reports 0 violations.

## 2. (I1) — `plugins[]` array is out of alpha order

**Invariant:** I1 ("plugins[] alpha-sorted by name, case-insensitive").

**How it was discovered:** replicated I1 check (first mismatch: `adobe-for-creativity`
at index 1, where `activecampaign` belongs). A longest-increasing-subsequence
analysis shows **10** misplaced entries, not just the first:

```
index  41: blackrock-advisor-center-plugin
index 178: mlflow
index 268: stackhawk-api
index 299: build-with-wordpress
index 308: langfuse
index 310: activecampaign
index 311: hubspot-sales
index 312: dynatrace
index 313: wingspan
index 314: amazon-selling-partner
```

The last six were appended to the end of the file instead of being inserted in
sort order.

**Impact:** I1 is a warning today; it cannot be tightened, and any tooling that
relies on sort order (the upstream action reports line numbers assuming sorted
input) gives misleading locations.

**Fix:** sort `plugins[]` by `name` (case-insensitive, matching the upstream
action's `ascii_downcase` comparison). Diff = the 10 entries moving.

**Verification:** replicated I1 check reports 0 violations.

## 3. Frontmatter CI validates nothing (`gh pr diff --diff-filter` is not a gh flag)

**Location:** `.github/workflows/validate-frontmatter.yml` →
`Get changed frontmatter files`.

**Current code:**

```yaml
FILES=$(gh pr diff "$PR_NUMBER" --name-only --diff-filter=AMRC | grep -E '(agents/.*\.md|skills/.*/SKILL\.md|commands/.*\.md)$' || true)
```

**How it was discovered:** while auditing the workflow; then verified directly:

```
$ gh pr diff 5823 --name-only --diff-filter=AMRC ; echo $?
unknown flag: --diff-filter
1
```

`gh pr diff` supports `--name-only`, `--patch`, `--web`, `--exclude`, `--color`
and `--allow-escape-sequences` — never `--diff-filter` (it is not a `git diff`
passthrough). Introduced in `802464cf` ("Fix frontmatter validation to skip
deleted files").

**Timeline (git archaeology):** the workflow was added 2026-02-04 (`25617fd4`);
the same day, 18 minutes earlier, `82d04122` (Anthropic) had converted the 4
agent files to `|` block scalars to fix invalid inline YAML. `802464cf`
(2026-03-20) then blinded the workflow entirely — from Mar 20 to this audit no
PR was checked, which also hid the validator incompatibility in #4 below.

**Impact / root cause:** `gh` exits 1 before printing anything, so `FILES` is
always empty; the trailing `|| true` swallows the failure and the guard
`if: steps.changed.outputs.files != ''` skips the validation step. **Since
2026-03-20 the frontmatter job has never validated a single file** — a silent
no-op on every PR.

**Fix:** use the paginated REST files endpoint, which reports each file's
status, and exclude deletions there:

```yaml
FILES=$(gh api --paginate "repos/$REPO/pulls/$PR_NUMBER/files" \
  --jq '.[] | select(.status != "removed") | .filename' \
  | grep -E '(agents/.*\.md|skills/.*/SKILL\.md|commands/.*\.md)$' || true)
```

This preserves the original intent (skip deleted files, keep
added/modified/renamed) with a supported API instead of an unsupported flag.

## 4. Frontmatter validator corrupts YAML block scalars

**Location:** `.github/scripts/validate-frontmatter.ts` → `quoteSpecialValues()`.

**How it was discovered:** running the validator over the whole repo (the CI
gate never does, see #3): 4 files fail with a YAML parse error.

```
plugins/plugin-dev/agents/agent-creator.md      (agent)
plugins/plugin-dev/agents/plugin-validator.md   (agent)
plugins/plugin-dev/agents/skill-reviewer.md     (agent)
plugins/pr-review-toolkit/agents/code-simplifier.md (agent)
  ERROR: YAML parse failed: All mapping items must start at the same column ...
```

**Root cause:** these agents use the standard block scalar form:

```yaml
---
name: agent-creator
description: |
  Use this agent when ...
---
```

`quoteSpecialValues()` sees `|` in `YAML_SPECIAL_CHARS`, rewrites the line to
`description: "|"`, and the following indented block then fails YAML parsing.
The raw frontmatter is valid YAML (verified with the same `yaml` parser minus
the preprocessor).

**Why it went unnoticed:** `82d04122` (2026-02-04, Anthropic) introduced the `|`
form *because* inline was invalid YAML (`user:`/`assistant:` lines inside
`<example>` blocks parse as mappings). The validator landed 18 minutes later
(`25617fd4`) and cannot parse the files its workflow exists to check. None of
the 4 files were touched again before `802464cf` blinded the workflow (#3), so
the conflict never surfaced in CI. The two fixes must land together — repairing
the workflow alone turns these 4 files red.

**Impact:** any PR that adds/edits an agent/command/skill using a block scalar
description fails CI with a false positive — and the 4 existing files would fail
as soon as they are touched (which became reachable once #3 is fixed).

**Fix:** never quote block-scalar indicators (`|`, `>`, and their chomping /
indentation modifiers such as `|-`, `>+`, `|2`); they introduce a multi-line
value and quoting them breaks the document:

```ts
if (/^[|>][-+\d]*$/.test(value.trim())) { result.push(line); continue; }
```

**Verification:** full-tree run — before: 4 errors / 102 files; after: 0 errors
(see Verification Record).

## 5. `revert-failed-bumps.yml` can never fire

**Location:** `.github/workflows/revert-failed-bumps.yml`.

**How it was discovered:** cross-reading the bump pipeline. The bump workflow
switched to per-entry PR mode in `0d82eac1` / PR
[#2051](https://github.com/anthropics/claude-plugins-official/pull/2051):
"one PR per bumped plugin on branch `bump/<slug>`". `revert-failed-bumps.yml`
(batch era, PR #1913) still gates on the literal branch name:

```yaml
if: >
  ... github.event.workflow_run.head_branch == 'bump/plugin-shas'
```

and then looks up an open PR on `bump/plugin-shas`.

**Impact / root cause:** with per-entry branches, the gate never matches, the PR
lookup can never resolve, and the auto-revert safety net is dead code. The
comments in `scan-plugins.yml` still describe the old flow ("Revert Failed Bumps
workflow reacts to that by dropping the failing entries from the bump PR"),
which misleads anyone debugging a failed bump scan. Per-entry mode intentionally
keeps a failing bump in its own PR for triage — a batch-level reverter has no
role.

**Fix:** remove the workflow and the stale references in `scan-plugins.yml`
(recoverable from git history if batch mode is ever re-enabled), and delete the
orphaned `discover_bumps.py` (#6) from the replaced hand-rolled bump flow.

> **Design note for the maintainer:** this is the one judgement call in the
> patch. The alternative (rewriting the reverter for per-entry branches) changes
> triage behaviour and is out of scope for a cleanup; happy to rework if the
> team prefers keeping it.

## 6. `discover_bumps.py` is unreferenced dead code

**Location:** `.github/scripts/discover_bumps.py`.

**How it was discovered:** grep for references — none anywhere in the repo.
Git history shows the old `bump-plugin-shas.yml` invoked it
(`python3 .github/scripts/discover_bumps.py`), but PR
[#1762](https://github.com/anthropics/claude-plugins-official/pull/1762)
replaced the hand-rolled bump workflow with the shared composite action
(`bump-plugin-shas@9247660a…`). The script was left behind.

**Impact:** dead code that looks maintained; readers may think it runs nightly.

**Fix:** delete.

## 7. Marketplace `$schema` URL 404s

**Location:** `.claude-plugin/marketplace.json` → `"$schema"`.

**How it was discovered:** `curl -L` on the declared URL:

```
https://anthropic.com/claude-code/marketplace.schema.json → 404
```

The docs say `$schema` is "JSON Schema URL for editor autocomplete. Ignored at
load time", so this is an editor-experience bug: editors resolve the dead URL
and provide no autocomplete/validation.

**Fix:** point at the maintained SchemaStore schema for this exact file
(`https://json.schemastore.org/claude-code-marketplace.json`, which links to
the official docs and validates the current file cleanly — see Verification
Record).

## 8. CLI advice: messaging plugins have no `author` in `plugin.json`

`claude plugin validate` flags `discord`, `fakechat`, `imessage`, `telegram`
("No author information provided"). Their marketplace entries have no author
either. Fix: add `"author": {"name": "Anthropic", "email": "support@anthropic.com"}`
to the four manifests (display-only field, no behavioural impact). After the
fix, the CLI reports zero `author` warnings — the remaining 27 warnings are all
`version` advice (see #9).

## 9. CLI advice: 25 `plugin.json` lack `version` + 31 READMEs lack install line

Reported by `claude plugin validate` as **warnings/advice** and intentionally
**not** changed:

- **25 × `plugin.json → version`**: adding a static version is not cosmetic —
  it changes how Claude Code computes a plugin's version (content hash →
  pinned), which changes update behaviour for users. This is a maintainer
  product decision, not an audit fix. (The upstream action runs with
  `fail-on-warnings: false`, so these do not block CI.)
- **31 × README "no install line"**: the CLI suggests adding e.g.
  `/plugin install <name> --marketplace anthropics/claude-plugins-official`.
  It is explicitly "Advice (does not change the verdict)". A 31-file sweep
  would balloon this patch; recommended as a follow-up (list reproducible with
  `claude plugin validate .`).

---

## Scope reviewed and clean (no issues found)

To be explicit about the audit's coverage:

- **Invariants I2, I4, I5, I9, I10, I11:** zero violations (no duplicate names,
  all source URLs safe https, all pins 40-hex, no shell metacharacters, no
  hidden Unicode; `wordpress.com` was already migrated via `renames`).
- **`renames`:** every source slug is retired and every target resolves to a
  live entry.
- **JSON validity/schema:** `marketplace.json` and every `plugin.json` parse;
  the file validates cleanly against the SchemaStore schema with `ajv`.
- **Licenses:** every `plugins/*` has a LICENSE passing the Apache-2.0 check
  (with the documented `claude-security` exemption).
- **MCP liveness:** the 4 vendored http/sse servers (context7, github, gitlab,
  linear) all answer 401 — a PASS under the workflow's rules (auth expected).
- **`bump-tracking.json`:** all 14 `releases-only` slugs match live entries.
- **Workflow security review:** no new injection surface; the frontmatter fix
  keeps file lists out of shell interpolation; scope-guard and scan logic
  reviewed (correct CAS/guard behavior, pagination, masking).

### Extra cross-checks (audit-script findings, all accounted for)

- **Description drift (20 plugins):** 20 marketplace entries carry a blurb that
  differs slightly from their `plugin.json` description (e.g. `code-review`,
  `context7`, `discord`). The entry description is the marketplace-facing text;
  no invariant or CLI check compares the two, and every reviewed difference is
  an editorial variant, not a defect. Left unchanged (a 20-file text sweep
  would balloon the patch).
- **`box` `skills[]` without `strict:false`:** valid per the marketplace schema —
  `skills` adds directories "in addition to those in the skills/ directory",
  independent of `strict` — and the upstream repo at the pinned SHA ships a real
  `.claude-plugin/plugin.json`. CLI validates clean.
- **`plugins/example-plugin` unlisted:** intentional — the root `README.md`
  (line 24) points to it as the reference implementation. It carries a manifest
  and passes `claude plugin validate`.

---

## Verification Record

All checks were run locally on the final tree; no claim here is aspirational.

| Check | Command | Before (HEAD `ea10b213`) | After |
|---|---|---|---|
| Upstream invariants I1–I11 (replica) | `node audit.js <repo>` | I1 ×10 misplaced, I8 ×14 manifest-less dirs | **0 violations** |
| Official CLI, whole marketplace | `claude plugin validate .` | pass, 29 warnings (25 `version` + 4 `author`) + 31 README advice | pass, **27 warnings** (all `version`: 25 pre-existing + 2 new manifests) + 31 README advice |
| Official CLI, changed plugin dirs | `claude plugin validate plugins/<name>` ×14 | `typescript-lsp` → ✘ `No manifest found in directory` | 14/14 ✔ (`receipts`/`session-report` pass with the expected `version` advice) |
| Frontmatter (full tree, fixed script) | `node validate-frontmatter.ts <repo>` | 102 files, 4 errors | 102 files, **0 errors** |
| Frontmatter CI file list (#3) | `gh api --paginate …/pulls/N/files --jq '… select(.status != "removed") …'` | `gh pr diff --diff-filter=AMRC` → exit 1, empty list (silent no-op) | PR #717 → the 4 expected `.md` files; jq filter unit-checked (`removed` dropped, `modified`/`renamed` kept) |
| Licenses | repo workflow script (verbatim) | pass | pass (40 plugins) |
| SchemaStore schema | `ajv` | pass | pass (315 entries) |
| MCP URLs | `curl` probes | 4× 401 (pass) | unchanged |

## Files Changed

- **Modified:** `.claude-plugin/marketplace.json` (sort, 12 LSP entry cleanups,
  `$schema`), `.github/workflows/validate-frontmatter.yml`,
  `.github/scripts/validate-frontmatter.ts`, `.github/workflows/scan-plugins.yml`
  (stale comments), `.github/workflows/validate-plugins.yml` (tighten
  `warn-invariants` to `"I1 I3"` as directed by the in-file TODO after the I8
  cleanup), 4 × `external_plugins/*/.claude-plugin/plugin.json` (author).
- **Added:** 14 × `.claude-plugin/plugin.json` (12 LSP + receipts +
  session-report).
- **Deleted:** `.github/workflows/revert-failed-bumps.yml`,
  `.github/scripts/discover_bumps.py`.

## PR description (condensed, ready to paste)

```markdown
### Summary

Audit of `main` @ `ea10b213` (replicated I1–I11 invariants, `claude plugin validate`
v2.1.302, full-tree frontmatter/license runs, SchemaStore schema with `ajv`) found 9
issues; this branch fixes 8 across 4 commits. The 9th (CLI `version`/README advice)
is documented as a deliberate non-change.

### Fixes

1. Add missing `.claude-plugin/plugin.json` to 14 vendored plugins (I8). The 12 LSP
   entries move `lspServers` into the manifests (entry-level `lspServers` stop
   applying once a manifest exists — a bare manifest would silently disable every
   language server). `receipts`/`session-report` get manifests without `version` so
   content-hash update semantics are unchanged.
2. Restore alpha order of `plugins[]` (I1; 10 misplaced entries).
3. Repair `validate-frontmatter.yml`: `gh pr diff` has no `--diff-filter`; exit 1 +
   `|| true` made the step a no-op since 802464cf (2026-03-20). Use the paginated
   REST file list, excluding status `removed`.
4. Stop `quoteSpecialValues()` from quoting block-scalar headers (`description: |`),
   which corrupted the YAML; 4 existing agents would fail the repaired check
   otherwise.
5. Delete `revert-failed-bumps.yml` — it gates on `bump/plugin-shas`, but per-entry
   mode (#2051) opens `bump/<slug>`; the gate can never match. Stale comments in
   `scan-plugins.yml` updated.
6. Delete orphaned `discover_bumps.py` (unreferenced since #1762).
7. Point `$schema` at the maintained SchemaStore schema (current URL 404s).
8. Add `author` to the 4 messaging manifests; tighten `warn-invariants` to `"I1 I3"`
   as #1762 directs, now that I8/I11 are clean.

### Verification

| Check | Before | After |
|---|---|---|
| Replicated I1–I11 invariants | I1 ×10, I8 ×14 | 0 violations |
| `claude plugin validate .` | 29 warnings (25 version + 4 author) | 27 warnings, all version advice |
| `claude plugin validate` on changed dirs ×14 | typescript-lsp: error | 14/14 pass |
| Full-tree frontmatter run | 4 errors / 102 files | 0 errors |
| SchemaStore schema (ajv) / licenses | pass | pass |

### Risk & compatibility

No slugs, sources, pins, or descriptions change; `marketplace.json` is a pure sort +
entry cleanup with byte formatting preserved. Deleting the reverter is a judgment
call — happy to rework if you'd rather keep auto-revert behavior.
```

## About this document

`SOLVED_ISSUES.md` is the audit deliverable (working tree only) — the commits
contain the fixes, and the PR description carries a condensed version.

## Delivery

- **Branch pushed to the fork** (4 commits):
  https://github.com/atharvaHJoshi/claude-plugins-official/tree/fix/marketplace-audit-and-ci-fixes
- **This report published** for shareable review:
  https://github.com/atharvaHJoshi/claude-plugins-official/tree/docs/audit-report
- **Upstream bug report filed** for the two frontmatter bugs (the highest-value
  findings): https://github.com/anthropics/claude-plugins-official/issues/6770
- **Upstream PR opened** for the exact diff, then auto-closed by
  `close-external-prs.yml` within a minute — non-member PRs are closed by
  policy (the bot's comment points to the plugin submission form):
  https://github.com/anthropics/claude-plugins-official/pull/6771
  The branch stays ready if a member wants to upstream it.
