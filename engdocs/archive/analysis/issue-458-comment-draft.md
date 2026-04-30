---
title: "Draft comment for gastownhall/gascity#458 — context-token re-analysis"
status: draft
target: https://github.com/gastownhall/gascity/issues/458
---

This file is a draft of a comment intended to be posted on
`gastownhall/gascity#458`. It is not a design document. Once posted, this
file can be deleted.

---

## Draft comment

> Revisiting this with what's shipped upstream in Claude Code between the
> filing date (2026-04-08) and now (2026-04-30). The cost diagnosis still
> holds, but the per-turn math in the body is largely obsolete.

### What changed in Claude Code

**Tool definitions are now deferred by default.** Since Claude Code v2.1.69
all built-in tools (Bash, Read, Edit, Write, Glob, Grep, Agent, …) and MCP
tools are routed through `ToolSearch`. Their full schemas are no longer in
the system-prompt prefix — only the `ToolSearch` tool itself plus a short
list of tool *names* (~120 tokens for MCP, ~1K for built-ins) sits in
context. Full schemas load on demand as `tool_reference` blocks; the
prefix is untouched, so prompt caching is preserved.

Reported impact in the wild:

- "System tools" context: **~14–16K → ~968 tokens**
- Multi-server MCP setups (5+ servers): **~55K → ~85% reduction**, only the
  3–5 tools the model actually needs are expanded per turn
- Configured by `ENABLE_TOOL_SEARCH=true|auto|false` (default `true`)

Refs:
[anthropics/claude-code#31002](https://github.com/anthropics/claude-code/issues/31002),
[Tool Search docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool),
[Finisky writeup](https://finisky.github.io/en/claude-code-deferred-tools/).

**Per-subagent tool scoping already exists.** The recommendation
*"Support per-agent tool scoping (allowlist approach)"* is asking for a
feature that's been shipped: subagent markdown files take a `tools:`
frontmatter allowlist and a `disallowedTools` denylist. Denylist resolves
first, then the allowlist resolves against the remaining pool.
Ref: [Claude Code subagents docs](https://code.claude.com/docs/en/sub-agents).

### Claim-by-claim status

| Claim in the issue | Status today | Notes |
|---|---|---|
| System prompt ~15K | partly stale | ~4.2K for the prompt itself; ~7–10K once you add memory + env + CLAUDE.md |
| Native tool defs 40–50K, "dominant 40–50% of context" | **obsolete** | Deferred behind ToolSearch since v2.1.69; ~1K of names now |
| MCP/external tools 5–15K | **obsolete by default** | Deferred too; ~120 tokens of names until used |
| CLAUDE.md + fragments 10–15K | workload-specific | Still real, but typical projects report ~2–3K |
| **Per-turn overhead 94–104K** | **wrong now** | Realistic baseline ~8–15K with ToolSearch on |
| "No per-agent tool scoping" / recommend allowlist | **already shipped** | `tools:` allowlist + `disallowedTools` in subagent frontmatter |
| ~10K turns, 14.5s cadence, 77% no-op, ~45 ev/min | still valid | Gas City workload property, unaffected by upstream |
| Suspension bug (root cause #1) | **fixed** (commit 513a1d12) | |
| `exec:` for deterministic patrol roles | still sound | Independently good advice |

### Suggested next step

Could whoever has a witness running today paste `/context` output for a
current baseline? My read is the per-turn overhead is now an order of
magnitude smaller than 94K, which materially changes the cost model and
the priority of the architectural fixes.

### What's still actionable on the Gas City side

The two recommendations that don't depend on Claude Code evolution remain
the durable wins:

1. **Event filtering by rig and actionable types** — `gc events --watch
   --rig=$GC_RIG --types=...`. Independent of any upstream change.
2. **`exec:` script provider for deterministic roles** — replaces an LLM
   turn whose only judgment is "if work then work" with zero-cost CLI
   logic. Aligns with the ZFC principle in `AGENTS.md`: if a turn's
   decision is judgment-free transport, it doesn't need a model.

The "per-agent tool scoping" recommendation collapses into plumbing the
upstream `tools:` frontmatter through pack/agent config, rather than
designing a new mechanism.

I can open a follow-up comment with a concrete proposal for #1 and #2 once
I've checked the current state of both in the codebase.
