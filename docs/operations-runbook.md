# Operations runbook

This runbook describes safe operator actions for an AppFlow-based Salesforce ingestion. Adapt names and commands to the authorized deployment; this repository contains no live environment identifiers or executable deployment credentials.

## Before triggering a run

- Confirm the target environment and Salesforce connector profile (sandbox vs production).
- Confirm the flow identifier is in the approved catalog and its filter field is valid for the object.
- Check whether another run for the same flow is active; avoid concurrent updates to the same AppFlow definition or watermark.
- Choose the intended mode: incremental, full load, or bounded backfill.
- For a full load, confirm expected source volume, AppFlow/AWS quotas, S3 capacity, retention, and downstream impact.
- For a backfill, confirm the date windows do not overlap, and that checkpoint updates are disabled until the campaign is reconciled.
- Verify Airflow connections/role access and relevant AWS service health without exposing secret values in logs.

## Routine incremental run

1. Trigger the approved flow or flow-group DAG.
2. Verify Airflow resolved the expected watermark and filter field.
3. Verify AppFlow reached a successful terminal status.
4. Confirm new files exist under the expected execution-scoped S3 prefix.
5. Confirm the catalog/crawler step completed and any required data checks passed.
6. Confirm the watermark was updated only after the successful run.
7. Confirm downstream transformations ran only when their dependencies succeeded.

## Full load

Use a forced full-load mode only with an explicit change/operational approval. Consider a separate destination prefix or controlled replacement strategy so a full extraction does not silently overwrite or mix with current data. Validate expected row/file counts and downstream duplicate behavior before publishing.

## Single-flow retry

1. Identify the failed flow, execution ID, and failing Airflow task.
2. Inspect the AppFlow execution state, AWS API error class, and target S3 prefix.
3. Determine whether extraction succeeded but cataloging or a later step failed; do not rerun source extraction unnecessarily if the landed data can be safely replayed.
4. Confirm the watermark is unchanged for an incomplete run.
5. Retry only the failed flow/task with the same or corrected time boundary.
6. Validate data and commit the watermark after all required steps succeed.

## Bounded historical backfill

For each flow, use a table or issue to track these fields:

| Field | Example |
| --- | --- |
| Flow | `Demo__Account` |
| Lower bound (inclusive) | `2024-01-01T00:00:00Z` |
| Upper bound (exclusive) | `2024-04-01T00:00:00Z` |
| Destination execution prefix | `run_id=<unique-id>` |
| Checkpoint writes | Disabled for each chunk |
| Status | Planned / running / reconciled |

1. Divide the total range into bounded, adjacent windows.
2. Run each chunk with a unique execution ID and checkpoint writes disabled.
3. Record run status and expected/observed completeness signals.
4. Retry only failed chunks; preserve the same boundaries and idempotency strategy.
5. Reconcile all chunks and confirm there are no gaps or unintended overlaps.
6. Commit the final watermark once, then resume normal incremental ingestion.

## Failure response

| Symptom | First checks | Recovery direction |
| --- | --- | --- |
| AppFlow start fails | Flow name, connector profile health, role permissions, API error details | Correct configuration or permissions; retry only after the cause is resolved. |
| Flow remains running | AppFlow execution status, timeout, source volume, service health | Allow bounded polling; terminate only under the approved procedure, then reconcile any partial landing. |
| Flow succeeds but no data appears | S3 destination, prefix, filters, source object permissions, empty-result possibility | Distinguish a valid zero-row interval from a destination or filter defect before retrying. |
| Glue catalog step fails | Crawler/catalog permissions, malformed or changed schema, S3 prefix | Repair cataloging from the landed files where possible; avoid duplicate source extraction. |
| Watermark did not advance | Upstream success and checkpoint task logs | Keep the old boundary until ingestion and required checks are complete; retry the commit task only if its inputs are verified. |
| Unexpected duplicate records | Overlap/lookback, object key, replayed intervals, downstream merge logic | Reconcile source keys and time ranges; fix deduplication or watermark policy before proceeding. |

## Observability and escalation

Correlate the Airflow DAG run ID, flow name, AppFlow execution ID, S3 run prefix, and watermark before escalating. Alerts should include the environment, failed task, flow identifier, and sanitized error summary—never access tokens, raw Salesforce payloads, or sensitive query parameters. Escalate authorization failures to the platform/security owner, schema or field changes to the Salesforce/data owner, and service/quota issues to the cloud platform owner.

## Post-run checklist

- [ ] AppFlow execution completed successfully.
- [ ] Expected S3 files and execution metadata are present.
- [ ] Catalog and required quality checks succeeded.
- [ ] Watermark reflects the last successful interval, not a failed/partial one.
- [ ] Downstream dependencies and alerts behaved as expected.
- [ ] Backfill windows were reconciled before checkpoint finalization.
