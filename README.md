# Azure Secret Expiry Monitor

<img width="1536" height="1024" alt="f3a1643f-2dd2-4f00-b183-ee2be6cf6420" src="https://github.com/user-attachments/assets/734b5a3b-077e-4396-a0e8-68499a716847" />


A GitHub Actions-based monitoring solution for Microsoft Entra ID application credentials. It scans application registrations through Microsoft Graph, monitors client secrets and certificates, creates or updates GitHub Issues for credentials requiring attention, manages lifecycle labels, closes duplicate/resolved Issues, and publishes a detailed monitoring report in the GitHub Actions run summary.

The design is intended to scale from a handful of applications to hundreds of application registrations without requiring one workflow per SPN.

## Features

- Scans Microsoft Entra application registrations through Microsoft Graph.
- Monitors client secrets and certificates.
- Handles Microsoft Graph pagination.
- Deduplicates credentials using Application ID, Key ID, and credential type.
- Calculates whole-number days remaining until expiry.
- Classifies credentials as `HEALTHY`, `UPCOMING`, `EXPIRING_SOON`, or `EXPIRED`.
- Creates GitHub Issues for credentials requiring attention.
- Updates the same Issue on subsequent runs instead of creating daily duplicates.
- Uses a hidden stable marker based on Application ID and Key ID for Issue matching.
- Detects and closes duplicate Issues for the same credential.
- Automatically closes managed Issues when the credential is no longer in the active alert set.
- Automatically manages credential and status labels.
- Publishes a detailed inventory and monitoring report to the Actions Summary.
- Uses GitHub Actions OIDC to authenticate to Azure without a long-lived Azure client secret.
- Uses workflow concurrency to prevent overlapping runs.
- Supports manual and scheduled execution.

## Architecture

```text
                         GitHub Actions
                              |
                              v
                 +---------------------------+
                 | Scan Entra Applications   |
                 | Microsoft Graph API        |
                 +-------------+-------------+
                               |
                               | scan results
                               v
                 +---------------------------+
                 | Credential Classification  |
                 |                            |
                 | Healthy                   |
                 | Upcoming                  |
                 | Expiring Soon             |
                 | Expired                   |
                 +-------------+-------------+
                               |
                               v
                 +---------------------------+
                 | Issue Management           |
                 | Create / Update            |
                 | Deduplicate               |
                 | Label                     |
                 | Close resolved             |
                 +-------------+-------------+
                               |
                               v
                 +---------------------------+
                 | Monitoring Summary         |
                 | Credential inventory       |
                 | Run details                |
                 +---------------------------+
```

### Workflow stages

The workflow is deliberately split into three visible jobs:

```text
Scan Entra application credentials
              |
              v
Create or update credential issues
              |
              v
Publish monitoring summary
```

This makes each run easier to troubleshoot than a single large job.

## Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── secret-expiry-monitor.yml
└── README.md
```

The main implementation is `.github/workflows/secret-expiry-monitor.yml`.

## How It Works

### 1. Authenticate to Azure

GitHub Actions authenticates to Azure using OIDC:

```yaml
- name: Login to Azure with OIDC
  uses: azure/login@v3
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    allow-no-subscriptions: true
```

No Azure client secret is stored in GitHub for the monitoring identity.

### 2. Obtain a Microsoft Graph token

The workflow uses Azure CLI:

```bash
az account get-access-token \
  --resource-type ms-graph \
  --query accessToken \
  -o tsv
```

It then queries:

```text
https://graph.microsoft.com/v1.0/applications
```

for:

```text
id
appId
displayName
passwordCredentials
keyCredentials
```

### 3. Discover credentials

For each application, the workflow extracts:

- Application display name
- Application ID
- Client-secret metadata
- Certificate metadata
- Credential display name
- Key ID
- Expiry date

The actual secret value is never retrieved or written to the report.

### 4. Classify credentials

Each credential is compared with the current UTC time.

```text
                 Expiry
                    |
        +-----------+-----------+
        |           |           |
      >60d       31-60d       <=30d
     HEALTHY    UPCOMING   EXPIRING_SOON
                                |
                            <=0 / expired
                                |
                             EXPIRED
