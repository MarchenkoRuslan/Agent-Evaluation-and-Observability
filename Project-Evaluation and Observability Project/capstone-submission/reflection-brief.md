# Reflection Brief - Evaluation and Observability Capstone

**Name:** Not supplied
**Date:** 2026-11-11

## 0. Environment

| Field | Value |
| --- | --- |
| OS & version | Windows, Microsoft Windows NT 10.0.26200.0 |
| Python version | 3.12.10, separate `.venv` for each final solution |
| Date run | 2026-11-11, UTC+02:00 |
| Live systems | Policy pipeline and copied-document experiment completed live; mortgage replay and supply-chain offline completed. |

[environment.txt](environment.txt) records setup. The full suites report policy **45 passed, 3 skipped**, mortgage **25 passed**, and supply chain **34 passed**. Each system's `static-checks.txt` shows mypy and ruff exiting 0. The policy skips are optional live tests run with the key absent.

## 1. Validated, Routed Pipeline

| Evidence | Value |
| --- | --- |
| Passing test count | 45 passed, 3 skipped: [tests.txt](01-policy-pipeline/tests.txt) |
| Routing output file | [routing_decisions.json](01-policy-pipeline/routing_decisions.json), produced by [pipeline-run.txt](01-policy-pipeline/pipeline-run.txt) |
| auto_approve / human_review / spot_check counts | 1 / 7 / 1; 9 decisions plus 1 extraction escalation account for all 10 input documents. |
| Supplementary offline evidence | [routing-tests.txt](01-policy-pipeline/routing-tests.txt): 9 passed; [offline-runtime-records.txt](01-policy-pipeline/offline-runtime-records.txt): separately labeled test objects. |

**1a. Retry boundary.** I removed only the stated premium line from a copy of `POL-2025-001`. The original live [extraction](01-policy-pipeline/perturbation-baseline.txt) returned `premium_amount: 1847.62`, `retry_count: 0`, and exit 0. The modified live [run](01-policy-pipeline/perturbation-run.txt) returned this escalation and exit 1:

```json
{
  "category": "missing_source",
  "detected_pattern": "premium_amount_absent",
  "field": "premium_amount",
  "kind": "escalation",
  "policy_id": "POL-EXP-MISSING-PREMIUM",
  "reason": "Field 'premium_amount' returned null \u2014 the source document does not contain this information. Retry is futile; escalate to human review."
}
```

The modified-run capture contains exactly **one logged Messages API request**, returning HTTP 200, followed immediately by validation and escalation; no corrective retry is logged. Independently, the passing recorded-client test asserts `client.call_count == 1` ([retry-contract.txt](01-policy-pipeline/retry-contract.txt), [retry-tests.txt](01-policy-pipeline/retry-tests.txt)). Repeating extraction cannot supply the deleted source fact; it wastes resources and invites fabrication. The useful next action is obtaining the missing premium, not another model guess.

**1b. Reading the router.** My live `POL-2025-001` record in [routing_decisions.json](01-policy-pipeline/routing_decisions.json) has `decision: human_review`, `reason: reviewer_disagreement=['coverage_limit']`, `fields_below_threshold: []`, and `integration_failures: []`. The coverage-limit confidence is **1.0**, and every field is at or above 0.90. Independent reviewer disagreement alone forced human review. A confidence-only gate would have cleared this record for auto-approval eligibility, not mandatory review; subsequent spot sampling is separate. This does not prove which model is correct, but it exposes the dispute. The [pipeline capture](01-policy-pipeline/pipeline-run.txt) reports 9 decisions and 1 escalation (`POL-2025-009`, `endorsements_absent`), plus a consistency failure for `POL-2025-010` in the pattern summary. The full [suite](01-policy-pipeline/tests.txt) verifies batch SLA, resubmission, and stratified sampling; the live `pipeline` command itself is sequential.

