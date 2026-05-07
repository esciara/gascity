---
title: "Draft second comment for gastownhall/gascity#458 — code-state findings"
status: draft
target: https://github.com/gastownhall/gascity/issues/458
---

This file is a draft of a follow-up comment intended to be posted on
`gastownhall/gascity#458` after the first comment (see
`issue-458-comment-draft.md` in this directory). Once posted, this file
can be deleted.

---

## Draft comment

> Code-state check on the two recommendations. Picture is asymmetric.

## 1. `exec:` provider — already shipped at the runtime layer

The mechanism this issue calls for is already a first-class session provider:

| Piece                                                                       | Location                                                                                                                                     |
|-----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Provider registry recognises `exec:<script>`                                | `cmd/gc/providers.go:118-149` (dispatch at L119-121 → `sessionexec.NewProvider`)                                                             |
| Full lifecycle implementation (Start/Stop/Peek/SendKeys/Nudge/SetMeta/etc.) | `internal/runtime/exec/exec.go`                                                                                                              |
| Script contract (operation as `$1`, JSON on stdin, exit codes 0/1/2)        | same file; covers `start`, `stop`, `is-running`, `peek`, `send-keys`, `nudge`, `attach`, `process-alive`, `watch-startup`, plus metadata ops |
| `Agent.Provider` field carries the value through config                     | `internal/config/config.go:1512`                                                                                                             |
| `AgentPatch.Provider` + `AgentOverride.Provider` + apply funcs              | `internal/config/patch.go:45,280-282`; `internal/config/config.go:475`                                                                       |
| `cmd/gc/pool.go` deep-copy of `Provider`                                    | `cmd/gc/pool.go:263`                                                                                                                         |
| Reconciler / sling is provider-agnostic                                     | `internal/sling/sling.go:215-290` (uses `runtime.Provider` interface)                                                                        |
| Low-level provider tests + mock script                                      | `internal/runtime/exec/exec_test.go` (mock at L1094-1173, integration at L1174+)                                                             |

What's missing for the recommendation in this issue is **documentation and a worked example**, not the feature itself. I'm filing a separate, narrowly-scoped issue to track that so it can land without waiting for the broader discussion here. Closing condition: documented TOML pattern + reference patrol script + optional dispatch test.

## 2. Event filtering — server-side filtering is genuinely missing

Current state:

| Piece                                                                                        | Status                                             | Location                                                                                     |
|----------------------------------------------------------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------|
| `gc events --watch`, `--type`, `--payload-match`, `--since`, `--after`, `--seq`, `--timeout` | exist, **filtering is client-side only**           | `cmd/gc/cmd_events.go:147-155`; filters applied at L1092-1097 + L1203-1207 (after streaming) |
| `gc events --rig=$GC_RIG`                                                                    | missing                                            | no flag                                                                                      |
| `events.Filter` struct (Type/Actor/Since/AfterSeq)                                           | exists but **only used by the local JSONL reader** | `internal/events/reader.go:12-18`                                                            |
| `Provider.Watch(ctx, afterSeq)` accepts a filter                                             | no                                                 | `internal/events/events.go:111-127`                                                          |
| `Multiplexer.Watch` filter parameter                                                         | no                                                 | streams all events from all cities                                                           |
| SSE `/v0/events/stream` server-side filter                                                   | no                                                 | `internal/api/huma_handlers_events.go:137-200` — every event, every subscriber               |
| "Actionable event" concept (`wake_reason`, actionability flag)                               | no                                                 | no marker; `bead.updated` is emitted uniformly                                               |

This recommendation crosses at least `events.Provider`, `Multiplexer`, the SSE handler (which is Huma-typed and OpenAPI-checked in CI), `EventStreamInput`, and `cmd/gc/cmd_events.go`. Per `AGENTS.md`, anything touching `internal/api/` or `internal/events/` lands through the typed-wire / typed-event invariants and a design doc in `engdocs/design/`.

I don't want to formalize that work yet. The per-turn token math in this issue (~94–104K) was based on pre-deferral Claude Code; the real per-no-op cost on current Claude Code may be an order of magnitude smaller, which would change whether this is worth the architectural work at all. Before opening a design doc I'd want:

1. `/context` output from a live patrol session on current Claude Code, to re-baseline the per-turn overhead.
2. A re-measured wakeup cadence after the suspension fix (513a1d12), `pending_create_claim` preservation (9af96b94), and especially wait-held wake suppression (73f3e1da) — all of which post-date or coincide with the original investigation.
3. Confirmation of whether moving the deterministic role to the existing `exec:` provider alone closes enough of the cost gap that this becomes a "nice to have" instead of a P0 architectural change.
4. A read on whether the proximate user pain is solvable by a client-side `--rig` filter on `cmd_events.go` (cheap, no API change) vs. genuinely needing server-side predicate subscription on the wire.

Happy to gather any of (1)–(4) here if someone has a witness running. (or will try to do it myself)
