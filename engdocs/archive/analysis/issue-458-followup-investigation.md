---
title: "#458 follow-up — static investigation results (items 2 and 5)"
status: working notes
target: feeds the third comment on gastownhall/gascity#458
references:
  - https://github.com/gastownhall/gascity/issues/458
  - engdocs/archive/analysis/issue-458-comment-draft.md
  - engdocs/archive/analysis/issue-458-comment-2-draft.md
---

Working notes from running the parts of the punch-list (in
`issue-458-comment-2-draft.md`) that don't require a live deployment.
Live items (1, 2-cadence, 3) are in
`issue-458-live-deployment-instructions.md`.

## What I could and could not run

| Item | Description | Status |
|---|---|---|
| (1) `/context` baseline | Per-turn overhead on current Claude Code | **cannot** — needs interactive `/context` in a Claude Code terminal attached to a live patrol session |
| (2) static | Read the four wakeup-fix commits cited in #458 | **partial** — see "SHA mismatch" below |
| (2) cadence | Live-measured wakeup interval after fixes | **cannot** — needs running deployment |
| (3) `exec:` win quantification | Compare LLM vs `exec:` cost for the deterministic role | **cannot** — needs running deployment |
| (5) Client-side `--rig` feasibility | Code analysis | **done** — see "Client-side rig filter" below |

## Item (2) static — SHA mismatch

The four short SHAs cited in #458 (`513a1d12`, `9af96b94`, `73f3e1da`,
`1543a77f`) **do not resolve in this checkout** (`esciara/gascity`,
branch `claude/review-claude-context-tokens-9paxM`). `git log --all`
finds none of them.

Search by message keyword turned up one thematically-adjacent commit:

> **cf64aca** (2026-04-28) — `fix(session): preserve in_progress claims
> across worker churn`. Three interacting bugs that orphaned in_progress
> beads when pool sessions churned. Restricts `reapStaleSessionBeads` to
> sessions stuck in the "creating" state (or with `pending_create_claim`
> set), resets cleared-assignee in_progress beads to `open` so a fresh
> worker can re-claim, and refuses to close a session bead while
> non-session work is still assigned to it.

That commit covers similar ground to #458's "preserve
`pending_create_claim` across named session reopen" but is a different
SHA, so the fork is either behind on the upstream fixes or has them
under different commit hashes (squash/rebase).

**Implication:** the prior comment's claim that the suspension fix
"already shipped (commit 513a1d12)" cannot be verified in this fork.
Whoever runs the live re-measurement (item 1, 2-cadence, 3) must
**explicitly verify the commit list** on the target deployment — pick
either upstream-by-SHA or build from the fork on a known commit and
record which fixes are in.

## Item (5) — client-side `--rig` feasibility (the real picture)

My earlier "30-minute fix" estimate was optimistic. Looking at the code,
client-side rig filtering splits into three tiers depending on which
event types you want to cover.

### What rig info events actually carry

Event envelope (`internal/events/events.go:91-99`):

```go
type Event struct {
    Seq     uint64          `json:"seq"`
    Type    string          `json:"type"`
    Ts      time.Time       `json:"ts"`
    Actor   string          `json:"actor"`
    Subject string          `json:"subject,omitempty"`
    Message string          `json:"message,omitempty"`
    Payload json.RawMessage `json:"payload,omitempty"`
}
```

**No `Rig` at the envelope level.** Per-event-type coverage in
`internal/api/event_payloads.go`:

| Event types | Payload | Rig location |
|---|---|---|
| `mail.*` (7 types) | `MailEventPayload` | **top-level `rig`** (line 24) |
| `bead.*` (3 types) | `BeadEventPayload{Bead}` | nested at `bead.rig` |
| `session.*`, `convoy.*`, `controller.*`, `city.suspended/resumed`, `order.*`, `provider.*` | `NoPayload{}` | none in payload — must be inferred from `Actor`/`Subject` |
| `city.*` lifecycle (created/ready/init_failed/unregister*) | `CityLifecyclePayload` | none (city name only, line 36) |
| `worker.operation` | `WorkerOperationEventPayload` | none (lines 79-94) |
| `extmsg.*` | (defined in `internal/extmsg`) | not surveyed |

