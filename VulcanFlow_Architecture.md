<!--
Source: Paperclip issue VFL-8, document `architecture` (revision db278028-856c-492d-8f68-bc388aff4ecd), authored by Cortana (architect), 2026-10-05.
Links of the form /VFL/issues/... point to Paperclip issues and their documents, not to paths in this repository.
Companion file in this repository: `VulcanFlow_Task_Matrix.md` (task matrix).
Body below is identical to the issue document revision named above.
-->

# VulcanFlow architecture (Option B workspace, local-first)

Status: architecture decision record for [VFL-8](/VFL/issues/VFL-8), derived from the accepted [development plan](/VFL/issues/VFL-1#document-plan) and [TDD v2.3](https://github.com/vulcanflow/docs/blob/main/VulcanFlow_Technical_Design_Document_v2.2.md) (file still named v2.2). Section references (§) are to that TDD. Quality gates are the seven in [team setup](/VFL/issues/VFL-3#document-team-setup). Repository layout follows the executed Option B of the [repo audit](/VFL/issues/VFL-6#document-repo-audit).

Governing constraint from the owner: no Kubernetes or cloud infrastructure exists yet. Every component in this document is designed to build, run and be tested on a developer machine with local stand-ins. The infrastructure adapter boundary (§A6) is how the stand-ins and the real Aether services are swapped without rewriting application code.

Evidence basis checked on 2026-10-05: `platform` main contains only the lane gate (`.github/workflows/lane-gate.yml`, `ci/lane-gate.sh`, `ci/lane-gate-test.sh`, `.gitattributes`); `infra` main returns no tree; `docs` holds the plan, the TDD and a stale README. No application code exists anywhere. Nothing from the closed `platform` pull requests is reused.

---

## A1. System context and component inventory

### A1.1 Planes and trust boundaries (§2.2, §3)

Two planes. The control plane is a stateful Rust web application; the execution plane is a hostile-workload sandbox. They communicate only through three narrow channels, and each channel is behind a port (trait) so the local harness can stand in for it:

| Channel | Direction | Real implementation (later) | Local stand-in (now) | Port |
|---|---|---|---|---|
| Scan work in | control → execution | secureCodeBox `Scan`/`CascadingRule` custom resources via kube-rs | in-process `FakeScanRuntime` driven by fixtures | `ScanRuntime` |
| Findings out | execution → control | Aether S3-compatible object storage (Ceph RGW / RustFS) | RustFS container (default) or MinIO; memory/filesystem for unit tests | `ArtifactStore` |
| Completion notification | execution → control | `vf-hook-notify` HMAC-signed POST to `vf-ingest` | same binary, invoked by `FakeScanRuntime` or by a fixture script | HTTP contract §A3.6 |

Tenant boundaries are the three layers of §3.2: namespace (execution), schema plus Row-Level Security (data), per-tenant object prefix (artifacts). On a developer machine the namespace layer is represented by the fake runtime's per-tenant partition; the data and artifact layers are the real mechanisms and are tested for real (§A6.3).

### A1.2 Repositories under Option B

| Repository | Visibility | Contents | Build unit |
|---|---|---|---|
| `platform` | public | One Cargo workspace: all VulcanFlow Rust crates below, local dev harness, migrations, conformance fixtures | `cargo` workspace, one `Cargo.lock`, one `rust-toolchain.toml` |
| `vf-web` | private | TypeScript SPA (§14), consumes the committed OpenAPI spec and the `vf-graph` WASM artifact | Vite/pnpm |
| `scanners` | private | Dockerfiles for the upstream scanner images (subfinder, dnsx, httpx, tlsx, nuclei, nmap, masscan, optional Amass), SCB parser definitions, image-level conformance | docker buildx; consumes adapter binaries released from `platform` |
| `infra` | public | GitOps manifests, Kustomize overlays, CRD YAML published from `platform` (later) | none now; no task in this document targets it |
| `docs` | public | Plan, TDD, this architecture mirrored later | none |
| `vf-aigw` | private | LiteLLM gateway deployment and policy (Phase 5) | none now |

### A1.3 `platform` crate inventory and the two settled mapping points

Layout is fixed by the lane gate already on `platform` main: production source in `crates/<crate>/src`, tests in `crates/<crate>/tests` (written against the public API; `#[cfg(test)]` modules in `src` are refused by the gate), fuzz targets in `fuzz/`, cross-crate integration and end-to-end tests in `tests/`, conformance fixtures reachable from `conformance/` (test lane) or `crates/vf-testkit/fixtures/` (neutral lane, so coders may add scanner output samples without crossing lanes).

| Crate | Kind | Responsibility (TDD) | Depends on | Owner |
|---|---|---|---|---|
| `vf-core` | lib, no I/O | Identity newtypes (§6.2), `CanonicalHost` and scope matching (§5.3), all state enums and transition functions (§6.3, §8.2, §15.1, §17.3), role×action policy (§4.2), Problem type catalogue (§13.3), port traits (§A6.1), event shapes (§A3.3) | serde, idna, url, psl, thiserror, uuid, chrono | Fred (crate); Kelly owns module `scope` |
| `vf-db` | lib | Migrations for control schema `vf` and tenant schema template, `TenantTx`/`ControlTx` (§3.5), repositories for every table in §A4, transactional outbox, infrastructure adapters implementing the `ArtifactStore` and `WakeBus` ports | vf-core, sqlx, object_store, redis | Fred (crate); Jorge owns module `adapters` |
| `vf-graph` | lib, native + wasm32 | Graph DSL types, node config structs, type lattice, validation rules, JSON schema generation, `wasm-bindgen` export (§7.1–7.3, Appendix A) | serde, schemars, wasm-bindgen (target-gated) | Kelly |
| `vf-translator` | lib | Validated graph + run scope + reservations → `WorkUnitPlan`, SCB `Scan`/`CascadingRule` typed objects, per-node argv builders (§7.4, §22.3) | vf-core, vf-graph, generated SCB CRD types | Kelly |
| `vf-authz` | lib | Track A challenge issue/verify (DNS-TXT, HTTP file, manual), authorization basis and lifecycle, live basis resolver with 90-day/14-day rules, the start-barrier decision function (§5.2, §5.6, §5.7) | vf-core, vf-db, hickory-resolver, reqwest | Kelly |
| `vf-meter` | lib now, bin in Phase 4 | Atomic reserve/settle/release over `TenantTx`, billing-period usage, append-only ledger, target admission (§17); Lago/Stripe sync is the Phase 4 binary | vf-core, vf-db | Fred |
| `vf-remediation` | lib now, bin in M3 | Three-tier guidance resolution and version pinning, verification plan derivation, verification outcome interpretation (§15); content publishing and feed mirror job become `[[bin]]` targets in M3 | vf-core, vf-db, vf-graph | Fred |
| `vf-api` | bin (+lib for tests) | REST, OpenAPI 3.1, SSE, auth middleware, role layer, **dispatcher module** (validate, estimate, reserve, outbox), outbox worker, tenant-level event stream (§4, §8.1, §9, §13) | all libs above | Fred |
| `vf-operator` | bin + lib | `ScanFlow` reconcile: live approval, per-candidate reservation through the barrier, Scan creation via `ScanRuntime`, fingerprint binding, cascade candidate registration, completion barrier, cancellation, deterministic adoption; `Tenant` provisioning (§3.3, §7.4, §8); holds both `ScanRuntime` implementations | vf-core, vf-db, vf-meter, vf-authz, vf-translator, kube-rs (feature-gated) | Jorge |
| `vf-admission` | bin + lib | Validating admission handler for Scan, Job and Pod (`AdmissionReview` v1), fail-closed, backed by the same barrier decision as the operator (§5.7) | vf-core, vf-db, vf-authz, k8s-openapi, axum, rustls | Kelly |
| `vf-ingest` | bin + lib | Signed notification receiver, artifact fetch and bounded streaming parse, fresh observations, false-positive matcher application, enrichment, reservation settlement, 60-second reconciliation sweep (§8.4, §10) | vf-core, vf-db, vf-meter, vf-remediation | Jorge |
| `vf-hook-notify` | bin | SCB completion hook image: builds and signs the notification, posts to ingest (§8.4) | vf-core, reqwest, hmac, sha2 | Kelly |
| `vf-scanner-adapter` | bin | Runs inside the scanner images: materializes validated inputs, builds argv from typed node config, executes the upstream tool via `Command::args`, classifies the outcome, writes `manifest.json` (§7.4, §8.2, §22.3) | vf-core, vf-translator (argv builders) | Kelly |
| `vf-report` | bin + lib | Report assembly, immutable input snapshot, templates with context-restricted escaping, renderer handoff, delivery (§16). **No task in this document**; contracts freeze after M1 | vf-core, vf-db, askama, ammonia | Fred (crate), Linda (templates) |
| `vf-abuse` | bin | Phase 4. Crate directory created empty by the scaffold so the inventory is complete; no task now | — | Kelly |
| `vf-testkit` | lib, dev-dependency only | Compose/testcontainers Postgres bootstrap with migrated template schema, fixture loaders, deterministic `Clock`/`IdGen`, dev token minting, fixture corpus under `fixtures/` | vf-core, vf-db, testcontainers | Jorge |

**Mapping point 1, `vf-remediation` workspace inclusion.** §2.3 lists it as a component; §2.5.2's workspace list omits it. Decision: **it is in the workspace as a library crate**. Its three responsibilities are all pure logic over data the API and ingest already hold: guidance resolution happens at ingest (§15.3), plan derivation is an ordinary pipeline-run submission with `run_kind = verification` (§15.4), and outcome interpretation runs when that run settles. A separate network service for read-mostly logic would add a hop on the ingest path for no isolation benefit. §2.5.2's omission is read as "no separate binary at GA", which this decision keeps; the M3 content publishing and feed mirror job are `[[bin]]` targets of the same crate.

**Mapping point 2, home of the Rust scanner adapters.** Decision: **`vf-scanner-adapter` and `vf-hook-notify` are `platform` crates; the `scanners` repository packages them.** Argument construction is the control that makes the authorization gate real (§22.3): it must share `CanonicalHost`, the node config structs and the argv builders with `vf-graph`/`vf-translator`, under one lockfile and one review. The `scanners` repo holds Dockerfiles, parser definitions and image conformance, and copies a released adapter binary built from a pinned `platform` commit into each image. The parser language stays the upstream JavaScript SDK until §27 item 20 is proven otherwise.

**Dispatcher** stays a module of `vf-api` (`vf-api::dispatcher`), per plan item 3. It owns the single transaction that creates the run, registers root work units, reserves allowance and writes outbox intent (§8.1 step 3–4).

### A1.4 Crate dependency graph

```
vf-core ◄── vf-graph (no vf-core dependency: stays I/O-free and wasm-clean; shares only serde types)
   ▲
   ├── vf-db ◄── vf-meter ◄── vf-api, vf-operator, vf-ingest
   │       ◄── vf-authz ◄── vf-api, vf-operator, vf-admission
   │       ◄── vf-remediation ◄── vf-api, vf-ingest
   ├── vf-translator (vf-core + vf-graph) ◄── vf-api (estimate), vf-operator, vf-scanner-adapter
   ├── vf-hook-notify
   └── vf-testkit (dev-dependency everywhere)
```

Rules enforced by `deny.toml` bans and a workspace test in `tests/workspace_graph.rs` (Halsey): `vf-core` and `vf-graph` depend on no I/O crate (no tokio, sqlx, reqwest, kube); `vf-graph` does not depend on `vf-core` (it exports its own small types that `vf-core` re-exports) so the WASM build carries nothing it does not need; no binary crate depends on another binary crate except through its `lib` target; the future AI crate may not depend on `vf-api` (§18.6, recorded now, enforced when the crate exists).

---

## A2. Ownership

| Developer | Owns (crate or module) | Review focus |
|---|---|---|
| Fred | `vf-core` (except `scope`), `vf-db` (except `adapters`), `vf-meter`, `vf-remediation`, `vf-api`, `vf-report` | Accounting invariants, data model, API contract, findings domain |
| Kelly | `vf-core::scope`, `vf-graph`, `vf-translator`, `vf-authz`, `vf-admission`, `vf-scanner-adapter`, `vf-hook-notify`, `vf-abuse`, `scanners` adapters and parsers | Scope, barriers, argv construction, signatures, untrusted input |
| Jorge | workspace scaffold, local dev harness, `vf-db::adapters`, `vf-operator`, `vf-ingest`, `vf-testkit`, `scanners` image builds, `infra` (later) | Durable queues, reconciliation, recovery, swappable infrastructure |
| Linda | `vf-web`, `vf-graph` WASM packaging into the SPA, `vf-report` templates (later) | Contract consumption, performance budgets, accessibility |
| Halsey | `crates/*/tests`, `tests/`, `fuzz/`, `conformance/`, `vf-web` test suites | Tests only; never runs them |
| Cortana | this document, interface contracts, test-expectation rulings, push approval | — |

A crate owner is the default assignee for defects in it and the reviewer of record for contract changes to it. Changing a published contract (§A3) requires an update to this document first.

---

## A3. Interface contracts

Contracts are given as Rust signatures (the compile-checked form Halsey writes tests against) and as wire shapes where a boundary crosses a process. Names are binding; field order and documentation comments are not.

### A3.1 Identity and value types (`vf-core`)

```rust
// vf-core::ids — each is a newtype over Uuid (v7, time-ordered), Display/FromStr/serde as the plain UUID string
pub struct TenantId; pub struct UserId; pub struct TargetId; pub struct ProjectId;
pub struct AuthorizationBasisId; pub struct PipelineRunId; pub struct WorkUnitId;
pub struct ScanId; pub struct FindingId; pub struct FpDecisionId; pub struct VerificationRunId;
pub struct ReportId; pub struct ScheduleId; pub struct OutboxId;
pub struct ScanFingerprint(String);     // SCB identifier, stored unchanged; never derived (§6.2, §10.2)
pub struct CandidateKey(String);        // deterministic per (run, node, operation scope); dedups retries (§6.3)
pub struct RequestKey(String);          // client idempotency key; bound to request_hash (§8.1)

// vf-core::scope — Kelly
pub struct CanonicalHost { /* private */ }
impl CanonicalHost {
    pub fn parse(input: &str) -> Result<CanonicalHost, HostError>; // lowercases, strips trailing dot, IDNA (UTS-46 nontransitional) to A-label, rejects public-suffix roots, IPs, ports, paths, empty labels
    pub fn as_str(&self) -> &str;                                  // A-label form
    pub fn registrable_domain(&self) -> Option<CanonicalHost>;     // via PSL incl. private suffixes
    pub fn is_descendant_of(&self, root: &CanonicalHost) -> bool;  // dot-boundary only; never label-prefix
}
pub enum ScopeType { DomainTree, ExactHost }                        // §5.6 scope_type
pub struct ApprovedScope { pub scope_type: ScopeType, pub root: CanonicalHost }
pub struct RunScope { pub target: CanonicalHost, pub include_subdomains: bool }
pub enum ScopeVerdict { InScope, OutsideApproval, DiscoveryNotSelected, PublicSuffix }
pub fn check_run_scope(approval: &ApprovedScope, run: &RunScope) -> ScopeVerdict;
pub fn check_candidate(approval: &ApprovedScope, run: &RunScope, candidate: &CanonicalHost) -> ScopeVerdict;
pub struct Ipv4Destination(std::net::Ipv4Addr);                     // constructor rejects private, loopback, link-local, multicast, reserved, broadcast
```

Scope rules fixed by §5.3 and tested by `authz/configured-scope`, `authz/psl-exact-root`: `ExactHost` approval permits only the identical host; `DomainTree` approval permits the root and dot-boundary descendants, but a run includes descendants only when `include_subdomains` is true; `check_candidate` must pass both the approval and the run scope. `registrable_domain` is informational and must never be used to broaden an `ExactHost` approval.

### A3.2 State machines (`vf-core::state`)

Every text-typed status column in §A4 maps to one enum with `as_str`/`from_str` that errors on unknown values. Transitions are total functions; a transition absent from the TDD is unrepresentable.

```rust
pub enum PipelineState { Validating, Refused, Dispatching, Running, Completed, PartiallyCompleted, TargetFailed, PlatformFailed, Cancelled, TimedOut }
pub enum WorkUnitStatus { Registered, Reserved, Skipped, Admitted, Running, Terminal }
pub enum OutcomeClass { Success, Target, Platform, Tool, Scope, Limit, Cancelled }       // §6.3 outcome_class; §8.2 table
pub enum ObservationState { New, Acknowledged, FixPending, Verifying, Fixed, FalsePositive, AcceptedRisk }
pub enum VerificationOutcome { NotDetected, StillPresent, Inconclusive }
pub enum ReservationState { Reserved, Consumed, Released }
pub enum Role { Admin, Member }  pub enum Action { /* one variant per cell of §4.2, e.g. ScanDispatch, ScanCancelOwn, FindingTriage, TemplateCrud, ReportGenerate, BillingRead, MemberManage, ... */ }
pub fn allowed(role: Role, action: Action) -> bool;                 // single exhaustive match, no wildcard arm
pub fn observation_transition(from: ObservationState, event: ObservationEvent) -> Result<ObservationState, IllegalTransition>;
pub fn apply_verification(prior: ObservationState, outcome: VerificationOutcome) -> ObservationState; // NotDetected→Fixed, StillPresent→New, Inconclusive→prior (§15.1)
pub fn outcome_consumes(class: OutcomeClass) -> bool;              // true only for Success (§8.2)
pub fn pipeline_outcome(units: &[UnitSummary], discovery_closed: bool, pending_fan_in: usize) -> Option<PipelineState>; // None while not finalizable (§8.2 completion rule)
```

### A3.3 Durable events, outbox and SSE (`vf-core::events`, `vf-db::outbox`, `vf-api::sse`)

Progress events are a view of durable state (§2.4). The pipeline SSE stream is fed from a Postgres table, not from Valkey; Valkey (or the in-process bus) only wakes the fan-out.

```rust
// vf-core::events — serialized with serde, tag = "event"; these are exactly the §9.2 names
pub enum PipelineEvent {
    #[serde(rename="pipeline.state")] PipelineState { pipeline_run_id, state: PipelineState, at },
    #[serde(rename="scan.state")]     ScanState { pipeline_run_id, work_unit_id, scan_fingerprint: Option<ScanFingerprint>, state: WorkUnitStatus, outcome_class: Option<OutcomeClass> },
    #[serde(rename="node.counter")]   NodeCounter { pipeline_run_id, node_id, registered: u32, completed: u32, skipped: u32 },
    #[serde(rename="log.line")]       LogLine { scan_fingerprint, node_id, stream, line, ts, sampled: bool, dropped: u32 },
    #[serde(rename="finding.new")]    FindingNew { finding_id, scan_fingerprint, state: ObservationState, severity },
    #[serde(rename="finding.state")]  FindingState { finding_id, state: ObservationState, decision_id: Option<FpDecisionId> },
    #[serde(rename="verification.completed")] VerificationCompleted { finding_id, verification_run_id, outcome: VerificationOutcome },
}
pub enum TenantEvent {
    #[serde(rename="allowance.updated")] AllowanceUpdated { period_id, consumed: u64, reserved: u64, remaining: u64 },
    #[serde(rename="report.state")]      ReportState { report_id, state },
}
```

Storage and delivery contract:

- Table `pipeline_events(tenant_id, pipeline_run_id, seq bigint, event jsonb, created_at)` with `UNIQUE (pipeline_run_id, seq)`; `seq` is allocated inside the same transaction that commits the state change it describes. Table `tenant_events` likewise keyed by `(tenant_id, seq)`.
- SSE `id:` is `seq`. `Last-Event-ID` replays from the table for events newer than 15 minutes; older cursors receive one `resync` event carrying a full `RunSnapshot` and the current `seq`. Report download URLs are never placed in events (§9.2).
- Log lines above 100 lines/second per node are sampled server-side with `sampled: true` and a `dropped` count; counters coalesce to at most 4 updates/second per node (§9.3).
- The transactional outbox is `vf.outbox(id, tenant_id, kind, payload jsonb, available_at, attempts, claimed_by, claimed_until, done_at)` with `kind ∈ {submit_run, cancel_run, provision_tenant, ingest_artifact, deliver_report}`. Workers claim with `FOR UPDATE SKIP LOCKED` and a lease; a worker crash leaves a reclaimable row; handlers are idempotent on the payload's deterministic identity. The `WakeBus` only shortens the poll interval.

### A3.4 Graph, estimate and translation (`vf-graph`, `vf-translator`)

```rust
// vf-graph — no I/O, no vf-core dependency, builds for wasm32-unknown-unknown
pub struct FlowGraph { pub version: u32, pub nodes: Vec<Node>, pub edges: Vec<Edge> }   // Appendix A shape, deny_unknown_fields
pub enum NodeType { Subfinder, Amass, Dnsx, Httpx, Tlsx, Nmap, Masscan, Nuclei, Aggregate }
pub enum DataType { Target, Subdomain, Host, Ip, Port, Service, HttpEndpoint, TlsEndpoint, Finding }
pub enum NodeConfig { Subfinder(SubfinderConfig), Dnsx(DnsxConfig), ... }               // one struct per node, deny_unknown_fields, no command/env/template/target override fields
pub struct Entitlements { pub nodes: BTreeSet<NodeType>, pub max_hosts_per_run: u32, pub max_scanner_minutes_per_run: u32 }
pub enum ValidationError { Cycle{..}, TypeMismatch{edge, produced, accepted}, MissingRequiredInput{node}, NoFindingSink, ConfigInvalid{node, reason}, NotEntitled{node, node_type}, CeilingExceeded{..}, UnknownNode{..}, DuplicateNodeId{..}, TooManyNodes }
pub fn validate(graph: &FlowGraph, entitlements: &Entitlements) -> Result<ValidatedGraph, Vec<ValidationError>>;
pub fn json_schema() -> serde_json::Value;                                                  // schemars; published as flow-graph/v1.json
#[cfg(target_arch = "wasm32")] #[wasm_bindgen] pub fn validate_json(graph: &str, entitlements: &str) -> String; // JSON of Result<(), Vec<ValidationError>>; identical semantics to validate()
```

Server-only checks (package entitlement from the tenant record, operation ceilings, allowance) are applied by the dispatcher after `validate`; `Entitlements` is passed in so the shared crate stays I/O-free and the browser can run the same check with the tenant's entitlement snapshot from `GET /v1/usage`.

```rust
// vf-translator
pub struct RunContext { pub tenant: TenantId, pub run: PipelineRunId, pub run_kind: RunKind, pub scope: RunScope, pub approval: ApprovedScope, pub basis: AuthorizationBasisId, pub ceilings: Ceilings, pub versions: PinnedVersions }
pub struct Candidate { pub node_id: String, pub scan_type: NodeType, pub operation: OperationScope, pub candidate_key: CandidateKey } // OperationScope = {host: CanonicalHost, ipv4: Option<Ipv4Destination>, port: Option<u16>, protocol: Option<Proto>, endpoint: Option<Url>}
pub fn root_candidates(graph: &ValidatedGraph, ctx: &RunContext) -> Vec<Candidate>;       // deterministic order and keys; nodes with no graph input only
pub fn estimate(graph: &ValidatedGraph, ctx: &RunContext) -> Estimate;                    // {known_units: u32, discovery_uncertain: bool, per_node: Vec<(node_id, EstimateKind)>}
pub fn derive_candidates(parent: &Candidate, parent_outputs: &[Output], graph: &ValidatedGraph, ctx: &RunContext) -> Vec<Candidate>; // cascade fan-out; every output host passes check_candidate or becomes a Skipped{Scope} record
pub fn scan_spec(c: &Candidate, unit: WorkUnitId, ctx: &RunContext) -> ScanSpec;           // typed SCB Scan from generated CRD types; labels vulcanflow.io/pipeline-run-id, work-unit-id, node-id; annotation vulcanflow.io/target = c.operation.host
pub fn cascading_rules(graph: &ValidatedGraph, ctx: &RunContext) -> Vec<CascadingRuleSpec>; // scoped by run label; spec.scanAnnotations/scanLabels per §7.4
pub fn argv(c: &Candidate, config: &NodeConfig, inputs: &MaterializedInputs) -> Result<Vec<OsString>, ArgvError>; // §22.3: discrete elements, no shell, leading '-' rejected, targets only from CanonicalHost/Ipv4Destination
```

The SCB CRD Rust types are generated with kopium from the CRD YAML of the pinned secureCodeBox release and committed under `crates/vf-translator/src/scb/` with the release tag in the file header. Proposed pin: **secureCodeBox v5.9.0** (latest upstream release, 2026-09-15; the TDD cites 5.8.0). This is open item §27-5 and needs the owner's confirmation, but regeneration from another release is a single command, so no task blocks on it.

### A3.5 Authorization and the start barrier (`vf-authz`)

```rust
pub enum ChallengeMethod { DnsTxt, HttpFile, ManualReview }
pub struct Challenge { pub id, pub tenant, pub target, pub host: CanonicalHost, pub method, pub token: Secret<String>, pub expires_at }   // 256-bit OS CSPRNG token; 7-day expiry; single use
pub async fn issue_challenge(tx: &mut TenantTx, target: TargetId, method: ChallengeMethod, clock: &dyn Clock) -> Result<Challenge>;
pub async fn verify_challenge(tx: &mut TenantTx, id: ChallengeId, probe: &dyn ChallengeProbe, clock: &dyn Clock) -> Result<AuthorizationBasisId, VerifyError>;
pub trait ChallengeProbe { async fn dns_txt(&self, name: &CanonicalHost) -> Result<Vec<String>>; async fn http_wellknown(&self, host: &CanonicalHost) -> Result<ProbeBody>; } // real: hickory-resolver ×3 public + authoritative, reqwest with rustls, ≤1 same-host redirect, 64 KiB body cap, private-destination refusal; local: fixture probe
pub enum BasisState { Live { basis, scope: ApprovedScope, verified_at }, Grace { basis, scope, grace_ends_at }, Expired, Revoked, None }
pub async fn resolve_live_basis(tx: &mut TenantTx, target: TargetId, now: DateTime) -> Result<BasisState>;   // 90-day validity, 14-day grace after failed recheck, revocation immediate (§5.2)

// barrier — one decision function used by the dispatcher (root units), the operator (every Scan), and the admission handler (Scan/Job/Pod)
pub struct BarrierInput<'a> { pub tenant: TenantId, pub unit: &'a WorkUnitRecord, pub basis: &'a BasisState, pub reservation: Option<ReservationState>, pub suspension: SuspensionState, pub observed: &'a ObservedWorkload }
pub enum BarrierVerdict { Admit, Refuse(RefuseReason) }           // RefuseReason ∈ {NoLiveBasis, ScopeMismatch, NotReserved, ReservationSettled, TenantSuspended, UnknownWorkUnit, TargetMismatch{annotated, actual}, ForbiddenSpecField(String), RunFinalized}
pub fn decide(input: &BarrierInput) -> BarrierVerdict;              // pure; fails closed on any Option::None
```

`ObservedWorkload` is what the caller actually sees: for the dispatcher the planned candidate; for the operator the `ScanSpec` it is about to create; for admission the decoded `Scan`, `Job` or `Pod` object (scan type, parameters, env, init containers, volumes, service account, target annotation, materialized target list). `decide` compares the observed target and parameters with the trusted work-unit record, never the other way round (§5.7, §22.2 "annotation differs from actual target").

### A3.6 Scan runtime, notification and ingest identity

```rust
// vf-core::ports (implemented in vf-operator)
pub trait ScanRuntime: Send + Sync {
    async fn create_scan(&self, spec: &ScanSpec, unit: WorkUnitId) -> Result<ScanRef, RuntimeError>;     // idempotent on deterministic name vf-{run}-{unit}; returns existing object on conflict (adoption, §6.3)
    async fn get_scan(&self, r: &ScanRef) -> Result<ScanStatus, RuntimeError>;                           // {fingerprint: Option<ScanFingerprint>, phase, started_at, finished_at, failure: Option<RuntimeFailure>}
    async fn cancel_scan(&self, r: &ScanRef) -> Result<(), RuntimeError>;                                // 10-second grace then delete; records attempt before deletion
    async fn list_scans(&self, run: PipelineRunId) -> Result<Vec<ScanRef>, RuntimeError>;               // for the reconciliation sweep and the completion barrier
    async fn apply_cascading_rules(&self, run: PipelineRunId, rules: &[CascadingRuleSpec]) -> Result<(), RuntimeError>;
}
```

Fingerprint: the Kubernetes `Scan` object's `metadata.uid` (§21.3 proposal), read back from `get_scan` after creation and bound to the work unit by the operator. The `FakeScanRuntime` mints a UUID per created Scan so the identity contract is exercised identically.

Completion notification (`vf-hook-notify` → `POST /internal/v1/notifications` on `vf-ingest`), JSON body:

```jsonc
{ "v": 1, "tenant_id": "...", "pipeline_run_id": "...", "work_unit_id": "...", "node_id": "n2",
  "scan_fingerprint": "<Scan metadata.uid>", "scb_namespace": "vf-tenant-acme", "scb_name": "vf-run-...-unit-...",
  "artifact": { "prefix": "{tenant_id}/{pipeline_run_id}/{scan_fingerprint}/", "findings_key": "findings.json", "manifest_key": "manifest.json", "findings_sha256": "...", "manifest_sha256": "..." },
  "outcome_hint": "success|tool_failed|target_failed|platform_failed",
  "ts": "2026-10-05T19:00:00Z", "nonce": "<128-bit hex>" }
```

Headers: `X-VF-Signature: v1=<hex HMAC-SHA256 over the raw body>`, `X-VF-Key-Id`. Ingest rejects: bad signature, `ts` outside ±5 minutes, nonce seen within 24 hours (`vf.notification_nonces`), unknown work unit, fingerprint not bound to that unit (or binds it if the operator has not yet, after cross-checking `scb_name`), artifact prefix not equal to the trusted prefix for that tenant/run/fingerprint, checksum mismatch after fetch, `findings.json` above the configured size bound (default 64 MiB) or failing the SCB finding envelope schema. Duplicate delivery after a successful ingest returns `200 {"status":"already_ingested"}` and changes nothing.

Observation identity (§6.3): a row in `findings` is unique on `(scan_id, source_finding_id)` where `source_finding_id` is the SCB finding `id` field; replaying the artifact is a no-op; a later scan always creates new rows. The false-positive matcher input is `FpMatchKey { host: CanonicalHost, finding_class, port: Option<u16>, protocol: Option<Proto>, path: Option<String>, parameter: Option<String>, service: Option<String>, match_version: u32 }` serialized canonically into `findings.false_positive_match`; matcher version 1 requires equality on every populated field and the same `finding_class` (no aliases). Enabling persistence of decisions across scans requires the §27-6 field evidence; until then the matcher runs and records what it would apply, and `applied_fp_decision_id` is set only behind the `fp_persistence_enabled` tenant flag (default off).

Manifest (`manifest.json`, written by `vf-scanner-adapter`): `{ v, work_unit_id, node_id, scan_type, image_digest, tool_version, template_pack_digest, argv, inputs_sha256, started_at, finished_at, exit_code, outcome_class, outcome_detail, destinations: [{host, ipv4, port}] }`. `outcome_class` is decided by the adapter from the tool's documented exit and output semantics, never from exit zero alone (§8.2 open item on mixed outcomes: adapter emits `Tool` when it cannot classify).

### A3.7 Metering unit accounting (`vf-meter`)

```rust
pub struct Reservation { /* move-only: work_unit, period, tenant */ }
pub async fn open_period(tx: &mut TenantTx, now) -> Result<PeriodId>;                       // creates billing_period_usage from the entitlement snapshot if absent
pub async fn admit_target(tx: &mut TenantTx, host: &CanonicalHost) -> Result<TargetAdmission>;   // Admitted{target_id} | Skipped{reason: TargetAllowanceExhausted}; canonical duplicates reuse the row
pub async fn reserve(tx: &mut TenantTx, unit: WorkUnitId, key: &str) -> Result<Option<Reservation>>;  // SELECT ... FOR UPDATE on the period row; Some only if period_limit - successful - reserved > 0; ledger row 'reserve'; None ⇒ caller records Skipped{Limit}
pub async fn settle_success(tx: &mut TenantTx, r: Reservation, key: &str) -> Result<()>;   // reserved→consumed, successful_units += 1, ledger 'consume'; errors if state ≠ reserved
pub async fn release(tx: &mut TenantTx, r: Reservation, key: &str, reason: &str) -> Result<()>; // reserved→released, ledger 'release'
pub async fn reservation_for(tx: &mut TenantTx, unit: WorkUnitId) -> Result<Option<(Reservation, ReservationState)>>; // re-hydrate after restart; only Reserved yields a usable Reservation
pub async fn usage(tx: &mut TenantTx, now) -> Result<UsageSnapshot>;                         // {period, limit, consumed, reserved, remaining, targets_used, targets_limit}
```

Invariants (tested by `usage/*`): `remaining = limit − successful − reserved` under concurrent transactions; a unit settles success at most once (`UNIQUE (work_unit_id, event_type)` plus state check); a released unit cannot be consumed; retries hold the same reservation (new `scans` row, same `work_unit_id`); port discovery plus three successful service checks on one target consumes four units, one failed service check reduces that to three (§17.1, §23.5). Target admission counts active registered targets for now; the §27-3 decision may change the counting rule but not the unit.

### A3.8 REST and SSE surface (`vf-api`)

The endpoint table of §13.2 is adopted verbatim as the GA contract; this document fixes the subset in scope for the first tasks and their payloads. Base path `/v1`; every handler is tenant-scoped through `TenantTx`; errors are RFC 9457 Problem Details from one `Problem` enum in `vf-core` with stable `type` URIs under `https://vulcanflow.io/errors/`.

| Endpoint | Request | Response | Notes |
|---|---|---|---|
| `POST /v1/targets` | `{hostname, project_id?}` | `201 Target{id, kind, normalized, project_id, authorization: BasisState, created_at}` | `kind` derived: `domain` if `hostname == registrable_domain`, else `subdomain`; duplicate canonical host returns the existing target with `200` |
| `GET /v1/targets`, `GET /v1/targets/{id}` | — | list / `Target` | keyset pagination `?after=<id>&limit≤200` |
| `POST /v1/targets/{id}/challenges` | `{method: dns-txt|http-file}` | `201 {challenge_id, method, record_name|url, token, expires_at}` | token shown once |
| `POST /v1/targets/{id}/challenges/{challenge_id}/verify` | — | `200 {basis_id, state: BasisState}` or Problem `challenge-failed` with `{reason}` | consumes the challenge on success |
| `POST /v1/targets/{id}/manual-review` | `{evidence, approved_scope}` | `201` | `admin` only; records reviewer identity |
| `DELETE /v1/targets/{id}/authorization` | — | `204` | explicit revocation; cancels pending and active work via outbox `cancel_run` |
| `GET/POST/PUT /v1/projects` | `{name}` | `Project` | labels only (§3.1) |
| `POST /v1/graphs/validate` | `FlowGraph` | `200 {valid, errors: [ValidationError]}` | server validator; same crate as WASM |
| `POST /v1/graphs/estimate` | `{graph, target_id, include_subdomains}` | `Estimate` | scope checked; no reservation |
| `POST /v1/pipeline-runs` | header `Idempotency-Key`; `{graph | template_id, target_id, include_subdomains, run_kind?: full}` | `202 PipelineRun{id, state, target_id, scope_snapshot, estimate, work_units: [{id, node_id, scan_type, operation, status}], created_at}` | same key + same hash ⇒ `200` same run; same key + different hash ⇒ `409 idempotency-conflict`; unapproved target ⇒ `403 authorization-required` with the track hints of §13.3; refused graph ⇒ `422 graph-invalid` and no run row |
| `GET /v1/pipeline-runs/{id}` | — | `PipelineRun` + `counters` | `RunSnapshot` shape also used by SSE resync |
| `DELETE /v1/pipeline-runs/{id}` | — | `202` | durable stop intent first (§8.3) |
| `GET /v1/pipeline-runs/{id}/events` | `Last-Event-ID?` | `text/event-stream` of `PipelineEvent` | §A3.3 |
| `GET /v1/events` | `Last-Event-ID?` | `text/event-stream` of `TenantEvent` | separate cursor |
| `GET /v1/scans/{scan_fingerprint}` | — | `Scan{fingerprint, work_unit, attempt_number, status, failure_class, destination_snapshot, started_at, completed_at}` | tenant-authorized lookup; the fingerprint is not a capability |
| `GET /v1/findings` | `?pipeline_run_id|scan_fingerprint|target_id|state|severity&after&limit` | `{items: [Finding], next: cursor}` | keyset on `(observed_at desc, id desc)`; never OFFSET |
| `GET /v1/findings/{id}` | — | `Finding` incl. `remediation: {tier, version, summary} | null` | |
| `GET /v1/findings/{id}/remediation` | — | `Remediation{tier, version, summary, steps, refs, effort, requires_restart, directly_verifiable}` or `{tier: "unavailable", refs}` | never fabricated |
| `POST /v1/findings/{id}/state` | `{state: acknowledged|fix_pending|accepted_risk|false_positive, note?, match_scope?}` | `200 Finding` | `false_positive` creates `FpDecision` and returns its id; illegal transition ⇒ `409 illegal-transition` |
| `POST /v1/findings/{id}/verify` | — | `202 {verification_run_id, pipeline_run_id, plan: {units: [...], known_units}}` | derived server-side; normal allowance |
| `DELETE /v1/false-positive-decisions/{id}` | — | `204` | revokes for future matching only |
| `GET /v1/usage` | — | `UsageSnapshot` + `entitlements` | the browser passes `entitlements` to the WASM validator |
| `GET /v1/usage/ledger` | `?period_id&after&limit` | ledger rows | |
| `GET /healthz`, `GET /readyz`, `GET /openapi.json` | — | — | unauthenticated |

Authentication: `Authorization: Bearer <JWT>` validated against the configured JWKS (Keycloak later; local dev issuer now, §A6.4). Claims `sub`, `tenant_id`, `roles[]`, `package`, `kyc_level` are read; protected write paths additionally read live `tenants.suspended_at` inside the transaction (§4.1). OpenAPI 3.1 is generated by utoipa at build time and the committed `openapi/vf-api.json` must equal it (`cargo test -p vf-api --test openapi_drift`, Halsey). `vf-web` generates its client from the committed file.

### A3.9 Tenant provisioning contract (`vf-operator`, `vf-db`)

```rust
pub struct TenantSpec { pub slug: Slug, pub package: Package, pub entitlements: Entitlements, pub quotas: Quotas, pub egress_profile: EgressProfile, pub data_region: Region }   // §3.3; kube-derive CustomResource behind the `kube` feature
pub trait TenantProvisioner { async fn ensure(&self, spec: &TenantSpec) -> Result<TenantStatus>; async fn suspend(&self, t: TenantId, reason) -> Result<()>; async fn restore(&self, t: TenantId) -> Result<()>; }
```

Steps are idempotent and each reports a condition (§3.3). Locally the Kubernetes steps (namespace, quota, NetworkPolicy, RBAC, pull secret) are recorded as `Skipped{reason: "no-kube"}` conditions by the `LocalTenantProvisioner`; schema creation with RLS, object prefix and wake-bus namespace run for real. `vf-operator provision-tenant --spec tenant.yaml` is the local CLI entry; the `Tenant` CRD controller wraps the same `ensure`.

---

## A4. Data model outline

Two schemas. `vf` holds control and reference tables; `tenant_<slug>` is created per tenant from one migration template and every table in it has `tenant_id` plus `FORCE ROW LEVEL SECURITY` with the `vulcanflow.tenant_id` GUC policy of §3.5. All tenant access goes through `TenantTx`, whose only constructor takes a `VerifiedTenant` produced by the auth layer or by a trusted work-unit record, opens the transaction, runs `SET LOCAL search_path` and `set_config('vulcanflow.tenant_id', …, true)`, and hands out `&mut TenantTx`. `ControlTx` is the separately named type for `vf.*` access. The application role `vf_app` is non-owner, non-superuser, no `BYPASSRLS`; migrations run as `vf_migrate`.

| Area | Tables (tenant schema unless noted) | Key columns and constraints (§ reference) |
|---|---|---|
| Tenants | `vf.tenants` (control) | `id, slug UNIQUE, package, entitlements jsonb, suspended_at, suspended_reason, schema_name, created_at`; `vf.tenant_users(tenant_id, user_id, role)` |
| Targets and projects | `projects`, `targets` | §6.3 verbatim: `UNIQUE (tenant_id, normalized)`; `kind ∈ {domain, subdomain}`; `normalized` is the `CanonicalHost` A-label |
| Authorization | `authorization_challenges`, `authorization_basis`, `authorization_events` | `authorization_basis`/`authorization_events` as §5.6 verbatim, `REVOKE UPDATE, DELETE … FROM vf_app`; challenges: `id, target_id, method, token_hash, expires_at, consumed_at` |
| Pipelines and work | `pipeline_runs`, `scan_work_units`, `scans`, `pipeline_events` | §6.3 verbatim plus `pipeline_events(pipeline_run_id, seq) UNIQUE`; `pipeline_runs.stop_requested_at`; `scan_work_units.parent_unit_id` for cascade lineage; `scan_work_units.skip_reason` |
| Allowances | `billing_period_usage`, `scan_usage_reservations`, `scan_usage_ledger`, `target_admissions` | §17.3 verbatim; `target_admissions(period_id, target_id) UNIQUE` |
| Observations | `findings`, `finding_states`, `assets` (`subdomain|host|port|service|endpoint` rows with `first_seen_at`, `last_seen_at`, per-run coverage in `run_coverage`) | `findings` as §6.3 incl. `embedding vector(768)` (pgvector, created now per §18.3); `UNIQUE (scan_id, source_finding_id)` |
| False positives | `false_positive_decisions`, `false_positive_events`, `false_positive_applications(decision_id, finding_id, match_version)` | §6.3 verbatim; applications record why a decision applied |
| Verification | `verification_runs` | §15.4 verbatim; `prior_observation_state` for the inconclusive restore |
| Reports (schema now, logic later) | `report_definitions`, `reports`, `report_deliveries(report_id, recipient, occurrence_key) UNIQUE` | §16.8 verbatim |
| Schedules (schema now, logic later) | `schedules`, `schedule_occurrences` | §11.1 verbatim |
| Reference (control) | `vf.remediation_content`, `vf.enrichment_cve`, `vf.enrichment_epss`, `vf.enrichment_kev`, `vf.templates(compliance_framing_required bool)` | §15.2 verbatim; mirrors are empty tables locally with a fixture loader |
| Plumbing (control) | `vf.outbox`, `vf.notification_nonces`, `vf.audit_log` (append-only, `REVOKE UPDATE, DELETE`), `vf.tenant_events` | §A3.3, §19.2 |

Migrations are forward-only SQL files under `crates/vf-db/migrations/{control,tenant}/`, applied by `sqlx::migrate!` with a per-tenant resumable fan-out (`vf-db migrate --all-tenants`) whose status lives in `vf.tenant_migrations`. sqlx compile-time checks run offline against committed `.sqlx/` query data generated from a migrated template schema.

---

## A5. Pinned toolchain and crate set (for review)

Versions are the latest stable on crates.io on 2026-10-05, chosen so the first `Cargo.lock` starts current. **These pins are a proposal for review (TDD §27-17).** Cortana approves the set as architect under the plan's "technical lead" responsibility; the owner may veto any entry on [VFL-8](/VFL/issues/VFL-8). The scaffold task records the approved set in `[workspace.dependencies]` and `Cargo.lock`; any later addition or bump is a reviewed change to this table.

| Concern | Pin | Why this pick |
|---|---|---|
| Toolchain | Rust **1.99.0** stable (2026-09-28) in `rust-toolchain.toml`, edition **2024**; targets `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `wasm32-unknown-unknown` | Current stable; one toolchain for all crates (§2.5.2). Nightly only for `cargo-fuzz`, pinned separately in `fuzz/rust-toolchain.toml` |
| Async | tokio **1.53** (`rt-multi-thread`, `macros`, `signal`), tokio-util 0.7, futures 0.3 | De facto runtime; required by axum, sqlx, kube |
| HTTP | axum **0.8**, axum-extra 0.12, tower 0.5, tower-http 0.7 (`trace`, `cors`, `limit`, `timeout`) | TDD choice; SSE built in; role policy as a tower layer |
| OpenAPI | utoipa **6.0**, utoipa-axum 0.3 | Code-first 3.1 generation; drift check in CI. The scaffold verifies 3.1 output and Problem Details modelling (§27-17) |
| Postgres | sqlx **0.9** (`runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `chrono`, `json`, `migrate`), sqlx-cli 0.9, pgvector 0.4 | Compile-time checked queries; offline mode. The harness runs PgBouncer in transaction mode so prepared-statement behaviour is tested now, not on Aether |
| Object storage | object_store **0.14** (`aws` feature) | One trait with S3, local filesystem and in-memory backends; path-style addressing and checksum behaviour are explicit config. Chosen over aws-sdk-s3 (heavier, AWS-default assumptions). Conformance is run against RustFS and MinIO locally, Ceph RGW later |
| Wake bus / rate limits | redis **1.7** (`tokio-comp`, `connection-manager`), governor 0.10 | Valkey-compatible; connection manager handles reconnects. `fred` is the alternate if pub/sub ergonomics prove poor |
| Kubernetes | kube **4.2** (`runtime`, `derive`, `rustls-tls`, `admission`), kube-runtime 4.2, k8s-openapi **0.28** (newest supported `v1_xx` feature, fixed in the scaffold and re-pinned when Aether's version is known), kopium 0.24 (tool) | TDD choice. Feature-gated behind `kube` so `vf-operator` and `vf-admission` build and test without it |
| OIDC / JWT | jsonwebtoken **11**, openidconnect 4 | JWKS caching; PKCE flow is browser-side |
| TLS / crypto | rustls **0.23**, hmac 0.13, sha2 0.11, rand 0.10 (OS CSPRNG), secrecy 0.10, hex 0.4, base64 0.23 | No OpenSSL; RustCrypto for HMAC and token generation |
| Serialization | serde 1, serde_json 1, schemars **1.2**, jsonschema 0.58, serde_yaml_ng 0.10 | `deny_unknown_fields` on every external input; schemars publishes the graph schema; jsonschema validates SCB finding envelopes |
| Domains | idna **1.1**, url 2.5, psl **2.1** | UTS-46 canonicalization; PSL compiled in (including private suffixes). Refresh is a reviewed crate bump on a monthly reminder; `publicsuffix` (runtime list) is the fallback if staleness becomes a problem |
| DNS and HTTP probes | hickory-resolver **0.26**, reqwest 0.13 (`rustls-tls`, `json`, no default features) | Challenge verification with explicit resolvers; redirect policy implemented by hand per §5.2 |
| Time and scheduling | chrono 0.4, chrono-tz 0.10, croner 4.0 (M4) | Timezone-aware cron with DST rules testable per §11.1 |
| Templating (M4) | askama 0.16, ammonia 4.2 | Compile-time templates so the disclaimer block cannot be omitted; allow-list sanitizer for rich text |
| Observability | tracing 0.1, tracing-subscriber 0.3, opentelemetry 0.33, tracing-opentelemetry 0.34, prometheus-client 0.25 | §19; JSON logs locally, OTLP exporter configured but optional |
| Errors and ids | thiserror 2, uuid 1.27 (`v7`, `serde`), ipnet 2.12 | Newtypes over v7 UUIDs; CIDR checks for private-destination refusal |
| WebAssembly | wasm-bindgen 0.2.129, wasm-bindgen-test 0.3, wasm-pack 0.15 (tool) | `vf-graph` browser build; size budget 300 KiB gzipped (proposed) |
| Testing | proptest 1.11, rstest 0.27, insta 1.49, testcontainers 0.28 + testcontainers-modules 0.15, wiremock 0.6, tower-test 0.4, cargo-fuzz 0.13 (tool) | §23.1 layers; wiremock for HTTP challenge and notification tests; tower-test for a mocked kube client |
| Supply chain | cargo-deny 0.20, cargo-audit 0.22; `deny.toml` with licence allow-list (MIT, Apache-2.0, BSD-2/3, ISC, Unicode-3.0, MPL-2.0 for listed crates only), advisories deny, sources = crates.io only, bans for openssl-sys and the crate-graph rules of §A1.4 | §21.3 |
| Email (M4) | lettre 0.11 | SMTP to Mailpit locally |
| Renderer (M4) | chromiumoxide 0.9 or headless Chromium CLI; decided in the M4 contract freeze | §16.4 |
| ClickHouse (M3+) | clickhouse 0.15 | Not used by any task in this document |

Engineering rules in every crate: `#![forbid(unsafe_code)]`, `#![deny(clippy::all, clippy::unwrap_used, clippy::expect_used)]` on library crates (binaries allow `expect` only in `main` setup), `clippy -D warnings` in `just check`, no `unwrap` on external input paths, panics caught at the service boundary and reported as platform errors.

---

## A6. Local development and test strategy without infrastructure

### A6.1 The adapter boundary

All infrastructure is reached through port traits declared in `vf-core::ports` and implemented in the owning crate. Application code depends on `Arc<dyn Port>` injected at startup from configuration; no application crate names a concrete adapter.

| Port | Real adapter (Aether, later) | Local adapter (now) | Test adapter | Where implemented |
|---|---|---|---|---|
| Postgres (not a port; same technology both sides) | CNPG cluster via PgBouncer | `postgres:17` + `pgbouncer` containers in `docker-compose.yml` | testcontainers Postgres with migrated template | `vf-db` |
| `ArtifactStore` | object_store S3 → Ceph RGW / RustFS | object_store S3 → `rustfs/rustfs` container (MinIO alternate profile) | `InMemoryArtifactStore`, `LocalFsArtifactStore` | `vf-db::adapters` |
| `WakeBus` (publish/subscribe wake-ups, token buckets) | redis → Valkey | redis → `valkey/valkey` container | `InProcessWakeBus` (tokio broadcast) | `vf-db::adapters` |
| `ScanRuntime` | kube-rs → secureCodeBox CRDs | `FakeScanRuntime` (fixture-scripted) | same fake with scripted failures | `vf-operator` |
| `TenantProvisioner` | kube + schema + prefix | `LocalTenantProvisioner` (schema + prefix + bus namespace; kube steps skipped) | same | `vf-operator` |
| `ChallengeProbe` | hickory + reqwest to the internet | same against a local authoritative DNS fixture (`hickory-server` in `vf-testkit`) and a wiremock HTTP host | `FixtureProbe` | `vf-authz` |
| `TokenVerifier` | Keycloak JWKS | dev issuer (§A6.4) or optional Keycloak compose profile | static test keys | `vf-api` |
| `Clock`, `IdGen` | system | system | deterministic | `vf-core` |
| `Mailer` (M4), `Renderer` (M4) | provider, Chromium | Mailpit, Chromium container | fakes | later |

Configuration selects adapters by name (`VF_ARTIFACT_STORE=s3|fs|memory`, `VF_SCAN_RUNTIME=kube|fake`, `VF_WAKE_BUS=redis|inprocess`, `VF_AUTH=jwks|dev`). The binaries refuse to start with `fake`, `memory` or `dev` adapters when `VF_ENV=production`; this refusal is tested.

### A6.2 Local harness

`docker-compose.yml` at the `platform` root (neutral lane) with services `postgres`, `pgbouncer`, `rustfs`, `valkey`, optional profiles `minio`, `keycloak`, `mailpit`. A `justfile` provides `dev-up`, `dev-down`, `db-migrate`, `db-template` (creates the sqlx template schema and refreshes `.sqlx/`), `check` (fmt, clippy, deny, audit), `test-unit`, `test-integration` (`--features integration`, needs compose), `run-local` (starts `vf-api`, `vf-operator --runtime fake`, `vf-ingest` together), `gate` (runs `ci/lane-gate.sh all` against `origin/main`). The walking loop on one machine is: `just dev-up && just db-migrate && just run-local`, then register a target, pass a fixture challenge, submit the fixed `subfinder → dnsx → httpx → nuclei` template, watch SSE, see findings from the fixture artifacts, settle the allowance. No Kubernetes is involved.

### A6.3 How each crate is proven in isolation

| Crate | Unit and property tests (`crates/<crate>/tests`) | Integration (`--features integration`, compose or testcontainers) | Fuzz (`fuzz/`) |
|---|---|---|---|
| `vf-core` | state transitions, role matrix exhaustiveness, `CanonicalHost` corpus (IDNA, PSL private suffixes, trailing dots, case), scope matcher properties (`is_descendant_of` never matches label prefixes; `ExactHost` never widens) | none | `canonical_host`, `scope_match` |
| `vf-graph` | validator corpus with golden results `conformance/graph/*.json` + `*.golden.json`; schema snapshot | none | `graph_dsl_parse` |
| `vf-translator` | argv golden files per node type; candidate key determinism; cascade scope refusal; serialized `Scan`/`CascadingRule` snapshots | none | `argv_builder` |
| `vf-db` | query builders, FP match key canonical serialization | RLS: wrong/missing tenant context, pooled connection reuse after commit/rollback/error through PgBouncer, cross-schema reference refusal, outbox claim/lease/reclaim, migration fan-out resume | none |
| `vf-meter` | `Reservation` type usage (compile-fail tests via trybuild for double settle) | concurrent reservations (N tasks against one period), retry holds reservation, settle-once, 4-units and 3-units examples | none |
| `vf-authz` | challenge token properties, basis resolver temporal rules, `decide()` table over every `RefuseReason` | DNS-TXT against local authoritative server, HTTP file through wiremock incl. cross-host redirect, downgrade, private destination, oversize body | `notification_verifier` (shared with ingest) |
| `vf-admission` | `AdmissionReview` fixtures (Scan/Job/Pod; allowed, forbidden field, target mismatch, unknown unit, suspended tenant); fail-closed on DB error | handler against compose Postgres | none |
| `vf-operator` | reconcile state machine with `FakeScanRuntime` scripts: success, tool failure, platform failure with retry, cancellation mid-run, revocation mid-run, late cascade after finalization refused, adoption of existing object | full loop with compose Postgres and in-process ingest | none |
| `vf-ingest` | notification verification matrix, bounded parser on fixture corpus (zero findings, malformed, oversize, duplicate ids), FP matcher application, settlement idempotency | replayed artifact is a no-op; reconciliation sweep recovers a missed notification | `findings_artifact`, `notification_verifier` |
| `vf-api` | role layer, Problem mapping, OpenAPI drift, idempotency conflict, SSE replay/resync | end-to-end `tests/walking_skeleton.rs`: approved target → run → fixture findings → triage → verify → usage | none |
| `vf-scanner-adapter`, `vf-hook-notify` | argv/`Command` construction with a stub tool script; manifest classification; signature output | invoked by `FakeScanRuntime` in the loop test | none |
| `vf-web` | vitest unit tests; MSW handlers generated from `openapi/vf-api.json`; Playwright smoke against `just run-local` | — | — |

Kubernetes-only paths (real admission registration, CRD watch, NetworkPolicy, pool) are the explicit later work: a `kind` profile is reserved in the harness (`just kind-up`) but **no task in this document is verified on it**. `KubeScanRuntime` is unit-tested against a mocked kube client (tower-test) so it compiles and serializes correctly today.

### A6.4 Local identity

`vf-api` feature `dev-auth` adds a `vf-api dev-issuer` subcommand that serves a JWKS and mints tokens with chosen `tenant_id`, `roles`, `package`, `kyc_level`; `vf-testkit::token(tenant, role)` wraps it for tests. The feature is excluded from release profiles and the binary refuses `VF_AUTH=dev` under `VF_ENV=production`. Keycloak remains the approved identity provider; a compose profile for it is provided for anyone who wants the real PKCE flow locally, but no task depends on it.

---

## A7. Security and isolation invariants every task preserves

From §1.2, §3, §5, §17, §22, §23 and the plan's M1 exit evidence. A pull request that weakens any of these fails review regardless of findings severity.

1. **No execution without basis, scope, reservation and evidence.** The dispatcher, the operator and the admission handler all call the same `decide()`; a Scan, Job or Pod that cannot be matched to a reserved work unit with a live basis is refused. Admission failure is closed, including on database error (`authz/start-barrier-all-paths`, `execution/gates-before-start`).
2. **Scope cannot widen.** Only `CanonicalHost` and `Ipv4Destination` reach argv, scope checks and admission. `ExactHost` approvals never cover descendants; `DomainTree` runs include descendants only when discovery was selected; a cascade candidate outside scope is recorded `Skipped{Scope}` with a visible reason, never silently run (`authz/configured-scope`, `authz/psl-exact-root`).
3. **Revocation stops work.** Revocation or suspension writes durable state first, then cancels pending outbox rows, running units and cascades; queued starts, retries and admission re-check live state. JWT claims are never sufficient on protected paths (`auth/suspended-token`).
4. **Accounting is atomic and settles once.** Reservation, settlement and release happen in one transaction with the period row locked; failed and skipped units consume zero; retries reuse the reservation; `UNIQUE (work_unit_id, event_type)` plus the state check make double settlement impossible (`usage/*`).
5. **Fresh observations, narrow carry-forward.** Every scan creates new `findings` rows; only an explicit false-positive decision applies to later observations, through the versioned matcher; acknowledgment, accepted risk and historical fixes never carry forward (`findings/new-per-scan`, `findings/fp-only-persistence`, `findings/new-after-fix`).
6. **Ingest is idempotent and bounded.** Duplicate notifications and artifacts change nothing; signature, timestamp, nonce, prefix and checksum are verified against trusted work records before any parse; parsing is size-bounded and streaming; malformed input is a platform outcome, never a target outcome (`findings/replayed-artifact`, `fuzz/untrusted-input`).
7. **Tenant context is a type.** Tenant tables are reachable only through `TenantTx`; RLS is forced on every tenant table; the app role cannot bypass RLS; background workers derive tenant identity from trusted records, never from payloads (`isolation/all-stores`, `isolation/background-queries`).
8. **Identities are distinct types.** `PipelineRunId`, `WorkUnitId`, `ScanFingerprint` and `FindingId` are never interchangeable; the fingerprint is stored unchanged and never derived from content (`execution/scan-identity`).
9. **Verification never claims clean without evidence.** Only an applicable, completed check against a reachable asset yields `not_detected`; blocks, skips, missing templates and missing allowance yield `inconclusive` (`verify/applicable-check-evidence`).
10. **Untrusted input is parsed defensively.** `deny_unknown_fields` on every external schema; no `unwrap` on external paths; `forbid(unsafe_code)`; attacker-influenced parsers are fuzzed; `Cargo.lock` committed; `cargo deny` and `cargo audit` pass.
11. **Local stand-ins cannot leak into production.** Fake, memory and dev-auth adapters are refused under `VF_ENV=production`, by test.
12. **No remote push without the gates.** Code stays local until Arbiter (CodeRabbit) and Opus Reviewer have reviewed, and Test Runner has passed, the exact candidate pair (code revision, test revision); Cortana records the approval naming both revisions. The lane gate on `platform` main mechanically enforces that no pull request mixes production source and tests.

---

## A8. Open decisions that remain with the owner

| # | Decision | Owner | Proposed default used meanwhile | Gates |
|---|---|---|---|---|
| O1 | Crate and toolchain pin set in §A5 (TDD §27-17) | Owner veto; Cortana approves as architect | Table §A5 | Workspace scaffold proceeds on the proposed set; a veto re-pins before the next task |
| O2 | secureCodeBox release for CRD type generation and fixtures (TDD §27-5, §27-20) | Owner, with Kelly's evidence | v5.9.0 | `vf-translator` CRD types and the scanner fixture corpus; regeneration is one command if changed |
| O3 | Re-affirm the `platform` lane gate (its header cites deleted ADR-0005 and its messages name retired agents Scribe, Ledger, Forge, Anvil, Kiln, Atlas) | Owner | Keep the gate as is; a gate-only pull request later updates the wording to Halsey, Fred, Kelly, Jorge, Linda, Cortana | Nothing; informational until the wording change is requested |
| O4 | AAAA observations (TDD §27-7) | Owner / product | `dnsx` node config exposes `aaaa` with default `false`; AAAA literals never enter execution | No first task; affects the `dnsx` config default before M2 conformance |
| O5 | False-positive equivalence fields and persistence enablement (TDD §27-6) | Owner accepts engineering evidence at M1 exit | Matcher v1 as §A3.6 runs and records; `fp_persistence_enabled` tenant flag default off | Turning the flag on in any environment |
| O6 | Target-slot semantics (TDD §27-3) | Product | Active registered targets per period | None now; `vf-meter::admit_target` counting rule only |
| O7 | Local S3 stand-in preference: RustFS (Aether candidate) vs MinIO | Owner / platform | RustFS default, MinIO profile kept | None; both conformance suites run |
| O8 | Report disclaimer wording, selection semantics, delivery consent (TDD §27-9, 11, 12) | Legal / product | None needed now | All `vf-report` tasks (not in this document) |
| O9 | Package values and entitlement defaults (TDD §27-2) | Product | Fixture entitlements in `vf-testkit`, labelled illustrative | Paid use only |
| O10 | Kubernetes version of Aether (selects the `k8s-openapi` feature) | Owner / platform | Newest feature the pinned kube-rs supports | Nothing locally; re-pin before the first `kind` run |

None of these blocks a task in the [task matrix](/VFL/issues/VFL-8#document-task-matrix). Each is referenced from the task it would later affect.
