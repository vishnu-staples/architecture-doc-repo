# Overview

## .CA Retail Platform — Architecture Repository

This repository is the canonical source for architecture, interfaces, and operational context for the **.ca Retail Platform**.  
It is maintained as a living set of documents and is intended for engineering, security, DevOps, and delivery stakeholders.

## Start here

  - **System entry point:** `systems/dot-ca-retail-platform/`
  - **Overview:** purpose, owners, lifecycle, criticality
  - **Architecture:** C4 context/container, boundaries, key dependencies
  - **Catalog:** inventory of applications, components, and external dependencies
  - **Interfaces:** APIs, events, contracts, and versioning policies
  - **Capabilities:** business capabilities mapped to components
  - **Security / Operability:** system deltas and links to standards

## Quick links

### Dot-ca Retail Platform

- [`Overview`](systems/dot-ca-retail-platform/overview.md)
- [`Architecture`](systems/dot-ca-retail-platform/architecture.md)
- [`Catalog`](systems/dot-ca-retail-platform/catalog.md)
- [`Interfaces`](systems/dot-ca-retail-platform/interfaces.md)
- [`Capabilities`](systems/dot-ca-retail-platform/capabilities.md)
- [`Security`](systems/dot-ca-retail-platform/security.md)
- [`Operability`](systems/dot-ca-retail-platform/operability.md)
- [`Diagrams`](systems/dot-ca-retail-platform/diagrams.md)
- [`Runbooks Links`](systems/dot-ca-retail-platform/runbooks-links.md)

### Org-wide standards

- [`Security`](standards/security.md)
- [`Operability`](standards/operability.md)
- [`Integration`](standards/integration.md)
- [`Logging & Observability`](standards/logging-observability.md)
- [`Resiliency`](standards/resiliency.md)
- [`Data Privacy & Retention`](standards/data-privacy-retention.md)
- [`I18N & Localization`](standards/i18n-localization.md)

### Architecture decision records

- [`Decision Records (ADRs)`](decision-records/index.md)

## Contribution and governance

- [`Governance (how this repo is maintained)`](governance/index.md)
- [`Architecture principles`](governance/principles.md)
- [`Documentation standards`](governance/doc-standards.md)
- [`Naming conventions`](governance/naming.md)
- [`Review/approval workflow`](governance/review-workflow.md)

## How to use this repo

- **Integrating with the platform:** start with **Interfaces** and **Catalog**.
- **Operating the platform:** start with **Operability**, then **Runbooks Links**.
- **Risk/compliance assessment:** start with **Security**, then **Standards → Security**.
- **Design changes:** update **Architecture**, then add/update an **ADR**.

## Ownership and currency

Each system page should identify an owner (team) and review cadence. If ownership is unclear, raise it via the contribution process described in **Governance**.
