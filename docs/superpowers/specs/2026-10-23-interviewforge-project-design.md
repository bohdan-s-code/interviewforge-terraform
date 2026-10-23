# InterviewForge: Project Design

**Date:** 2026-10-23 (revised after grilling session, same day)
**Source:** [`terraform-learning-plan.md`](../../../terraform-learning-plan.md)
**Repository name:** `interviewforge-terraform` (public, personal GitHub account, non-commercial)

This document fixes the cross-cutting decisions for the whole learning project and defines what each phase delivers. Each phase still gets its own detailed spec and implementation plan, written when that phase starts, based on what actually exists by then.

---

## 1. Decisions

Reasons and rejected options live in the linked ADRs. Phase-specific decisions (module boundaries, secrets handling, state layout, policy tooling) get their own ADRs when their phase starts.

| Topic | Decision | ADR |
|---|---|---|
| Cloud and emulator | AWS, **fully emulated** on **MiniStack** (`localhost:4566`, `PERSIST_STATE=1`). Fallbacks: Floci, then LocalStack Hobby. | [ADR-0001](../../adr/0001-emulated-aws-on-ministack.md) (Proposed until Phase 0) |
| IaC tool and providers | Terraform **1.17.0** in `mise.toml`; `aws ~> 6.0`, `random ~> 3.7`, `docker ~> 4.0`, plus `local`, `archive`, `http`. | [ADR-0002](../../adr/0002-terraform-and-version-pins.md) |
| App | Node.js + TypeScript, Hono on Node server and Lambda (`nodejs22.x`), bundled with esbuild. | [ADR-0003](../../adr/0003-typescript-hono-dual-runtime.md) |
| Code evolution | One evolving codebase, git tag per phase (`phase-N-done`). | [ADR-0004](../../adr/0004-single-evolving-codebase.md) |
| Repo and CI | Public GitHub repo; GitHub Actions runs an ephemeral plan → apply → smoke → destroy per PR. | [ADR-0005](../../adr/0005-ephemeral-ci-integration.md) |

## 2. How the learner works

- **Claude writes all code**: Terraform, TypeScript, Makefile, CI.
- **The learner runs every `terraform plan` and `apply`** (own terminal or `! <cmd>` in Claude Code). Claude runs only builds and tests. Plan output is the main textbook.
- **Predict, then run:** before every infrastructure-changing `plan`, the learner writes the expected result (for example "1 to replace, image hash changed") in `docs/notes/phase-N.md`, then compares it with the real output.
- **"What to look at" notes:** each task in an implementation plan includes 2–4 lines pointing at the concept to study (for example "`for_each` keys become the resource address; try `terraform state list`"). Notes live in the plan, not in code comments, so the code stays idiomatic.
- The learner does all break-and-fix exercises by hand.

## 3. Architecture

### 3.1 Runtime

- **Phases 0–3:** no cloud. Terraform's `docker` provider manages a network, volumes and the `api` and `worker` containers.
- **Phases 4–6:** MiniStack runs from `compose.yaml` with a named data volume and is **not** managed by Terraform, because the `aws` provider needs a running endpoint before it can plan. Terraform manages every AWS resource inside it.
- Docker Desktop's "Use containerd for pulling and storing images" option is turned **off**; the docker provider's `docker_image` `build {}` is documented to fail with it on.

### 3.2 App

- One storage interface with two adapters: `fs` (Docker volume) and `aws` (S3 + DynamoDB). Environment variables set by Terraform select the adapter and its connection details.
- `npm run build` writes one bundle per entry point to `app/dist/<name>/index.js`. Terraform's `archive_file` zips Lambda bundles.
- The web UI is a single static HTML page. No UI polish.

### 3.3 End state on MiniStack

```
Browser → S3 static site (web UI; nginx container fallback, see Phase 5)
        → API Gateway (REST v1) → api Lambda ─┬→ S3 transcripts (KMS, CORS, versioning, lifecycle)
                                              ├→ DynamoDB plans + feedback (deletion-protected)
                                              ├→ Secrets Manager (login secret, api role only)
                                              └→ SQS feedback queue → feedback Lambda → DynamoDB
                                                      └→ DLQ
                                                 export Lambda → S3 exports, second region (same bucket module)
EventBridge schedule → cleanup Lambda
CloudWatch alarms (DLQ depth, Lambda errors) → SNS → SQS test subscriber
```

