---
title: "MEOW and Role Abstraction"
---

> Last verified against code: 2026-06-01

## Summary

Gas Town proved multi-agent orchestration works, but every role —
mayor, deacon, witness, refinery, polecat — was hardwired in Go. Gas
City removes all of them. The mechanism that makes that possible is
the **MEOW stack** (Molecular Expression of Work).

This document does two things:

1. **Defines MEOW** and maps its canonical components onto Gas City's
   implementation (the definition is cited; see [References](#references)).
2. **Argues the role-abstraction thesis**: *because work is the
   primitive and a role's procedure is itself data, the only
   role-specific thing left is a prompt — which is also data. Go never
   needs to know what a "mayor" is.* That synthesis is Gas City's own;
   it is not stated in the upstream Gas Town material, so it is
   presented here as argument, not citation.

If you only remember one sentence: **MEOW turns the parts of a role
that used to live in Go (its work and its procedure) into beads and
formulas, leaving only judgment in the prompt — so roles become
configuration.**

## What MEOW is

MEOW — the **Molecular Expression of Work** — is the Beads-based
substrate that makes Work the first-class primitive of the Gas
universe. The upstream definition:

> **MEOW (Molecular Expression of Work)**: Breaking large goals into
> detailed instructions for agents. Supported by Beads, Epics,
> Formulas, and Molecules. MEOW ensures work is decomposed into
> trackable, atomic units that agents can execute autonomously.
>
> — [Gas universe glossary](https://github.com/gastownhall/website/blob/main/docs/src/content/docs/glossary.md)

Steve Yegge's framing is that MEOW "places Work front and center, as
the first-class system primitive, creating a versioned knowledge graph
of all your issues and tasks. Work is the currency that drives the Gas
Universe ecosystem. It's Beads all the way down" — persisted in the
git-versioned [Dolt](https://github.com/dolthub/dolt) database (see
[Welcome to Gas City](https://github.com/gastownhall/website/blob/main/docs-fodder/steve-blog-posts/Welcome%20to%20Gas%20City.md)).

The property that matters for role abstraction is **durability of
work independent of the worker**:

> It doesn't matter if Claude Code crashes, or runs out of context. As
> soon as another session starts up for this agent role, it will start
> working on that step immediately.
>
> — [Welcome to Gas Town](https://github.com/gastownhall/website/blob/main/docs-fodder/steve-blog-posts/Welcome%20to%20Gas%20Town.md)

Work survives sessions because it is a bead. Sessions are
interchangeable. That interchangeability is the seed of role
abstraction.

## The MEOW stack in Gas City

The canonical Gas Town stack has more layers than Gas City currently
surfaces. Both are recorded here so the subset is explicit.

| Canonical component (Gas Town) | Definition | Gas City status |
|---|---|---|
| **Beads** | Lightweight work units in a git-versioned store. | First-class. The universal persistence substrate — *everything* is a bead. See [Bead Store](beads.md). |
| **Epics** | Beads with children; hierarchical planning, parallel by default. | Present as an ordinary bead type (not a first-class container). See [Glossary](glossary.md#derived-mechanisms). |
| **Formulas** | TOML workflow templates, "cooked" into molecules. | First-class. Gas City owns discovery + layer resolution; the beads backend owns materialization. See [Formulas & Molecules](formulas.md). |
| **Protomolecules** | Compiled formula templates — "made of actual Beads, with instructions and dependencies set up in advance." | **Not surfaced as a distinct stage.** Gas City resolves a formula and asks the backend to instantiate it directly via `Store.MolCook`; the protomolecule compile step is internal to the `bd` backend, not a Gas City concept. |
| **Molecules** | A formula instantiated at runtime: a root bead plus per-step beads the agent checks off. | First-class. See [Life of a Molecule](life-of-a-molecule.md). |
| **Wisps** | Ephemeral molecules created for dispatch/orders; GC'd after a TTL. | First-class. Created by `gc sling --formula` and order dispatch. |

So Gas City's working stack is **beads → formulas → molecules/wisps**,
with epics and convoys as bead-shaped grouping on top. The
protomolecule compile stage exists in the production `bd` backend but
is not a Gas City primitive — Gas City treats `Store.MolCook(formula,
title, vars)` as the single seam (see
[`formulas.md` § Instantiation](formulas.md)).

## How MEOW abstracts roles into configuration

> The rest of this section is Gas City's synthesis, not an upstream
> citation. It is the argument behind the `AGENTS.md` claim that "the
> MEOW stack was powerful enough to abstract roles into configuration."

### What a role actually is

Decompose any Gas Town role into its parts:

1. **Identity & lifecycle** — its name, scope, working directory,
   idle timeout, how many can run.
2. **The work it does** — the units of work that land on its hook.
3. **The procedure it follows** — the ordered steps for a unit of
   work (create branch → implement → test → submit).
4. **Its judgment** — the decisions it makes when reality doesn't
   match the plan.
5. **What wakes it** — schedules, events, conditions.

In Gas Town, all five were entangled in Go: a `Polecat` type carried
identity, owned its workflow code, and embedded its decision logic.
Adding a role meant adding a type.

MEOW dissolves four of those five into data:

| Part of a role | Where MEOW puts it | Gas City artifact |
|---|---|---|
| Identity & lifecycle | Config | `agents/<name>/agent.toml`, `[[named_session]]` |
| The work it does | **Beads** | bead routing (labels, assignee, pool) |
| The procedure it follows | **Formulas → Molecules** | `*.formula.toml`, instantiated via `MolCook` |
| What wakes it | Config + Event Bus | `orders/<name>.toml` triggers |
| Its judgment | **Prompt Template** | `prompt.template.md` |

The procedure — historically the most role-specific *code* — becomes
the most ordinary MEOW data: a formula is just a list of step beads.
The agent walks the steps by closing beads. There is no `if role ==
"polecat"` because the step list isn't in Go at all; it's a molecule.

### Why this works: every role runs the same loop

Once work is beads and procedure is molecules, every agent — whatever
you call it — runs one generic loop, rendered into its prompt by
[GUPP](glossary.md#design-principles): *"If you find work on your hook,
YOU RUN IT."* The SDK's job shrinks to four role-agnostic operations:

- **route** a bead to a session (Dispatch / Sling),
- **instantiate** a formula as a molecule (`MolCook`),
- **run** a session (Session primitive),
- **fire** an order when a trigger opens (Orders).

None of those four needs a role name. A "mayor" and a "polecat" exit
the same `MolCook` call; they differ only in *which* formula and
*which* prompt — both supplied by config. This is the
[ZFC](glossary.md#design-principles) invariant in action: Go handles
transport, the prompt handles reasoning. The leftover fifth part of a
role — judgment — is the one thing that *should* stay outside Go,
because per the Bitter Lesson it gets better as models improve.

That is the whole abstraction. Roles are not removed; they are
**relocated** — out of Go types and into the MEOW data plane plus a
prompt.

### The invariant it produces

Because nothing role-specific remains in Go, Gas City can enforce a
hard rule: **no role name may appear in Go source.** `if role ==
"mayor"` is a build error, not a style nit. The
[nine-concepts](nine-concepts.md) layering invariant #6 states it from
the other direction — *no SDK mechanism may require a specific
user-configured agent role* — and the test is mechanical: if removing
a `[[agent]]` entry breaks an SDK feature, the abstraction has leaked.

## Worked example: the Gastown pack

The `examples/gastown/` pack rebuilds the entire Gas Town role tree
with zero Go. Each role is the four config artifacts plus a prompt.

**Identity & lifecycle** — `agents/polecat/agent.toml`:

```toml
scope = "rig"
wake_mode = "fresh"
work_dir = ".gc/worktrees/{{.Rig}}/polecats/{{.AgentBase}}"
nudge = "Run gc hook; it checks assigned work first, then routed pool work."
idle_timeout = "2h"
min_active_sessions = 0
max_active_sessions = 5
```

**Procedure** — `formulas/mol-polecat-work.toml` is the polecat's
entire workflow as a formula. Its steps (create feature branch →
implement → test → push → reassign to refinery) are molecule beads the
agent checks off. The "Polecat Contract" lives in the formula
description, not in Go:

```toml
formula = "mol-polecat-work"
# steps: feature-branch setup, implement, run affected tests,
# push + set metadata.branch/target, reassign to refinery, exit.
```

**Judgment** — `agents/mayor/prompt.template.md` carries the Mayor's
coordination behavior ("Dispatch Liberally, Fix When Fast"). It is
pure prompt; the SDK never reads it for control flow.

**Instantiation** — `pack.toml` declares the canonical sessions from
those templates, again as config:

```toml
[[named_session]]
template = "mayor"
scope = "city"

[[named_session]]
template = "polecat"   # rig-scoped, pooled to max 5
scope = "rig"
```

**What wakes it** — `orders/digest-generate.toml` fires a formula on a
schedule, dispatched to a pool, with no role name anywhere:

```toml
[order]
formula = "mol-digest-generate"
trigger = "cooldown"
interval = "24h"
pool = "dog"
```

Nothing in `internal/` or `cmd/gc/` knows that "polecat," "mayor," or
"refinery" exist. Delete the gastown pack and the SDK is unchanged —
which is exactly the test the abstraction is meant to pass.

## References

Upstream definitions of MEOW (cited above):

- [Gas universe glossary — MEOW entry](https://github.com/gastownhall/website/blob/main/docs/src/content/docs/glossary.md)
- [Welcome to Gas City](https://github.com/gastownhall/website/blob/main/docs-fodder/steve-blog-posts/Welcome%20to%20Gas%20City.md)
- [Welcome to Gas Town](https://github.com/gastownhall/website/blob/main/docs-fodder/steve-blog-posts/Welcome%20to%20Gas%20Town.md)
- [Molecular Expression of Work (Medium)](https://medium.com/welcome-to-gas-town-4f25ee16dd04)

Internal architecture:

- [Nine Concepts](nine-concepts.md) — the primitive/derived map and
  layering invariants
- [Bead Store](beads.md), [Formulas & Molecules](formulas.md),
  [Life of a Molecule](life-of-a-molecule.md) — the MEOW data plane
- [Prompt Templates](prompt-templates.md) — where judgment lives
- [Dispatch](dispatch.md), [Orders](orders.md) — the role-agnostic
  routing and trigger mechanisms
- [Glossary](glossary.md) — authoritative term definitions

User-facing companion: [MEOW and Roles](../../docs/internals/meow-and-roles.md).
