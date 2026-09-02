# Marketplace minority partition experiment

## Question and choice

What happens when a shared registry lists every seller, while 10 percent of
the sellers cannot receive requests?

I chose this question because an agent market depends on service discovery
and current reachability. A newsroom service can remain in a registry during
a cloud outage or network isolation.

This scenario copies the built-in `marketplace` scenario and adds one
supported setting: `failures.network_partition`. The main group contains all
50 buyers and 45 sellers. The isolated group contains `seller-45` through
`seller-49`. Every other runtime setting remains unchanged. The scenario has a
distinct name, description, and default trace path so a default run cannot
overwrite the baseline trace.

## Hypothesis

Each seller choice has a 10 percent chance of selecting an isolated seller.
A buyer starts its next round only after it receives `sold` or `reject`. The
buyer has no timeout or retry path. The probability that one buyer avoids the
isolated sellers for all ten rounds is:

```text
0.9^10 = 34.9%
```

I expected about 35 percent of buyers to finish all ten rounds. I also
expected the effect on total trades to exceed the 10 percent seller outage.

## Run the experiment

```bash
nest run marketplace -o ./traces/baseline.jsonl
nest run ./examples/marketplace-minority-partition/scenario.yaml \
  -o ./traces/minority-partition.jsonl

nest inspect ./traces/baseline.jsonl
nest inspect ./traces/minority-partition.jsonl
```

## Results

| Metric | Baseline | Minority partition |
|---|---:|---:|
| Purchase requests | 500 | 342 |
| Answered requests | 500 | 312 |
| Sales | 266 | 187 |
| Unanswered requests | 0 | 30 |
| Buyers that finished all rounds | 50 | 20 |
| Sales per purchase request | 53.2% | 54.7% |

The marketplace validators reported:

```text
PASS marketplace_no_double_sell - checked 187 sales
FAIL marketplace_all_responded - 30 unanswered buy requests
PASS marketplace_price_agreement
```

## Investigation

All 30 dropped messages targeted the five isolated sellers. The failures came
from 30 different buyers. The seller counts were 5, 4, 6, 8, and 7 dropped
requests for `seller-45` through `seller-49`.

I then checked the buyer state logic. A `sold` or `reject` response advances
the buyer to its next round. A dropped request produces no response, so the
buyer never advances. The shared registry still returns the isolated sellers
because it has no live reachability check.

The observed completion rate was 40 percent, close to the 34.9 percent
hypothesis. Sales per attempted request stayed near the baseline rate. Total
sales fell from 266 to 187 because stalled buyers never made 158 later
requests.

## What I learned

Registry membership does not establish current reachability. A practical
agent market needs health-aware discovery plus client deadlines, limited
retries, and selection of another qualified provider. In a newsroom setting,
the agent should report that verified information is unavailable when no
approved provider can respond.

## AI and other help

I used OpenAI Codex to read the NANDA Town documentation, edit the scenario,
run the CLI, inspect traces, count dropped events, check the buyer state logic,
and draft this README. I chose the research question and final partition
design, ran the experiment in my local repository, and reviewed each output.
