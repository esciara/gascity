---
title: "PreCompact Handoff Mail — Injection Timing"
---

> Investigation note. Captures the runtime behavior of the
> `PreCompact → gc handoff --auto` hook and the open concern that the
> handoff note can be re-injected into an already-compacted, already-used
> context window — potentially duplicating information that is still
> present. Written for further study; not a settled design.

## The concern in one line

A handoff note created at compaction time is **durable** but its
**re-injection is not synchronized to the compaction boundary**. It is
pulled back on the *next prompt submission*, which in an autonomous run is
not necessarily the post-compaction turn — so it may land late, into a
window that has already moved on, duplicating context the model still has.

## The three GC-managed hooks (Claude config)

Wired in `internal/hooks/config/claude.json`:

| Claude event | GC command | Purpose |
|---|---|---|
| `SessionStart` | `gc prime --hook --hook-format codex` | Load context at startup, drain hooks |
| `PreCompact` | `gc handoff --auto "context cycle"` | Capture state just before compaction |
| `UserPromptSubmit` | `gc nudge drain --inject` **and** `gc mail check --inject` | Inject pending nudges + unread mail into the upcoming prompt |

Cross-provider event names and coverage are tabulated in
`internal/hooks/config/README.md`. Notably, `gemini`, `omp`, and `pi`
have **no** "user prompt submit" event and fall back to a "before agent
run" event; `opencode` has neither wired today (parity gap).

## What `gc handoff --auto` actually does

From `cmd/gc/cmd_handoff.go` (doc comment, lines ~48-50):

> *"Auto handoff (`--auto`): sends mail to self and returns without
> requesting a restart. This is for PreCompact hooks, where the provider
> is already managing the context compaction lifecycle."*

Concretely, on `PreCompact` it:

1. **Creates a self-addressed handoff mail** — a bead with
   `sender == recipient == this session`, titled `context cycle`
   (`createHandoffMail(store, rec, sessionAddress, sessionAddress, …)`,
   `cmd/gc/cmd_handoff.go:274`). Durable, `open`, unread.
2. **Returns without requesting a restart** — unlike a normal
   `gc handoff`, which both mails and restarts. `--auto` deliberately
   skips the restart because the provider is compacting, not cycling.
3. **Also writes the message to the hook's stdout**
   (`writeProviderHookContextForEvent(stdout, hookFormat, "PreCompact", message)`,
   `cmd/gc/cmd_handoff.go:279`).

### Important nuance about the stdout path

`writeProviderHookContextForEvent` only emits a structured
`hookSpecificOutput.additionalContext` envelope for the `codex` and
`gemini` formats (`cmd/gc/hook_output.go:19-31`). In Claude's config the
PreCompact command carries **no `--hook-format`**, so `format == ""` and
the message is written as **plain stdout**, not a structured
`additionalContext` envelope. Whether Claude Code folds PreCompact stdout
into the compaction summary is **provider behavior not verifiable from
this repo** — so we cannot assume the stdout path reliably carries the
note across the compaction boundary.

## Claude Code hook lifecycle facts that matter

(See https://code.claude.com/docs/en/hooks — hook lifecycle.)

- `UserPromptSubmit` fires on **every** prompt submission, throughout the
  session — *not* only once after `SessionStart`.
- `PreCompact` fires before a compaction, including **auto-compaction**,
  which triggers mid-agentic-loop when the window fills up while the agent
  is working through tool calls — with **no** human/nudge prompt submission
  around it.
- After an auto-compaction, the agent typically **continues** on the
  compacted summary. There is no guaranteed `UserPromptSubmit` immediately
  afterward, so `gc mail check --inject` does **not** necessarily run on
  the next turn.

## Guaranteed vs. not guaranteed

| Property | Status |
|---|---|
| The handoff note is not **lost** | **Guaranteed** — it is a persistent bead, unread until read (NDI). It will be injected on *some* future `UserPromptSubmit`, possibly in a later session. |
| The note lands in the **freshly-compacted** window | **Not guaranteed** — re-injection timing is opportunistic, decoupled from the compaction event. |
| The note arrives **exactly once, at the right time** | **Not guaranteed** — see the duplication concern below. |

## The duplication / late-injection concern (to study)

Because re-injection is gated on the next `UserPromptSubmit`, and that
event is not aligned with compaction, several awkward orderings are
possible:

1. **Late injection.** The agent runs N autonomous turns after
   auto-compaction with no prompt submission. The handoff note sits unread
   the whole time, then is injected on the eventual next prompt — by which
   point the agent has already re-derived or moved past that context.
2. **Redundant injection.** If compaction *did* preserve the relevant
   information in its summary (or the agent never actually lost it), the
   later-injected handoff note **duplicates** content already in the
   window — spending tokens and possibly re-anchoring the model on stale
   framing.
3. **Window pressure.** Injecting a handoff note into a window that is
   again near-full (or freshly compacted and already partly re-used) can
   re-pressure the context it was meant to relieve.
4. **Mis-timed in a multi-purpose hook.** `UserPromptSubmit` also carries
   peer mail (`gc mail check --inject`, `cmd/gc/cmd_mail.go:586`). The
   self-handoff rides the same injection path as genuine inter-agent mail,
   so a self "context cycle" note can surface mixed in with unrelated
   incoming mail at an arbitrary prompt.

## Open questions

- Does Claude Code inject **PreCompact** hook stdout into the compaction
  summary? If yes, the self-mail is a redundant second channel; if no, the
  self-mail is the only channel and its timing problem stands.
- Should the handoff self-mail be **marked/segregated** so injection logic
  can dedupe it against what compaction already retained?
- Should re-injection be **gated** (e.g. only inject a `context cycle`
  self-note if it is the first prompt after a compaction, then archive it)
  rather than riding the generic unread-mail path?
- Is there a cleaner mechanism than self-mail for surviving compaction —
  e.g. a structured `additionalContext` on a post-compaction event, if one
  exists per provider?
- How do non-`UserPromptSubmit` providers (gemini/omp/pi via "before agent
  run", opencode with nothing wired) change the timing and the duplication
  risk?

## Code references

- `internal/hooks/config/claude.json` — the three managed hooks.
- `internal/hooks/config/README.md` — cross-provider event vocabulary.
- `cmd/gc/cmd_handoff.go:~48-50, 274, 279` — `--auto` semantics, self-mail,
  PreCompact stdout.
- `cmd/gc/hook_output.go:19-64` — per-format hook output; plain stdout for
  unspecified format.
- `cmd/gc/cmd_mail.go:586, 692` — `gc mail check --inject` output path.
