# Architecture Decision Records (ADRs)

Durable, dated records of architectural decisions for the BRAC POC OpenShift
fleet. Wiki prose captures the *current* state; these ADRs capture *why* it's
that state, what alternatives were rejected, and the date the call was made.

| # | Title | Status |
|---|-------|--------|
| [0001](0001-spoke-dr-platform-standby.md) | spoke-dr is platform standby, not app-ready hot standby | Accepted 2026-05-07 |
| [0002](0002-argocd-agent-topology.md) | Argo CD Agent topology — managed mode, single Principal on hub-dc, Vault-backed PKI | Accepted 2026-05-07 |

## Format

Each ADR follows the [MADR](https://adr.github.io/madr/) lineage:
`Status`, `Context`, `Decision`, `Consequences`, `Alternatives Considered`,
`References`. Numbering is zero-padded sequence (`0001`, `0002`, …); slug is
kebab-case after the number.

## Scope

This folder records **fleet / platform decisions** for the OCP four-cluster
environment. Project-specific decisions for individual workloads (e.g.,
`brac-poc-openliberty-v5`) live in that project's own `docs/adr/` folder
(<https://github.com/zeshaq/brac-poc-openliberty-v5/tree/main/docs/adr>).
