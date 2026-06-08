# Nirmata Pipeline Scanning — Helm Charts

Automated Kyverno policy scanning for Helm chart repositories, integrated with [Nirmata Control Hub (NCH)](https://nirmata.io). Violations appear in the CI job summary with one-click actions to fix or request an exception.

Works with **GitHub Actions** and **GitLab CI**.

---

## How it works

1. A push to your Helm chart branch triggers a scan via `nctl scan helm`
2. Violations are published to NCH and displayed in the job summary
3. For each violation you can:
   - **🛠️ Fix a violation** — triggers the Remediator Agent to open a PR with an automated fix
   - **📋 File an Exception** — opens the NCH findings page to request a policy exception
4. On a fix PR, use `@nirmatabot` commands to split the PR or request exceptions

See [docs/helm-scanning-guide.md](docs/helm-scanning-guide.md) for the full workflow, PolicyException management, and planned enhancements.

---

## Setup

### Prerequisites

- [Nirmata Control Hub (NCH)](https://nirmata.io) account
- Helm chart repository on GitHub or GitLab
- A separate policy repository with Kyverno policies (can be this same repo on a different branch — see `kyverno-policies` branch)

---

### GitHub Actions

#### Step 1 — Add secrets and variables

Go to **Settings → Secrets and variables → Actions** in your repository.

**Secrets** (encrypted):

| Secret | Required for | Description |
|---|---|---|
| `NIRMATA_USER_TOKEN` | Scan | NCH user token — used by `nctl login` |
| `NIRMATA_TEAM_TOKEN` | Scan (publish) | NCH team token — used to publish scan results |
| `NIRMATA_SERVICE_ACCOUNT_TOKEN` | Remediator | NCH service account token for the agent pod |
| `PAT` | Remediator | GitHub Personal Access Token with `repo` scope — used to create fix PRs |
| `GHCR_TOKEN` | Remediator | GitHub PAT with `read:packages` scope — pulls the remediator image from GHCR |

**Variables** (plain text):

| Variable | Description | Example |
|---|---|---|
| `NIRMATA_URL` | Your NCH endpoint | `https://nirmata.io` |
| `NIRMATA_USERID` | Your NCH user ID (email) | `user@example.com` |

#### Step 2 — Copy the workflow files

Copy both workflow files from this repository into your repository:

```
.github/workflows/helm-policy-scan.yml       # scan on push
.github/workflows/install-remediator.yml     # manual fix trigger
```

#### Step 3 — Configure environment variables

In `helm-policy-scan.yml`, update:

```yaml
env:
  CHART_PATH: charts/my-nginx-chart        # path to your Helm chart
  POLICIES_REPO: https://github.com/your-org/your-repo
  POLICIES_BRANCH: kyverno-policies        # branch containing your policies
```

In `install-remediator.yml`, update:

```yaml
env:
  NGINX_REPO: https://github.com/your-org/your-repo
  CHART_REF: main                          # branch with your Helm chart
  POLICIES_BRANCH: kyverno-policies
```

#### Step 4 — Set up your policies branch

Your policies branch must contain a `policies/` directory with Kyverno policy YAML files and a `policies/manifest.yaml`. See the `kyverno-policies` branch of this repository for the expected structure.

---

### GitLab CI

GitLab CI support uses the same `nctl` tool and NCH integration. Adapt the workflow as follows:

#### Step 1 — Add CI/CD variables

Go to **Settings → CI/CD → Variables** in your project.

| Variable | Protected | Description |
|---|---|---|
| `NIRMATA_USER_TOKEN` | ✅ | NCH user token |
| `NIRMATA_TEAM_TOKEN` | ✅ | NCH team token for publishing |
| `NIRMATA_SERVICE_ACCOUNT_TOKEN` | ✅ | NCH service account token for the agent |
| `GL_TOKEN` | ✅ | GitLab access token with `api` scope — used to create fix PRs |
| `GHCR_TOKEN` | ✅ | GitHub PAT with `read:packages` scope — pulls the remediator image |
| `NIRMATA_URL` | | Your NCH endpoint, e.g. `https://nirmata.io` |
| `NIRMATA_USERID` | | Your NCH user email |

#### Step 2 — Create `.gitlab-ci.yml`

```yaml
stages:
  - scan

helm-policy-scan:
  stage: scan
  image: ubuntu:22.04
  rules:
    - if: '$CI_COMMIT_BRANCH == "helm-app"'
    - when: manual
  script:
    - apt-get update -qq && apt-get install -y -qq curl jq python3
    - curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    - curl -fsSL https://downloads.nirmata.io/nctl/install.sh | NCTL_VERSION=4.10.17 bash
    - git clone --no-checkout --depth=1 --filter=blob:none
        --branch kyverno-policies
        "$CI_SERVER_URL/$CI_PROJECT_PATH.git" pipeline-scanning
    - cd pipeline-scanning && git sparse-checkout init --cone
    - git sparse-checkout set policies && git checkout HEAD && cd ..
    - nctl scan helm -r "charts/my-nginx-chart"
        -p "pipeline-scanning/policies/baseline"
        -p "pipeline-scanning/policies/restricted"
        -o json | tee report.json
    - nctl login --url "$NIRMATA_URL" --userid "$NIRMATA_USERID" --token "$NIRMATA_USER_TOKEN"
    - nctl scan repository --policies="pipeline-scanning/policies" --publish-token "$NIRMATA_TEAM_TOKEN"
```

> The Remediator Agent workflow for GitLab follows the same pattern as GitHub Actions — replace the GitHub-specific steps with GitLab equivalents and use `GL_TOKEN` instead of `PAT`.

---

## Policy repository structure

```
kyverno-policies/
└── policies/
    ├── manifest.yaml            # lists available policy groups
    ├── baseline/
    │   └── *.yaml               # Kyverno policy files
    ├── restricted/
    │   └── *.yaml
    ├── seccomp/
    │   └── *.yaml
    └── resource-limits/
        └── *.yaml
```

`manifest.yaml` controls which policy groups appear in the Remediator dropdown:

```yaml
policies:
  - name: restrict-seccomp-strict
    path: policies/seccomp
    ref: kyverno-policies
    resourceName: helm-chart-resources
    branchPrefix: fix/seccomp-
  - name: require-requests-limits
    path: policies/resource-limits
    ref: kyverno-policies
    resourceName: helm-chart-resources
    branchPrefix: fix/resource-limits-
```

---

## Further reading

- [Helm Scanning Guide](docs/helm-scanning-guide.md) — detailed workflow, PolicyException management, open issues, and planned enhancements
- [Nirmata Documentation](https://docs.nirmata.io)
- [Kyverno Policy Library](https://kyverno.io/policies/)
