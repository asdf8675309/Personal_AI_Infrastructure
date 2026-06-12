# Proposal: Single-Line Statusline Mode

**Status:** Ready for PR — 4-mode structure verified in the v5.0.0 bundle (`.claude/PAI/statusline-command.sh`, mode selector at lines 448-454) 2026-06-10
**Co-authored-by:** Claude (Anthropic)
**Target:** `Releases/v5.0.0/.claude/PAI/statusline-command.sh`
**Owner:** @asdf8675309
**Patch:** `Proposals/patches/0008-statusline-single-mode.patch`

## Platform & testing

Tested on macOS (Darwin, bash). The change is pure POSIX-ish bash within the script's existing style; no platform-specific calls added. Not tested on Linux.

## Problem

The current statusline script renders 4 layouts driven by terminal width: `nano` (<35), `micro` (35–54), `mini` (55–79), `normal` (80+). The `normal` layout emits ~11 lines of multi-line output (header, dashed separator, version line, CONTEXT bar, USE rate-limits, LEARNING with 3 sparkline rows, quote).

In some Claude Code environments — observed locally on Claude Code 2.1.128 in a 204-col terminal — the statusline area only renders **a single row** of the script's stdout, hiding the rest. Multi-line is documented as supported, but in practice the visible region collapses to one line for at least some host configurations (no obvious config knob on the user's side).

When this happens, all useful runtime state (CONTEXT %, USE %, LEARNING signal, quote) is invisible because it lives on rows 2+.

## Proposed Mode: `single`

Add a fifth statusline mode named `single` that packs the highest-value fields onto a single line:

```
PAI │ <flag> CITY, ST  HH:MM  <weather> │ CC:<v> PAI:<v> ALG:<v> │ CTX:<n>% │ 5HR:<n>% WK:<n>% │ LEARN:<n><src> 60m:<n> │ "<quote…>" —<author>
```

Layout intent:
- Brand + identity-of-place (location, time, weather) is the leftmost segment.
- Versions next (CC + PAI + Algorithm) — quick visual on what's running.
- Three quantitative segments after: context %, usage %, learning rating.
- Quote tail-ends the line, auto-truncated with ellipsis to roughly `term_width / 3` so it never dominates.

Total line length at typical 200-col terminals: ~150–180 chars (headroom for longer quotes / longer location names).

Dropped vs `normal`: SK / WF / HK count detail, sparkline rows, and the multi-line presentation. All underlying data is still tracked — the mode is a **presentation collapse**, not a data loss.

## Activation

Two activation paths, ordered by simplicity:

1. **Explicit override** (preferred): a config flag in `settings.json` under `statusLine.mode = "single"`, plumbed into the script via env var (e.g. `PAI_STATUSLINE_MODE`). When set, it bypasses the width-based selector and forces the named mode. This is a generic mechanism that also lets users force `nano` / `mini` / `normal` regardless of width.
2. **Auto-detect host**: probe the host environment (e.g. `CLAUDE_CODE_VERSION` if exposed, or a heuristic on TERM) and auto-select `single` for hosts known to render only one line. More fragile; only useful if (1) isn't present.

## Implementation outline

- Add `single` to the existing mode case statement that currently emits `nano|micro|mini|normal`.
- Wrap the current `MODE` selection block so an env var (`PAI_STATUSLINE_MODE`) can short-circuit it.
- Single-line composition is straightforward: gather already-prefetched values (`location_*`, `current_time`, `weather_str`, `cc_version`, `PAI_VERSION`, `ALGO_VERSION`, `context_pct`, `usage_5h`, `usage_7d`, latest learning rating, quote cache) and emit one `printf` with the existing color palette.
- Quote truncation: compute `_qmax = term_width / 3`, find last space at-or-before that index, append `…` if cut.

## Open questions for upstream review

1. **Naming.** `single` reads cleanly, but `compact` or `oneline` are alternatives. Defer to maintainer preference.
2. **Mode override mechanism.** Should the env-var override apply to all five modes, or be `single`-specific? Generic is more useful but bigger surface area.
3. **Behavior at very narrow terminals.** Even `single` mode will overflow under ~70 cols. Should it fall back to `nano` automatically, or just truncate?
4. **Sparkline alternative.** Some users will miss the sparklines. A possible compromise: a `single+spark` variant that adds a second line containing only the 60m sparkline. Out of scope for the first PR but worth noting.

## Why this belongs upstream

Multi-line statusline rendering is a host-dependent behavior. Until Claude Code's behavior stabilizes (or gains a documented config knob for vertical capacity), shipping a single-line layout in the official script is the most reliable way for users on affected hosts to get full visibility into their runtime state. The four existing modes already establish the precedent — `single` slots in cleanly as a fifth.

## Local reference

A working single-line implementation is running in this user's local PAI install. The script lives in the user's private PAI directory and was derived from the upstream `statusline-command.sh`. It is not vendored here — the PR will be a clean diff against the upstream script, drafted from the design notes above rather than copied from the local file.
