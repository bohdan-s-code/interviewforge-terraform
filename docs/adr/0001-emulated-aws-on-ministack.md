# ADR-0001: Run AWS fully emulated on MiniStack

**Status:** Proposed (becomes Accepted when the Phase 0 emulator checklist passes)
**Date:** 2026-10-23
**Deciders:** Project owner (bohdan-s-code)

## Context

InterviewForge exists to learn Terraform against AWS. The learner requires zero cost and zero risk, so no real AWS account is used. Phases 4–6 need S3, SQS with DLQ, Lambda (`nodejs22.x`) with SQS event source mappings, DynamoDB, Secrets Manager, KMS, API Gateway REST v1, EventBridge schedules, CloudWatch alarms, SNS, two isolated accounts, two regions, and an S3 state backend that survives restarts.

LocalStack Community, the usual choice, was retired on 2026-03-23. Its replacement free tier (Hobby) needs an account and auth token, is non-commercial only, has no persistence, and cannot start on pull requests from forks because GitHub withholds secrets from them.

## Decision

Use **MiniStack** (MIT, `ministackorg/ministack`) in Docker on `localhost:4566` with `PERSIST_STATE=1`. It runs from `compose.yaml` and is not managed by Terraform. Fallback order if the Phase 0 checklist fails: Floci, then LocalStack Hobby.

## Options Considered

### Option A: MiniStack (chosen)
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low: one container, no account |
| Cost | Free, MIT |
| Fidelity | README covers every required service, including SQS → Lambda event source mapping; S3 website hosting and Scheduler execution unconfirmed |
| Familiarity | Low: smaller community, fewer docs |

**Pros:** no token; free persistence; 12-digit account IDs with per-region isolation; works on fork PRs.
**Cons:** younger project; gaps must be found by testing, not documentation.

### Option B: LocalStack Hobby
| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium: account, token, CI secret |
| Cost | Free, non-commercial only |
| Fidelity | Highest; all required services documented |
| Familiarity | High: most tutorials use it |

**Pros:** mature; best docs; `tflocal` and `awslocal` tooling.
**Cons:** no persistence (state backend lost on restart); CI fails on fork PRs; licence breaks if the project becomes work-related.

### Option C: Floci
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Cost | Free, MIT |
| Fidelity | Broad, with persistence and multi-account; SQS → Lambda mapping not listed |
| Familiarity | Medium: larger community than MiniStack |

**Pros:** popular; same port and model as MiniStack.
**Cons:** the missing SQS event source mapping would break Feature 3.

### Option D: Moto server
**Pros:** long-lived, Apache-2.0, broad API coverage.
**Cons:** event source mappings and schedules do not trigger anything; suited to API-level tests, not a running app.

### Option E: Real AWS free tier
**Pros:** full fidelity, including IAM enforcement and OIDC.
**Cons:** needs a card; misconfiguration can cost money. Rejected by the zero-cost requirement.

## Trade-off Analysis

The deciding forces are persistence (the Phase 5 S3 backend must survive restarts), no-token CI (public repo, fork PRs) and the SQS event source mapping (Feature 3). Only MiniStack documents all three. Its weaker documentation is offset by a Phase 0 checklist that tests every service the project uses. All candidates serve `localhost:4566`, so switching changes only `compose.yaml`.

## Consequences

- Easier: no accounts or secrets anywhere; state persists across sessions; CI works on any PR.
- Harder: IAM is not enforced, S3 lifecycle expiry likely does not run, OIDC cannot be exercised. Each gap has a substitute in spec section 3.7.
- Revisit: if any checklist item fails, or MiniStack stops being maintained.

## Action Items

1. [ ] Phase 0: build `sandbox/emulator-check/` and run the full checklist (spec, Phase 0).
2. [ ] Set this ADR to Accepted, or supersede it with the chosen fallback.
3. [ ] Pin the MiniStack image tag in `compose.yaml`.
