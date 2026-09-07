---
name: ship
description: >
  Automates the implement → review → fix loop for a feature/task, unit by
  unit, until each unit is clean per /review or a per-unit safety cap is hit.
  Low-risk findings are auto-applied and logged; medium/high-risk findings
  always stop and wait for explicit confirmation before anything is changed.
  Exposes a single command: /ship <task description>. Use when the user
  invokes /ship, or asks to implement something and have it reviewed and
  cleaned up automatically rather than reviewing it themselves afterward.
---

# Ship — Implement → Review → Fix Loop

This skill defines no rules of its own. It orchestrates two existing skills — writing code per
`../develop`'s conventions, then checking it via `/review` (which itself composes `../develop` and
`vendor/ponytail`, and scores every finding per `../review/references/RISK.md`) — and adds only the
loop/triage logic below. Read `../develop/SKILL.md` and `../review/SKILL.md` at the time each step
runs; never copy their rules or `RISK.md`'s criteria into this file.

## 1. Task sizing

Before implementing anything, estimate the task's scope from the description. If it clearly spans
multiple independent components/modules — touching more files/lines than a reasonably-sized single
diff, and the pieces don't depend on each other to make sense in isolation — split it into
independent units, each implementable and reviewable on its own. If it's small enough to be one
coherent change, treat it as a single unit.

State the split (or the decision not to split, and why) before starting implementation, so the
plan is visible before any code is written.

## 2. Per-unit loop

Run this loop independently for each unit (the whole task, if not split, or each sub-task):

**a. Implement.** Write the unit's code following `../develop`'s conventions — read
`../develop/references/GENERAL.md` first, then whichever other reference files `develop`'s own
Workflow says apply to the layers this unit's files touch.

**b. Review.** Invoke `/review` scoped to just this unit's diff.

**c. Triage every finding `/review` returns:**

- **`risk: low`** — apply the fix automatically, no confirmation needed. Log what changed and why
  (the finding it resolved).
- **`risk: medium` or `risk: high`** — stop immediately. Show the full finding (file, lines, rule
  violated and its source, evidence, suggested fix, and the finding's specific risk reason from
  `RISK.md`). Ask explicitly whether to apply it. Wait for the answer before doing anything else —
  don't queue it, don't continue to other findings or other units until this one is answered.
  - **If the answer is yes** (with or without a modification): apply the fix as instructed, log it,
    resume the loop from where it stopped.
  - **If the answer is no:** do not apply it. Record this exact finding as **seen-and-declined for
    this unit, for the remainder of this `/ship` run** — its identity is its file, line range, rule
    violated, and evidence. Resume the loop from where it stopped.
  - **If the answer is a modified instruction** (e.g. "apply it, but do X instead"): apply what was
    actually instructed, log it as a modified fix (not the original suggestion), resume the loop.

**d. Re-review after auto-fixes.** If any low-risk fixes were applied in this pass, re-run `/review`
on the same unit to confirm they didn't introduce new findings, then re-triage (back to step c) on
whatever comes back.

**e. Handling a re-triggered declined finding.** When a `/review` re-run (step d, or a later
iteration) returns a finding that matches an already-declined one's identity (same file/lines, same
rule/source, same evidence) **unchanged**, do not prompt again — it stays declined, carried forward
silently for the rest of this unit's loop in this run. Only re-prompt when the finding has
*materially* changed (different lines, or the rule/evidence/reasoning is different from what was
declined) or when it's a genuinely new finding never shown before. This tracking is scoped to the
current `/ship` run for this unit — it does not persist across separate `/ship` invocations.

**f. Stopping condition.** A unit is clean once a `/review` pass returns nothing that is either
auto-appliable (low risk) or a new/changed finding requiring a prompt — a previously-declined
medium/high finding reappearing unchanged does not, by itself, keep the loop going; it's carried
into the final summary as declined, not re-litigated. Repeat b–d until the unit reaches that state,
or until **5 iterations** (one iteration = one full b–d pass) have run for this unit, whichever
comes first. The cap is per unit, never shared or accumulated across units.

**g. Safety cap hit.** If 5 iterations pass without the unit reaching the clean state in **f**, stop
looping this unit. Show the current state and the remaining (non-declined, unresolved) findings, and
ask explicitly how to proceed: continue past the cap, accept the unit as-is, or abandon this unit.
Do not loop silently past the cap.

## 3. Final summary

After all units are clean (or resolved via input during the loop), produce one summary covering,
per unit: what was implemented, and how many review iterations it took.

Then, across all units, split findings into two distinct, separately-headed sections — never
merged into one list, so the summary reads as a risk register rather than prose:

- **Resolved** — every auto-applied low-risk fix, and every medium/high finding answered yes
  (including applied-with-modification, noted as such). One line per finding: file, lines, what was
  fixed.
- **Declined — accepted as-is** — every medium/high finding answered no and carried forward per
  step 2e. One line per finding: file, lines, the finding's risk reason, and that it was declined.

Close with any unit that stopped at the safety cap and how it was resolved (continue past cap /
accepted as-is / abandoned).

## Constraints

- Never auto-apply a medium or high risk fix, regardless of how many times it recurs across
  iterations or units, and regardless of a prior "yes" on a similar-looking finding elsewhere —
  each medium/high finding is confirmed on its own.
- The 5-iteration safety cap is per unit, not global across the task.
- This skill must not reimplement `/review`'s analysis or `develop`'s conventions — only orchestrate
  calls to them and follow their output, referencing both by path.
