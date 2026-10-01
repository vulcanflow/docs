# VulcanFlow docs

Authoritative project documentation for VulcanFlow — a multi-tenant SaaS security scanning
platform: authorization-gated discovery, remediation guidance, verification rescan, and
reporting, running on **Aether** (a self-operated, IPv4-only Kubernetes platform — not AWS).

VulcanFlow discovers, guides, and verifies. It does not exploit, does not remediate by
action, does not take credentials into customer systems, and is **not** a
compliance-evidence product (TDD §1.1, §1.3).

## Start here

1. **[Technical Design Document v2.3](./VulcanFlow_Technical_Design_Document_v2.3.md)** —
   the design of record. Read **§0.0** (confirmed decisions) and **§2.5** (implementation
   language and stack) before writing anything.
2. **[Decision records](./decisions/)** — ADRs that settle TDD §27 open items. Where an ADR
   and the TDD body disagree, the ADR is newer; `decisions/README.md` lists the corrections
   to fold into the next TDD revision.

> **If you have read an older copy of this repo:** the implementation language changed from
> Go to Rust in TDD v2.3. The Go application frameworks named in v2.2 — **chi, Huma,
> controller-runtime, Connect-Go** — are withdrawn (§0.0, §2.5) and are not coming back
> ([ADR-0001](./decisions/ADR-0001-retire-go-scaffold-vf-api.md)). The TDD file was named
> `..._v2.2.md` until 2026-10-01 while already containing v2.3; the filename now matches
> the contents.

## Implementation language: Rust

`[CONFIRMED]` **Rust is the primary language for all VulcanFlow-owned backend code** (TDD
§0.0, §2.5): API, dispatcher, authorization, operator and controllers, admission webhooks,
ingest, remediation, reporting, metering, abuse services, the secureCodeBox completion
hook, and the shared graph validator (compiled natively *and* to WebAssembly).

The crate set is **approved and pinned** — it is no longer `[PROPOSED]`. The authoritative
list of pins is [ADR-0002 §3](./decisions/ADR-0002-rust-crate-set-and-phase0-pins.md); TDD
§2.5.2 is superseded by it. Headline pins:

| Concern | Choice |
|---|---|
| Toolchain | Rust **1.98.1**, edition 2024, pinned in `rust-toolchain.toml`; `Cargo.lock` committed |
| Async runtime | `tokio` 1.53.1 (explicit features, never `full`) |
| HTTP server | `axum` 0.8.9 + `tower` 0.5.3 / `tower-http` 0.7.1 |
| OpenAPI 3.1 | `utoipa` 6.0.0 + `utoipa-axum` 0.3.0 (code-first) |
| RPC | **None at GA** — REST only ([ADR-0003](./decisions/ADR-0003-rpc-scb-parser-and-billing-clients.md)) |
| Postgres | `sqlx` 0.9.0, compile-time-checked queries, tenant-scoped access only |
| Kubernetes | `kube` 4.2.0 + `k8s-openapi` 0.28.0 (`=`-pinned) |
| Object storage | `aws-sdk-s3` 1.151.0, configured for Ceph RGW / RustFS — AWS defaults are wrong here |
| Templating | `askama` 0.16.1, context-restricted escaping |
| secureCodeBox | release **v5.9.0**, pinned by digest |

Workspace shape (TDD §2.5.2): pure library crates `vf-core`, `vf-graph`, `vf-translator`,
`vf-authz`, `vf-db`; binary crates `vf-api`, `vf-operator`, `vf-admission`, `vf-ingest`,
`vf-report`, `vf-meter`, `vf-abuse`, `vf-hook-notify`. Safety-critical logic — scope
matching, allowance reservation and settlement, observation state, role policy — lives in
the library crates with **no I/O**, so it can be property-tested and fuzzed in isolation.
`#![forbid(unsafe_code)]` in every VulcanFlow crate.

## What is *not* being rewritten in Rust

Per TDD §2.5.3. Wrapping, pinning and conformance-testing these is in scope; reimplementing
them is not.

| Component | Stays in | Why |
|---|---|---|
| Browser SPA (`vf-web`) | TypeScript / React 19 | Approved frontend stack (§14). Only the graph validator is shared, as Rust-compiled WASM |
| Upstream scanners — subfinder, dnsx, httpx, tlsx, nuclei, Amass (Go); masscan (C); nmap | As shipped upstream | Rewriting maintained security tools adds risk and permanent cost with no product benefit |
| secureCodeBox operator, lurker, stock hooks | Upstream (Go / JS) | Third-party platform component |
| secureCodeBox parsers | Upstream parser SDK (JavaScript) | A Rust parser is permitted only after conformance proves the SCB parser contract — ADR-0003 |
| AI gateway (`vf-aigw`) | LiteLLM (Python) | Approved third-party gateway (§18.1); VulcanFlow does not maintain its code |
| Keycloak customisations | Java | Keycloak's extension model |
| Terraform provider (post-GA) | Go | Terraform's plugin framework is Go |
| SQL migrations, Kubernetes manifests | SQL, YAML | Declarative artifacts |

