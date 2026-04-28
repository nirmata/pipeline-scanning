# Helm Chart Policy Enforcement with Nirmata

This branch demonstrates an automated policy enforcement pipeline for a Helm chart using Nirmata's tooling. When a policy violation is detected, the developer can either **auto-fix it via a PR** or **request a policy exception through NCH** — all from the GitHub Actions job summary, without any local tooling.

---

## Repository Structure

| Branch | Contents |
|---|---|
| `remediation-helm-chart` | Helm chart, workflow files, remediator values |
| `exceptions-in-pipelines` | Custom Kyverno policies, `policies/manifest.yaml` |

**Key files on this branch:**

```
charts/my-nginx-chart/          # Helm chart being scanned
remediator-helm-values.yaml     # Base Helm values for the remediator agent
.github/workflows/
  helm-policy-scan.yml          # Scans the chart on every push, publishes to NCH
  install-remediator.yml        # Manually triggered — deploys the remediator and opens fix PRs
```

**Key files on `exceptions-in-pipelines`:**

```
policies/
  manifest.yaml                 # Policy metadata registry (used by the remediator)
  seccomp/
    restrict-seccomp-strict.yaml
  resource-limits/
    require_pod_requests_limits.yaml
```

---

## Secrets and Variables

Configure these under **Settings → Secrets and variables → Actions**:

| Name | Type | Description |
|---|---|---|
| `NIRMATA_TEAM_TOKEN` | Secret | Nirmata team token — used by `nctl` to publish scan results to NCH |
| `NIRMATA_SERVICE_ACCOUNT_TOKEN` | Secret | Nirmata service account token — used by the remediator agent |
| `PAT` | Secret | GitHub Personal Access Token with `repo` and `workflow` scopes — used by the remediator to open PRs |
| `NIRMATA_URL` | **Variable** | NCH base URL, e.g. `https://nirmata.io` — must be a variable (not a secret) so it renders correctly in job summary links |

---

## Workflow 1 — Helm Policy Scan

**File:** `.github/workflows/helm-policy-scan.yml`
**Triggers:** Push to `remediation-helm-chart`, or manual `workflow_dispatch`

### What it does

1. **Scans** `charts/my-nginx-chart` against the two custom Kyverno policies using `nctl scan helm`:
   - `policies/seccomp/restrict-seccomp-strict.yaml` — requires `seccompProfile: RuntimeDefault`
   - `policies/resource-limits/require_pod_requests_limits.yaml` — requires CPU/memory requests and limits
2. **Publishes** results to NCH using `nctl scan repository --publish` so violations appear in the NCH policy report.
3. **Writes a job summary** listing every violation with two action buttons:
   - **▶ Fix** — opens the Install Remediator Agent workflow to auto-fix the violation via a PR
   - **🔒 Exception** — links to the NCH policy report page where you can log in and submit a policy exception request
4. **Fails the pipeline** if any violations are found.

### Job summary example

When violations are detected the summary shows:

| Violation | Policy | Rule | Severity | Actions |
|---|---|---|---|---|
| `my-nginx-chart` | `seccomp-policy` | `check-seccomp` | medium | ▶ Fix · 🔒 Exception |
| `my-nginx-chart` | `resource-limits-policy` | `check-limits` | medium | ▶ Fix · 🔒 Exception |

Below the table, the summary explains both options and maps each policy group to the correct remediator dropdown value.

---

## Workflow 2 — Install Remediator Agent

**File:** `.github/workflows/install-remediator.yml`
**Triggers:** Manual only (`workflow_dispatch`) — typically launched by clicking **▶ Fix** in the scan job summary

### Inputs

| Input | Options |
|---|---|
| `policy_group` | `All violations` / `Seccomp violations` / `Resource limit violations` |

### What it does

1. **Reads `policies/manifest.yaml`** from the `exceptions-in-pipelines` branch to look up policy metadata for the selected group.
2. **Spins up a temporary [kind](https://kind.sigs.k8s.io/) Kubernetes cluster** on the GitHub Actions runner.
3. **Deploys the Nirmata remediator agent** (`go-agent-remediator` Helm chart from `nirmata/kyverno-charts`) using `remediator-helm-values.yaml` as the base config. For a specific policy group, a generated patch overrides the VCS target to scope the agent to that policy only.
4. **Bounces the agent** immediately after startup (scale down → delete RemediationRecords → scale up) to trigger an immediate remediation cycle instead of waiting for the cron schedule.
5. **Tails the agent logs** until all expected PRs are created or 8 minutes elapse:
   - `All violations` → waits for **2 PRs** (one per policy group, on separate branches)
   - Specific group → waits for **1 PR**

### PRs opened by the agent

| Policy group | PR branch prefix | What the PR fixes |
|---|---|---|
| Seccomp violations | `remediation-seccomp-` | Adds `seccompProfile: RuntimeDefault` to all pod specs |
| Resource limit violations | `remediation-resource-limits-` | Adds CPU/memory `requests` and `limits` to all containers |
| All violations | both prefixes | Opens one PR per policy group |

---

## End-to-End Flow

```
Push to remediation-helm-chart
        │
        ▼
┌─────────────────────┐
│  Helm Policy Scan   │──── no violations ────► pipeline passes
│  (automatic)        │
└─────────────────────┘
        │ violations found → pipeline fails
        │
        │  Job summary shows per-violation buttons
        │
        ├── click ▶ Fix ──────────────────────────────────────────────┐
        │                                                              ▼
        │                                             ┌───────────────────────────┐
        │                                             │  Install Remediator Agent │
        │                                             │  (manual, workflow_dispatch)│
        │                                             │  → kind cluster           │
        │                                             │  → remediator agent       │
        │                                             │  → PR(s) opened with fix  │
        │                                             └───────────────────────────┘
        │
        └── click 🔒 Exception ──► NCH policy report ──► submit exception request
```

---

## Adding a New Policy

1. Add the Kyverno policy YAML under `policies/<new-dir>/` on the `exceptions-in-pipelines` branch.
2. Add an entry to `policies/manifest.yaml` with `name`, `displayName`, `path`, `ref`, `resourceName`, and `branchPrefix`.
3. Add the new `displayName` as a `choice` option in the `policy_group` dropdown in `install-remediator.yml`.
4. Add a `-p "nginx/policies/<new-dir>"` flag to the scan step in `helm-policy-scan.yml`.
5. Add a `case` entry in the job summary section of `helm-policy-scan.yml` to map the policy name to the correct dropdown label.
