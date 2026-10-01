# ADR-0002 — Approved Rust crate set, toolchain pin, and secureCodeBox pin

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | TDD **§27 item 17** |
| **Also settles** | the engineering half of **§27 item 5** and the `[OPEN — eng]` secureCodeBox paragraph in **§21.3** (selected release, digests, arm64, CRD type generation) |
| **Does not close** | §27 items 16a, 18 (see [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md)), 19, 20, 21 (see [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md)) |
| **Supersedes** | the `[PROPOSED]` crate table in TDD §2.5.2 |
| **Design of record** | [`VulcanFlow_Technical_Design_Document_v2.3.md`](../VulcanFlow_Technical_Design_Document_v2.3.md) — **TDD v2.3** — §2.5, §21.3, §24.2, §24.4, §25 |
| **Issue** | VUL-3 |

## Question

TDD §2.5.2 lists tokio, axum, utoipa, sqlx, kube-rs, askama and "an S3 client" as
`[PROPOSED]`, and says plainly: *"do not treat any crate named here as installed,
compatible, or approved until §27 item 17 closes."* §24.4 makes approval of the crate set
used on the execution path a blocking dependency before execution. §27 item 17 asks
engineering to:

> Approve the §2.5.2 crate set and pin versions; confirm utoipa OpenAPI 3.1 output, kube-rs
> admission/controller coverage, sqlx behaviour behind PgBouncer transaction pooling
> (prepared statements), and the S3 client's compatibility with Ceph RGW/RustFS.

So: approve the set, pin exact versions, and produce evidence for the four named risks
rather than an assurance.

### A scope correction, recorded because it would otherwise propagate

The VUL-3 issue body instructed that pinning the secureCodeBox release and specifying CRD
Rust type generation would close **§27 item 21**. Reading §27 directly, that mapping is
wrong. Item 21 is *"Lago and Stripe client approach in Rust"* and blocks Phase 4; it is
decided in [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md). The secureCodeBox
release/digest/compatibility question is **§27 item 5** plus the `[OPEN — eng]` paragraph in
**§21.3**, and it is decided in §6 of this document. The work asked for was the right work;
only the item number was wrong. Recording the correction here so that a future reader
reconciling ADRs against §27 does not conclude an item was closed twice or not at all.

---

## 1. Decision

**The crate set in §3 is approved and pinned. §27 item 17 is closed.**

Every version below was read from the crates.io sparse index and API on 2026-10-01, not
from memory. Declared dependency ranges were cross-checked (§3.3) so the set resolves as a
unit rather than as a list of individually plausible crates.

---

## 2. Toolchain pin

`rust-toolchain.toml`, committed at every workspace root:

```toml
[toolchain]
channel = "1.98.1"
components = ["rustfmt", "clippy"]
targets = ["aarch64-unknown-linux-gnu"]
profile = "minimal"
```

Rust 1.98.1 is current stable. The binding MSRV constraints in the approved set are
`aws-sdk-s3` (1.94.1) and `sqlx` (1.94.0); 1.98.1 clears both with room.
`aarch64-unknown-linux-gnu` is listed because Aether is arm64 and §21.3 requires
reproducible arm64 builds.

`Cargo.lock` is committed for every crate in the workspace, **including library crates**.
Version churn in the AWS SDK (weekly releases) is therefore invisible to builds: the pins
below are the *decision* points and `Cargo.lock` is the *reproducibility* mechanism. AWS
SDK crates are refreshed deliberately, never automatically.

Baseline, in every crate root, per §2.5.2:

```rust
#![forbid(unsafe_code)]
```

---

## 3. Approved and pinned crate set

Versions are the exact pins for `Cargo.toml`. Use `=`-pinning only where noted; elsewhere
caret ranges plus the committed `Cargo.lock` are sufficient.

### 3.1 Runtime, HTTP, API surface

