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

**Tool definitions are now deferred by default.** Since Claude Code v2.1.69 all built-in tools (Bash, Read, Edit, Write, Glob, Grep, Agent, …) and MCP tools are routed through `ToolSearch`. Their full schemas are no longer in the system-prompt prefix — only the `ToolSearch` tool itself plus a short list of tool *names* (~120 tokens for MCP, ~1K for built-ins) sits in context. Full schemas load on demand as `tool_reference` blocks; the prefix is untouched, so prompt caching is preserved.

Reported impact in the wild:

- "System tools" context: **~14–16K → ~968 tokens**
- Multi-server MCP setups (5+ servers): **~55K → ~85% reduction**, only the 3–5 tools the model actually needs are expanded per turn
- Configured by `ENABLE_TOOL_SEARCH=true|auto|false` (default `true`)

Refs:

* [anthropics/claude-code#31002](https://github.com/anthropics/claude-code/issues/31002),
* [Tool Search docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool),
* [Finisky writeup](https://finisky.github.io/en/claude-code-deferred-tools/).

**Per-subagent tool scoping already exists.** 

The recommendation *"Support per-agent tool scoping (allowlist approach)"* is asking for a feature that's been shipped: subagent markdown files take a `tools:` frontmatter allowlist and a `disallowedTools` denylist. Denylist resolves first, then the allowlist resolves against the remaining pool.

Ref: [Claude Code subagents docs](https://code.claude.com/docs/en/sub-agents).

### Claim-by-claim status

| Claim in the issue                                    | Status today                | Notes                                                                      |
|-------------------------------------------------------|-----------------------------|----------------------------------------------------------------------------|
| System prompt ~15K                                    | partly stale                | ~4.2K for the prompt itself; ~7–10K once you add memory + env + CLAUDE.md  |
| Native tool defs 40–50K, "dominant 40–50% of context" | **obsolete**                | Deferred behind ToolSearch since v2.1.69; ~1K of names now                 |
| MCP/external tools 5–15K                              | **obsolete by default**     | Deferred too; ~120 tokens of names until used                              |
| CLAUDE.md + fragments 10–15K                          | workload-specific           | Still real, but typical projects report ~2–3K                              |
| **Per-turn overhead 94–104K**                         | **wrong now**               | Realistic baseline ~8–15K with ToolSearch on                               |
| "No per-agent tool scoping" / recommend allowlist     | **already shipped**         | `tools:` allowlist + `disallowedTools` in subagent frontmatter             |
| ~10K turns, 14.5s cadence, 77% no-op, ~45 ev/min      | still valid                 | Gas City workload property, unaffected by upstream                         |
| Suspension bug (root cause #1)                        | **fixed** (commit 513a1d12) |                                                                            |
| `exec:` for deterministic patrol roles                | still sound                 | Independently good advice                                                  |

### Suggested next step

Could whoever has a witness running today paste `/context` output for a current baseline? My read is the per-turn overhead is now an order of magnitude smaller than 94K, which materially changes the cost model and the priority of the architectural fixes.

### What's still actionable on the Gas City side

The two recommendations that don't depend on Claude Code evolution remain the durable wins:

1. **Event filtering by rig and actionable types** — `gc events --watch --rig=$GC_RIG --types=...`. Independent of any upstream change.
2. **`exec:` script provider for deterministic roles** — replaces an LLM turn whose only judgment is "if work then work" with zero-cost CLI logic. Aligns with the ZFC principle in `AGENTS.md`: if a turn's decision is judgment-free transport, it doesn't need a model.

The "per-agent tool scoping" recommendation collapses into plumbing the upstream `tools:` frontmatter through pack/agent config, rather than designing a new mechanism.

I can open a follow-up comment with a concrete proposal for #1 and #2 once I've checked the current state of both in the codebase.



# Technical Investigation: Witness Agent Cost Drivers in Gastown/Gascity

## Executive Summary

Gastown witness agents — designed as per-rig work-health monitors — are the
dominant cost driver in gascity deployments. In one observed deployment running
the gastown pack, witnesses accounted for **~69% of total city API spend**.
A single witness session (monorepo rig) accumulated **~10,000 API turns** in
a single long-lived session, making a round-trip call every **14.5 seconds**
on average.

This investigation identifies four compounding root causes, traces them through
the gascity controller and gastown formula code, and proposes architectural
fixes.

---

## 1. Architecture: How Witnesses Run

### Role

The witness is a per-rig singleton defined in `gastown/pack.toml`:

```toml
[[agent]]
name = "witness"
scope = "rig"
idle_timeout = "1h"
prompt_template = "prompts/witness.md.tmpl"
```

One witness is spawned per active rig. With N rigs, the city runs N witness
LLM sessions simultaneously.

### Patrol Loop Mechanism

The witness runs a continuous patrol via `mol-witness-patrol` (formula version 7).
Each iteration is a **wisp** (ephemeral molecule):

```
Pour wisp → Execute 4 patrol steps → Pour NEXT wisp → Burn THIS wisp → repeat
```

The 4 patrol steps per cycle:

| Step | Purpose | LLM Required? |
|------|---------|---------------|
| `check-inbox` | Triage mail (HELP, blocked, HANDOFF) | Minimal — structured message types |
| `recover-orphaned-beads` | Find beads assigned to dead agents, salvage work | No — `bd list` + `git` commands |
| `check-refinery` | Monitor merge queue staleness | No — `bd list` + timestamp math |
| `check-polecat-health` | Detect alive-but-stuck workers | No — session list + bead timestamps |

### Wake Triggers

Three mechanisms can wake or keep a witness active:

1. **Controller reconciliation tick** — runs every `patrol_interval` (default 30s in city.toml). Each tick calls `ComputeAwakeSet()` which decides if the witness should be running. If the witness has assigned work (a patrol wisp), it stays in the desired set.

2. **Event-driven wake within patrol** — the formula's `next-iteration` step uses `gc events --watch --type=bead.updated --timeout=Ns` to reactively patrol. When a bead state change occurs anywhere in the system, the witness wakes.

3. **Nudge delivery** — other agents (deacon, mayor) can nudge the witness, which triggers immediate activity via direct session injection or a FIFO-based queue.

---

## 2. Root Cause Analysis

### 2.1 Bug: Named Sessions Bypass Rig Suspension (FIXED)

**Severity: Critical — accounted for majority of observed cost in deployments with suspended rigs**

**Status: Fixed in `513a1d12` (2026-04-02), merged to main.**

When a rig was suspended (`suspended = true` in city.toml), scaled pool agents
(polecats) correctly stopped spawning. But **named sessions with `mode = "always"`
skipped the suspension check entirely**.

The bug was in two locations:

- `compute_awake_bridge.go`: Only propagated agent-level suspension, ignoring
  rig and city suspension hierarchy
- `compute_awake_set.go`: Named-session loop had no suspension gate, while
  scaled agents were properly gated

The fix introduced `isAgentEffectivelySuspended()` to capture the full suspension
hierarchy, and added suspension checks to both the named-session and
assigned-work-fallback paths. Test coverage was added for suspended named sessions.

**Impact before fix:** In a deployment where most rigs are suspended, witnesses
ran on every rig. The controller kept them alive, they patrolled continuously,
and accumulated API costs on rigs with no active work.

**Related recent fixes on main:**

- `9af96b94` — Preserve `pending_create_claim` across named session reopen/materialization
- `73f3e1da` — Suppress demand-driven wake reasons for wait-held sessions
- `1543a77f` — Test coverage for fresh-wake startup context

### 2.2 Continuous Patrol Loop: No Meaningful Idle Period

The witness runs in **one long-lived session** making thousands of API calls:

| Agent | API Turns | Avg Interval |
|-------|----------:|-------------:|
| Rig A witness | ~10,000 | 14.5s |
| Rig B witness | ~4,650 | 31s |
| Rig C witness | ~3,280 | 44s |
| Boot agent | ~2,500 | 58s |

Sample session log showing the patrol cadence:

```
23:20:50 [ASSISTANT] Resuming patrol after restart. Checking for assigned work.
23:20:56 [ASSISTANT] Creating fresh patrol wisp.
23:21:04 [ASSISTANT] Wisp claimed. Executing patrol.
23:21:12 [ASSISTANT] All clear: no mail, no orphaned beads, empty refinery queue.
23:21:33 [ASSISTANT] Cycle 1 complete. Starting cycle 2.
23:21:42 [ASSISTANT] All clear. Continuing patrol.
23:21:59 [ASSISTANT] Cycle 2 complete. Starting cycle 3.
23:22:04 [ASSISTANT] All clear. Continuing patrol.
23:22:17 [ASSISTANT] Cycle 3 complete. Starting cycle 4.
```

**~30 seconds per cycle**, 5-7 tool calls per cycle, and the answer is "all clear"
on 77% of cycles. Each cycle involves:

1. LLM reads formula step description (~15K tokens of formula text)
2. LLM calls `bd list`, `gc mail inbox`, `gc session list`
3. LLM evaluates results: "nothing actionable"
4. LLM pours next wisp, burns current wisp
5. Repeat

### 2.3 Event Storm Prevents Exponential Backoff

The formula configures `event_timeout = 30s` (default) with exponential backoff:
timeout doubles on each idle cycle up to 300s max. This should reduce API
calls when nothing is happening.

**But it doesn't work.** The event stream produces **~45 `bead.updated` events
per minute** from normal system operation (order tracking, label changes, bead
lifecycle). The witness watches `--type=bead.updated`, so there is ALWAYS an
event to process:

```json
{"event_type":"label_added","issue_id":"gc-xxx","created_at":"2026-04-03T16:08:05Z"}
{"event_type":"closed","issue_id":"gc-yyy","created_at":"2026-04-03T16:08:05Z"}
{"event_type":"created","issue_id":"gc-zzz","created_at":"2026-04-03T16:08:07Z"}
{"event_type":"label_added","issue_id":"gc-aaa","created_at":"2026-04-03T16:08:09Z"}
```

4 events in 4 seconds, and this is during a relatively quiet period. During
active work periods with multiple polecats, event rates are much higher.

The backoff resets every time an event arrives. With 45+ events/minute and a
30s timeout, the timeout **never expires** — the witness wakes on every cycle.

### 2.4 Massive Per-Turn Context Overhead

The witness prompt itself is small (~2,100 tokens). But each API turn carries
~94K tokens of overhead that the witness can't control:

| Component | Tokens (est.) |
|-----------|-------------:|
| Claude Code system prompt | ~15,000 |
| Claude Code native tool definitions (bash, read, write, edit, glob, grep, agent, worktree, task, etc.) | ~40,000-50,000 |
| External tool definitions (MCP servers, if configured) | ~5,000-15,000 |
| CLAUDE.md + global fragments | ~10,000-15,000 |
| Witness prompt template | ~2,100 |
| Conversation history | ~5,000-20,000 |
| **Total per turn** | **~94,000-104,000** |

The **Claude Code native tool definitions** are the dominant component — roughly
40-50% of total context. These are the built-in tool schemas (Bash, Read, Write,
Edit, Glob, Grep, Agent, NotebookEdit, WebFetch, WebSearch, TaskCreate, etc.)
that ship with Claude Code itself. External MCP tools add to this but are a
smaller share, and can be tuned per-deployment.

Witnesses need very few of these tools — they primarily use Bash (to run `bd`
and `gc` commands). But Claude Code currently loads all tool definitions into
every session regardless of role.

**First-turn cache write:** 94,242 tokens (establishes the prompt cache).
Subsequent turns read from cache at reduced cost, but the cache must be
re-established on every session restart (hourly due to idle_timeout).

---

## 3. Compounding Effects

These four issues compound multiplicatively:

```
Cost = (N rigs, including suspended)
     x (turns per hour, driven by event storm)
     x (tokens per turn, driven by tool overhead)
     x (cost per token, driven by model selection)
```

In the observed deployment:

- 4+ witness sessions running (including on suspended rigs)
- ~120-240 turns/hour per witness (event storm prevents backoff)
- ~100K tokens per turn (tool definition overhead)
- Opus-tier pricing

### The "All Clear" Tax

77% of patrol cycles find nothing actionable. Each "all clear" cycle still
costs a full API round-trip with ~100K tokens of context. This is the core
inefficiency: **the LLM's contribution on most cycles is evaluating "nothing
changed" — a determination that a shell script could make for $0.**

---

## 4. Trigger Comparison: Controller vs Formula vs Events

Understanding where costs originate requires separating three trigger layers:

### Controller Layer (gascity)

| Trigger | Frequency | Witness Impact |
|---------|-----------|----------------|
| Reconcile tick | Every `patrol_interval` (30s) | Checks if witness should be awake; keeps it in desired set if wisp assigned |
| Poke (work assigned) | On `gc sling` dispatch | Immediate tick; can wake sleeping witness |
| Idle timeout check | Per tick | After 1h idle, drains and recreates session |
| Crash recovery | On session death | New wisp poured, session recreated |

The controller itself doesn't directly cause API calls — it manages session
lifecycle. But it ensures the witness **stays awake** as long as a patrol wisp
is assigned, which is always (the formula pours the next wisp before burning
the current one).

### Formula Layer (gastown)

| Trigger | Frequency | API Calls |
|---------|-----------|-----------|
| Patrol cycle (4 steps) | Every 30-300s | 5-7 tool calls per cycle |
| Wisp pour/burn | Each cycle end | 2-3 tool calls |
| Formula re-read | Each cycle start | ~15K tokens of formula text processed |
| Context check | Each cycle start | 1-2 tool calls |

The formula drives the continuous patrol loop. It is the primary generator
of API turns.

### Event Layer (gascity)

| Trigger | Frequency | Effect |
|---------|-----------|--------|
| `bead.updated` | ~45/minute | Resets backoff timer, triggers immediate patrol |
| `session.draining` | Occasional | May trigger orphan recovery |
| `convoy.closed` | Rare | Not watched by witness |

The event layer prevents the backoff from engaging, turning the formula's
30-300s adaptive range into a constant ~30s cycle.

### Overlap Analysis

The three layers create a feedback loop:

1. **Controller keeps witness awake** (wisp always assigned)
2. **Formula runs patrol loop continuously** (pours next wisp before burning)
3. **Events prevent backoff** (45 events/min resets timeout)
4. **Each patrol turn finds nothing** (but still costs ~100K tokens)
5. **Session stays alive until idle_timeout** (1h, but never actually idle)
6. **On timeout/restart, full context re-established** (94K cache write)

The witness effectively runs at maximum frequency at all times, regardless
of whether there's actual work to monitor.

---

## 5. What the Script Replacement Changes

A shell script replacement (`gc-witness-patrol.sh`) has been developed that
implements the same 4 patrol steps using direct CLI commands:

```bash
# check-inbox: gc mail inbox → parse structured types → route
# recover-orphans: bd list --assignee=<dead> → git push → bd update
# check-refinery: bd list --assignee=refinery → age check → alert
# check-polecats: gc session list + bd list → staleness → warrant
```

**Key behavioral differences:**

| Aspect | LLM Witness | Script Witness |
|--------|-------------|----------------|
| API cost per cycle | ~100K tokens | $0 |
| Decision quality (routine) | Identical | Identical |
| Decision quality (edge cases) | LLM judgment | Escalates to deputy/mayor |
| Wake mechanism | `gc events --watch` | `gc events --watch` (same) |
| Backoff | Defeated by event storm | Adaptive (checks actionable events only) |
| Nudge handling | Inline in session | FIFO pipe to patrol loop |

The script uses gascity's `exec:` provider interface — a first-class provider
type that the reconciler treats identically to Claude sessions. The controller
manages its lifecycle (start/stop/health) through the provider script, not
through tmux/Claude Code.

---

## 6. Recommendations for Gascity Architecture

### 6.1 ~~Fix the Suspension Bug~~ (DONE)

Fixed in `513a1d12`. Named sessions now respect the full suspension hierarchy.
Test coverage added.

### 6.2 Support Per-Agent Tool Scoping

Claude Code's native tool definitions are the largest context component (~40-50K
tokens). Witnesses and other infrastructure agents need very few tools — primarily
just Bash. If Claude Code or gascity could support a tool allowlist per agent
definition, this would dramatically reduce per-turn context for infrastructure
roles:

```toml
[[agent]]
name = "witness"
tool_filter = ["bash", "read", "grep"]  # only what witness needs
```

**Note:** This is a Claude Code platform constraint, not a gascity bug.
Gascity could potentially work around it by configuring agent-specific
`settings.json` overlays that disable unused tools, but native support
for per-session tool scoping would be the clean solution.

### 6.3 Event Filtering for Patrol Agents

The `gc events --watch` command should support filtering by relevance. Witnesses
only care about events in their own rig, and only about specific event types
(bead assignment changes, session death). A `--rig` filter would prevent cross-rig
event noise from defeating backoff:

```bash
gc events --watch --type=bead.updated --rig=$GC_RIG --timeout=60s
```

### 6.4 First-Class Script Providers in Gastown

The `exec:` provider interface already exists in gascity. Gastown should ship
script providers for deterministic roles (witness, boot) as the default, with
LLM providers as an opt-in for deployments that need LLM judgment in patrol.

This could be a pack-level configuration:

```toml
[[agent]]
name = "witness"
provider = "exec:scripts/gc-witness-provider.sh"  # default: script
# provider = "claude"  # opt-in: LLM
```

### 6.5 Patrol Frequency Governors

The formula's backoff mechanism is defeated by event volume. Consider:

- **Minimum cycle interval**: Enforce a floor (e.g., 60s) regardless of events
- **Actionable-event filtering**: Only wake on events that could require witness
  action (assignment changes, session deaths), not on all `bead.updated` events
- **Cycle budget**: Cap the number of patrol cycles per hour

---

## 7. Data Sources

- Session logs via `gc session logs <id>`
- Controller source: `cmd/gc/` (city_runtime.go, compute_awake_set.go, compute_awake_bridge.go, idle_nudge.go)
- Formula definition: `gastown/formulas/mol-witness-patrol.formula.toml` (v7)
- Agent configuration: `gastown/pack.toml`, `town-ops/pack.toml`
- Event stream: `.beads/events.jsonl`
- Prior investigation: deputy witness-investigation (internal)
- Design document: witness-to-script replacement design (internal)

---

## Appendix A: Script Witness Implementation

The following script implements the same 3-step patrol loop as the LLM witness,
using direct CLI commands instead of LLM reasoning. It runs as a gascity `exec:`
provider — a first-class provider type that the reconciler manages identically
to Claude sessions.

Key design choices:

- **Adaptive backoff with minimum floor** (60s-300s) — unlike the LLM formula,
  backoff is based on whether anything *actionable* was found, not on event arrival
- **FIFO-based nudge handling** — nudges delivered via named pipe, drained
  non-blocking on each cycle
- **Escalation for edge cases** — uncommitted work in orphaned worktrees gets
  mailed to the deputy rather than attempting LLM-style judgment

```bash
#!/usr/bin/env bash
# gc-witness-patrol.sh — Event-driven patrol loop for rig health monitoring.
#
# Replaces the LLM-based witness agent with a shell script that performs
# a 3-step patrol cycle at zero API cost with adaptive backoff.
set -uo pipefail

# ── Configuration ────────────────────────────────────────────────────
RIG_NAME="${GC_RIG:-}"
AGENT_NAME="${GC_AGENT:-${RIG_NAME}/witness}"
WORK_DIR="${GC_WORK_DIR:-.gc/agents/${RIG_NAME}/witness}"
STATE_DIR="${GC_WITNESS_STATE_DIR:-/tmp/gc-witness-${RIG_NAME}}"
LOG_FILE="${WORK_DIR}/patrol.log"
NUDGE_FIFO="${STATE_DIR}/fifo"

STALE_THRESHOLD_MIN=30      # refinery staleness
MIN_INTERVAL=60             # minimum seconds between patrol cycles
MAX_INTERVAL=300            # maximum backoff interval

# ── Helpers ──────────────────────────────────────────────────────────
log() {
    echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) [${RIG_NAME}] $*" >> "$LOG_FILE"
}

epoch_from_iso() {
    local ts="${1%%.*}"
    ts="${ts%%Z}"
    date -j -f "%Y-%m-%dT%H:%M:%S" "$ts" +%s 2>/dev/null || echo 0
}

# ── Step 1: check-inbox ─────────────────────────────────────────────
check_inbox() {
    local messages
    messages=$(gc mail inbox --json 2>/dev/null) || return 0
    [ -z "$messages" ] || [ "$messages" = "[]" ] && return 0

    echo "$messages" | jq -c '.[]' 2>/dev/null | while read -r msg; do
        local msg_id subject from
        msg_id=$(echo "$msg" | jq -r '.id // empty')
        [ -z "$msg_id" ] && continue
        subject=$(echo "$msg" | jq -r '.subject // ""')
        from=$(echo "$msg" | jq -r '.from // "unknown"')

        case "$subject" in
            *HELP*|*help*|*blocked*|*stuck*)
                log "Forwarding help request from $from: $subject"
                gc mail send mayor -s "FWD from ${AGENT_NAME}: $subject" \
                    -m "Witness forwarding help request. Original: $msg_id" \
                    2>/dev/null || true
                ;;
            *) log "Archiving unhandled mail from $from: $subject" ;;
        esac
        gc mail archive "$msg_id" 2>/dev/null || true
    done
}

# ── Step 2: recover-orphaned-beads ───────────────────────────────────
recover_orphans() {
    local beads sessions
    beads=$(bd list --status=in_progress --json 2>/dev/null) || return 0
    [ -z "$beads" ] || [ "$beads" = "[]" ] && return 0
    sessions=$(gc session list --json 2>/dev/null) || return 0

    echo "$beads" | jq -c '.[]' 2>/dev/null | while read -r bead; do
        local bead_id assignee
        bead_id=$(echo "$bead" | jq -r '.id // empty')
        assignee=$(echo "$bead" | jq -r '.assignee // empty')
        [ -z "$assignee" ] || [ -z "$bead_id" ] && continue

        # Is assignee alive?
        local is_alive
        is_alive=$(echo "$sessions" | jq -r \
            --arg a "$assignee" \
            '[.[] | select(.template == $a or .title == $a)] | length')
        [ "${is_alive:-0}" != "0" ] && continue

        # Orphaned — check for salvageable work
        local branch worktree_path
        branch=$(bd show "$bead_id" --json 2>/dev/null | jq -r '.metadata.branch // empty')

        if [ -n "$branch" ] && git ls-remote --heads origin "$branch" 2>/dev/null | grep -q "$branch"; then
            if git merge-base --is-ancestor "origin/$branch" origin/main 2>/dev/null; then
                log "Orphan $bead_id: branch merged. Closing."
                bd close "$bead_id" --reason "Branch merged to main" 2>/dev/null || true
                continue
            fi
        fi

        # Escalate if worktree has uncommitted changes
        worktree_path=$(bd show "$bead_id" --json 2>/dev/null | jq -r '.metadata.worktree // empty')
        if [ -n "$worktree_path" ] && [ -d "$worktree_path" ]; then
            if [ -n "$(git -C "$worktree_path" status --porcelain 2>/dev/null | head -1)" ]; then
                log "Orphan $bead_id: uncommitted changes. ESCALATING."
                gc mail send deputy \
                    -s "Orphan with uncommitted work: $bead_id" \
                    -m "Needs manual salvage." 2>/dev/null || true
                continue
            fi
            git worktree remove "$worktree_path" --force 2>/dev/null || true
        fi

        bd update "$bead_id" --status=open --assignee="" 2>/dev/null || true
        log "Recovered orphan $bead_id (was: $assignee)"
    done
}

# ── Step 3: check-refinery ───────────────────────────────────────────
check_refinery() {
    local refinery_beads now
    refinery_beads=$(bd list --assignee="${RIG_NAME}/refinery" --status=in_progress --json 2>/dev/null) || return 0
    [ -z "$refinery_beads" ] || [ "$refinery_beads" = "[]" ] && return 0
    now=$(date +%s)

    echo "$refinery_beads" | jq -c '.[]' 2>/dev/null | while read -r bead; do
        local bead_id updated_at age_min
        bead_id=$(echo "$bead" | jq -r '.id // empty')
        updated_at=$(echo "$bead" | jq -r '.updated_at // empty')
        [ -z "$updated_at" ] && continue

        age_min=$(( (now - $(epoch_from_iso "$updated_at")) / 60 ))
        if [ "$age_min" -gt "$STALE_THRESHOLD_MIN" ]; then
            log "Refinery bead $bead_id stale (${age_min}m)"
            gc session nudge "${RIG_NAME}/refinery" \
                "Stale bead $bead_id — ${age_min}m without update" 2>/dev/null || true
        fi
    done
}

# ── Main loop ────────────────────────────────────────────────────────
mkdir -p "$(dirname "$LOG_FILE")" "$STATE_DIR" 2>/dev/null || true
[ ! -p "$NUDGE_FIFO" ] && rm -f "$NUDGE_FIFO" && mkfifo "$NUDGE_FIFO" 2>/dev/null

log "Witness patrol starting for rig: $RIG_NAME (PID: $$)"
trap 'log "Witness patrol stopping (PID: $$)"; exit 0' TERM INT

interval=$MIN_INTERVAL

while true; do
    patrol_start=$(date +%s)
    actionable=0
    check_inbox    && actionable=1 || log "WARN: check_inbox failed"
    recover_orphans && actionable=1 || log "WARN: recover_orphans failed"
    check_refinery  && actionable=1 || log "WARN: check_refinery failed"

    # Drain pending nudges (non-blocking)
    if [ -p "$NUDGE_FIFO" ]; then
        exec 3<>"$NUDGE_FIFO"
        while read -t 0.1 nudge_msg <&3 2>/dev/null; do
            log "Nudge received: $nudge_msg"
            actionable=1
        done
        exec 3>&-
    fi

    # Adaptive backoff: grow when idle, reset when busy
    if [ "$actionable" -eq 0 ]; then
        interval=$(( interval * 2 ))
        [ "$interval" -gt "$MAX_INTERVAL" ] && interval=$MAX_INTERVAL
    else
        interval=$MIN_INTERVAL
    fi

    # Enforce minimum interval
    elapsed=$(( $(date +%s) - patrol_start ))
    remaining=$(( interval - elapsed ))
    [ "$remaining" -gt 0 ] && sleep "$remaining"
done
```

---

*Investigation conducted 2026-04-07. Data reflects observed behavior across
multiple rigs over a multi-week period.*