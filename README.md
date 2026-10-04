# Azure Secret Expiry Monitor

<img width="1536" height="1024" alt="Azure Credential Expiry Monitor" src="https://github.com/user-attachments/assets/734b5a3b-077e-4396-a0e8-68499a716847" />

A GitHub Actions workflow that monitors Microsoft Entra ID application credentials and tracks client-secret and certificate expiry across all applications visible to the monitoring identity.

It automatically scans credentials, calculates remaining days, creates or updates one GitHub Issue per credential requiring attention, applies the existing status labels, closes duplicates and resolved Issues, and publishes a monitoring report for every run.

## How It Works

```text
GitHub Actions
      |
      v
Authenticate to Azure with OIDC
      |
      v
Microsoft Graph
      |
      v
Scan all Entra applications
      |
      v
Read client secrets + certificates
      |
      v
Calculate days remaining
      |
      v
Classify credential
      |
      +-------------------+--------------------+----------------+
      |                   |                    |                |
   Healthy            Upcoming          Expiring soon       Expired
   >60 days           8-60 days           0-7 days          <0 days
      |                   |                    |                |
      |                   +--------------------+----------------+
      |                                        |
      |                                        v
      |                              Create / update Issue
      |                                        |
      |                                        v
      |                                  Apply labels
      |                                        |
      +----------------------------------------+
                       |
                       v
               Publish run summary
```

The workflow is split into three jobs so the run shows clear stages:

```text
Scan Entra application credentials
              |
              v
Create or update credential issues
              |
              v
Publish monitoring summary
```

## Credential Monitoring

The workflow discovers application registrations through Microsoft Graph and reads credential metadata only.

For each client secret or certificate it records:

- Application name
- Application ID
- Credential type
- Credential name
- Key ID
- Expiry date
- Whole-number days remaining
- Current status

The actual secret value is never retrieved or written to GitHub Issues or the monitoring report.

## Expiry Classification

The current default policy is:

| Days remaining | Status | Issue label |
|---:|---|---|
| More than 60 | `HEALTHY` | No status label / no active Issue |
| 8–60 | `UPCOMING` | `upcoming` |
| 0–7 | `EXPIRING_SOON` | `expiring-soon` |
| Less than 0 | `EXPIRED` | `expired` |

The thresholds are controlled by repository variables:

```text
THRESHOLD_DAYS=7
UPCOMING_DAYS=60
```

If the variables are not configured, the workflow uses these defaults.

## GitHub Issue Lifecycle

<img width="1774" height="887" alt="Azure Credential Lifecycle Dashboard" src="https://github.com/user-attachments/assets/b8094d30-8ec1-4f7e-842e-9f15989fecbb" />


The workflow maintains one Issue for each monitored credential that is in the active alert range.

A hidden marker identifies the credential:

```text
<!-- azure-credential:APPLICATION_ID:KEY_ID -->
```

The visible title remains simple:

```text
Azure credential expiry: Business_Banking_SPN_DEV / Business_Banking_Secret_DEV
```

On every run the workflow:

1. Scans the current Azure credential state.
2. Finds an existing Issue using the credential marker.
3. Updates the existing Issue if it already exists.
4. Creates an Issue only when no matching Issue exists.
5. Removes stale type/status labels.
6. Applies the current labels.
7. Closes duplicate Issues for the same credential.
8. Closes managed Issues when the credential is no longer in the active alert set.

This makes the Issue lifecycle idempotent and prevents a new Issue from being created every day for the same credential.

## Labels

The workflow uses the existing repository labels only. It does not create labels.

```text
credential-expiry
client-secret
certificate
upcoming
expiring-soon
expired
```

Examples:

```text
credential-expiry + client-secret + upcoming
credential-expiry + client-secret + expiring-soon
credential-expiry + certificate + expired
```

The status labels are mutually exclusive. A credential will not retain `upcoming` after it moves into `expiring-soon`, for example.

## Secret Rotation

If a credential's expiry metadata changes while its identity remains the same, the next scan updates the existing Issue and its labels.

If a secret is rotated by deleting the old secret and creating a new secret, Microsoft Entra normally gives the new credential a different Key ID. The monitor therefore treats it as a new credential:

```text
Old credential
     |
     v
Old Issue becomes resolved/closed

New credential
     |
     v
New Key ID detected
     |
     v
New Issue created if it is inside the alert window
```

This tracks the actual Azure credential rather than only the application.

## Scaling to Hundreds of SPNs

No SPN list is maintained in the workflow.

The monitor queries Microsoft Graph for application registrations and evaluates every returned application. Therefore, when a new Service Principal/Application Registration is added and is visible to the monitoring identity, it is automatically included in the next scan.

```text
One workflow
     |
     +-- Application A
     |     +-- Secret
     |     +-- Certificate
     |
     +-- Application B
     |     +-- Secret
     |
     +-- Application C
     |     +-- Secret
     |     +-- Secret
     |
     +-- ...
     |
     +-- Hundreds of applications
```

No workflow change is required for each new application.

## Authentication and Permissions

GitHub Actions authenticates to Azure using workload identity federation / OIDC.

Required GitHub repository secrets:

```text
AZURE_CLIENT_ID
AZURE_TENANT_ID
```

Required workflow permissions:

```yaml
permissions:
  id-token: write
  contents: read
  issues: write
```

`id-token: write` is used for Azure OIDC authentication. `issues: write` is required to create, update, label, and close monitoring Issues.

The Azure identity must have the required Microsoft Graph permissions to read application and credential metadata.

## Schedule

The workflow supports both scheduled and manual execution:

```yaml
on:
  workflow_dispatch:
  schedule:
    - cron: "17 2 * * *"
```

The current schedule is daily at **02:17 UTC**.

It can also be started manually from:

```text
GitHub → Actions → Azure Secret Expiry Monitor → Run workflow
```

## Monitoring Summary

Every successful scan produces one run summary containing:

- Applications scanned
- Credentials scanned
- Healthy credentials
- Upcoming credentials
- Expiring credentials
- Expired credentials
- Credentials requiring attention
- Full credential inventory
- Expiry dates and days remaining
- Credential type and Key ID
- Run information

The report is published once by the final `Publish monitoring summary` job.

## Repository Structure

```text
.
├── .github/
│   └── workflows/
│       └── secret-expiry-monitor.yml
└── README.md
```

The workflow implementation is:

```text
.github/workflows/secret-expiry-monitor.yml
```

## Run Behavior

The workflow uses concurrency so two monitoring runs do not modify Issues at the same time.

```yaml
concurrency:
  group: azure-secret-expiry-monitor
  cancel-in-progress: false
```

The scan runs first. Issue management runs only after a successful scan. The monitoring summary is then published from the scan results.

## What This Automation Does Not Do

The monitor does not rotate secrets, modify Azure applications, or retrieve secret values.

Its purpose is to **discover credential expiry, track the required action in GitHub Issues, and provide a current monitoring report**.
