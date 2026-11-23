# Security and governance

Security controls belong in the deployment and operational process, not only in documentation. This public repository intentionally contains no real credentials, production account identifiers, internal bucket names, customer records, or principal ARNs.

## Identity and credentials

- Use a dedicated Salesforce integration identity with only the object and field permissions required for approved flows.
- Store OAuth client secrets and refresh tokens in an approved secrets manager or managed Airflow connection; never commit them or print them to task logs.
- Use workload identity/assumed roles for Airflow-to-AWS access instead of long-lived access keys.
- Separate the AppFlow execution role from Airflow orchestration, cataloging, and analyst/query roles.
- Review trust policies and resource permissions for least privilege, including access limited to the intended S3 prefixes and required AppFlow actions.

## Data protection

- Maintain an explicit field allowlist for each object; default to excluding unneeded or sensitive fields.
- Confirm data classification, legal basis, retention, deletion, and regional requirements with the data owner before extraction.
- Encrypt S3 objects and metadata at rest with an approved key policy; enforce TLS for service connections.
- Use S3 Block Public Access, bucket policies, and lifecycle policies appropriate to the data classification.
- Keep raw data private. Use synthetic data for screenshots, examples, and demonstrations.
- Define safe replay and deletion procedures for data-subject requests or source corrections.

## Governance and discoverability

- Register datasets in Glue with an accountable owner, source, refresh cadence, schema description, and sensitivity classification.
- Grant analyst access through Lake Formation or the platform's approved governance layer; avoid broad bucket-level grants as a substitute.
- Review table/column access and validate both allowed and denied principals as part of onboarding.
- Version schema and field-selection changes; communicate breaking changes to downstream model owners.

## Operational controls

- Restrict production triggers and full-load/backfill actions to authorized operators.
- Require bounded time windows and explicit checkpoint handling for historical backfills.
- Audit flow changes, role grants, trigger requests, and watermark commits.
- Redact secrets and personal data from logs; retain execution IDs and sanitized error codes for traceability.
- Set concurrency limits, timeouts, retries with backoff, alert routing, and runbook ownership.
- Periodically review connector health, IAM access, key policies, data retention, and AppFlow/API quotas.

## Public portfolio review checklist

Before publishing any addition, verify it contains none of the following:

- OAuth tokens, passwords, private keys, or environment files.
- Real AWS account IDs, internal bucket names, role/user ARNs, endpoints, or connector-profile identifiers.
- Salesforce exports, customer data, screenshots containing records, or identifiable email addresses.
- Internal run identifiers, hostnames, incident links, or proprietary operational details.

Use fictional identifiers such as `example-data-lake` and `Demo__Account`. A secret scanner is useful but does not replace human review, especially for data and infrastructure metadata.
