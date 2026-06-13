# Proposal: Auto-Trigger a Deep Companion Note When Ingesting Long Sources

**Status:** Ready for PR — all six Before-blocks verified verbatim against the v5.0.0 bundle 2026-06-10
**Co-authored-by:** Claude (Anthropic)
**Target:** `Packs/Knowledge/src/SKILL.md` and `Releases/v5.0.0/.claude/skills/Knowledge/SKILL.md` (currently byte-identical — apply the same six edits to both)
**Owner:** @asdf8675309
**Branch (when ready):** `proposal/knowledge-deep-ingest`

## Problem

`/Knowledge ingest` produces one primary note per source. That works well for short articles but reliably under-yields on long, dense sources — vendor whitepapers, research reports, multi-page PDFs, academic papers. The first pass captures the headline framework and a handful of stats; the rest of the source — sample prompts, secondary frameworks, named SKUs, customer quotes, hidden sub-playbooks — gets compressed away.

Concrete case that motivated this proposal (2026-05-08 session):

A 42-page Google Cloud whitepaper ("AI Starter Kit for Lean Teams", services.google.com PDF, ~5,940 words) was ingested into `Ideas/google-ai-starter-kit-lean-teams.md`. The first-pass note (80 lines, quality 6, seedling) captured the 4-step ladder, the ABCD prompting acronym, 7 stats, and 3 customer stories. Then the user asked: "any other wisdom to extract from that pdf? It was pretty big." A second-pass mining session surfaced **9 distinct categories of net-new content** the first pass had missed:

| Category | Net-new items |
|---|---|
| Customer stories | +2 (Altumatim eDiscovery, IT-Development telecom; one with a "1/5 the cost" claim) |
| Stats | +1 (85% SMBs recognize AI potential — paired with the 91% overload stat) |
| Hidden sub-playbook | The 4-main-step framework actually contains 5 sub-steps each → really a 4×5 = 20-step playbook |
| Best-practice rules | 4 → 8 (incl. specific model-tier guidance: Gemini 3 Pro for orchestration, 2.5 Flash for repetitive, 3 Deep Think for hardest) |
| Named SKUs | +11 (AI Studio, Customer Experience Agent Studio, Agent Engine, Idea Generation agent, NotebookLM Enterprise, Model Garden, Cloud Vision, Gemini Deep Research agent, Jira Cloud Connector, ADK Evaluation, Gen AI Evaluation Service) |
| Architecture map | The 2-axis "Everyone can build" vs "Developers have full control" agent ecosystem diagram |
| Sample prompts | +6 (copy-paste-ready prompts for email triage, vibe-coded sentiment sorter, image gen, social copy, deep research, Looker conversational analytics) |
| Use-case patterns | +12 concrete agent recipes (customer returns multi-step, supply chain coordination, lead nurturing, etc.) |
| Framing language | +6 coined terms / slogans worth borrowing ("pilot purgatory", "execution gap", etc.) |

After the second pass, the note grew **80 → 192 lines** and the status promoted seedling → developed.

The pattern is invariant: **first-pass-only ingest of long sources misses ~50% of the tactical content.** The user shouldn't have to issue a second prompt to recover it.

## Diagnosis

The current `ingest` workflow has two structural shortcomings:

1. **No depth signal.** The workflow runs the same compression on a 500-word blog post and a 42-page whitepaper. The headline note shape (Thesis / Evidence / Implications / Tensions / Context) is right-sized for the former and lossy for the latter.
2. **Fix-on-prompt instead of fix-on-detect.** A long PDF is detectable at fetch time (page count, word count, source domain, doc-type keywords). The skill currently doesn't notice. The user has to.

The minimal fix is to detect the depth signal at fetch time and emit a **second, sibling companion note** that holds what didn't fit in the primary — without changing the primary's shape. No new schema, no new tools, no new dependencies.

## Proposed fix

Add three things to the `ingest` workflow:

