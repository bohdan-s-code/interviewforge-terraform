# ADR-0002: Use Terraform 1.17 with pinned providers

**Status:** Accepted
**Date:** 2026-10-23
**Deciders:** Project owner (bohdan-s-code)

## Context

The project teaches Terraform specifically. Two features need recent language capabilities: secrets kept out of state (write-only attributes, Terraform ≥ 1.11; ephemeral resources, ≥ 1.10) and native S3 state locking (`use_lockfile`, GA in 1.11). The learning plan lists "Terraform vs OpenTofu" as a reading topic in Phase 0.

## Decision

Use **Terraform 1.17.0**, pinned in `mise.toml`, with these provider constraints:

| Provider | Constraint | Why |
|---|---|---|
| `hashicorp/aws` | `~> 6.0` | `secret_string_wo` needs ≥ 5.88 |
| `hashicorp/random` | `~> 3.7` | ephemeral `random_password` needs ≥ 3.7 |
| `kreuzwerker/docker` | `~> 4.0` | maintained v4 line, `docker_image` `build {}` |
| `local`, `archive`, `http` | latest major | stable, small |

The lock file `.terraform.lock.hcl` is committed.

## Options Considered

### Option A: Terraform 1.17 (chosen)
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Cost | Free (BSL licence; personal use is fine) |
| Ecosystem | Largest: registry, docs, job market |
| Features | Write-only attributes, ephemeral resources, `use_lockfile` |

**Pros:** matches the project goal; most learning material targets it.
**Cons:** BSL licence, not open source.

### Option B: OpenTofu
| Dimension | Assessment |
|-----------|------------|
| Complexity | Low; near drop-in |
| Cost | Free, MPL-2.0 |
| Ecosystem | Smaller; diverging features (for example state encryption) |
| Features | Equivalent for most of this plan |

**Pros:** truly open source; client-side state encryption.
**Cons:** the project's stated goal is Terraform; feature drift makes some docs not apply.

## Trade-off Analysis

The learning goal decides it. OpenTofu stays a Phase 0 reading topic, and the code avoids anything that would make a later switch hard.

## Consequences

- Easier: every doc and example applies as written.
- Harder: nothing significant.
- Revisit: when bumping Terraform or a provider major version (for example aws 7.x); record the upgrade in notes.

## Action Items

1. [ ] Phase 0: create `mise.toml` with Terraform 1.17.0 and the supporting tools.
2. [ ] Phase 0: write a short Terraform-vs-OpenTofu comparison in `docs/notes/phase-0.md`.
