# InterviewForge: Project Design

**Date:** 2026-10-23
**Source:** [`terraform-learning-plan.md`](../../../terraform-learning-plan.md)
**Repository name:** `interviewforge-terraform` (public, GitHub)

This document fixes the cross-cutting decisions for the whole learning project and defines what each phase delivers. Each phase still gets its own detailed spec and implementation plan, written when that phase starts, based on what actually exists by then.

---

## 1. Decisions

| Topic | Decision | Reason |
|---|---|---|
| Cloud | AWS, **fully emulated**. No real AWS account is used. | Zero cost and zero risk. |
| Emulator | LocalStack (free edition) in Docker. Fallback: Moto server. | Covers S3, SQS, Lambda, DynamoDB, Secrets Manager, KMS, IAM, EventBridge, API Gateway v1, CloudWatch, SNS. |
| IaC tool | Terraform, latest 1.x (≥ 1.11), pinned in `mise.toml`. | Write-only attributes and ephemeral resources need ≥ 1.11. |
| App | Node.js + TypeScript, Hono framework, bundled with esbuild. | Hono runs as a Node server (Phases 1–3) and as a Lambda handler (Phases 4+) with the same code. |
| Repo / CI | Public GitHub repo, GitHub Actions. | Free minutes and free Environment protection rules on public repos. Data is synthetic. |
| Code evolution | One evolving codebase, git tag per phase (`phase-N-done`). | Phase 2 refactors code that Phase 1 really applied, so `moved`, import and state lessons are real. |

## 2. Architecture

### 2.1 Runtime

- **Phases 0–3:** no cloud. Terraform's `docker` provider (`kreuzwerker/docker`) manages a network, volumes and the `api` and `worker` containers.
- **Phases 4–6:** LocalStack runs from `compose.yaml` and is **not** managed by Terraform, because the `aws` provider needs a running endpoint before it can plan. Terraform manages every AWS resource inside it.

### 2.2 App

- One storage interface with two adapters: `fs` (Docker volume) and `aws` (S3 + DynamoDB). Environment variables set by Terraform select the adapter and its connection details.
- `npm run build` writes one bundle per entry point to `app/dist/<name>/index.js`. Terraform's `archive_file` zips Lambda bundles.
- The web UI is a single static HTML page. No UI polish.

### 2.3 End state on LocalStack

```
Browser → S3 static site (web UI)
        → API Gateway (REST v1) → api Lambda ─┬→ S3 transcripts (KMS, CORS, versioning, lifecycle)
                                              ├→ DynamoDB plans + feedback (deletion-protected)
                                              ├→ Secrets Manager (login secret, api role only)
                                              └→ SQS feedback queue → feedback Lambda → DynamoDB
                                                      └→ DLQ
                                                 export Lambda → S3 exports (same bucket module)
EventBridge schedule → cleanup Lambda
CloudWatch alarms (DLQ depth, Lambda errors) → SNS → SQS test subscriber
```

### 2.4 Repository layout (end state)

```
app/            TypeScript source: api, functions/{feedback,export,cleanup}, web/
infra/          Phase 1: flat root module → Phase 2: modules/ → Phase 5: envs/{dev,prod}
bootstrap/      Phase 5: state bucket + OIDC provider + CI role (local state)
policy/         Phase 6: conftest (OPA) rules
sandbox/        Phase 0 throwaway configuration
tests/smoke/    End-to-end script driven by `terraform output`
compose.yaml    LocalStack only (Phase 4+)
Makefile        up / build / plan / apply / smoke / env / down
mise.toml       Pinned tool versions
.github/workflows/
docs/           specs, plans, notes/phase-N.md, runbook.md
```

### 2.5 Tooling (all free)

terraform, tflint, Trivy, Checkov, terraform-docs, pre-commit, conftest, Infracost (free API key), `awslocal`, vitest.

### 2.6 Known emulator gaps

| Gap | Affected exercise | Substitute |
|---|---|---|
| IAM policies are not enforced | P4 "remove a permission and trace the failure" | `terraform test` assertion on the policy document fails instead |
| S3 lifecycle expiry is not executed | P4 Feature 7 | Assert lifecycle configuration in a test; P5 cleanup Lambda really deletes |
| OIDC authentication cannot be emulated | P5 CI auth | OIDC provider and CI role are applied in `bootstrap/` but CI authenticates with LocalStack test credentials |
| No AWS console | P5 drift exercise | Create drift with `awslocal` |
| LocalStack state is in-memory (persistence is a paid feature) | P5 remote state, every session | Stopping the container wipes all resources, including the state bucket. `make down` runs `terraform destroy` for each root (envs, then bootstrap) before stopping LocalStack; `make up` rebuilds everything from scratch. CI is ephemeral for the same reason. |

