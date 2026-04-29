---
purpose: schema contract for the polyforge workspace doc system (4 layers: A / B / C / graduation)
audience: polyforge agents + humans maintaining this doc-repo
applies_to: this entire doc-repo (the repo with `role: doc` in workspace.yaml)
spec: https://github.com/GMISWE/GMI-marketplace/blob/main/plugins/polyforge/docs/specs/2026-04-28-polyforge-workspace-doc-system-design.md
status: scaffold + Phase 1 path migration shipped (v0.16+); ingest / lint / log skills shipping incrementally (v0.17+) — see §8
---

# wiki/SKILL.md — schema contract

This file is the contract that polyforge agents and humans editing this repo MUST follow. Violations are surfaced by `polyforge:doc-lint` once it ships (Phase 2).

> ⚠️ **Local additions not yet upstreamed** — this file has been edited beyond the polyforge plugin's `wiki-skill-template.md`. Specifically: §"Cross-repo references" and §"Immutability exception — lint-class mechanical rewrites" are local. Running `/polyforge:doc-init --force` will rewrite this file from the upstream template and back the live version up to `wiki/SKILL.md.bak-<ts>` — diff before accepting. The proper fix is to upstream these additions to the polyforge plugin's `wiki-skill-template.md`; tracked in `wiki/lessons/_raw.jsonl` (key: `local-skill-md-additions-need-upstream`).

## 4 layers — what lives where

| Layer | Dir | Author | Mutability |
|---|---|---|---|
| A. Human SoT | `architecture/`, `decisions/`, `operations/`, `talks/` | humans only | mutable, PR-reviewed |
| B-sources | `wiki/specs/`, `wiki/plans/`, `wiki/retros/` | polyforge agents | **immutable on write** (corrections = new file, mark old superseded in frontmatter) |
| B-wiki | `wiki/lessons/`, `wiki/glossary/` | polyforge agents | mutable, regenerable |
| B-meta | `wiki/index.md`, `wiki/SKILL.md` (this file) | polyforge agents | mutable |
| C. Archive | `archive/` | humans only | read-only |

**Layer A is read-only to LLMs.** Polyforge skills MUST NOT write to `architecture/`, `decisions/`, `operations/`, `talks/`, or `archive/`. Graduation from `wiki/lessons/` to `decisions/` is a manual human flow (see §Graduation).

### Immutability exception — lint-class mechanical rewrites

B-sources are immutable for **content**, not for syntax. The following narrow classes of edits are allowed **in-place** without the new-file + `superseded_by` dance, provided the commit message uses the `docs(lint):` prefix:

- **Path / link rot fixes** — e.g. cross-repo prefix corrections (`poc/...` → `<repo>/poc/...`) per §Cross-repo references; broken anchor or URL replacements; renamed-file citations
- **Formatting normalizations** — whitespace, list-marker consistency, code-fence language tags — anything that does not alter rendered semantic content
- **Typo / spelling fixes** in non-load-bearing positions (i.e. NOT in decision IDs `D-XX`, numeric values, or quoted citations)

Anything that changes a decision, alters a cited fact, adds or removes material, or shifts an interpretation is NOT lint-class and still requires the new-file flow. Reviewers MAY revert any in-place commit that crosses the line — the contract is "lint-class only".

## wiki/lessons/_raw.jsonl format

Append-only stream of insights. One JSON object per line. Schema:

```json
{
  "schema_version": 1,
  "workspace": "ieops",
  "skill": "retro",
  "type": "pitfall",
  "key": "webhook-mock-race",
  "insight": "Mocked webhook tests pass while real concurrency catches races; use httptest.Server",
  "confidence": "high",
  "source": "retro:IEBE-1543",
  "files": ["api/webhook.go"],
  "ticket": "IEBE-1543",
  "ts": "2026-04-15T08:32:00Z",
  "extra": {}
}
```

Required: `schema_version`, `workspace`, `skill`, `type`, `key`, `insight`, `confidence`, `source`, `ts`.
Optional: `files` (repo-relative paths), `ticket`, `extra` (free-form map for forward-compat).

### Closed enums

- `type` ∈ `{pattern, pitfall, preference, architecture, tool}`
- `confidence` ∈ `{low, medium, high}` (was 1–10 numeric in spec v6/v7; settled on enum in Phase 3 to remove false precision)
- `skill` ∈ `{retro, spec, plan, debug, commit, review, triage, user}` — `user` = manually logged via `doc-log`

### Append rules

1. **Never edit a line in place.** Corrections = append a new entry with the corrected insight. Lint surfaces same-`key` conflicts for human resolution.
2. **Atomic write.** Use polyforge's `atomic_write_file` helper or `flock`. Naive `>>` redirection is NOT safe under concurrent skill execution.
3. **No newlines inside JSON values.** One record per line, full stop.

### Concurrency

`doc-log` and `doc-ingest` serialise on `<doc-repo>/wiki/lessons/_raw.jsonl.lock` (sibling lock file, advisory `flock`). The lock file is created on first append and persists; you may see it in `git status` as untracked. **Add `*.lock` to `<doc-repo>/.gitignore`** to exclude these from git tracking. Suggested addition:

```
# polyforge doc-* sibling lock files (flock advisory locks; not state)
*.lock
```

