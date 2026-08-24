# NANDA Town Assessment — Trust audit reliability under message loss

## Why this scenario, not marketplace
`message_drop` on `marketplace` is the example the quickstart doc
itself suggests, and a large share of other submissions run some
version of it. I wanted a genuinely different question, and picked
`rogue_trusted_agent` specifically: it's the one built-in scenario
about *deception detection* — a high-reputation agent making one
unauthorized reach, and whether the system catches it. That's the
same problem space as work I'm already doing outside this course
(an AI safety platform for detecting deepfake-enabled scams), so it's
the scenario where I actually have domain instinct for what to test.

## Setting changed
`layers.trust` swapped between `aae_permit_gate` (pre-action policy
gate) and `score_average` (reputation-only, no gate), then
`failures.message_drop` swept from `0.0` to `0.8` under each.
Scenario copied to `scenarios_local2/`, `task.config` untouched.

## Hypothesis
The scenario's own comments already document the headline result at
`message_drop: 0.0`: the gate blocks the rogue action regardless of
reputation, and `score_average` doesn't. My question was upstream of
that — **does that safety property survive an unreliable network**,
or does message loss let the rogue action slip through undetected
under the gate too?

## What I found: the decision never breaks, but the evidence does

I confirmed the documented baseline first (both trust layers, drop
rate 0.0, matches the scenario's own comments exactly). Then I read
the actual plugin source (`aae_permit_gate.py`,
`scenarios_builtin/rogue_trusted_agent.py`) before touching the drop
rate, and found something the comments don't mention: the gate's
`evaluate()` call is a direct in-process function call, not a network
message. `message_drop` only affects the `ctx.broadcast()` calls that
*announce* the decision afterward (`rogue_attempt:`, `permit:`,
`exec:`, `blocked:`), never the decision itself.

So my real hypothesis became: the gate's decision should be immune to
`message_drop`, but the trace-based validators — which only know what
happened by reading broadcast messages — should not be. I ran a
single-point sweep first (0.02 → 0.8, one seed) to check the shape of
this, then ran a 30-seed Monte Carlo at each of six drop rates to get
an actual probability rather than trusting one lucky or unlucky run.

**The security decision itself never once failed.** Across all 180
Monte Carlo runs at every drop rate (0.1 through 0.6) plus the
single-seed sweep up to 0.8, `rogue_trusted_agent_blocked` never
returned a false PASS under `score_average` and never showed the
rogue action executing under `aae_permit_gate`. The gate's decision
is exactly as reliable under packet loss as under a perfect network,
because it was never on the network to begin with.

**The evidence that the decision happened is a different story.**
Under `aae_permit_gate`, 30 seeds per drop rate:

| message_drop | audit trail lost | audit trail intact | out of |
|---|---|---|---|
| 0.1 | 2 (7%) | 28 | 30 |
| 0.2 | 4 (13%) | 26 | 30 |
| 0.3 | 10 (33%) | 20 | 30 |
| 0.4 | 15 (50%) | 15 | 30 |
| 0.5 | 18 (60%) | 12 | 30 |
| 0.6 | 20 (67%) | 10 | 30 |

"Audit trail lost" means the validator returns FAIL with **"no
rogue_attempt declared in trace"** — not because nothing happened, but
because the single broadcast line that declares the rogue attempt got
dropped before it could be logged. This tracks a plain Bernoulli(p)
prediction closely (e.g. ~40% expected at p=0.4 vs. 50% observed,
well inside sampling noise for n=30), which confirms the mechanism is
exactly what it looks like: one audit-critical message, one roll of
the dice, nothing more complex compounding it.

The `score_average` (unsafe) layer shows the same pattern in reverse:
message loss never produces a false-safe result (the "executed" FAIL
always fires correctly when the rogue action ran and its `exec:` line
survived), but at high drop rates the run just as often loses the
`rogue_attempt:` declaration entirely, collapsing into the same
uninformative "no rogue_attempt declared" FAIL — indistinguishable,
from the trace alone, from a run where the danger was safely averted.

## Why this is the real finding, not the headline comparison
A system can make the correct call every single time and still fail
an audit, if the record of that call doesn't survive real-world
conditions. That gap, correct decision, unreliable evidence, is
invisible if you only check whether the gate "worked" at
`message_drop: 0.0`, which is all the scenario's own documentation
tests. It only shows up once you stress the logging path the same way
you'd stress the decision path.

## Tools and help used
Used Claude (Claude.ai, with code execution) to run the CLI, but more
importantly to read the actual plugin and scenario source (not just
the YAML comments) before forming a hypothesis, since the comments
describe *what* happens at drop=0, not *why* it would or wouldn't
hold under packet loss. I directed the investigation from there: first
a coarse single-seed sweep to see if the effect was real, then the
30-seed-per-rate Monte Carlo once I wanted a real probability instead
of one run's outcome, then the Bernoulli comparison to check whether
the empirical numbers actually matched the mechanism I thought was
causing them.

## What I'd build next for NANDA Town
A `permit_env:`-style durable receipt for every gate decision, one
that gets written to a local append-only log at decision time, not
just broadcast on the wire. That would decouple "did the system decide
correctly" from "did the network deliver the proof," which right now
are silently the same question under every trust-layer scenario in
this repo.