| Crate | Pin | Role | Notes |
|---|---|---|---|
| `tokio` | 1.53.1 | async runtime | features `rt-multi-thread`, `macros`, `net`, `time`, `signal`, `sync`. **Not** `full`. |
| `axum` | 0.8.9 | HTTP server | 0.8.x since 2025-04. Role policy as a tower layer (§4.2). |
| `tower` | 0.5.3 | middleware | |
| `tower-http` | 0.7.1 | middleware | `trace`, `timeout`, `limit`, `request-id`. |
| `utoipa` | 6.0.0 | OpenAPI 3.1 generation | See probe §4.1. |
| `utoipa-axum` | 0.3.0 | axum router integration | Requires `utoipa ^6.0.0` — this is why 6.x and not 5.5.0. |
| `utoipa-swagger-ui` | 10.0.1 | spec UI | Requires `utoipa ^6.0.0`. Non-production surface only. |
| `askama` | 0.16.1 | report templating | See §5. |
| `serde` | 1.0.229 | serialization | `derive`; `deny_unknown_fields` on all external input types. |
| `serde_json` | 1.0.151 | JSON | |
| `thiserror` | 2.0.21 | error types in library crates | Libraries use `thiserror`, never `anyhow`. |
| `anyhow` | 1.0.104 | error type at binary boundaries | Binaries and tests only. |
| `tracing` | 0.1.44 | structured logging | §19. |
| `tracing-subscriber` | 0.3.23 | log/trace export | `env-filter`, `json`. |
| `uuid` | 1.26.1 | identifiers | `v7`, `serde`. v7 for time-ordered keys. |
| `jiff` | 0.2.37 | time | Chosen over `chrono`: correct civil/zoned modelling, which §11.1's DST tests need. |
| `reqwest` | 0.13.5 | outbound HTTP | `rustls-tls`, `json`; `default-features = false` (no OpenSSL). |

### 3.2 Database, Kubernetes, object store

