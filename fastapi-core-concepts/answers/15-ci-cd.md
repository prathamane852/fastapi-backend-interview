# 15 — CI/CD

---

## 📖 Vocabulary

> *Definition* is the precise meaning; *Say it in an interview* is the quotable one-liner.

| Term | Definition | Say it in an interview |
|---|---|---|
| **Continuous Integration (CI)** | Practice of merging changes frequently with automated build and test on every push to detect integration problems early. | "CI validates every change automatically before merge." |
| **Continuous Delivery vs Deployment (CD)** | Delivery: every passing build is deployable with a manual release step. Deployment: every passing build is deployed automatically. | "Delivery keeps a manual gate; deployment removes it." |
| **Pipeline** | The automated, ordered sequence of stages a change passes through from commit to production. | "The pipeline is the codified path to production." |
| **Stage / job** | A stage groups jobs that run in parallel; stages execute sequentially. | "Jobs within a stage run in parallel; stages are sequential." |
| **Artifact** | A file produced by a pipeline job (image, report, SBOM) persisted for later stages or inspection. | "Build once, promote the same artifact everywhere." |
| **Immutable artifact** | An artifact that is never rebuilt after creation; the identical image is promoted through environments. | "Immutability means prod runs exactly what was tested." |
| **Linting** | Static analysis for style violations and simple errors (ruff, flake8). | "Lint is the cheapest, fastest gate." |
| **Type checking** | Static verification of type annotations for consistency (mypy, pyright). | "mypy catches type errors before runtime." |
| **SAST** | *Static Application Security Testing*: analyzing source for vulnerability patterns (bandit, Semgrep). | "SAST finds insecure code patterns statically." |
| **Dependency (SCA) scanning** | *Software Composition Analysis*: checking third-party packages against vulnerability databases (pip-audit, Snyk). | "SCA blocks known-vulnerable dependencies." |
| **CVE** | *Common Vulnerabilities and Exposures*: a public identifier for a disclosed vulnerability, scored by CVSS severity. | "Gate on HIGH and CRITICAL CVEs with available fixes." |
| **SBOM** | *Software Bill of Materials*: a machine-readable inventory of all components in an artifact (CycloneDX, SPDX). | "An SBOM lets me answer 'are we affected' the day a CVE drops." |
| **Image signing (cosign)** | Cryptographically signing an image digest so consumers can verify provenance; part of Sigstore. | "Signed images plus an admission policy stop untrusted images running." |
| **Provenance / SLSA** | Attestations describing how an artifact was built; SLSA defines maturity levels for supply-chain integrity. | "Provenance proves the image came from our pipeline." |
| **Admission controller / policy engine** | Kubernetes component validating or mutating objects on creation (Kyverno, OPA Gatekeeper). | "Admission policies enforce signed images and no `latest` tags." |
| **Secret scanning** | Detecting credentials committed to source (gitleaks, trufflehog). | "Secret scanning on every push prevents leaked keys." |
| **GitOps** | Declarative desired state in git, reconciled continuously by an in-cluster controller (ArgoCD, Flux). CI writes to git; CD is the controller. | "CI updates a manifest; ArgoCD does the deploy." |
| **ArgoCD `Application`** | The CRD binding a git path (source) to a cluster/namespace (destination) with a sync policy. | "An Application maps a repo path to a namespace." |
| **Sync hooks (PreSync/PostSync)** | ArgoCD annotations running Jobs before or after a sync, e.g., migrations before rollout, smoke tests after. | "Migrations run as a PreSync hook." |
| **Kustomize overlay** | Environment-specific patches applied on a shared base to produce final manifests. | "Overlays hold per-environment differences." |
| **Helm chart / values** | A packaged set of templated manifests parameterized by a values file per environment. | "Charts template manifests; values vary per env." |
| **Blue-green deployment** | Two identical environments; deploy to the idle one, verify, then switch traffic atomically; instant rollback by switching back. | "Blue-green is instant cutover at double capacity." |
| **Canary deployment** | Gradually shifting a small, increasing fraction of traffic to the new version while comparing metrics; abort on regression. | "Canary limits blast radius with automated analysis." |
| **Progressive delivery** | Umbrella for canary/blue-green with automated metric analysis and rollback (Argo Rollouts, Flagger). | "Progressive delivery automates safe rollouts." |
| **Rollback vs roll forward** | Rollback: return to the previous version. Roll forward: ship a fix on top. Prefer roll forward for DB-affecting changes. | "Roll back code freely; roll forward on schema." |
| **Compensating migration** | A new forward migration that safely undoes a previous change, avoiding destructive downgrades. | "Undo schema with a new migration, not a downgrade." |
| **OIDC federation (CI → cloud)** | CI jobs present a short-lived identity token to the cloud provider to assume a role, eliminating stored credentials. | "OIDC federation removes long-lived cloud keys from CI." |
| **Kaniko / BuildKit rootless** | Tools building images without a privileged Docker daemon, safer for shared runners. | "Kaniko builds images in CI without a Docker socket." |
| **Protected branch / environment** | Repository settings restricting who can push/deploy and requiring approvals. | "Prod deploys only from protected branches with approval." |
| **Masked variable** | A CI variable whose value is hidden in job logs. | "Mask secrets so they never appear in logs." |
| **Smoke test (post-deploy)** | A minimal automated check after deployment; failure triggers rollback. | "Smoke tests gate promotion." |
| **Feature flag** | Runtime toggle separating deploy from release. | "Flags let me deploy dark and release later." |
| **Renovate / Dependabot** | Bots opening automated dependency-update PRs. | "Automated bump PRs keep dependencies patched." |
| **Lockfile with hashes** | Dependency lock pinning exact versions and artifact hashes to prevent substitution. | "Hash-pinned lockfiles defend against tampered packages." |

