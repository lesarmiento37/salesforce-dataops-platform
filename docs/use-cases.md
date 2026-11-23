# Use cases

The scenarios below describe how a DataOps team can operate the integration. Object names and data are illustrative; production field selection and retention require approval from the Salesforce and data-governance owners.

## 1. Onboard a Salesforce object

**Need:** Make a new standard or custom object available to analytics without hand-building a one-off pipeline.

**Approach:** Add an object entry to the flow catalog, select an appropriate incremental field, define an explicit field allowlist, configure a destination prefix, and deploy the on-demand AppFlow definition through infrastructure as code. Validate the connector permissions and a small sandbox extraction before enabling downstream consumers.

**Acceptance checks:**
- The API object and selected fields are available to the integration user.
- The flow writes only to the intended sandbox destination.
- The first-load strategy is approved for the estimated volume.
- The Glue table has expected columns and types, and access grants are reviewed.
- A second run selects only records changed since the committed boundary.

## 2. Routine incremental ingestion

**Need:** Keep analytics current without re-extracting the full object on every run.

**Approach:** Airflow reads the flow's last successful watermark, applies a time filter to the AppFlow run, starts and monitors the flow, validates landing/catalog tasks, then commits the next watermark. Mutable objects generally use a last-modified timestamp; append-only objects use a creation timestamp when supported.

**Operational outcome:** A failure before commit leaves the prior checkpoint intact, so the interval can be retried safely. If the source has timestamp delays or coarse precision, configure an overlap and make the downstream deduplication key explicit.

## 3. Repair one failed object

**Need:** Recover a failed Case or Account extraction without rerunning unrelated data domains.

**Approach:** Trigger the single-flow entry point with a validated flow identifier. Reuse the last committed watermark or provide a reviewed custom lower bound. Confirm that the prior run did not commit a new checkpoint before retrying; inspect AppFlow execution status and S3 run prefix before initiating a replay.

**Success criteria:** The failed object is reconciled, duplicate handling is deterministic, and unrelated flows remain untouched.

## 4. Historical backfill for a high-volume object

**Need:** Load historical data while controlling run duration and avoiding API or platform limits.

**Approach:** Divide the date range into non-overlapping windows `[start, end)`. Run each window with checkpoint updates disabled, track completed windows, and reconcile row/file counts or another agreed completeness signal. Commit the final watermark once all windows are complete. Preserve failed-window details for targeted retries.

**Guardrails:** Use a sandbox or approved maintenance window, confirm source API limits and target capacity, apply concurrency limits, and never advance the steady-state watermark after only a partial campaign.

## 5. Coordinated domain refresh and downstream models

**Need:** Refresh several independent Salesforce domains and then publish analytics models.

**Approach:** A master Airflow DAG triggers object-group DAGs in parallel and waits for their outcomes. After all required groups succeed, it reconciles/commits group state and triggers downstream transformations. A failed group blocks dependent models while successful independent groups remain inspectable.

**Success criteria:** Dependencies are visible in Airflow, alerts identify the failed domain, and downstream jobs cannot run on an incomplete required refresh.

## 6. Salesforce export files arrive through a separate storage integration

**Need:** Process a Salesforce export delivered as files to managed object storage rather than connected directly through AppFlow.

**Approach:** Use a separate Airflow Kubernetes task to retrieve/transform the export and sync the result to S3. Treat this as a file-ingestion path with its own source credentials, checksum/completeness checks, schema contract, and lineage. Do not mix its watermark or failure state with direct AppFlow flows.

## 7. Data-access review

**Need:** Share curated data with analysts while restricting raw or sensitive fields.

**Approach:** Keep extraction permissions separate from query permissions. Register the target dataset in Glue, grant Lake Formation access to approved principals, and verify access using both an authorized and unauthorized test role. Review field-level minimization and retention with data owners before broadening access.

## Suggested portfolio walkthrough

1. Explain the separation between CDK flow definitions and Airflow run-time control.
2. Show how a run resolves its timestamp filter and why the watermark is written only after success.
3. Demonstrate a single-object retry and a bounded backfill with checkpoint writes disabled.
4. Trace an object group through S3, Glue, governance grants, and downstream model dependencies.
5. Close with the operational controls: scoped credentials, alerts, reconciliation, and no production data in the public example.