1. **Two new flags** — `--deep` (force deep) and `--no-deep` (suppress deep) — for explicit override.
2. **A new "Step 1.5 — Deep-pass detection"** that fires the deep pass automatically when any of: word count >5,000, PDF page count >20, doc-type keyword in title/URL (whitepaper, research report, playbook, starter kit, handbook, case study, annual report, state of, field guide, ebook), or vendor-analyst / academic source domain (services.google.com, gartner.com, mckinsey.com, deloitte.com, kpmg.com, pwc.com, idc.com, forrester.com, arxiv.org, alphaxiv.org, nature.com, science.org, acm.org).
3. **A new "Step 6 — Deep companion pass"** that produces a sibling `<slug>-deep.md` file with three-lens extraction (stats/quotes, products/entities, tactical recipes) and typed `part-of` ↔ `extends` cross-links to the primary.

The primary note keeps its current shape and length. The companion holds the long tail. Both are first-class notes parseable by the existing `KnowledgeHarvester.ts` and `KnowledgeGraph.ts` because they use the same schema.

### Why this is the minimal fix

- **Zero new schema fields.** Companion is identified by filename pattern (`<slug>-deep.md`), not by a new frontmatter field. No migration of existing notes.
- **Zero code changes required.** Verified against `KnowledgeHarvester.ts` and `KnowledgeGraph.ts`: both walk `KNOWLEDGE/<Type>/*.md` and parse the standard schema. A sibling file with the same shape is automatically picked up.
- **One file changed in the PR.** `Packs/Knowledge/src/SKILL.md`.
- **Bitter Pill compliant.** No new abstractions; the companion is just another note. The fast path stays fast for short sources.
- **Reversible.** `--no-deep` flag suppresses the new behavior unconditionally; if a maintainer wants the historical behavior back, removing the auto-trigger and the Step 6 section in `SKILL.md` is a single revert.

### Why not the alternatives

| Alternative | Rejected because |
|---|---|
| Add a `depth: deep` frontmatter field | Doubles the schema surface. Every downstream tool (harvester, graph, MOC generator, contradiction finder, MemoryRetriever) needs an update. Fails Bitter Pill. |
| Make every ingest a deep pass | Slow on short articles where the deep pass produces empty sections. Wastes tokens. The user explicitly didn't want this. |
| `--deep` flag only, no auto | Defers the problem to user discipline. The original failure mode was the user *forgetting* to flag a long source for deep pass; making it explicit-only doesn't solve the underlying gap. |
| Inline the deep content into the primary note | Bloats the primary, breaks its scannable shape, doubles its length on every long source. The companion-note pattern keeps both notes right-sized for their roles. |
| Separate skill (e.g., `DeepIngest`) | Adds a parallel surface that has to be remembered separately. Auto-trigger inside the existing workflow is the cleaner integration. |

## Edits

### `Packs/Knowledge/src/SKILL.md`

**1. Frontmatter description (L3)** — extend to mention the new behavior so it's discoverable in skill listings:

Before:
```
description: "...ingest (fetch URL or file, create primary note, ripple updates to related notes), contradictions (find conflicting claims..."
```
After:
```
description: "...ingest (fetch URL or file, create primary note, ripple updates to related notes; auto-produces companion `<slug>-deep.md` for sources >5,000 words / >20 PDF pages / vendor-analyst doc types — three-lens mining: stats+quotes, products+entities, tactical recipes — overridable with `--deep` / `--no-deep`), contradictions..."
```

Also extend the `USE WHEN` keyword list with: `deep ingest, deep dive on PDF`.

**2. `ingest` section header (L194)** — extend the usage signature to surface the new flags:

