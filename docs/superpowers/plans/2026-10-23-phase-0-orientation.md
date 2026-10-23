# Phase 0: Orientation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install the pinned toolchain, run the core Terraform workflow end to end on a local-only config, and prove (or disprove) that MiniStack supports everything the later phases rely on, so ADR-0001 can be accepted or superseded.

**Architecture:** Three throwaway configurations under `sandbox/`: `phase-0/` (local + random providers, the learning exercise), `emulator-check/` (one Terraform config per AWS feature the spec relies on, plus a probe script), and `docker-build-check/` (the `docker_image` `build {}` check). MiniStack runs from a root `compose.yaml`, driven by `make up / stop / down`.

**Tech Stack:** Terraform 1.17.0, mise, MiniStack, Docker Desktop, AWS CLI v2, bash, plain Node.js 22 ESM (no build step in the sandbox).

**Spec:** `docs/superpowers/specs/2026-10-23-interviewforge-project-design.md` (Phase 0, sections 2, 3.5, 3.7) and `docs/adr/0001-emulated-aws-on-ministack.md`.

## Global Constraints

- Terraform `1.17.0`, pinned in `mise.toml`.
- Providers: `hashicorp/aws ~> 6.0`, `hashicorp/random ~> 3.7`, `kreuzwerker/docker ~> 4.0`, `hashicorp/local ~> 2.5`, `hashicorp/archive ~> 2.7`.
- Emulator endpoint `http://localhost:4566`; MiniStack runs with `PERSIST_STATE=1`.
- Emulator accounts: dev access key ID `111111111111`, prod `222222222222`; secret key `test` (a dummy value the emulator ignores; not a credential).
- Regions: primary `eu-central-1`, secondary `eu-west-1`.
- Resource names in the sandbox start with `if-check-`.
- Lambda runtime `nodejs22.x`.
- API Gateway REST v1 only, never v2.
- **Learner workflow (spec section 2):** Claude writes all code and runs builds, `fmt`, `validate`, `terraform test` and probe scripts. The **learner** runs every `terraform init / plan / apply / destroy`. Before every infrastructure-changing `plan`, the learner writes a prediction in `docs/notes/phase-0.md`, then records the actual result next to it.
- No `*.tfstate`, `.terraform/` or generated `out/` files are ever committed.

## Review Focus

1. **Emulator not running** when a probe runs: the probe must fail fast with `emulator not reachable at http://localhost:4566`, not a wall of AWS CLI errors. Pinned in Task 3 (`preflight` test).
2. **Asynchronous effects** (SQS → Lambda, DLQ, schedules, alarms): a single immediate read gives false FAILs. Probes poll with a timeout (90 s default). Pinned in Task 5.
3. **Re-applying the same config** must show `No changes.`: an emulator that reports perpetual diffs would make every later plan unreadable. Pinned in Task 4 and Task 5 (second plan step).
4. **The containerd image store is still on**: the docker build check must record the result in both states, not assume one. Pinned in Task 7.
5. **The secret value leaking into state** through `secret_string_wo`: the probe greps local state for the live secret value. Pinned in Task 4.

---

## File Structure

```
mise.toml                                   pinned tools
Makefile                                    up / stop / down / check
compose.yaml                                MiniStack service + named volume
bin/awslocal                                AWS CLI wrapper pointed at the emulator
.gitignore                                  (modify) add sandbox outputs
docs/notes/phase-0.md                       learner notes: predictions, findings, checklist results
docs/adr/0001-emulated-aws-on-ministack.md  (modify) status after the checklist
sandbox/phase-0/
  versions.tf  main.tf  variables.tf  outputs.tf
  tests/greeting.tftest.hcl
sandbox/emulator-check/
  check.sh              probe runner: ./check.sh <group|all>
  lib.sh                preflight, check, soft_check, wait_for helpers
  versions.tf           terraform + provider constraints
  providers.tf          default (dev account, eu-central-1), aws.secondary, aws.prod
  storage.tf            S3 + KMS + DynamoDB + Secrets Manager + accounts + regions
  compute.tf            SQS/DLQ + Lambda + API Gateway + schedules + alarm/SNS
  outputs.tf
  lambda/index.mjs      probe handler
  backend-check/main.tf S3 backend with use_lockfile
sandbox/docker-build-check/
  main.tf  Dockerfile
```