**1c. Where the aggregate lies.** My [calibration-report.txt](01-policy-pipeline/calibration-report.txt) prints `umbrella  exclusions  n=2 conf=0.93 acc=0.00 brier=0.865`, but `OVERALL brier=0.291`. Auto-premium has `n=3 conf=0.95 acc=1.00 brier=0.003`. Pooling dilutes the umbrella-exclusions failure with correct labels elsewhere. Slicing identifies where to investigate and collect labels. These are the supplied six synthetic labels, not live extraction accuracy; two examples do not establish population calibration.

## 2. Schema-Enforced Two-Pass Extraction

| Evidence | Value |
| --- | --- |
| Passing test count | 25: [tests.txt](02-mortgage-extraction/tests.txt) |
| Documents run | All three bundled fixtures, `--mode replay --verbose` |
| Classified types | `appraisal` in [extract-run.txt](02-mortgage-extraction/extract-run.txt); `income_verification` for both income runs |

**2a. Two guarantees.** [discrepancy-run.txt](02-mortgage-extraction/discrepancy-run.txt) prints this validation result and exits 1:

```json
{
  "consistent": false,
  "discrepancies": [
    {
      "field": "total_monthly_income",
      "calculated": 9642.17,
      "stated": 10892.17,
      "delta": -1250.0
    }
  ]
}
```

The extracted monthly components are 5416.67 + 1250.0 + 2140.0 + 385.5 + 450.0 = 9642.17; the source states 10892.17. Structured tool output and typed parsing constrain machine-readable shape, not arithmetic agreement. The validator independently computes the sum and preserves the stated total rather than silently correcting it. Structure alone cannot catch this contradiction. The arithmetic validator does not validate JSON structure and cannot detect consistently wrong extracted numbers whose sum matches. Neither proves source truth.

My follow-up [controlled tolerance experiment](02-mortgage-extraction/tolerance-comparison.txt) illustrates an additional validator limitation: the same extraction is flagged at the original $1 tolerance but marked consistent when the tolerance is deliberately raised to $1250, exactly equal to the discrepancy. The changed-setting result is in [tolerance-perturbed-run.txt](02-mortgage-extraction/tolerance-perturbed-run.txt); this is an unsafe experimental setting, not a correction to the original project or source data.

**2b. Refusing to fabricate.** [missing-field-run.txt](02-mortgage-extraction/missing-field-run.txt) prints `"bonus_monthly": null` and `"bonus_ytd": null`. The corresponding [input](02-mortgage-extraction/inputs/income_missing_bonus.txt) has no bonus line. The nullable field `bonus_monthly: float | None = None` is retained in [nullable-contract.txt](02-mortgage-extraction/nullable-contract.txt), and the extraction tool instructs null for unstated information. Null preserves absence instead of asserting an unsupported zero or guessed bonus. The schema permits null; this replay shows that option being used. Its `consistent: true` does not prove completeness: `stated_monthly_total` is also null, so there is no stated total to compare.

**2c. Normalization.** My [source appraisal](02-mortgage-extraction/inputs/appraisal_informal_sqft.txt) says `Gross Living Area:   approximately 2,400 sq ft (above-grade finished)`. [extract-run.txt](02-mortgage-extraction/extract-run.txt) returns `"gross_living_area_sqft": 2400`, `type=appraisal`, and `consistent: true`. Normalizing at extraction gives downstream consumers one integer/unit contract instead of repeated comma/unit parsing. Keeping the source matters: conversion to an integer does not make an approximate measurement exact. The consistency validator checks income arithmetic, not appraisal accuracy.

## 3. Multi-Source Synthesis

| Evidence | Value |
| --- | --- |
| Passing test count | 34: [tests.txt](03-supply-chain/tests.txt) |
| Briefing file | [briefing.md](03-supply-chain/briefing.md) |
| Conflict section | Contested: `on_time_delivery_rate` |

