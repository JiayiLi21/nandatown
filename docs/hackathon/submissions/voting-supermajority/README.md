# Voting scenario: simple majority vs. a 75% threshold

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

## What I expected

In the code, each voter has a 70% chance of voting yes. I expected that moving
the threshold from 50% to 75% would turn some passing proposals into rejections.
I did not expect it to change message delivery or the validator results, because
the agents would still send and count the same votes.

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

The votes were identical in both runs. All 90 votes were delivered; only the
coordinator's final decision changed.

| Round | Yes votes | 50% baseline | 75% experiment |
| --- | ---: | --- | --- |
| 1 | 12/18 | passed | rejected |
| 2 | 11/18 | passed | rejected |
| 3 | 12/18 | passed | rejected |
| 4 | 9/18 | passed | rejected |
| 5 | 15/18 | passed | passed |

Changing the threshold flipped four of the five decisions. However, both runs
still reported `success_rate: 1.0`,
`delivery_rate: 1.0`, `message_count: 370`, and `dropped_count: 0`. All three
voting validators passed in both runs:

```text
PASS voting_tally_correct - checked 5 rounds
PASS voting_all_counted
PASS voting_no_double_vote - checked 90 votes
```

## What I checked

Four rejections out of five was more than I expected, so I checked the raw trace.
A 75% threshold means at least 14 of the 18 voters must vote yes. Only the last
round reached that number. I also compared all 90 vote messages between the two
runs; they were the same, so the threshold did not change the random votes.

As a final sanity check, I calculated the probability using the 70% yes rate in
the code. A round has about a one-in-three chance of getting at least 14 yes
votes, so seeing one passing round out of five is not unusual.

## What I learned

The surprising part was not the vote math. It was that the reports looked
exactly as healthy before and after the change. Here, `success_rate` describes
message delivery, while the validators check that votes were counted correctly
and only once. They do not say how many proposals passed. I would add a simple
`proposal_pass_rate` metric and show each round's margin, so a reader can see the
effect of changing the decision rule.

## AI and other help

I used OpenAI Codex to help me read the repository, run the two scenarios,
compare the JSONL traces, and edit this README. I used the repository's
quickstart, scenario guide, voting implementation, validators, and
`CONTRIBUTING.md`. I did not use an external tutorial or human help.
