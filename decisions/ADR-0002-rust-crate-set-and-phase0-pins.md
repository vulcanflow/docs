# ADR-0002 — Approved Rust crate set, toolchain pin, and secureCodeBox pin

| | |
|---|---|
| **Status** | Accepted — amended **A1**, **A3** |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | TDD **§27 item 17** |
| **Also settles** | the engineering half of **§27 item 5** and the `[OPEN — eng]` secureCodeBox paragraph in **§21.3** (selected release, digests, arm64, CRD type generation) |
| **Does not close** | §27 items 16a, 18 (see [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md)), 19, 20, 21 (see [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md)) |
| **Supersedes** | the `[PROPOSED]` crate table in TDD §2.5.2 |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §2.5, §21.3, §24.2, §24.4, §25 |
| **Amendments** | **A1** (2026-10-01) — adds §3.6: crypto, TLS and encoding pins. **A2** — *reserved, not yet on `main`*: the §3.7 pin corrections and OIDC pins bundled into the closed PR docs#23. **A3** (2026-10-01) — adds the two identifier columns and §7.1–7.2 to the §7 risk table. See §10. |
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

### 3.6 Crypto, TLS, and encoding — added by amendment A1

These were missing from §3.1–§3.2, and the gap was load-bearing: ADR-0003 §4.4 decided that
Stripe and Lago webhook signatures are verified in-house with RustCrypto, and described those
crates as *"already pinned in §2.5.2"*. That is wrong in one word. §2.5.2 **names** rustls,
`hmac`, `sha2` and `rand`; it pins nothing — the whole table is `[PROPOSED]`, which is the
thing this document exists to replace. Pins belong here, so here they are.

| Crate | Pin | Role | Notes |
|---|---|---|---|
| `hmac` | 0.13.0 | HMAC-SHA256 for webhook signature verification (ADR-0003 §4.4) and the `vf-hook-notify` completion-hook signing in §9 | Verify with `Mac::verify_slice`, **never** by comparing the decoded hex/base64 to a recomputed string — comparison must be constant-time, and `verify_slice` is the constant-time path. A `==` on signature bytes is a required change at review. |
| `sha2` | 0.11.0 | SHA-256 | |
| `rand` | 0.10.3 | OS CSPRNG — domain-verification challenge tokens (§5.3), idempotency nonces | `OsRng` only. No seeded or user-space PRNG on any security path; a predictable verification challenge defeats the authorization boundary the product rests on. |
| `base64` | 0.23.1 | Lago signature decoding, presigned-URL handling | Lago emits **standard base64 with padding** — `engine::general_purpose::STANDARD`, not `URL_SAFE`, not `*_NO_PAD`. ADR-0003 §4.4 read this from upstream source. |
| `hex` | 0.4.3 | Stripe `v1=` signature decoding, artifact checksum rendering | |
| `rustls` | 0.23.45 | TLS | Pinned **directly** in the workspace, even though every consumer pulls it transitively, so that one version and **one** crypto provider resolve. See the provider note below. |

#### Resolution cross-check (performed, not assumed)

Declared ranges read from the crates.io API on 2026-10-01:

- `hmac 0.13.0` → `digest ^0.11.2`; `sha2 0.11.0` → `digest ^0.11` ✓ one `digest` generation,
  so the two resolve as a unit. The previous generation (`hmac 0.12` / `sha2 0.10` /
  `digest 0.10`) would not.
- **`aws-sdk-s3 1.151.0` itself declares `hmac ^0.13` and `sha2 ^0.11`.** This is the strongest
  evidence in the amendment: the crate already pinned in §3.2 brings exactly this generation
  into the tree, so pinning 0.12/0.10 here would compile two RustCrypto stacks side by side
  and give us two SHA-256 implementations in one binary.
- `reqwest 0.13.5` → `base64 ^0.23` ✓ matches the `base64 0.23.1` pin; → `rustls ^0.23.4` ✓
  matches the `rustls 0.23.45` pin.

#### The rustls crypto provider must be chosen once, and installed explicitly

`rustls 0.23.45` declares **both** providers optional: `aws-lc-rs ^1.18` and `ring ^0.17`.
Feature unification across dependents is what picks them, and if any two dependents enable
different providers both get compiled; rustls then cannot determine a process-level default
and panics on the first TLS handshake rather than at build time. We reach rustls through at
least `reqwest`, `sqlx`, `kube` and `aws-sdk-s3`, so this is not a hypothetical.

