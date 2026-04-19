# Helm Policy Scan & Exception Automation

This branch (`exceptions-in-pipelines`) contains a two-workflow system that scans a Helm chart for Kyverno policy violations and lets you generate **PolicyException YAMLs** or **remediated resource YAMLs** directly from the GitHub Actions UI — no local tooling required.

---

## Overview

```
Push to exceptions-in-pipelines
        │
        ▼
┌─────────────────────────────────┐
│  Workflow 1: nctl Helm policy   │  ← runs automatically on every push
│  scan  (nctl-helm-scan-pr.yml)  │
│                                 │
│  • Scans Helm chart             │
│  • Identifies violating pod     │
│    controllers (Deployment,     │
│    StatefulSet, DaemonSet, etc) │
│  • Renders chart to per-        │
│    resource YAML files          │
│  • Publishes violations table   │
│    + copyable policy list to    │
│    the job summary              │
│  • Uploads scan-artifacts ZIP   │
└──────────────┬──────────────────┘
               │  review summary, copy resource name + policy list
               ▼
┌─────────────────────────────────┐
│  Workflow 2: nctl Helm —        │  ← triggered manually from GitHub UI
│  process resources              │
│  (nctl-helm-process.yml)        │
│                                 │
│  • Re-scans chart for fresh     │
│    violation data               │
│  • Feeds rendered YAML +        │
│    violations into nctl ai      │
│  • Generates PolicyException    │
│    YAML  OR  remediated YAML    │
│  • Strips any hallucinated      │
│    policies from output         │
│  • Enforces namespace: kyverno  │
│  • Uploads result as artifact   │
└─────────────────────────────────┘
```

---

## Prerequisites

### Repository Secrets

Configure these under **Settings → Secrets and variables → Actions → Secrets**:

| Secret | Required | Description |
|---|---|---|
| `NIRMATA_URL` | Yes | Your Nirmata instance URL, e.g. `https://www.nirmata.io` |
| `NIRMATA_TOKEN` | Yes | Nirmata API token |
| `NIRMATA_USERID` | No | User email — required only if your `nctl` version needs `nctl login` with `--userid` |

### Repository Variable (optional)

Configure under **Settings → Secrets and variables → Actions → Variables**:

| Variable | Default | Description |
|---|---|---|
| `HELM_CHART_PATH` | `.` (repo root) | Path to the directory containing `Chart.yaml`. Setting this avoids nctl's 5 MB zip-file limit when scanning the whole repo. Example: `charts/myapp` |

### Branch layout

| Branch | Purpose |
|---|---|
| `exceptions-in-pipelines` | Helm chart source, scan workflow, process workflow copy |
| `main` | Process workflow (must live here for the GitHub UI **Run workflow** button to appear) |

> **Why two branches?** GitHub only shows `workflow_dispatch` workflows in the Actions sidebar when they exist on the repository's **default branch** (`main`). The scan workflow lives on `exceptions-in-pipelines` and triggers on pushes to that branch. The process workflow is present on both branches but the one on `main` is what the GitHub UI uses.

---

## How It Works

### Workflow 1 — Scan (`nctl-helm-scan-pr.yml`)

**Triggers:** every `push` to `exceptions-in-pipelines`

**Steps:**

