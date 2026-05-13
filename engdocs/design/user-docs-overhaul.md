---
title: User Docs Overhaul
description: Plan to make `docs/` usable for end users — especially Gas Town users migrating to Gas City — rather than a half-contributor half-user mix.
status: Proposed
---

# User Docs Overhaul

**Status:** Proposed — first draft, expected to evolve before issues are cut.

**Audience this plan targets:** engineers who want to *use* Gas City (not
contribute to it). The motivating persona is a Gas Town user evaluating or
executing a switch to Gas City.

**Scope:** `docs/` (the Mintlify site). Contributor material in `engdocs/`
is in scope only as a *source* — content that should be promoted, adapted,
or referenced from `docs/`.

**Out of scope:** contributor-facing docs in `engdocs/`, CLAUDE/AGENTS
files, internal architecture rewrites.

---

## 1. Premise

The current `docs/` tree is roughly 75% solid: installation, quickstart,
and the seven tutorials work and are progressive. But the conceptual
foundation lives in `engdocs/` (contributor docs), tutorials are split
between PackV1 and PackV2 syntax, and several user-facing pages still
send users into `engdocs/`. A new user — especially a Gas Town migrator —
will bounce between conflicting sources.

The goal of this overhaul is to make `docs/` self-sufficient for users:
no link to `engdocs/` from a user-facing page should be required to
understand or operate Gas City.

---

## 2. Findings (initial critical analysis)

Numbered for traceability into the work plan in §4. Severity:
**P0** (blocks user onboarding), **P1** (significant friction), **P2**
(polish / longer-term).

### F1 — Mental model lives in the wrong repo (P0)

The Nine Concepts (Session, Task Store, Event Bus, Config, Prompt
Templates, Messaging, Formulas, Dispatch, Health Patrol) — the entire
conceptual spine of the SDK — are documented in
[engdocs/architecture/nine-concepts.md](../architecture/nine-concepts.md)
and never explained in `docs/`.

- [docs/index.mdx](../../docs/index.mdx) tells users to read `engdocs/`.
- [docs/getting-started/coming-from-gastown.md](../../docs/getting-started/coming-from-gastown.md)
  explicitly sends Gas Town migrators to
  `engdocs/architecture/nine-concepts`.

Relevant engdocs source material that should be promoted/adapted:

- `engdocs/architecture/nine-concepts.md`
- `engdocs/architecture/glossary.md`
- `engdocs/architecture/life-of-a-bead.md`
- `engdocs/architecture/life-of-a-molecule.md`
- `engdocs/architecture/index.md`