Before:
```
## ingest <url-or-file>

Ingest a source into the Knowledge Archive...

**If no argument provided:** Show usage: `/knowledge ingest <url-or-file-path>`
```
After:
```
## ingest <url-or-file> [--deep | --no-deep]

Ingest a source into the Knowledge Archive. Long or dense sources additionally produce a **companion deep-dive note** with the full mining pass.

**If no argument provided:** Show usage: `/knowledge ingest <url-or-file-path> [--deep | --no-deep]`

**Flags:**
- `--deep`: Force the companion deep-dive pass (Step 6) regardless of size or doc-type detection. Use when the auto-trigger is too conservative for a source you know is dense.
- `--no-deep`: Suppress the companion deep-dive pass even if auto-triggers fire. Use when you only want the headline seedling and don't have time for the full mining pass.
```

(Note: the live PAI install has an additional `--auto` flag from a prior in-flight feature; the upstream Pack source does not. This proposal is scoped to the deep flags only.)

**3. After Step 1 (Fetch the source)** — extend the fetch step with PDF tooling notes, then insert a new Step 1.5 before Step 2:

Before:
```
### Step 1 — Fetch the source

- **URL:** Use WebFetch to retrieve and read the content. If WebFetch fails, try `curl -sL` via Bash.
- **File path:** Use Read tool to read the local file.

Summarize the source in 2-3 sentences. Identify key entities, claims, and insights.

### Step 2 — Classify and create primary note
```
After:
```
### Step 1 — Fetch the source

- **URL:** Use WebFetch to retrieve and read the content. If WebFetch fails, try `curl -sL` via Bash.
- **File path:** Use Read tool to read the local file.
- **PDFs:** prefer `pdftotext <file> <out>` for full-text extraction; PDF page count via `pdfinfo` for the size heuristic below.

Summarize the source in 2-3 sentences. Identify key entities, claims, and insights.

### Step 1.5 — Deep-pass detection

Decide whether this ingest produces a **companion deep-dive note** in addition to the standard primary seedling. The decision is automatic unless overridden.

**Force rules (override auto-detection):**
- `--deep` flag → deep pass fires unconditionally
- `--no-deep` flag → deep pass suppressed unconditionally

**Auto-trigger rules (any one fires the deep pass):**

| Trigger | Threshold |
|---------|-----------|
| **Word count** | source body >5,000 words (use `wc -w` after fetch) |
| **PDF page count** | `pdfinfo` reports >20 pages |
| **Doc-type keyword in title or URL** | one of: `whitepaper`, `white paper`, `research report`, `playbook`, `starter kit`, `handbook`, `case study`, `annual report`, `state of`, `field guide`, `ebook`, `e-book` |
| **Vendor analyst / academic domain** | URL host matches: `services.google.com`, `cloud.google.com/.*\.pdf`, `gartner.com`, `mckinsey.com`, `mck.com`, `deloitte.com`, `kpmg.com`, `pwc.com`, `idc.com`, `forrester.com`, `arxiv.org`, `alphaxiv.org`, `nature.com`, `science.org`, `acm.org` |

If any auto-trigger fires AND `--no-deep` is NOT set, schedule the deep pass for Step 6.

If no trigger fires AND `--deep` is NOT set, skip Step 6 (fast path).

**Output the decision** so the user sees it before the work happens:

```
📥 INGEST DEPTH:
  source: <url-or-file>
  size: <N> words / <P> pages
  doc-type match: <keyword(s) hit, or "none">
  decision: deep-pass (reason: <which trigger fired>)
                  | fast-path (no triggers fired)
```

### Step 2 — Classify and create primary note
```

**4. Step 5 (Log and index)** — extend the log line to record when deep pass ran:

Before:
```
- Ripple: N notes updated, N contradictions flagged
```
After:
```
- Ripple: N notes updated, N contradictions flagged
- Deep companion: <Type>/<slug>-deep.md (created)  # only when Step 6 ran
```

**5. After Step 5** — insert a new Step 6 before the `---` separator that precedes the trailing sections:

```
### Step 6 — Deep companion pass (when triggered)

This step runs only when Step 1.5 decided `deep-pass`. It produces a **sibling companion note** at `KNOWLEDGE/<Type>/<slug>-deep.md` containing the full mining pass. The primary note stays the scannable seedling; the companion holds everything that didn't fit.

**Companion note convention:**
- **Filename:** same slug as primary plus `-deep` suffix.
- **Frontmatter:** identical schema to primary — `title`, `type`, `tags`, `created`, `updated`, `quality`, `related`, `source_url`. Title gets a "— Deep Dive" suffix; quality typically equals or is one less than primary.
- **Linking:** companion's `related:` MUST include the primary with `type: part-of`. Primary's `related:` MUST gain a typed entry pointing at the companion with `type: extends`.

**Three-lens extraction** — read the source end-to-end and pull every net-new item across these three lenses:

1. **Stats / quotes lens** — every numerical claim (with source citation), every direct quote (with attribution), every named statistic.
2. **Products / SKUs / entities lens** — every named product, service, SKU, model name, agent name, framework name, person, company.
3. **Tactical recipes lens** — every step-by-step procedure, every sample prompt, every concrete config, every named best-practice rule, every numbered playbook.

**Companion note body sections** (fixed order, populate only those with content):

`## Source`, `## Stats & Quotes`, `## Products & Entities`, `## Tactical Recipes`, `## Frameworks & Mnemonics`, `## Concrete Use-Case Patterns`, `## Framing Language Worth Borrowing`, `## Marketing Filter`.

**Anti-rules for the companion note:**
- **Do not duplicate** content already in the primary note. The companion is a delta, not a superset.
- **Do not include** vendor marketing or "why choose us" content as wisdom — flag it explicitly under Marketing Filter.
- **Source-attribute** every claim by section, page, or location.
- **Do not invent ripple updates** for the companion note — the primary already did that work; the companion only typed-links to the primary.