---

## Q1. How would you build a CI/CD pipeline for FastAPI?

Principles: fast feedback (< 5 min for lint/unit), one immutable artefact promoted through environments, everything as code, blocking security gates, deploy via GitOps.

```
PR opened ──► lint + type-check ──► unit tests ──► integration tests (DB/Redis services)
          ──► dependency & secret scan ──► build image (multi-stage, SBOM) ──► image scan
          ──► push :sha to registry ──► update dev overlay in GitOps repo ──► ArgoCD syncs dev
          ──► smoke tests vs dev ──► [approval] ──► promote same tag to staging ──► e2e
          ──► promote to prod (canary) ──► smoke + SLO check ──► auto-rollback on failure
```
```yaml
# .gitlab-ci.yml (abridged; the `test` job is in file 12)
stages: [lint, test, security, build, scan, push, deploy-dev, verify, deploy-prod]

.python: { image: python:3.12-slim, cache: { key: { files: [uv.lock] }, paths: [.venv/] } }

lint:
  extends: .python
  stage: lint
  script: [pip install ruff mypy, ruff check ., ruff format --check ., mypy app]

deps-scan:
  extends: .python
  stage: security
  script: [pip install pip-audit, pip-audit -r requirements.txt --strict]

secret-scan:
  stage: security
  image: zricethezav/gitleaks:latest
  script: [gitleaks detect --source . --no-git -v]

sast:
  extends: .python
  stage: security
  script: [pip install bandit, bandit -r app -ll]

build:
  stage: build
  image: gcr.io/kaniko-project/executor:debug
  script:
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile Dockerfile
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --cache=true
        --label org.opencontainers.image.revision=$CI_COMMIT_SHA

image-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - trivy image --format cyclonedx --output sbom.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  artifacts: { paths: [sbom.json] }

sign:
  stage: push
  image: gcr.io/projectsigstore/cosign:latest
  script: [cosign sign --yes $CI_REGISTRY_IMAGE@$IMAGE_DIGEST]

deploy-dev:
  stage: deploy-dev
  image: alpine/git
  script:
    - git clone https://oauth2:${GITOPS_TOKEN}@gitlab.com/org/gitops.git && cd gitops
    - cd apps/orders-api/overlays/dev && kustomize edit set image orders-api=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - git commit -am "orders-api dev → $CI_COMMIT_SHA" && git push
  rules: [{ if: $CI_COMMIT_BRANCH == "main" }]

smoke-dev:
  stage: verify
  script: [./scripts/smoke.sh https://dev.api.example.com]

deploy-prod:
  stage: deploy-prod
  script: [same as deploy-dev, but overlays/prod]
  when: manual
  environment: { name: production, url: https://api.example.com }
```
### 🔍 Job by job
- `.python` — a hidden template job others `extends`; caches the venv keyed on the lockfile.
- `lint` — ruff (lint + format) and mypy. Cheap, first.
- `deps-scan` — `pip-audit --strict` fails on known-vulnerable packages.
- `secret-scan` — gitleaks blocks committed secrets.
- `sast` — bandit finds insecure code patterns (`-ll` = medium+ severity).
- `build` — Kaniko builds rootless in CI (no Docker socket); tags with the commit SHA; labels the image with its source revision.
- `image-scan` — Trivy fails on unfixed HIGH/CRITICAL CVEs; also emits a CycloneDX SBOM.
- `sign` — cosign signs by **digest**; the cluster's admission policy refuses unsigned images.
- `deploy-dev` — CI's only "deploy" action is editing the image tag in the GitOps repo; ArgoCD does the actual apply.
- `smoke-dev` — post-deploy verification.
- `deploy-prod` — manual gate (or automatic with canary analysis); `environment` gives GitLab's deployment tracking.

## Q2. What checks should happen before deployment?

Matching the diagram in the question:
- **Lint / format / type-check** — cheap, catches trivial bugs.
- **Unit tests** — logic.
- **Integration tests** — SQL, wiring, migrations apply.
- **Security scan** — deps (pip-audit), secrets (gitleaks), SAST (bandit/Semgrep), IaC (checkov).
- **Build Docker image** — reproducible, pinned base.
- **Image scan** — Trivy/Grype; SBOM; signature.
- **Push registry** — immutable `:sha` tag, signed.
- **Deploy** — dev → staging → prod.
- **Smoke tests** — health, auth, one critical write; rollback on failure.
Also: OpenAPI contract diff (no breaking changes), licence check, coverage floor, migration dry-run (`alembic upgrade head --sql`), load test on staging for risky changes.

