# Policy Pipeline: On-Call Note

## First Checks

Open the run capture before interpreting routing counts. Confirm the native exit code and that the routing JSON belongs to this run. A missing file, an API failure, or fewer decisions without an accompanying escalation count is not evidence that the remaining documents were approved. Account for all input documents using decisions plus extraction escalations.

For `human_review`, inspect `fields_below_threshold`, `reviewer_disagreements`, and `integration_failures` on the individual record. Any one can be sufficient. The confidence threshold is 0.90; high self-confidence does not override reviewer disagreement or an integration failure. The passing `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` in [tests.txt](tests.txt) verifies that boundary independently of live model behavior.

## Retry or Escalate?

A negative numeric value or an inconsistent component sum can justify error-feedback retry. A null required field is classified `missing_source`: inspect the source document and request missing information rather than paying for another guess. In [tests.txt](tests.txt), `test_ac_01_04_missing_source_halts_immediately` passes; its one-call assertion is a recorded-client guarantee, not a measured live HTTP request count. SDK transport retries are a different layer from extraction retries.

Batch errors and absent source information need different handling. The passing batch tests in [tests.txt](tests.txt) cover error resubmission, immediate missing-source escalation, SLA scheduling, and the sample-rate gate. The `pipeline` CLI itself runs extraction and review sequentially; do not describe its output as a Message Batches submission.

## Which Signal Can Mislead?

Model confidence is a prediction, not a correctness label. Inspect calibration by policy type and field, not just an overall score. Keep independently checked labels separate from the model's ratings. Retain the independent review and deterministic integration checks even when confidence looks reassuring.

`spot_check` is intentional drift monitoring, not a validation failure. With the same already-computed decisions and seed, sampling is reproducible. A fixed seed does not make live model answers deterministic.

## Healthy Reference State

This run's offline policy suite reports **45 passed, 3 skipped** in [tests.txt](tests.txt). Both checks in [static-checks.txt](static-checks.txt) exit 0: mypy reports no issues in 11 source files and ruff reports `All checks passed!`. The skipped cases are optional live tests. These results establish the reference contracts; they do not prove that a later API request succeeded. Never copy credentials or an environment-variable dump into an incident report.