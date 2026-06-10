# Proposal: Pulse Performance — Per-Model Pricing Weights (Fable 5), Model-Attribution Fix, and Pinned codeburn with Drift-Check Cron

**Status:** Ready for PR — implemented and verified locally 2026-06-09/10
**Target:** `PAI/PULSE/Performance/cost-aggregator.ts`, `PAI/PULSE/Observability/src/app/performance/page.tsx`, `PAI/PULSE/PULSE.toml`, `PAI/PULSE/Performance/codeburn-version-check.ts` (new)
**Owner:** @asdf8675309
**Branch (when ready):** `proposal/pulse-fable-model-pricing`
**Patches:** `patches/0004-pulse-cost-aggregator-fable-weights-peek-fix-pin.patch`, `patches/0005-pulse-performance-page-fable-display.patch`, `patches/0006-pulse-codeburn-version-check-cron.patch`

## Problem

The Performance dashboard's session cost pipeline (codeburn → cost-aggregator → `/api/performance/cost` → Cost tab) has four related defects exposed by the release of Claude Fable 5:

1. **No Fable support.** `MODEL_NAME_MAP` can't canonicalize codeburn's "Fable 5" display name, and `shortModel()` in the dashboard renders Fable sessions as a raw truncated model id.
2. **Stale, single-model cost-split weights.** `buildShares()` splits each session's total cost into input/output/cacheWrite/cacheRead buckets using one hardcoded weight table (15/75/18.75/1.5) — Opus 4.0-era pricing applied to every token regardless of model. With current pricing (Opus 4.x = $5/$25, Fable 5 = $10/$50), mixed-model periods split incorrectly.
3. **Model-attribution peek window too small.** `resolveSessionModel()` reads only the first 4KB of a session transcript to find the first `"model":"..."` token. Sessions with large first records (loaded context) carry that token far past 4KB — observed at byte ~97K — so those sessions silently fall back to the period's top model. Any modern session with substantial startup context is mis-attributed.
4. **Unpinned `bunx codeburn` on a cron surface.** The aggregator runs every 15 minutes via Pulse cron and invoked `bunx codeburn` (@latest). A bad upstream publish would break cost aggregation silently.

## Change

**0004 — cost-aggregator.ts**
- Add `"Fable 5": "claude-fable-5"` to `MODEL_NAME_MAP`.
- Replace the single weight table with `FAMILY_WEIGHTS` (per-family current public $/MTok: fable 10/50, opus 5/25, sonnet 3/15, haiku 1/5; cacheWrite = 1.25× input, cacheRead = 0.1× input) and accumulate weighted components per model row. Weights only SPLIT codeburn's per-session total — absolute pricing still comes from codeburn → LiteLLM, so totals are unchanged. Unknown/non-Anthropic models fall to the opus default.
- Replace the 4KB peek in `resolveSessionModel()` with a chunked scan (64KB chunks, 1MB cap, cross-boundary carry).
- Pin `bunx codeburn@0.9.12`.

**0005 — performance/page.tsx**
- `shortModel()`: add `if (m.includes("fable")) return "Fable";`.

**0006 — drift-check cron (companion to the pin)**
- New `Performance/codeburn-version-check.ts`: reads the pinned version out of cost-aggregator.ts, queries the npm registry, and POSTs a Pulse notification only on drift. Supports `--simulate-latest X` for testing.
- New `[[job]]` in PULSE.toml: weekly (Mondays 09:23 local). Update flow stays human: drift alert → review release notes → bump pin → run aggregator once → confirm `schema=codeburn.export.v2` parses.

## Verification performed (local)

- Aggregator: exit 0, merge baseline preserved across runs, 2 previously-invisible Fable sessions now resolve (`{"model":"claude-fable-5","cost":41.34,"sessions":2,"tokens":34664767}` in `/api/performance/cost`).
- Dashboard: rebuilt static export; "Fable" row renders in Cost by Model; verified in real Chrome (Interceptor), console clean.
- Drift check: current path exits 0 silently; `--simulate-latest 9.9.9` delivers the Pulse notification.
- Pulse restarts clean with the new job parsed.

## Platform & testing (per PLATFORM.md)

Tested on macOS (Darwin 25.4, Bun runtime). No platform-specific code paths added: the new script uses `fileURLToPath`/`dirname`/`readFileSync` and `fetch` only; all paths are relative to the script or come from existing helpers. Both registry and notify fetches carry 10s `AbortSignal.timeout` per SECURITY.md's web-request guidance. Not tested on Linux, but nothing in the diff branches on platform; patch paths use the post-#1259 `PULSE` directory casing.

## Notes for maintainers

- Diffs are generated against a live 2026-06 tree that includes the multisource cost refactor (2026-05-23). The `Releases/v5.0.0` copy of cost-aggregator.ts predates that refactor — 0004 will need a trivial rebase if applied there.
- The pinned version (0.9.12) and `FAMILY_WEIGHTS` values are point-in-time; both are single-constant updates, and the drift-check job exists precisely to surface when the pin goes stale.
- `useAdvancedMetrics.ts:206` (separate event-stream estimate) still hardcodes $15/$75 Opus pricing — out of scope here, noted as a follow-up candidate.
