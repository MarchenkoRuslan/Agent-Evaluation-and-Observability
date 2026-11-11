# Submission Status

Run date: 2026-11-11. All three systems were reproduced and run. The policy pipeline and copied-input experiment completed live with the explicitly supplied course key; mortgage used replay and supply chain used offline mode. The reflection and perturbation log are grounded in the saved results.

## Reproduce

| System | Full suite | mypy | ruff |
| --- | --- | --- | --- |
| Policy | 45 passed, 3 skipped | Clean, 11 source files | All checks passed |
| Mortgage | 25 passed | Clean, 11 source files | All checks passed |
| Supply chain | 34 passed | Clean, 8 source files | All checks passed |

Evidence: [policy tests](01-policy-pipeline/tests.txt), [mortgage tests](02-mortgage-extraction/tests.txt), and [supply-chain tests](03-supply-chain/tests.txt), plus [policy static checks](01-policy-pipeline/static-checks.txt), [mortgage static checks](02-mortgage-extraction/static-checks.txt), and [supply-chain static checks](03-supply-chain/static-checks.txt). This is **104 passed, 3 skipped** across the suites. The policy skips are optional live tests run with the key absent. Mortgage's actual collected count is 25, not a count inferred from course prose.

## Run and Perturb

- Policy calibration completed: [calibration-report.txt](01-policy-pipeline/calibration-report.txt), including umbrella/exclusions confidence 0.93 versus accuracy 0.00 and overall Brier 0.291.
- Policy live [pipeline-run.txt](01-policy-pipeline/pipeline-run.txt) exits 0 and generates [routing_decisions.json](01-policy-pipeline/routing_decisions.json): **1 auto_approve, 7 human_review, 1 spot_check**. Nine decisions plus one extraction escalation account for all ten bundled documents. `POL-2025-009` escalates for `endorsements_absent`; the pattern summary also captures a consistency failure for `POL-2025-010`.
- Live `POL-2025-001` routes to human review solely on reviewer disagreement about coverage limit despite confidence 1.0 for that field; its below-threshold and integration-failure lists are empty. This is the reflection's primary routing example.
- The live [original extraction](01-policy-pipeline/perturbation-baseline.txt) returns premium 1847.62 and exit 0. The [premium-deleted copy](01-policy-pipeline/perturbation-run.txt) returns `missing_source` / `premium_amount_absent`, one logged Messages request, and expected exit 1. This is a genuine same-document controlled perturbation.
- Supplementary [routing-tests.txt](01-policy-pipeline/routing-tests.txt) has 9 passing tests; [retry-tests.txt](01-policy-pipeline/retry-tests.txt) has 13 passed and 1 skipped. [offline-runtime-records.txt](01-policy-pipeline/offline-runtime-records.txt) remains separately labeled test evidence, not a replacement for live routing.
- Mortgage's [appraisal](02-mortgage-extraction/extract-run.txt) and [missing-bonus](02-mortgage-extraction/missing-field-run.txt) replay runs exit 0. The [mismatch](02-mortgage-extraction/discrepancy-run.txt) exits 1 as expected, reporting calculated 9642.17 versus stated 10892.17, delta -1250.0. All classification types are captured with `--verbose`.
- Supply-chain [baseline run](03-supply-chain/investigation-run.txt) and [timeout run](03-supply-chain/timeout-run.txt) both exit 0. Their separate [briefing](03-supply-chain/briefing.md) and [timeout briefing](03-supply-chain/timeout-briefing.md) preserve conflicts and explicit missing coverage. Native-output decoding was set to UTF-8 for the final captures.
- [reflection-brief.md](reflection-brief.md) answers all prompts using these artifacts. [perturbation-log.md](perturbation-log.md) includes pre-run predictions, commands, observed outcomes, and honest baseline-comparison limitations.

## Execution Correction

Initial requests accidentally used a different credential inherited from the terminal environment. Its HTTP 400 failures were incorrectly attributed to the learner-supplied key. After explicitly using the supplied course key, the API returned HTTP 200 and all live work completed. The old [pipeline failure](01-policy-pipeline/initial-credential-attempts/pipeline-run.txt) and other initial captures are retained separately; they are historical setup evidence, not current live results. No API-access blocker remains.

The learner's name was not provided and remains marked `Not supplied` in the reflection. The submission uses text evidence, which the instructions allow, rather than fabricated screenshots.

## Reading the Pack

Start with `environment.txt`, then the final reflection and perturbation log. Each run capture records its command and exit code. The optional [policy on-call note](01-policy-pipeline/on-call-note.md) explains triage using the passing reference contracts.

The live policy command is sequential. Batch scheduling/resubmission behavior is demonstrated by the full policy suite, not by claiming that the live `pipeline` command uses Message Batches. Calibration uses the supplied synthetic labels; it is not an accuracy estimate of the live document run.

No application source, tests, or recorded API fixtures have been edited. Python environments and generated dependency metadata are setup artifacts, not part of this submission. Screenshots are not fabricated; this pack uses the text captures allowed by the instructions.