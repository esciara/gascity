---
title: "#458 — live deployment instructions for items 1, 2-cadence, 3"
status: working notes
target: laptop / dev machine, not the agent loop
references:
  - https://github.com/gastownhall/gascity/issues/458
  - engdocs/archive/analysis/issue-458-followup-investigation.md
---

Three of the punch-list items from the second comment on
`gastownhall/gascity#458` need a running gascity deployment with a
patrol-style agent active. None can be run from inside an agent
session. Instructions below assume a developer laptop (macOS or Linux).

## What we want to measure

| Item | Number we want | Why |
|---|---|---|
| (1) `/context` baseline | Per-turn overhead in tokens on current Claude Code | Validate or refute the ~94–104K figure in #458 |
| (2) Cadence | Wakeup interval (s) and no-op-cycle ratio (%) | See if recent fixes (suspension / wait-held / pending_create_claim) tamed the storm |
| (3) `exec:` win | API spend per hour: LLM-backed patrol vs `exec:`-backed patrol | Decide whether `exec:` alone closes the cost gap, making event-filtering a non-priority |

## Prerequisites

```bash
# macOS
brew install tmux git jq dolt flock
brew install gastownhall/gascity/bd     # or download from github releases

# Linux (Debian/Ubuntu)
sudo apt install tmux git jq procps lsof util-linux
# dolt + bd: download release tarballs from
# https://github.com/dolthub/dolt/releases and
# https://github.com/gastownhall/beads/releases

# Claude Code (one of)
brew install --cask claude        # macOS
# or follow https://docs.claude.com/en/docs/claude-code/quickstart
```

You also need an Anthropic API key for the witness/patrol session(s):

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

## Build gas city from a known commit

Either upstream or fork — **but record which.** The four SHAs cited in
#458 (`513a1d12`, `9af96b94`, `73f3e1da`, `1543a77f`) are all valid
merged commits on upstream `gastownhall/gascity`, but none of them
resolve in `esciara/gascity` under those identifiers. The fork
appears to carry rebased equivalents (e.g. `cf64aca`,
`fix(session): preserve in_progress claims across worker churn`,
2026-04-28). Build from a deployment whose commit list you can match
against the four upstream SHAs.

```bash
# Option A — upstream (matches the SHAs cited in #458 directly)
git clone https://github.com/gastownhall/gascity.git
cd gascity
git log --oneline | head -5  # record this in your measurement notes

# Option B — fork (for parity with this analysis branch)
git clone https://github.com/esciara/gascity.git
cd gascity
git checkout claude/review-claude-context-tokens-9paxM
# Add upstream so you can compare
git remote add upstream https://github.com/gastownhall/gascity.git
git fetch upstream
git cherry -v upstream/main HEAD | head -40  # what's missing/diverged
git log --oneline | head -5  # record this

# Build
make install      # installs `gc` to $GOBIN (defaults to ~/go/bin)
gc version
```

Confirm the fixes that matter are in:

```bash
# On upstream, direct lookup works:
git show 513a1d12 9af96b94 73f3e1da 1543a77f --stat --format="=== %h %s ==="

# On the fork, search by theme since the SHAs differ:
git log --all --oneline --grep="pending_create_claim" --grep="wait-held" \
  --grep="named session" --grep="suspension" -i | head
```

If you're on the fork and one of the four upstream SHAs has no
equivalent locally, note which thematically-matching commits *are*
present (or absent) and call that out in the measurement
report. Re-measurement on a deployment without those fixes would not
refute (or confirm) #458's cost numbers.

## Bring up a city with a patrol-style agent

The witness role is a *configuration*, not a built-in. The simplest
fast path is to crib from an existing example pack:

```bash
gc init ~/measure-458
cd ~/measure-458

# Inspect the example city configs to pick one that has a patrol-style
# always-on agent (or the closest available)
ls /path/to/gascity/examples/
cat /path/to/gascity/examples/gastown/city.toml | head -80

# Copy one as a starting template
cp /path/to/gascity/examples/gastown/city.toml ./city.toml

# Add a rig
mkdir hello-world && cd hello-world && git init && cd ..
gc rig add ./hello-world
```

