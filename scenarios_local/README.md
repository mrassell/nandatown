# NANDA Town Assessment — Marketplace / message_drop

## Setting changed
`failures.message_drop`: `0.0` → `0.05` (scenario copied to
`scenarios_local/my_marketplace.yaml`, all other settings untouched).

## Hypothesis
A 5% chance of any message getting dropped would lower `delivery_rate`
by roughly 5%, and would probably also lower `deal_rate`, since fewer
completed round-trips should mean fewer completed deals.

## What happened (baseline → 5% drop)

| metric | baseline | 5% drop |
|---|---|---|
| delivery_rate | 1.0000 | 0.9410 |
| dropped_count | 0 | 35 |
| deal_rate | 0.5320 | 0.5724 |
| rejection_rate | 0.4680 | 0.3783 |
| message_count | 2000 | 1151 |
| unique_pairs | 467 | 286 |
| validator: marketplace_all_responded | PASS | **FAIL — 15 unanswered buy requests** |
| validator: marketplace_no_double_sell | PASS | PASS |
| validator: marketplace_price_agreement | PASS | PASS |

`delivery_rate` moved about where the hypothesis predicted. `deal_rate`
did not — it went *up*, not down, and message volume collapsed far more
than 5% would explain (2000 → 1151).

## What surprised me, and how I investigated it

The `deal_rate` increase was the surprise. I inspected the raw trace
(`nest_core` JSONL, not just the report) and counted `send` events per
buyer agent:

- **Baseline:** all 50 buyers send exactly 10 times each (one attempt
  per configured round, no exceptions).
- **5% drop:** only 16 of 50 buyers complete all 10 rounds. The other
  34 stop early, some after a single attempt. Average sends per buyer
  drops from 10.0 to 6.08.

So the buyer agents have no retry or timeout logic: if the one message
that resolves a given round gets dropped, that buyer's state machine
never advances to its remaining rounds for the rest of the run. A
single 5%-probability event early in a buyer's sequence can silently
cancel most of its remaining negotiation attempts. That's what the
"15 unanswered buy requests" validator failure is actually catching —
not 15 independent drops, but a small number of drops each stalling a
buyer that then never resolves its pending request.

That also explains the `deal_rate` increase without needing any
change in negotiation dynamics: negotiations here resolve in a single
request/response pair (avg. 2.0 messages/negotiation in both runs, no
multi-round counter-offers observed), so the shift isn't about
negotiations behaving differently. It's that the *sample* of buyers
who make it to a resolved outcome is a survivorship-biased subset —
whichever buyers happened not to get an early drop — not a
representative one. `deal_rate` here reports on the ones who got
through, not on demand as a whole.

## Tools and help used
Explored the repo, installed `nest-core[plugins]`, ran `nest doctor`,
`nest scenarios cp/show`, `nest run`, `nest report`, and the
`validate_trace` snippet from the quickstart doc. Used Claude
(Claude.ai, with code execution) to drive the CLI, write the Python
trace-inspection snippets (send/receive/dropped counts, per-buyer send
counts, corr-id chain lengths), and draft this README from the actual
metric output. All commands and their output are reproducible from
this scenario file and the two trace files included here.

## What I'd build next for NANDA Town
A retry/backoff-and-resume layer for state-machine agents, so a single
dropped message stalls only that round instead of silently canceling
every subsequent round for that agent. Right now `message_drop` isn't
really testing "how well does the protocol tolerate an unreliable
network" — it's mostly testing "does this agent implementation have
any recovery logic at all" (it doesn't). A resumable agent would let
`message_drop` sweeps actually isolate protocol-level robustness from
agent-implementation gaps like this one.
