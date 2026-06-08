# Helm Chart Pipeline Scanning Guide

This guide covers automated Kyverno policy scanning for Helm chart repositories using the
[Nirmata Control Hub (NCH)](https://nirmata.io) pipeline scanning integration. It applies to
**GitHub Actions** and **GitLab CI** repositories.

For setup instructions, see the [repository README](../README.md).

---

## 1. Overview

### What it does

The pipeline scanning integration:

1. Scans your Helm chart repository for Kyverno policy violations using `nctl scan repository`
2. Publishes scan findings to NCH for tracking and exception management
3. Displays violations in the CI job summary with one-click actions:
   - **🛠️ Fix a violation** — triggers the Remediator Agent to open an automated fix PR
   - **📋 File an Exception** — navigates to the NCH findings page to request a policy exception
4. On fix PRs, `@nirmatabot` commands let developers split PRs or request exceptions without leaving the PR review

### How `nctl scan repository` handles Helm charts

`nctl scan repository` detects Helm charts in the scanned path and processes them natively —
no separate `helm template` step is needed. Violations are reported against the chart's resources
as they would be deployed.

When the Remediator Agent creates a fix PR, it uses **HelmMapper** to trace each violation back
to the Helm chart source file (template or `values.yaml`) so the patch is applied to the correct
location in your chart rather than to any rendered output.

### Architecture

![Architecture diagram](diagrams/architecture.svg)

> **Note (3a — NCH UI path):** Approved PERs suppress violations automatically on the next scan run — no YAML file is needed in the repository.
>
> **Note (3b — nirmatabot path):** A `PolicyException` YAML must be created and merged in the repository before violations are suppressed. Automatic creation of this YAML PR after PER approval is blocked by open issues [#417](https://github.com/nirmata/go-service-agent/issues/417) and [#418](https://github.com/nirmata/go-service-agent/issues/418).

---

## 2. Remediation

The Remediator Agent automatically creates fix PRs for Kyverno policy violations found during a
scan. Fixes are applied directly to the Helm chart source — templates and `values.yaml` — using
**HelmMapper**, which traces each violation back to its chart source file rather than to any
rendered manifest.

---

### 2a. Triggering a Fix

Use this path when a scan finds violations and you want the agent to attempt an automated fix.

**Steps:**

1. In the CI job summary (GitHub Actions) or pipeline output (GitLab), locate the violations table
2. Click **🛠️ Fix a violation** next to any violation row
3. On GitHub Actions, the **Install Remediator** workflow dispatch page opens. Select a policy
   group from the dropdown:
   - `All violations` — remediates every policy violation found in the scan
   - Individual policy names — remediates only violations from that policy
4. Click **Run workflow** — the Remediator Agent is deployed and begins reconciling

**Policy groups available:**

| Group | Description |
|---|---|
| `All violations` | Remediates all policy violations found in the scan |
| `restrict-seccomp-strict` | Adds or corrects seccomp profile annotations |
| `require-requests-limits` | Sets CPU and memory requests/limits |
| `disallow-capabilities-strict` | Removes or restricts Linux capabilities |
| `disallow-privilege-escalation` | Sets `allowPrivilegeEscalation: false` |
| `require-run-as-non-root-user` | Sets a non-zero `runAsUser` |
| `require-run-as-nonroot` | Sets `runAsNonRoot: true` |
| `restrict-volume-types` | Restricts volume types to the allowed set |

**GitHub vs GitLab:**

| | GitHub Actions | GitLab CI |
|---|---|---|
| Trigger | Workflow dispatch from job summary button | Pipeline trigger via GitLab API |
| Agent cluster | Ephemeral `kind` cluster created inside the Actions runner | Long-lived cluster with `nirmata-agent` already running |
| Policy group selection | Dropdown in the workflow dispatch UI | Variable passed to the triggered pipeline |

---

### 2b. What the Agent Does

Once triggered, the Remediator Agent runs the following steps:

1. **Cluster setup (GitHub Actions only):** A `kind` cluster is created inside the GitHub Actions
   runner. The `nirmata-agent` Helm chart is installed into the cluster, pointing to NCH.
2. **Scan results loaded:** The agent receives the scan findings from NCH — the same violations
   published by the initial `nctl scan repository` run.
3. **HelmMapper resolution:** For each violation, HelmMapper traces the affected resource back
   to its Helm chart source file (template YAML or `values.yaml`) to identify exactly where the
   patch must be applied.
4. **LLM remediation plan:** The agent generates a remediation plan using an LLM. The plan
   specifies the exact file, line, and change needed for each violation.
5. **Fix PR created:** The agent commits the patches to a new branch and opens a fix PR in the
   source repository. The PR description lists each violation addressed, the file patched, and the
   policy rule satisfied.

> **Note:** LLM-generated fixes should be reviewed before merging. The agent operates on
> best-effort — not all violations have deterministic fixes and some suggestions may require
> manual adjustment, especially for complex Helm templates with conditionals or `range` blocks.

---

### 2c. @nirmatabot Commands on Fix PRs

After the fix PR is created, `@nirmatabot` is active for **20 minutes**. Post commands as PR
comments within this window.

**Splitting a multi-policy PR:**

If the fix PR addresses violations from multiple policies and you want separate PRs per policy:

```
@nirmatabot split-pr <policy1> <policy2>
```

Example:

```
@nirmatabot split-pr disallow-privilege-escalation require-run-as-nonroot
```

This creates one new PR per listed policy, each containing only that policy's patches. Useful
when different policies have different owners or review timelines.

**Requesting an exception instead of a fix:**

If a fix is not feasible or not desired, use the `request-exception` command to enter the
[PolicyException workflow](#3-policyexception-workflow):

```
@nirmatabot request-exception <policy-name>
@nirmatabot request-exception <policy-name> duration=30d reason="Upstream fix pending in v2.1"
@nirmatabot request-exception <policy-name> duration=permanent reason="Third-party chart, no control over spec"
```

A `PolicyExceptionRequest` (PER) is created in NCH and routed to an approver. See
[Section 3b](#3b-creating-exceptions-via-nirmatabot-remediation-prs) for what happens next.

**GitHub vs GitLab:** The same `@nirmatabot` command syntax works for both platforms. GitHub
triggers via the PR comment webhook; GitLab uses the GitLab comment event webhook.

**Limitations:**

| Limitation | Detail |
|---|---|
| 20-minute window | `@nirmatabot` commands are only accepted for 20 minutes after the fix PR is created. After that, `split-pr` must be done manually and exceptions must be filed via the NCH UI. |
| LLM fix quality | Patches are AI-generated. Complex Helm templates with conditionals, `range` blocks, or shared helper templates may need manual review before merging. |
| Ephemeral cluster (GitHub Actions) | The `kind` cluster is destroyed when the workflow job completes. If reconciliation does not finish within the job timeout, the fix PR may be incomplete. |
| Policy group scope | Only predefined policy groups are available in the trigger UI. Arbitrary per-resource or per-namespace targeting is not supported at trigger time. |

---

## 3. PolicyException Workflow

A PolicyException tells Kyverno to skip a specific policy check for a named resource. There are two
ways to create one: through the NCH UI, or through `@nirmatabot` commands on a remediation PR.

---

### 3a. Creating Exceptions via the NCH UI

Use this path for ad-hoc exceptions, bulk management, or exceptions not tied to a fix PR.

**Steps:**

1. In NCH, go to **Policies → Exception Requests → New Exception Request**
2. Fill in the form:
   - **Policy name** — the Kyverno policy to exempt (e.g. `restrict-seccomp-strict`)
   - **Branch scope** — a specific branch (e.g. `helm-app`) or **all branches**
   - **TTL** — duration (e.g. `30d`, `90d`) or `permanent`
   - **Justification** — reason for the exception
3. Submit — the PER enters `pendingApproval` state
4. An approver receives an email notification and reviews the request in NCH
5. On approval, NCH records the exception. The next scan run against this repository
   automatically skips violations covered by the approved exception — no YAML file needs
   to be committed to the repository

**Limitations:**

| Limitation | Detail |
|---|---|
| Branch-only scoping | Exceptions can only be scoped to a specific branch or all branches. **Resource kind, name, and namespace cannot be selected.** The exception applies to every resource that triggers the policy on the scoped branch. |
| No link to CI findings | Exceptions created via UI are not linked to a specific scan run or fix PR |
| TTL management is manual | Renewal and revoke must be done manually via NCH UI |

---

### 3b. Creating Exceptions via Nirmatabot (Remediation PRs)

Use this path when reviewing a fix PR and deciding a fix is not feasible or not desired.

**When it's available:** The nirmatabot window is open for **20 minutes** after the fix PR is
created. Post your command as a comment on the PR within that window.

**Commands:**

```
@nirmatabot request-exception <policy-name>
@nirmatabot request-exception <policy-name> duration=30d reason="Upstream fix pending in v2.1"
@nirmatabot request-exception <policy-name> duration=permanent reason="Third-party chart, no control over spec"
```

**What happens:**

1. Nirmatabot parses the comment and identifies the matching violation in the fix PR
2. The agent creates a `PolicyExceptionRequest` (PER) in NCH scoped to the specific policy and
   affected resources
3. A confirmation comment is posted on the PR with a direct link to the PER in NCH
4. The PER enters `pendingApproval` state — an approver reviews it in NCH
5. _(Pending [#418](#open-issues))_ Once approved, the agent should automatically open a PR in
   this repository with the generated `PolicyException` YAML at `kyverno-exceptions/<name>.yaml`
6. Merge that PR — subsequent scan runs will then skip the violations covered by the exception

**GitHub vs GitLab:** The same command syntax works for both. GitHub triggers via the PR comment
webhook; GitLab uses the GitLab comment event webhook.

**Splitting a multi-policy PR:**

If the fix PR covers multiple policies and you want separate PRs:

```
@nirmatabot split-pr disallow-privilege-escalation require-run-as-nonroot
```

This creates one new PR per named policy, each containing only the fixes for that policy. Useful
when different policies have different approvers or urgency.

#### Open Issues {#open-issues}

| Issue | Description | Status |
|---|---|---|
| [#417](https://github.com/nirmata/go-service-agent/issues/417) | PER created via nirmatabot shows "All Violations" scope in NCH instead of the specific requested policy | Fix in PR [#415](https://github.com/nirmata/go-service-agent/pull/415) — pending merge |
| [#418](https://github.com/nirmata/go-service-agent/issues/418) | No `PolicyException` YAML PR is created in the repository after the PER is approved in NCH | Design options in draft PR [#419](https://github.com/nirmata/go-service-agent/pull/419) — pending dev team decision |

**Workaround for #418:** After approving a PER in NCH, manually download the `PolicyExceptionSpec.yaml` (**Exception Requests → [PER name] → Download YAML**), commit it to `kyverno-exceptions/<per-name>.yaml` in your repository, open a PR, and merge it. Subsequent scans will then skip the covered violations.

---

## 4. Enhancements

### Proposed: Ticketing System Integration for Exception Requests

**Motivation:** Today there is no structured approval trail when a developer requests a policy
exception from a scan finding. Exceptions flow through NCH but are disconnected from standard
engineering ticketing systems (Jira, GitHub Issues, ServiceNow) that security and compliance teams
already use for approval workflows.

**Proposed flow:**

When a developer clicks **📋 File an Exception** in the CI job summary, an exception request form
opens pre-filled with the policy name, resource name, and branch from the scan finding. On submit,
a ticket is automatically created in the configured ticketing system. When the ticket is approved
by the security team, a webhook triggers NCH to create and approve the PER, which then generates
the `PolicyException` YAML PR in the repository.

![Ticketing integration diagram](diagrams/ticketing-integration.svg)

**Integration points:**

| Component | Proposed behavior |
|---|---|
| "File an Exception" button | Opens a form pre-filled from the scan finding; on submit, creates a ticket in the configured system |
| Ticket body | Includes policy name, rule, resource kind/name/namespace, branch, link to NCH finding |
| Ticket approval action | Configurable: Jira transition, GitHub label (e.g. `exception-approved`), ServiceNow approval |
| Webhook → NCH | On ticket approval, NCH API call creates and approves the PER |
| Auto-skip on next scan | Once the PER is approved, the next scan run automatically skips matching violations |
| Ticket auto-close | After the next clean scan, the ticket is updated with the result and closed |

**Supported ticketing systems (proposed):**

- **Jira** — project key + approval transition name configured in NCH cluster settings
- **GitHub Issues** — repository + approval label configured in NCH cluster settings
- **ServiceNow** — service catalog item + approval group

**Configuration:** Ticketing system type, project/board, and approval action are set per cluster or
per policy group in NCH settings. No per-repository configuration is needed.

---