```

The exact windows are configurable.

## Configuration

The workflow uses repository variables:

```yaml
env:
  THRESHOLD_DAYS: ${{ vars.THRESHOLD_DAYS || 30 }}
  UPCOMING_DAYS: ${{ vars.UPCOMING_DAYS || 60 }}
```

Default values:

| Variable | Default | Meaning |
|---|---:|---|
| `THRESHOLD_DAYS` | `30` | At or below this value = `EXPIRING_SOON` |
| `UPCOMING_DAYS` | `60` | At or below this value = `UPCOMING` |

To change them:

**Repository → Settings → Secrets and variables → Actions → Variables**

Example:

```text
THRESHOLD_DAYS=45
UPCOMING_DAYS=90
```

No workflow code change is required.

## Credential States

### `HEALTHY`

More than the configured upcoming window remains. No Issue is created.

### `UPCOMING`

The credential is inside the upcoming window but outside the immediate warning threshold. It remains visible in the report.

### `EXPIRING_SOON`

The credential is at or below the warning threshold. A GitHub Issue is created or updated.

### `EXPIRED`

The expiry time has passed. A GitHub Issue is created or updated and marked as expired.

## GitHub Configuration

### Required repository secrets

Create:

| Secret | Description |
|---|---|
| `AZURE_CLIENT_ID` | Client/Application ID of the Azure workload identity used by GitHub Actions |
| `AZURE_TENANT_ID` | Microsoft Entra tenant ID |

### Required Actions permissions

The workflow requests:

```yaml
permissions:
  id-token: write
  contents: read
  issues: write
```

`id-token: write` is required for Azure OIDC. `issues: write` is required to create, edit, label, and close monitoring Issues.

## Azure / Entra Permissions

The Azure workload identity must trust the GitHub repository and the `master` branch through a federated credential.

The identity also needs sufficient Microsoft Graph permissions to read application registrations and credential metadata.

The exact Graph permissions depend on the tenant security model. Follow least privilege and obtain the required consent through your organization's normal process.

The workflow only needs credential metadata; it does not need the actual secret values.

## Schedule and Manual Runs

The workflow supports both manual and scheduled execution:

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: "17 2 * * *"
```

The scheduled run is daily at **02:17 UTC**.

Manual execution:

```text
GitHub → Actions → Azure Secret Expiry Monitor → Run workflow
```

The repository currently uses the `master` branch.

## Issue Management

### Clean visible title

Issues use a human-readable title such as:

```text
Azure credential expiry: Business_Banking_SPN_DEV / Business_Banking_Secret_DEV
```

Application IDs and Key IDs are deliberately not placed in the visible title.

### Stable hidden marker

Each managed Issue contains a marker similar to:

```text
<!-- azure-credential:APPLICATION_ID:KEY_ID -->
```

This marker is the automation identity for the credential.

This design provides both:

- A clean title for engineers.
- A stable machine identifier for deduplication.

### Create or update behavior

For every alert credential:

```text
Does an open Issue contain the marker?
       |
   +---+---+
   |       |
  Yes      No
   |       |
Update   Create
```

The same Issue is therefore reused on every subsequent scan.

### Duplicate handling

If multiple open Issues contain the same credential marker:

1. The first matching Issue is retained.
2. Additional matching Issues are closed.
3. The duplicate Issue receives an automated comment pointing to the retained Issue.

This protects the repository from accumulating duplicate Issues after repeated workflow executions.

### Automatic closure

After processing current alerts, the workflow checks existing managed Issues.

If the credential marker is no longer present in the current active alert set, the Issue is closed automatically.

