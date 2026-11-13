# Leverage Matrix

How to build the defect × element matrix and compute leverage from it. Do every calculation in code and label each figure observed, seeded, or estimated.

## Definitions

```
escape_cost[class]      cost of one escaped defect of that class, in the common unit
                        (order-of-magnitude buckets: 1, 10, 100, 1000; confirmed by the owner)

unique_value_per_week   = Σ over defects caught ONLY by this element, per week, of escape_cost[class]
shared_value_per_week   = Σ over defects also caught later by another element, of
                          (cost if caught at the later element − cost if caught here)
value_per_week          = unique_value_per_week + shared_value_per_week

cost_per_week           = runs × (compute_cost + added_latency × delay_cost)
                          + false_positives × rework_cost
                          + maintenance

leverage                = value_per_week / cost_per_week
```

Notes:

- **Shared catches** earn only the saving from catching earlier, which is usually small. This is what stops duplicate checks from each looking valuable.
- **Delay cost** is the owner's price for one unit of waiting. If they cannot give one, report leverage with latency excluded and list the latency separately.
- **False positives** count fully. A check that blocks good work and triggers rework can have negative net value even with real catches.
- **Maintenance** includes keeping the check's specification current and dealing with flakiness.

## Filling the matrix

Rows are defects, columns are catchers, cells are caught / not caught / not evaluated.

### Source 1 — Real defects

- **Caught defects:** from each catcher's verdict log. For each, find out whether any other catcher would also have caught it. If the pipeline stops at the first failure, later catchers never saw the artifact, so the cell is "not evaluated", not "not caught". Resolve it with shadow mode.
- **Escaped defects:** from the escape record (bug reports, reverts, rework, incidents). These rows are all-zero by definition and are the input to the addition side.

### Source 2 — Shadow mode

Run every catcher on every artifact without blocking, and log the verdicts. This fills in the "not evaluated" cells and shows true overlap. It also lets you trial a candidate element before adopting it, and observe a removal before making it: switch the element to non-blocking, keep logging, and see what it would have caught that nothing else did.

Shadow mode costs the run cost of every catcher on every artifact. Run it for a bounded window, or on a random sample.

### Source 3 — Seeded defects

Insert known defects into otherwise good artifacts and record which catchers flag them. This gives rows quickly, including for defect classes that are rare in the logs.

- Draw the seeds from the classes in the escape record first, in proportion to their escape cost. Add classes the owner fears but has not seen.
- Make seeds resemble real agent errors: plausible, locally consistent, wrong in substance. Trivially broken seeds flatter every catcher.
- Keep seeded and real rows distinguishable in every table.
- Seeding also estimates what is still hidden. If a catcher finds s of S seeded defects and r real ones, the real defects it missed are roughly r × (S − s) / s. Use this as a rough bound only.

Seeded rows measure whether a catcher *can* catch a class. They do not measure how often that class occurs. Take frequencies from real data, or state that they are assumed.

## Reading the matrix

**Removal side.** For each catcher, compute unique and shared value. Zero unique catches with a meaningful cost is the clearest removal candidate. Before proposing removal of any catcher, check that the element that duplicates it is not itself on the removal list.

**Addition side.** Take the rows nothing catches. Cluster by defect class and by cause (specification, context, model limitation, verification gap). Weight each cluster by frequency × escape cost. For the heaviest cluster, list candidate elements and estimate each one's cost; the best candidate is the highest predicted leverage, not the highest coverage.

**Complements.** If an element's catches depend on another element being present, compute leverage for the pair.

## How much data a claim needs

Most audit claims are about rare events, and small samples cannot support them.

- **Zero events in n trials** bounds the rate at roughly 3 / n with 95% confidence. A check that rejected nothing in 50 runs may still have a true rejection rate of up to 6%.
- **Detecting a change in a rate p** between two periods, at conventional significance and power, needs roughly 24 / p runs per period to see a doubling and 8 / p to see a tripling. At a 1% escape rate that is about 2,400 and 800 runs.
- Consequence: an "observe after removal" plan based on escape counts is usually too weak to detect harm within a few weeks. Prefer shadow mode, which measures what the removed element would have caught directly, and seeded defects, which need tens of trials per class instead of thousands of runs.

State the sample size behind every rate in the report. Where it is too small to support the dead-band thresholds, mark the element "undetermined" and leave it in place.

## Confidence labels

Use three levels in the report's elements table:

- **High:** leverage is outside the dead band by a factor of 3 or more, on mostly real data.
- **Medium:** outside the dead band, on mixed real and seeded data.
- **Low:** inside or near the dead band, or mostly seeded or estimated. No move is proposed on low confidence alone.
