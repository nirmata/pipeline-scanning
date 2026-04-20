# Helm Policy Scan & Exception Automation.

Automatically scans your Helm chart for Kyverno policy violations and lets you generate **PolicyException YAMLs** or **remediated resource YAMLs** from the GitHub Actions UI — no local tooling needed.

---

## What It Does

1. **Scans** your Helm chart against the `pss-baseline` and `pss-restricted` Kyverno policy sets on every push to this branch.
2. **Shows** which pod controllers are violating policies and exactly which policies failed.
3. **Lets you choose** — via a simple GitHub UI form — whether to generate a PolicyException or a remediated resource YAML for any violating resource.

---

## Prerequisites

Add these under **Settings → Secrets and variables → Actions**:

| Name | Type | Description |
|---|---|---|
| `NIRMATA_URL` | Secret | Your Nirmata instance URL, e.g. `https://www.nirmata.io` |
| `NIRMATA_TOKEN` | Secret | Nirmata API token |
| `NIRMATA_USERID` | Secret | *(optional)* User email, if your nctl version requires login |
| `HELM_CHART_PATH` | Variable | *(optional)* Path to the folder containing `Chart.yaml`. Defaults to `.` (repo root). Example: `charts/myapp` |

---

## How to Use

### Step 1 — Push to trigger the scan

Push any change to the `exceptions-in-pipelines` branch. The scan runs automatically.

### Step 2 — Review the scan summary

Go to **Actions → nctl Helm policy scan → [latest run] → Summary**.

The summary shows:
- Which pod controllers have violations
- A table of failing policies and rules per resource
- A ready-to-copy policy list for each resource

### Step 3 — Generate exceptions or remediate

1. Go to **Actions → nctl Helm — process resources**
2. Click **Run workflow** (Branch: `main`)
3. Fill in the form:

   | Field | What to enter |
   |---|---|
   | `selected_resources` | Copy the resource name from the scan summary, e.g. `Deployment/release-name-my-nginx-chart` |
   | `action` | `generate-exception` or `remediate` |
   | `selected_policies` | *(optional)* Paste the policy list from the scan summary to scope to specific violations. Leave blank to cover all. |

4. Click **Run workflow**

### Step 4 — Download the results

Once the run completes, download from the **Artifacts** section:

| Action chosen | Artifact name | Contents |
|---|---|---|
| `generate-exception` | `policyexceptions-<run-id>` | Kyverno `PolicyException` YAML(s), scoped to the violations you selected, with `namespace: kyverno` |
| `remediate` | `remediated-yamls-<run-id>` | Full Kubernetes resource YAML(s) with only the non-compliant fields corrected |

---

For a detailed explanation of how the workflows work internally, see [howitworks.md](howitworks.md).