**Decision:** one provider workspace-wide — **`aws-lc-rs`**, rustls 0.23's own default, because
it is the path upstream tests and because `ring` would have to be opted into deliberately by
every dependent. And each binary calls
`rustls::crypto::aws_lc_rs::default_provider().install_default()` once in `main`, before any
client is constructed. Installing explicitly means the behaviour is ours rather than a
property of feature resolution, and a provider collision surfaces as a startup error on the
line that caused it.

**The limit, stated:** this is read from `rustls 0.23.45`'s declared features, not from a built
dependency tree. What it does not tell me is whether our actual tree pulls `ring` in anyway
through something I have not enumerated.

**Executable confirmation:** once the workspace exists, `cargo tree -i ring` must come back
empty and `cargo tree -d` must show no duplicated `rustls`, `hmac`, `sha2` or `digest`.
Wire it into the workspace CI gate as part of `build/rust-supply-chain`. Owner **Crucible** to
run it; **Forge** to make it pass. Tracked on **VUL-6**, not deferred to a phase.

#### What this amendment deliberately does not pin

A1's scope is the crypto and encoding gap ADR-0003 opened. §2.5.2 names other crates —
`pgvector`, `clickhouse`, `redis`/`fred`, `idna`/`url`, a Public Suffix List crate, a cron
crate, `jsonschema`, `ammonia`, `chromiumoxide`, `wasm-bindgen` — that stay unpinned and stay
off the approved set until the phase that needs them starts. Each gets pinned by amendment, at
the same evidence bar as §3.

One of those is closer than the rest and should not be allowed to drift: **`jsonwebtoken` and
`openidconnect`** (§4.1 OIDC / JWKS). The walking skeleton in §24.1 terminates an
authenticated request, so these are Phase 1, not later. Owner **Atlas**; trigger is the first
`vf-api` PR that validates a bearer token, and that PR should not be the place the versions get
decided.

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

| # | Risk | Why Testcontainers cannot settle it | **§25 identifiers a cluster would strengthen** | **Confirmations whose *assertion* a cluster would not strengthen** | Owner |
|---|---|---|---|---|---|
| R1 | Admission webhook TLS: certificate issuance, rotation, and `caBundle` injection into the webhook configuration | Needs a real API server calling us over TLS with a trust chain it accepted | `authz/start-barrier-all-paths`, `execution/gates-before-start` | `admission/cascade-gate-delete-oldobject`, `admission/workload-gate-dryrun` — **ADR-derived**, settled in full without a cluster (§4.2) | Kiln |
| R2 | Admission ordering and failure policy under real load — `failurePolicy`, timeout behaviour, reinvocation of mutating webhooks | Emergent property of the API server's admission chain | `authz/start-barrier-all-paths`, `execution/gates-before-start` | `admission/cascade-gate-delete-oldobject`, `admission/workload-gate-dryrun` — **ADR-derived**, settled in full without a cluster (§4.2) | Kiln |
| R3 | RLS enforcement through PgBouncer at realistic concurrency, including the `SET LOCAL` property of §4.3 under connection churn | Testcontainers proves the mechanism; it does not reproduce production churn | `isolation/all-stores`, `isolation/background-queries` | `db/tenanttx-set-local-isolation`, `db/pgbouncer-transaction-pooling-prepared` — **ADR-derived**, settled in full without a cluster (§4.3). Neither is weakened by R3 | Forge + Ledger |
| R4 | Ceph RGW conformance against the real Aether RGW deployment — its version, tuning and bucket policy — as distinct from a containerised RGW | A containerised RGW is a different deployment from Aether's | `isolation/all-stores` (object-store half), `deletion/all-stores-restore` (object-store half; Phase 4) | `storage/s3-compat-conformance` — **ADR-derived**. Its **assertion** is not strengthened by a cluster, which is why it belongs in this column. Its **deployment coverage** is: R4 is the only row in this table where a cluster adds anything at all to an ADR-derived confirmation, and what it adds is a second backend, not a better result. Ledger entry is a PASS whose scope names the backend — see §7.1(c) | Forge |
| R5 | arm64 build reproducibility end to end on Aether nodes | Needs the real build and runtime platform | `build/rust-supply-chain` (reproducibility half), `perf/service-baseline` | The cargo-deny / `cargo audit` / SBOM / `#![forbid(unsafe_code)]` half of `build/rust-supply-chain` — CI-only, no cluster. No ADR-derived identifier depends on R5 | Crucible |
| R6 | secureCodeBox v5.9.0 operator behaviour against the cluster's actual Kubernetes minor, with `garage.enabled: false` and an external object store; Harbor mirroring of the pinned digests | Operator / API-server interaction | `execution/scan-identity`, `supply-chain/check-catalog` (Phase 2), and the **pool half** of `authz/start-barrier-all-paths` and `execution/gates-before-start` — §27 item 5 names the pool start barrier explicitly | `scb/hook-invocation-contract` and `scb/parser-contract-conformance` (ADR-0003 §3.4) and the §6.3 CRD-codegen drift gate — all **ADR-derived** and all cluster-free | Kiln |

