---
title: MEOW and Roles
description: Why Gas City has no built-in roles — and how the MEOW stack lets you build any role from configuration.
---

If you came from Gas Town, the first surprise is that Gas City has no
mayor, deacon, witness, or polecat anywhere in its code. There is no
`Role` type to extend and no role name in any Go file. Yet you can
still build every one of those roles. This page explains the idea that
makes that possible: the **MEOW stack**.

## What MEOW is

MEOW — the **Molecular Expression of Work** — is the work substrate
underneath the whole Gas universe. The upstream definition:

> Breaking large goals into detailed instructions for agents.
> Supported by Beads, Epics, Formulas, and Molecules. MEOW ensures
> work is decomposed into trackable, atomic units that agents can
> execute autonomously.

In Gas City terms, the working stack is:

- **Beads** — every unit of work is a bead, stored durably. A task is
  a bead; so is a message, a molecule, a convoy.
- **Formulas** — reusable workflow templates written in TOML. A
  formula is a recipe: an ordered list of steps with dependencies and
  variables.
- **Molecules** — a formula instantiated for a real piece of work. The
  steps become beads the agent checks off one by one. (A throwaway
  molecule created for a single dispatch is called a **wisp**.)

The key property: **work outlives the worker.** Because a molecule is
just beads, an agent can crash mid-task and a fresh session picks up
at the exact next step. Sessions are disposable; the work is not.

## Why that means roles can be configuration

Think about what a Gas Town role like the *polecat* actually is. Strip
it down and you find five things:

1. an **identity** and lifecycle (name, scope, how long it lives),
2. the **work** it does,
3. the **procedure** it follows for that work,
4. its **judgment** when things go sideways,
5. what **wakes** it up.

In Gas Town, all five were baked into a Go type. MEOW lets four of
them become data:

| Part of the role | Becomes | Where you write it |
|---|---|---|
| Identity & lifecycle | config | `agents/<name>/agent.toml` |
| The work it does | **beads** | bead routing (labels, pools) |
| The procedure | **a formula → molecule** | `*.formula.toml` |
| What wakes it | config + events | `orders/<name>.toml` |
| Its judgment | a **prompt** | `prompt.template.md` |

The procedure — the part that used to be the most role-specific
*code* — turns into the most ordinary MEOW data: a formula is just a
checklist of step beads. So the SDK never needs `if role ==
"polecat"`. The steps aren't in the code; they're in a molecule.

What's left over — judgment — is exactly the part you *want* outside
the code, written in plain language in a prompt, because it keeps
getting better as the models improve.

## The result: every agent runs the same loop

Once work is beads and procedure is molecules, every agent runs one
generic loop: **if there's work on your hook, run it.** The SDK only
has to route work to a session, turn a formula into a molecule, run
the session, and fire scheduled work when a trigger opens — none of
which needs to know a role's name.

So a "role" in Gas City is not a type. It's a small bundle of config:

```text
agents/polecat/
├── agent.toml          # identity + lifecycle
└── prompt.template.md  # judgment

formulas/mol-polecat-work.toml   # the procedure
orders/...                       # what wakes it (optional)
```

Add a role by writing those files. Change a role by editing them.
Delete the whole pack and the SDK is byte-for-byte unchanged — that's
the test that proves no role leaked into the code.

## See also

- [Coming from Gas Town](/getting-started/coming-from-gastown) —
  translating Town's role tree into Gas City config
- [Formulas tutorial](/tutorials/05-formulas) and
  [Beads tutorial](/tutorials/06-beads) — the hands-on mechanics
- For the full architecture argument and citations, see
  `engdocs/architecture/meow-and-role-abstraction.md` in the
  repository.