---

## Q3. How do you manage environment-specific configuration?

- Same image everywhere; only configuration differs.
- **Kustomize overlays** (`base/` + `overlays/{dev,staging,prod}`) or **Helm values** per env, in the GitOps repo.
- Non-secret config → `ConfigMap`; secrets → External Secrets referencing per-env paths (`/dev/orders-api`, `/prod/orders-api`).
- App validates config at startup; avoid `if env == "prod"` sprinkled through code.
- Feature flags (LaunchDarkly/Unleash) for runtime toggles, not env-specific builds.

## Q4. How do you perform database migrations during deployment?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q4–Q5 — expand-contract migrations and where they run in the pipeline.

---

## Q5. How do you rollback application code?

Revert the GitOps commit (image tag back to last-good) → ArgoCD syncs. Emergency: `kubectl rollout undo` / `helm rollback`, then reconcile git. Because migrations are backward-compatible by policy, the previous code works against the new schema. Automate: failed canary analysis or smoke test triggers the revert.

## Q6. How do you rollback database changes?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q5 — why you mostly *can't* roll back a migration.

---

## Q7. How would you implement blue-green deployment?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q3 — the strategy comparison, and why none of them protect you from the database.

---

## Q8. What is canary deployment?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q8 — automated canary analysis with Argo Rollouts.

---

## Q9. How would you deploy FastAPI using ArgoCD?

> 🔴 **Moved to the advanced section** → [../../advanced-backend/07-deployments-and-release.md](../../advanced-backend/07-deployments-and-release.md)
> 
> Q7 — what GitOps actually buys you, and its honest trade-offs.

---

## Q10. How would you manage secrets using AWS Secrets Manager?

- One JSON secret per service per environment: `prod/orders-api` = `{database_url, jwt_secret, …}`.
- Pods get an IAM role via **IRSA**; the policy allows `secretsmanager:GetSecretValue` on that ARN only.
- Delivery: External Secrets Operator (syncs to a K8s Secret) or Secrets Store CSI (mounts files). Or the app fetches at startup with `aiobotocore` — simplest, but rotation requires a restart.
- **Rotation**: Secrets Manager rotation Lambda for RDS creds; ESO picks up new values on `refreshInterval`; a Reloader annotation restarts pods on change. Better: IAM DB auth / RDS Proxy — no password at all.
- Audit via CloudTrail; customer-managed KMS key; VPC endpoint only.

## Q11. How would you handle configuration changes without rebuilding the image?

Config lives outside the image: ConfigMap/Secret change → pod restart (`kubectl rollout restart` or Stakater Reloader on checksum change). Live changes without restart: mount the ConfigMap as a file and watch it, or use a dynamic-config/feature-flag service (AWS AppConfig, LaunchDarkly, Consul). Validate on load so a bad change fails readiness rather than crashing silently.

---

## Q12. How would you secure the CI/CD pipeline?

The pipeline holds prod credentials — a prime target.
- **Ephemeral, isolated runners**; no shared Docker socket (Kaniko / rootless BuildKit).
- **Short-lived credentials**: OIDC federation from GitLab/GitHub to AWS — no stored keys.
- **Protected branches & environments**: prod deploys only from `main` with approvals; masked/protected variables.
- **Pin actions/images by digest**; scan the pipeline's own dependencies.
- **Signed commits**; **signed images** verified by admission policy; SLSA provenance.
- **Separate CI from CD**: CI pushes images; only in-cluster ArgoCD deploys; CI never holds a prod kubeconfig.
- **Secret scanning** on every push; audit logs; protect `.gitlab-ci.yml` itself.
- No fork pipelines with secret access.

## Q13. How do you scan Python dependencies?

- `pip-audit` (PyPA; OSV/PyPI advisories) against requirements/lockfile.
- `safety`, Snyk, Dependabot / GitLab Dependency Scanning, Renovate for auto-bump PRs.
- Generate an SBOM (`cyclonedx-py`) and monitor continuously — new CVEs appear after you ship.
- Lockfile with hashes (`--require-hashes`); a private PyPI proxy (Artifactory/Nexus).

## Q14. How do you scan Docker images?

Trivy / Grype / Snyk Container / ECR native scanning: OS packages, language deps, misconfigurations; fail on HIGH/CRITICAL with a fix available; ignore-file with expiry for accepted risks; rescan images already in the registry on a schedule (CVEs are discovered later); rebuild base images weekly.

## Q15. How do you prevent vulnerable dependencies from reaching production?

Layered:
1. **Pre-merge**: dependency scan on every PR; Renovate auto-PRs; lockfiles.
2. **Build**: image scan gate; SBOM; signature.
3. **Registry**: continuous rescanning; block pulls of images with new critical CVEs.
4. **Cluster admission**: Kyverno/Gatekeeper — only signed images from our registry, no `latest`, SBOM attestation required, block critical CVEs.
5. **Runtime**: Trivy Operator reports on running workloads; scheduled base-image rebuilds.
6. **Process**: patch SLAs by severity; documented exceptions with expiry.