| Crate | Pin | Role | Notes |
|---|---|---|---|
| `sqlx` | 0.9.0 | Postgres | `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `macros`. See probe §4.3. |
| `kube` | 4.2.0 | K8s client, admission, controllers | `client`, `derive`, `runtime`, `admission`, `rustls-tls`; `default-features = false`. See probe §4.2. |
| `k8s-openapi` | 0.28.0 | K8s API types | **`=`-pin.** `kube 4.2.0` declares `k8s-openapi ^0.28.0`; mismatch here is the classic kube-rs build break. Pin the k8s version feature to the Aether cluster minor. |
| `json-patch` | 4.2.0 | mutating-admission patches | Re-exported through `kube::core::admission`; pin to what `kube 4.2.0` resolves. |
| `aws-sdk-s3` | 1.151.0 | object store client | **Must be constructed per §4.4 — the defaults are wrong for non-AWS backends.** |
| `aws-config` | 1.12.0 | credential/region resolution | Static credential provider only. No IMDS, no profile files. |

### 3.3 Resolution cross-check (performed, not assumed)

Declared ranges read from the index on 2026-10-01:

- `kube 4.2.0` → `k8s-openapi ^0.28.0`, `kube-core =4.2.0`, `kube-client =4.2.0` ✓
- `utoipa-axum 0.3.0` → `utoipa ^6.0.0`, `axum ^0.8.4` ✓
- `utoipa-swagger-ui 10.0.1` → `utoipa ^6.0.0`, `axum ^0.8.4` ✓

The set is internally consistent, and this also settles the utoipa 5-vs-6 question: the
axum integration crate requires 6.x, so pinning 5.5.0 would mean hand-rolling router
integration. **Named risk:** utoipa 6.0.0 is nine days old at decision time. Accepted,
because the mitigation (§4.1, `api/openapi-3_1-conformance`) is a CI gate that fails loudly
rather than a hope.

### 3.4 Test-side crates

Pinned here because pinning is a workspace convention. Test *content* belongs to Scribe and
Ledger; test *execution* to Crucible. Listing a crate here does not make test authorship
mine.

| Crate | Pin | Used by |
|---|---|---|
| `proptest` | 1.11.0 | Scribe — property tests on scope matching, allowance arithmetic, observation state |
| `arbitrary` | 1.4.2 | Scribe — fuzz target input derivation (`fuzz/untrusted-input`) |
| `insta` | 1.48.0 | Scribe / Ledger — snapshot assertions on generated OpenAPI and reports |
| `testcontainers` | 0.28.0 | Ledger — Postgres, PgBouncer, object-store containers |
| `testcontainers-modules` | 0.15.0 | Ledger — `postgres` module |

### 3.5 Explicitly excluded

- **Go and every v2.2 Go framework** — chi, Huma, controller-runtime, Connect-Go are
  withdrawn by §0.0/§2.5 and are not reinstated here. See ADR-0001.
- **`chrono`** — superseded by `jiff` for new code.
- **Floating-point and decimal crates anywhere in the allowance path** — scan allowances
  reserve and settle in `i64`. No `f64`, no `rust_decimal`. This is a hard constraint, not
  a preference: §17.3 accounting errors are billing bugs.
- **`object_store` / `rust-s3`** — see §4.4 for why `aws-sdk-s3` won.
- **`openssl` / `native-tls`** — rustls everywhere; `default-features = false` on anything
  that would otherwise pull OpenSSL in.

---

## 4. Risk probes — what was actually observed

**Scope statement, stated up front so the evidence is not over-read.** This work had
outbound network access but **no `cargo`, no `rustc`, and no container runtime**. Each probe
below is a *source-level* probe: upstream source at the exact pinned tag was read directly
and what it says is reported. That is strong evidence for questions settled by the code's
own structure — which OpenAPI version is emitted, which admission fields exist, which
config knobs are available, what the defaults are — and it is **not** a substitute for
execution where the question is behavioural. Each probe therefore ends with the executable
confirmation that is still required and who owns it. Nothing below is an assurance
presented as a result.

### 4.1 utoipa — is the generated output really OpenAPI 3.1?

**Observed.** `utoipa/src/openapi.rs` and `utoipa/src/openapi/schema.rs` at tag
`utoipa-6.0.0`; `openapi.rs` at `utoipa-5.5.0` for comparison.

- The version field is typed, not stringly: `pub openapi: OpenApiVersion`, with
  `OpenApiVersion::Version31` carrying `#[serde(rename = "3.1.0")]`. The builder documents
  "Defaults to `OpenApiVersion::Version31`", and an in-tree unit test asserts
  `serde_json::to_value(&OpenApiVersion::Version31)? == "3.1.0"`. 5.5.0 is 3.1-only; 6.0.0
  adds opt-in `Version32` (`"3.2.0"`) and keeps 3.1 as the default.
- The declared version being right is the cheap half. The real failure mode is a document
  that *claims* 3.1.0 while emitting 3.0-dialect schema constructs, which a validator
  rejects. Checked specifically, and 6.0.0 is 3.1-correct by construction:
  - **No `nullable` keyword.** There is no `pub nullable` field and no
    `rename = "nullable"` anywhere in `schema.rs`. `nullable` is 3.0-only.
  - **Nullability via JSON Schema type arrays**, the 3.1 form: `Array::new_nullable` sets
    `schema_type: SchemaType::from_iter([Type::Array, Type::Null])`.
  - **`exclusive_minimum: Option<Number>`** — the numeric JSON Schema form. 3.0 modelled
    this as a boolean modifier; emitting a boolean here would fail validation.
  - **`SchemaType::AnyValue` omits the `type` key entirely** rather than inventing one,
    which is the correct 3.1 treatment of an unconstrained schema.
  - **`jsonSchemaDialect` is supported** (`pub json_schema_dialect: Option<String>`, plus a
    `$schema` override), a 3.1-and-later keyword.

**Conclusion.** utoipa 6.0.0 emits OpenAPI 3.1 with a 3.1-correct schema dialect. Approved
for the §2.5.2 "OpenAPI 3.1 (code-first)" slot.

