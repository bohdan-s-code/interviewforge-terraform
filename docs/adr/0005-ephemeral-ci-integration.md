# ADR-0005: Run CI as an ephemeral integration test on GitHub-hosted runners

**Status:** Accepted
**Date:** 2026-10-23
**Deciders:** Project owner (bohdan-s-code)

## Context

Phase 5 needs CI that runs `plan` on pull requests, deploys dev automatically and deploys prod after manual approval. The repository is public on GitHub. The "cloud" is an emulator (ADR-0001), and a GitHub-hosted runner starts a fresh emulator container on every run, so CI never sees the learner's local state.

## Decision

Every PR runs **plan → apply → smoke → destroy** against a fresh MiniStack service container. Merge to `main` runs the same flow for dev, then a prod job gated by a GitHub Environment with a required reviewer. CI authenticates with emulator test credentials; the OIDC provider and role exist in `bootstrap/` but are not used for authentication.

## Options Considered

### Option A: Ephemeral integration test (chosen)
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Cost | Free on public repos |
| Security | Safe: no secrets, runs in GitHub's sandbox |
| Realism | Partial: plans always show a full create |

**Pros:** proves destroy-and-rebuild on every PR; tests the whole stack; works on fork PRs.
**Cons:** no real diff-on-PR; "deploy dev" deploys into a throwaway container.

### Option B: Plan on PR, apply on merge, as in the learning plan
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Cost | Free |
| Security | Safe |
| Realism | Misleading: the apply target disappears when the job ends |

**Pros:** matches the plan's wording.
**Cons:** looks like deployment but deploys nothing lasting.

### Option C: Self-hosted runner on the learner's machine
| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium |
| Cost | Free |
| Security | Unsafe: a public repo would run strangers' PR code on the learner's Mac |
| Realism | High: real diffs against persistent state |

**Pros:** real plan diffs and persistent environments.
**Cons:** unacceptable security risk.

## Trade-off Analysis

Option A is the most that a GitHub-hosted runner can do truthfully. The approval gate, environment protection and workflow structure are still learned exactly as in real projects. The lost lesson, a real diff on a PR, is recorded in notes as needing persistent shared state.

## Consequences

- Easier: CI is self-contained and secret-free; every PR tests a full rebuild.
- Harder: CI runs take longer (full apply and destroy).
- Revisit: if the project moves to real AWS, switch to Option B with OIDC and remote state.

## Action Items

1. [ ] Phase 3: `ci.yml` with static checks and tests.
2. [ ] Phase 5: add the integration job and the prod Environment with a required reviewer.