**After writing the companion note:**
- Append the typed `extends` entry to the primary note's `related:` array.
- Update the primary's `updated:` field.
- Update Step 5's `_log.md` entry to include the `Deep companion: ...` line.
- Re-run the MOC regen so the companion shows up.
```

**6. Gotchas section** — append three entries:

```
- **Deep companion notes are siblings, not supersets.** When Step 6 fires, the companion `<slug>-deep.md` is a *delta* — it holds what didn't fit in the primary, not a duplicate of it. The primary stays scannable; the companion stays exhaustive. Never write the same content into both.
- **The deep auto-trigger is conservative.** Auto fires only on >5,000 words, >20 PDF pages, vendor-analyst doc-type keywords, or known long-form domains. A normal blog post will go fast-path. If you hit a long source the heuristic missed, run with `--deep` once — and consider whether the heuristic should be extended.
- **Deep companion's `extends` link counts as an exception to the 2–4 outbound cap.** When Step 6 fires, the primary's `related:` gets one extra entry (typed `extends`) pointing at the companion. This exceeds the cap by one and is acceptable because the companion is structurally part of the same ingest, not an editorial choice.
```

(Note: the third gotcha references the public-Pack outbound cap of 2–4. The user's local install runs the in-flight cap-scope-to-outbound rule with 2–8; that's a separate proposal — `KnowledgeRelatedCapScopedToOutbound.md` — and the cap value here matches whichever rule is active in the merged version.)

## Rollback

Revert the six edits in `Packs/Knowledge/src/SKILL.md`. No code, no schema, no tools changed. Existing companion notes already on disk continue working — the harvester treats them as ordinary notes — but no new ones are produced. A user who wants the historical behavior simply runs every ingest with `--no-deep` until the revert lands.

## Open questions for upstream review

1. **Doc-type keyword vocabulary.** The list is empirical from one user's ingest history. Maintainer may want to broaden (`syllabus`, `prospectus`, `RFP`, `RFQ`, `reading list`) or tighten. Worth a short audit.
2. **Vendor / academic domain list.** Same concern — the list is what's been ingested locally. International equivalents (`ey.com`, `bcg.com`, `bain.com`, `ssrn.com`, `springer.com`) likely belong but aren't in the list. Maintainer call.
3. **Companion-note slug suffix.** `<slug>-deep.md` is one convention; alternatives include `<slug>.deep.md` (dot-separator), `<slug>/deep.md` (subdirectory), or a `Deep/` parallel folder. The flat-file `-deep` suffix is the lightest-weight option but worth confirming.
4. **Three-lens spec.** Stats+quotes / products+entities / tactical recipes is what the original session needed. Some sources (e.g., academic papers) might warrant a fourth lens for methodology / experimental design. Worth flagging but not required for v1.
5. **Quality cap exception clarity.** The companion's `extends` link to the primary takes the primary one entry over the outbound cap. The Gotchas entry calls this out, but the cap rule itself in the Canonical Linking Requirement section doesn't currently anticipate exceptions. Maintainer may want a one-line clarification there too.
6. **Re-running deep on an existing primary.** If a user originally fast-pathed an ingest and later wants the companion, there's no current command for "deep-only on an existing slug." A `/knowledge ingest <existing-slug> --deep-only` mode could be added but is out of scope for this PR.

## Why this belongs upstream

Missing half the content from a long source is a structural property of the fast-path-only design, not a user error. Today's session demonstrates the failure mode in a single sitting: a 42-page source produced ~50% of its tactical content on the first pass. The fix is local (one file, six edit blocks), reversible (single-flag suppression, single-file revert), and doesn't change runtime behavior of any tool — only what the workflow tells the executor to write. That's the right shape for a doctrinal upstream PR.

The three-lens extraction (stats+quotes, products+entities, tactical recipes) is the empirical recipe that recovered the 9 categories of net-new content in the Google PDF case study. Codifying it means future ingests don't have to re-derive the recipe in conversation.

## Local reference

The user's private PAI install at `~/.claude/skills/Knowledge/SKILL.md` already carries this change as of 2026-05-08. The local version is functionally identical to the proposed upstream change with two deltas:

1. The local version retains the in-flight `--auto` flag (skip-ripple-approval), which is not in the public Pack source.
2. The local version's outbound-cap reference uses 2–8 (per the user's local cap-scope-to-outbound rule, also pending upstream as `KnowledgeRelatedCapScopedToOutbound.md`); the public-Pack proposal references 2–4 to match the upstream cap.

The local copy is deliberately ahead of upstream so the user can validate the workflow before the public PR. When this proposal is accepted, the public Pack and local install will converge on the same shape (modulo the `--auto` and cap-value differences which track separate proposals).

---

## PR-Later Note

This proposal is staged in the working tree. It is **not** committed, **not** pushed, **not** opened as a PR. To bring it forward when ready:

```bash
cd ~/Projects/Personal_AI_Infrastructure
git checkout main && git pull upstream main
git checkout -b proposal/knowledge-deep-ingest
git add Proposals/DeepIngestForLongerSources.md
# (apply the six edits to Packs/Knowledge/src/SKILL.md described above)
git add Packs/Knowledge/src/SKILL.md
git commit -m "Knowledge skill: auto deep-ingest companion note for long sources

Long or dense sources (>5,000 words, >20 PDF pages, vendor-analyst doc
types) auto-produce a sibling <slug>-deep.md companion note via a
three-lens extraction (stats+quotes, products+entities, tactical recipes).
Short articles still fast-path. New --deep / --no-deep flags override
the auto-trigger when needed."
git push origin proposal/knowledge-deep-ingest
gh pr create --base main --title "Knowledge: auto deep-ingest companion note for long sources" --body-file Proposals/DeepIngestForLongerSources.md
```

The proposal is intentionally narrow: it does *not* bundle the optional `/knowledge ingest <existing-slug> --deep-only` retroactive mode, the methodology fourth lens for academic sources, or migration of historical seedlings. Each of those is its own PR if the maintainer wants them.