### 3.4 Repository layout (end state)

```
app/            TypeScript source: api, functions/{feedback,export,cleanup}, web/
infra/          Phase 1: flat root module → Phase 2: modules/ → Phase 5: envs/{dev,prod}
bootstrap/      Phase 5: state bucket + OIDC provider + CI role (local state, gitignored)
policy/         Phase 6: conftest (OPA) rules
sandbox/        Phase 0 throwaway configuration and emulator checklist
tests/smoke/    End-to-end script driven by `terraform output`
compose.yaml    MiniStack only (Phase 4+)
Makefile        up / build / plan / apply / smoke / env / stop / down
mise.toml       Pinned tool versions
.github/workflows/
docs/           specs, plans, notes/phase-N.md, runbook.md
```

### 3.5 Session commands

- `make up`: start MiniStack (data persists in its volume).
- `make stop`: stop MiniStack, **keep** all state. Use after every session.
- `make down`: `terraform destroy` for envs, then bootstrap; then delete the MiniStack data volume. Use at the end of each phase, or to recover from corrupted emulator data (`make down up`).

### 3.6 Tooling (all free)

terraform, tflint, Trivy, Checkov, terraform-docs, pre-commit, conftest, `awslocal` (AWS CLI pointed at `localhost:4566`), vitest. Infracost (free account and API key) is optional and local only.

### 3.7 Known emulator gaps

| Gap | Affected exercise | Substitute |
|---|---|---|
| IAM policies are not enforced | P4 "remove a permission and trace the failure" | `terraform test` assertion on the policy document fails instead |
| S3 lifecycle expiry is likely not executed | P4 Feature 7 | Assert lifecycle configuration in a test; P5 cleanup Lambda really deletes |
| OIDC authentication cannot be emulated | P5 CI auth | OIDC provider and CI role are applied in `bootstrap/` but CI authenticates with emulator test credentials |
| No AWS console | P5 drift exercise | Create drift with `awslocal` |
| CI runners start a fresh emulator per run | P5 CI | CI is an ephemeral integration test (see Phase 5) |
| S3 website hosting and Scheduler execution are unconfirmed in MiniStack | P5 Features 7 and 8 | Verified in Phase 0; fallbacks defined in Phase 5 |

## 4. Phases

Each phase ends with its exit check passing, notes written to `docs/notes/phase-N.md`, and the tag `phase-N-done`.

### Phase 0: Orientation (week 0)

- Install all tools through `mise.toml`.
- In `sandbox/phase-0/`, write a config using the `local` and `random` providers (`random_pet` → `local_file`).
- Run `init`, `plan`, `apply`, `destroy`; annotate the plan output.
- Edit the generated file by hand and observe drift in `plan`.
- **Emulator checklist** in `sandbox/emulator-check/`: a small Terraform config plus script against MiniStack proving each item works:
  S3 (KMS encryption, CORS, versioning, lifecycle config, static website hosting), SQS with DLQ redrive, `nodejs22.x` Lambda invoked from an SQS event source mapping, DynamoDB with `deletion_protection_enabled`, Secrets Manager with `secret_string_wo`, API Gateway REST v1 → Lambda, an EventBridge schedule that actually fires, CloudWatch alarm → SNS → SQS, two isolated accounts (access keys `111111111111` and `222222222222`), two regions, S3 backend with `use_lockfile`, and data surviving `make stop` / `make up`.
  Any failure: try Floci, then LocalStack Hobby. The endpoint stays `localhost:4566`, so switching only changes `compose.yaml`. Record results in notes.
- Confirm `docker_image` `build {}` with the containerd image store off (and note whether it fails with it on).
- **Exit check:** `docs/notes/phase-0.md` explains each command and each generated file (`.terraform/`, `.terraform.lock.hcl`, `terraform.tfstate`), and records the emulator checklist result.

### Phase 1: Core language (week 1)

`infra/` is one flat root module using the `docker` provider.