## 3. Phases

Each phase ends with its exit check passing, notes written to `docs/notes/phase-N.md`, and the tag `phase-N-done`.

### Phase 0: Orientation

- Install all tools through `mise.toml`.
- In `sandbox/phase-0/`, write a config using the `local` and `random` providers (`random_pet` → `local_file`).
- Run `init`, `plan`, `apply`, `destroy`; annotate the plan output.
- Edit the generated file by hand and observe drift in `plan`.
- Verify the risks in section 5.
- **Exit check:** `docs/notes/phase-0.md` explains each command and each generated file (`.terraform/`, `.terraform.lock.hcl`, `terraform.tfstate`).

### Phase 1: Core language

`infra/` is one flat root module using the `docker` provider.

- **Feature 1, interview plans:** plan templates come from a typed `map(object)` variable with validation on role and seniority. Implement with `count` over a list first, then refactor to `for_each`. Locals build container environment variables. Outputs expose `api_url`.
- **Feature 2, transcript upload:** the API stores files on a Docker volume.
- **Feature 3, feedback:** the API POSTs a job to the `worker` container over the Docker network; the worker writes stub feedback. The worker URL comes from Terraform (implicit dependency). Use `depends_on` where no reference exists.
- Terraform values reach the app through container environment variables; `make env` writes `terraform output -json` to `.env` for running outside Docker.
- **Break and fix:** `docker rm -f` a container and let Terraform recreate it; bump the Node base image tag to force replacement and explain why; switch `count` → `for_each` and read the address changes.
- **Exit check:** on a clean clone, `make build apply` then `curl "$(terraform -chdir=infra output -raw api_url)/plans"` returns plans.

### Phase 2: State, refactoring, modules

- Extract `modules/service` (container, env, port, volume mounts), used by `api` and `worker`, and `modules/storage`.
- Refactor without `moved` first, read the destructive plan, then fix it with `moved` blocks.
- Create a volume with `docker volume create` and bring it under management with an `import` block.
- Stop managing one seed resource with a `removed` block, without destroying it.
- Add validation rules to module inputs; generate module READMEs with terraform-docs.
- Remove a resource from state with `terraform state rm` and re-import it.
- Find the Phase 1 sensitive variable in plaintext in `terraform.tfstate`; record it in notes (fixed in Phase 4).
- **Exit check:** the module refactor applies with zero create/destroy actions.

### Phase 3: Quality gates and testing

- pre-commit hooks: `terraform fmt`, `validate`, tflint, Trivy, terraform-docs.
- `modules/*/tests/*.tftest.hcl` with `run` blocks and assertions; at least one test uses `mock_provider "docker"`.
- A precondition rejects a worker URL that does not point at the managed worker service.
- A `check` block uses the `http` data source to health-check the API after apply.
- `.github/workflows/ci.yml` runs all checks on every PR; branch protection requires it.
- Variable precedence exercise: `terraform.tfvars`, `*.auto.tfvars`, `TF_VAR_*`, `-var`. Record the observed order in notes.
- **Break and fix:** make a test fail on purpose; push unformatted code and confirm CI blocks the merge.
- **Exit check:** a PR with a formatting error or failing test cannot be merged.

### Phase 4: Cloud concepts on LocalStack

- The `aws` provider uses explicit `endpoints {}` for LocalStack, plus `default_tags`.
- The API moves to Lambda behind API Gateway (REST v1). The Docker containers leave management through `removed` blocks.
- **Feature 2:** `modules/bucket` with public-access block, versioning, KMS encryption, CORS.
- **Feature 3:** SQS queue with DLQ redrive, event source mapping to the `feedback` Lambda. Each function has its own least-privilege role built with `aws_iam_policy_document`.
- **Feature 4:** DynamoDB table with `deletion_protection_enabled = true` and `lifecycle { prevent_destroy = true }`. Use data sources for account ID, region and policy documents.
- **Feature 5:** a second instance of `modules/bucket` for exports, with a replica bucket in a second region through a provider alias passed with `providers = {}`.
- **Feature 6:** Secrets Manager secret whose value comes from `ephemeral "random_password"` and the write-only `secret_string_wo` attribute, so the secret never reaches state. Only the `api` role may read it.
- **Feature 7:** S3 lifecycle rule expiring transcripts; a test asserts the configuration.
- `tests/smoke/run.sh` runs plan → upload → feedback → export using `terraform output`.
- **Break and fix:** remove a permission and see the policy test fail; send a poison message and confirm it lands in the DLQ; rename a resource that forces replacement, then block it with `prevent_destroy`.
- **Exit check:** `make up build apply smoke` passes; `grep` finds no secret value in state.

