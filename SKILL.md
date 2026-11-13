---
name: workflow-audit
description: Audit an LLM-agent development workflow for quality per cost. Applies to any pipeline where agents produce engineered artifacts and something checks them — coding agents, planner/implementer/reviewer chains, agent CI, eval suites, guardrails, prompt rulebooks, and comparable agent-driven design or document workflows. Ranks every check, gate, rule, and feedback loop by marginal leverage (defect cost averted per unit of cost), then proposes the lowest-leverage elements to remove and the highest-leverage elements to add. Use whenever the user wants to audit, slim down, or speed up an agent workflow; asks whether a reviewer agent, gate, eval, or prompt rule is worth its cost; says their agent stack is slow, expensive, or drowning in process; or wants to design the feedback loop that makes agent output improve over time.
---

# Workflow Audit

The objective is **quality per cost**, not quality and not leanness. A workflow is a set of *elements* — checks, gates, reviewer agents, prompt rules, evals, structural constraints, feedback loops. Each one costs something and averts something. The audit estimates both, ranks the elements, and proposes a small number of moves: remove what averts least per unit cost, add what would avert most.

"Nothing to change" is a legal result. So is "add a check." Do not start from a belief that the workflow is bloated; start from the numbers and say how good the numbers are.

## Core ideas

Read these before starting. The method is these ideas applied in order.

**Leverage is marginal.** An element's value is what would escape if only it were removed, given everything else in the workflow. Two reviewers that catch the same defects are each worth almost nothing given the other. So never rank elements by what they catch; rank them by what *only they* catch. Elements that work only together (a test suite, and the rule that the agent cannot edit the tests) are ranked as one bundle.

**Leverage has one unit.** Leverage = defect cost averted per week ÷ cost per week, both in the same currency (money or human minutes). Cost includes compute, added latency, false-positive rework, and maintenance. Token counts are an input, not a result. Formulas are in `references/leverage-matrix.md`.

**Use a dead band.** Estimates are noisy and leverage spans orders of magnitude, so only the top and bottom of the ranking can be trusted. Propose removal below 0.5, addition above 2, and leave the middle alone. If the data cannot support those thresholds, widen them and say so.

**An oracle is only an oracle if the generator cannot write to it.** A deterministic check (compile, tests, schema, rule check, simulation against reference results) is the cheapest and most reliable signal, but its verdict is only as good as its specification. An agent that can edit the tests, the rule set, the constraints, the waiver list, or the lint config can make the check pass without doing the work. Treat a writable oracle as a proxy until that write access is removed.

**Know where the optimization pressure is.** A deployed model does not adapt to your gates between runs. Pressure to game a check comes from three places: retry loops inside a run, the human tuning prompts against an eval set, and any memory that carries across runs. Look there for gaming, and require a held-out eval split for the second.

**Reversibility decays.** A mistake is cheap to undo only until other work is built on it. Classify work by consequence, by cost of undo, and by how long a defect goes undetected. Slow detection turns reversible work into irreversible work.

**Not everything can be ranked.** Elements fall into five lanes, and only one of them is ranked by the matrix:

| Lane | What it is | How it is judged |
|------|-----------|------------------|
| Structural | Makes a failure impossible: sandbox, scoped credential, schema, write-protection | Handled first. Near-zero running cost; cost is blocked tasks |
| Tail control | Guards rare, severe, hard-to-undo failures | By the consequence map, never by catch counts |
| Catcher | Detects defects in produced work: tests, reviewers, judges, gates | Ranked by unique catches per cost |
| Feedback loop | Turns an escaped defect into a change to the generator | By recurrence rate |
| Generator-side | Improves the input: specs, acceptance criteria, context, task sizing | By escape rate before and after, per task class |

## Method

Scale the effort to the target. A single agent loop with one reviewer needs a short pass; a multi-agent production pipeline deserves the full method. State the scale in the report.

### Step 1 — Check the data

The audit needs three things: a list of elements with their costs, a verdict log per catcher (which artifact, pass or fail, why), and a record of defects that escaped. Read the workflow's actual definitions and run traces before its documentation; where they disagree, the traces describe the real workflow and the disagreement is a finding.

If verdict logs or escape records do not exist, the deliverable is an **instrumentation plan** plus hypotheses labelled as hypotheses. Stop there. A ranked table built on guesses looks rigorous and is not. This is a common and useful first result.

Never invent timings, rates, or counts. Mark every figure as observed, seeded, or estimated. Do all arithmetic in code.

### Step 2 — Inventory the elements and their costs

List every element: what it checks or constrains, whether its verdict blocks or only advises, who or what consumes its output, how often it runs, and its cost per week in the common unit. Also list every capability each agent holds (file write, shell, network, credentials, tools, authority to trigger costly or external actions) and how often each was used in the traces.

An output nobody consumes has zero value by definition. It needs no matrix: propose removing it. The one exception is an output required by an outside party (regulator, customer, contract); generate that automatically from traces instead of with agent labor.

### Step 3 — Build the consequence map and assign lanes

For each kind of work the workflow produces, record consequence of a defect, cost of undo, and detection latency. From this, set the **cost of an escape per defect class** in order-of-magnitude buckets, and ask the workflow's owner to confirm them. These weights drive every later number, so state them in the report.

Then put each element in one of the five lanes. Anything guarding severe, hard-to-undo outcomes is a tail control: it stays out of the ranking, because a control that prevents a rare disaster will show zero catches and the ranking would delete it. For each tail control, check instead that it is the cheapest structure that does the job.

