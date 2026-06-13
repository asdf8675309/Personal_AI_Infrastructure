# Proposal: Scope the Knowledge `related:` Cap to Outbound Author Intent

**Status:** Ready for PR — all four line citations verified against the v5.0.0 bundle 2026-06-10
**Co-authored-by:** Claude (Anthropic)
**Target:** `Packs/Knowledge/src/SKILL.md` lines 91, 108, 215, 252 — apply the same edits to the byte-identical `Releases/v5.0.0/.claude/skills/Knowledge/SKILL.md`
**Owner:** @asdf8675309
**Branch (when ready):** `proposal/knowledge-related-cap-scope`

## Problem

The Knowledge skill's Canonical Linking Requirement caps the `related:` frontmatter array at **2 minimum, 4 maximum** typed entries per note (with a related "find 2-3" instruction in the `add` workflow). This rule appears in **four** places in `SKILL.md`, with a small numerical inconsistency:

- L91  — `add` workflow Step 4: "MANDATORY: Find **2-3** related notes first"  *(note: 2-3, not 2-4)*
- L108 — Canonical Linking Requirement: "2-4 typed entries"
- L215 — `ingest` Step 2: "MANDATORY: Include `related:` array with 2-4 typed links"
- L252 — `ingest` Step 4: "ensure `related:` frontmatter array has 2-4 typed entries"

The 2-3 vs 2-4 inconsistency at L91 is a symptom of the same root cause: the rule covers two semantic roles that aren't named.

Across a single 19-source ingestion session, the cap fired against the wrong signal at least five times. Concrete cases (each shows a reverse-direction typed link being silently downgraded because the *target* note was already at 4):

| Inbound source | Target hit cap | Workaround applied |
|---|---|---|
| `compactifai-tensor-network-llm-compression` | `local-deep-research-agentic-autonomy`, `caveman-token-compression-skill`, `huihui-granite-4-1-30b-abliterated` | Body wikilink only; reverse `related:` entry skipped |
| `littlelamb-0-3b-compactifai-compressed-edge-models` | both prior CompactifAI notes | Dropped a weaker link to make room |
| `the-illiterate-organization` | `ai-changed-my-work-judgment-as-the-uncompressed-asset` | Body wikilink only; frontmatter entry skipped |
| `huihui-granite-4-1-30b-abliterated` (multiple ripples) | several already-popular hubs | Frontmatter entry skipped |
| `reinforcement-learning-an-overview-murphy-2024` | `local-deep-research`, `convergence-of-agent-customization` | Frontmatter entries skipped |

The pattern is invariant: **the cap punishes already-well-connected notes for being well-connected.** Reverse links — the ones the ripple pass auto-discovers from a *new* note — get downgraded into body-only wikilinks (which the harvester parses differently, with a weaker edge type) or dropped entirely.

## Diagnosis: one bag, two semantic roles

The `related:` array currently mixes two categorically different things into one bag:

1. **Outbound (author-stated).** When a new note is created, its author picks the 2–4 most important relationships. This is **curation discipline** — capping it prevents kitchen-sink notes where every link is meaningless because there are 50 of them.
2. **Inbound (ripple-added).** When a *subsequent* note is created and its `ingest` ripple pass discovers a typed connection to this note, a reverse-direction entry should accumulate. This is **graph emergence** — capping it forfeits exactly the signal the graph is built to surface (which notes are hubs).

The cap is correct for (1) and incorrect for (2). The schema doesn't distinguish them, so the cap fires on whichever role happens to be writing.

## Proposed fix

Re-scope the cap. Keep one bag (no new schema field), change the rule:

> **Outbound (author-stated entries on a new note):** 2–4 typed links. Curate.
> **Inbound (ripple-added entries from notes that subsequently link AT this one):** no upper bound. Accumulate.

Operationally, "outbound" = entries written when the note is *created*. "Inbound" = entries written by the `ingest` Step 4 ripple pass into a *pre-existing* note's frontmatter. The harvester (`KnowledgeHarvester.ts`) already treats every entry identically when reading the graph, so accumulated inbound entries surface naturally in graph-stats, contradiction-detection, and 2-hop traversal.

### Why this is the minimal fix

- **Zero new schema fields.** No `inbound_related:` array, no migration of existing notes, no harvester changes.
- **Zero code changes required.** Verified: `KnowledgeHarvester.ts` parses the array but does not enforce a length cap. `KnowledgeGraph.ts` likewise has no cap. The cap is purely policy in `SKILL.md`.
- **One file changed in the PR.** `Packs/Knowledge/src/SKILL.md`.
- **Bitter Pill compliant.** The cap was a workaround for a missing distinction; we remove the workaround by stating the distinction directly.

### Why not the alternatives

| Alternative | Rejected because |
|---|---|
| Just raise the cap to 8 | Kicks the can. Doesn't fix the conflated semantics. The same failure recurs at the new ceiling. |
| Add `inbound_related:` as a second array | Doubles the schema surface. Every downstream tool (harvester, graph, MOC generator, contradiction finder, MemoryRetriever) needs an update. Fails Bitter Pill. |
| Eliminate the cap entirely | Loses the discipline that prevents kitchen-sink author behavior on new notes. |
| Per-relationship-type caps | More rules, no better signal. |

## Edits

### `Packs/Knowledge/src/SKILL.md`

**L91 — `add` workflow Step 4 (the *find* instruction):**

