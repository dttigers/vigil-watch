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

> Phase 120 of the v3.8 Claude Code Companion milestone — empirical verification gate locked before any production-mapping code is written.

### Verdict

**`spec-correct-and-proceed`** — Observed JSONL has empirical signal for all 5 Vigil event types (`needs_input`, `task_complete`, `task_failed`, `milestone`, `heartbeat`), so D-06's pragmatic-fallback rule does NOT escalate to a fallback path. However, the spec's assumed mapping is structurally wrong on 4 rows: top-level `tool_use` and `tool_result` types do not exist (they are inner `message.content[].type` discriminators), and there is no dedicated `session_end` line type at all (sessions end by the file ceasing to append). Field-path corrections are recorded row-by-row below. No fundamental capability gap, no goal shifts for Phase 122/123 — only mapping-rule corrections.

**Captured:** 2026-05-08 on host `Jamesons-iMac.local` against Claude Code session JSONL written by VS Code Claude extension `2.1.133`.

### JSONL Line-Type Mapping

8-row table — one row per assumed JSONL line type from spec §"Expected JSONL line types". Each row's status is `Confirmed`, `Corrected: <fix>`, or `Missing`. Field paths use JSONPath-style notation against a single JSONL line (one JSON object per line).

| Spec-assumed type | Observed type / field path | Vigil event emitted | Confirmed / Corrected / Missing |
|---|---|---|---|
| `user` | `$.type == "user"` (top-level, confirmed verbatim — 14,366 hits in corpus). Also carries `$.permissionMode` ∈ `{"default", "bypassPermissions"}`. → [Excerpt 1](#excerpt-1-user-prompt-line-default-permission-mode) | (none — track session liveness) | `Confirmed` |
| `assistant` | `$.type == "assistant"` (top-level, confirmed verbatim — 21,614 hits in corpus). Carries `$.message.stop_reason` ∈ `{"tool_use", "end_turn", "stop_sequence", "refusal", null}`. → [Excerpt 2](#excerpt-2-assistant-line-with-stop_reason-end_turn) | (none — informational) | `Confirmed` |
| `tool_use` (with `awaiting_approval`) | `$.type == "assistant"` AND `$.message.content[?(@.type=="tool_use")]` (inner content discriminator, NOT top-level). The `awaiting_approval` substring does NOT exist anywhere in the corpus. → [Excerpt 3](#excerpt-3-assistant-line-with-inner-tool_use-block) | `needs_input` | `Corrected: top-level "tool_use" does not exist; the signal is "$.type=assistant AND $.message.content[].type=tool_use". needs_input must be inferred via tool_use→tool_result gap exceeding N seconds AND $.permissionMode != "bypassPermissions"; no awaiting_approval marker exists.` |
| `tool_result` (success) | `$.type == "user"` AND `$.message.content[?(@.type=="tool_result" && @.is_error==false)]` (inner content discriminator). Also `$.toolUseResult` (camelCase sibling) is a structured object with `{stdout, stderr, interrupted, isImage, noOutputExpected}`. → [Excerpt 4](#excerpt-4-user-line-with-tool_result-success-is_error-false) | (none — resets silence timer) | `Corrected: top-level "tool_result" does not exist; the signal is "$.type=user AND $.message.content[].type=tool_result AND $.message.content[].is_error==false".` |
| `tool_result` (error) | `$.type == "user"` AND `$.message.content[?(@.type=="tool_result" && @.is_error==true)]`. Inner `content[].content` carries the error text (often prefixed `"Exit code N\n"`). The `is_error: true` boolean is the deterministic single-field discriminator. → [Excerpt 5](#excerpt-5-user-line-with-tool_result-error-is_error-true) | `task_failed` | `Corrected: top-level "tool_result" does not exist; same level-shift as success row. is_error:true is the load-bearing single-field discriminator (not a separate type, not a separate subtype).` |
| `session_end` / `stop` | NO JSONL line carries `$.type == "session_end"` or `$.type == "stop"` (zero hits across 47,357 corpus lines). Sessions end by the file simply ceasing to append. The closest natural proxy is `$.message.stop_reason == "end_turn"` on an `assistant` line followed by sustained silence (no further appends). → [Excerpt 6](#excerpt-6-session-tail-no-explicit-end-marker) | `task_complete` | `Missing` |
| (silence ≥60s while running) | Inferred from timestamp deltas — comparing `$.timestamp` (ISO-8601) of the most recent JSONL line against `now()`. Empirically observed 127.6s gap between scripted-session line 21 (`assistant` `stop_reason:end_turn`) and line 22 (next user `queue-operation`). → [Excerpt 7](#excerpt-7-silence-gap-127s-between-consecutive-lines) | `heartbeat` | `Confirmed` (this row is a meta-rule — heartbeat is a watcher inference, not a JSONL line type) |
| (regex match in any text) | Free-text match within `$.message.content[?(@.type=="text")].text` on `assistant` lines OR within `$.message.content` (string form) on `user` lines. Pattern is user-defined per `watch.toml` rules. → [Excerpt 8](#excerpt-8-assistant-text-content-block-grep-target-for-milestone-regex) | `milestone` | `Confirmed` (this row is a meta-rule — milestone is a watcher inference applied to text fields) |

**Pragmatic-fallback assessment for the `Missing` row:** The `session_end`/`stop` row has no JSONL line type, but the corresponding `task_complete` Vigil event still has signal — combining `stop_reason:"end_turn"` with sustained silence yields a heuristic detection rule. Per D-06, missing-row fallback is triggered only when a Vigil event fundamentally lacks signal; here the signal is just inferred rather than literal, so the verdict stays `spec-correct-and-proceed`.

#### Vigil event detection rules (inverse view)

The same mapping, keyed by Vigil event for Phase 122's parser:

| Vigil event | Detection rule | Has direct line-type signal? |
|---|---|---|
| `needs_input` | `$.type=="assistant"` AND `$.message.content[?(@.type=="tool_use")]` AND no matching `tool_result` within N seconds AND `permissionMode != "bypassPermissions"` | No — inferred from timing + permission state |
| `task_complete` | `$.message.stop_reason == "end_turn"` AND `(now - latestTimestamp) > taskCompleteSeconds` AND no further `user` line | No — inferred from silence + stop_reason |
| `task_failed` | `$.type=="user"` AND any `$.message.content[].is_error == true` | Yes — single-field deterministic |
| `milestone` | regex match against `$.message.content[?(@.type=="text")].text` (assistant lines) per user-defined `watch.toml` patterns | Yes — string match on content |
| `heartbeat` | `(now - latestTimestamp) > heartbeatSeconds` | No — purely a timestamp-delta computation |

### Spec-Flagged Questions

Each answer cites a specific JSONL field path and references one or more raw excerpts in the appendix.

1. **How do tool-approval prompts appear in JSONL?** They do NOT appear as a dedicated line type. Two field-path signals exist: (a) `$.permissionMode` on `user` prompt lines indicates whether approval prompts will be shown at all (`"default"` enables them; `"bypassPermissions"` skips them, in which case there is no awaiting state); (b) the time gap between an `assistant` line carrying `$.message.content[?(@.type=="tool_use")]` and the matching `user` line carrying `$.message.content[?(@.type=="tool_result" && @.tool_use_id==<same-id>)]`. In `default` mode, that gap captures the user's approval-click time (5.2s in the scripted session). Field path: `$.message.content[].type` (= `"tool_use"`). Evidence: [Excerpt 3](#excerpt-3-assistant-line-with-inner-tool_use-block) (lines 11 of `2072cbce-...jsonl`) and [Excerpt 4](#excerpt-4-user-line-with-tool_result-success-is_error-false) (line 12 of the same file).

2. **What fields indicate "awaiting input"?** None directly — the JSONL is silent on this state. The watcher must INFER it from `(assistant emitted $.message.content[].type==tool_use) AND (no follow-up user line with matching content[].tool_use_id within N seconds) AND ($.permissionMode != "bypassPermissions" on the most recent user line for this session)`. Field path: combinatorial — `$.message.content[].type` flips from observed `"tool_use"` (no result) to absent (when result arrives). The `awaiting_approval` string is empirically absent from all 47,357 corpus lines. Evidence: [Excerpt 3](#excerpt-3-assistant-line-with-inner-tool_use-block) (assistant tool_use without immediate result), [Excerpt 1](#excerpt-1-user-prompt-line-default-permission-mode) (`$.permissionMode` field).

3. **How is session end signaled?** It is NOT signaled by any line type. There is no `$.type == "session_end"` or `$.type == "stop"` anywhere in the corpus (47,357 lines, 184 files). The session ends by the JSONL file simply ceasing to append. The detector must rely on sustained silence — `(now - latestLineTimestamp) > sessionCloseSeconds` — combined with the most recent `$.message.stop_reason == "end_turn"` on an `assistant` line to disambiguate "session done" from "session mid-tool-call". Of 184 corpus files, 80% end on a `last-prompt` line (an overwrite-style sticky tail marker) and the remaining 20% scatter across `assistant`, `ai-title`, and other types — none is a deliberate end-signal. Field path: `$.message.stop_reason` (= `"end_turn"`) plus a timestamp-delta computation. Evidence: [Excerpt 6](#excerpt-6-session-tail-no-explicit-end-marker) (line 33 of `2072cbce-...jsonl` — file simply stops after `ai-title`).

4. **What is the structure of an errored tool result?** Identical shape to a successful tool result, with the SOLE distinguishing field `$.message.content[?(@.type=="tool_result")].is_error` flipped to `true`. The error text lives in `$.message.content[].content` (often prefixed `"Exit code N\n"` for failed shell commands). The sibling top-level field `$.toolUseResult` is a structured object `{stdout, stderr, interrupted, isImage, noOutputExpected}` whose `stdout` carries the same error text. In extension v2.1.133, `$.toolUseResult` remains an object on errors (not a string starting `"Error: …"`). Phase 122's parser implements this answer directly: emit `task_failed` IFF `$.type=="user"` AND any `$.message.content[].is_error == true`. Evidence: [Excerpt 5](#excerpt-5-user-line-with-tool_result-error-is_error-true) (line 20 of `2072cbce-...jsonl`).

### Downstream Phase Impact

Verdict is `spec-correct-and-proceed` — no Phase 122 / Phase 123 goal shifts. Implementation notes for the corrected mapping rows:

- **Phase 122 implementation note (`needs_input` emission):** when emitting `needs_input`, read field path `$.type=="assistant" AND $.message.content[?(@.type=="tool_use")]` (combined with timeout + `$.permissionMode` check) instead of spec's `$.type=="tool_use" AND $.awaiting_approval`. The spec's top-level `tool_use` line type does not exist; replace it with the inner `content[].type` discriminator on `assistant` lines. Suppress emission when the most recent `user.permissionMode` is `"bypassPermissions"` (otherwise every Bash call in a bypass session generates spurious `needs_input` events).

- **Phase 122 implementation note (`task_failed` emission):** when emitting `task_failed`, read field path `$.type=="user" AND $.message.content[?(@.type=="tool_result" && @.is_error==true)]` instead of spec's `$.type=="tool_result" AND $.is_error==true`. The level-shift (top-level → inner content) applies symmetrically to both success and error variants.

- **Phase 122 implementation note (`task_complete` emission):** there is no `session_end` line type. Replace the spec's literal `$.type=="session_end"` rule with: "emit `task_complete` when `(now - latestLineTimestamp) > taskCompleteSeconds` AND the latest line's `$.message.stop_reason == "end_turn"` AND no further `user` line has been appended". Tune `taskCompleteSeconds` separately from heartbeat threshold — it is necessarily heuristic and conflates "user reading reply" with "user walked away".

- **Phase 122 implementation note (project-namespace enumeration):** the watcher daemon must enumerate ALL `~/.claude/projects/*/` subdirectories, NOT a single fixed namespace. Claude Code partitions JSONL by `cwd`-derived namespace, and a single user can produce JSONL into multiple namespaces over time (this Phase 120 work alone touched 2 namespaces because the user opened a fresh VS Code window at the parent folder rather than the dailybrief sub-folder). The `watch.toml` `projects_dir` key should default to `~/.claude/projects/` (the parent), not a single subdirectory.

- **Phase 122 implementation note (line-type noise filtering):** in addition to the 8-row spec mapping, the watcher will encounter 7 non-spec line types in real corpus traffic — `attachment`, `queue-operation`, `file-history-snapshot`, `last-prompt`, `ai-title`, `summary`, `system`. None carries Vigil-event signal. The parser should treat them as no-ops (advance offset, do not emit) but should NOT error on encountering them.

- **Phase 123 unaffected** — the CLI surface, launchd plist, and 24h-soak success criteria are valid against the corrected mapping above. No CLI flags or surface-area changes required for the spec corrections.

---

## Appendix: Raw JSONL Excerpts

Each excerpt below grounds a specific claim in the main body. Excerpts are hand-curated subsets of `verification-log/excerpts/*.jsonl` (gitignored, kept locally only). Long stdout/thinking/usage/attachment payloads have been truncated with `[truncated]` markers — the structural envelope is preserved so the field paths cited in the main body are verifiable, while incidental session content (filenames, code snippets, etc.) is not republished.

**Sanitization summary:**
- 1 long Bash stdout payload truncated (Scenario B Desktop listing — preserved JSON envelope, replaced 27-line file listing with `[truncated]`)
- 1 long `attachment.skill_listing.content` truncated (preserved structural envelope, replaced full skill listing with `[truncated]`)
- 1 long `attachment.deferred_tools_delta` truncated (preserved envelope, replaced 165-tool list with `[truncated]`)
- 2 `thinking` content blocks truncated (preserved envelope, replaced base64 signature payload with `[truncated]`)
- All `usage` objects truncated (preserved `{...}` placeholder)
- No `vk_*`, `sk-ant-*`, or `Bearer` tokens encountered — no credential redactions needed
- All `cwd` paths preserved as-is (user's home only — `/Users/jamesonmorrill/...`)
- All session/tool/UUID identifiers preserved as-is (no credential value, only graph identifiers)

---

### Excerpt 1: user prompt line, default permission mode

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl` (from `~/.claude/projects/-Users-jamesonmorrill-Desktop-Local-AI/`)
**Lines:** 16
**Captured:** 2026-05-08T18:08:24Z (Scenario C prompt)
**Sanitization applied:** none — line is structural metadata + a known test-prompt string only

```jsonl
{"parentUuid":"832353fe-1e58-433f-9acf-d119d155137d","isSidechain":false,"promptId":"4dd84e5f-2b5d-...","type":"user","message":{"role":"user","content":[{"type":"text","text":"Please run `ls /this/path/definitely/does/not/exist/xyz123` and report what happens."}]},"uuid":"d12fc9bc-...","timestamp":"2026-05-08T18:08:24.330Z","permissionMode":"default","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: `$.type == "user"`, `$.permissionMode == "default"` (NOT bypassPermissions, so the next tool_use will trigger an approval prompt), `$.message.content[0].type == "text"` for a typed user prompt.

---

### Excerpt 2: assistant line with stop_reason end_turn

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 26 (Scenario E reply to "Thanks, that's all for now.")
**Captured:** 2026-05-08T18:10:46Z
**Sanitization applied:** truncated `usage` object

```jsonl
{"parentUuid":"...","isSidechain":false,"message":{"model":"claude-opus-4-7","id":"msg_...","type":"message","role":"assistant","content":[{"type":"text","text":"Sounds good — let me know if you need anything else."}],"stop_reason":"end_turn","stop_sequence":null,"stop_details":null,"usage":"[truncated]"},"requestId":"req_...","type":"assistant","uuid":"d8a9c3...","timestamp":"2026-05-08T18:10:46.515Z","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: `$.type == "assistant"`, `$.message.stop_reason == "end_turn"` (the natural "Claude yielded" signal that combines with sustained silence to produce `task_complete`), `$.message.content[0].type == "text"` for prose replies.

---

### Excerpt 3: assistant line with inner tool_use block

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 11 (Scenario A — assistant calls Bash to list ~/Desktop)
**Captured:** 2026-05-08T18:07:31Z
**Sanitization applied:** truncated `usage` object

```jsonl
{"parentUuid":"...","isSidechain":false,"message":{"model":"claude-opus-4-7","id":"msg_01BpojnRntJZ8wFcHPK5V8sa","type":"message","role":"assistant","content":[{"type":"tool_use","id":"toolu_01AZQjRGQqPNJYpHVrAY68dG","name":"Bash","input":{"command":"ls -la ~/Desktop | wc -l && ls -la ~/Desktop","description":"List Desktop contents and count"},"caller":{"type":"direct"}}],"stop_reason":"tool_use","stop_sequence":null,"stop_details":null,"usage":"[truncated]"},"requestId":"req_...","type":"assistant","uuid":"cb75ba89-85c9-4f7d-b449-defa0519f1d5","timestamp":"2026-05-08T18:07:31.536Z","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: `$.type == "assistant"` (NOT `tool_use` at top level), inner `$.message.content[0].type == "tool_use"` carrying `name`, `input.command`, `id` (for matching to subsequent tool_result), and the top-level `$.message.stop_reason == "tool_use"`. This is the load-bearing pattern for `needs_input` detection — the spec's assumption that a top-level `type:"tool_use"` line exists is wrong; this is what's actually emitted.

---

### Excerpt 4: user line with tool_result success (is_error false)

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 12 (Scenario B — successful Bash result for ~/Desktop listing)
**Captured:** 2026-05-08T18:07:32Z
**Sanitization applied:** truncated 27-line Desktop file listing in both `content[0].content` and `toolUseResult.stdout` (preserves JSON shape, replaces user file inventory with `[truncated]`)

```jsonl
{"parentUuid":"cb75ba89-85c9-4f7d-b449-defa0519f1d5","isSidechain":false,"promptId":"392dd95e-...","type":"user","message":{"role":"user","content":[{"tool_use_id":"toolu_01AZQjRGQqPNJYpHVrAY68dG","type":"tool_result","content":"      30\ntotal 173816\ndrwx------@ 29 jamesonmorrill  staff       928 May  5 18:00 .\n[truncated — 27 user-private filenames]","is_error":false}]},"uuid":"0771265e-...","timestamp":"2026-05-08T18:07:32.301Z","toolUseResult":{"stdout":"      30\ntotal 173816\n[truncated]","stderr":"","interrupted":false,"isImage":false,"noOutputExpected":false},"sourceToolAssistantUUID":"cb75ba89-85c9-4f7d-b449-defa0519f1d5","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: `$.type == "user"` (NOT `tool_result` at top level), inner `$.message.content[0].type == "tool_result"`, `$.message.content[0].tool_use_id` matches the prior assistant tool_use's `id` (for correlation), `$.message.content[0].is_error == false`, and the camelCase sibling `$.toolUseResult.{stdout, stderr, interrupted, isImage, noOutputExpected}` object.

---

### Excerpt 5: user line with tool_result error (is_error true)

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 20 (Scenario C — forced ENOENT from `ls /this/path/definitely/does/not/exist/xyz123`)
**Captured:** 2026-05-08T18:08:32Z
**Sanitization applied:** none — error stdout is the test-fixture string only

```jsonl
{"parentUuid":"...","isSidechain":false,"promptId":"4dd84e5f-...","type":"user","message":{"role":"user","content":[{"type":"tool_result","content":"Exit code 1\nls: /this/path/definitely/does/not/exist/xyz123: No such file or directory","is_error":true,"tool_use_id":"toolu_012tub88PzdyzAoQLDbkVQrL"}]},"uuid":"e3b1...","timestamp":"2026-05-08T18:08:32.237Z","toolUseResult":{"stdout":"Exit code 1\nls: /this/path/definitely/does/not/exist/xyz123: No such file or directory","stderr":"","interrupted":false,"isImage":false,"noOutputExpected":false},"sourceToolAssistantUUID":"...","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: identical envelope to Excerpt 4 with the SOLE single-field discriminator `$.message.content[0].is_error == true`. The `$.toolUseResult` remains a structured object (NOT a string starting `"Error: …"` as some older corpus lines showed). This is the deterministic single-field signal for `task_failed`.

---

### Excerpt 6: session tail — no explicit end marker

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 32-33 (final two lines of the scripted session — the file simply stopped appending after these)
**Captured:** 2026-05-08T18:11:21Z
**Sanitization applied:** none — these are tail-marker lines only

```jsonl
{"type":"last-prompt","lastPrompt":"Thanks, that's all for now.","leafUuid":"55d8c860-...","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a"}
{"type":"ai-title","aiTitle":"Check Desktop directory contents","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a"}
```

Demonstrates: NO `$.type == "session_end"` line was written when the user clicked away from the conversation. The file ends on the standard rewrite-style tail markers `last-prompt` (which mirrors the latest user prompt at any time) and `ai-title` (auto-generated session label). Of 184 corpus files mined, 80% end on `last-prompt` for the same reason — it is the most-recently-rewritten line at any quiescent moment, not a deliberate end-signal. Detection of session-end therefore requires sustained silence + `stop_reason` heuristic, NOT a literal field lookup.

---

### Excerpt 7: silence gap (127s between consecutive lines)

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 21 and 22 (Scenario D — last pre-silence line + first post-silence line)
**Captured:** 2026-05-08T18:08:34 → 18:10:42 (gap = 127.605 seconds)
**Sanitization applied:** truncated `usage` object on the assistant reply line

```jsonl
{"parentUuid":"...","isSidechain":false,"message":{"model":"claude-opus-4-7","id":"msg_...","type":"message","role":"assistant","content":[{"type":"text","text":"The command failed as expected:\n\n- **Exit code:** 1\n- **stderr:** `ls: /this/path/...`"}],"stop_reason":"end_turn","stop_sequence":null,"stop_details":null,"usage":"[truncated]"},"requestId":"req_...","type":"assistant","uuid":"f21a...","timestamp":"2026-05-08T18:08:34.843Z","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
{"type":"queue-operation","operation":"enqueue","timestamp":"2026-05-08T18:10:42.448Z","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a"}
```

Demonstrates: silence detection is purely a `$.timestamp` delta computation — `2026-05-08T18:10:42.448Z` minus `2026-05-08T18:08:34.843Z` = **127.605s** gap, observed at the file-write level (no JSONL line type is needed for `heartbeat`). The watcher tails the file and computes `now() - latestLineTimestamp` continuously; when the delta exceeds `heartbeatSeconds`, it emits `heartbeat` (and the user can configure separate thresholds for `task_complete` and session-end).

---

### Excerpt 8: assistant text content block (grep target for milestone regex)

**Source:** `2072cbce-9eb3-4f2d-a69d-a219d997aa2a.jsonl`
**Lines:** 13 (Scenario A — assistant's prose reply with the count)
**Captured:** 2026-05-08T18:07:36Z
**Sanitization applied:** truncated `usage` object; replaced personal directory names in the reply text with generic placeholders (this excerpt grounds the structural shape, not the specific filenames)

```jsonl
{"parentUuid":"...","isSidechain":false,"message":{"model":"claude-opus-4-7","id":"msg_01E47Z3q3taT4hHd6YP81pcc","type":"message","role":"assistant","content":[{"type":"text","text":"Your `~/Desktop` contains **27 items** (excluding `.` and `..`):\n\n- 4 directories: [truncated — directory names redacted as user-private]\n- 22 files (including `.DS_Store`, `.localized`, PDFs, images, and notes)\n\nTotal: **27 entries** in the directory."}],"stop_reason":"end_turn","stop_sequence":null,"stop_details":null,"usage":"[truncated]"},"requestId":"req_...","type":"assistant","uuid":"832353fe-1e58-433f-9acf-d119d155137d","timestamp":"2026-05-08T18:07:36.433Z","userType":"external","entrypoint":"claude-vscode","cwd":"/Users/jamesonmorrill/Desktop/Local AI","sessionId":"2072cbce-9eb3-4f2d-a69d-a219d997aa2a","version":"2.1.133","gitBranch":"main"}
```

Demonstrates: `$.message.content[?(@.type=="text")].text` is the field path against which user-defined regex patterns from `watch.toml` are matched to emit `milestone` events. The text field is plain UTF-8 markdown — no JSON-escaping concerns beyond standard backslash unescaping that any JSONL parser already handles.

---

## After Phase 120

Once verification is locked, Phase 122 fills this repo with the Swift daemon implementation per the verdict captured above.