**Residual, and it is a real one.** Source inspection proves the *library* can express 3.1
correctly. It does not prove that *our* annotations — our newtypes, enums, flattened
structs and `Option<T>` fields — produce a document that passes a 3.1 validator. That is a
property of our code, not of utoipa, and only generating our spec settles it. Problem
Details modelling (§2.5.2 names it) is in the same category.

**Executable confirmation — required, not optional.** Test ID
`api/openapi-3_1-conformance`: `vf-api` emits its spec to a file in CI; the spec is
validated against the OpenAPI 3.1 metaschema; the job fails on any error. Asserts both
`openapi == "3.1.0"` and metaschema validity, and snapshots the Problem Details schema.
Spec-emitting binary target: **Anvil**. Test: **Ledger**. CI wiring: **Crucible**.

### 4.2 kube-rs — does admission cover what `vf-admission` needs?

**Observed.** `kube-core/src/admission.rs` at tag `4.2.0`.

- `AdmissionReview<T: Resource>`, `AdmissionRequest<T>` and `AdmissionResponse` are all
  present, with `into_review()` producing the wire response.
- **Validating path:** `AdmissionResponse::deny(reason)` sets `allowed = false`;
  `invalid(reason)` is provided for malformed input.
- **Mutating path:** `with_patch(json_patch::Patch) -> Result<Self, SerializePatchError>`,
  which sets `patch_type = Some(PatchType::JsonPatch)`. JSON Patch is the only patch type
  Kubernetes accepts for admission, so this is complete rather than merely partial.
- **Every field the §5.7 gates need is present on `AdmissionRequest`:** `operation`
  (`Operation` enum), `object: Option<T>`, `old_object: Option<T>`, `dry_run: bool`,
  `sub_resource`, `request_kind`, `request_resource`, `request_sub_resource`,
  `user_info: UserInfo`, `options: Option<RawExtension>`, `namespace`, `name`, `uid`.
  - This is the load-bearing detail for the **cascade gate**: on `DELETE`, Kubernetes sends
    `object: null` and populates `oldObject`. A gate that only reads `object` silently
    passes every delete. `old_object` is present, so the gate is implementable correctly —
    and a PR whose cascade gate reads only `object` is a required change at review.
  - `dry_run` is present, so the **workload gate** can avoid side effects on dry-run
    requests, as an admission webhook is required to.
- kube 4.x additionally exposes `AdmissionRequest::to_cel_request()`, with `dry_run` passed
  through (covered by upstream tests). CEL-based policy is available later if wanted;
  Phase 1 does not use it.
- **One thing kube-rs does not provide:** there is no webhook *server*. `admission.rs`
  contains no HTTP or TLS machinery. `vf-admission` serves the webhook itself on axum 0.8.9
  over rustls and owns its own certificate lifecycle. That is a scope fact for Kiln, not a
  defect — and it is the origin of risk R1 in §7.

**Conclusion.** kube-rs 4.2.0 covers both the validating and mutating cases, including the
cascade and workload gates. Approved. **Named risk:** kube 4.x is a recent major line
(4.0.0 on 2026-06-16, 4.2.0 on 2026-07-22). Mitigation is the `=`-pin on
`k8s-openapi 0.28.0` and not adopting 4.x minors without a deliberate bump.

**Executable confirmation.** Test IDs `admission/cascade-gate-delete-oldobject` and
`admission/workload-gate-dryrun`, driven by replaying recorded `AdmissionReview` JSON
through the handler — **no cluster required**, which is why these are not in §7. Upstream's
own `WEBHOOK_BODY` fixture in `admission.rs` is a usable shape reference. Handler: **Kiln**.
Tests: **Ledger**. These feed `authz/start-barrier-all-paths` and
`execution/gates-before-start` in §25.

### 4.3 sqlx — do prepared statements survive PgBouncer in transaction pooling mode?