### Step 4 — Do the structural pass

Before ranking anything, look for failures that can be made impossible instead of checked for. Use the substitution table and capability menu in `references/improvement-patterns.md`. Three checks matter most:

- **Oracle integrity:** can any agent modify a check's specification? If so, remove that write access. This usually has the highest leverage in the audit.
- **Edicts that could be structure:** prompt rules of the form "never do X" where X can be made impossible.
- **Capabilities beyond use:** narrow or remove them, but measure the cost. Count tasks that failed, stalled, or prompted a human because a capability was missing. Rarely used is not the same as unneeded.

### Step 5 — Build the defect × element matrix

Rows are defects, columns are catchers, each cell says whether that catcher catches that defect. Fill it from verdict logs, from running catchers in shadow mode (non-blocking, verdict logged), and from seeded defects. `references/leverage-matrix.md` has the procedure, including how many samples a claim needs.

State what share of the rows are real and what share are seeded. With mostly seeded rows, the ranking is only as good as the resemblance between seeded and real defects; say this plainly in the report.

### Step 6 — Rank and propose moves

From the matrix:

- **Removal candidates:** catchers with leverage below the dead band. A catcher whose catches are all duplicated elsewhere has no unique value, whatever its rejection rate.
- **Addition candidates:** cluster the defects nothing catches by class and by cause. For the costliest cluster, find the cheapest element that would cover it, preferring in order: structure, a deterministic check, a calibrated judge, human review of samples and outliers.

Use `references/hypotheses.md` as a list of places to look. A matched pattern is a reason to measure something, not a finding.

Propose at most five moves, ordered by expected gain. Apply them one at a time unless they touch unrelated defect classes, so that each effect can be attributed. Each move states a prediction that can be checked later.

On inspection depth: sampling estimates a defect rate; it does not remove defects. For each risk tier, inspect everything if defect rate × escape cost exceeds the inspection cost per unit, and nothing otherwise. Keep a small random sample in the uninspected tiers to keep the rate estimate current.

### Step 7 — Check the feedback loop

Trace what happened after recent escapes.

- **Attribute each escape to a cause:** unclear or wrong specification, missing context, model limitation, or a gap in verification. The fix belongs where the cause is. Many escapes blamed on weak checking are specification problems, and a generator-side fix is usually cheaper than another catcher.
- **Compute the recurrence rate:** the share of escapes belonging to a class that escaped before. A working loop drives this toward zero. This is the loop's one headline metric.
- **See where fixes land.** In order of preference: a structural or tool change; a regression eval plus the generator change that makes it pass; a stored case retrieved when relevant; a prompt rule. A prompt rule is acceptable when an eval shows it changes the outcome. A rule with no measured effect is removed.
- **Check the eval suite itself:** does it have a held-out split that prompts are not tuned against, and a retirement rule for cases that have passed for a long time? An eval suite that only grows is the same ratchet in a different place.
- **Check ownership:** one named person can change the workflow. If changes need a committee, nothing above will happen.

### Step 8 — Report, then re-audit

Write the report using the template below. On the next audit, score the previous predictions first: an auditor whose predictions are not checked is an uncalibrated judge.

Re-run the removal side of the audit whenever the underlying model changes. Rules and checks built for a weaker model are the largest source of dead weight; ablate each prompt rule against the eval suite and drop those that no longer change the result.

## Report template

Keep it short. The reader operates the workflow and wants the moves and the evidence.

```
# Workflow audit: <name>
Scale: <single loop | pipeline | fleet>   Date: <date>
Data basis: <share observed / seeded / estimated>; escape-cost weights; oracles relied on

## 1. Elements
element | lane | cost/wk | unique value/wk | leverage | confidence | basis

## 2. Moves (max 5, in order)
move | remove / add / substitute / reduce capability | predicted effect | how it will be tested | reversal condition

## 3. Feedback loop
Recurrence rate; escapes by cause; where fixes land; eval hold-out and retirement; owner

## 4. Outside the ranking
Structural changes; tail controls kept, with the risk each one covers

## 5. Gaps
What could not be measured, and the instrumentation that would fix it

## 6. Previous predictions (re-audits only)
prediction | outcome
```

If the audit stopped at Step 1, the report is sections 2 (as hypotheses) and 5 only.

## Judgment rules

**Do not delete on absence of evidence.** "Never caught anything" means low leverage only for a catcher on frequent, observable defects. For a tail control it means nothing.

**Price the cost of every reduction.** Removing a capability or a check has a cost too: blocked tasks, workarounds, human interruptions. A reduction that sends agents to a human for permission has added the most expensive check of all.

**A reviewer agent is not wrong in principle.** A checker does not need to be trusted; it needs a measured error rate and errors that are not the same as the generator's. An unmeasured reviewer is a hypothesis. A measured one is ranked like any other catcher.

**Say what the cheap signals miss.** Some quality has no fast signal: maintainability, design coherence, slow accumulation of complexity. A workflow tuned only on what is cheap to measure will trade these away. Name them in the report as unmeasured, and suggest a slow periodic check if the owner cares about them.

**Flag bypasses.** If operators routinely skip or override an element, treat that as strong evidence about its real leverage, and as a sign the documented workflow is not the one being run.

**Flag missing structure too.** If a decision that matters is made informally (one person's judgment, an unversioned script, a prompt that lives outside the repository), the move is to make it explicit, not to remove something.
