---
name: preregister
description: Generate a pre-registration file for a search or content experiment under docs/experiments/. Use when starting any experiment that will be read at a later date, or when asked to pre-register, freeze a baseline, or define thresholds and a decision rule before a change ships.
---

# preregister

Writes `docs/experiments/<ticket>-<slug>.md`. Pre-registration exists so the read at the end of the
window cannot be argued with, which only works if every number is fixed before the change ships.

## The rule that overrides everything else

Never invent a number. Not a baseline, not a threshold, not a window, not a sample size. Every
number in the output must have been given to you or read out of a pasted export. Anything you were
not given is written as the literal string `FILL IN`, left in place for a human. A plausible
guess is worse than a blank, because a blank gets filled and a guess gets trusted.

## Required sections

1. **Hypothesis.** One sentence, stating the change and the expected direction of the effect.
2. **Cohorts.** The exact segment definitions, as regexes where applicable, case sensitivity
   stated. Reference `docs/analytics/query-cohorts.md` for the canonical definitions rather than
   restating them loosely.
3. **Primary metric.** The one metric the decision turns on. One metric, not a list.
4. **Guardrail metrics.** What must not regress for the result to count, with the regression
   threshold for each.
5. **Frozen baseline.** Copied verbatim, with the exact window it covers and the export date.
   Marked `FROZEN, do not modify`.
6. **Thresholds.** The numeric cut points, each mapped to a named outcome.
7. **Read window.** The start trigger and the read date, for example deploy timestamp plus 28 days.
   If the deploy has not happened, write `DEPLOY_TIMESTAMP: fill after deploy` and derive the read
   date from it rather than from today.
8. **Confounder checks.** What else shipped or could ship in the window that could move the metric,
   each with how it will be separated out. Name specific known events, not a generic caution.
9. **Decision rule.** Written so that the outcome follows mechanically from the numbers. Cover
   every branch including the inconclusive one, and state what happens on inconclusive: extend,
   abandon, or escalate.
10. **Freeze rule.** What may not be changed during the window, and why.

## Steps

1. Confirm the ticket number and slug. Confirm `docs/experiments/` exists, create it if not.
2. Collect every number from the requester. List back which of the required numbers you were not
   given, before writing the file.
3. Write the file with all ten sections. Mark unknowns `FILL IN`.
4. Report which fields are `FILL IN` and who needs to fill each one. Do not close the task
   silently with blanks left in the file.
5. Writing style: no em dashes, exact strings, exact numbers.