### Phase 5: "Production" on LocalStack

- `bootstrap/` (local state, applied once) creates a versioned, encrypted state bucket, the OIDC provider and the CI role.
- Migrate `infra/` state to the S3 backend with `use_lockfile = true` using `terraform init -migrate-state`.
- `envs/dev` and `envs/prod` call the same modules. They are isolated as separate LocalStack accounts (access key ID `111111111111` for dev, `222222222222` for prod) and read bootstrap outputs through `terraform_remote_state`.
- **Feature 8:** web UI served from an S3 website bucket.
- **Feature 7:** an EventBridge schedule triggers the `cleanup` Lambda, which deletes old transcripts.
- Naming: `if-<env>-<component>`. Tags: `project`, `env`, `managed_by` via `default_tags`.
- **Feature 9, CI:** plan for dev and prod on every PR (posted as a PR comment); auto-apply dev on merge to `main`; prod apply gated by a GitHub Environment with a required reviewer.
- **CI is ephemeral:** GitHub-hosted runners start a fresh LocalStack service container per run, so each CI run bootstraps and applies from scratch. This continuously proves destroy-and-rebuild works. Locally, dev and prod persist only while the LocalStack container runs (see section 2.6). Self-hosted runners are excluded: on a public repo they would run untrusted PR code locally.
- **Break and fix:** create drift with `awslocal` and detect it; run two applies at once and watch the lock; roll back a bad change with `git revert`; destroy dev completely and rebuild.
- **Exit check:** a merged PR applies dev in CI, prod waits for approval, and `make down apply smoke` for dev succeeds locally.

### Phase 6: Production hardening

- Fix every high-severity Trivy or Checkov finding, or suppress it inline with a written justification.
- Tag modules (`modules/bucket/v1.0.0`) and pin each environment with `?ref=`.
- Read `terraform-aws-modules/s3-bucket` and `terraform-aws-modules/lambda`; record differences from own modules in notes.
- **Feature 10:** CloudWatch log groups with retention; alarms on DLQ depth and Lambda errors that publish to SNS, with an SQS subscriber to verify alert delivery.
- Infracost comment on PRs (real AWS prices, for cost awareness only).
- One conftest rule against `terraform show -json` plan output: every S3 bucket is encrypted and carries the required tags.
- `docs/runbook.md` covering bootstrap, deploy, rollback, destroy and secret rotation.
- **Exit check:** following only `docs/runbook.md` on a fresh clone produces a working dev environment.

### Phase 7 (optional): Make it real

- Replacing the stub with a real AI provider is the only step that costs money; the stub stays the default.
- Add real interview plan templates; decide on real candidate data and harden first if yes; write the article.

## 4. Testing strategy

| Layer | From | What it checks |
|---|---|---|
| `terraform validate` + tflint | P1 | Syntax and provider-specific mistakes |
| `terraform test` + mocked providers | P3 | Module logic without infrastructure; P4 adds IAM-policy and lifecycle assertions for emulator gaps |
| `check` blocks | P3 | Live health after apply |
| vitest unit tests | P1 | Storage adapters and feedback stub only |
| Smoke script | P4 | Full flow end to end via outputs |
| Trivy / Checkov / conftest | P3 / P6 | Security and policy |

## 5. Risks to verify in Phase 0

1. **LocalStack licensing:** confirm the current free edition and whether it needs an account or auth token. If unacceptable, switch to Moto server (endpoint change only).
2. **LocalStack free-edition coverage** of API Gateway v1, EventBridge Scheduler and KMS. If Scheduler is missing, use an EventBridge rule with `schedule_expression`.
3. **Terraform ≥ 1.11** is available through `mise`.

## 6. Ground rules (from the learning plan, adapted)

1. Synthetic data only.
2. Change infrastructure only through Terraform; use `awslocal` only to look, except in drift exercises.
3. End each session with `make down` (destroy, then stop LocalStack); start the next with `make up`.
4. Commit after every exercise; tag `phase-N-done` at each exit check.
5. Keep the app minimal.
6. Session split: about 20% learning, 50% building, 30% breaking and fixing.

## 7. Out of scope

Real AWS accounts, working OIDC authentication, self-hosted CI runners, UI polish beyond one HTML page.