### 7.1 How to read the last two columns

This section is meant to be sufficient on its own. Someone pricing a cluster reads §7 and
nothing else, so the edges that §4.2 and §4.3 state in prose are restated here rather than
left to be found.

**The right-hand column's heading says *assertion* deliberately, and the word is doing work.**
Five of the six rows would read the same without it. R4 would not: a cluster adds nothing to what
`storage/s3-compat-conformance` *asserts*, and it does add a second backend to what that assertion
*covers* (c). Heading the column "confirmations a cluster would not strengthen" would make the R4 cell
contradict the column it sits in, and the distinction between an unchanged assertion and incomplete
deployment coverage is the whole content of (c).

**(a) Two kinds of identifier appear above, and they are not interchangeable.**

- **§25 identifiers** are the 45 in the TDD's traceability matrix. They are the product's
  acceptance surface. Only these appear in the "would strengthen" column.
- **ADR-derived confirmations** are the test IDs this document and ADR-0003 invented as the
  condition of approving a crate — `api/openapi-3_1-conformance`,
  `admission/cascade-gate-delete-oldobject`, `admission/workload-gate-dryrun`,
  `db/pgbouncer-transaction-pooling-prepared`, `db/tenanttx-set-local-isolation`,
  `storage/s3-compat-conformance` (this document, §4) and `scb/hook-invocation-contract`,
  `scb/parser-contract-conformance` (ADR-0003 §3.3–3.4). **These are not §25 identifiers**
  and must never be counted into the 45. They sit **upstream** of §25: §7.2 names the feed
  edge for each one, and for two of the eight that edge is **None**. A confirmation can be a
  required test and still assert something §25 has no row about — which is why §7.2 exists as
  a table of edges rather than as an assertion that every confirmation has one.

**(b) "Strengthen" is the right verb; "degrade" is not.** With one exception noted in (c),
every ADR-derived confirmation above runs green, with its full intended assertion, on a
laptop with Testcontainers. A cluster does not rescue a weakened test — it closes the gap
between *"our handler behaves correctly when handed the right input"* and *"the real
platform hands it that input, under load, in production shape."* That gap is a property of
the **§25** identifiers, which assert the system behaviour, not of the ADR-derived
confirmations, which assert a unit of our code against a read upstream contract.

**(c) R4 is the single exception.** `storage/s3-compat-conformance` is a conformance suite
run against a *backend*; the containerised RGW it runs against in Phase 1 is a real backend
and the result is real. What a cluster adds is a second, different backend — Aether's own
RGW deployment, with its version, tuning and bucket policy. So the assertion is unchanged
and its coverage is incomplete, and the honest ledger entry is a PASS whose scope names the
backend it ran against.

**(d) What this table is not.** It does not say that a §25 identifier in the middle column
cannot be turned green in Phase 1. `authz/start-barrier-all-paths` and
`execution/gates-before-start` are Phase 1 Ledger work and must go green in Phase 1,
asserted against replayed `AdmissionReview` JSON through the real handler. The middle column
says what a cluster would *add* to that green, and the right-hand column says where it would
add nothing **to the assertion** — which, for R4 alone, is not the same as adding nothing at all (c).

### 7.2 The feed edges, restated from §4 so §7 stands alone

| ADR-derived confirmation | Stated in | Feeds these §25 identifiers |
|---|---|---|
| `api/openapi-3_1-conformance` | §4.1 | None. It confirms a crate's output shape; §25 has no OpenAPI-version row. No cluster risk bears on it |
| `admission/cascade-gate-delete-oldobject` | §4.2 | `authz/start-barrier-all-paths`, `execution/gates-before-start` |
| `admission/workload-gate-dryrun` | §4.2 | `authz/start-barrier-all-paths`, `execution/gates-before-start` |
| `db/pgbouncer-transaction-pooling-prepared` | §4.3 | `isolation/background-queries`, `isolation/all-stores` |
| `db/tenanttx-set-local-isolation` | §4.3 | `isolation/background-queries`, `isolation/all-stores` |
| `storage/s3-compat-conformance` | §4.4 | `isolation/all-stores` (object-store half); `deletion/all-stores-restore` in Phase 4 |
| `scb/hook-invocation-contract` | ADR-0003 §3.4 | **None.** Its assertions are argv positions, the required environment and ReadOnly two-URL tolerance — not duplicate delivery. See the note below |
| `scb/parser-contract-conformance` | ADR-0003 §3.4 | `execution/scan-identity` — stated there explicitly |

