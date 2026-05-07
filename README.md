# vigil-watch

Vigil daemon for the Even G2 Companion HUD.

Observes `~/.claude/projects/<project-id>/<session-id>.jsonl` files written by
the Claude Code VS Code extension and POSTs 5 Vigil event types
(`needs_input`, `task_complete`, `task_failed`, `milestone`, `heartbeat`) to
Vigil Core's `/v1/agent-events` endpoint. Read-only on disk.

**Status:** Day-1 verification gate (Phase 120). No daemon code yet.

**Spec:** See [`vigil` repo `v3.8-CLAUDE-CODE-COMPANION-SPEC.md`](https://github.com/dttigers/vigil-core) for the milestone goal. This README captures the load-bearing schema verification before any code is written.

**License:** MIT. **Platform:** macOS only. **Language:** Swift.

---

## Day-1 JSONL Schema Verification

> Filled by Phase 120 of the v3.8 Claude Code Companion milestone.
> **Status:** awaiting Plan 120.02 (verification log) + Plan 120.03 (findings authoring).

### Verdict

_Filled by Plan 120.03. Will contain exactly one of:_

- `proceed-as-spec` — observed JSONL matches the spec's assumed mapping verbatim.
- `spec-correct-and-proceed` — observed JSONL matches semantically; specific row corrections recorded below.
- `fallback-path-A` — JSONL approach unviable; switch to Notification Center observation.
- `fallback-path-B` — JSONL approach unviable; switch to VS Code extension hook.
- `fallback-path-C` — JSONL approach unviable; switch to process inspection.

### JSONL Line-Type Mapping

_Filled by Plan 120.03. 8-row table, one row per assumed JSONL line type from spec §"Expected JSONL line types"._

| Spec-assumed type | Observed type / field path | Vigil event emitted | Confirmed / Corrected / Missing |
|---|---|---|---|
| `user` | _TBD_ | (none — track session liveness) | _TBD_ |
| `assistant` | _TBD_ | (none — informational) | _TBD_ |
| `tool_use` (with `awaiting_approval`) | _TBD_ | `needs_input` | _TBD_ |
| `tool_result` (success) | _TBD_ | (none — resets silence timer) | _TBD_ |
| `tool_result` (error) | _TBD_ | `task_failed` | _TBD_ |
| `session_end` / `stop` | _TBD_ | `task_complete` | _TBD_ |
| (silence ≥60s while running) | _TBD_ | `heartbeat` | _TBD_ |
| (regex match in any text) | _TBD_ | `milestone` | _TBD_ |

### Spec-Flagged Questions

_Filled by Plan 120.03. Each answer cites a specific JSONL field path (e.g., `$.message.content[].type`) and references one or more raw excerpts in the appendix._

1. **How do tool-approval prompts appear in JSONL?** _TBD_
2. **What fields indicate "awaiting input"?** _TBD_
3. **How is session end signaled?** _TBD_
4. **What is the structure of an errored tool result?** _TBD_

### Downstream Phase Impact

_Filled by Plan 120.03. If Verdict is `proceed-as-spec` or `spec-correct-and-proceed`, this section may be a single line: "No downstream goal shifts." If Verdict is `fallback-path-N`, this section enumerates which Phase 122 / Phase 123 success-criteria items shift (e.g., "Phase 122 SC #1 'FSEventStream rooted at ~/.claude/projects/' becomes 'NSDistributedNotificationCenter observer for VS Code notifications'")._

---

## Appendix: Raw JSONL Excerpts

_Filled by Plan 120.03. At least 8 excerpts (≥1 grounding each mapping-table row). Each excerpt is a fenced ``json`` block with a heading naming the source file and approximate line range, e.g., `### Excerpt 1: tool_use awaiting approval (00697e82-fe0a-4bac-b30e-44473f8a7be4.jsonl, lines 142-148)`._

---

## After Phase 120

Once verification is locked, Phase 122 fills this repo with the Swift daemon implementation per the verdict captured above.
