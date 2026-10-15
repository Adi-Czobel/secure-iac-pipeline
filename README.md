# Demo

![CI](https://github.com/Adi-Czobel/Demo/actions/workflows/ci.yml/badge.svg?branch=main)
[![Policy Checks](https://github.com/Adi-Czobel/Demo/actions/workflows/policy.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/policy.yml)
[![Release](https://github.com/Adi-Czobel/Demo/actions/workflows/release.yml/badge.svg)](https://github.com/Adi-Czobel/Demo/actions/workflows/release.yml)
[![Security Checks](https://github.com/Adi-Czobel/Demo/actions/workflows/security.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/security.yml)
[![Dependabot Badge](https://github.com/Adi-Czobel/Demo/actions/workflows/dependabot-badge.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/dependabot-badge.yml)
[![Update Badges](https://github.com/Adi-Czobel/Demo/actions/workflows/update-badges.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/update-badges.yml)




> **Note:** The Dependabot badge will update once the `dependabot-badge.yml` workflow runs at least once.

A reference infrastructure-as-code repository where every change is run through a layered set
of security and quality checks in GitHub Actions.

**The pipeline is the point, not the infrastructure.** The Terraform here is a deliberately
trivial `null_resource` and the Dockerfile is a three-line Alpine stub. What is being
demonstrated is the control set that wraps them — the same workflows drop into a real repo
unchanged.

---

## 🛡️ Control set

| Layer | Tool | Catches | Blocks merge? |
|---|---|---|---|
| IaC misconfiguration | **Checkov** (terraform, kubernetes) | Public buckets, unrestricted ingress, missing encryption, unsafe pod specs | ✅ Yes |
| Terraform correctness | **`terraform fmt` / `validate`** | Formatting drift, syntax and schema errors | ✅ Yes |
| Dockerfile hygiene | **Hadolint** | Root users, unpinned base tags, `ADD` misuse | ✅ Yes |
| Vulnerable dependencies in a PR | **Dependency Review** | Known-vulnerable packages introduced by the diff | ✅ Yes (PRs) |
| SAST | **CodeQL** | Injection, unsafe deserialisation, taint-flow bugs | ⚠️ Reports only |
| SCA + IaC + image CVEs | **Trivy** (filesystem and image) | HIGH/CRITICAL CVEs, ignoring unfixed | ⚠️ Reports only |
| Committed secrets | **gitleaks** (full history, `fetch-depth: 0`) | API keys and tokens in the diff *and* in history | ⚠️ Reports only |
| Supply chain inventory | **Anchore SBOM** | Produces an SPDX SBOM as a build artifact | ℹ️ Artifact |
| Dependency drift | **Dependabot** | Stale Actions, npm, and pip dependencies, weekly | ℹ️ Opens PRs |

The "Reports only" rows carry `continue-on-error: true` and upload SARIF to the repository's
**Security** tab instead of failing the build. That split is deliberate: scanners with a
meaningful false-positive rate inform, and deterministic checks gate. A noisy scanner wired as
a hard gate gets disabled within a week, which is strictly worse than one that reports.

---

## 🚀 CI/CD Pipeline

This repository includes a robust CI/CD pipeline that automatically validates code quality and security.

### **Policy Checks Workflow**
The **Policy Checks** workflow (`policy.yml`) runs automatically on every pull request and includes:

#### 🐳 Dockerfile Linting  
- **Runs when**: Dockerfile is present in the repository  
- **Tool**: Hadolint  
- **Purpose**: Validates Docker best practices and catches common mistakes  
- **Checks**: Security vulnerabilities, optimization opportunities, and style consistency  

#### 🏗️ Terraform Validation  
- **Runs when**: `.tf` files are present  
- **Tools**: Terraform CLI with HashiCorp's setup action  
- **Validations**:  
  - Format Check: `terraform fmt -check`  
  - Initialization: Validates syntax without remote state  
  - Validation: Checks for resource configuration errors  

#### 🔒 Security Scanning (Checkov)  
- **Runs when**: Terraform or Kubernetes YAML files are present  
- **Tool**: Checkov by Bridgecrew  
- **Scope**: Terraform configurations & Kubernetes manifests  
- **Purpose**: Detects misconfigurations, compliance violations, and best practice deviations  

---

## 🤖 Dependency Management

Dependabot is configured to automatically monitor and update dependencies:  
- GitHub Actions: Weekly updates to workflow dependencies  
- NPM packages: Weekly updates for Node.js dependencies  
- Python packages: Weekly updates for pip dependencies  
- Limit: Maximum 10 open PRs to prevent spam  

---

## 📋 Workflow Triggers

### Automatic  
- Pull Requests: All policy checks run automatically  
- Scheduled Updates: Dependabot runs weekly  

### Manual  
- **Workflow Dispatch**: Any workflow can be triggered manually from the [Actions tab](https://github.com/Adi-Czobel/Demo/actions)  

---

## 🛡️ Pipeline design decisions

**Least-privilege workflow tokens.** Every workflow declares an explicit `permissions:` block
starting from `contents: read` and adds only what a given job needs — `security-events: write`
to upload SARIF, `pull-requests: write` for dependency review comments. The default
`GITHUB_TOKEN` is far broader than any of these jobs require.

**gitleaks runs against full history, not just the diff.** `fetch-depth: 0` is set for that
reason. A secret removed in a later commit is still present in history and still valid until
it is rotated; diff-only scanning gives false comfort.

**Checks are conditional on the relevant files existing.** Each job probes for `*.tf`,
`Docker/dockerfile`, or YAML before running, so the workflows can be lifted into a repo that
has only some of them without producing red crosses for absent file types.

**Kubernetes manifests ship with a default-deny NetworkPolicy.** `40-networkpolicy.yaml`
declares both `Ingress` and `Egress` policy types, which switches the pod from allow-all to
deny-all, then adds back same-namespace ingress on :80 and egress for DNS and HTTPS. Egress is
still broader than it should be for a real workload — it allows :80/:443 to any namespace —
and is marked as such inline.

**SBOM as a build artifact.** An SPDX SBOM is generated per build and uploaded. It is not
gated on, but it means any image this pipeline produces can be retrospectively checked against
a new CVE without rebuilding.

---

## 📊 Reporting

- SARIF from CodeQL and Trivy lands in the repository **Security** tab, so findings persist
  and are diffable across runs rather than living in expired job logs.
- Dependabot raises PRs weekly for Actions, npm, and pip, capped at 10 open to avoid spam.
- Status badges at the top reflect the latest run on `main`.

---

## 🔧 Local Development

**Prerequisites**
- Terraform >= 1.0
- Docker

**Run the same checks locally before pushing**
```bash
# Terraform
terraform -chdir=Terraform fmt -check -recursive
terraform -chdir=Terraform init -backend=false
terraform -chdir=Terraform validate

# Dockerfile lint
docker run --rm -i hadolint/hadolint < Docker/dockerfile

# IaC misconfiguration scan (terraform + kubernetes)
docker run --rm -v "${PWD}:/tf" bridgecrew/checkov -d /tf --framework terraform kubernetes

# Secret scan across full history
docker run --rm -v "${PWD}:/repo" zricethezav/gitleaks:latest detect \
  --source /repo --config /repo/.gitleaks.toml --verbose --redact

# Filesystem CVE scan
docker run --rm -v "${PWD}:/src" aquasec/trivy fs /src --severity HIGH,CRITICAL --ignore-unfixed
```

---

## 📁 Layout

```
.
├── .github/
│   ├── workflows/        ci, policy, security, release, badges
│   └── dependabot.yml
├── Terraform/main.tf     null_resource stub — a hook for the pipeline
├── Docker/dockerfile     Alpine stub — a target for Hadolint and Trivy
├── kubernetes/           namespace, serviceaccount, deployment, service, networkpolicy
└── .gitleaks.toml        secret-scanning rules and reviewed allowlist
```