Before:
```
4. **MANDATORY: Find 2-3 related notes first.** Before writing the new note, grep existing Knowledge for related entities by topic/tags/name. This becomes the `related:` frontmatter array. No Knowledge note ships without typed links. See Canonical Linking Requirement below.
```
After:
```
4. **MANDATORY: Find 2–4 related notes first.** Before writing the new note, grep existing Knowledge for related entities by topic/tags/name. These become the *outbound author-stated* entries in the `related:` frontmatter array (the cap applies only here — see Canonical Linking Requirement below). No Knowledge note ships without typed links.
```

(Resolves the 2-3 vs 2-4 inconsistency in passing — same number on both sides.)

**L108 — Canonical Linking Requirement (the *write* instruction):**

Before:
```
1. **`related:` frontmatter array** — 2-4 typed entries linking to other Knowledge entries (any domain: People, Companies, Ideas, Research)
```
After:
```
1. **`related:` frontmatter array** — 2–4 typed entries at note creation time (outbound author intent). Notes also accumulate ripple-added inbound entries from subsequent ingests; **inbound entries are not capped** — a hub note collecting 12 typed reverse links is the system working as intended, not a violation.
```

**L215 — `ingest` Step 2:**

Before:
```
- **MANDATORY: Include `related:` array with 2-4 typed links** — the ripple pass (Step 3) identifies these, and they must be baked into the frontmatter of the primary note at creation time, not added after
```
After:
```
- **MANDATORY: Include `related:` array with 2–4 typed links** — these are the *outbound author-stated* entries on the primary (new) note. The ripple pass (Step 3) identifies these, and they must be baked into the primary note's frontmatter at creation time. The 2–4 cap applies only here.
```

**L252 — `ingest` Step 4:**

Before:
```
- **Primary note**: ensure `related:` frontmatter array has 2-4 typed entries — this is mandatory, not optional
```
After:
```
- **Primary note**: ensure `related:` frontmatter array has 2–4 typed *outbound* entries — this is mandatory, not optional
- **Related (existing) notes**: append the reverse-direction typed entry to their `related:` frontmatter **regardless of how many entries they already hold** — inbound ripple entries are unbounded by design. Do *not* drop existing entries to make room. Do *not* downgrade to body-only wikilinks when the frontmatter slot is the right home.
```

### Optional clarifying paragraph

Add a short subsection just after the "8 relationship types" table (around L123) stating the outbound/inbound distinction once, so the rule is read in one place even if a contributor only sees one of the three locations.

## Rollback

Revert the four edits in `Packs/Knowledge/src/SKILL.md`. No code, no schema migration. Existing notes that have accumulated >4 entries under the new rule keep working — the harvester never enforced the cap.

## Open questions for upstream review

1. **Wording.** "Outbound" / "inbound" is precise but jargon-ish. "Author-stated" / "ripple-added" is plainer. Defer to maintainer preference.
2. **Should we publish a soft inbound ceiling for *display* purposes?** E.g., MOCs render the first 12 inbound and link to "view all." Worth flagging but not required for this PR.
3. **Migration.** Existing notes that artificially dropped reverse links under the old cap will not auto-heal. A one-shot `bun KnowledgeHarvester.ts repair-ripple` could re-derive missing inbound entries by scanning for `extends/supports/contradicts` outbound entries that lack a matching inbound. Out of scope for this PR; tracked separately.
4. **Schema docs.** This repo doesn't ship `_schema.md` (it's user-local). The same correction should land in user installs that already have one — possibly via a `harvest --upgrade-schema` command later.

## Why this belongs upstream

Today's session makes the failure concrete: roughly a quarter of ripple updates silently degraded. The longer the cap stays in place, the more the graph drifts away from reflecting actual connectivity — the very thing the graph exists to surface. The fix is one file, four edits, reversible, and doesn't change runtime behavior — only what the workflow tells authors to write. It's doctrinal, not mechanical.

## Local reference

The user's private PAI install carries the same three-location cap rule in `~/.claude/skills/Knowledge/SKILL.md` and an additional copy in `~/.claude/PAI/MEMORY/KNOWLEDGE/_schema.md` (L20, L198). The local copies are deliberately **not** edited as part of this proposal; they will be brought into line when this PR is accepted upstream so the local working copy and the upstream skill stay synchronized.

---

## PR-Later Note

To bring this forward as a PR:

```bash
cd ~/Projects/Personal_AI_Infrastructure
git checkout main && git pull upstream main
git checkout -b proposal/knowledge-related-cap-scope
git add Proposals/KnowledgeRelatedCapScopedToOutbound.md
# (apply the four edits to Packs/Knowledge/src/SKILL.md described above)
git add Packs/Knowledge/src/SKILL.md
git commit -m "Knowledge skill: scope related: cap to outbound author intent

Reverse-direction (ripple-added) entries are no longer capped, so hub
notes can accumulate the typed inbound links they accrue. Cap remains
on outbound author-stated entries to preserve curation discipline."
git push origin proposal/knowledge-related-cap-scope
gh pr create --base main --title "Knowledge: scope related: cap to outbound author intent" --body-file Proposals/KnowledgeRelatedCapScopedToOutbound.md
```

The proposal is intentionally narrow: it does *not* bundle the optional `repair-ripple` migration, the schema-docs sync, or the display-time soft ceiling. Each of those should be its own PR if the maintainer wants them.