Typical reasons include:

- The credential was rotated.
- The old credential was deleted.
- The credential is no longer returned by Microsoft Graph.
- The credential moved outside the active alert set.

## Labels

The workflow expects these repository labels to exist:

| Label | Purpose |
|---|---|
| `credential-expiry` | Main category for monitored credential Issues |
| `client-secret` | Credential is a client secret |
| `certificate` | Credential is a certificate |
| `upcoming` | Credential is in the upcoming window |
| `expiring-soon` | Credential is inside the immediate warning window |
| `expired` | Credential has expired |

### Label examples

An expiring client secret:

```text
credential-expiry
client-secret
expiring-soon
```

An upcoming certificate:

```text
credential-expiry
certificate
upcoming
```

An expired client secret:

```text
credential-expiry
client-secret
expired
```

The workflow removes stale type/status labels before applying the current ones. A credential should not end up with both `upcoming` and `expired` simultaneously.

## Monitoring Report

Every scan generates a Markdown report and the final `publish-report` job publishes it once to `GITHUB_STEP_SUMMARY`.

The report includes:

### Overview

- Applications scanned
- Credentials scanned
- Healthy credentials
- Upcoming credentials
- Expiring credentials
- Expired credentials

### Credentials requiring attention

For each alert:

- Application
- Application ID
- Credential type
- Credential name
- Expiry date
- Whole-number days remaining
- Status
- Key ID

### Upcoming credentials

Shows credentials approaching the immediate warning threshold.

### Full credential inventory

Shows the complete credential inventory returned by Microsoft Graph.

### Monitoring policy

Shows the active warning and upcoming thresholds.

### Run details

The summary also records the workflow, branch, run number, trigger, commit, Issue-management result, and completion time.

## Scaling to Hundreds of SPNs

The monitor uses centralized discovery rather than a static list of SPNs.

A single workflow can discover many applications:

```text
1 Workflow
   |
   +-- Application 001
   |     +-- Secret A
   |     +-- Certificate B
   |
   +-- Application 002
   |     +-- Secret A
   |
   +-- Application 003
   |     +-- Secret A
   |     +-- Secret B
   |
   +-- ...
   |
   +-- Application 100+
```

When a new application is created, no workflow change is required. Once Microsoft Graph exposes it to the monitoring identity, the next scan automatically evaluates its credentials.

This is the recommended model when monitoring hundreds of SPNs because there is no per-SPN workflow configuration to maintain.

## Operational Process

The monitor identifies work; it does not automatically rotate credentials.

Recommended lifecycle:

```text
Credential enters warning window
          |
          v
GitHub Issue created/updated
          |
          v
Owner reviews Issue
          |
          v
Replacement credential created
          |
          v
Application/workloads updated
          |
          v
Authentication validated
          |
          v
Old credential removed
          |
          v
Next monitoring scan
          |
          v
Issue automatically closes
```

For production environments, credential rotation should use the organization's approved secret-management and deployment process.

## Recommended Ownership Model

For a larger environment, the next logical enhancement is application ownership.

For example:

| Application | Owner | Environment |
|---|---|---|
| `Business_Banking_SPN_DEV` | Banking DevOps | DEV |
| `Direct_Lending_SPN_PROD` | Lending Platform | PROD |

Ownership can later be used to automatically assign Issues and add environment/team labels.

## Security Considerations

### OIDC instead of long-lived Azure credentials

The monitoring identity is authenticated through GitHub Actions OIDC. This avoids storing a long-lived Azure client secret in GitHub.

### Secret values are not exposed

The workflow reads metadata such as credential name, Key ID, start/end dates, and credential type. It does not retrieve or print the actual client-secret value.

### Protect workflow changes

Because the workflow can authenticate to Azure and modify Issues, workflow changes should be reviewed carefully.

For production repositories, consider:

- Protected `master` branch
- Pull request review
- Required status checks
- Restricted direct pushes
- CODEOWNERS approval