Edit `city.toml` so exactly one always-on agent is present (the witness
analog), backed by the Claude (Anthropic) provider — i.e. the LLM-backed
shape that #458 was measuring. Mark it `mode = "always"` (or whatever
the patrol-style equivalent in the example is).

Start the city:

```bash
gc start
```

## Item (1) — `/context` per-turn baseline

In a separate terminal, attach to the patrol session:

```bash
gc session attach <patrol-session-name>
```

This drops you into Claude Code running inside the agent's tmux pane.
Once the prompt is ready, type:

```
/context
```

Record the readout:

- **Total used** (tokens)
- **Per-category breakdown** (system prompt, memory files, skills, MCP
  tool names, conversation messages)

Then send a single short user message (e.g. `noop`) and re-run
`/context`. The delta is the marginal cost of one turn.

**What to compare against #458:** the issue's per-turn-overhead total
of ~94–104K. With ToolSearch deferral (default since v2.1.69) we expect
something like ~8–15K cold-start, plus whatever the witness prompt and
CLAUDE.md add.

## Item (2) cadence — wakeup interval and no-op ratio

Let the patrol session run for a measured window (the original
investigation used "a single long-lived session", let's say at least
30 minutes; an hour is better):

```bash
# Time-stamp the start
date -u +"%Y-%m-%dT%H:%M:%SZ" > /tmp/run-start.txt

# Tail the session log to record turn boundaries.
# Adjust path: gc session log <name> writes to .gc/sessions/<name>/...
gc session log <patrol-session-name> | tee /tmp/patrol.log

# Let it run; in a third terminal, watch the event firehose
gc events --follow | tee /tmp/events.log
```

After your measurement window, stop and analyse:

```bash
# Count turns (one stimulus + response per "Human:" prefix in most logs)
grep -c '^Human:' /tmp/patrol.log

# Mean interval = window-seconds / turns
# No-op ratio: count "all clear" / "nothing actionable" lines vs total turns
grep -ciE 'no.*action|all clear|nothing.*actionable' /tmp/patrol.log

# Event volume — bead.updated rate per minute over the window
jq -r 'select(.type=="bead.updated") | .ts' /tmp/events.log | wc -l
```

Compare against #458's numbers: ~14.5s mean interval, 77% no-op,
~45 `bead.updated`/min.

## Item (3) — `exec:` vs LLM cost comparison

Run the same workload **twice**, once with the patrol agent's
`provider = "anthropic"` (or whatever LLM the deployment uses) and
once with `provider = "exec:./gc-witness-patrol.sh"`, where the script
is adapted from the appendix of #458 to the gascity exec contract
(see `internal/runtime/exec/exec_test.go:1094-1173` for the mock
provider script template).

For each run, record over the same window length:

- API spend (from your Anthropic dashboard, or from `gc events
  --follow --type worker.operation` if the metric is captured locally)
- Number of turns / script invocations
- Wall-clock duration

The LLM-backed cost should equal whatever you measure in $/hour. The
`exec:` cost should equal $0/hour (script invocations don't hit any
API). The "win" is the difference.

If the `exec:` patrol detects the same set of actionable conditions as
the LLM patrol over the same window — which is the assumption behind
the recommendation in #458 — then `exec:` *fully* solves the cost
problem and event filtering becomes a nice-to-have rather than a
P0.

## Reporting back

Summarise into a short table and post on `#458` (or send the numbers
back to me and I'll fold them into the third comment draft):

| Metric | #458 (2026-04-08) | Your run (yyyy-mm-dd) | Notes |
|---|---|---|---|
| Per-turn overhead | ~94–104K tokens | _ | from `/context` |
| Mean wakeup interval | 14.5s | _ | |
| No-op cycle ratio | 77% | _ | |
| `bead.updated` rate | ~45/min | _ | |
| LLM cost / hour | _ | _ | reported by Anthropic dashboard |
| `exec:` cost / hour | $0 | _ | structurally |
| gascity commit | (upstream SHAs cited) | _ | record yours |
| Claude Code version | (pre-deferral) | _ | `claude --version` |

If the per-turn overhead has dropped roughly 10× and `exec:` cleanly
covers the workload, the design-doc path for server-side event
filtering can be deferred indefinitely.
