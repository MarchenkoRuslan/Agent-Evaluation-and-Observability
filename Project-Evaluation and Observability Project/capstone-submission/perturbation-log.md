# Perturbation Log

Predictions below were recorded before the corresponding experiment runs.
Commands run from the final solution for the named system, using its own `.venv`.
Evidence paths are relative to this submission folder.

## System 1 - Validated, Routed Pipeline

- **Planned change:** Copy `POL-2025-001.txt` and remove its stated premium amount, leaving the remaining document unchanged. Do not modify the bundled original.
- **Baseline command:** `.venv\Scripts\policy-extractor.exe extract data/policies/POL-2025-001.txt --policy-id POL-2025-001`
- **Perturbed command:** `.venv\Scripts\policy-extractor.exe extract <submission>/01-policy-pipeline/inputs/POL-2025-001-missing-premium.txt --policy-id POL-EXP-MISSING-PREMIUM`
- **Prediction:** The extractor returns a null required premium; validation classifies this as `missing_source` and escalates without a corrective retry. Do not infer the number of live network requests from the CLI output.
- **Independent call-count check:** Run `tests/test_us01_retry.py` unchanged. Its `test_ac_01_04_missing_source_halts_immediately` asserts exactly one recorded-client call. This is separate evidence from the live experiment.
- **Observed live result:** With the explicitly supplied course key, [perturbation-baseline.txt](01-policy-pipeline/perturbation-baseline.txt) returned `kind: extraction`, `premium_amount: 1847.62`, `retry_count: 0`, and exit 0. [perturbation-run.txt](01-policy-pipeline/perturbation-run.txt) returned `kind: escalation`, `category: missing_source`, `field: premium_amount`, `detected_pattern: premium_amount_absent`, and exit 1. Its reason states that the field returned null, the source lacks the information, and retry is futile. The modified run contains one logged HTTP 200 Messages request and no corrective retry. That is an observed request-log count, not a packet trace.
- **Supplementary offline checks actually run:** `.venv\Scripts\python.exe -m pytest tests/test_us01_retry.py -v` ([retry-tests.txt](01-policy-pipeline/retry-tests.txt)), plus unchanged success/missing-source/routing tests under `--trace` ([offline-runtime-records.txt](01-policy-pipeline/offline-runtime-records.txt)). Debugger commands only ran each test to return and read existing locals; no test inputs or code were edited.
- **Observed fallback output:** The well-formed response produced `PolicyExtraction(policy_id='POL-2025-001', premium_amount=1847.62, ...)`, `retry_count=0`, and one recorded-client call. Selecting the null-endorsements case produced `RetryFutileEscalation(policy_id='POL-2025-009', field='endorsements', category='missing_source', detected_pattern='endorsements_absent', ...)`, also with exactly one recorded-client call. All three debugger-observed tests passed. Absent required information becomes an escalation, not a guessed value or a second extraction attempt.
- **Difference and scope:** The live baseline and modified run use the same source document except for the deleted premium line; both copied inputs are in `01-policy-pipeline/inputs/`. Unlike the successful baseline, the modified document escalates instead of inventing 1847.62 or retrying for absent information. The prediction is supported. The separate offline tests compare different controlled fixtures and only supplement this live experiment; their call-count assertion is retained in [retry-contract.txt](01-policy-pipeline/retry-contract.txt). Initial access failures used a different, inherited credential and are retained under `01-policy-pipeline/initial-credential-attempts/`; they do not describe the supplied course key.

## System 2 - Schema-Enforced Two-Pass Extraction

- **Change selected:** Switch the CLI input from bundled `appraisal_informal_sqft.txt` to bundled `income_sum_mismatch.txt`, using the course's offline perturbation starter.
- **Baseline command:** `.venv\Scripts\mortgage-extract.exe fixtures/documents/appraisal_informal_sqft.txt --mode replay`
- **Perturbed command:** `.venv\Scripts\mortgage-extract.exe fixtures/documents/income_sum_mismatch.txt --mode replay`
- **Prediction:** The appraisal validates cleanly and normalizes its living area. The mismatch fixture still produces structured extraction, but arithmetic validation reports a discrepancy and exits with code 1.
- **Scope:** This compares two different prerecorded fixture documents, not a freshly edited same-document pair. Arbitrary edits change the content-hashed replay request and would test a cache miss rather than arithmetic validation.
- **Observed output:** [extract-run.txt](02-mortgage-extraction/extract-run.txt) records `type=appraisal`, `gross_living_area_sqft: 2400`, `consistent: true`, and exit 0. [discrepancy-run.txt](02-mortgage-extraction/discrepancy-run.txt) records `type=income_verification`, `consistent: false`, `field: total_monthly_income`, `calculated: 9642.17`, `stated: 10892.17`, `delta: -1250.0`, and exit 1.
- **Baseline comparison:** Both runs complete structured extraction; only the second triggers the arithmetic safeguard. This supports the prediction. It is a comparison of bundled replay fixtures, not a same-document live mutation. Both source documents are retained under `02-mortgage-extraction/inputs/`.

## System 3 - Multi-Source Synthesis

- **Configuration change:** Add `--simulate-timeout` to the otherwise identical offline investigation.
- **Baseline command:** `.venv\Scripts\supply-chain-investigate.exe meridian --offline`
- **Perturbed command:** `.venv\Scripts\supply-chain-investigate.exe meridian --offline --simulate-timeout`
- **Prediction:** The run completes despite the simulated logistics failure, labels the unavailable source, and annotates missing coverage as Incomplete instead of treating it as evidence that no risk exists. Partial results may survive.
- **Observed output:** [timeout-briefing.md](03-supply-chain/timeout-briefing.md) reports `Sources unavailable: logistics unavailable (timeout)` and places `late_shipment_count` under Incomplete with `missing source: timeout reading logistics`. [timeout-run.txt](03-supply-chain/timeout-run.txt) exits 0. Baseline [briefing.md](03-supply-chain/briefing.md) reports `11.0 shipments` from logistics as of `2026-04-05`, and [investigation-run.txt](03-supply-chain/investigation-run.txt) also exits 0.
- **Baseline comparison:** The baseline retains on-time delivery at 95.0% (audit, 2026-04-10) versus 78.0% (logistics, 2026-04-05), classed Contested. After timeout, failed-reader claims are excluded, audit's 95.0% is `single source only`, and Contested is empty. The late-shipment count becomes an explicit unknown, not zero. Audit, quality, and news findings remain and the investigation finishes. The prediction is supported, with a clarification: reader partial results are not promoted into successful-source synthesis. Losing the conflict is evidence loss, not reconciliation.