## Troubleshooting

### Azure login fails

Check:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- OIDC federation configuration
- Federated credential subject/branch
- Azure workload identity permissions
- Microsoft Graph permissions
- Repository Actions permissions

### Scan succeeds but Issue management fails

Verify:

```yaml
permissions:
  issues: write
```

Also verify that Actions is permitted to create and modify Issues in the repository.

### Duplicate Issues appear

Check that the Issue body contains:

```text
<!-- azure-credential:APPLICATION_ID:KEY_ID -->
```

Issues created manually without the marker cannot be reliably associated with a monitored credential.

### Labels are not applied

Verify that all six expected labels exist exactly as named:

```text
credential-expiry
expired
expiring-soon
upcoming
client-secret
certificate
```

### Decimal days appear

The workflow intentionally converts remaining days to an integer:

```python
c["days"] = max(0, int(remaining_days))
```

Reports therefore display values such as `15 days`, not `15.42 days`.

### Report appears twice

Only the final `publish-report` job should append the generated report to `GITHUB_STEP_SUMMARY`. The scan job generates the report but does not publish it separately.

## Failure Behavior

The jobs are intentionally separated:

```text
Scan
  |
  v
Issue management
  |
  v
Publish summary
```

Issue management runs only if the scan succeeds.

The final summary job is configured to publish the scan report when the scan succeeds, even if Issue management fails. This keeps the monitoring data available during troubleshooting.

Concurrency is enabled:

```yaml
concurrency:
  group: azure-secret-expiry-monitor
  cancel-in-progress: false
```

This prevents overlapping executions of the monitor.

## Maintenance Guidelines

When modifying `.github/workflows/secret-expiry-monitor.yml`, preserve these principles:

1. Use OIDC instead of a long-lived Azure credential.
2. Keep scanning separate from Issue management.
3. Use Application ID + Key ID as the credential identity.
4. Keep Issue titles human-readable.
5. Use hidden markers for automation matching.
6. Make Issue operations idempotent.
7. Prevent duplicate Issues.
8. Keep status labels mutually exclusive.
9. Publish one monitoring report per run.
10. Never expose secret values in logs, reports, or Issues.

## Future Enhancements

Potential production-grade improvements include:

- Automatic Issue assignment based on application owners.
- Environment labels such as `dev`, `test`, and `prod`.
- Team/application ownership metadata.
- Escalation when a credential reaches 7 days.
- Teams, Slack, or email notifications for critical expirations.
- Rotation history tracking.
- A grace-period state after expiry.
- Verification that replacement credentials are actually being used before closing an Issue.
- Integration with Azure Key Vault or another approved secret-management platform.
- Automated tests for classification and deduplication logic.
- Pull-request validation for workflow changes.

## Quick Reference

### Files

```text
.github/workflows/secret-expiry-monitor.yml
README.md
```

### Secrets

```text
AZURE_CLIENT_ID
AZURE_TENANT_ID
```

### Variables

```text
THRESHOLD_DAYS   # default 30
UPCOMING_DAYS    # default 60
```

### Labels

```text
credential-expiry
expired
expiring-soon
upcoming
client-secret
certificate
```

### Schedule

```text
Daily at 02:17 UTC
```

### Trigger

```text
Manual: workflow_dispatch
Scheduled: cron
```

## Summary

Azure Secret Expiry Monitor provides centralized visibility into Microsoft Entra application credentials using GitHub Actions.

The solution follows a simple lifecycle:

```text
Discover → Classify → Alert → Resolve
```

It automatically discovers credentials, determines which ones require attention, maintains a clean set of GitHub Issues, applies useful labels, prevents duplicates, closes resolved alerts, and publishes a detailed monitoring report for every run.

Because discovery is centralized, the same workflow can monitor hundreds of application credentials without requiring per-SPN workflow configuration.
