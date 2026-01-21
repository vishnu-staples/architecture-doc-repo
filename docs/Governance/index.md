# Overview

This section defines how the **.ca Retail Platform — Architecture Repository** is organized, maintained, and kept current. It establishes the rules that prevent duplication, drift, and inconsistent documentation across teams.

## Purpose

Governance exists to ensure that:

- Architecture documentation remains **authoritative**, **traceable**, and **current**
- Content is **consolidated** (single source of truth) and avoids duplication.
- Changes follow a consistent review and decision process (including ADRs where applicable).
- Readers can reliably find information across **security**, **DevOps**, **delivery**, and **engineering** use cases.

## What belongs in this repo

This repo contains:

- **System documentation** under `systems/dot-ca-retail-platform/`
- **Org-wide standards** under `standards/`
- **Architecture Decision Records (ADRs)** under `decision-records/`
- **Reference material** (e.g., glossary) under `reference/`

This repo does **not** contain:

- Detailed runbooks (`runbooks-links.md`)

## Ownership model

Each system page includes:

- **Owner team**
- **Primary contact / escalation**
- **Review cadence** (e.g., quarterly or on major change)
- **Last reviewed date**

If ownership is unclear, raise it via the contribution workflow described below.

## When an ADR is required

Create or update an ADR when:

- You introduce or replace a major platform component or integration approach
- You change system boundaries, ownership boundaries, or responsibility splits
- You make a decision affecting security posture, compliance scope, or resiliency guarantees
- You introduce breaking changes to interface contracts or versioning policies

ADRs live in: [`decision-records/`](../decision-records/index.md)

## Documentation quality gates

Before merging a PR, ensure:

- Links are valid and point to the **single source of truth**
- No duplicate narrative was introduced (prefer links + tables)
- Interfaces and versioning policy are updated if contracts changed
- Security/operability deltas are updated if posture or reliability changed
- Diagrams index is updated if new diagrams were added

## Related governance documents

- [`Architecture principles`](principles.md)
- [`Documentation standards`](doc-standards.md)
- [`Naming conventions`](naming.md)
- [`Review/approval workflow`](review-workflow.md)
