# Demo

![CI](https://github.com/Adi-Czobel/Demo/actions/workflows/ci.yml/badge.svg?branch=main)
[![Policy Checks](https://github.com/Adi-Czobel/Demo/actions/workflows/policy.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/policy.yml)
[![Release](https://github.com/Adi-Czobel/Demo/actions/workflows/release.yml/badge.svg)](https://github.com/Adi-Czobel/Demo/actions/workflows/release.yml)
[![Security Checks](https://github.com/Adi-Czobel/Demo/actions/workflows/security.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/security.yml)
[![Dependabot Badge](https://github.com/Adi-Czobel/Demo/actions/workflows/dependabot-badge.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/dependabot-badge.yml)
[![Update Badges](https://github.com/Adi-Czobel/Demo/actions/workflows/update-badges.yml/badge.svg?branch=main)](https://github.com/Adi-Czobel/Demo/actions/workflows/update-badges.yml)

A practice repo for shift-left security: every push and pull request runs through a stack of
scanners before anything can merge, and tagged releases produce a signed container image with
a bill of materials attached.

The point of this repo is the pipeline, not the infrastructure. The Terraform is a one-line
placeholder and the Dockerfile is three lines of Alpine. They exist so the scanners have
something to scan. The workflows are the actual work, and they drop into a real project
unchanged.

## What runs, and when

**On every pull request** (these block the merge if they fail):

- **Checkov** checks the Terraform and Kubernetes files for bad settings, like storage open
  to the public or a security group that lets anything in
- **Hadolint** checks the Dockerfile for things like running as root or using an unpinned
  base image
- **terraform fmt and validate** catch formatting drift and syntax errors
- **Dependency Review** flags any package the pull request adds that has a known
  vulnerability

**Also on every push** (these report instead of blocking):

- **CodeQL** looks for code-level bugs like injection flaws
- **Trivy** scans the files and the built image for known CVEs, high and critical only
- **gitleaks** looks for API keys and passwords committed by mistake

Those three upload their results to the repo's **Security** tab rather than failing the
build. That split is on purpose. Scanners that produce occasional false positives should
inform you, not stop you, because a noisy check that blocks merges just gets switched off
after a week. The deterministic checks are the ones that gate.

**Weekly**, Dependabot opens pull requests for outdated GitHub Actions, npm packages, and
pip packages, capped at 10 open at a time.

## Releases

Pushing a tag like `v1.0.0` kicks off `release.yml`, which does five things:

1. Builds the image and pushes it to GHCR, with build provenance attached so you can trace
   where it came from
2. Generates an SBOM with Syft, which is a list of everything inside the image
3. Scans that image with Trivy and sends the results to the Security tab
4. Signs the image with **cosign in keyless mode**
5. Creates a GitHub Release with the SBOM attached

Anyone can check the signature:

```bash
cosign verify ghcr.io/adi-czobel/demo:latest \
  --certificate-identity-regexp "^https://github.com/Adi-Czobel/" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

**Why keyless signing.** The usual way to sign images is to keep a private key in a
repository secret. If that key ever leaks, an attacker can sign malicious images that pass
verification, and you would not know. Keyless signing skips the stored key entirely. The
signature is tied to a short-lived certificate proving it came from this repo's workflow,
and it gets recorded in a public log. There is no standing secret to steal.

## A few other choices

- **Workflow permissions start at read-only** and each job adds only what it needs. The
  default token has far more access than any of these jobs require.
- **gitleaks scans the full git history**, not just the latest changes (`fetch-depth: 0`).
  Deleting a secret in a later commit does not help, because it is still sitting in history
  and still works until someone rotates it.
- **Each check looks for its files first** before running, so these workflows can be dropped
  into a repo with no Terraform, or no Dockerfile, without showing red crosses for things
  that are not there.
- **The Kubernetes manifests include a default-deny NetworkPolicy.** Listing both `Ingress`
  and `Egress` flips the pod from allow-everything to deny-everything, and then specific
  rules add back what is needed: traffic from the same namespace on port 80, and outbound
  DNS and HTTPS. The outbound rules are still wider than they should be for a real workload,
  which is flagged in a comment in the file.

## Running the checks on your own machine

```bash
# Terraform
terraform -chdir=Terraform fmt -check -recursive
terraform -chdir=Terraform init -backend=false
terraform -chdir=Terraform validate

# Dockerfile
docker run --rm -i hadolint/hadolint < Docker/dockerfile

# Terraform and Kubernetes config check
docker run --rm -v "${PWD}:/tf" bridgecrew/checkov -d /tf --framework terraform kubernetes

# secrets, across the whole history
docker run --rm -v "${PWD}:/repo" zricethezav/gitleaks:latest detect \
  --source /repo --config /repo/.gitleaks.toml --verbose --redact

# known CVEs in the files
docker run --rm -v "${PWD}:/src" aquasec/trivy fs /src --severity HIGH,CRITICAL --ignore-unfixed
```

## Layout

```
.
├── .github/
│   ├── workflows/        ci, policy, security, release, badges
│   └── dependabot.yml
├── Terraform/main.tf     placeholder, gives the scanners something to read
├── Docker/dockerfile     Alpine stub, same reason
├── kubernetes/           namespace, serviceaccount, deployment, service, networkpolicy
└── .gitleaks.toml        secret scanning rules
```
