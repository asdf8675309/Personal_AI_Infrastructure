# Proposal: Mark DocIntegrity Stop hook async so it doesn't block the response cycle

**Status:** Ready for PR — verified against the v5.0.0 release bundle 2026-06-10
**Co-authored-by:** Claude (Anthropic)
**Target:** `.claude/settings.json` (Stop block registration of `DocIntegrity.hook.ts`)
**Patches:** `Proposals/patches/0003-docintegrity-stop-hook-async.patch`
**Severity:** Medium. Worst-case Stop chain latency can run into tens of seconds when the embedded inference call fires; combined with heavy forked-execution skills (`/Knowledge ingest`), this interacts badly with Claude Code's `/remote-control` bridge idle tolerance.

## Problem

The four Stop hooks registered in `.claude/settings.json` all run synchronously in registration order:

```json
"Stop": [
  {
    "hooks": [
      { "type": "command", "command": "$HOME/.claude/hooks/LastResponseCache.hook.ts" },
      { "type": "command", "command": "$HOME/.claude/hooks/ResponseTabReset.hook.ts" },
      { "type": "command", "command": "$HOME/.claude/hooks/VoiceCompletion.hook.ts" },
      { "type": "command", "command": "$HOME/.claude/hooks/DocIntegrity.hook.ts" }
    ]
  }
]
```

The first three are fast and bounded — they manage immediate post-response state that has to settle before the next user prompt is meaningful:

- `LastResponseCache` writes the last assistant message to disk for the rating bridge (~10ms).
- `ResponseTabReset` updates the kitty terminal tab title/color (~20ms).
- `VoiceCompletion` extracts the 🗣️ line and posts it to the local Pulse `/notify` endpoint (~50ms; fire-and-forget after that).

`DocIntegrity.hook.ts` is the outlier:

1. It calls `handleDocCrossRefIntegrity()`, which parses the entire transcript JSONL, builds a filesystem inventory of hooks/handlers/libs/SYSTEM docs, runs four pattern checks for broken refs and stale counts, then **invokes `inference()` against Sonnet with a 15-second timeout** to detect semantic drift and emit surgical edits.
2. It then calls `handleRebuildArchSummary()`, which (today, see `RebuildArchSummaryPathMismatch.md`) unconditionally spawns a `bun` subprocess to regenerate the architecture summary.

Best case: a few hundred milliseconds. Worst case: 10–20 seconds when the inference call fires after a session that actually modified system files.

That worst-case window lives squarely inside the post-response idle window the Claude Code `/remote-control` bridge expects to see activity in. The bridge silently stalls its input channel after heavy forked-execution skills (`/Knowledge ingest`), and `DocIntegrity` running synchronously after the response is a contributor — the user types the next message, but it never reaches the model until the bridge re-establishes.

## Why the other three Stop hooks should NOT become async

This proposal is deliberately narrow. `LastResponseCache`, `ResponseTabReset`, and `VoiceCompletion` all manage state that's part of "this response settling" — caching the response text, painting the terminal tab to a completed state, speaking the closing voice line. Making them async risks the next user prompt arriving before they've finished, which produces visible glitches (stale tab title, wrong voice line, missed rating context). `DocIntegrity` is different — it audits doc cross-references *after* this response is structurally complete; nothing downstream depends on it finishing before the next prompt.

## Proposed change

Register `DocIntegrity.hook.ts` with the `async: true` flag and a generous `timeout: 30` (its inference call alone is 15s; subprocess spawn for the architecture summary regen adds another few hundred ms). The hook continues to run on every Stop — it's just no longer in the critical path for the next prompt.

```json
{
  "type": "command",
  "command": "$HOME/.claude/hooks/DocIntegrity.hook.ts",
  "timeout": 30,
  "async": true
}
```

Other PAI hooks that do heavy work already follow this pattern:

- `UserPromptSubmit` → `PromptProcessing.hook.ts` (`async: true`, Sonnet classifier call)
- `UserPromptSubmit` → `SatisfactionCapture.hook.ts` (`async: true`, Haiku rating call)
- `SessionStart` → `KVSync.hook.ts` (`async: true`, remote KV sync)
- `PostToolUse` (Write/Edit) → `TelosSummarySync.hook.ts` (`async: true`)
- `PostToolUse` (any) → `ToolActivityTracker.hook.ts` + `ContentScanner.hook.ts` (`async: true`)

Bringing `DocIntegrity` in line with this established convention.

The patch also updates one description string in `settings.json` (`"DocIntegrity.hook.ts keeps documentation cross-references current and regenerates ARCHITECTURE_SUMMARY.md on Stop (async, non-blocking)."`) so the documented behavior matches and removes the stale `PAI_`-prefixed filename reference.

## Verification (already done locally)

After applying the patch:

```
$ jq '.hooks.Stop[0].hooks[] | select(.command | contains("DocIntegrity"))' ~/.claude/settings.json
{
  "type": "command",
  "command": "$HOME/.claude/hooks/DocIntegrity.hook.ts",
  "timeout": 30,
  "async": true
}
$ jq -e . ~/.claude/settings.json > /dev/null && echo "JSON valid"
JSON valid
```

Behavioral verification can't be a clean closed-loop test — the symptom is upstream Claude Code behavior, not something PAI controls. The clean test is "after the change, did the user stop needing to type `ping` after heavy ingests?" — that requires observing real bridge sessions, not a synthetic harness.

## How to apply

```bash
cd ~/Projects/Personal_AI_Infrastructure
patch -p0 < Proposals/patches/0003-docintegrity-stop-hook-async.patch
```

Unified-diff format against `Releases/v5.0.0/.claude/settings.json`. Dry-run (`patch --dry-run -p0`) succeeds.

## Upstream filing recommendation

The root cause is in Claude Code's remote-control implementation, not PAI. This proposal removes one PAI-side contributor — the synchronous Stop chain after heavy responses — but the underlying bridge tolerance is still worth filing at `github.com/anthropics/claude-code/issues` with a repro:

1. Start `/remote-control <name>` from a Claude Code session.
2. Run any `context: fork` skill that takes >30s (e.g. `/Knowledge ingest <large-PDF-URL>`).
3. After the skill returns and Claude responds, observe whether the next input from the remote browser reaches the local session within a reasonable window, or whether the user has to send a wake-up message ("ping") first.
4. If the latter, suggest the bridge needs an explicit keepalive during long forked tool executions.

## Related

- `Proposals/RebuildArchSummaryPathMismatch.md` — fixes the actual bug inside `DocIntegrity.hook.ts → handleRebuildArchSummary` that caused the architecture summary to regenerate on every Stop. Together with this proposal, the Stop chain becomes both faster (no redundant regen) and non-blocking (async).

## Platform & testing

Tested on macOS (Darwin, Bun runtime). The change is a settings.json registration flag plus a description string — no platform-specific behavior. Not tested on Linux.
