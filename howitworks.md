# How It Works

This document describes the internal mechanics of the two GitHub Actions workflows that power the Helm policy scan and exception automation.

---

## Architecture

Two workflows run in sequence:

```
Push to exceptions-in-pipelines
        │
        ▼
┌─────────────────────────────────┐
│  Workflow 1: nctl Helm policy   │  ← runs automatically on every push
│  scan  (nctl-helm-scan-pr.yml)  │
└──────────────┬──────────────────┘
               │  user reviews summary, copies resource + policy names
               ▼
┌─────────────────────────────────┐
│  Workflow 2: nctl Helm —        │  ← triggered manually from GitHub UI
│  process resources              │
│  (nctl-helm-process.yml)        │
└─────────────────────────────────┘
```

> **Why two branches?** GitHub only shows `workflow_dispatch` workflows in the Actions sidebar when they exist on the repository's **default branch** (`main`). The scan workflow lives on `exceptions-in-pipelines` and triggers on pushes there. The process workflow must also exist on `main` so the **Run workflow** button appears in the UI. Both branches are kept in sync.

---

## Workflow 1 — Scan (`nctl-helm-scan-pr.yml`)

**Triggers:** every `push` to `exceptions-in-pipelines`

### Steps

1. **Install Helm + nctl** — downloads both tools fresh on the runner each run.

2. **Authenticate nctl** — runs `nctl login` if the `NIRMATA_USERID` secret is set; skipped otherwise.

3. **Validate chart path** — checks that `HELM_CHART_PATH` is a valid directory. Also removes `nctl.zip` from the repo root if present — nctl rejects zip files larger than 5 MB when scanning, and an accidentally committed nctl binary would break the scan.

4. **Scan Helm chart** — runs:
   ```
   nctl scan helm -r <chart-path> --policy-sets pss-baseline,pss-restricted -o json
   ```
   Output is saved as `report.json`. The step uses `continue-on-error: true` because nctl exits with code `1` when violations are found — that is expected behaviour, not an error.

5. **Parse violations** — processes `report.json` with `jq` to extract pod controllers (Deployment, StatefulSet, DaemonSet, ReplicaSet, Job, CronJob) that have at least one failing rule. The nctl report format embeds the resource kind inside the name string, e.g. `"release-name-my-nginx-chart (Deployment)"`, so `jq` uses `split(" (")` to extract it (named-capture regex groups are not supported on the GitHub Actions runner's bundled jq version). Results are written to `violating_controllers.txt`.

6. **Render Helm chart** — runs `helm template scan-release <chart-path>` and splits the multi-document YAML into individual files under `rendered-resources/` (one file per Kubernetes resource, named `<kind>-<name>.yaml`). These files are passed verbatim into the `nctl ai` prompt in Workflow 2 so the AI sees the fully rendered spec, not raw Helm templates.

7. **Upload scan artifacts** — uploads `report.json` + `rendered-resources/` as `scan-artifacts-<run-id>` (retained 7 days).

8. **Write job summary** — publishes to the run summary:
   - Table of all violating pod controllers.
   - Per-resource breakdown: policy name, rule name, severity.
   - A copyable comma-separated policy list per resource for direct pasting into Workflow 2's `selected_policies` field.
   - A direct link to Workflow 2 with step-by-step instructions.

---

## Workflow 2 — Process (`nctl-helm-process.yml`)

**Triggers:** manual only — Actions → nctl Helm — process resources → Run workflow (Branch: `main`)

### Inputs

| Input | Required | Description |
|---|---|---|
| `selected_resources` | Yes | Comma-separated `Kind/name` strings copied from the scan summary |
| `action` | Yes | `generate-exception` or `remediate` |
| `selected_policies` | No | Comma-separated policy names to scope the output; leave blank to cover all violations |

### Steps

1. **Checkout `exceptions-in-pipelines`** — always checks out the chart branch regardless of which branch the workflow file was loaded from. This means the process workflow on `main` still reads the Helm chart from `exceptions-in-pipelines`.

2. **Install Helm + nctl** (including the `nctl ai` subcommand).

3. **Re-scan Helm chart** — produces a fresh `report.json`. This is necessary because the scan artifact from Workflow 1 is not passed between workflows; re-scanning ensures the violation data is current.

4. **Render Helm chart** — same split-per-resource rendering as Workflow 1.

5. **Process each selected resource** — iterates over every `Kind/name` in `selected_resources` and for each:

   a. **Extract violations** — queries `report.json` with `jq` for all failing rules on this resource. If `selected_policies` is non-empty, only rules belonging to those policies are included.

   b. **Locate rendered YAML** — finds the matching file from `rendered-resources/`. Falls back to a glob search if the exact filename doesn't match.

   c. **Build prompt** — constructs a structured prompt that includes:
      - The rendered Kubernetes YAML of the resource
      - The list of violations (policy name, rule name, reason message)
      - An exact output schema to follow (for `generate-exception`) or fix instructions (for `remediate`)

   d. **Run `nctl ai`** — calls:
      ```
      nctl ai --new-session --allowed-dirs <pwd> --prompt <prompt> --skip-permission-checks --force
      ```

   e. **Strip hallucinated policies** — `nctl ai` sometimes adds policy entries that were not in the actual violations. To fix this deterministically:
      - The directory is snapshotted before `nctl ai` runs.
      - After it completes, new files are identified by comparing before/after snapshots (this handles cases where `nctl ai` ignores the suggested output filename).
      - A Python script reads `report.json`, builds the set of policies that actually failed for this resource, intersects with `selected_policies` if provided, and rewrites each new PolicyException file keeping only matching `spec.exceptions` entries.

6. **Enforce `namespace: kyverno`** (`generate-exception` only) — a second Python pass over all files in `policyexceptions/` ensures every `PolicyException` resource has `metadata.namespace: kyverno`. This is a hard Kyverno requirement that `nctl ai` occasionally ignores.

7. **Upload artifact** — `policyexceptions-<run-id>` or `remediated-yamls-<run-id>`.

8. **Write summary** — lists what was processed, which policies were scoped, artifact download name.

---

## nctl Report JSON Schema

The nctl scan output (`report.json`) uses this structure:

```
{
  "policies": [
    {
      "name": "disallow-capabilities-strict",
      "namespaceScopedResources": [
        {
          "resources": [
            {
              "name": "release-name-my-nginx-chart (Deployment)",
              "rules": [
                {
                  "Name": "autogen-require-drop-all",
                  "Status": "fail",
                  "Severity": "medium",
                  "Message": "..."
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

Key points:
- The top-level key is `policies`, not `results`.
- Each policy entry holds its own resource list.
- The resource `name` field encodes the kind as a suffix: `"<name> (<Kind>)"`.
- Failing rules have `"Status": "fail"`.

---

## PolicyException Output Format

Generated exceptions always follow this structure:

```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: deployment-<resource-name>-exception
  namespace: kyverno          # enforced by post-processing; never changes
spec:
  match:
    any:
    - resources:
        kinds:
        - Deployment
        names:
        - <resource-name>
  exceptions:
  - policyName: disallow-capabilities-strict
    ruleNames:
    - autogen-require-drop-all
  - policyName: require-run-as-nonroot
    ruleNames:
    - autogen-run-as-non-root
```

Only policies with actual scan violations appear in `spec.exceptions` — hallucinated entries are removed by post-processing.