1. **Install Helm + nctl** — downloads both tools fresh on the runner.
2. **Authenticate nctl** — logs in to Nirmata if `NIRMATA_USERID` secret is set.
3. **Validate chart path** — confirms `HELM_CHART_PATH` is a valid directory; removes `nctl.zip` from root if present (it would exceed nctl's 5 MB package size limit).
4. **Scan Helm chart** — runs `nctl scan helm -r <chart-path> --policy-sets pss-baseline,pss-restricted -o json` and saves output as `report.json`. Uses `continue-on-error: true` because nctl exits with code `1` when violations are found (that is expected behaviour, not a failure).
5. **Parse violations** — uses `jq` to walk `report.json` and extract every pod controller kind (`Deployment`, `StatefulSet`, `DaemonSet`, `ReplicaSet`, `Job`, `CronJob`) that has at least one failing rule. Results go into `violating_controllers.txt`.
6. **Render Helm chart** — runs `helm template scan-release <chart-path>` and splits the output into individual per-resource YAML files under `rendered-resources/` (one file per Kubernetes resource). These are fed verbatim into the AI prompt in Workflow 2 so nctl ai can see the actual rendered spec.
7. **Upload scan artifacts** — uploads `report.json` + `rendered-resources/` as `scan-artifacts-<run-id>` (retained 7 days).
8. **Write job summary** — publishes to the GitHub Actions job summary:
   - A table of all violating pod controllers.
   - A per-resource breakdown table: policy name, rule name, severity.
   - A copyable comma-separated policy list per resource for pasting into Workflow 2.
   - A direct link to Workflow 2 with fill-in-the-field instructions.

---

### Workflow 2 — Process (`nctl-helm-process.yml`)

**Triggers:** manual only — **Actions → nctl Helm — process resources → Run workflow** (Branch: `main`)

**Inputs:**

| Field | Required | Description |
|---|---|---|
| `selected_resources` | Yes | Comma-separated `Kind/name` identifiers copied from the scan summary. Example: `Deployment/release-name-my-nginx-chart` |
| `action` | Yes | `generate-exception` or `remediate` |
| `selected_policies` | No | Comma-separated policy names to scope the output to. Copy the ready-made line from the scan summary. Leave blank to cover **all** violations for the selected resource. Example: `disallow-capabilities-strict,require-run-as-nonroot` |

**Steps:**

1. **Checkout chart branch** — checks out `exceptions-in-pipelines` so it has access to the Helm chart regardless of which branch triggered the workflow.
2. **Install Helm + nctl** (including `nctl ai`).
3. **Authenticate nctl** — same as Workflow 1.
4. **Validate chart path**.
5. **Re-scan Helm chart** — produces a fresh `report.json` used to validate violations.
6. **Render Helm chart** — same split-per-resource rendering as Workflow 1; provides rendered YAML for the AI prompt.
7. **Process each selected resource** — for each `Kind/name` in `selected_resources`:
   - Extracts the list of violations from `report.json` using `jq`, optionally filtered to only the policies named in `selected_policies`.
   - Locates the rendered YAML file for that resource.
   - Builds a precise prompt that includes the rendered YAML and violation details, then calls `nctl ai`.
   - **Strips hallucinated policies:** snapshots the output directory before `nctl ai` runs, compares after, and runs a Python filter on every newly created file. The filter reads `report.json` to confirm which policies actually failed, removes any `spec.exceptions` entries for policies that did not violate, and rewrites the file.
8. **Enforce `namespace: kyverno`** (`generate-exception` only) — a second Python pass guarantees all `PolicyException` resources carry `metadata.namespace: kyverno`, regardless of what nctl ai wrote.
9. **Upload artifact**:
   - `generate-exception` → artifact named `policyexceptions-<run-id>`
   - `remediate` → artifact named `remediated-yamls-<run-id>`
10. **Write summary** — shows what was processed, which policies were scoped, and how to download the artifact.

---

## Step-by-Step Usage

### 1. Push a change to trigger the scan

Any commit pushed to `exceptions-in-pipelines` automatically triggers Workflow 1.

```
git checkout exceptions-in-pipelines
# ... edit your Helm chart ...
git push origin exceptions-in-pipelines
```

### 2. Review the scan summary

Go to **Actions → nctl Helm policy scan → [latest run] → Summary**.

You will see:

```
⚠️ Violating Pod Controllers
# | Resource
1 | Deployment/release-name-my-nginx-chart

Policy violations per resource

Deployment/release-name-my-nginx-chart
| Policy                            | Rule                           | Severity |
|-----------------------------------|--------------------------------|----------|
| disallow-capabilities-strict      | autogen-require-drop-all       | medium   |
| disallow-privilege-escalation     | autogen-privilege-escalation   | medium   |
| require-run-as-nonroot            | autogen-run-as-non-root        | medium   |
| restrict-seccomp-strict           | autogen-check-seccomp-strict   | high     |

> Copy into `selected_policies` to scope to only these violations:
> `disallow-capabilities-strict,disallow-privilege-escalation,require-run-as-nonroot,restrict-seccomp-strict`
```

### 3. Trigger the process workflow

1. Go to **Actions → nctl Helm — process resources**.
2. Click **Run workflow** (leave Branch as `main`).
3. Fill in the fields:

   | Field | Example |
   |---|---|
   | `selected_resources` | `Deployment/release-name-my-nginx-chart` |
   | `action` | `generate-exception` |
   | `selected_policies` | `disallow-capabilities-strict,require-run-as-nonroot` *(or leave blank for all)* |

4. Click **Run workflow**.

### 4. Download the artifact

Once the run completes, scroll to the **Artifacts** section of the run summary and download:
- `policyexceptions-<run-id>.zip` for PolicyException YAMLs, or
- `remediated-yamls-<run-id>.zip` for remediated resource YAMLs.

---

## Output Examples

### PolicyException YAML (`generate-exception`)

```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: deployment-release-name-my-nginx-chart-exception
  namespace: kyverno
spec:
  match:
    any:
    - resources:
        kinds:
        - Deployment
        names:
        - release-name-my-nginx-chart
  exceptions:
  - policyName: disallow-capabilities-strict
    ruleNames:
    - autogen-require-drop-all
  - policyName: require-run-as-nonroot
    ruleNames:
    - autogen-run-as-non-root
```

### Remediated YAML (`remediate`)

The full Kubernetes resource YAML with only the non-compliant fields corrected — `securityContext` fields added, hostPID/hostIPC/hostNetwork set to false, etc. The rest of the spec is unchanged.

---

## Selecting Specific Policies

By default (leaving `selected_policies` blank), the process workflow covers **all** violations for the selected resource.

To generate a PolicyException for only a subset of violations — for example, you accept the seccomp violation but need an exception for capabilities only:

1. Copy the policy list shown under the resource in the scan summary.
2. Remove the policies you do not want to exempt.
3. Paste the remaining names into the `selected_policies` field.

The filter is applied both when building the `nctl ai` prompt (so the AI only sees the chosen violations) and as a post-processing step on the generated file (so any extra entries the AI may have added are removed).

---

## Notes

- **Exit code 1 from scan is normal.** `nctl scan` exits with `1` when violations exist. The scan step uses `continue-on-error: true` so the workflow continues and produces `report.json` as expected.
- **`nctl ai` may create files with different names** than the expected path. The process workflow handles this by comparing the directory before and after `nctl ai` runs and filtering every newly created file.
- **`namespace: kyverno` is always enforced.** Even if `nctl ai` writes a different namespace, a post-processing step rewrites it to `kyverno`.
- **Multiple resources.** Comma-separate multiple `Kind/name` values in `selected_resources` to process several resources in a single run.
- **Scan artifacts expire in 7 days.** Download `scan-artifacts-<run-id>` if you want to keep `report.json` or the rendered YAMLs longer.
