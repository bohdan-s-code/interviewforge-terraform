# ADR-0003: Write the app in TypeScript with Hono on two runtimes

**Status:** Accepted
**Date:** 2026-10-23
**Deciders:** Project owner (bohdan-s-code)

## Context

The app must stay minimal, because the goal is Terraform. It runs as Docker containers in Phases 1–4 and as Lambda functions behind API Gateway from the end of Phase 4. Rewriting the API at that point would be work that teaches no Terraform.

## Decision

Use **Node.js + TypeScript** with the **Hono** framework. The same Hono app runs through `@hono/node-server` in containers and through Hono's AWS Lambda adapter on `nodejs22.x`. **esbuild** bundles each entry point to `app/dist/<name>/index.js`; Terraform's `archive_file` zips the Lambda bundles. Storage sits behind one interface with `fs` and `aws` adapters, selected by environment variables that Terraform sets.

## Options Considered

### Option A: TypeScript + Hono (chosen)
| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium: needs a build step |
| Cost | Free |
| Runtime portability | One codebase for Node server and Lambda |
| Familiarity | Chosen by the project owner |

**Pros:** no API rewrite at the Lambda move; types for AWS SDK v3; small bundles.
**Cons:** esbuild step before every apply that ships code.

### Option B: Python + FastAPI
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low: no build step |
| Cost | Free |
| Runtime portability | Needs an adapter (Mangum) for Lambda |
| Familiarity | Not preferred by the owner |

**Pros:** zip source directly; most emulator examples use Python.
**Cons:** not the owner's choice.

### Option C: Go
**Pros:** single static binary; fast cold starts.
**Cons:** cross-compiling for `provided.al2023` adds non-Terraform friction.

## Trade-off Analysis

The build step costs one `npm run build` and is isolated from Terraform: Terraform only zips `app/dist/`. Hono's two adapters remove the largest app-side cost in the plan, the container-to-Lambda move.

## Consequences

- Easier: the Phase 4 Lambda move changes infrastructure, not app logic.
- Harder: Terraform plans show Lambda changes only after a rebuild; `make apply` must depend on `make build`.
- Revisit: if MiniStack's `nodejs22.x` support fails the Phase 0 checklist.

## Action Items

1. [ ] Phase 1: scaffold `app/` with Hono, esbuild and vitest.
2. [ ] Phase 1: make `make apply` depend on `make build`.