---

### Task 1: Toolchain

**Files:**
- Create: `mise.toml`, `bin/awslocal`
- Modify: `.gitignore`

**Interfaces:**
- Produces: `terraform` 1.17.0 on PATH inside the repo; `bin/awslocal <aws-cli-args>` (defaults: endpoint `http://localhost:4566`, access key `111111111111`, secret `test`, region `eu-central-1`; each overridable through the standard `AWS_*` env vars).

**What to look at:** how mise activates per directory; `terraform version` reports the platform and whether a newer version exists.

- [ ] **Step 1: Install mise** (learner, once): `brew install mise`, then add `eval "$(mise activate zsh)"` to `~/.zshrc` and open a new terminal.
- [ ] **Step 2: Write `mise.toml`** with `[tools]`: `terraform = "1.17.0"`, plus `tflint`, `trivy`, `terraform-docs`, `pre-commit`, `conftest`, `"pipx:checkov"` (uv is already installed, so mise uses it). Pin each non-Terraform tool to the exact version `mise latest <tool>` reports today.
- [ ] **Step 3: Write `bin/awslocal`**: a bash script that sets the defaults above with `${VAR:-default}` and runs `exec aws --endpoint-url="${AWS_ENDPOINT_URL:-http://localhost:4566}" "$@"`. Make it executable.
- [ ] **Step 4: Append `sandbox/**/out/` to `.gitignore`.** Leave `.terraform.lock.hcl` tracked (ADR-0002 commits it).
- [ ] **Step 5: Verify**

Run: `mise trust && mise install && terraform version && tflint --version && trivy --version && checkov --version`
Expected: `Terraform v1.17.0` first; every command prints a version.

- [ ] **Step 6: Commit**

```bash
git add mise.toml bin/awslocal .gitignore
git commit -m "chore: pin toolchain with mise and add awslocal wrapper"
```

---

### Task 2: Phase 0 sandbox and the core workflow

**Files:**
- Create: `sandbox/phase-0/versions.tf`, `variables.tf`, `main.tf`, `outputs.tf`, `tests/greeting.tftest.hcl`
- Create: `docs/notes/phase-0.md`

**Interfaces:**
- Produces: variable `greeting_prefix` (string, default `"Hello"`, validation: 1–20 characters); `random_pet.name` (`length = 2`); `local_file.greeting` at `${path.module}/out/greeting.txt` with content `"${var.greeting_prefix}, ${random_pet.name.id}!\n"`; outputs `pet_name` and `greeting_path`.
- Produces: `docs/notes/phase-0.md` with headings `## Commands`, `## Generated files`, `## Predictions log` (table: step, prediction, actual, why), `## Drift`, `## Terraform vs OpenTofu`, `## Emulator checklist`, `## Docker build check`.

**What to look at:** `.terraform/providers/` after `init`; the hashes in `.terraform.lock.hcl`; the `+`, `~`, `-/+` symbols in plan output; `terraform.tfstate` stores the file content in plaintext; `terraform show` vs reading the JSON.

- [ ] **Step 1: Write the failing test** `tests/greeting.tftest.hcl`

