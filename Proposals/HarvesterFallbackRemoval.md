# Proposal: Remove KnowledgeHarvester `scanWorkISAs` Fallback Path

**Status:** Ready for PR — fallback verified present in the v5.0.0 release bundle 2026-06-10
**Co-authored-by:** Claude (Anthropic)
**Target:** `PAI/TOOLS/KnowledgeHarvester.ts` (single function: `scanWorkISAs`)
**Owner:** @asdf8675309
**Branch (when ready):** `proposal/harvester-fallback-removal`

## Problem

`KnowledgeHarvester.ts::scanWorkISAs()` has a silent fallback path: when an ISA has no `## Knowledge` section but the session sentiment ≥ 7, the harvester extracts `## Decisions` and `## Verification` sections verbatim, runs them through `classifyDomain()`, and writes them into `MEMORY/KNOWLEDGE/{Ideas|People|Companies|Research}/` as seedling notes.

This produces **systematic junk:**

- `Decisions` and `Verification` are session metadata — what the Algorithm chose during a run and how it verified completion. They are by definition session-scoped, not domain knowledge.
- `classifyDomain()` is heuristic. Session decisions about, e.g., a customer's pricing strategy get classified as `Companies/`; decisions about a specific person get classified as `People/`. The result is real-looking notes that aren't actually about those entities — they're about the session that touched those entities.
- Users who don't review every harvested seedling accumulate notes that look authoritative but encode session ephemera, polluting `KNOWLEDGE/` retrieval surfaces (`MemoryRetriever.ts`, `ContextSearch`, `KnowledgeRetrieve.hook.ts`).

The fallback was originally added as a back-compat path for pre-v3.16.0 ISAs that predate the explicit `## Knowledge` section convention. v3.17.0+ writes directly to `KNOWLEDGE/`, and the Algorithm LEARN phase populates explicit flags on completed ISAs. The fallback's reason-to-exist has expired; the junk it produces remains.

## Diagnosis

The bad path is bounded:

```typescript
// Lines ~196-217 of pre-change KnowledgeHarvester.ts
// No explicit flags — apply sentiment filter before falling back to section scanning
const sentiment = getSentimentForSession(dir);
if (sentiment !== null && sentiment < 7) continue;

// Fallback: extract Decisions/Verification sections (pre-v3.16.0 ISAs)
const decisions = extractSection(content, "Decisions");
const verification = extractSection(content, "Verification");
if (!decisions && !verification) continue;

const domain = classifyDomain(content, frontmatter);
candidates.push({
  sourcePath: isaPath,
  title: frontmatter.task || dir,
  content: [decisions, verification].filter(Boolean).join("\n\n"),
  domain,
  type: "idea",
  tags: extractTags(content),
});
```

Removing it changes behavior only for ISAs missing a `## Knowledge` section. Those ISAs go from "harvest junk" to "skip entirely" — strictly fewer writes, all of them were noise.

## Proposal

Delete lines ~198-217 of `scanWorkISAs`. Replace with a short comment explaining why (Algorithm LEARN phase via `## Knowledge` is the canonical source; pre-v3.16.0 ISAs can be retro-tagged if desired).

Net diff: **-20 / +6 = -14 lines** in one function in one file. Zero callers to update; zero state-schema change.

## Verification (post-change, on host 2026-05-16)

- `bun build --target=bun` clean (29.0 KB, 1 module, 0 errors).
- `bun KnowledgeHarvester.ts harvest --dry-run`: 0 candidates from `WORK/`, 0 from auto-memory/reflections/research. (10 recent ISAs lack `## Knowledge` sections, so the correct behavior is to skip — confirmed.)
- `bun KnowledgeHarvester.ts harvest --source work --dry-run --backfill`: 0 candidates — confirms backfill mode no longer retroactively scoops Decisions/Verification.

## Risk

None observed. The fallback was strictly an additive write path; removing it cannot break any caller. The state file (`MEMORY/KNOWLEDGE/.harvest-state.json`) already records the 265 prior harvests; removing the path doesn't re-process them.

## Retro-harvesting

Users who *want* pre-v3.16.0 ISAs to be harvestable: add a `## Knowledge` section with `- NEW domain/slug — description` lines, optionally delete the corresponding `harvestedPaths` entry from `.harvest-state.json`, then re-run.

## Files Changed

| File | Type | Lines |
|---|---|---|
| `PAI/TOOLS/KnowledgeHarvester.ts` | modified | -20 / +6 (net -14) |

Patch: `Proposals/patches/0007-harvester-fallback-removal.patch` (unified diff against `Releases/v5.0.0/.claude/PAI/TOOLS/KnowledgeHarvester.ts`; apply with `patch -p0` from the repo root).

## Platform & testing

Tested on macOS (Darwin, Bun runtime). Pure code deletion in one function; no platform-specific behavior. Not tested on Linux.