## wiki/lessons/<pattern>.md (derived view)

Regenerable from `_raw.jsonl`. **Not git-tracked** — listed in `.gitignore`, regenerated by `doc-render`. Avoids diff churn from synthesis runs. One file per stable pattern with frontmatter:

```yaml
---
key: webhook-mock-race
type: pitfall
confidence: high
sources: [IEBE-1234, IEBE-1543, IEBE-1788]
last_synthesized: 2026-04-28
---
```

Synthesis threshold: ≥ N entries sharing `key`, where N = `workspace.yaml > docs.graduation.lessons_min_tickets` (default 3).

## wiki/glossary/

One markdown file per term. Polyforge agents may seed new terms during ingest; humans may edit freely. No schema beyond standard markdown.

## wiki/index.md

Catalog of patterns + glossary entries. Regenerated by `doc-ingest` when the source set changes. **Do not hand-edit during an active ingest session** — manual edits are clobbered. Edit between sessions only, or extend via `doc-ingest` directly.

## Cross-repo references

When any document in this doc-repo (Layer A or `wiki/`) cites a file that lives in **another repo** of the same polyforge workspace (typical case: a `wiki/specs/` design doc citing PoC code in a `role: primary` repo), the reference MUST use a **repo-name-prefixed path**:

```
<repo-name>/<path-from-that-repo's-root>

# example (citing PoC code in the `tether` primary repo from a spec in tether-doc)
tether/poc/go-quic-wt/step1_server.go
```

`<repo-name>` is the key under `repos:` in `<workspace-root>/.claude/workspace.yaml`.

**Why** — polyforge splits content by role (`primary` / `config` / `doc` / `reference`), so there is no longer a single "repo root" that contains everything. Bare repo-root-relative paths like `poc/go-quic-wt/step1_server.go` were valid in the pre-split monorepo era; once content is split they resolve relative to no specific repo and confuse readers about which repo to clone.

**Don't:**

- Bare repo-root-relative paths (`poc/go-quic-wt/...`) — ambiguous after split.
- Spec-file-relative paths (`../../poc/...`) — break the moment a doc moves between `wiki/specs/` and `architecture/`.
- Hand-pasted GitHub blob URLs (`https://github.com/.../blob/main/...`) — unless you need to pin an exact line range across future edits, they drift on rename / refactor and read poorly in plaintext.

`doc-lint` (Phase 2+) will flag any path-shaped string in a wiki document that matches a known repo-key-rooted directory (e.g. `poc/...` when `tether` repo has a `poc/`) and propose the prefixed form as a fix.

## Lint rules (Phase 2 `doc-lint`)

- File-existence check on `files[]` entries (warn if path no longer exists; do not error — paths legitimately rot).
- Contradiction detection on same `key` (multiple entries with conflicting insights → flag for human review).
- Orphan check on `wiki/lessons/<pattern>.md` (sources all archived → mark stale).
- Stale claim detection (`last_synthesized > 90 days` AND no new contributing entries → flag).
- **Cross-repo reference shape** (per "Cross-repo references" above): warn on bare repo-root-relative paths that match `<workspace-root>/.repo/<other-repo>/<path>` patterns; suggest the `<other-repo>/<path>` rewrite.

## Graduation (Layer B → Layer A)

A `wiki/lessons/<pattern>.md` page is **graduation-eligible** when:

- ≥ M contributing tickets (default M=5; configurable)
- average confidence ≥ `medium`
- age ≥ T days (default T=60)

Graduation is a **fully manual human flow**: open a PR that adds `decisions/<NNNN>-<topic>.md` summarizing the pattern + back-linking to `wiki/lessons/<pattern>.md`. The ADR becomes team SoT in Layer A. The lessons page stays as evidence trail.

**Polyforge does NOT auto-suggest graduation candidates** (chair verdict v7: ADR cadence is too slow for auto-hints to be signal; humans drive graduation deliberately).

## Status

| Component | Phase | Status |
|---|---|---|
| Directory scaffold (this file + dirs) | Phase 0 — `/polyforge:doc-init` | **shipped** in polyforge v0.15.0 |
| Path migration: retro / commit M5 promotion / health glob → `wiki/{retros,specs,plans}/` | Phase 1 | **shipped** in polyforge v0.16.0 (legacy `<root>/{retros,specs,plans}/` shim sunsets v0.18.0) |
| `/polyforge:doc-log`, `:doc-ingest`, `:doc-lint` | Phase 2 | **shipped** in polyforge v0.17.0 |
| `_raw.jsonl` synthesis (`<pattern>.md` regen via `doc-ingest --synthesize`) | Phase 3 | shipped v0.18.0 |
| `/polyforge:retro` end → `doc-ingest` prompt | Phase 4 | shipped v0.18.0 |
| `/polyforge:health` wiki summary line + `doc-lint` stale/contradiction reports | Phase 5 | shipped v0.18.0 |
| Graduation flow tooling | — | manual indefinitely (no auto-tooling) |

## See also

- Design spec: https://github.com/GMISWE/GMI-marketplace/blob/main/plugins/polyforge/docs/specs/2026-04-28-polyforge-workspace-doc-system-design.md
- Karpathy LLM Wiki: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
- gstack `/learn`: https://github.com/garrytan/gstack