- **Image:** Terraform builds the app image with `docker_image` `build {}`; `triggers` holds a hash of `app/src`, so a code change shows as an image replacement. Notes explain why production pipelines build artifacts in CI instead.
- **Feature 1, interview plans:** plan templates come from a typed `map(object)` variable with validation on role and seniority. Each template is written as a JSON file by `local_file` and mounted read-only into the container. Implement with `count` over a list first, then refactor to `for_each`. Plans created through the API go to the data volume. Locals build container environment variables. Outputs expose `api_url`.
- **Feature 2, transcript upload:** the API stores files on a Docker volume.
- **Feature 3, feedback:** the API POSTs a job to the `worker` container over the Docker network; the worker writes stub feedback. The worker URL comes from Terraform (implicit dependency). Use `depends_on` where no reference exists.
- Terraform values reach the app through container environment variables; `make env` writes `terraform output -json` to `.env` for running outside Docker.
- **Break and fix:** `docker rm -f` a container and let Terraform recreate it; bump the Node base image tag to force replacement and explain why; switch `count` → `for_each` and read the address changes.
- **Exit check:** on a clean clone, `make build apply` then `curl "$(terraform -chdir=infra output -raw api_url)/plans"` returns plans.

### Phase 2: State, refactoring, modules (week 2)

- Extract `modules/service` (container, env, port, volume mounts), used by `api` and `worker`, and `modules/storage`.
- Refactor without `moved` first, read the destructive plan, then fix it with `moved` blocks.
- Create a volume with `docker volume create` and bring it under management with an `import` block.
- Stop managing one seed resource with a `removed` block, without destroying it.
- Add validation rules to module inputs; generate module READMEs with terraform-docs.
- Remove a resource from state with `terraform state rm` and re-import it.
- Find the Phase 1 sensitive variable in plaintext in `terraform.tfstate`; record it in notes (fixed in Phase 4).
- **Exit check:** the module refactor applies with zero create/destroy actions.

### Phase 3: Quality gates and testing (week 3)

- pre-commit hooks: `terraform fmt`, `validate`, tflint, Trivy, terraform-docs.
- `modules/*/tests/*.tftest.hcl` with `run` blocks and assertions; at least one test uses `mock_provider "docker"`.
- A precondition rejects a worker URL that does not point at the managed worker service.
- A `check` block uses the `http` data source to health-check the API after apply.
- `.github/workflows/ci.yml` runs all checks on every PR; branch protection requires it.
- Variable precedence exercise: `terraform.tfvars`, `*.auto.tfvars`, `TF_VAR_*`, `-var`. Record the observed order in notes.
- **Break and fix:** make a test fail on purpose; push unformatted code and confirm CI blocks the merge.
- **Exit check:** a PR with a formatting error or failing test cannot be merged.

### Phase 4: Cloud concepts on MiniStack (weeks 4–6)

One phase, three weeks. Every cloud feature is first wired to the **existing API container**, which now talks to MiniStack (`docker` and `aws` providers in one configuration). Moving the API to Lambda is the **last** task, so only one thing changes at a time.

- The `aws` provider uses explicit `endpoints {}` for MiniStack, plus `default_tags`.
- **Feature 2:** `modules/bucket` with public-access block, versioning, KMS encryption, CORS. Plan templates move to `aws_s3_object` per template (same `for_each`).
- **Feature 3:** SQS queue with DLQ redrive, event source mapping to the `feedback` Lambda. Each function has its own least-privilege role built with `aws_iam_policy_document`.
- **Feature 4:** DynamoDB table with `deletion_protection_enabled = true` and `lifecycle { prevent_destroy = true }`. Use data sources for account ID, region and policy documents.
- **Feature 5:** a second instance of `modules/bucket` for exports, created in a second region through the `aws.secondary` provider alias passed with `providers = {}` (framed as data residency). No replication.
- **Feature 6, login:** one Secrets Manager secret holds the password and an HMAC signing key. Its value comes from `ephemeral "random_password"` and the write-only `secret_string_wo` attribute, so it never reaches state. `secret_string_wo_version` is a plain number; rotation means bumping it. The UI sends the password; the API compares it in constant time and returns an HMAC-signed token. The learner reads the password with `awslocal secretsmanager get-secret-value`. Only the `api` role may read the secret.
- **Feature 7:** S3 lifecycle rule expiring transcripts; a test asserts the configuration.
- `tests/smoke/run.sh` runs plan → upload → feedback → export using `terraform output`.
- **Final task:** move the API to Lambda behind API Gateway (REST v1, never v2). The Docker containers leave management through `removed` blocks.
- **Break and fix:** remove a permission and see the policy test fail; send a poison message and confirm it lands in the DLQ; rename a resource that forces replacement, then block it with `prevent_destroy`.
- **Exit check:** `make up build apply smoke` passes; `grep` finds no secret value in state.

