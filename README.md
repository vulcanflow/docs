# VulcanFlow docs

Authoritative project documentation for VulcanFlow — a multi-tenant SaaS security scanning platform (authorization-gated discovery, remediation guidance, verification rescan, and reporting).

## Source of truth

- [Technical Design Document v2.1](./VulcanFlow_—_Technical_Design_Document.md) — architecture, components, phasing, open items

## Repository architecture (polyrepo)

Chosen direction: **polyrepo** (requester preference; factory confirmed). One GitHub repository per independently deployable component or shared library, plus platform and docs.

| Repository | TDD component / role | Language | Phase |
|---|---|---|---|
| [docs](https://github.com/vulcanflow/docs) | Architecture & design (this repo) | Markdown | 0 |
| `infra` | Phase 0 foundation: GitOps (Argo CD/Kargo), CNPG, ClickHouse, Valkey, Keycloak, secureCodeBox, Harbor, secrets, image signing | Kustomize/Helm/YAML | 0 |
| `vf-api` | REST + Connect-RPC API; **includes** `vf-dispatcher` module | Go 1.26, chi, Huma | 1 |
| `vf-authz` | Authorization Track A/B, basis records | Go | 1 |
| `vf-operator` | `Tenant` / `ScanFlow` CRDs + reconciler | Go, controller-runtime | 1 |
| `vf-translator` | Flow graph → secureCodeBox Scan/CascadingRule (library) | Go | 1 |
| `vf-ingest` | findings.json normalize, fingerprint, dedup, enrich | Go | 1 |
| `scanners` | Custom arm64 SCB scanner images + parsers + conformance | Container/Go | 2 |
| `vf-web` | SPA: builder, findings, remediation, reports, billing | React 19, Vite 8 | 3 |
| `vf-remediation` | Remediation content + verification-rescan orchestration | Go | 3 |
| `vf-report` | Report assembly, PDF/HTML, scheduled delivery | Go + Chromium | 3 |
| `vf-meter` | Credits, wallet ledger, Lago/Stripe, packages | Go | 4 |
| `vf-abuse` | KYC, anomaly, suspend/kill, egress reputation | Go | 4 |
| `vf-aigw` | Self-hosted OpenAI-compatible AI gateway | Gateway | 5 |

Derived from TDD §2.3 Component inventory and §24 Phasing. `vf-dispatcher` is a module of `vf-api` (not a separate repo).

## Setup backlog

Tracked as issues in this repository, labeled `factory:vulcanflow`. Start at Phase 0 → Phase 1 (walking skeleton prerequisites).

## Domain

- Product domain: `vulcanflow.io`
- GitHub org: [`vulcanflow`](https://github.com/vulcanflow)
