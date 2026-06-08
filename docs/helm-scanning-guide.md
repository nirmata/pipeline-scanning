# Helm Chart Pipeline Scanning Guide

This guide covers automated Kyverno policy scanning for Helm chart repositories using the
[Nirmata Control Hub (NCH)](https://nirmata.io) pipeline scanning integration. It applies to
**GitHub Actions** and **GitLab CI** repositories.

For setup instructions, see the [repository README](../README.md).

---

## 1. Overview

### What it does

The pipeline scanning integration:

1. Renders your Helm chart and scans the output against Kyverno policies using `nctl scan helm`
2. Publishes scan findings to NCH for tracking and exception management
3. Displays violations in the CI job summary with one-click actions:
   - **🛠️ Fix a violation** — triggers the Remediator Agent to open an automated fix PR
   - **📋 File an Exception** — navigates to the NCH findings page to request a policy exception
4. On fix PRs, `@nirmatabot` commands let developers split PRs or request exceptions without leaving the PR review

### Why Helm charts are scanned differently

Helm charts contain Go templates, not raw Kubernetes YAML. The scanner first renders the chart
(equivalent to `helm template`) before passing the output to `nctl`. This means:

- Violations are reported against the _rendered_ resource, but the fix must be applied to the
  chart _source_ (template or `values.yaml`)
- The Remediator Agent uses `HelmMapper` to trace a rendered violation back to its source template
  file so fix PRs patch the right file

### Architecture

![Architecture diagram](diagrams/architecture.svg)

> **Note:** The dashed arrow from "PER in NCH" to "PolicyException YAML PR" represents a step that
> is **not yet implemented** — see [Open issue #418](#open-issues).

---

## 2. PolicyException Workflow

A PolicyException tells Kyverno to skip a specific policy check for a named resource. There are two
ways to create one: through the NCH UI, or through `@nirmatabot` commands on a remediation PR.

---

### 2a. Creating Exceptions via the NCH UI

Use this path for ad-hoc exceptions, bulk management, or exceptions not tied to a fix PR.

**Steps:**

1. In NCH, go to **Policies → Exception Requests → New Exception Request**
2. Fill in the form:
   - **Policy name** — the Kyverno policy to exempt (e.g. `restrict-seccomp-strict`)
   - **Resource** — kind, name, and namespace of the target resource
   - **TTL** — duration (e.g. `30d`, `90d`) or `permanent`
   - **Justification** — reason for the exception
3. Submit — the PER enters `pendingApproval` state
4. An approver receives an email notification and reviews the request in NCH
5. On approval, NCH generates a `PolicyExceptionSpec` YAML
6. **Manual step:** download the YAML, commit it to `kyverno-exceptions/<name>.yaml` in your
   repository, open a pull request, and merge it

**Limitations:**

| Limitation | Detail |
|---|---|
| Manual YAML deployment | Step 6 is always manual unless NCH GitOps integration is fully configured |
| No automatic PR creation | NCH's PR-creation pipeline only fires for `DeploymentType: GitOps`; RemediatorAgent users are excluded |
| TTL management is manual | Renewal and revoke PRs must be created and merged by hand |
| No link to CI findings | Exceptions created via UI are not linked to specific scan run findings |

---

### 2b. Creating Exceptions via Nirmatabot (Remediation PRs)

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
5. _(Pending [issue #418](#open-issues))_ Once approved, the agent will automatically open a PR
   with the generated `PolicyException` YAML in `kyverno-exceptions/<name>.yaml`

**GitHub vs GitLab:** The same command syntax works for both. GitHub triggers via the PR comment
webhook; GitLab uses the GitLab comment event webhook.

**Splitting a multi-policy PR:**

If the fix PR covers multiple policies and you want separate PRs:

```
@nirmatabot split-pr disallow-privilege-escalation require-run-as-nonroot
```

This creates one new PR per named policy, each containing only the fixes for that policy. Useful
when different policies have different approvers or urgency.

#### Open Issues

| Issue | Description | Status |
|---|---|---|
| [#417](https://github.com/nirmata/go-service-agent/issues/417) | PER created via nirmatabot shows "All Violations" scope in NCH instead of the specific requested policy | Fix in PR [#415](https://github.com/nirmata/go-service-agent/pull/415) — pending merge |
| [#418](https://github.com/nirmata/go-service-agent/issues/418) | No `PolicyException` YAML PR is created in the repository after a PER is approved in NCH | Design options in draft PR [#419](https://github.com/nirmata/go-service-agent/pull/419) — pending dev team decision |

**Workaround for #418 until the fix ships:** After approving a PER in NCH, manually download the
`PolicyExceptionSpec.yaml` from NCH (**Exception Requests → [PER name] → Download YAML**), commit
it to `kyverno-exceptions/<per-name>.yaml` in your repository, and merge the PR.

---

## 3. Enhancements

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
| YAML PR creation | Follows the existing [issue #418](#open-issues) fix once it ships |
| Ticket auto-close | After the YAML PR is merged, the ticket is updated with the PR link and closed |

**Supported ticketing systems (proposed):**

- **Jira** — project key + approval transition name configured in NCH cluster settings
- **GitHub Issues** — repository + approval label configured in NCH cluster settings
- **ServiceNow** — service catalog item + approval group

**Configuration:** Ticketing system type, project/board, and approval action are set per cluster or
per policy group in NCH settings. No per-repository configuration is needed.

---

## 4. Limitations

### Scanning

| Limitation | Detail |
|---|---|
| Helm rendering requires values | If your chart has required values without defaults, rendering may fail — provide a `ci/values.yaml` or set `HELM_ARGS` to supply test values |
| No incremental scanning | The full chart is re-scanned on every push — there is no diff-based mode |
| Multi-document YAML | One violation entry per rendered resource; complex umbrella charts may produce many entries |

### Remediation

| Limitation | Detail |
|---|---|
| Nirmatabot window | Commands must be posted within 20 minutes of PR creation; the window is fixed and tied to the agent pod lifetime |
| Ephemeral cluster state | RemediationRecord CRs are lost when the GitHub Actions job ends; there is no cross-run persistence |
| LLM fix quality | Automated fixes are best-effort; complex policies or Helm-specific patterns may require manual adjustment |
| Autogen rules | Kyverno `autogen-` rule violations (for pod controllers) are not always handled correctly by the Helm patcher |

### Policy Exceptions

| Limitation | Detail |
|---|---|
| No auto YAML PR | After approving a PER via nirmatabot, no YAML PR is created automatically ([issue #418](#open-issues)) |
| "All Violations" scope | PERs created via nirmatabot may show "All Violations" in NCH instead of the specific policy ([issue #417](#open-issues)) |
| Revoke / expiry PRs | Not yet supported for RemediatorAgent-type PERs; must be done manually |
| 20-minute approval window | Approvals that arrive after the nirmatabot window has closed do not trigger any automatic action (until [issue #418](#open-issues) is resolved) |
