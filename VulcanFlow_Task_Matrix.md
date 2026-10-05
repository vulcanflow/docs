<!--
Source: Paperclip issue VFL-8, document `task-matrix` (revision 4d6f7f86-ed53-446e-999b-7ad261d68ab0), authored by Cortana (architect), 2026-10-05.
Links of the form /VFL/issues/... point to Paperclip issues and their documents, not to paths in this repository.
Companion file in this repository: `VulcanFlow_Architecture.md` (architecture).
Body below is identical to the issue document revision named above.
-->

# VulcanFlow task matrix: projects and first coding tasks

Companion to the [architecture document](/VFL/issues/VFL-8#document-architecture); section references `§A…` point there, `§…` to the TDD. Shaped per the owner's direction of 2026-10-05: projects first, tasks grouped under each, cross-project dependencies at the end. MasterChief creates one Paperclip project per heading in part 1 and the tasks under it; nothing here is created by Cortana.

Conventions used by every task:

- **Owner** is one of Fred, Kelly, Jorge, Linda, Halsey. **Priority** P0 = first wave, P1 = starts when its blockers close, P2 = fills idle capacity.
- **Blockers** name tasks that must be `done` first. "None" tasks state why they can start now.
- **Local verification** is the proof on a developer machine: `just dev-up` (Postgres 17 + PgBouncer + RustFS + Valkey in docker compose), `cargo test`, and fixtures. No task needs Kubernetes, Aether, Harbor, Argo CD or hosted CI. A `needs real infrastructure later` note marks follow-up work; it never blocks.
- **Gates** (team setup 1–7) apply unchanged: the coder commits production code on a local branch; Halsey's tests land on a separate test-only branch (the `platform` lane gate refuses mixed pull requests); Test Runner runs the pair; Arbiter (CodeRabbit) and Opus Reviewer review the same pair; Cortana approves the push naming both revisions. Nothing is pushed before that.
- **Test sequencing.** Tests are always a separate deliverable from the code, so Halsey has her own project (part 1.9) with one test pack per contract area rather than one task per coding task. Each coding task's first local commit is its public API skeleton (types and signatures per the architecture contract, bodies returning `Err(NotImplemented)`); the coder comments "API skeleton committed" on their task, which is Halsey's trigger to start the matching pack. Packs are therefore blocked on the contract, not on the coding task being `done`.
- **Definition of done for every coding task**, in addition to its own acceptance criteria: `just check` passes (fmt, clippy `-D warnings`, cargo deny, cargo audit); the crate builds with `#![forbid(unsafe_code)]`; no `#[cfg(test)]` in `src`; the matching Halsey pack passes under Test Runner; both reviews recorded with no finding above LOW; residual LOW findings listed in the task; Cortana's push approval recorded for the exact pair.

---

## Part 1. Projects

### 1.1 Project `platform-foundation`

- **Goal.** Stand up the `platform` Cargo workspace exactly as the component mapping in §A1.3 prescribes, with the pinned toolchain and crate set of §A5, and give every developer a one-command local environment (compose stand-ins, fixtures, test support crate) so that all later work builds, runs and is tested without any cluster.
- **Definition of done.** `git clone platform && just dev-up && just db-migrate && just check && just test-unit && just test-integration` succeeds on a clean Linux or macOS machine on x86_64 and arm64; every crate in §A1.3 exists and compiles; `vf-graph` builds for `wasm32-unknown-unknown`; the lane gate self-test passes locally; the adapter ports of §A6.1 have their local and test implementations; `vf-testkit` provides Postgres bootstrap, fixtures, deterministic clock and dev tokens.
- **Lead.** Jorge. **Repository.** `platform`.

#### F1. Workspace scaffold and pins

- **Owner / priority.** Jorge, P0.
- **Scope.** Root `Cargo.toml` with `[workspace]` listing every crate of §A1.3 (`vf-core`, `vf-db`, `vf-graph`, `vf-translator`, `vf-authz`, `vf-meter`, `vf-remediation`, `vf-api`, `vf-operator`, `vf-admission`, `vf-ingest`, `vf-hook-notify`, `vf-scanner-adapter`, `vf-report`, `vf-abuse`, `vf-testkit`) as compiling empty crates with the `lib`/`bin` kinds given there; `[workspace.dependencies]` pinned to §A5 (exact `=` versions), committed `Cargo.lock`; `rust-toolchain.toml` (1.99.0, components rustfmt, clippy, targets aarch64-unknown-linux-gnu and wasm32-unknown-unknown); `deny.toml` (licence allow-list, advisories deny, crates.io only, bans for `openssl-sys`, and the crate-graph rules of §A1.4 expressed as bans with `wrappers`); `clippy.toml` and `rustfmt.toml`; `#![forbid(unsafe_code)]` and the deny-lints of §A5 in every crate root; `justfile` with `check`, `test-unit`, `test-integration`, `gate`, `wasm` recipes; `README.md` reproducing the §A1.3 mapping table and the lane rules; `.github/workflows/rust-check.yml` running `just check` and `just test-unit` on GitHub-hosted `ubuntu-latest` (informational only; no task depends on it).
- **Acceptance criteria.** `cargo build --workspace --all-targets` and `cargo clippy --workspace --all-targets -- -D warnings` pass; `cargo deny check` and `cargo audit` pass; `cargo build -p vf-graph --target wasm32-unknown-unknown` passes and `vf-graph` has no I/O crate in `cargo tree -p vf-graph --target wasm32-unknown-unknown`; `cargo tree -p vf-core` contains no tokio/sqlx/reqwest/kube; `ci/lane-gate-test.sh ci/lane-gate.sh` passes locally; crate list equals §A1.3 (no `vf-store`, no extra crates); every pin equals §A5 or the deviation is recorded in the task with the reason; utoipa emits `openapi: 3.1.0` from a hello-world handler (records §27-17 evidence).
- **Provides.** The workspace every other task edits; `vf-core::ports` module skeleton (§A6.1 trait names only). **Consumes.** §A1.3, §A5.
- **Blockers.** None: the repository is empty on `main`, the mapping and pins are fixed in the architecture document, and open decision O1 proceeds on the proposed set.
- **Local verification.** The acceptance commands above on the developer machine; no services needed.
- **Why separate.** Hard dependency for every other `platform` task; single owner; review boundary (pins are reviewed once here).
- **Needs real infrastructure later.** arm64 release images, Harbor signing, SBOM publication (Phase 0 items not needed locally).

#### F2. Local dev harness and `vf-testkit`

- **Owner / priority.** Jorge, P0.
- **Scope.** `docker-compose.yml` with `postgres:17` (with `pgvector` extension image), `pgbouncer` (transaction pooling, pointing at it), `rustfs/rustfs`, `valkey/valkey`, profiles `minio`, `keycloak`, `mailpit`; `.env.example`; `just dev-up/dev-down/db-migrate/db-template/run-local/kind-up` (the last prints "reserved for later infrastructure work" and exits 0); `vf-testkit` with `TestDb` (testcontainers or compose Postgres, applies migrations from `vf-db`, creates a template tenant schema, returns pools through PgBouncer and direct), `fixtures::load(name)`, `DeterministicClock`, `SeqIdGen`, `token(tenant, role, claims)` minting against the dev issuer keys, `entitlements::fixture()` labelled illustrative; `crates/vf-testkit/fixtures/` tree with a README.
- **Acceptance criteria.** `just dev-up` on a clean machine brings all default services healthy within two minutes; `vf-testkit::TestDb::new().await` yields a migrated database usable from an integration test; the S3 conformance test (part of pack T4) passes against both `rustfs` and the `minio` profile with path-style addressing; `just db-template` regenerates `.sqlx/` deterministically; the `keycloak` profile is documented but not required by any recipe.
- **Provides.** The environment named in every "Local verification" line below. **Consumes.** F1; migrations from C1 (until C1 lands, the testkit applies an empty migration set and the acceptance test for `TestDb` is re-run when C1 closes).
- **Blockers.** F1.
- **Local verification.** `just dev-up && cargo test -p vf-testkit --features integration`.
- **Why separate.** Parallel deliverable with the library crates; different reviewers' focus (operational tooling vs domain code).

#### F3. Infrastructure adapters: `ArtifactStore` and `WakeBus`

- **Owner / priority.** Jorge, P0.
- **Scope.** In `vf-db::adapters`: `ArtifactStore` trait finalization in `vf-core::ports` (`put(key, bytes|stream, sha256)`, `get`, `head`, `list(prefix)`, `delete`, tenant-prefix scoping wrapper `TenantArtifactStore` that refuses keys outside `{tenant_id}/`), implementations `S3ArtifactStore` (object_store, path-style, explicit checksum mode, endpoint/region/credentials from config), `LocalFsArtifactStore`, `InMemoryArtifactStore`; `WakeBus` trait (`publish(topic, bytes)`, `subscribe(topic) -> Stream`, `token_bucket(key, rate, burst) -> Permit`), implementations `RedisWakeBus` (connection manager, reconnect) and `InProcessWakeBus`; adapter selection by `VF_ARTIFACT_STORE`/`VF_WAKE_BUS`; production-refusal of `memory`/`inprocess` under `VF_ENV=production`.
- **Acceptance criteria.** The same conformance test module runs against all three artifact stores and both buses (pack T4) and passes; a key outside the tenant prefix is refused before any I/O; S3 multipart upload works for a 100 MiB object against RustFS; `RedisWakeBus` recovers from a Valkey restart without losing the subscription (test restarts the container); the production refusal is tested.
- **Provides.** §A6.1 adapters for ingest, API SSE, operator. **Consumes.** F1 ports skeleton.
- **Blockers.** F1 (compiles against the port skeleton; F2 needed only to run the integration tests, which Test Runner executes after F2 closes).
- **Local verification.** `cargo test -p vf-db --features integration adapters::` with compose up.
- **Why separate.** Different owner from the rest of `vf-db` (Fred); parallel deliverable; review boundary (infrastructure conformance).
- **Needs real infrastructure later.** Ceph RGW conformance and scoped per-tenant credentials on Aether.

### 1.2 Project `core-libraries`

- **Goal.** Deliver the safety-relevant domain logic of §2.5.2 as I/O-free Rust (`vf-core`), the tenant-safe data access layer (`vf-db`), integer allowance accounting (`vf-meter`) and guidance/verification logic (`vf-remediation`), so that services are thin shells over property-tested libraries.
- **Definition of done.** All contracts in §A3.1, §A3.2, §A3.7 and the data model of §A4 are implemented; packs T1, T2 (Kelly's module), T4 pass; RLS and `TenantTx` isolation tests pass through PgBouncer; the four-units/three-units accounting example passes concurrently; `vf-core` and `vf-graph` remain free of I/O dependencies.
- **Lead.** Fred. **Repository.** `platform`.

#### C1. `vf-core` identities, state machines, policy and Problem catalogue

- **Owner / priority.** Fred, P0.
- **Scope.** `vf-core::ids` newtypes (§A3.1, uuid v7, serde, Display/FromStr); `vf-core::state` enums and transition functions exactly as §A3.2; `Role`/`Action` matrix with one exhaustive match covering every cell of §4.2 (`viewer` excluded at GA); `vf-core::events` (§A3.3) ; `vf-core::problem` enum with one variant per stable `type` URI listed in §A3.8 (authorization-required with track hints, target-allowance-exceeded, graph-invalid, idempotency-conflict, illegal-transition, challenge-failed, not-found, forbidden, rate-limited, internal) carrying status and a `vf` extension object; `vf-core::ports` trait definitions (`Clock`, `IdGen`, `ArtifactStore`, `WakeBus`, `ScanRuntime`, `TenantProvisioner`, `TokenVerifier`, `ChallengeProbe`) with doc comments; no I/O crates.
- **Acceptance criteria.** Every enum round-trips `as_str`/`from_str` and errors on unknown strings; `observation_transition` admits exactly the edges of §15.1 and `apply_verification` maps `Inconclusive` to the prior state; `outcome_consumes` is true only for `Success`; `pipeline_outcome` returns `None` while discovery is open or fan-in is pending and otherwise derives the §8.2 state from unit summaries; the Problem enum serializes to RFC 9457 JSON with `type`, `title`, `status`, `detail`, `instance`, `vf`; `cargo tree -p vf-core` shows no I/O crate.
- **Provides.** §A3.1 ids, §A3.2, §A3.3 event types, Problem catalogue, ports. **Consumes.** F1.
- **Blockers.** F1.
- **Local verification.** `cargo test -p vf-core` (pack T1).
- **Why separate.** Hard dependency for every service crate; review boundary for the state machines.

#### C2. `vf-core::scope`: canonical hosts, scope matching, IPv4 destinations

- **Owner / priority.** Kelly, P0.
- **Scope.** `CanonicalHost` (idna UTS-46 non-transitional to A-label, lowercase, trailing-dot strip, label length and charset rules, rejects IP literals, ports, paths, public-suffix roots using `psl` including private suffixes), `registrable_domain`, `is_descendant_of` (dot boundary), `ApprovedScope`/`RunScope`/`ScopeVerdict`, `check_run_scope`, `check_candidate`, `Ipv4Destination` (rejects RFC 1918, loopback, link-local, multicast, reserved, broadcast, 0.0.0.0/8, CGNAT 100.64/10), a `HostError` enum with a reason per rejection.
- **Acceptance criteria.** The corpus in pack T2 (IDNA, mixed case, trailing dots, `xn--` inputs, private suffixes such as `*.github.io`, label-prefix lookalikes `notexample.com` vs `example.com`) passes; property tests show `check_candidate` never admits a host outside a `DomainTree` root or different from an `ExactHost` root, and never admits descendants when `include_subdomains` is false; fuzz targets `canonical_host` and `scope_match` run 10 minutes locally without panic; module has no dependency outside idna, url, psl, ipnet, thiserror, serde.
- **Provides.** §A3.1 scope types used by `vf-authz`, `vf-translator`, `vf-meter`, `vf-admission`. **Consumes.** F1.
- **Blockers.** F1 (module lives beside C1 in the same crate; separate pull requests avoid conflict because the modules are disjoint).
- **Local verification.** `cargo test -p vf-core scope::`; `cargo +nightly fuzz run canonical_host -- -max_total_time=600`.
- **Why separate.** Different owner (security control owned by Kelly); review boundary (threat model rows "subdomain approval expands", "annotation differs from actual target").

#### C3. `vf-db` schemas, migrations, `TenantTx`, repositories, outbox

- **Owner / priority.** Fred, P0.
- **Scope.** Migrations under `crates/vf-db/migrations/{control,tenant}` creating every table in §A4 with the §3.5 RLS policy on each tenant table and the `REVOKE`s of §5.6, §17.3, §19.2; roles `vf_app` and `vf_migrate`; `TenantTx` (sole constructor `TenantTx::begin(pool, VerifiedTenant)` setting `search_path` and the GUC transaction-locally) and `ControlTx`; repository modules for tenants, projects, targets, authorization (challenges, basis, events), pipeline runs, work units, scans, findings, finding states, FP decisions/events/applications, verification runs, usage tables, pipeline/tenant events, outbox (claim with `FOR UPDATE SKIP LOCKED`, lease, reclaim, complete), audit log append; `vf-db migrate` CLI (`--control`, `--tenant <slug>`, `--all-tenants` resumable with `vf.tenant_migrations`); committed `.sqlx/` offline data; `provision_tenant_schema(slug)` used by the local provisioner.
- **Acceptance criteria.** `SQLX_OFFLINE=true cargo build -p vf-db` passes; a query on a tenant table without `TenantTx` does not compile (trybuild test in pack T4); with the wrong GUC a tenant query returns zero rows and an insert fails `WITH CHECK`; through PgBouncer transaction pooling, a connection reused after commit, rollback and error carries no tenant context; `vf_app` cannot `UPDATE`/`DELETE` on `authorization_basis`, `scan_usage_ledger`, `vf.audit_log`; outbox claim/lease/reclaim behaves under two concurrent workers; `--all-tenants` resumes after being killed mid-way; every §A4 table and constraint exists (schema snapshot test).
- **Provides.** §A4, `TenantTx`, outbox API, repositories used by every service. **Consumes.** C1, C2 types; F2 for integration runs.
- **Blockers.** C1, C2.
- **Local verification.** `cargo test -p vf-db --features integration` with compose up (pack T4).
- **Why separate.** Hard dependency for `vf-meter`, `vf-authz`, services; review boundary (isolation suite).
- **Needs real infrastructure later.** CNPG, PgBouncer deployment, PITR drills.

#### C4. `vf-meter` accounting library

- **Owner / priority.** Fred, P0.
- **Scope.** Exactly §A3.7: move-only `Reservation`, `open_period`, `admit_target`, `reserve`, `settle_success`, `release`, `reservation_for`, `usage`; billing-period attribution from the entitlement snapshot; ledger idempotency keys of the form `{work_unit_id}:{event}`; `UsageSnapshot` and the `allowance.updated` tenant event emission within the same transaction.
- **Acceptance criteria.** Under 50 concurrent tasks reserving against a period with limit 10, exactly 10 succeed and `remaining` never goes negative; a second `settle_success` for the same unit is a compile error for the moved value and a database error when re-hydrated; a released unit cannot be consumed; a retry (new `scans` row, same unit) holds the reservation; the §17.1 example yields 4 and the one-failed variant yields 3; `admit_target` reuses canonical duplicates and records a skipped admission when the target allowance is exhausted; each ledger row is immutable.
- **Provides.** §A3.7 for dispatcher, operator, ingest. **Consumes.** C1, C2, C3.
- **Blockers.** C3.
- **Local verification.** `cargo test -p vf-meter --features integration` (pack T1).
- **Why separate.** Review boundary (atomicity and accounting invariants, M1 exit evidence "concurrent runs cannot overspend"); consumed by three services owned by two developers.
- **Needs real infrastructure later.** Lago/Stripe sync binary (Phase 4).

#### C5. `vf-remediation` library: guidance resolution, verification plan and outcome

- **Owner / priority.** Fred, P1.
- **Scope.** `resolve_guidance(finding_class, template_text: Option, cwe_ids) -> Resolved{tier, version, content|unavailable}` over `vf.remediation_content` with the three-tier order of §15.2 and version pinning; `derive_verification_plan(finding, approval) -> VerificationPlan` producing a `FlowGraph` of `kind: verification` as Appendix B (liveness dnsx and httpx plus the locked check), target locked to the original observation's host and endpoint, refusing anything outside the live approval; `interpret_outcome(plan, unit_results, evidence) -> VerificationOutcome` applying the §15.4 table (only an applicable, completed check yields `not_detected`/`still_present`); a fixture content set of ten curated classes for local use, labelled unreviewed.
- **Acceptance criteria.** Resolution returns `unavailable` rather than fabricated text when no tier matches; the plan for a nuclei finding on `https://api.example.com/admin` locks exactly that host and endpoint and contains three units; blocked, skipped, missing-template, missing-allowance and liveness-only results yield `inconclusive`; version pinning is stable across content edits.
- **Provides.** Guidance and verification contracts for A4, S3. **Consumes.** C1, C3, G1 types.
- **Blockers.** C3, G1.
- **Local verification.** `cargo test -p vf-remediation` (pack T6 partial).
- **Why separate.** Parallel deliverable behind the API; review boundary (`verify/applicable-check-evidence`).

### 1.3 Project `graph-and-execution-contracts`

- **Goal.** Build the shared graph rule set that runs natively and in the browser, the translator that turns a validated graph into reserved, scoped, typed secureCodeBox objects and safe scanner arguments, Track A authorization with the single start-barrier decision, the admission handler, and the two binaries that live in scanner images. This project is where "no execution without authorization, scope, reservation and evidence" is implemented.
- **Definition of done.** §A3.4, §A3.5 and the adapter/hook halves of §A3.6 implemented; packs T2, T3, T5, T9 pass; the golden corpus is shared with `vf-web`; `decide()` is the only barrier used by dispatcher, operator and admission; SCB types are generated from the pinned release; argv construction cannot be reached with an unvalidated string.
- **Lead.** Kelly. **Repository.** `platform`.

#### G1. `vf-graph` validator, schema and WASM export

- **Owner / priority.** Kelly, P0.
- **Scope.** Exactly §A3.4 `vf-graph`: DSL types with `deny_unknown_fields`, per-node config structs with no command/env/template-path/target override fields, the type lattice table of §7.2 as one exhaustive match, `validate` (acyclic, edge types, required inputs, finding sink, config validity, `Entitlements` check, node cap 200, duplicate ids), `json_schema()` via schemars matching Appendix A, `validate_json` behind `wasm-bindgen` for `wasm32`, `just wasm` producing `pkg/` via wasm-pack with a size check (300 KiB gzipped proposed).
- **Acceptance criteria.** The corpus `conformance/graph/*.json` with golden results (pack T3) passes natively; `validate_json` compiled to WASM run under `wasm-bindgen-test` (node) produces byte-identical JSON for the same corpus; the generated schema snapshot is committed and a drift test fails on change; `cargo tree -p vf-graph --target wasm32-unknown-unknown` shows no I/O crate; `.wasm` size is within budget.
- **Provides.** §A3.4 types for translator, dispatcher and `vf-web`; the golden corpus for W5. **Consumes.** F1.
- **Blockers.** F1.
- **Local verification.** `cargo test -p vf-graph && just wasm && wasm-pack test --node crates/vf-graph`.
- **Why separate.** Parallel deliverable consumed by two developers (Fred's dispatcher, Linda's SPA); review boundary (`graph/native-wasm-parity`).

#### G2. `vf-translator`: candidates, typed SCB objects, argv builders

- **Owner / priority.** Kelly, P1.
- **Scope.** Exactly §A3.4 `vf-translator`: `root_candidates`, `estimate`, `derive_candidates` (every discovered host passes `check_candidate` or becomes `Skipped{Scope}`), `scan_spec`, `cascading_rules`, `argv` per node type for subfinder, dnsx, httpx, nuclei (tlsx, nmap, masscan builders stubbed `NotSupportedYet` until M2); SCB CRD types generated with kopium from the v5.9.0 CRD YAML (O2) committed under `src/scb/` with a `just scb-types` regeneration recipe; `MaterializedInputs` (validated hostnames written as an input file by the adapter); deterministic `CandidateKey` = `{node_id}:{host}:{port?}:{proto?}:{endpoint_sha256?}`.
- **Acceptance criteria.** Golden argv files per node and config (pack T3) match; any argument element starting with `-` that originates from input data is refused; argv never contains a host that did not come from `CanonicalHost`/`Ipv4Destination` (type-level, plus a test that a raw string cannot be passed); serialized `Scan`/`CascadingRule` snapshots carry the run, unit and node labels and the target annotation; cascade candidates from a fixture subfinder output outside scope are all `Skipped{Scope}`; `estimate` marks discovery uncertainty when any node has dynamic fan-out; re-running `just scb-types` yields no diff.
- **Provides.** Plans and specs for A3 (dispatcher), O3 (operator), G5 (adapter). **Consumes.** C1, C2, G1.
- **Blockers.** C2, G1 (C1 for ids).
- **Local verification.** `cargo test -p vf-translator` (pack T3). No cluster; CRD types come from the checked-in YAML.
- **Why separate.** Hard dependency for operator and dispatcher; review boundary (§22.3 argument construction).
- **Needs real infrastructure later.** Release-specific cascade integration test on a cluster (M2).

#### G3. `vf-authz`: Track A challenges, basis lifecycle, start-barrier decision

- **Owner / priority.** Kelly, P0.
- **Scope.** Exactly §A3.5: challenge issue/verify for DNS-TXT and HTTP file with the §5.2 rules (256-bit tokens, 7-day expiry, single use, three public resolvers plus authoritative, at most one same-host redirect, TLS verification on, 64 KiB body cap, private/link-local destination refusal, timeouts), manual review recording, `resolve_live_basis` with 90-day validity, 14-day grace, immediate revocation, `revoke` writing `authorization_events` and enqueuing outbox `cancel_run` for every non-terminal run of the target; `decide()` over `BarrierInput` with every `RefuseReason`; `ChallengeProbe` real implementation (hickory-resolver, reqwest) and `FixtureProbe`.
- **Acceptance criteria.** Pack T5 matrix passes: valid DNS-TXT, missing record, wrong token, expired challenge, reused challenge, HTTP downgrade, cross-host redirect, second redirect, private destination, oversized body; `resolve_live_basis` temporal cases at 89/90/104/105 days; `decide()` refuses on every `None` and every reason, admits only when basis is `Live`, scope matches, reservation is `Reserved`, tenant not suspended, observed target and parameters equal the trusted record; evidence hash recorded on the basis; `authorization_basis` rows are never updated or deleted by the app role.
- **Provides.** §A3.5 for the API (A2), operator (O3), admission (G4). **Consumes.** C1, C2, C3, F2 fixtures (local DNS server, wiremock).
- **Blockers.** C2, C3.
- **Local verification.** `cargo test -p vf-authz --features integration` with compose up; DNS served by `vf-testkit`'s hickory test server, HTTP by wiremock.
- **Why separate.** Review boundary (the control of §1.2 item 1); hard dependency for three services.

#### G4. `vf-admission` validating handler

- **Owner / priority.** Kelly, P1.
- **Scope.** axum handler for `AdmissionReview` v1 (`k8s-openapi` types) on `/validate/scan`, `/validate/job`, `/validate/pod`; decodes the object, resolves the trusted work unit from labels `vulcanflow.io/work-unit-id` via `TenantTx` opened from the work-unit record (tenant identity never from the object), builds `ObservedWorkload` (scan type, parameters, env, init containers, volumes, service account, target annotation, input file digest) and calls `decide()`; fail-closed on any error including database unavailability; dry-run never writes; rustls TLS from files; `vf-admission` binary with health endpoint; generated `ValidatingWebhookConfiguration` YAML (`failurePolicy: Fail`) emitted by `vf-admission print-config` for later use by `infra`.
- **Acceptance criteria.** Fixture `AdmissionReview` corpus (pack T5): admitted Scan matching its unit; refused for unknown unit, missing label, target annotation ≠ record, extra `parameters`, changed `scanType`, added init container, different service account, suspended tenant, released reservation, finalized run; a Job and a Pod derived from the fixture Scan get the same verdicts; database error yields `allowed: false` with a reason; dry-run leaves no row.
- **Provides.** The Kubernetes-side half of the barrier (used by `infra` later). **Consumes.** C3, G3.
- **Blockers.** G3.
- **Local verification.** `cargo test -p vf-admission --features integration` with compose Postgres; fixtures only, no cluster.
- **Why separate.** Separate binary and review boundary (`authz/start-barrier-all-paths`); can be built while the operator is in progress.
- **Needs real infrastructure later.** Registration on a `kind` cluster, then Aether; `execution/gates-before-start` with the webhook disabled (§23.4.1).

#### G5. `vf-scanner-adapter` and `vf-hook-notify`

- **Owner / priority.** Kelly, P1.
- **Scope.** `vf-scanner-adapter` subcommands `materialize` (writes the validated host list from a signed `WorkUnitEnvelope` file to the scanner input path), `exec --node-type <t> --config <json> --inputs <dir>` (builds argv through `vf-translator::argv`, runs the upstream tool via `Command::args` with a wall-clock budget, captures raw output to `raw/{node}-{attempt}.out`, classifies `outcome_class` from documented exit codes and output markers per tool, writes `manifest.json` per §A3.6); `vf-hook-notify` building the notification body of §A3.6, signing with HMAC-SHA256 from a key file, posting with bounded retries and jittered backoff, exit non-zero on final failure; both as static binaries (`-C target-feature=+crt-static` where supported) on `scratch`-style Dockerfiles kept in `platform/docker/` for local builds only.
- **Acceptance criteria.** With a stub tool script in `vf-testkit/fixtures/stub-scanners/` the adapter produces the golden `manifest.json` for success, non-zero exit, timeout and empty output; a target that begins with `-` never reaches the tool; signature verifies with the ingest verifier (same test vector in both crates); replayed notification has a fresh nonce; `docker build` of both images succeeds locally for the host architecture.
- **Provides.** Execution-side half of §A3.6, consumed by the fake runtime (O2) and the `scanners` repo. **Consumes.** C1, C2, G2.
- **Blockers.** G2.
- **Local verification.** `cargo test -p vf-scanner-adapter -p vf-hook-notify`; `docker build -f docker/vf-hook-notify.Dockerfile .`.
- **Why separate.** Deployable artifacts with their own review boundary (memory-safe handling of attacker-influenced artifact references, §8.4); consumed by another repository.
- **Needs real infrastructure later.** SCB hook invocation conformance for the pinned release (§27-20), arm64 images in Harbor.

### 1.4 Project `api`

- **Goal.** Deliver `vf-api`: the single REST and SSE surface of §A3.8, generated OpenAPI 3.1, authentication and the role layer, and the dispatcher module that turns an authorized request into durable, reserved, outboxed work. Everything the SPA and the operator need from the control plane goes through here.
- **Definition of done.** Every endpoint in §A3.8 marked in scope below responds per contract; the committed OpenAPI file equals the generated one; packs T6 and T7 pass; the walking skeleton end-to-end test passes on `just run-local` with the fake runtime; a browser disconnect does not affect a running pipeline.
- **Lead.** Fred. **Repository.** `platform`.

#### A1. API skeleton: app, OpenAPI, auth, roles, tenant context, errors

- **Owner / priority.** Fred, P0.
- **Scope.** axum application with utoipa router and `GET /openapi.json`; `openapi/vf-api.json` committed with a drift test; `TokenVerifier` port with JWKS implementation (jsonwebtoken, cached keys, `iss`/`aud` checks) and the `dev-auth` feature with `vf-api dev-issuer` (§A6.4); extractor `VerifiedTenant` from claims plus live `vf.tenants.suspended_at` check on write methods; role policy tower layer mapping route+method to `Action` and calling `vf_core::allowed`; `Problem` → `IntoResponse`; tracing with request and trace ids; config via figment (`VF_*` env), adapter selection per §A6.1, production refusal of dev/fake adapters; `/healthz`, `/readyz` (DB and bus reachable); rate limit middleware using `WakeBus::token_bucket` per `(tenant, endpoint-class)` with `429` + `Retry-After` + `RateLimit-*` headers (classes read, dispatch, verification, report).
- **Acceptance criteria.** Pack T7: every route×role cell of §4.2 behaves (member cannot manage members, cannot cancel another user's run); a valid JWT for a suspended tenant is refused on write paths with `403 tenant-suspended` and allowed on reads; expired, wrong-issuer and wrong-audience tokens are refused; Problem bodies validate against RFC 9457 shape and the catalogue; drift test fails when a handler changes without the committed spec; `VF_ENV=production VF_AUTH=dev` refuses to start.
- **Provides.** The app shell every endpoint task extends; `openapi/vf-api.json` for `vf-web`. **Consumes.** C1, C3 (`TenantTx`), F3 (bus for rate limits; an in-process bus suffices for tests).
- **Blockers.** C1, C3.
- **Local verification.** `cargo test -p vf-api` plus `cargo run -p vf-api --features dev-auth -- dev-issuer` and a curl smoke documented in the README.
- **Why separate.** Hard dependency for every endpoint task; review boundary (authn/authz layer).
- **Needs real infrastructure later.** Keycloak realm configuration.

#### A2. Targets, projects and authorization endpoints

- **Owner / priority.** Fred, P1.
- **Scope.** `POST/GET /v1/targets`, `GET /v1/targets/{id}`, `POST …/challenges`, `POST …/challenges/{id}/verify`, `POST …/manual-review`, `DELETE …/authorization`, `GET/POST/PUT /v1/projects` per §A3.8, calling `vf-authz`; audit rows for registration, challenge issue, verification, manual approval, revocation.
- **Acceptance criteria.** Registering `Example.COM.` and `example.com` yields one target; a public suffix is refused with `422 invalid-hostname` and a reason; the challenge response shows the token once; verify with the fixture probe produces a `Live` basis visible on `GET /v1/targets/{id}`; revocation returns `204`, the target's `authorization` becomes `Revoked`, and an outbox `cancel_run` row exists for each non-terminal run; member role can register and verify, only admin can manual-review.
- **Provides.** Target and authorization API for W2. **Consumes.** A1, G3.
- **Blockers.** A1, G3.
- **Local verification.** `cargo test -p vf-api --features integration targets::` with compose up and the fixture probe.
- **Why separate.** Parallel deliverable with A3 (different contract area, same owner but independent reviewers' focus); first slice the SPA can consume.

#### A3. Dispatcher module, outbox worker, run and usage endpoints

- **Owner / priority.** Fred, P0.
- **Scope.** `vf-api::dispatcher`: `POST /v1/graphs/validate`, `POST /v1/graphs/estimate`, `POST /v1/pipeline-runs` implementing §8.1 steps 1–4 in one transaction (resolve live basis, scope check, `vf_graph::validate` with the tenant's entitlement snapshot, normalize and hash the request bound to `Idempotency-Key`, insert `pipeline_runs` with graph and scope snapshots, `root_candidates` → `scan_work_units`, `vf_meter::reserve` per root unit or `Skipped{Limit}`, audit intent, outbox `submit_run`, `pipeline.state` and `allowance.updated` events), `GET /v1/pipeline-runs/{id}` (`RunSnapshot`), `DELETE /v1/pipeline-runs/{id}` (durable `stop_requested_at` then outbox `cancel_run`), `GET /v1/usage`, `GET /v1/usage/ledger`; outbox worker loop in `vf-api` claiming `submit_run`/`cancel_run` and calling the `ExecutionBackend` (local: in-process call into `vf-operator`'s lib; kube: creates/deletes the `ScanFlow` CR, feature-gated); a fixed template `skeleton-v1` (`subfinder → dnsx → httpx → nuclei`) seeded in `vf.templates`.
- **Acceptance criteria.** Pack T6/T7: same key and body twice returns the same run with `200`; same key different body returns `409`; unapproved target returns `403 authorization-required` with track hints and writes no row; invalid graph returns `422` and reserves nothing; a run against a period with 2 remaining units registers 4 root units, reserves 2 and records 2 `Skipped{Limit}` with reasons; `GET /v1/usage` reflects `reserved` immediately; cancel on a `Running` run sets `stop_requested_at` and emits `cancel_run`; outbox worker crash before completion is recovered by lease expiry without a second `submit_run` effect (deterministic run identity).
- **Provides.** Run submission for the operator (O3) and SPA (W3); usage for W3 and the WASM validator. **Consumes.** A1, C3, C4, G1, G2, G3, O1 (`ExecutionBackend` local impl).
- **Blockers.** A1, C4, G2, G3, O1.
- **Local verification.** `cargo test -p vf-api --features integration dispatcher::` with compose; `just run-local` smoke.
- **Why separate.** Hard dependency for the operator loop and the SPA; review boundary (atomic submission, M1 "concurrent runs cannot overspend").
- **Needs real infrastructure later.** `ScanFlow` CR creation on a cluster.

#### A4. SSE streams and durable pipeline events

- **Owner / priority.** Fred, P1.
- **Scope.** `GET /v1/pipeline-runs/{id}/events` and `GET /v1/events` per §A3.3: events read from `pipeline_events`/`tenant_events`, `id: seq`, `Last-Event-ID` replay within 15 minutes, `resync` with `RunSnapshot` beyond it, keep-alive comments every 15 seconds, per-connection bounded buffer, wake via `WakeBus` subscription with a 2-second poll fallback; log-line sampling and counter coalescing helpers used by the operator when it writes events.
- **Acceptance criteria.** Pack T7: a client reconnecting with a stale id gets exactly the missed events once; beyond 15 minutes it gets one `resync` then live events; a slow client does not block other subscribers (test with a paused reader); events are emitted only for committed rows (test: a rolled-back transaction's event never appears); a tenant cannot subscribe to another tenant's run; the SSE end-to-end latency from commit to client receipt is measured and printed (budget p75 < 1 s is informational locally).
- **Provides.** Live progress for W3. **Consumes.** A1, C3, F3.
- **Blockers.** A1, F3.
- **Local verification.** `cargo test -p vf-api --features integration sse::` with compose.
- **Why separate.** Parallel deliverable with A3; review boundary (replay and backpressure rules).

#### A5. Findings, triage, false-positive decisions, verification submission

- **Owner / priority.** Fred, P1.
- **Scope.** `GET /v1/findings` (keyset, filters of §A3.8, facet counts from Postgres for now), `GET /v1/findings/{id}`, `GET …/remediation`, `POST …/state` (transition via `vf_core::observation_transition`; `false_positive` creates the decision with the canonical `FpMatchKey`, audit row, `finding.state` event), `POST …/verify` (plan via `vf-remediation`, then the same dispatcher path with `run_kind = verification` and a `verification_runs` row), `DELETE /v1/false-positive-decisions/{id}` (`false_positive_events` revoked row), `GET /v1/scans/{fingerprint}`.
- **Acceptance criteria.** Pack T6: 10 000 fixture observations paginate with stable cursors and no OFFSET (query plan asserted); illegal transitions return `409`; a false-positive decision records the match key and version; verify on a finding whose target lost its basis returns `403`; verify creates a pipeline run whose units equal the plan and consume normal allowance; revoking a decision leaves historical applications intact; a fingerprint from another tenant returns `404`.
- **Provides.** Findings API for W4. **Consumes.** A1, A3, C5, I2 data shapes.
- **Blockers.** A3, C5.
- **Local verification.** `cargo test -p vf-api --features integration findings::` with compose and the fixture artifact corpus ingested by I2 (or directly seeded through `vf-testkit` before I2 closes).
- **Why separate.** Parallel deliverable with A4; review boundary (`findings/fp-only-persistence`, `findings/new-after-fix`).

### 1.5 Project `operator-and-ingest`

- **Goal.** Drive execution and bring results back durably: the `ScanFlow` reconcile loop that reserves, admits and tracks every scan through the `ScanRuntime` port, the two runtime implementations (fake now, kube-rs compiled and mocked for later), tenant provisioning, and the ingest service that verifies notifications, records fresh observations and settles allowances exactly once. With the fake runtime this project closes the local walking loop.
- **Definition of done.** `just run-local` executes the skeleton template end to end against fixture scanner output; packs T6 pass including cancellation, revocation, retry, late cascade and replayed artifact; `KubeScanRuntime` compiles and its serialization is tested against a mocked kube client; the 60-second sweep recovers a dropped notification.
- **Lead.** Jorge. **Repository.** `platform`.

#### O1. `ScanRuntime` implementations and the local `ExecutionBackend`

- **Owner / priority.** Jorge, P0.
- **Scope.** In `vf-operator` lib: `FakeScanRuntime` (in-memory Scans keyed by deterministic name; `create_scan` returns the existing object on repeat; a per-node script from `fixtures/runtime-scripts/*.yaml` decides phase transitions, fingerprint minting, artifact writing through `ArtifactStore` from the fixture corpus, invocation of `vf-hook-notify` as a subprocess or an in-process equivalent, injected failures: tool, platform, timeout, delayed hook, duplicate hook); `KubeScanRuntime` behind feature `kube` (kube-rs client, server-side apply with deterministic names, `metadata.uid` as fingerprint, cascading rules apply, cancel with 10-second grace); `ExecutionBackend` trait (`submit(PipelineRunId)`, `cancel(PipelineRunId)`) with `LocalExecutionBackend` (calls the reconcile entry in-process) and `KubeExecutionBackend` (creates the `ScanFlow` CR, feature `kube`); `Tenant`/`ScanFlow` CRD types via kube-derive behind the feature and `print-crds`.
- **Acceptance criteria.** Fake runtime scripts cover every §8.2 outcome; `create_scan` twice yields the same `ScanRef` and one fingerprint; `KubeScanRuntime` unit tests through a tower-test mock assert the exact `Scan` JSON applied, the label set, and that `cancel_scan` deletes with the grace period; `print-crds` output is committed under `crates/vf-operator/crds/` with a drift test.
- **Provides.** §A3.6 runtime port for O3 and A3. **Consumes.** C1, F3, G2 (`ScanSpec` types).
- **Blockers.** F3, G2 (the fake can be started against stub `ScanSpec` types and finalized when G2 lands; MasterChief may relax this blocker to F3 only).
- **Local verification.** `cargo test -p vf-operator runtime::`; `cargo test -p vf-operator --features kube` for the mocked client tests.
- **Why separate.** The swappable boundary of owner constraint 1; different owner from the translator; consumed by two services.
- **Needs real infrastructure later.** `kind` then Aether runs of `KubeScanRuntime`.

#### O2. Tenant provisioning (local provisioner and CLI)

- **Owner / priority.** Jorge, P1.
- **Scope.** `TenantProvisioner` per §A3.9: `LocalTenantProvisioner` (insert/update `vf.tenants`, `vf_db::provision_tenant_schema`, apply tenant migrations, create the artifact prefix marker, bus namespace; Kubernetes steps recorded `Skipped{no-kube}`), `suspend`/`restore` writing durable state and enqueuing cancellation for every non-terminal run; `vf-operator provision-tenant --spec file.yaml`, `suspend-tenant`, `restore-tenant` CLI; the `Tenant` CRD controller (feature `kube`) wrapping `ensure` with conditions.
- **Acceptance criteria.** Provisioning twice is a no-op with the same conditions; a provisioned tenant's schema has RLS on every table (reuses the C3 snapshot test); `suspend` makes the A1 extractor refuse writes and the barrier refuse admission within the same test; `restore` requires an explicit reason and is audited; entitlements from the spec appear in `GET /v1/usage`.
- **Provides.** Tenant creation for every integration test and `just run-local`. **Consumes.** C3, F3.
- **Blockers.** C3.
- **Local verification.** `cargo test -p vf-operator --features integration provision::` with compose.
- **Why separate.** Parallel deliverable with O1; review boundary (suspension semantics, §20.3).
- **Needs real infrastructure later.** Namespace, quota, NetworkPolicy, RBAC, pull secret steps.

#### O3. `ScanFlow` reconcile loop

- **Owner / priority.** Jorge, P0.
- **Scope.** `vf-operator::reconcile(run_id)` driven by `LocalExecutionBackend` now and the CR watch later: load the run and units in a `TenantTx` from the trusted run record; for each `Registered`/`Reserved` root unit, re-resolve the live basis, build `ScanSpec`, call `decide()`, `create_scan` through the runtime, bind the fingerprint (`scans` row, attempt 1), emit `scan.state`; poll `get_scan` and classify terminal phases; on ingest-confirmed success nothing more (ingest settles); on tool/target failure `release` and record; on platform failure retry up to 3 attempts holding the reservation (new `scans` row, same unit); cascade handling: when ingest records outputs for a unit, call `derive_candidates`, register each as a new unit with `parent_unit_id`, `reserve` or `Skipped{Limit|Scope}`, then admit through the same barrier; completion barrier per §8.2 (`pipeline_outcome` only when discovery closed, no pending fan-in, all units terminal, required ingestion complete); cancellation path (`stop_requested_at` → refuse new candidates, `cancel_scan` on running, release reservations, `Cancelled`); revocation and suspension observed on every iteration; deterministic adoption on retry after an uncertain `create_scan`; `node.counter` and `log.line` events via the A4 helpers.
- **Acceptance criteria.** Pack T6 with the fake runtime: the skeleton template completes with the expected unit count from the fixture outputs; a fixture subfinder output containing out-of-scope hosts yields `Skipped{Scope}` units and no Scan; a platform failure on attempt 1 and success on attempt 2 settles exactly one unit with two `scans` rows; cancel mid-run leaves no `Reserved` reservation and no running Scan; revocation mid-run cancels running work and refuses the next candidate; a hook arriving after finalization is refused and logged; killing the loop between `create_scan` and fingerprint binding and restarting adopts the existing Scan without a second creation; the loop never reads tenant identity from the runtime object.
- **Provides.** Execution for A3's submissions; outputs for I2. **Consumes.** C3, C4, G2, G3, O1, I2 (for the end-to-end loop).
- **Blockers.** C4, G2, G3, O1.
- **Local verification.** `cargo test -p vf-operator --features integration reconcile::` with compose; `just run-local`.
- **Why separate.** Core of M1 exit evidence; different owner from dispatcher and ingest; review boundary (atomicity, idempotency, cancellation).
- **Needs real infrastructure later.** CR watch, controlled pool, real cascade hook behaviour (§27-5).

#### I1. Notification verification and the ingest service shell

- **Owner / priority.** Jorge, P1.
- **Scope.** `vf-ingest` binary with `POST /internal/v1/notifications` implementing the §A3.6 checks in order (signature over raw body with key id lookup, timestamp window, nonce table insert as the dedup primitive, trusted work-unit lookup through a `TenantTx` opened from the record, fingerprint binding or cross-check, prefix equality, artifact fetch through `TenantArtifactStore`, checksum, size bound) and then enqueuing a durable `ingest_artifact` outbox row; the 60-second reconciliation sweep (`list_scans` per non-terminal run, compare with `scans` and ingest state, enqueue missing ingests, flag stale reservations for the operator); shared verifier module reused by `vf-hook-notify` tests.
- **Acceptance criteria.** Pack T6 verification matrix: bad signature, skewed timestamp, replayed nonce, unknown unit, fingerprint bound to a different unit, prefix outside the trusted path, checksum mismatch, oversize artifact each return the specified Problem and write nothing except the nonce; a valid notification enqueues exactly one job; a second identical valid notification returns `already_ingested`; the sweep enqueues ingest for a Scan whose hook never arrived and does not duplicate an existing job.
- **Provides.** Ingest entry point for G5 and O1's fake. **Consumes.** C3, F3, G5 (signing test vectors).
- **Blockers.** C3, F3.
- **Local verification.** `cargo test -p vf-ingest --features integration notify::` with compose and RustFS.
- **Why separate.** Review boundary (attacker-influenced input before any parse; §8.4); parallel with I2.

#### I2. Artifact parse, fresh observations, false-positive matching, settlement

- **Owner / priority.** Jorge, P1.
- **Scope.** Outbox `ingest_artifact` handler: streaming, size-bounded `serde_json` parse of the SCB findings envelope validated against a committed JSON schema (`crates/vf-ingest/schema/scb-findings-v1.json` from the pinned release), mapping to `findings` rows with `UNIQUE (scan_id, source_finding_id)` upsert-ignore, asset rows and `run_coverage`, `FpMatchKey` computation and matcher v1 application recording `false_positive_applications` and, only when `fp_persistence_enabled`, `applied_fp_decision_id` and initial state `FalsePositive`; enrichment from the local mirror tables (empty is fine); guidance resolution via `vf-remediation` pinning `remediation_id`; outcome classification from `manifest.json` → `scan_work_units.outcome_class`; `settle_success` or `release` through `vf-meter` in the same transaction as the observations; `finding.new`, `scan.state`, `allowance.updated` events; outputs (discovered hosts, endpoints) handed to the operator for cascades; malformed or schema-failing artifacts classify the unit `Platform` and never `Target`.
- **Acceptance criteria.** Pack T6: ingesting the fixture nuclei artifact twice yields one set of rows and one `consume` ledger row; two different scans with identical findings yield distinct `finding_id`s and `scan_id`s; a zero-finding successful artifact settles success; a malformed artifact releases the reservation with `Platform`; an active FP decision matches only on full key equality (host, class, port, path) and never across tenants; with the flag off the application is recorded but state stays `New`; the parser rejects a 65 MiB artifact before buffering it; fuzz target `findings_artifact` runs 10 minutes without panic.
- **Provides.** Observation data for A5 and W4; cascade outputs for O3. **Consumes.** C1, C3, C4, C5, I1, scanner fixture corpus (S1).
- **Blockers.** I1, C4, C5, S1.
- **Local verification.** `cargo test -p vf-ingest --features integration ingest::` with compose and RustFS; `cargo +nightly fuzz run findings_artifact -- -max_total_time=600`.
- **Why separate.** Review boundary (fresh observations and narrow carry-forward; M1 "duplicate events cannot create duplicate observations or consumption"); parallel with I1.

### 1.6 Project `web`

- **Goal.** Deliver the `vf-web` SPA for the first customer loop against the committed API contract: sign in, register and verify a target, run the skeleton pipeline, watch progress live, inspect findings, read guidance, mark false positives and request verification. The builder canvas is later; the validator WASM is wired now so the contract is exercised from the browser.
- **Definition of done.** The flows above work against `just run-local` with the fake runtime; the API client is generated from `openapi/vf-api.json` with no hand-edited types; pack T8 (vitest and Playwright) passes; bundle budgets of §14.3 are enforced in the build; core flows pass automated WCAG 2.2 AA checks (axe) with no serious violations.
- **Lead.** Linda. **Repository.** `vf-web`.

#### W1. SPA scaffold, auth, generated client, mock server

- **Owner / priority.** Linda, P0.
- **Scope.** Vite 8, React 19, TypeScript strict, TanStack Router and Query v5, Tailwind v4 (pinned), shadcn/ui base, Zustand; OIDC Authorization Code + PKCE via `oidc-client-ts` configured for the local dev issuer and later Keycloak; `pnpm gen:api` running openapi-typescript against `openapi/vf-api.json` copied from a pinned `platform` commit (until A1 lands, a hand-written stub spec containing only `/healthz` and the §A3.8 paths as placeholders, replaced on A1 close); typed fetch client with Problem Details handling and `Retry-After` support; MSW handlers generated from the spec for local development without the API; route-level code splitting; CI budget script (`size-limit`) per route chunk; eslint, prettier, vitest, Playwright config.
- **Acceptance criteria.** `pnpm install && pnpm build && pnpm test` passes on a clean machine; login round-trip works against `vf-api dev-issuer`; a 401 triggers silent renew then re-login; `pnpm gen:api` is deterministic and CI fails if the committed client is stale; initial route chunk is under 200 KiB gzipped and contains no xyflow or WASM.
- **Provides.** App shell for W2–W5. **Consumes.** §A3.8, A1's spec when available.
- **Blockers.** None: the endpoint table in §A3.8 fixes the paths and shapes, so scaffolding and mock handlers can start today; the generated client is refreshed when A1 commits the real spec.
- **Local verification.** `pnpm test && pnpm build && pnpm e2e:mock` (Playwright against MSW).
- **Why separate.** Different repository and owner; hard dependency for every web task.

#### W2. Targets and authorization UI

- **Owner / priority.** Linda, P1.
- **Scope.** Targets list and detail, registration form with inline canonicalization feedback from the API's Problem reasons, challenge flow (method choice, DNS record or URL instructions with copy buttons, verify button, result states `Live`/`Grace`/`Expired`/`Revoked`), manual-review request (admin), revoke with confirmation, projects as labels.
- **Acceptance criteria.** Pack T8: each `BasisState` renders its distinct state and next action; a `422 invalid-hostname` shows the server reason; revocation requires confirmation and refreshes state on `204`; flows pass axe with no serious violations; works against MSW and against `just run-local`.
- **Provides.** First real user slice. **Consumes.** W1, A2.
- **Blockers.** W1 (A2 for the live check; MSW suffices to build).
- **Local verification.** `pnpm test` and Playwright against `just run-local` once A2 closes.
- **Why separate.** Parallel deliverable with W3 and W4 within the project; each is a reviewable screen set with its own contract area.

#### W3. Run submission, live progress and usage

- **Owner / priority.** Linda, P1.
- **Scope.** Template picker (skeleton template), target and discovery toggle, estimate call with known units and discovery uncertainty shown next to `GET /v1/usage` remaining; submit with a client-generated `Idempotency-Key` persisted until the response; run page with the `RunSnapshot`, per-node counters, unit list with states and skip reasons, log pane with the `sampled` marker, cancel; SSE client with `Last-Event-ID`, reconnect with backoff, `resync` handling; TanStack Query invalidation from events rather than polling; usage page with the ledger.
- **Acceptance criteria.** Pack T8: a simulated disconnect and reconnect shows no duplicate or missing unit states; a `resync` event replaces the snapshot; skipped units show their reason; cancel is disabled on terminal runs; the estimate warns when known units exceed remaining allowance; SSE handling is covered by vitest with a fake event source.
- **Provides.** Live loop for demos and M1 evidence (`execution/browser-disconnect` from the client side). **Consumes.** W1, A3, A4.
- **Blockers.** W1.
- **Local verification.** Playwright against `just run-local` with the fake runtime; vitest for the SSE client.
- **Why separate.** Parallel deliverable; review boundary (replay and resync behaviour).

#### W4. Findings table, remediation panel, triage, verification action

- **Owner / priority.** Linda, P1.
- **Scope.** Findings table with TanStack Table + Virtual over keyset pages, filters (run, target, state, severity), sort, 10k-row fixture; detail drawer with observation history and verification outcomes; remediation panel showing tier badge, version, steps, refs, effort, `unavailable` state; state actions with illegal-transition handling; false-positive dialog showing the match scope that will carry forward; verify button showing the derived plan's unit count and allowance impact before confirming.
- **Acceptance criteria.** Pack T8: filter and sort on a 10 000-row MSW fixture stays under 100 ms p75 in the Playwright benchmark; the tier badge renders all three tiers and `unavailable`; a `409 illegal-transition` is surfaced without state corruption; the verify confirmation shows units and remaining allowance; findings flows pass axe.
- **Provides.** Findings slice for M2 pilot. **Consumes.** W1, A5.
- **Blockers.** W1.
- **Local verification.** `pnpm test`, Playwright benchmark against MSW, then against `just run-local` once A5 closes.
- **Why separate.** Parallel deliverable; performance budget review boundary.

#### W5. `vf-graph` WASM packaging and browser parity harness

- **Owner / priority.** Linda, P2.
- **Scope.** Script that builds `vf-graph` with wasm-pack from a pinned `platform` commit into `packages/vf-graph-wasm/` (committed artifact with its source commit and sha256 in a manifest), lazy-loaded module wrapper exposing `validate(graph, entitlements)`, vitest parity test running the shared golden corpus (copied from `platform/conformance/graph` with its commit recorded) and asserting equality with the golden results, size budget check for the `.wasm` chunk, a minimal "validate this JSON" developer page behind a route not on the activation path.
- **Acceptance criteria.** Parity test passes for the whole corpus; the WASM chunk is excluded from the initial bundle; rebuilding from the pinned commit reproduces the same sha256.
- **Provides.** Browser-side validator for the later builder. **Consumes.** W1, G1.
- **Blockers.** W1, G1.
- **Local verification.** `pnpm wasm:build && pnpm test parity`.
- **Why separate.** Cross-repository artifact with a review boundary (`graph/native-wasm-parity`); fills capacity after W2–W4.

### 1.7 Project `scanners`

- **Goal.** Provide the real scanner material that the fake runtime, ingest and the later images depend on: a fixture corpus of genuine upstream tool and SCB parser outputs, and locally buildable scanner image definitions for the skeleton pipeline. Image publication, Harbor and arm64 signing are later.
- **Definition of done.** The corpus covers success, zero findings, blocked, DNS error, timeout, malformed and retry cases for subfinder, dnsx, httpx and nuclei; ingest and the fake runtime consume it; the four images build locally with pinned tool versions and auto-update disabled.
- **Lead.** Kelly. **Repository.** `scanners` for image definitions; corpus committed to `platform/crates/vf-testkit/fixtures/scanner-output/` (neutral lane) so `platform` tests can read it without a cross-repo dependency.

#### S1. Scanner output and SCB parser fixture corpus

- **Owner / priority.** Kelly, P0.
- **Scope.** Run the upstream images of subfinder, dnsx, httpx and nuclei locally (docker, pinned tags) against a controlled local fixture target (a compose `fixture-target` service: nginx with a known vulnerable-looking configuration and a hosts entry; no internet target) to capture raw outputs; run the secureCodeBox v5.9.0 parsers (from the upstream repository at the pinned tag, via node) over them to capture `findings.json` envelopes; hand-build the error cases (empty, malformed JSON, truncated, oversize marker, non-zero exit, timeout); write `manifest.json` examples per §A3.6; document provenance (tool version, image digest, command) in a README per fixture; commit the SCB findings JSON schema used by I2.
- **Acceptance criteria.** Every fixture has provenance; the envelope files validate against the committed schema; `nuclei` fixtures include at least one finding with CVE and CWE ids and one with template-provided remediation text; a `dnsx` fixture includes an AAAA record (left untested per O4); fixtures are loadable through `vf-testkit::fixtures::load`.
- **Provides.** Inputs for O1, I2, C5, T6. **Consumes.** F2 (fixture tree and loader).
- **Blockers.** None for the capture work (docker and the upstream images are enough); the commit into `vf-testkit/fixtures/` waits for F2's tree, which is expected to close first.
- **Local verification.** `cargo test -p vf-testkit fixtures::` (schema validation of every fixture).
- **Why separate.** Different owner and skill set (running real tools) from the Rust ingest; parallel deliverable needed by three tasks.

#### S2. Local scanner image definitions for the skeleton pipeline

- **Owner / priority.** Kelly, P2 (Jorge reviews the build tooling).
- **Scope.** In `scanners`: Dockerfiles for subfinder, dnsx, httpx, nuclei on pinned upstream versions, with `vf-scanner-adapter` copied from a pinned `platform` release build (local `cargo build --release` path for now), nuclei template pack vendored at a pinned digest and auto-update disabled, non-root user, read-only root filesystem compatible layout, SCB `ScanType` and parser definitions for v5.9.0 as YAML (not applied anywhere yet), `docker buildx bake` for `linux/amd64` and `linux/arm64`, a `conformance/` script that runs each image against the compose fixture target and compares the manifest with S1.
- **Acceptance criteria.** `docker buildx bake --load` succeeds for the host platform; each image runs `vf-scanner-adapter exec` against the fixture target and produces a manifest matching S1's golden within documented volatile fields; no image contacts the internet during a run (test with `--network` restricted to the fixture network); versions and digests are listed in a `VERSIONS.md`.
- **Provides.** Images for the M2 conformance work. **Consumes.** G5, S1.
- **Blockers.** G5, S1.
- **Local verification.** `make conformance` in `scanners` with compose up.
- **Why separate.** Different repository; packaging review boundary (§21.3 pinning and no auto-update).
- **Needs real infrastructure later.** Harbor push, signing, SCB install on a cluster.

### 1.8 Projects with no tasks yet

- **`reporting`** (`vf-report`, lead Fred with Linda for templates): starts after the observation and verification contracts of this matrix freeze (plan M4 "after stable observation/verification contracts"); gated by O8.
- **`infra`** (lead Jorge): deployment tasks are explicitly deferred by the owner; the artifacts this matrix produces for it are `print-crds`, `vf-admission print-config` and the compose definitions.
- **`ai-gateway`** (`vf-aigw`): Phase 5; nothing before GA.

Listing them here lets MasterChief create the project shells now if desired, with no tasks.

### 1.9 Project `test-packs`

- **Goal.** Provide the independent test suites required by gate 4 for every contract in this matrix, authored only by Halsey, living in the test lane of `platform` (`crates/*/tests`, `tests/`, `fuzz/`, `conformance/`) and in `vf-web`'s test directories, so that Test Runner has a named, versioned suite for each candidate.
- **Definition of done.** Every pack below exists, compiles against the published API of its crates, covers the acceptance criteria of the tasks it names and the TDD §25 identifiers it lists, and has been run by Test Runner against at least one candidate. Halsey never runs tests; compile errors reported by Test Runner route back to her as test defects, assertion failures route to the coder, and disputed expectations route to Cortana with requirement evidence.
- **Lead.** Halsey. **Repository.** `platform` (and `vf-web` for T8).

Each pack starts when the architecture contract it tests is final (it is, as of this document) and the coder has commented "API skeleton committed" on the named coding task; MasterChief may additionally wire a blocker on that coding task if a stricter sequence is preferred. Packs are ordered by the first wave they serve.

| Pack | Covers (coding tasks) | Contents and TDD §25 identifiers | Trigger | Why separate |
|---|---|---|---|---|
| **T1** core state and accounting | C1, C4 | `crates/vf-core/tests`: enum round-trips, transition tables, role matrix exhaustiveness (`auth/oidc-jwt-roles` policy half), `pipeline_outcome` cases; `crates/vf-meter/tests` (integration): `usage/unit-definition`, `usage/failure-retry-settlement`, `usage/concurrent-reservations`, trybuild double-settle compile-fail | C1 skeleton | First wave; distinct crates and invariants |
| **T2** scope and canonicalization | C2 | `crates/vf-core/tests/scope_*.rs` corpus and proptest; `fuzz/fuzz_targets/canonical_host.rs`, `scope_match.rs`; `authz/configured-scope`, `authz/psl-exact-root` | C2 skeleton | Security control with its own corpus |
| **T3** graph, parity, translator | G1, G2 | `conformance/graph/*.json` + `*.golden.json`, `crates/vf-graph/tests` native and `wasm-bindgen-test` runs over the same corpus (`graph/native-wasm-parity`), schema drift; `crates/vf-translator/tests` argv goldens, candidate keys, cascade scope refusal, `Scan`/`CascadingRule` snapshots (`graph/validator-translator-conformance`); `fuzz/graph_dsl_parse.rs`, `argv_builder.rs` | G1 skeleton | Shared corpus consumed by two repositories |
| **T4** data isolation and adapters | C3, F2, F3 | `crates/vf-db/tests` (integration): RLS context matrix, PgBouncer reuse, revoke checks, outbox lease, migration resume, schema snapshot, trybuild `TenantTx`-only (`isolation/all-stores` Postgres part, `isolation/background-queries`); adapter conformance over all `ArtifactStore` and `WakeBus` implementations incl. prefix refusal and production refusal; `crates/vf-testkit/tests` bootstrap | C3 skeleton | Isolation suite is its own release gate (§23.4) |
| **T5** authorization, barrier, admission | G3, G4 | `crates/vf-authz/tests` (integration): challenge matrix, temporal rules, `decide()` table (`authz/approval-tracks` Track A, `audit/authorization-history`); `crates/vf-admission/tests`: `AdmissionReview` fixture corpus, fail-closed, dry-run (`authz/start-barrier-all-paths` local half, `execution/gates-before-start` local half); `fuzz/notification_verifier.rs` shared with T6 | G3 skeleton | Barrier evidence for M1 exit |
| **T6** execution loop and ingest | A3, A5, O1, O2, O3, I1, I2, C5, G5 | `crates/vf-operator/tests` with fake runtime scripts (`execution/scan-identity`, `execution/late-cascade-barrier`, adoption, cancellation, revocation); `crates/vf-ingest/tests` (verification matrix, `findings/new-per-scan`, `findings/replayed-artifact`, `findings/fp-only-persistence`, `findings/new-after-fix`, settlement idempotency, sweep); `crates/vf-remediation/tests` (`remediation/version-tier-no-ai`, `verify/applicable-check-evidence`); `tests/walking_skeleton.rs` end to end on compose with `just run-local` components in-process; `fuzz/findings_artifact.rs` | O1 and I1 skeletons | Cross-crate loop needs one coherent suite |
| **T7** API contract | A1, A2, A3, A4 | `crates/vf-api/tests`: route×role matrix, suspended token (`auth/suspended-token`), Problem shapes, OpenAPI drift, idempotency, SSE replay/resync/backpressure, `execution/browser-disconnect` (server side) | A1 skeleton | Contract consumed by another repository |
| **T8** web | W1–W5 | `vf-web`: vitest unit tests for the client, SSE handling and stores; Playwright flows against MSW and `just run-local`; axe checks on core flows; parity test for W5; `ui/performance-accessibility` local benchmark | W1 scaffold | Different repository and toolchain |
| **T9** workspace rules | F1 | `tests/workspace_graph.rs`: `cargo metadata` assertions for §A1.4 (no I/O in `vf-core`/`vf-graph`, no binary→binary deps, AI→dispatcher ban placeholder), pin equality with §A5, `forbid(unsafe_code)` present in every crate root, no `#[cfg(test)]` in `src` | F1 skeleton | Guards the scaffold decisions mechanically |

---

## Part 2. Cross-project dependencies

Edges where a task in one project must be `done` before a task in another can close. Inside-project blockers are on each task above.

| From (blocking task, project) | To (blocked task, project) | What crosses | Can the blocked task start early? |
|---|---|---|---|
| F1 scaffold (`platform-foundation`) | every `platform` task in `core-libraries`, `graph-and-execution-contracts`, `api`, `operator-and-ingest`, and S1's commit | the workspace, pins, crate skeletons | No; F1 is small and first |
| F2 harness (`platform-foundation`) | C3, C4, G3, G4, A1–A5, O2, O3, I1, I2 integration runs; S1 fixture commit | compose services, `vf-testkit` | Yes for unit-level work; integration tests wait |
| F3 adapters (`platform-foundation`) | A1 (rate limits), A4 (SSE wake), O1 (artifact writes), I1 (artifact fetch) | `ArtifactStore`, `WakeBus` | Yes with the in-process and memory adapters once F3's traits exist |
| C1 core (`core-libraries`) | G2, G3, A1, O1, I1 | ids, states, events, ports, Problem | No |
| C2 scope (`core-libraries`, Kelly) | G2, G3, C4 (`admit_target`) | `CanonicalHost`, scope checks | No |
| C3 `vf-db` (`core-libraries`) | G3, G4, A1, O2, I1 | `TenantTx`, repositories, outbox | Skeleton only |
| C4 `vf-meter` (`core-libraries`) | A3, O3, I2 | reserve/settle/release | No |
| C5 `vf-remediation` (`core-libraries`) | A5, I2 | guidance and plan contracts | Yes with `unavailable` stubs |
| G1 `vf-graph` (`graph-and-execution-contracts`) | A3 (validate), C5 (plan graph), W5 (WASM) | validator and types | W5 no; A3 skeleton yes |
| G2 `vf-translator` (`graph-and-execution-contracts`) | A3, O1, O3, G5 | candidates, `ScanSpec`, argv | O1 fake can start on stub types |
| G3 `vf-authz` (`graph-and-execution-contracts`) | A2, A3, O3, G4 | basis resolution, `decide()` | No |
| G5 adapter and hook (`graph-and-execution-contracts`) | S2, O1 (hook invocation in the fake), I1 (test vectors) | binaries and signing contract | O1 and I1 can use an in-process signer first |
| A1 API shell (`api`) | W1 client regeneration, all other `api` tasks | `openapi/vf-api.json` | W1 starts on the stub spec |
| A2 targets (`api`) | W2 live verification | endpoints | W2 builds on MSW |
| A3 dispatcher (`api`) | O3 (submissions to reconcile), W3, A5 | runs, usage | O3 can be driven by `vf-testkit` seeded runs |
| A4 SSE (`api`) | W3 | event stream | W3 builds on a fake event source |
| A5 findings (`api`) | W4 | findings endpoints | W4 builds on MSW |
| O1 runtime (`operator-and-ingest`) | A3 (`ExecutionBackend`), O3 | runtime port implementations | No for O3 |
| I1 notification (`operator-and-ingest`) | I2, O1 fake hook target | ingest entry | I2 skeleton yes |
| I2 ingest (`operator-and-ingest`) | A5 data, O3 cascade outputs, walking skeleton | observations | A5 can seed through `vf-testkit` |
| S1 corpus (`scanners`) | I2, O1 scripts, C5 fixtures, T6 | fixture files | O1 can start on synthetic outputs and swap |
| G1 corpus (`graph-and-execution-contracts`, via T3) | W5 parity (`web`) | golden files | No |
| Every coding task | its pack in `test-packs` | published API skeleton | Packs start on the skeleton comment |
| Every pack in `test-packs` | push approval of its coding tasks | passing run by Test Runner | Not applicable |

### First wave (no blockers or only F1)

Day one, in parallel: **F1** (Jorge), **W1** (Linda), **S1** capture work (Kelly), and, as soon as F1 lands, **C1** (Fred), **C2** (Kelly), **G1** (Kelly after C2 or in parallel; both are hers and disjoint), **F2** and **F3** (Jorge), **T9** and **T1/T2/T3** drafting (Halsey). This gives each of the four developers and Halsey independent work from the first day without any infrastructure.

### Second wave

**C3** (Fred) and **G3** (Kelly) as soon as C1/C2 close; **O1** (Jorge) on F3 and G2 stubs; **A1** (Fred) on C1/C3; **W2/W3/W4** (Linda) on W1 and MSW. Third wave is the loop: **C4**, **G2**, **A3**, **O3**, **I1**, **I2**, closing with the walking-skeleton test in T6.

### Task count

| Project | Coding tasks | Test packs |
|---|---|---|
| platform-foundation | 3 | T9, part of T4 |
| core-libraries | 5 | T1, T2, part of T4 |
| graph-and-execution-contracts | 5 | T3, T5 |
| api | 5 | T7 |
| operator-and-ingest | 5 | T6 |
| web | 5 | T8 |
| scanners | 2 | (fixtures validated by T4/T6) |
| **Total** | **30** | **9** |

Nothing in the 30 depends on Kubernetes, Aether, Harbor, hosted CI or any cloud service. Items marked "needs real infrastructure later" are the deferred deployment set the owner asked to keep separate.
