# Proposal: Anvil Context Selection — Dependency Closure, Not "the Whole Repo"

**Status:** Ready for PR — insertion anchors verified at capabilities.md lines 60-74 of the v5.0.0 bundle 2026-06-11
**Co-authored-by:** Claude (Anthropic)
**Target:** `Releases/v5.0.0/.claude/PAI/ALGORITHM/capabilities.md` — extends the existing "Anvil invocation binding (E3-E5 long-context coding tasks)" section
**Owner:** @asdf8675309

## Platform & testing

Documentation-only change (doctrine insert); no code, no platform behavior. Running locally since 2026-05-18.
**Branch (when ready):** `proposal/anvil-context-selection`

## Problem

The current Anvil binding in `capabilities.md` tells the executor *when* to pick Anvil over Forge ("long-context breadth, project-shape focus, does-this-fit is the dominant question") but does not tell it *what to load into Anvil's 256K context window when it picks him.* In practice, this gap produces three failure modes, especially for monorepo users:

1. **Whole-repo dump.** The executor reads "long-context" as a directive to maximize input — dumps the entire repo tree, including `node_modules`, lock files, generated build artifacts, and unrelated packages. Noise drowns signal; Anvil's quality degrades below Forge's.
2. **Wrong unit.** In a monorepo the executor either (a) loads only the single app the file lives in and misses the cross-package dependency, or (b) loads the whole monorepo and exhausts the window before reasoning. Both fail.
3. **Anvil-when-Forge-would-do.** Without a tell-tale-signs heuristic, single-file surgical edits get routed to Anvil because the prompt mentioned "the project." Forge is faster, cheaper, and equally correct for bounded surfaces.

The fix is doctrinal: name **dependency closure** as the unit and give the executor a task-shape table plus a tell-tale-signs checklist.

## Diagnosis

The existing Anvil binding (lines 60-72 of `capabilities.md`) is structured as:
- Trigger (when to consider Anvil)
- Behavior (where in the Algorithm to invoke)
- Forge-vs-Anvil decision rule
- Parallel-both pattern
- Explicit-name override

None of these answer "what files should I actually load." That decision is left implicit to the executor's judgment, which routinely defaults to "as much as possible." The 256K window then biases toward over-loading rather than precision.

A short additive sub-section after the explicit-name override closes the gap without touching existing structure.

## Proposal — Single Additive Insert

After the "Explicit-name override" line in the Anvil binding section, before `## Delegation & Infrastructure Capabilities`, insert:

````markdown
**Context selection — dependency closure, not "the whole repo":** Anvil's 256K window is a budget, not a directive to dump everything. The unit to feed Anvil is the **dependency closure of the work** — the smallest tree containing every file the answer could depend on. Concretely:

| Task shape | Anvil's context unit |
|------------|---------------------|
| Inside one app/package | That app + the `packages/*` it directly imports |
| Cross-app refactor (rename shared type used by N packages) | The N affected packages + root workspace config (`turbo.json`, `pnpm-workspace.yaml`, root `package.json`) |
| Build/CI/workspace change | Root config files + one representative leaf to prove it still builds |
| New shared util consumed by M apps | The M consumer apps + the new package |
| Single-file surgical edit | **Don't call Anvil** — Forge or direct edit is correct |

**Working budget:** ~150K tokens of *relevant* code (leaving headroom for prompt + response). For a monorepo, the whole repo almost certainly doesn't fit and shouldn't — noise hurts. Use `tokei` / `cloc` on the subtree if unsure.

**Tell-tale signs Anvil (not Forge) is the right call:**
- Bug could be in any of 4+ files and you don't yet know which
- A type change ripples across packages — every caller must be visible at once
- Pattern audit across a subtree ("are we using the wrong logger anywhere in `apps/`?")
- Reading the answer requires holding the dependency graph in working memory

If none apply → Forge with targeted Reads is faster and cheaper.

**Prompt phrasing that activates this binding:** name the closure explicitly, not the gesture. ❌ "Use Anvil on the monorepo to refactor X" → ✅ "Use Anvil — load `apps/web` + `packages/ui` + `packages/api-client` + root `package.json` / `turbo.json`. Refactor X."
````

Total insertion: ~30 lines, purely additive, no existing content modified.

## Why This Is Upstream-Eligible

1. **Universal applicability.** Every PAI user who picks up Anvil hits the same "what do I load" question. The heuristic is independent of codebase, personal data, or workflow.
2. **No infrastructure dependency.** No new tools, hooks, or skills. Pure doctrine clarification in a file other PAI users already read.
3. **Codifies a known-good pattern.** Experienced engineers already reason this way about cross-file work — this makes the executor do the same.
4. **Complements rather than competes with the existing Anvil section.** Surgically inserted, doesn't restructure or contradict the prior text.
5. **Matches the prior pattern of small additive doctrine clarifications** (e.g., `HarvesterFallbackRemoval.md`, `DeepIngestForLongerSources.md`).

## Reversibility

Pure additive — no removal, no restructuring. Reverting is `git apply -R` on the diff against `PAI/ALGORITHM/capabilities.md`. No `revert.sh` ceremony needed.

## Re-application Procedure (if upstream restructures `capabilities.md`)

If a future PAI upgrade rewrites the Anvil section:

1. Locate the new Anvil binding block (search for "Anvil" in `capabilities.md`).
2. Verify the existing block does not already contain "dependency closure" or "Anvil's 256K window is a budget."
3. If absent, apply the insertion above immediately after the new Anvil block's last sub-section, before the next H2/H1 heading.
4. If the new structure has a dedicated "context selection" sub-heading anywhere, integrate the table and tell-tale-signs into it rather than duplicating.

## Provenance

Derived from a 2026-05-18 working session on how "whole project" applies to a monorepo. The session surfaced that the existing binding had no answer, and the dependency-closure framing emerged as the missing doctrine. Local deployment took the form of an Edit to `~/.claude/PAI/ALGORITHM/capabilities.md` immediately after the "Explicit-name override" line.