**The direction of the edge matters.** An ADR-derived confirmation going green does not make
the §25 identifier it feeds green; it removes one way for that identifier to fail. Counting a
feed edge as coverage of the §25 row is the error this table exists to prevent.

**Two rows read "None", and the second one is the instructive case.**
`api/openapi-3_1-conformance` is the easy one: it confirms a crate's output shape and §25 has
no row about OpenAPI versions. `scb/hook-invocation-contract` is the trap. ADR-0003 §3.4 states
an idempotency *obligation* — "the scan fingerprint is the deduplication key, not the hook
invocation" — in the same subsection that names the test, and it reads as though the test
carries it. It does not. The confirmation invokes the built binary with `SCAN_NAME` and
`NAMESPACE` set against a local HTTP server and asserts three things: that it reads the argv
positions §3.3 names, that it exits non-zero with a clear message when `SCAN_NAME` is absent,
and that two URLs do not make it index past the end. **All three pass against a `vf-ingest`
that persists a replayed artifact twice.** §8.4 idempotency is asserted by the §25 identifier
`findings/replayed-artifact` on its own, which is complete without any ADR-derived feed; an
edge drawn from this confirmation to that identifier would claim coverage that no test
provides, which is precisely the error the paragraph above names. Writing an obligation into
this table as if it were an assertion is how that error gets made.

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
- A dependency pulling in a second rustls crypto provider, which `cargo tree -i ring` and
  `cargo tree -d` are the detectors for — §3.6.
- Stable Rust raising its MSRV past what a pinned crate supports.

---

## 9. Provenance

Every version, digest, default and quoted sentence above was read on **2026-10-01** from:
the crates.io sparse index and API; `raw.githubusercontent.com` at the exact tags named; the
GitHub contents API at tag `v5.9.0`; the Docker Hub registry manifest API; and
`static.rust-lang.org/dist/channel-rust-stable.toml`. Where a claim rests on source
inspection rather than execution, §4 says so at the point of the claim and names the test
that will execute it.

---

## 10. Amendment history

Amendments are recorded here rather than silently edited in, so that a reader who reviewed an
earlier revision can see exactly what moved.

### A1 — 2026-10-01 — crypto, TLS and encoding pins (§3.6)

**Raised by CEO** in review of this document on VUL-3: ADR-0003 §4.4 asserts that `hmac` and
`sha2` are "already pinned in §2.5.2", and §3 of this document did not list them.

The assertion was wrong in both directions worth recording. §2.5.2 names those crates but pins
nothing — it is the `[PROPOSED]` table this document supersedes, so "pinned in §2.5.2" could not
have been true of anything. And §3 was genuinely incomplete: a decision to write signature
verification in-house (ADR-0003 §4.4) is a decision to take a direct dependency, and a direct
dependency without a pin is the exact debt §27 item 17 exists to clear.

A1 adds §3.6 with pins for `hmac`, `sha2`, `rand`, `base64`, `hex` and `rustls`, a resolution
cross-check against the already-pinned `aws-sdk-s3` and `reqwest`, and a decision on the rustls
crypto provider that the pin exposed. ADR-0003 §4.4's cross-reference is corrected in the same
change.

**It changes no acceptance criterion of VUL-3 and reverses nothing.** §27 item 17 stays closed;
§3.6 is an addition to the set it approved, not a revision of it. CEO's alternative — pinning
these in the workspace when VUL-6 lands and leaving the record alone — would have worked, and I
chose the amendment because a pin that exists only in a `Cargo.toml` has no recorded reason, and
the provider question would then have been settled by whoever hit the panic.

### A2 — reserved, not on `main`

**A2 is the §3.7 pin corrections and OIDC pins.** It was bundled into PR docs#23 alongside
ADR-0004, docs#23 was closed unmerged, and ADR-0004 came back on its own (docs#27) without it.
The number is therefore **reserved, not skipped**, and this document does not carry A2 yet.
`platform#1`'s workspace `Cargo.toml` names §3.7 as its pin authority, so A2 must be re-filed —
with every pin re-derived against crates.io rather than copied from the orphaned head — before
`platform#1` merges. Tracked in `decisions/README.md` under "Still open".