### Phase 5: "Production" on MiniStack (weeks 7–8)

- `bootstrap/` (local state, gitignored, applied once) creates a versioned, encrypted state bucket, the OIDC provider and the CI role.
- Migrate `infra/` state to the S3 backend in MiniStack with `use_lockfile = true` using `terraform init -migrate-state`. State persists across `make stop` / `make up`.
- `envs/dev` and `envs/prod` call the same modules. They are isolated as separate emulator accounts (access key ID `111111111111` for dev, `222222222222` for prod) and read bootstrap outputs through `terraform_remote_state`.
- **Feature 8:** web UI served from an S3 website bucket. If the Phase 0 checklist showed MiniStack cannot host websites, serve the HTML from S3 through an `nginx` container managed by the docker provider in the same configuration as `aws` (the plan's optional second provider).
- **Feature 7:** an EventBridge schedule triggers the `cleanup` Lambda, which deletes old transcripts. If Scheduler does not fire, use an EventBridge rule with `schedule_expression`.
- Naming: `if-<env>-<component>`. Tags: `project`, `env`, `managed_by` via `default_tags`.
- **Feature 9, CI:** every PR runs **plan → apply → smoke → destroy** against a fresh MiniStack service container, as an integration test. Merge to `main` runs the same flow for dev, then a prod job gated by a GitHub Environment with a required reviewer. Because CI state is ephemeral, PR plans always show a full create; notes record that a real diff-on-PR needs persistent shared state. Self-hosted runners are excluded: on a public repo they would run untrusted PR code locally.
- **Break and fix:** create drift with `awslocal` and detect it; run two applies at once and watch the lock; roll back a bad change with `git revert`; `make down`, then rebuild dev from scratch.
- **Exit check:** a merged PR runs dev in CI and prod waits for approval; locally `make down up` followed by bootstrap, dev apply and smoke succeeds.

### Phase 6: Production hardening (week 9)

- Fix every high-severity Trivy or Checkov finding, or suppress it inline with a written justification.
- Tag modules (`modules/bucket/v1.0.0`) and pin each environment with `?ref=`.
- Read `terraform-aws-modules/s3-bucket` and `terraform-aws-modules/lambda`; record differences from own modules in notes.
- **Feature 10:** CloudWatch log groups with retention; alarms on DLQ depth and Lambda errors that publish to SNS, with an SQS subscriber to verify alert delivery.
- Optional: run Infracost locally against a plan (real AWS prices, for cost awareness only). Not in CI.
- One conftest rule against `terraform show -json` plan output: every S3 bucket is encrypted and carries the required tags.
- `docs/runbook.md` covering bootstrap, deploy, rollback, destroy and secret rotation.
- **Exit check:** following only `docs/runbook.md` on a fresh clone produces a working dev environment.

### Phase 7 (optional): Make it real (week 10+)

- Replacing the stub with a real AI provider is the only step that costs money; the stub stays the default.
- Add real interview plan templates; decide on real candidate data and harden first if yes; write the article.

## 5. Testing strategy

| Layer | From | What it checks |
|---|---|---|
| `terraform validate` + tflint | P1 | Syntax and provider-specific mistakes |
| `terraform test` + mocked providers | P3 | Module logic without infrastructure; P4 adds IAM-policy and lifecycle assertions for emulator gaps |
| `check` blocks | P3 | Live health after apply |
| vitest unit tests | P1 | Storage adapters, feedback stub, token signing |
| Smoke script | P4 | Full flow end to end via outputs |
| CI integration run | P5 | plan → apply → smoke → destroy on every PR |
| Trivy / Checkov / conftest | P3 / P6 | Security and policy |

## 6. Ground rules (from the learning plan, adapted)

1. Synthetic data only.
2. Change infrastructure only through Terraform; use `awslocal` only to look, except in drift exercises.
3. `make stop` after each session; `make down` at the end of each phase.
4. Commit after every exercise; tag `phase-N-done` at each exit check.
5. Keep the app minimal.
6. Session split: about 20% learning, 50% building, 30% breaking and fixing.
7. Predict every infrastructure-changing plan before running it.

## 7. Out of scope

Real AWS accounts, working OIDC authentication, self-hosted CI runners, commercial use, UI polish beyond one HTML page.
