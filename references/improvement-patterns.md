# Improvement Patterns

Reference material for the structural pass (Step 4), for choosing additions (Step 6), and for the feedback loop (Step 7).

## Substitution table: edict to structure

Each row replaces a rule the agent is asked to follow with an arrangement in which the failure cannot happen. Prefer substitution to plain removal when the rule guards a real failure. The last column is the cost to watch for.

| ID | Rule (edict) | Structural replacement | Cost to watch |
|----|--------------|------------------------|---------------|
| S1 | "Do not modify anything outside your task" | Sandboxed workspace; allowlisted write paths | Tasks that legitimately span areas |
| S2 | "Do not weaken or edit the checks" | Check specifications (tests, rule sets, constraints, waivers, configs) read-only to the generator; changes go through a separate path | Legitimate spec updates slow down |
| S3 | "Do not touch live data or systems" | Read-only replica; scoped, short-lived credentials | Tasks needing fresh or live state |
| S4 | "Only call approved services" | Network allowlist or egress proxy | New dependencies need an allowlist change |
| S5 | "Format the output correctly" | Schema validation; constrained decoding; typed tool interfaces | Schema too rigid for edge cases |
| S6 | "Ask before doing anything irreversible" | Tool layer where irreversible operations are dry-run first and need a second key to apply | Latency on the second key |
| S7 | "Make sure the checks pass before submitting" | Required status checks enforced by the system, not by the agent | Flaky checks block good work |
| S8 | "Review everything" | Risk-tiered routing: deterministic checks clear low-risk work; judgment only for consequential classes | Misclassified risk tier |
| S9 | "Get approval before releasing" (work that is quickly detected and cheaply undone) | Staged release with automatic rollback on a measured signal | Requires a trustworthy signal |
| S10 | "Follow the checklist" | Steps as executable pipeline; gates as checks | Pipeline maintenance |
| S11 | "Do not leak secrets or restricted content" | Scanner on output; secrets never placed in the agent's context | Scanner false positives |
| S12 | "Use only approved components or dependencies" | Pinned versions; approved-component library the tool resolves against | Library upkeep |
| S13 | "Double-check the numbers" | Deterministic recomputation; comparison against reference results | Reference results go stale |
| S14 | "Keep the status report updated" | Status derived from the system of record | None significant |
| S15 | "Follow the design rules" (prose in a prompt) | Rules encoded once in the checking tool and versioned; the prompt does not restate them | Rules that cannot be encoded remain as judgment |

## Capability reduction menu

For each capability in the inventory, try in this order:

1. **Remove** — no legitimate use found.
2. **Narrow** — a smaller scope does the job (two hosts instead of open network; one directory instead of the filesystem).
3. **Shorten** — a per-run credential instead of a standing one.
4. **Downgrade** — read-only instead of read-write; propose instead of execute.
5. **Move** — the capability belongs to the orchestrator or a tool boundary, not the model.

Every reduction removes the checks that existed to police that capability; count those savings. Every reduction also risks blocked tasks. Measure the blocked-task rate (tasks that failed, stalled, or required a human because of a missing capability) before and after. Rare use is not proof of no need: a capability used in 1% of runs may be what makes those runs possible.

## Feedback ladder

When adding a signal, take the cheapest rung that can produce it.

1. **Deterministic checks.** Compile, type-check, tests, schema validation, rule checks, simulation against reference results, "did the change apply". Cheapest and most reliable, with two conditions: the generator cannot write to the check's specification (S2), and passing the check is understood as necessary, not sufficient. A common finding is that such a check exists but is not wired into the agent's loop.
2. **Calibrated judgment.** A model judging the *artifact* (the diff, the output, the final state) against explicit criteria, with a measured error rate from a human-labelled calibration set that is refreshed when the task mix or the model changes. A judge with no measured error rate is a hypothesis, not a signal. Prefer a judge whose errors differ from the generator's: different model, fresh context, or different information.
3. **Human review.** For aggregates, outliers, and the high-consequence tier. Not the default for volume.

For any signal, verify four properties:

- **Latency:** time from the defect being introduced to the signal firing.
- **Destination:** its output reaches someone or something able to change the generator.
- **Anchor:** what ties it to reality (rung 1: a protected specification; rung 2: a calibration set; rung 3: agreement checks between reviewers).
- **Gaming move:** what a retry loop or a prompt-tuner optimizing against this signal would do, and whether that still serves the intent.

## Where fixes should land

When an escape is traced to its cause, the fix goes in the highest slot that works:

1. **Structure or tooling.** The failure class becomes impossible or is caught deterministically. Zero cost per run afterward.
2. **Regression eval plus generator change.** The eval detects recurrence; it does not fix anything by itself, so it comes with the change (context, tool, example, or prompt edit) that makes it pass.
3. **Experience store.** The failure is stored as a minimal case with its fix and retrieved only when a similar task appears. Each stored case needs: the triggering situation, what went wrong, the fix, and the date and model it was recorded under. Stored cases go stale and need the same retirement rule as evals.
4. **Prompt rule.** Acceptable when an eval shows the rule changes the outcome. Cheap to add, applies immediately, but dilutes attention as rules accumulate, and it is the slot with the least evidence required. Any rule without a measured effect is removed.

Generator-side fixes deserve a first look for any escape attributed to specification or context: clearer acceptance criteria, an executable acceptance test written before the work, smaller tasks, or better context supplied to the agent. These prevent defects instead of catching them.

## Keeping the workflow from growing back

A small set of standing mechanisms, to propose where they fit:

- **Re-ablate on model change.** When the underlying model changes, test each prompt rule and each catcher again. Remove what no longer changes results.
- **Dead band as standing policy.** New elements are adopted only with predicted leverage above the upper threshold and are measured in shadow mode first.
- **Eval retirement.** Cases that have passed for many consecutive versions move to an occasional sample. Keep a held-out split that prompt tuning never sees.
- **Origin pointer.** Every rule, check, and eval records the failure that produced it. One with no traceable origin is the first to be measured.
- **One named owner** with authority to add and remove elements.
- **Incident outputs.** An incident review ends in an attributed cause and a fix in one of the four slots above. Adding a standing check is allowed only with a leverage estimate.
