# Voting threshold experiment: when healthy metrics hide changed decisions

## Question

What changes when the built-in voting scenario requires a 75% supermajority
instead of a simple majority?

## One setting changed

The experiment copies the built-in `voting` scenario and changes only one
supported task setting:

```yaml
task:
  type: voting
  config:
    rounds: 5
    threshold: 0.75  # baseline: 0.5
```

The seed, agents, layers, failures, duration, and other task settings are
unchanged.

## Hypothesis (written before the experiment)

Each voter independently votes yes with probability 0.7. I therefore expect a
75% threshold to reject some proposals that pass under the 50% baseline. Since
every voter should still receive a proposal and return one vote, I expect the
message counts, delivery rate, validators, and generic `success_rate` to remain
unchanged. If so, the standard health metrics will hide a meaningful governance
change: whether proposals actually pass.

## Run it

```bash
uv sync

# Baseline
uv run nest run scenarios/voting.yaml -o traces/voting-baseline.jsonl
uv run nest report traces/voting-baseline.jsonl -o traces/voting-baseline-report.html

# One-setting experiment
uv run nest run scenarios/voting_supermajority.yaml
uv run nest report traces/voting_supermajority.jsonl \
  -o traces/voting-supermajority-report.html
```

## Results

The vote stream was identical in both traces: 90 vote sends, with all messages
delivered. Only the coordinator's decision changed.

| Round | Yes votes | 50% baseline | 75% experiment |
| --- | ---: | --- | --- |
| 1 | 12/18 | passed | rejected |
| 2 | 11/18 | passed | rejected |
| 3 | 12/18 | passed | rejected |
| 4 | 9/18 | passed | rejected |
| 5 | 15/18 | passed | passed |

The one-setting change flipped four of five decisions. It did not change any
standard report metric: both runs reported `success_rate: 1.0`,
`delivery_rate: 1.0`, `message_count: 370`, and `dropped_count: 0`. All three
voting validators passed in both runs:

```text
PASS voting_tally_correct - checked 5 rounds
PASS voting_all_counted
PASS voting_no_double_vote - checked 90 votes
```

The experiment is deterministic: a second run produced the same SHA-256 trace
hash, `23394e121367cd51a407c289fc9b0ef71f0597b5bd1d6d3aa8ebdebe96af1cee`.

## Investigation and lesson

I expected some decisions to flip, but four of five initially looked extreme.
I checked the raw `result:` events and confirmed that a 75% threshold requires
at least 14 of 18 yes votes; only round 5 reached it. I then compared all 90
`vote:` sends between traces and found them byte-for-byte identical, ruling out
a changed random stream. Finally, under the code's 0.7 yes probability, a
binomial calculation gives a 0.333 probability that any round reaches 14 yes
votes and a 0.330 probability of exactly one pass in five. The result was
therefore plausible, not a simulator bug.

The deeper surprise was observability: `success_rate` measures delivery, and
the validators check tally integrity, completeness, and duplicate votes. None
reports proposal outcomes. The system can therefore look perfectly healthy
while a policy change reverses 80% of its decisions. A useful next metric would
be `proposal_pass_rate`, reported alongside the configured threshold and each
round's yes-vote margin.

## AI and other help

I used OpenAI Codex to inspect the course instructions and repository,
short-list a small experiment, preserve the hypothesis before execution,
run the scenarios, analyze the JSONL traces, and edit this README. I checked
the generated work by reading the scenario implementation, rerunning the
commands, inspecting the concrete trace events, and reviewing the final diff.
No human help or external tutorial was used.