### What `--payload-match` already supports

`matchPayloadObject` (`cmd/gc/cmd_events.go:1388-1407`) only checks
**top-level keys**:

```go
for key, wants := range payloadMatch {
    value, ok := obj[key]
    if !ok { return false }
    ...
}
```

That means **today, with no code change**:

- `gc events --watch --type mail.sent --payload-match=rig=$GC_RIG`
  works (rig is top-level on `MailEventPayload`).
- `gc events --watch --type bead.updated --payload-match=rig=$GC_RIG`
  **does not work** (rig is at `bead.rig`, one level deep).
- `gc events --watch --type session.woke --payload-match=rig=$GC_RIG`
  **does not work** (no payload).

So mail-only filtering is essentially free. Everything else needs a
small change.

### Implementation tiers

**Tier 1 — dotted-path payload-match** (estimated 1–2 hours)

Teach `matchPayloadObject` to descend on `.`-separated keys:
`--payload-match=bead.rig=$GC_RIG` would work for `bead.*` events.
Local change to `cmd/gc/cmd_events.go`, plus tests in
`cmd_events_test.go`. No API change, no Huma/OpenAPI/TS regeneration.

This alone covers the high-volume case from #458 (the ~45/min
`bead.updated` storm) without any architectural review. It's the
cheapest viable cut and a likely good first lever to pull.

**Tier 2 — `--rig` flag with multi-source extraction** (estimated half-day)

Add `--rig $GC_RIG` as a first-class flag. The CLI normalises rig
extraction across event types: top-level `rig` for `mail.*`, nested
`bead.rig` for `bead.*`, parsed from `Actor`/`Subject` for the
`NoPayload{}` types. Policy decision: events with no rig info are
either always passed through (`--rig` is opt-in narrow filter) or
always filtered out (`--rig` is opt-in strict filter). Pick one and
document.

Still client-side, still no API change. The work is mostly bookkeeping
and tests; the policy call is the only judgment-y bit.

**Tier 3 — server-side rig filter** (estimated multi-day, design-doc required)

Promote `Rig` to the `Event` envelope, add `Rig string \`query:"rig"\``
to `EventStreamInput`, push the predicate into `Provider.Watch`, update
the SSE handler to filter before sending, regenerate the OpenAPI spec,
regenerate the TS dashboard types. This crosses every load-bearing
invariant in `AGENTS.md`: typed-wire, typed-event registry coverage,
`TestOpenAPISpecInSync`, `TestEveryKnownEventTypeHasRegisteredPayload`.
This is the only tier that reduces wire volume — Tiers 1 and 2 still
ship every event over the wire, the CLI just filters before printing.

### Cost calculus

If the goal is to reduce **witness wakeups** (which seems to be the
real driver behind #458's recommendation), Tiers 1 and 2 are sufficient
because the witness only wakes when the CLI returns a match. Wire
volume is irrelevant to LLM cost; only CLI-return frequency matters.

If the goal is to reduce **dashboard event volume** or **third-party
SSE consumer load**, Tier 3 is needed — but that's a different
motivation than the one in #458, and it should be argued separately.

So the recommendation should be: **start at Tier 1**, measure the
witness-wakeup reduction, decide whether to escalate.

## Net advice for #458

1. The "30-minute client-side fix" framing was wrong; the correct
   framing is "Tier 1 is cheap and covers the dominant case." Update
   the prior comment or post a correction.
2. The SHA mismatch means *don't* assert the suspension fix is in this
   fork without checking. Anyone re-measuring needs to record the
   commit they're on.
3. The architectural ask (server-side filter on the event bus) is
   **not justified by anything in #458 alone** — Tier 1/2 already
   solves the witness-wakeup problem without touching the wire. A
   separate motivation (wire volume / dashboard / third-party SSE)
   would be needed to open a design doc.

## What goes in the third comment

A short follow-up to the second comment, with:

- The Tier 1/2/3 framing and cost estimates
- The mail-only-works-today observation
- The note that the witness-wakeup goal is fully addressable client-side
- The SHA mismatch caveat (the prior comment shouldn't have asserted
  the fixes are in)