**Observed.** `sqlx-postgres/src/options/mod.rs` at tag `v0.9.0`, and PgBouncer's
`doc/config.md` on `master`.

- **sqlx side:** `PgConnectOptions` carries `statement_cache_capacity: usize`, **defaulting
  to 100**, with a public setter and documented LRU eviction. Setting it to `0` disables the
  cache. sqlx uses the extended query protocol either way — capacity 0 means statements are
  not *retained and reused* across the connection, not that sqlx stops preparing.
- **PgBouncer side:** `max_prepared_statements`, quoted from upstream docs — *"When this is
  set to a non-zero value PgBouncer tracks protocol-level named prepared statements related
  commands sent by the client in transaction and statement pooling mode. PgBouncer makes
  sure that any statement prepared by a client is available on the backing server
  connection. Even when the statement was originally prepared on another server
  connection."* It does this by rewriting client statement names to internal
  `PGBOUNCER_{unique_id}` names and transparently re-preparing on whichever server
  connection the client lands on. *"When the setting is set to 0 prepared statement support
  for transaction and statement pooling is disabled."* The documented limitation is that
  **SQL-level `PREPARE`/`EXECUTE`/`DEALLOCATE` are not tracked** and pass straight through;
  `DEALLOCATE ALL` and `DISCARD ALL` are handled. Available in PgBouncer 1.21+; current
  releases 1.26.0 (2026-09-23) and 1.25.2 (2026-05-08).

**Decision — the statement-cache strategy, stated explicitly as item 17 requires:**

1. **Run PgBouncer in transaction pooling mode with `max_prepared_statements = 200`**, set
   above sqlx's cache capacity so PgBouncer is never the binding constraint. Minimum
   PgBouncer **1.25.2**; 1.26.0 permitted.
2. **Leave sqlx's `statement_cache_capacity` at its default of 100.** Do not disable it.
   Protocol-level prepared statements are exactly what PgBouncer's tracking covers;
   disabling the cache would forfeit the benefit to defend against a problem
   `max_prepared_statements` already solves.
3. **Never issue SQL-level `PREPARE`/`EXECUTE`/`DEALLOCATE`** from any VulcanFlow code.
   These are explicitly outside PgBouncer's rewriting and would break under transaction
   pooling. A `PREPARE` in a diff is a required change at review.
4. **`TenantTx` must set tenant context with `SET LOCAL`, never a bare `SET`.** Under
   transaction pooling a session-level `SET` leaks to the next borrower of that server
   connection. **This is a tenant-isolation requirement, not a performance note** — a leaked
   session GUC is a cross-tenant RLS bypass, and §3.5 already requires transaction-local RLS
   context. It is the single highest-consequence consequence of choosing transaction
   pooling, and it is a mandatory review check on every `vf-db` PR.
5. **Pin the PgBouncer image by digest** in the deployment manifests when they are written.

**Revisit trigger.** If PgBouncer's prepared-statement tracking misbehaves under our load,
fall back to `statement_cache_capacity(0)` plus `max_prepared_statements = 0` and accept the
per-query parse cost. That fallback is a one-line change in both places, which is why this
decision is low-risk to make now.

**Executable confirmation.** Test IDs `db/pgbouncer-transaction-pooling-prepared` and
`db/tenanttx-set-local-isolation`. Testcontainers stands up Postgres + PgBouncer in
transaction mode; the second test asserts that a tenant GUC set inside `TenantTx` is **not**
observable on a subsequently borrowed pooled connection. `TenantTx`: **Forge**. Tests:
**Ledger**. Execution: **Crucible**. These feed `isolation/background-queries` and
`isolation/all-stores` in §25.

### 4.4 S3 client — does it conform against Ceph RGW and RustFS, not AWS?

**Observed.** `sdk/s3/src/config.rs` in `awslabs/aws-sdk-rust` and
`aws-smithy-types/src/checksum_config.rs` in `smithy-lang/smithy-rs`.