## Phase map (TDD §24)

**Phase 1 is a walking skeleton, and everything larger waits behind it** (§24.1): one
tenant, one verified domain, one fixed `subfinder → dnsx → httpx → nuclei` pipeline,
durable scope/audit/outbox, cascade and workload gates, integer scan reservations, fresh
per-scan findings, explicit false-positive persistence, a small curated guidance set,
normal-consumption verification, one technical PDF report, and SSE. Authorization, usage,
audit and cancellation ship *with* the first execution path, not after it.

| Phase | Deliverables |
|---|---|
| **0 — Foundation** | Approved Aether stack: CNPG, ClickHouse, Valkey, Keycloak 26, secureCodeBox; Harbor signing; Argo CD / Kargo; platform secret store. **v2.3:** Rust workspace and pinned toolchain; crate approval (done — ADR-0002); arm64 build pipeline; cargo-deny / cargo-audit / SBOM gates; S3-client conformance against Ceph RGW / RustFS; generated SCB CRD types; team Rust enablement (§27 item 16a) |
| **1 — Control plane** | `Tenant` / `ScanFlow` via kube-rs, Track A authorization, scope rules, identity mapping, outbox, atomic allowance reservations, audit, cascade and start barriers (Rust admission webhooks), basic cancellation, ingest / fresh observations / false positives, SSE |
| **2 — Scanners** | Custom arm64 images wrapping upstream tools, with Rust input adapters and completion hook; parsers per §27 item 20; template and outcome conformance; controlled masscan pool under the same approval, accounting and cancellation guarantees |
| **3a — Findings and verification** | SPA findings, builder and attack graph (Rust WASM validator), guidance, observation triage, verification plans and outcomes at normal consumption |
| **3b — Reporting and automation** | Both audiences and groupings, immutable report assembly, Rust templating with context-restricted escaping, HTML / PDF, scheduling, notification recovery and delivery |
| **4 — Commercial operations and hardening** | Final package policy, Lago / Stripe usage sync, reconciliation, abuse operations and KYC, egress operations, branding entitlement, full isolation and recovery evidence |
| **5 — Ecosystem and AI** | Track B at scale, approved AI gateway configuration, NL graph suggestions, narrative assistance, pgvector features, later SDKs and template marketplace |

**GA** covers Phases 0–4 core controls (§24.3). **Blocking dependencies** are in §24.4: the
crate-set gate is cleared by ADR-0002; the remaining pre-execution gates need a cluster, and
the pre-paid-use gates are product and commercial decisions.

### Current state, honestly

Every component repo's default branch is empty. There is no Rust code yet; Phase 0 is in
progress and the first thing to land is the workspace skeleton. The `infra` repo's
`factory/phase0-foundation` branch is **deliberately unmerged** — no cluster is stood up
until there is code worth deploying, and lifting that hold is an owner decision, not drift.

## Repository map (polyrepo)

One GitHub repository per independently deployable component or shared library, plus
platform and docs. The **15 component repos** below are exactly TDD §2.3's component
inventory. `vf-dispatcher` is a *module* of `vf-api`, not a repo.