```hcl
run "greeting_uses_prefix" {
  command = apply
  variables { greeting_prefix = "Hi" }
  assert {
    condition     = startswith(local_file.greeting.content, "Hi, ")
    error_message = "greeting must start with the prefix"
  }
}

run "rejects_empty_prefix" {
  command = plan
  variables { greeting_prefix = "" }
  expect_failures = [var.greeting_prefix]
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `terraform -chdir=sandbox/phase-0 init -backend=false && terraform -chdir=sandbox/phase-0 test`
Expected: FAIL: references to undeclared resource `local_file.greeting` / variable.

- [ ] **Step 3: Implement** `versions.tf` (`required_version = "~> 1.17.0"`, `random ~> 3.7`, `local` `~> 2.5`), `variables.tf`, `main.tf`, `outputs.tf` per the Interfaces block.
- [ ] **Step 4: Run tests and format checks**

Run: `terraform -chdir=sandbox/phase-0 fmt -check && terraform -chdir=sandbox/phase-0 validate && terraform -chdir=sandbox/phase-0 test`
Expected: `Success! 2 passed, 0 failed.`

- [ ] **Step 5: Create `docs/notes/phase-0.md`** with the headings above, empty under each.
- [ ] **Step 6: Learner runs the workflow**, writing a prediction before each step into the Predictions log:
  1. `terraform -chdir=sandbox/phase-0 init`: list what appeared in `.terraform/` and the lock file.
  2. `plan`: predict `2 to add`.
  3. `apply`, then `terraform state list` and `terraform show`.
  4. `plan` again: predict `No changes.`
  5. **Drift:** edit `out/greeting.txt` by hand, run `plan`; record what Terraform proposes and why (`local_file` reads the file back and finds the content changed).
  6. `plan -var greeting_prefix=Hey`: predict whether `random_pet` is replaced (it is not; only the file is).
  7. `destroy`.
- [ ] **Step 7: Learner fills `## Commands` and `## Generated files`** in their own words, and `## Terraform vs OpenTofu` (licence, state encryption, registry, feature drift; 5–10 lines).
- [ ] **Step 8: Verify nothing generated is staged**

Run: `git status --porcelain | grep -E 'tfstate|/\.terraform/|/out/' ; test $? -eq 1 && echo clean`
Expected: `clean`

- [ ] **Step 9: Commit**

```bash
git add sandbox/phase-0 docs/notes/phase-0.md
git commit -m "feat(sandbox): phase 0 local workflow config, test and notes"
```

---

### Task 3: Emulator runtime and probe harness

**Files:**
- Create: `compose.yaml`, `Makefile`, `sandbox/emulator-check/lib.sh`, `sandbox/emulator-check/check.sh`

**Interfaces:**
- Produces: `make up` (start MiniStack, wait until healthy), `make stop` (stop, keep the volume), `make down` (remove container and the `ministack-data` volume; later phases prepend `terraform destroy` steps), `make check` (runs `sandbox/emulator-check/check.sh all`).
- Produces (`lib.sh`): `preflight` (exit 2 with `emulator not reachable at http://localhost:4566` if `curl -sf` on the endpoint fails); `check <name> <cmd...>` (blocking: prints `PASS name` / `FAIL name`, increments `FAILS`); `soft_check <name> <cmd...>` (prints `PASS`/`WARN`, never fails the run; used only for items with a spec fallback); `wait_for <timeout_s> <cmd...>` (retries every 5 s, succeeds when the command succeeds).
- Produces (`check.sh`): `./check.sh <storage|compute|backend|persistence|all>`; runs `preflight` first; exit code = number of blocking failures (capped at 1).

**What to look at:** why MiniStack is outside Terraform (spec 3.1: the `aws` provider needs a running endpoint before it can plan); the named volume is what makes `PERSIST_STATE=1` survive `make stop`.

- [ ] **Step 1: Write `compose.yaml`**: service `ministack`, image `ministackorg/ministack` pinned to the newest release tag on its GitHub releases page (record the tag in notes), port `4566:4566`, env `PERSIST_STATE=1`, named volume `ministack-data` mounted at the data path from MiniStack's README, and `/var/run/docker.sock` mounted if the README requires it for Lambda execution. Add a healthcheck if the README documents a health endpoint.
- [ ] **Step 2: Write the failing preflight test**: with MiniStack **not** running:

Run: `sandbox/emulator-check/check.sh all; echo "exit=$?"`
Expected (before `lib.sh` exists): a bash error. After Step 3: `emulator not reachable at http://localhost:4566` and `exit=2`.

- [ ] **Step 3: Implement `lib.sh`, `check.sh`** (groups are empty functions for now) **and the `Makefile`** per the Interfaces block. `make up` uses `docker compose up -d --wait`.
- [ ] **Step 4: Verify both paths**

Run: `sandbox/emulator-check/check.sh all; echo "exit=$?"` → `emulator not reachable...`, `exit=2`.
Run: `make up && bin/awslocal sts get-caller-identity --query Account --output text` → `111111111111`.
Run: `AWS_ACCESS_KEY_ID=222222222222 bin/awslocal sts get-caller-identity --query Account --output text` → `222222222222`.

- [ ] **Step 5: Commit**

