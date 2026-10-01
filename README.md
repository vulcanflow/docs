# VulcanFlow docs

Authoritative project documentation for VulcanFlow — a multi-tenant SaaS security scanning platform (authorization-gated discovery, remediation guidance, verification rescan, and reporting).

## Source of truth

- [Technical Design Document v2.2](./VulcanFlow_Technical_Design_Document_v2.2.md) — architecture, components, phasing, open items

## Repository architecture (polyrepo) — SUPERSEDED

> **Superseded by [ADR-0004](./decisions/ADR-0004-workspace-repository.md).** All VulcanFlow
> Rust crates belong in one Cargo workspace in [`vulcanflow/platform`](https://github.com/vulcanflow/platform),
> and the rule is that every VulcanFlow Cargo crate lives there — no other repository contains a
> `Cargo.toml`. The nine per-crate repositories below are **to be archived, and are not archived
> yet**: ADR-0004 §4.2 orders that step after the workspace reaches `platform@main`, so all
> fifteen repositories are still writable today. ADR-0004 §2.1 dispositions every one of them as
> active or to-be-archived, with a reason. The table below is also pre-TDD-v2.3 — the languages and frameworks it names
> were withdrawn by [ADR-0001](./decisions/ADR-0001-retire-go-scaffold-vf-api.md) and
> [ADR-0002](./decisions/ADR-0002-rust-crate-set-and-phase0-pins.md). It is retained only until
> this README is rewritten; read it as history, not as a decision.

Chosen direction: **polyrepo** (requester preference; factory confirmed). One GitHub repository per independently deployable component or shared library, plus platform and docs.

| Repository | TDD component / role | Language | Phase |
|---|---|---|---|
| [docs](https://github.com/vulcanflow/docs) | Architecture & design (this repo) | Markdown | 0 |
| `infra` | Phase 0 cluster baseline + foundation: GitOps (Argo CD/Kargo), CNPG, ClickHouse, Valkey, Keycloak 26, secureCodeBox (**version pin / K8s compatibility confirmation required** — TDD §24.2, §21.3, open item 5), Harbor, secrets, image signing | Kustomize/Helm/YAML | 0 |
| `vf-api` | REST + Connect-RPC API; **includes** `vf-dispatcher` (estimate work units, reserve allowances, enforce scope) | Go 1.26, chi, Huma | 1 |
| `vf-authz` | Authorization Track A/B, basis records | Go | 1 |
| `vf-operator` | `Tenant` / `ScanFlow` CRDs + reconciler | Go, controller-runtime | 1 |
| `vf-translator` | Flow graph → secureCodeBox Scan/CascadingRule (library) | Go | 1 |
| `vf-ingest` | Findings artifacts, fresh observations, explicit false-positive matching | Go | 1 |
| `vf-meter` | Integer scan allowances/reservations, usage ledger; Lago/Stripe commercial sync | Go | 1 core / 4 commercial |
| `scanners` | Custom arm64 SCB scanner images + parsers + conformance | Container/Go | 2 |
| `vf-web` | SPA: builder, findings, remediation, reports, billing | React 19, Vite 8, TanStack Router | 3a / 3b |
| `vf-remediation` | Remediation content, verification orchestration, historical observation outcomes | Go | 3a |
| `vf-report` | Report assembly, PDF/HTML, scheduled delivery | Go + Chromium | 3b |
| `vf-abuse` | KYC, anomaly, suspend/kill, egress reputation | Go | 4 |
| `vf-aigw` | Self-hosted OpenAI-compatible AI gateway | Gateway | 5 |

Derived from TDD §2.3 Component inventory and §24 Phasing (3a findings/verification, 3b reporting/automation). `vf-dispatcher` is a module of `vf-api` (not a separate repo).

## Setup backlog

Tracked as issues in this repository, labeled `factory:vulcanflow`. Prerequisite order:

1. [#3 Create vulcanflow polyrepo set (org admin)](https://github.com/vulcanflow/docs/issues/3) — create the private repos (factory cannot `createRepository`)
2. [#4 Bootstrap infra repo (Phase 0 foundation)](https://github.com/vulcanflow/docs/issues/4) — cluster baseline / GitOps; gated on secureCodeBox pin (TDD open item 5)
3. Phase 1 service boots after repos exist: #5–#9 and [#16 Bootstrap vf-meter core (Phase 1 allowances)](https://github.com/vulcanflow/docs/issues/16)
4. [#14 Wire new repos into vulcanFlow factory code-forge config](https://github.com/vulcanflow/docs/issues/14) — after repos exist so agents can operate on them

Full list: issues #3–#14 and #16.

## Domain

- Product domain: `vulcanflow.io`
- GitHub org: [`vulcanflow`](https://github.com/vulcanflow)