| Repository | TDD role (§2.3) | Language | Phase |
|---|---|---|---|
| [`vf-api`](https://github.com/vulcanflow/vf-api) | REST, OpenAPI 3.1, SSE, webhook receiver, authn/authz enforcement; **includes** the `vf-dispatcher` module (validate graph, estimate work units, reserve allowances, emit Scan CRDs, enforce scope and concurrency) | Rust — axum, utoipa | 1 |
| [`vf-authz`](https://github.com/vulcanflow/vf-authz) | Track A challenge issue/verify; Track B attestation, scope signal, `security.txt` probe; basis records | Rust — library crate used by `vf-api` and workers | 1 |
| [`vf-operator`](https://github.com/vulcanflow/vf-operator) | Reconciles `Tenant` and `ScanFlow` CRDs: namespace, quota, NetworkPolicy, RBAC, schema | Rust — kube-rs | 1 |
| `vf-admission` *(not created yet)* | Validating webhooks for Scan, Job and Pod admission, fail-closed (§5.7) | Rust — kube-rs `admission` + axum | 1 |
| [`vf-translator`](https://github.com/vulcanflow/vf-translator) | Flow graph → secureCodeBox `Scan` + `CascadingRule`; graph validation | Rust — library crate | 1 |
| `vf-graph` *(not created yet)* | The single graph rule set shared by server and browser (§7.3) | Rust — library crate, native + `wasm32-unknown-unknown` | 1 |
| [`vf-ingest`](https://github.com/vulcanflow/vf-ingest) | Consumes findings artifacts; fresh observations per SCB scan; idempotent ingest, enrichment, explicit false-positive matching | Rust | 1 |
| `vf-hook-notify` *(not created yet)* | Signed secureCodeBox completion notification (§8.4) | Rust — SCB completion-hook image | 1 |
| [`vf-meter`](https://github.com/vulcanflow/vf-meter) | Atomic target and scan allowance accounting, usage ledger, Lago sync, Stripe reconciliation, entitlements | Rust | 1 core / 4 commercial |
| [`scanners`](https://github.com/vulcanflow/scanners) | SCB-compliant arm64 scanner images and parsers with conformance; upstream tools pinned, **not** rewritten | Upstream tools + Rust input adapters | 2 |
| [`vf-remediation`](https://github.com/vulcanflow/vf-remediation) | Remediation content resolution, verification orchestration, historical observation outcomes | Rust | 3a |
| [`vf-report`](https://github.com/vulcanflow/vf-report) | Report assembly, template rendering, PDF/HTML output, scheduled delivery, branding | Rust assembler + isolated headless Chromium renderer | 3b |
| [`vf-web`](https://github.com/vulcanflow/vf-web) | SPA: builder, attack graph, findings, remediation, reports, billing | TypeScript — React 19, Vite 8, TanStack Router | 3a / 3b |
| [`vf-abuse`](https://github.com/vulcanflow/vf-abuse) | KYC orchestration, anomaly detection, suspend/kill automation, egress reputation | Rust | 4 |
| [`vf-aigw`](https://github.com/vulcanflow/vf-aigw) | Single egress point for model calls; tenant tagging, token metering, audit | LiteLLM (Python) — third-party, not rewritten | 5 |

Plus two non-component repos:

| Repository | Role | Language | Phase |
|---|---|---|---|
| [`docs`](https://github.com/vulcanflow/docs) | Design of record and decision records (this repo) | Markdown | 0 |
| [`infra`](https://github.com/vulcanflow/infra) | Aether cluster baseline and Phase 0 foundation: GitOps (Argo CD / Kargo), CNPG, ClickHouse, Valkey, Keycloak 26, secureCodeBox v5.9.0, Harbor, secrets, image signing | Kustomize / Helm / YAML | 0 |

`vf-admission`, `vf-graph` and `vf-hook-notify` are in TDD §2.3 and in the Phase 1 build
order, but **do not exist in the `vulcanflow` org yet**; they need creating alongside the
rest of the Phase 1 set.

## Build order

Pure library crates first, then the services that consume them, then the Kubernetes
surfaces — so that the safety-critical state machines are testable before anything can call
them:

1. `vf-core` (domain types; scope, allowance and finding state machines), `vf-graph`,
   `vf-db`'s tenant-scoped transaction wrapper
2. `vf-translator`, `vf-authz`
3. `vf-api` (with the dispatcher module), `vf-ingest`, `vf-meter` core
4. `vf-operator`, `vf-admission`, `vf-hook-notify`
5. Then §24.2 order: scanners, SPA, reporting, commercial operations

## Test discipline

TDD §25 lists roughly 45 named test identifiers (`authz/configured-scope`,
`usage/failure-retry-settlement`, `report/xss-network-corpus`, `graph/native-wasm-parity`,
…). Each is a required test, and **§23.2 is explicit that listing one does not assert it
passes**. A named test that exists and fails is the correct early state; progress is
measured by test IDs turning green, not by lines written.

Writing code, writing tests, and running tests are separate roles. A failing test is a
defect in the code — nobody weakens, skips, `#[ignore]`s or rewrites a test to make a suite
go green. If a test is genuinely wrong, that is an escalation, not a silent edit.

## Open work

Delivery is tracked in Paperclip. The remaining GitHub issues in this repo are bootstrap
tasks, all retitled for Rust after [ADR-0001](./decisions/ADR-0001-retire-go-scaffold-vf-api.md):

- [#4 Bootstrap `infra` repo (Phase 0 foundation)](https://github.com/vulcanflow/docs/issues/4) — held pending the cluster go/no-go decision
- [#13 Define org-wide CI conventions](https://github.com/vulcanflow/docs/issues/13) — Rust / frontend / infra lanes, no Go lane
- [#14 Wire active-phase repos into the code-forge config](https://github.com/vulcanflow/docs/issues/14)
- [#10 Bootstrap `scanners`](https://github.com/vulcanflow/docs/issues/10), [#11 Bootstrap `vf-web` shell](https://github.com/vulcanflow/docs/issues/11), [#12 Scaffold later-phase service READMEs](https://github.com/vulcanflow/docs/issues/12) — Phase 2+

Open TDD §27 items that are still not decided are listed in
[`decisions/README.md`](./decisions/README.md#still-open).

## Domain

- Product domain: `vulcanflow.io`
- GitHub org: [`vulcanflow`](https://github.com/vulcanflow)