The decisive finding is a live defect waiting to happen, not a theoretical concern:

- `RequestChecksumCalculation` and `ResponseChecksumValidation` both have
  **`#[default] WhenSupported`**, confirmed in the enum definitions. With the default, the
  SDK attaches a CRC-based checksum to every upload and validates checksums on every
  download, and `WhenSupported` means it sends them even where the operation does not
  require them. Non-AWS S3 implementations have historically rejected or mishandled those
  headers. **An out-of-the-box `aws-sdk-s3` client is therefore *expected* to fail against
  Ceph RGW and RustFS until these are set to `WhenRequired`.** §2.5.2's warning that
  "AWS-oriented defaults must not be assumed" is concretely about these two.
- Both are configurable: `Builder::request_checksum_calculation(...)` and
  `Builder::response_checksum_validation(...)`, with `set_*` variants.
- `force_path_style(bool)` is present (`Builder::force_path_style`, backed by a
  `ForcePathStyle` config type). Virtual-host-style addressing needs wildcard DNS per
  bucket, which Aether — IPv4-only and self-operated — does not provide.

**Decision.** `aws-sdk-s3 1.151.0` is approved, **and the following configuration is
mandatory, not advisory**. A client constructed without it is a bug:

```rust
// The ONE place an S3 client is constructed. A second `Client::new` anywhere
// in the tree is a required change at review.
let cfg = aws_sdk_s3::config::Builder::from(&shared_config)
    .endpoint_url(endpoint)                 // explicit; never AWS-resolved
    .force_path_style(true)                 // Aether has no per-bucket wildcard DNS
    .request_checksum_calculation(RequestChecksumCalculation::WhenRequired)
    .response_checksum_validation(ResponseChecksumValidation::WhenRequired)
    .build();
```

**Rejected alternatives.**

- **`object_store` / `rust-s3`** — lighter, but neither exposes the checksum-negotiation and
  addressing controls that non-AWS conformance actually turns on. The reason to accept
  `aws-sdk-s3`'s weight is precisely that it has the knobs this risk requires.
- **Writing our own S3 client** — SigV4 is not a thing to reimplement.

**Residual, honestly stated.** Source inspection tells us the knobs exist and the defaults
are wrong. It does not tell us that RGW and RustFS accept our multipart sequencing,
conditional headers (`If-Match` / `If-None-Match` on PUT), listing semantics
(`ListObjectsV2` continuation tokens, delimiter and prefix handling), or that their error
*shapes* deserialize into the SDK's typed errors. Those four diverge the most and all need
execution.

**Executable confirmation.** Test ID `storage/s3-compat-conformance`: one conformance suite
run against each backend via Testcontainers — multipart upload/abort/complete, conditional
PUT, paginated listing with delimiters, and typed-error mapping for `NoSuchKey`,
`PreconditionFailed` and `BucketAlreadyOwnedByYou`. Storage wrapper: **Forge**. Suite:
**Ledger**. Execution: **Crucible**. §24.2 lists this conformance explicitly as Phase 0
work. Its cluster-only residue is risk R4 in §7.

---

## 5. Askama

Approved at **0.16.1**. It escapes HTML by default for `.html` templates, which is what
makes it right for the report renderer: §25's `report/xss-network-corpus` exists because
scanner output is attacker-influenced text that lands in a customer-facing report, and
§16.4 requires compensation for non-contextual escaping.

**Review rule.** `|safe` in a report template is a required change **unless the PR explains,
in the diff, why the value cannot carry attacker-controlled content.** Default-escaping is
only a defence if nobody opts out casually. Attribute, URL and script contexts still need
the per-context treatment §16.4 calls for; askama's default escaper is HTML-body-shaped and
is not a substitute for that.

`minijinja` is not adopted. Compile-time template checking is worth more here than runtime
template flexibility, and nothing in §16 needs templates that are not known at build time.

---

## 6. secureCodeBox pin and CRD type generation — §27 item 5 / §21.3