```bash
git add compose.yaml Makefile sandbox/emulator-check/lib.sh sandbox/emulator-check/check.sh
git commit -m "feat(sandbox): MiniStack runtime, make targets and probe harness"
```

---

### Task 4: Storage, data, secrets, accounts and regions checklist

**Files:**
- Create: `sandbox/emulator-check/versions.tf`, `providers.tf`, `storage.tf`, `outputs.tf`
- Modify: `sandbox/emulator-check/check.sh` (fill `storage` group)

**Interfaces:**
- Consumes: `lib.sh` helpers, `bin/awslocal` (Task 3).
- Produces (`providers.tf`): default `aws` provider (dev account, `eu-central-1`), `aws.secondary` (dev account, `eu-west-1`), `aws.prod` (access key `222222222222`, `eu-central-1`). All three: `skip_credentials_validation`, `skip_metadata_api_check`, `s3_use_path_style = true`, and an `endpoints {}` block pointing `s3, sqs, lambda, dynamodb, secretsmanager, kms, apigateway, scheduler, events, cloudwatch, logs, sns, sts, iam` at `http://localhost:4566` (one shared `local.endpoint`).
- Produces (`storage.tf`, outputs in parentheses):
  - `aws_kms_key.check`; bucket `if-check-data` with public-access block, versioning `Enabled`, SSE-KMS with that key, CORS (`GET, PUT` from `http://localhost:8080`), lifecycle rule expiring prefix `transcripts/` after 30 days (`data_bucket`).
  - bucket `if-check-site` with website config `index_document = "index.html"` and an `aws_s3_object` `index.html` containing `if-check-site-ok` (`site_bucket`).
  - bucket `if-check-tfstate`, versioned and encrypted (`state_bucket`), used in Task 6.
  - DynamoDB table `if-check-history`, hash key `pk` (S), `PAY_PER_REQUEST`, `deletion_protection_enabled = true` (`history_table`).
  - `ephemeral "random_password" "login"` (length 32) → `aws_secretsmanager_secret.login` + `aws_secretsmanager_secret_version` with `secret_string_wo = ephemeral.random_password.login.result`, `secret_string_wo_version = 1` (`secret_id`).
  - bucket `if-check-secondary` through `provider = aws.secondary` (`secondary_bucket`).
  - SQS queue `if-check-prod-only` through `provider = aws.prod` (`prod_queue_name`).

**What to look at:** provider aliases and `provider =` on a resource; the `ephemeral` block never appears in `terraform state list`; `secret_string_wo` shows as `(write-only attribute)` in the plan; why `deletion_protection_enabled` and `prevent_destroy` are different layers.