### A3 — 2026-10-01 — the §7 risk table now names the identifiers each risk bears on (§7, §7.1, §7.2)

**Raised by me** in review of the open-decision register (docs#24). The register's §1.3 mapped
ADR-0002 §7's risks R1–R6 onto test identifiers and got it wrong in both directions: six
ADR-derived confirmations were presented as §25 identifiers, two tests that are settled in full
without a cluster (`db/tenanttx-set-local-isolation`,
`db/pgbouncer-transaction-pooling-prepared`) were listed as cluster-weakened, and
`execution/gates-before-start` and `isolation/all-stores` were omitted entirely.

**The register was not the defect.** §4.2 ends with *"These feed `authz/start-barrier-all-paths`
and `execution/gates-before-start` in §25"* and §4.3 ends with the equivalent sentence for
`isolation/background-queries` and `isolation/all-stores`. Those two sentences were the only
place the feed edges existed, and **§7 named neither set**. Anyone pricing a cluster reads §7;
§7 did not carry enough to answer the question it is about, so the reader reconstructed the
mapping and reconstructed it wrong. A record that requires a correct inference from a distant
section is an incomplete record, and the reader who makes the inference is not the one at fault.

A3 adds two columns to the §7 table — *"§25 identifiers a cluster would strengthen"* and
*"Confirmations a cluster would **not** strengthen"* — plus **§7.1**, which states the §25 /
ADR-derived distinction and why "strengthen" rather than "degrade" is the accurate verb, and
**§7.2**, which restates every feed edge from §4 and ADR-0003 §3.4 in one table so §7 is
sufficient on its own.

**It changes no decision and no assertion.** R1–R6 are unchanged, their owners are unchanged,
and no test's behaviour moves. What changes is that §7 now answers, for each risk, which §25
identifiers a cluster would strengthen and which confirmations it would not — without the reader
having to read §4.

**One substantive judgement is recorded rather than assumed.** R4 is the only risk where a
cluster bears on an ADR-derived confirmation: `storage/s3-compat-conformance` runs against a
*backend*, and Aether's RGW is a second backend rather than a better one. §7.1(c) records that
the assertion is unweakened and the deployment coverage is incomplete, which makes the correct
ledger entry a PASS whose scope names the backend it ran against — not a silent green.

**One edge in the first draft of this amendment did not exist, and the correction is kept here
rather than quietly applied.** The draft §7.2 row for `scb/hook-invocation-contract` read
*"`findings/replayed-artifact` (§8.4 idempotency, via the scan fingerprint)"*. CodeRabbit's
review of docs#28 rejected it, and it was right: ADR-0003 §3.4 states the idempotency
obligation next to the test but does not put it inside the test, whose assertions are argv
positions, required environment and two-URL tolerance. The row now reads **None**, with the
reasoning under §7.2. The error is worth recording because it is the mirror image of the one
A3 was filed to fix — §7 omitted edges that §4 states, and the first attempt to add them
invented one §4 does not. **A missing edge and an invented edge are the same defect**, and
only reading the named test's assertions distinguishes them.

**And the R4 judgement above was true in §7.1(c) and contradicted by the column it sits in.** The
right-hand column was headed *"Confirmations a cluster would not strengthen"*, and the R4 cell says a
cluster does add Aether's RGW deployment coverage. CodeRabbit's review of docs#28 at `98411fb` caught
it. The heading now reads *"Confirmations whose **assertion** a cluster would not strengthen"*, which
is true of all six rows, and §7.1 says why that word is load-bearing: five rows would read the same
without it and R4 would not. The R4 cell and §7.1(d) are reworded to the same distinction. This is the
third correction in this amendment that was a **presentation** defect rather than a wrong claim, and
all three were in §7 — the section whose entire purpose is to be sufficient on its own. A table whose
heading is false for one of its cells is not sufficient on its own, however correct the prose beneath
it.

**Revisit trigger for A3 specifically.** Any new executable confirmation added to §4, or any new
risk added to §7, must land with its §7.2 feed edge in the same change. A confirmation whose
feed edge is stated only in §4 reproduces the exact defect this amendment fixes. **None** is a
legitimate value for that edge and is not a gap to be filled; what it requires is the same
thing every other value requires — that the edge be read off the test's stated assertions and
not off the prose surrounding them.