§21.3 says *"secureCodeBox 5.8.0 is published upstream … `[OPEN — eng]` Confirm the actual
selected release, Harbor artifacts, node Kubernetes compatibility, architectures, and SCB
CRD/hook behavior. Do not treat upstream release existence as proof of installation."* This
section selects the release and specifies type generation. It does **not** claim anything is
installed; Harbor mirroring and cluster compatibility are cluster work and appear as risk R6
in §7. The hook/parser *behaviour* half is decided in
[ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) §3.

### 6.1 Pinned release

**secureCodeBox v5.9.0**, released 2026-09-15. Images resolved from Docker Hub on
2026-10-01 — the charts reference `docker.io/securecodebox/*`, not ghcr.io
(`ghcr.io/securecodebox/operator` returns 404, which is worth knowing before someone writes
a mirror rule against the wrong registry):

| Image | Tag | Digest |
|---|---|---|
| `docker.io/securecodebox/operator` | `5.9.0` | `sha256:cae306050b736b7f7f9b23ba1579c61f14a4a2e561d9e9206c310a4ce12f4bb7` |
| `docker.io/securecodebox/lurker` | `5.9.0` | `sha256:eeb51600216d69a1543d88a92c36722be63d590a6843bcb0daaa96db324bf7e3` |

Both manifest lists advertise `linux/amd64` **and `linux/arm64`**, verified by reading the
manifest index platform entries. This was a gating check: Aether is arm64, and an
amd64-only operator would have forced emulation.

Rollback reference: v5.8.0 (2026-07-07), operator
`sha256:c14bd5c550e3edcc93e43fd59062962fb7bf11060703061aa7c8d9d2c9a55036`, lurker
`sha256:4278663652d62f54c8c09e78519b94e2db61c8083ac9bc7efc259210c06d651d`, also
arm64-capable.

### 6.2 Why v5.9.0 and not v5.8.0

v5.8.0 was the initial preference on maturity grounds — three months of field time versus
two weeks — on a suspicion that v5.9.0's newly bundled Garage object store added
infrastructure surface we did not want. Reading the charts instead of assuming reversed
that:

- v5.8.0's operator chart bundles **MinIO** (`docker.io/minio/minio`, `minio.enabled`).
- v5.9.0 **replaces MinIO with Garage** (`docker.io/dxflrs/garage:v2.3.0`, `garage.enabled`,
  default `true`), and the values file states an existing `minio.enabled: false` continues
  to be honoured and disables Garage too.

So v5.9.0 does not *add* a bundled object store, it *swaps* one, and it is disableable by
exactly the same one-line mechanism. The maturity argument did not survive contact with the
facts, and v5.9.0 brings current CRDs. **We set `garage.enabled: false`** and point
secureCodeBox at the object store chosen in §4.4. VulcanFlow does not run two object stores.

### 6.3 How the CRD Rust types are generated and committed

**Generated, committed, and CI-verified — not generated at build time.** A `build.rs` that
reaches for CRDs makes builds non-reproducible and non-offline, which §21.3's reproducible
arm64 requirement forbids.

- **Tool:** `kopium` 0.24.1, the kube-rs project's CRD-to-Rust generator. Pinned, installed
  with `cargo install --locked kopium@0.24.1`.
- **Source of truth:** the nine CRD manifests at tag `v5.9.0`, path `operator/crds/` —
  `execution.securecodebox.io_scans.yaml`, `_scheduledscans.yaml`, `_scantypes.yaml`,
  `_clusterscantypes.yaml`, `_parsedefinitions.yaml`, `_clusterparsedefinitions.yaml`,
  `_scancompletionhooks.yaml`, `_clusterscancompletionhooks.yaml`, and
  `cascading.securecodebox.io_cascadingrules.yaml`.
- **Output:** committed Rust in a dedicated crate, one module per CRD, each file carrying a
  generated-code header naming the kopium version and the secureCodeBox tag it came from.
  `vf-translator` (§7.4) consumes this crate.
