# ADR-0004: Keep one evolving codebase with a tag per phase

**Status:** Accepted
**Date:** 2026-10-23
**Deciders:** Project owner (bohdan-s-code)

## Context

The learning plan runs over eight phases. Phase 2 teaches state commands, `import`, `moved` and `removed` blocks, which only mean something when there is applied infrastructure with real state to refactor. The repository layout has to support that.

## Decision

Keep **one codebase** that evolves: `infra/` starts as a flat root module (Phase 1), gets split into `modules/` (Phase 2), and into `envs/{dev,prod}` (Phase 5). Tag each finished phase `phase-N-done`.

## Options Considered

### Option A: One evolving codebase (chosen)
| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium: early mistakes carry forward |
| Learning value | High: real refactoring of live state |
| Comparability | Via `git diff phase-1-done phase-2-done` |

**Pros:** `moved`, `import` and state lessons act on real resources; mirrors real projects.
**Cons:** a bad early design must be fixed in place.

### Option B: A folder per phase
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Learning value | Low: no state carries over |
| Comparability | Easy side by side |

**Pros:** every phase starts clean.
**Cons:** removes the Phase 2 lessons; heavy copy-paste.

### Option C: Final layout from day one
**Pros:** "correct" structure from the start.
**Cons:** skips the Phase 2 refactor; modules arrive before they are needed.

## Trade-off Analysis

Carrying early mistakes forward is the point: fixing them without recreating infrastructure is what Phase 2 teaches. Tags give the comparability that Option B offers.

## Consequences

- Easier: realistic refactoring; one history to read.
- Harder: each phase's plan must start from what actually exists, so detailed plans are written per phase, not up front.
- Revisit: never expected; if a phase needs a clean slate, use a branch.

## Action Items

1. [ ] Tag `phase-N-done` at each exit check.