**3a. Annotate, don't arbitrate.** My [briefing.md](03-supply-chain/briefing.md) retains `95.0 percent` from `supplier_audit` as of `2026-04-10` and `78.0 percent` from `logistics` as of `2026-04-05`. It labels the metric `2 sources, conflicting` and escalates the high-impact conflict. A reader can inspect periods, populations, and source reliability. Averaging to 86.5% invents a measure neither source reported and hides disagreement. The [investigation capture](03-supply-chain/investigation-run.txt) records successful completion and the logistics table.

**3b. Source goes dark.** My [timeout-briefing.md](03-supply-chain/timeout-briefing.md) contains:

```text
> Sources unavailable: logistics unavailable (timeout)
### late_shipment_count  _[missing source: timeout reading logistics]_
- missing source: timeout reading logistics
```

The baseline instead reports `11.0 shipments` from logistics as of `2026-04-05`. The [timeout run](03-supply-chain/timeout-run.txt) exits 0: the coordinator isolates the failed reader and synthesizes successful-source claims. A timeout is a coverage gap, not evidence of zero late shipments. Separately, `production_capacity_utilization` says `no source reported this metric` in both runs. These are different reasons for unavailable knowledge. An important caveat: on-time delivery becomes `single source only` at 95.0% and Contested becomes `_none_`; losing contradictory evidence does not establish that the audit is correct. Failed-reader claims are excluded from synthesis even if partial results survive in failure context.

**3c. Dates as a guardrail.** The same supplier's two dated on-time claims are `95.0 percent` (`supplier_audit`, `2026-04-10`) and `78.0 percent` (`logistics`, `2026-04-05`) in [briefing.md](03-supply-chain/briefing.md). Preserving dates stops me silently treating them as simultaneous measurements and prompts a reporting-window check. The implementation still flags the numeric disagreement as Contested. Dates enable interpretation; they do not automatically align periods or prove that this is merely change over time.

## 4. Synthesis

**4a. One principle.** Mortgage's `calculated: 9642.17`, `stated: 10892.17`, `delta: -1250.0` in [discrepancy-run.txt](02-mortgage-extraction/discrepancy-run.txt) most clearly caught something a trusting design would accept. A parseable extraction was insufficient; recomputing a redundant quantity exposed the contradiction. Across all three systems I would preserve uncertainty and keep an independently checkable decision boundary: null/escalation for absent evidence, validation/routing for disputed output, and attributed conflicts or coverage gaps rather than silent reconciliation.

**4b. Confidence is not correctness.** Policy makes this explicit in my live `POL-2025-001`: coverage-limit confidence is **1.0**, but reviewer disagreement on that field still routes it to `human_review` ([routing_decisions.json](01-policy-pipeline/routing_decisions.json)). Separately, umbrella exclusions are 0% accurate despite 0.93 confidence in the supplied labels ([calibration-report.txt](01-policy-pipeline/calibration-report.txt)). Disagreement is a reason to investigate, not proof that the reviewer is always right. Correctness labels must remain independent of both models.

**4c. Apply it.** For a proposed bug-report/ticket workflow, I would extract affected component, reproduction steps, severity, and a source link. I would start with independent review and deterministic routing before automated assignment or closure: my live `POL-2025-001` in [routing_decisions.json](01-policy-pipeline/routing_decisions.json) shows why confidence alone is insufficient. I would log input ID/version, extracted fields, nulls, reviewer disagreements, validator failures, routing reason, retry/call counts, and latency. I would stratify spot checks and labeled accuracy by component and field, following the failure hidden by the aggregate in [calibration-report.txt](01-policy-pipeline/calibration-report.txt). I would separately monitor source coverage: the disappearing conflict in [timeout-briefing.md](03-supply-chain/timeout-briefing.md) shows that fewer disagreements can mean less evidence, not improvement. These are proposed controls, not instrumentation added here.

## Execution Notes

Initial requests used a different, inherited credential and failed. After explicitly switching to the learner-supplied course key, all required live policy runs completed. Initial failures remain separately labeled for transparency; they are not evidence that the supplied key was suspended. See [submission-status.md](submission-status.md). No application code was changed. The learner's name was not supplied.