- **Drift gate:** a CI job regenerates and fails if the result differs from what is
  committed. A silently stale CRD type is a deserialization failure at runtime against a
  live cluster — the worst place to find it.
- **Owner:** **Kiln**, as a child issue. The generation is code; the pin and the method are
  this decision.

**All nine CRD files differ in content between v5.8.0 and v5.9.0** (compared by blob SHA),
so the generated types are release-specific and must be regenerated on every secureCodeBox
bump. This is not a one-time task.

**Revisit trigger.** Any secureCodeBox minor or patch bump: regenerate types, re-resolve
digests, re-verify arm64. A release that changes a CRD group or version is a compatibility
review, not a silent bump.

---

## 7. Phase 1 risks that only a real cluster can settle

Recorded per §24.4. **None of these justifies standing up a cluster now.** Testcontainers
answers most of the sqlx and S3 questions and all of the admission-handler questions; what
is left is listed here as named risk with an owner, which is the honest alternative to
building infrastructure to dodge it. The `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is CEO's call (tracked separately), not a consequence
of this list.

| # | Risk | Why Testcontainers cannot settle it | Owner |
|---|---|---|---|
| R1 | Admission webhook TLS: certificate issuance, rotation, and `caBundle` injection into the webhook configuration | Needs a real API server calling us over TLS with a trust chain it accepted | Kiln |
| R2 | Admission ordering and failure policy under real load — `failurePolicy`, timeout behaviour, reinvocation of mutating webhooks | Emergent property of the API server's admission chain | Kiln |
| R3 | RLS enforcement through PgBouncer at realistic concurrency, including the `SET LOCAL` property of §4.3 under connection churn | Testcontainers proves the mechanism; it does not reproduce production churn | Forge + Ledger |
| R4 | Ceph RGW conformance against the real Aether RGW deployment — its version, tuning and bucket policy — as distinct from a containerised RGW | A containerised RGW is a different deployment from Aether's | Forge |
| R5 | arm64 build reproducibility end to end on Aether nodes | Needs the real build and runtime platform | Crucible |
| R6 | secureCodeBox v5.9.0 operator behaviour against the cluster's actual Kubernetes minor, with `garage.enabled: false` and an external object store; Harbor mirroring of the pinned digests | Operator / API-server interaction | Kiln |

**Blocking status.** §24.4's gate on "approval of the Rust crate set used on the execution
path" is cleared by §3. The executable confirmations in §4 are required Phase 1 work, listed
as test IDs with named owners; they are **not** preconditions for starting Phase 1, because
every one of them is a test that should exist and fail before the code that makes it pass.
§23.2 is explicit that a named test which exists and fails is the correct early state.

---

## 8. What would make me revisit this document as a whole

- A pinned crate turning out to be unmaintained or carrying an unpatched advisory —
  `build/rust-supply-chain` and cargo-deny are the detectors.
- `api/openapi-3_1-conformance` failing in a way that is utoipa's fault rather than our
  annotations' — that reopens §4.1 and puts a hand-written spec back on the table.
- PgBouncer prepared-statement tracking misbehaving under load — fallback in §4.3.
- RGW or RustFS failing `storage/s3-compat-conformance` in a way `aws-sdk-s3` cannot be
  configured around — that reopens the client choice in §4.4.
- A secureCodeBox release that changes a CRD group or version — §6.3 regeneration plus a
  compatibility review.
- Stable Rust raising its MSRV past what a pinned crate supports.

---

## 9. Provenance

Every version, digest, default and quoted sentence above was read on **2026-10-01** from:
the crates.io sparse index and API; `raw.githubusercontent.com` at the exact tags named; the
GitHub contents API at tag `v5.9.0`; the Docker Hub registry manifest API; and
`static.rust-lang.org/dist/channel-rust-stable.toml`. Where a claim rests on source
inspection rather than execution, §4 says so at the point of the claim and names the test
that will execute it.
