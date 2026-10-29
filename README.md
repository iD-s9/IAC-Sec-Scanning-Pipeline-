# IAC-Sec-Scanning-Pipeline-
# IaC Security Scanning Pipeline (Bicep + Checkov + GitHub Actions)

## Problem

In the earlier projects, I secured Azure resources by clicking through the portal and checking each setting by hand. That doesn't scale, and it only catches mistakes after a resource already exists. This project "shifts security left": the storage account is written as code (Azure Bicep), and every commit is automatically scanned for misconfigurations before anything could be deployed.

## What I built

- **Two Bicep templates** for an Azure Storage account:
  - `main-insecure.bicep` (since deleted) was deliberately misconfigured: public blob access enabled, HTTP traffic allowed, TLS 1.0, and single-datacenter (LRS) replication.
  - `main-secure.bicep` is hardened: no public blob access, HTTPS only, TLS 1.2, shared key access disabled, public network access disabled, a default-deny network rule, and geo-redundant (GRS) replication.
- **A GitHub Actions workflow** (`.github/workflows/iac-scan.yml`) that runs [Checkov](https://www.checkov.io/) against every Bicep file on each push and pull request. With `soft_fail: false`, any failed check fails the run, so insecure code can't pass quietly.

## Before / After

| State | Screenshot |
|---|---|
| Before: insecure template fails the Checkov scan | [`before.png`](.Screenshots/before.png) |
| After: hardened template passes all checks (7 passed, 0 failed, 0 skipped) | [`after.png`](.Screenshots/after.png) |

## What happened along the way

Getting to a clean pass took more than writing a "secure" file, and that turned out to be the most useful part of the project.

1. **A false pass.** After deleting the insecure file, the pipeline went green, but the secure file had been saved with a `.yml` extension instead of `.bicep`. Checkov only scans Bicep files, so it had nothing to scan and passed by default. I caught it by noticing the filename in the commit history and confirming the log showed no checks had run. **Lesson: a green checkmark only means something if the scanner actually examined the file.** Now I check the `Passed checks` count, not just the color.
2. **Real findings in the "secure" file.** Once the file was renamed and actually scanned, Checkov flagged gaps I had missed:
   - **CKV_AZURE_206** (replication): the account used locally redundant storage (`Standard_LRS`). Changed to geo-redundant `Standard_GRS`.
   - **CKV_AZURE_35** (default network rule should deny): `publicNetworkAccess: 'Disabled'` alone wasn't enough. Added a `networkAcls` block with `defaultAction: 'Deny'`, allowing only trusted Azure services.
   - **CKV_AZURE_43** (storage account naming rules): a **false positive**. The name was built with `uniqueString()`, which produces a valid name at deploy time, but a static scanner can't evaluate it. Replaced it with a literal, valid name. In a team setting the alternative would be a documented suppression for this check.
3. **Final result:** 7 checks passed, 0 failed, 0 skipped.

## What I learned / next steps

- Scanning infrastructure code catches misconfigurations before deployment, rather than auditing them afterwards in the portal like my earlier projects.
- Security tooling needs verifying too. The false pass showed that a pipeline can look healthy while doing nothing.
- Not every finding is a real vulnerability. Telling a true positive (replication, network rules) from a false positive (naming) and handling each on purpose is part of the job.

Possible next steps: convert the RBAC setup from my first project into Bicep and scan it the same way, or add a deployment stage that only runs once the scan passes.

## Tools used

Azure Bicep, Checkov, GitHub Actions