These are written for contributors today; they need a user-tone
rewrite (less "invariant" / "layer N", more "what this is and why
you care").

### F2 — PackV1 / PackV2 contradiction in the tutorial path (P0)

[Tutorial 02 — Agents](../../docs/tutorials/02-agents.md) and
[Tutorial 03 — Sessions](../../docs/tutorials/03-sessions.md) still use
the legacy `[[agent]]` TOML blocks, while
[coming-from-gastown.md](../../docs/getting-started/coming-from-gastown.md)
and [guides/shareable-packs.md](../../docs/guides/shareable-packs.md)
teach the PackV2 `agents/<name>/` directory layout. A user following
tutorials in order will write config that contradicts the migration
guide they hit next.

Note: project is currently at **v1.1.0**, well past the PackV2 cutover.
PackV1 syntax in tutorials is unambiguous staleness, not a pre-release
hedge.

### F3 — Gas Town migration guide assumes you still remember Gas Town (P1)

[coming-from-gastown.md](../../docs/getting-started/coming-from-gastown.md)
maps roles ("Deacon watchdog logic → Controller and supervisor")
without recalling what each Gas Town role actually *did*. A user who
last touched Gas Town six months ago, or who is evaluating whether to
migrate at all, cannot recover operational meaning from these mappings
alone.

### F4 — Migration guide points at files that aren't in the docs (P1)

The "Fast Ramp Checklist" in `coming-from-gastown.md` references
`examples/gastown/city.toml` and similar repo paths. These are not
embedded in the rendered docs, nor linked as downloadable assets, nor
guaranteed to exist at those paths. Users following the rendered site
dead-end.

### F5 — Contributor material leaking into user docs (P1)

- [docs/packv2/](../../docs/packv2/) holds 9 files; only
  `migration.mdx` is in the user nav. The rest
  (`doc-pack-v2.md`, `doc-agent-v2.md`, `doc-loader-v2.md`,
  `doc-rig-binding-phases.md`, `doc-conformance-matrix.md`,
  `doc-consistency-audit.md`, `doc-packman.md`, `doc-commands.md`,
  `skew-analysis.md`) read as contributor scratch.
- [docs/internals/](../../docs/internals/) is labeled "not required
  reading" but `beads-topology.md` is exactly what an operator needs.
- [docs/index.mdx](../../docs/index.mdx) literally states the tree is
  "organized for external contributors first" — wrong primary
  audience for `docs/`.

### F6 — Reference docs are complete but not navigable (P1)

- [reference/config.md](../../docs/reference/config.md): flat
  auto-generated field dump, no grouped "Common Patterns".
- [reference/formula.md](../../docs/reference/formula.md): mentions
  conditions/loops/checks but never shows a working example.
- [reference/api.md](../../docs/reference/api.md): points at OpenAPI
  with no "how do I query my city" narrative.
- [reference/cli.md](../../docs/reference/cli.md): comprehensive but
  flat; no task-oriented entry points.

### F7 — Significant workflow gaps (P1)

Things a real user will hit and not find:

- How to write a first agent prompt from scratch (template variables,
  prompt design).
- Debugging a stuck or failing session (`gc session peek` output,
  detection, kill/restart).
- Choosing between crew (persistent) and polecats (on-demand).
- Hooks and external integrations.
- Multi-machine / Kubernetes deployment.
- Health monitoring, what `gc doctor [--fix]` actually does.

Source material exists in `engdocs/architecture/` (e.g.
`session.md`, `health-patrol.md`, `dispatch.md`, `prompt-templates.md`)
and `engdocs/design/machine-wide-supervisor-v0.md` for scaling. None
is promoted to user docs.

### F8 — No full end-to-end example (P1)

There is no canonical "minimal complete city" showing `pack.toml` +
`city.toml` + `agents/<name>/agent.toml` + `prompt.md` + a formula +
an order, together. Each piece is shown in isolation across different
tutorials; assembly is left to the reader.

### F9 — Single-runbook troubleshooting catalog (P2)

[docs/troubleshooting/](../../docs/troubleshooting/) holds one
runbook (`dolt-bloat-recovery.md`). The Oh-My-Zsh `gc`-alias trap —
a real blocker for affected users — is in
[getting-started/troubleshooting.md](../../docs/getting-started/troubleshooting.md)
but should be a prerequisite check at the top of Quickstart.

### F10 — Staleness signals (P2)

- [Tutorial 01 line ~26](../../docs/tutorials/01-cities-and-rigs.md)
  shows version `v0.13.4` in sample output. Current tag is
  **v1.1.0**.
- [docs/index.mdx](../../docs/index.mdx): "organized for external
  contributors first" — contradicts the goal of `docs/`.
- General audit needed for stale concept names (the former Agent
  Protocol primitive was removed `dd90ac0a` on 2026-03-08; user docs
  appear clean but worth a sweep).

---

## 3. Cross-cutting principles for the overhaul

Before issues are cut, the following are non-negotiable so individual
PRs stay coherent:

1. **`docs/` is user-first.** No user-facing page may link into
   `engdocs/` as required reading. If `engdocs/` content is needed,
   promote it (with rewrite) into `docs/`.
2. **One pack syntax in tutorials.** PackV2 (`agents/<name>/`) is
   canonical. PackV1 lives only in a clearly-labeled migration
   appendix.
3. **Every concept page has a runnable example.** No conceptual page
   ships without at least one snippet a user can copy-paste.
4. **Examples are versioned with the docs.** Reference output uses
   the current release (v1.1.0 at time of writing). Add a CI check
   that flags hard-coded version strings drifting from the current
   tag (or use placeholder substitution at build time).
5. **`engdocs/` stays the source of truth for contributors.** When
   content is promoted, the original stays as a deeper contributor
   reference; the user-doc version links *back* (one-way).

---

## 4. Work plan (candidate GitHub issues)

This is the first cut. Each item below is sized roughly to be a single
PR / issue. Grouped by milestone so we can sequence cleanly.

### Milestone 1 — Stop the bleeding (P0, blockers)

**Issue 1.** *Add a user-facing "Mental Model" page to `docs/`.*
- New page: `docs/concepts/mental-model.md` (or equivalent in IA).
- Source: promote and rewrite `engdocs/architecture/nine-concepts.md`
  and `glossary.md` for a user audience.
- Update `docs/index.mdx` and `docs/getting-started/coming-from-gastown.md`
  to link to the new page instead of `engdocs/`.
- Add to `docs/docs.json` nav.

**Issue 2.** *Migrate Tutorials 02 and 03 to PackV2 syntax.*
- Rewrite `agents/<name>/agent.toml` + `prompt.md` flow.
- Verify all cross-references (Tutorial 04+, guides) stay
  consistent.
- Move the existing PackV1 examples to a clearly-labeled appendix
  inside `guides/migrating-to-pack-vnext.md` (or kill if redundant
  with that guide).

**Issue 3.** *Refresh stale version strings and remove "contributors
first" framing.*
- Update Tutorial 01 sample output (and any others) to v1.1.0.
- Update `docs/index.mdx` opening framing to "user-first".
- Decide on version-substitution mechanism for future releases
  (build-time replace vs. linting rule).

### Milestone 2 — Migration realism (P1)

**Issue 4.** *Embed or link real example assets from
`coming-from-gastown.md`.*
- Replace dangling `examples/gastown/city.toml` references with
  inline TOML blocks **or** downloadable links served by Mintlify.
- Ensure offline / PDF rendering works.

**Issue 5.** *Add a "Gas Town role recap" section to
`coming-from-gastown.md`.*
- One paragraph per Gas Town role (mayor, deacon, witness,
  refinery, polecat, crew) explaining what it *did*, before the
  mapping table.
- Goal: usable as a standalone migration evaluation doc, not a
  cheat sheet for people who already know.

**Issue 6.** *Publish a "Complete Minimal City" example.*
- New page under `docs/guides/` (or a top-level
  `docs/examples/minimal-city.md`).
- Full tree: `pack.toml`, `city.toml`, `agents/<name>/`, one
  formula, one order — assembled, runnable end-to-end.
- Link from index, quickstart, and Tutorial 07.

### Milestone 3 — Reference & workflow gaps (P1)

**Issue 7.** *Add "Common Config Patterns" to `reference/config.md`.*
- Hand-written grouped patterns above the auto-generated field
  dump: changing beads provider, adding a rig, configuring pools,
  overriding an agent's provider, etc.

**Issue 8.** *Expand `reference/formula.md` with conditions, loops,
checks.*
- Working examples of each. Source material:
  `engdocs/architecture/formulas.md`,
  `engdocs/design/formula-v2-transient-retries.md`.

**Issue 9.** *Add narrative API guide to `reference/api.md`.*
- "How to query your city's state" walkthrough with curl + a tiny
  client snippet. Keep the OpenAPI link.

**Issue 10.** *Add a "Debugging Sessions" guide.*
- Cover `gc session peek` output, stuck-session detection, restart
  flow.
- Source: `engdocs/architecture/session.md`,
  `engdocs/contributors/reconciler-debugging.md` (sanitized for
  users — the trace artifact workflow stays contributor-only).

**Issue 11.** *Add a "Choosing crew vs. polecats" guide.*
- Decision criteria and config snippets.
- Source: `engdocs/architecture/session.md`,
  `engdocs/design/agent-pools.md`.

**Issue 12.** *Add a "Writing your first agent prompt" guide.*
- Template variables, GUPP principle in user terms, examples.
- Source: `engdocs/architecture/prompt-templates.md`.

### Milestone 4 — IA cleanup (P1/P2)

**Issue 13.** *Audit and reclassify `docs/packv2/`.*
- For each of the 8 unlinked files: promote into Guides/Reference
  with proper nav entry, or move to `engdocs/`.
- Update `docs/docs.json`.

**Issue 14.** *Rename `docs/internals/` → "How It Works" (or
similar), and promote `beads-topology.md` into the main IA.*
- Adjust framing from "not required" to operator-relevant.

**Issue 15.** *Move OMZ alias warning to top of Quickstart and
expand troubleshooting catalog.*
- New runbooks (initial set): recovering from dolt corruption,
  resetting a rig, debugging order triggers.

### Milestone 5 — Scaling and integrations (P2)

**Issue 16.** *Add a multi-machine / Kubernetes deployment guide.*
- Source: `engdocs/design/machine-wide-supervisor-v0.md`.

**Issue 17.** *Document hooks and external integrations.*
- Source: `engdocs/architecture/messaging.md`,
  `engdocs/design/external-messaging-fabric.md`.

---

## 5. Process

1. Land this design doc as `Proposed`.
2. Iterate on §4 (issue list) in this file until the work plan is
   agreed.
3. Cut the issues from §4. Each issue links back here for context.
4. Flip status to `Accepted` once issues are filed.
5. Flip individual items in §4 to checkboxes as their issues land.
6. Flip whole doc to `Implemented` when Milestones 1–3 are done.
   Milestones 4–5 may continue independently.

---

## 6. Open questions

- Should the "Mental Model" page live under a new `concepts/` section
  in the nav, or as `getting-started/mental-model.md` so it's
  encountered earlier?
- Do we want Mintlify "snippets" / includes for shared TOML
  examples, so the minimal-city tree and tutorial examples don't
  drift?
- Is there a version-substitution mechanism already used in
  `make check-docs`, or do we need to add one?
- For Gas Town migration: do we keep `coming-from-gastown.md` as a
  single long page, or split into "Concept Map" + "Command Cheat
  Sheet" + "Fast Ramp Checklist"?
