---
title: "Draft new issue: document exec: provider as deterministic-role pattern"
status: draft
target: new issue, esciara/gascity (or upstream)
references:
  - https://github.com/gastownhall/gascity/issues/458
---

This file is a draft of a new issue, split out of
`gastownhall/gascity#458`, narrowly scoped to the `exec:` provider
documentation work. Once filed, this file can be deleted.

---

## Draft issue

**Title:** Document `exec:` provider as the recommended path for
deterministic patrol roles

**Labels:** `kind/docs`, `priority/p2`

---

## Body

The `exec:` script provider is already a first-class session provider in gascity, but it's not documented as a deployment pattern. This issue tracks the documentation work split out of `gastownhall/gascity#458`, where one of the recommendations is *"use script providers (`exec:`) as defaults for deterministic roles"* — the mechanism is shipped, only docs and an example are needed.

## What's already in place

- Provider registry: `cmd/gc/providers.go:118-149` recognises `exec:<script>` and dispatches to `sessionexec.NewProvider`.
- Full lifecycle implementation:
  `internal/runtime/exec/exec.go` (Start/Stop/Peek/SendKeys/Nudge/SetMeta and metadata ops).
- Script contract uses the Git-credential-helper style: operation name as `$1`, JSON config on stdin, exit codes 0 (success) / 1 (error) / 2 (unknown op — forward-compatible).
- `Agent.Provider` plumbs through config:
  `internal/config/config.go:1512`, with `AgentPatch.Provider`, `AgentOverride.Provider`, apply functions, and the `cmd/gc/pool.go:263` deep-copy all in place.
- Reconciler dispatch is provider-agnostic via `runtime.Provider`:
  no LLM assumption (`internal/sling/sling.go:215-290`).
- Low-level tests and a mock provider script:
  `internal/runtime/exec/exec_test.go:1094-1173`.

## What's missing

- A documented TOML pattern showing how to declare an `[[agent]]` that uses `provider = "exec:..."` for a deterministic role.
- A reference patrol script (the `gc-witness-patrol.sh` shipped in the body of `gastownhall/gascity#458` is a reasonable starting point, adapted to the `exec:` script contract).
- An end-to-end / integration test demonstrating the deterministic patrol path through the reconciler (current tests cover the provider in isolation, not the dispatch context).
- A short note in `engdocs/` explaining when to reach for `exec:` vs an LLM-backed provider, framed against the ZFC principle in `AGENTS.md` (judgment-free transport → no model needed).

## Acceptance criteria

- [ ] TOML pattern documented in engdocs (with a worked `[[agent]]` example)
- [ ] Reference patrol script committed somewhere discoverable
- [ ] Integration test exercising the dispatch path with an exec-backed agent (extends the mock provider in `exec_test.go` if useful)
- [ ] Decision-rule note: when to use `exec:` vs LLM (one paragraph, cross-linked from `engdocs/contributors/primitive-test.md`)

## Out of scope

- Event filtering — under discussion in `gastownhall/gascity#458`.
- Any change to the provider abstraction itself.
- Any specific role name. Per the ZFC principle, role names don't appear in Go code; this issue is about the provider mechanism, not any specific role.

## Cost context (why this matters)

See `gastownhall/gascity#458` for the original cost analysis. Note that the per-turn overhead figures cited in that issue (~94–104K tokens)predate Claude Code's ToolSearch deferral (v2.1.69+) and need re-measurement — but the rationale for moving deterministic roles off LLMs entirely (cost goes to zero, no judgment is involved) is independent of that re-measurement.
