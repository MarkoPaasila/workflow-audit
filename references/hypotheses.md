# Hypotheses: where to look

Patterns that often mark low-leverage elements or missing ones. A match is a reason to measure, not a finding. Each entry gives the signal, how to confirm it, and the usual move.

## Likely low leverage

**L1 — Output with no consumer.** A report, summary, review comment, dashboard, or log that no person or tool reads or acts on.
*Confirm:* find the consumer; check whether any decision ever changed because of it.
*Move:* remove. If an outside party requires it, generate it from traces.

**L2 — Duplicate catcher.** Two elements checking the same property: a reviewer restating what a linter reports, a pre-check of something a later deterministic check tests anyway, overlapping rubrics.
*Confirm:* unique catches in the matrix.
*Move:* remove the costlier one, or the later one if catching early saves little.

**L3 — Unmeasured judge.** A reviewer agent, model-based judge, or scoring heuristic with no known false-positive or false-negative rate. Agreement between two unmeasured judges proves nothing.
*Confirm:* is there a calibration set, and when was it last used?
*Move:* measure it (seeded defects, labelled sample), then rank it like any catcher.

**L4 — Prevention where detection is fast and undo is cheap.** Approval or review ahead of work that would be detected quickly and reverted easily.
*Confirm:* detection latency and cost of undo from the consequence map. If detection is slow, this pattern does not apply.
*Move:* substitute staged release with automatic rollback (S9), or risk-tier the review (S8).

**L5 — Uniform depth across risk tiers.** Every unit of work gets the same verification regardless of consequence.
*Confirm:* compare defect rate × escape cost with inspection cost per tier.
*Move:* inspect all or none per tier; keep a small sample in uninspected tiers.

**L6 — Rule with no measured effect.** A prompt rule, often added after an incident, never tested against an eval; or a rule restating what a tool already enforces.
*Confirm:* ablate it against the eval suite.
*Move:* remove if no effect; substitute structure if it guards a real failure.

**L7 — Capability beyond use.** Broad tools, standing credentials, or open network access granted for convenience, along with checks that exist only to police them.
*Confirm:* join the capability inventory against tool use in traces; check blocked-task risk.
*Move:* reduce by the capability menu, and remove the policing checks with it.

**L8 — Routinely bypassed element.** Operators skip, override, or ignore it; the real decision happens elsewhere.
*Confirm:* override rates, time between verdict and action, operator accounts.
*Move:* remove or make advisory; investigate what the operators rely on instead.

## Likely missing

**M1 — Writable oracle.** The generating agent can edit tests, rule sets, constraints, waivers, or check configuration.
*Confirm:* write permissions; traces showing checks changed in the same run that made them pass.
*Move:* S2. Usually the highest-leverage change available.

**M2 — Deterministic check available but not in the loop.** A compiler, type checker, linter, test suite, simulator, or validator exists and the agent's loop does not run it, while a judge or reviewer stands in.
*Confirm:* per defect class, ask what deterministic check would pin it and whether the loop runs it.
*Move:* wire it in; then re-rank the judge that was covering for it.

**M3 — No escape record.** Nobody can state the escaped-defect rate, or escapes are not recorded with class and cause.
*Confirm:* ask for last month's escapes.
*Move:* instrumentation first; the audit cannot rank without it.

**M4 — Loop ends in rules, not measurement.** Incidents produce new prompt rules or new gates, never a regression eval or a structural change; the same class recurs.
*Confirm:* recurrence rate; where the last several fixes landed.
*Move:* redirect fixes up the slots in `improvement-patterns.md`.

**M5 — Prompts tuned on the same evals that judge them.** No held-out split; eval scores rise while escapes do not fall.
*Confirm:* compare the eval trend with the escape trend.
*Move:* hold out a split; add retirement.

**M6 — Specification-caused escapes treated as verification gaps.** Escapes where the agent built the wrong thing correctly, answered with more review.
*Confirm:* attribution of recent escapes by cause.
*Move:* generator-side fix: acceptance criteria, executable acceptance test, smaller tasks, better context.

**M7 — Informal decision point.** A decision that determines outcomes is made in one person's head, an unversioned script, or a prompt kept outside the repository.
*Confirm:* ask what happens when that person is away.
*Move:* make it explicit and versioned. This is an addition.

**M8 — No owner.** Changes to the workflow need consensus or a ticket queue; improvement requests sit for months.
*Confirm:* name the person who can remove an element today.
*Move:* assign one. Without this, no other move will be carried out.

**M9 — Unmeasured slow quality.** Nothing tracks properties with no fast signal: maintainability, design coherence, accumulating complexity.
*Confirm:* ask whether work is getting harder to change over time, and what would show it.
*Move:* a periodic slow check on a sample; name the gap in the report either way.