- [ ] **Step 1: Write the failing probes** in the `storage` group. Blocking `check`s:
  - `s3-encryption`: `get-bucket-encryption` on `if-check-data` returns `aws:kms`.
  - `s3-versioning`: `get-bucket-versioning` returns `Enabled`.
  - `s3-cors`: `get-bucket-cors` includes `http://localhost:8080`.
  - `dynamodb-deletion-protection`: `dynamodb delete-table` on `if-check-history` **fails**.
  - `secret-readable`: `secretsmanager get-secret-value` returns a 32-character string.
  - `secret-not-in-state`: that string is **not** found by `grep -F` in `sandbox/emulator-check/terraform.tfstate`.
  - `region-isolation`: `AWS_REGION=eu-west-1 s3api get-bucket-location --bucket if-check-secondary` returns `eu-west-1`.
  - `account-isolation`: `sqs get-queue-url --queue-name if-check-prod-only` fails with the dev key and succeeds with `AWS_ACCESS_KEY_ID=222222222222`.

  `soft_check`s (spec has a fallback):
  - `s3-lifecycle-config`: `get-bucket-lifecycle-configuration` shows the 30-day rule.
  - `s3-website`: `curl -sf` on the website endpoint (try `http://if-check-site.s3-website.localhost:4566/` and the form documented in MiniStack's README) returns `if-check-site-ok`.
- [ ] **Step 2: Run to verify they fail**

Run: `make up && sandbox/emulator-check/check.sh storage; echo "exit=$?"`
Expected: every item `FAIL`/`WARN`, `exit=1`.

- [ ] **Step 3: Implement** `versions.tf` (`required_version = "~> 1.17.0"`, `aws ~> 6.0`, `random ~> 3.7`, `archive ~> 2.7` for Task 5), `providers.tf`, `storage.tf`, `outputs.tf` per the Interfaces block. Use `aws_s3_bucket_*` sub-resources (not inline blocks).
- [ ] **Step 4: Static checks**

Run: `terraform -chdir=sandbox/emulator-check fmt -check && terraform -chdir=sandbox/emulator-check init && terraform -chdir=sandbox/emulator-check validate`
Expected: `Success! The configuration is valid.`

- [ ] **Step 5: Learner predicts, plans and applies** (`terraform -chdir=sandbox/emulator-check apply`). Then a second `plan`: predict and confirm `No changes.` Any perpetual diff is a checklist finding: record the attribute in notes.
- [ ] **Step 6: Run the probes**

Run: `sandbox/emulator-check/check.sh storage; echo "exit=$?"`
Expected: all blocking items `PASS`, `exit=0`. Record every `FAIL`/`WARN` with its error under `## Emulator checklist`.

- [ ] **Step 7: Commit**

```bash
git add sandbox/emulator-check
git commit -m "feat(sandbox): emulator checklist for storage, secrets, accounts and regions"
```

---

### Task 5: Compute and messaging checklist

**Files:**
- Create: `sandbox/emulator-check/compute.tf`, `sandbox/emulator-check/lambda/index.mjs`
- Modify: `sandbox/emulator-check/outputs.tf`, `sandbox/emulator-check/check.sh` (fill `compute` group)

**Interfaces:**
- Consumes: providers and helpers from Tasks 3–4.
- Produces (`lambda/index.mjs`): `export const handler = async (event) => ...`. It logs exactly one line `IF_CHECK source=<src>` per invocation, where `src` is `sqs` (event has `Records`), `apigw` (event has `httpMethod`), or `event.source` otherwise (`schedule` / `rule`). It throws if any SQS record body equals `poison`. For `apigw` it returns `{ statusCode: 200, body: "ok" }`. No SDK imports.
- Produces (`compute.tf`, outputs in parentheses):
  - `archive_file` zipping `lambda/`; `aws_lambda_function.probe` (`if-check-probe`, `nodejs22.x`, handler `index.handler`, timeout 10) with a role allowing `logs:CreateLogGroup/CreateLogStream/PutLogEvents` (`function_name`).
  - Queues `if-check-jobs` (visibility timeout 15 s, redrive to `if-check-jobs-dlq` with `maxReceiveCount = 1`) and the DLQ (`jobs_queue_url`, `dlq_url`); `aws_lambda_event_source_mapping` from `if-check-jobs`, `batch_size = 1`; role also allows `sqs:ReceiveMessage/DeleteMessage/GetQueueAttributes` on the queue.
  - REST v1 API `if-check-api`: resource `/ping`, `GET`, `AWS_PROXY` integration to the probe, deployment and stage `check`, `aws_lambda_permission` for `apigateway.amazonaws.com` (`rest_api_id`).
  - `aws_scheduler_schedule` `if-check-every-minute` (`rate(1 minute)`, input `{"source":"schedule"}`, role allowing `lambda:InvokeFunction`) **and** `aws_cloudwatch_event_rule` `if-check-rule` (`rate(1 minute)`) with target input `{"source":"rule"}` and a matching `aws_lambda_permission`.
  - SNS topic `if-check-alerts`, SQS queue `if-check-alerts-sub` subscribed to it (with a queue policy allowing the topic), `aws_cloudwatch_metric_alarm` `if-check-alarm` on namespace `IFCheck`, metric `Probe`, `Sum >= 1` over 60 s, `alarm_actions = [topic]` (`alerts_sub_url`).

**What to look at:** implicit dependencies (the event source mapping references queue and function, so Terraform orders them); why `aws_lambda_permission` is a separate resource; the IAM policies are correct even though the emulator does not enforce them (spec 3.7).

- [ ] **Step 1: Write the failing probes** in the `compute` group, each using `wait_for 90`:
  - `check sqs-to-lambda`: `send-message` body `hello` to `if-check-jobs`; then `logs filter-log-events --log-group-name /aws/lambda/if-check-probe --filter-pattern '"IF_CHECK source=sqs"'` returns ≥ 1 event.
  - `check dlq-redrive`: send body `poison`; `receive-message` on the DLQ returns a message with body `poison`.
  - `check apigw-v1`: `curl -sf` on `http://localhost:4566/restapis/<rest_api_id>/check/_user_request_/ping` or `http://<rest_api_id>.execute-api.localhost:4566/check/ping` returns `ok`; print which form worked.
  - `soft_check scheduler-fires`: a log event `IF_CHECK source=schedule` appears.
  - `check schedule-or-rule-fires`: a log event for `source=schedule` **or** `source=rule` appears (blocking: the cleanup Lambda needs one of them).
  - `check alarm-to-sns-to-sqs`: `cloudwatch set-alarm-state --alarm-name if-check-alarm --state-value ALARM --state-reason probe`; `receive-message` on `if-check-alerts-sub` returns a body containing `if-check-alarm`.
- [ ] **Step 2: Run to verify they fail**

Run: `sandbox/emulator-check/check.sh compute; echo "exit=$?"`
Expected: all `FAIL`, `exit=1` (it takes about 90 s per item; that is the timeout working).

- [ ] **Step 3: Implement** `lambda/index.mjs`, `compute.tf` and the new outputs per the Interfaces block.
- [ ] **Step 4: Static checks**: same command as Task 4 Step 4. Expected: valid.
- [ ] **Step 5: Learner predicts, plans and applies**, then a second `plan` predicting `No changes.` Record any perpetual diff.
- [ ] **Step 6: Run the probes**

Run: `sandbox/emulator-check/check.sh compute; echo "exit=$?"`
Expected: blocking items `PASS`, `exit=0`. Record results and the working API Gateway URL form under `## Emulator checklist` (Phase 4 smoke tests will use it).

- [ ] **Step 7: Commit**

```bash
git add sandbox/emulator-check
git commit -m "feat(sandbox): emulator checklist for SQS, Lambda, API Gateway, schedules and alarms"
```

---

### Task 6: Remote state backend and persistence

**Files:**
- Create: `sandbox/emulator-check/backend-check/main.tf`
- Modify: `sandbox/emulator-check/check.sh` (fill `backend` and `persistence` groups)

**Interfaces:**
- Consumes: bucket `if-check-tfstate` (Task 4); `make stop` / `make up` (Task 3).
- Produces (`backend-check/main.tf`): `backend "s3"` with `bucket = "if-check-tfstate"`, `key = "backend-check/terraform.tfstate"`, `region = "eu-central-1"`, `use_lockfile = true`, `use_path_style = true`, `endpoints = { s3 = "http://localhost:4566" }`, `access_key = "111111111111"`, `secret_key = "test"`, `skip_credentials_validation = true`, `skip_requesting_account_id = true`, `skip_s3_checksum = true`; one `resource "terraform_data" "marker" { input = "backend-ok" }`.

**What to look at:** the bootstrap problem in miniature: the state bucket had to exist (Task 4) before this config could `init`; the `.tflock` object only exists while an operation holds the lock.

- [ ] **Step 1: Write the failing probes**
  - `backend` group, `check remote-state-object`: `s3api head-object --bucket if-check-tfstate --key backend-check/terraform.tfstate` succeeds.
  - `persistence` group, `check data-survives-restart`: if the object `persistence/marker.txt` in bucket `if-check-data` is missing, put it with body `persist-<epoch>`, print `marker written; run make stop && make up, then rerun`, and exit non-zero; otherwise pass when the stored body starts with `persist-`.
- [ ] **Step 2: Run to verify** `./check.sh backend` → `FAIL remote-state-object`.
- [ ] **Step 3: Implement `backend-check/main.tf`.**
- [ ] **Step 4: Learner runs** `terraform -chdir=sandbox/emulator-check/backend-check init` then `apply`. **Lock observation:** start a second `apply` and leave it at the approval prompt; in another terminal run `bin/awslocal s3 ls s3://if-check-tfstate/backend-check/` and record whether `terraform.tfstate.tflock` is listed; answer `no` to the prompt and list again.
- [ ] **Step 5: Verify backend**: `sandbox/emulator-check/check.sh backend` → `PASS remote-state-object`.
- [ ] **Step 6: Verify persistence**: `./check.sh persistence` (writes marker) → `make stop && make up` → `./check.sh persistence` → `PASS data-survives-restart`. Then run `terraform -chdir=sandbox/emulator-check plan` and confirm `No changes.` (Terraform's view and the emulator's survived together).
- [ ] **Step 7: Commit**

```bash
git add sandbox/emulator-check
git commit -m "feat(sandbox): S3 backend locking and persistence checks"
```

---

### Task 7: Docker image build check

**Files:**
- Create: `sandbox/docker-build-check/main.tf`, `sandbox/docker-build-check/Dockerfile`

**Interfaces:**
- Produces: `docker_image.check` named `if-build-check:dev`, `build { context = path.module }`, `triggers = { dockerfile = filesha256("${path.module}/Dockerfile") }`; Dockerfile `FROM node:22-alpine` and `CMD ["node", "-e", "console.log('build-ok')"]`.

**What to look at:** `triggers` forcing replacement; this is the mechanism Phase 1 uses with a hash of `app/src`.

- [ ] **Step 1: Implement both files** (`kreuzwerker/docker ~> 4.0`).
- [ ] **Step 2: Learner records the current store**: `docker info --format '{{json .DriverStatus}}'` (`io.containerd.snapshotter.v1` means the containerd store is **on**).
- [ ] **Step 3: Learner runs `init` and `apply`.** Record success or the exact error under `## Docker build check`, noting which store was active.
- [ ] **Step 4: If the containerd store was on**, turn it off (Docker Desktop → Settings → General → untick "Use containerd for pulling and storing images" → Apply & restart), confirm `DriverStatus` no longer shows `io.containerd.snapshotter.v1`, and re-run `apply`.
- [ ] **Step 5: Verify**

Run: `docker run --rm if-build-check:dev`
Expected: `build-ok`

- [ ] **Step 6: Learner edits the Dockerfile** (any change to `CMD`), predicts the plan (`docker_image.check must be replaced`), runs `plan`, records it. Then `destroy`.
- [ ] **Step 7: Commit**

```bash
git add sandbox/docker-build-check docs/notes/phase-0.md
git commit -m "feat(sandbox): docker_image build check"
```

---

### Task 8: Decide ADR-0001 and close Phase 0

**Files:**
- Modify: `docs/adr/0001-emulated-aws-on-ministack.md`, `docs/notes/phase-0.md`, `compose.yaml` (only if switching emulator)

**Interfaces:**
- Consumes: probe results from Tasks 4–6 and the Docker result from Task 7.

**What to look at:** an ADR is never edited to hide a reversal; a different choice is a new ADR that supersedes the old one.

- [ ] **Step 1: Full run**

Run: `make check; echo "exit=$?"`
Expected: `exit=0`. Paste the full PASS/FAIL/WARN list into `## Emulator checklist`.

- [ ] **Step 2: Decide**
  - **All blocking items pass:** set ADR-0001 to `**Status:** Accepted`; tick its action items; record the pinned MiniStack tag and each `WARN` with the spec fallback it triggers (`s3-website` → nginx fallback in Phase 5; `scheduler-fires` → EventBridge rule; `s3-lifecycle-config` → test-only assertion).
  - **Any blocking item fails:** change `compose.yaml` to Floci, rerun Tasks 4–6 probes (`make down up`, learner re-applies, `make check`); if Floci also fails, try LocalStack Hobby (needs `LOCALSTACK_AUTH_TOKEN`). Write `docs/adr/0006-<emulator>.md` with status Accepted, and set ADR-0001 to `Superseded by ADR-0006`. Update the spec's section 1 row to link the new ADR.
- [ ] **Step 3: Clean up**: learner runs `terraform destroy` in `sandbox/emulator-check/backend-check`, then `sandbox/emulator-check`; then `make down`.
- [ ] **Step 4: Exit check**: `docs/notes/phase-0.md` explains every command (`init`, `plan`, `apply`, `destroy`) and every generated file (`.terraform/`, `.terraform.lock.hcl`, `terraform.tfstate`), and has the checklist and Docker results filled. Every Predictions log row has an actual result.
- [ ] **Step 5: Commit and tag**

```bash
git add docs compose.yaml
git commit -m "docs: record phase 0 results and decide ADR-0001"
git tag phase-0-done
```
