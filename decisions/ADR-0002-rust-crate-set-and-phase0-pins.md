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
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §2.5, §21.3, §24.2, §24.4, §25 |
| **Amendments** | **A1** (2026-10-01) — adds §3.6: crypto, TLS and encoding pins. **A2** (2026-10-01) — adds §3.7: feature-name corrections, the `sqlx`/`kube` provider fixes, `k8s-openapi v1_32`, `jsonwebtoken` pinned and `openidconnect` rejected, CI tool pins, `utoipa-swagger-ui` withdrawn, and A1 assertion 2 restated. See §10. |
| **Issue** | VUL-3 (A1 on VUL-3; A2 on VUL-39, from the VUL-6 workspace build) |

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

### 3.7 Corrections, feature names, and the OIDC pins — added by amendment A2

§3.1–§3.6 pinned versions. Writing the manifests on VUL-6 found that **six of those pins
name a feature that does not exist, or names one that means the opposite of what §3 said it
meant**, and that two of the crates §3.6 deferred cannot be adopted at all. A version pin
with a wrong feature is not a weaker pin, it is a different dependency graph — on the AWS
SDK it is the difference between rustls 0.23 with `aws-lc-rs` and rustls 0.21 with `ring`.

Four of these were found and escalated by **Forge** on VUL-6 and are recorded here with the
evidence re-derived independently; two more (`sqlx`, `kube`) are corrections Forge applied
without flagging, and they are the two that matter most, so they are written down rather
than left in a manifest comment.

**Everything in this amendment is resolved against the crates.io index on 2026-10-01 and
cross-checked against the `Cargo.lock` committed in `vulcanflow/platform#1`.** As in §4, no
crate was compiled: the sandbox has `cargo` but no C linker, so `cargo metadata` and
`cargo tree` are evidence and `cargo check` is not. §25 `build/rust-supply-chain` is where
that gap closes; it is Crucible's to run (VUL-38).

#### 3.7.1 The corrected feature sets

| Crate | §3 said | Corrected to | Why the original is wrong |
|---|---|---|---|
| `reqwest` 0.13.5 | `rustls-tls` | **`rustls`** | The feature was renamed in 0.13. `rustls-tls` does not exist; the build fails. `rustls` expands to `__rustls-aws-lc-rs` + `rustls-platform-verifier`, so it selects the §3.6 provider — see §3.7.3 on the verifier. |
| `aws-sdk-s3` 1.151.0 | `rustls` (from §3.5 "rustls everywhere") | **`default-https-client`**, `rt-tokio`, `http-1x`; **not** `sigv4a` | `aws-sdk-s3/rustls` and `aws-sdk-s3/legacy-https-client` both expand to `aws-smithy-runtime/tls-rustls`, which is `aws-smithy-http-client/legacy-rustls-ring` + `connector-hyper-0-14-x`. The feature named `rustls` **is** the legacy hyper-0.14 client, and its provider is `ring` — the crate name says so. It breaks both A1 assertions at once. `default-https-client` is `aws-smithy-http-client/rustls-aws-lc`: hyper 1.x, rustls 0.23, `aws-lc-rs`. |
| `aws-config` 1.12.0 | — | `rt-tokio`, **`default-https-client`** | Same trap: `aws-config/rustls` expands to the same legacy client. |
| `sqlx` 0.9.0 | `tls-rustls` | **`tls-rustls-aws-lc-rs`** | **This is the correction worth the most.** In sqlx 0.9, `tls-rustls` = `tls-rustls-ring` = `tls-rustls-ring-webpki`, which enables `rustls/ring`. §3.2 as written pins `ring` into every binary that opens a database connection, which is every binary. |
| `sqlx` 0.9.0 | `runtime-tokio`, `tls-rustls`, `postgres`, `uuid`, `macros` | as corrected, **plus `json`**, **minus `migrate`** | `json` is needed for `JSONB` columns in §6.3. `migrate` is removed from the workspace pin — see §3.7.2. |
| `kube` 4.2.0 | `client`, `derive`, `runtime`, `admission`, `rustls-tls` (`default-features = false`) | plus **`aws-lc-rs`**, `config`, `jsonpatch` | `kube`'s own default is `["client", "rustls-tls", "ring"]`. With `default-features = false` and no provider feature, `kube-client` gets rustls with no provider enabled by itself; with defaults it gets `ring`. Neither is what A1 decided, so the provider is named explicitly. `config` is needed for in-cluster and kubeconfig resolution; `jsonpatch` is what re-exports the `json-patch` already pinned in §3.2. |
| `testcontainers` | 0.28.0 | **0.27.3** | `testcontainers 0.28.0` + `testcontainers-modules 0.15.0` cannot resolve. 0.15.0 is the newest `testcontainers-modules` release and declares `testcontainers ^0.27.0`; there is no modules release for 0.28. 0.27.3 is the newest version satisfying both. **Revisit** when a `testcontainers-modules` release declaring `^0.28` publishes; raise both together. |

`k8s-openapi` keeps its `=0.28.0` pin and gains a decided version feature — §3.7.4.

#### 3.7.2 `sqlx/migrate` is not a workspace feature

Cargo unifies features across a workspace build, so a feature enabled by one member is
enabled for every member built alongside it. `sqlx/migrate` enables `sqlx-core/migrate`,
which enables `sqlx-core`'s **optional** `sha2 ^0.10` — the previous RustCrypto generation —
as a real, linked dependency of anything that uses `sqlx`.

§21.2 puts migrations in exactly one place: *"per-tenant schema migrations run as a resumable
fan-out job (a Rust binary using the §2.5.2 migration tool)"*. That is one binary, and it is
not one of the fourteen crates in PR #1 yet.

**Decision:** `migrate` is **not** in `[workspace.dependencies].sqlx`. The migration-job
crate — `vf-migrate`, to be created with the §21.2 work — adds it on top of the workspace
pin:

```toml
sqlx = { workspace = true, features = ["migrate"] }
```

so `sha2 0.10` is linked into that one binary and nothing else. The eleven service binaries
do not carry a second SHA-256 implementation in order to own a feature they never call.

#### 3.7.3 Two trust stores, and the one that will fail on Aether

The corrected feature sets produce **two different certificate-verification paths**, and
this is not cosmetic on a self-operated platform:

- `reqwest` → `rustls-platform-verifier` → the OS trust store. A CA mounted into the
  container image or the system bundle is trusted.
- `kube-client` → `rustls-platform-verifier` → same, plus the kubeconfig/in-cluster CA,
  which `kube` supplies explicitly.
- `sqlx` → `_tls-rustls-aws-lc-rs` → **`webpki-roots`**, a compiled-in set of *public* CA
  roots, and nothing else.

Aether is self-operated and CNPG issues Postgres server certificates from an internal CA.
A public-root-only verifier **cannot** verify them. The failure mode is a TLS handshake
rejection at first connection, which looks like a networking problem and is not one.

**Decision:** `vf-db` constructs `PgConnectOptions` with `ssl_mode(Require)` *and* an
explicit `ssl_root_cert` pointing at the mounted CA bundle. `webpki-roots` being compiled in
is then irrelevant, because the root set is supplied per connection. This is a **required
change at review** on the first `vf-db` PR, not a note: the default is a connection that
either fails closed (good) or is downgraded by someone reaching for `ssl_mode(Prefer)` to
make it work (the actual risk). `sqlx/tls-rustls-ring-native-roots` would use the system
store instead, and is rejected — it brings `ring`.

Carried to **VUL-26** (`vf-db` / `TenantTx`) as an acceptance criterion.

#### 3.7.4 `k8s-openapi`'s version feature — decided, not deferred

`k8s-openapi` requires exactly **one** version feature, and 0.28.0 offers `v1_32` through
`v1_36`. The TDD does not state Aether's control-plane minor, so Forge pinned `v1_34` as an
explicit placeholder and flagged it.

A placeholder in a `=`-pinned manifest is the kind of debt that gets discovered by a
production reconcile, so it is decided here instead.

**Decision: `v1_32`** — the floor of what 0.28.0 offers.

The reason is the same instinct as deny-by-default. `k8s-openapi`'s generated types for an
older minor remain valid against a newer API server; the risk runs the other way. Compiling
against the floor means `vf-operator` and `vf-admission` can only reference API surface that
every supported minor serves, so a field that exists on 1.36 and not on the real cluster
becomes a **compile** error rather than a reconcile that silently stops short. Raising the
floor is then a deliberate act with a recorded reason, which is what we want.

**The real risk this exposes, and it is not resolved by choosing a floor:** if Aether's
control plane is **older than 1.32**, then `k8s-openapi 0.28.0` — and therefore `kube 4.2.0`,
which declares `^0.28.0` — is the wrong pin outright, and §3.2 needs a different pair.
Confirming the minor is a CEO question (Aether is operated, not ours to inspect) and is
filed as such. It does **not** block the §24.1 skeleton: §24.2 puts the operator and
admission surfaces after the services, and nothing in Phase 1 reconciles a CRD.

#### 3.7.5 `openidconnect` is not adopted, and `jsonwebtoken` is pinned

§3.6 named these two as the closest unpinned pair, owner Atlas, trigger "the first `vf-api`
PR that validates a bearer token". Resolving them for this amendment found that one of them
cannot be adopted at all, which is exactly why the trigger should not have been that PR.

**`openidconnect` 4.0.1 — rejected.** Its *non-optional* dependencies are:

| Dependency | Conflict |
|---|---|
| `chrono ^0.4` | Banned by §3.5 and by `deny.toml`. Not a feature we can turn off. |
| `hmac ^0.12.1`, `sha2 ^0.10.6` | The previous RustCrypto generation, **non-optional**. A1's duplicate assertion exists for precisely this, and here it lands on a JWT **signature-verification** surface — the one place in the product where two SHA-256 implementations in one binary is a security question and not a size question. |
| `reqwest ^0.12` | We pin 0.13. A second `reqwest`, a second hyper stack, two TLS configurations. |
| `base64 ^0.21` | A third `base64` generation. |

There is no feature combination that avoids any of this. Adopting it would mean reversing
§3.5 and A1 to buy OIDC discovery.

**What we do instead.** The part of `openidconnect` the skeleton needs is small and is
better off in our own code: fetch the OIDC discovery document, fetch and cache JWKS, select
the key by `kid`, validate the token. That is `reqwest` + `jsonwebtoken` + a cache, and
writing it ourselves means the `iss`/`aud`/`exp`/`alg` checks are explicit, denied by
default, and property-testable in `vf-authz` — which §2.5.2 wants anyway. **`alg` comes from
the JWKS entry, never from the token header**; honouring the header's `alg` is the classic
JWT confusion bug and a required change at review.

**`jsonwebtoken` 11.1.0 — pinned**, `default-features = false`, features
**`["aws_lc_rs", "use_pem"]`**.

| | |
|---|---|
| Why 11.1.0 | It offers `aws_lc_rs` as a crypto backend. The alternative, `rust_crypto`, pulls `hmac 0.12`, `sha2 0.10`, `p256`, `rsa` and `ed25519-dalek` — the same old-generation problem as `openidconnect`, in a crate we do want. |
| Why `aws_lc_rs` | It is the provider A1 already chose for rustls, so one crypto implementation serves TLS and JWT verification. One backend to review, one to patch. |
| `use_pem` | Keycloak 26 JWKS keys arrive as JWK; PEM support is needed for the static-key path and for tests. |
| Recorded limit | This is read from declared features and dependencies, not from a compiled tree. **Confirmation owed:** the first `vf-api` token-validation PR must show `aws_lc_rs` covering the algorithm Keycloak 26 actually signs with (RS256 by default), and `cargo tree -i ring` still empty with `jsonwebtoken` in the graph. |
| Accepted noise | `jsonwebtoken` carries `getrandom ^0.2` and `rand ^0.8.5`. Neither is an A1 assertion and neither is on a signature surface — `rand 0.10`/`OsRng` remains the only generator on the challenge-token and nonce paths per §3.6. |

#### 3.7.6 CI tool pins

§3 pins what the workspace builds against and said nothing about the cargo subcommands CI
runs. An unpinned `cargo install` inside the gate that exists to close supply-chain holes is
a supply-chain hole. Forge pinned them in `ci/tool-versions.env` and asked for them to live
here; agreed.

| Tool | Pin | Role |
|---|---|---|
| `cargo-deny` | 0.20.2 | advisories, licences, bans, sources (§21.3) |
| `cargo-audit` | 0.22.2 | RUSTSEC advisories read from `Cargo.lock` directly |
| `cargo-cyclonedx` | 0.5.9 | CycloneDX SBOM per §21.3 |
| `cargo-auditable` | 0.7.6 | embeds the dependency list in the shipped binary, so an SBOM is recoverable from the artifact and not only from the build |

Installed with `--locked` so each tool's own lockfile is used. `ci/tool-versions.env` stays
as the machine-readable copy; this table is the decision.

#### 3.7.7 `utoipa-swagger-ui` is withdrawn from the approved set

§3.1 pinned `utoipa-swagger-ui 10.0.1`. Its build script **downloads the Swagger UI
distribution over the network** unless the `vendored` feature is on, and `vendored` pulls
`utoipa-swagger-ui-vendored ^0.2`, which §3 does not pin. A network fetch inside a build
cannot be reproducible, which is the §21.3 requirement the whole build gate exists to serve.

**Decision: drop it.** The deliverable in §1081 is the *published OpenAPI 3.1 document*, and
`utoipa` + `utoipa-axum` produce that; `vf-api` serves the JSON and the generated TypeScript
client is produced from the committed spec. A spec-rendering UI is a developer convenience
that any local tool renders from that same JSON without putting a network fetch in our
release path.

**Revisit** if an in-cluster spec UI is actually asked for. The answer then is `vendored`
plus a pin for `utoipa-swagger-ui-vendored` by amendment — not the current configuration.

#### 3.7.8 Accepted duplicates, named

A1's duplicate assertion covers `rustls`, `hmac`, `sha2` and `digest`. Resolving the real
graph produced two duplicates outside that set, and they are accepted here so they are not
mistaken for drift later:

- **`base64` 0.22.1 and 0.23.1.** 0.22 arrives from `k8s-openapi`, `kube-client`, `sqlx`,
  `tonic` and `pem`; 0.23 is our pin and `reqwest`'s. Accepted: `base64` is an encoding, it
  has no key material and no constant-time requirement, and the §3.6 decision that matters —
  Lago signatures are `STANDARD` with padding — is a decision about the *engine at the call
  site*, which is unaffected by a second copy of the crate existing.
- **`tower-http` 0.6.11 and 0.7.1.** 0.6 from `kube-client` and `reqwest`; 0.7 is our direct
  pin for the `vf-api` middleware stack. Accepted: no shared state, no security surface.

Both are in `deny.toml`'s `skip` list with the same reasons. Anything **not** listed there
is a prompt to look, which is the point of keeping the list short and specific.

#### 3.7.9 A1's second assertion, restated so it is achievable and still strict

A1 asked for `cargo tree -i ring` empty and `cargo tree -d` showing no duplicated `rustls`,
`hmac`, `sha2` or `digest`. On the real graph, `ring`, `rustls` and `hmac` hold. `sha2` and
`digest` do not, and Forge's gate reported them as findings behind a `STRICT_RUSTCRYPTO`
flag rather than narrowing the rule — correctly, because narrowing a security assertion is
not a coding agent's call.

Resolving it properly changes the answer. A1's stated worry was *"two SHA-256
implementations in one binary"*, and three of the four duplicate sources are not that:

| Source of the old generation | Is it in a shipped binary? |
|---|---|
| `sqlx-core 0.9.0` → `sha2 0.10` | **Only with `migrate`.** It is optional and `sqlx-core/migrate` is its only gate. Removed from the workspace pin by §3.7.2. |
| `sqlx-macros-core 0.9.0` → `sha2 0.10` | **No.** It is a *required* dependency, but `sqlx-macros` is a proc-macro crate: it runs on the build host at compile time and is never linked into the artifact. |
| `aws-sigv4` → `p256` → old generation | **No.** Only under `sigv4a`, excluded in §3.7.1. |
| `crc-fast 1.10.0` → `digest 0.10` | **Yes, genuinely.** `aws-smithy-checksums 0.65.0` depends on `crc-fast` non-optionally, and `crc-fast`'s `digest 0.10` comes in via `alloc` ← `std` ← its defaults. S3 needs the checksum path. Nothing we pin moves this. |

So the assertion is achievable for `sha2` and has exactly **one** irreducible exception for
`digest`. A1 assertion 2 is restated as:

> Evaluated **per shipped binary** (`cargo tree -p <bin> --edges normal,no-proc-macro`,
> default features), the graph must contain **no `ring`**, **no `chrono`**, and exactly one
> version each of **`rustls`**, **`hmac`** and **`sha2`**. `digest` may resolve to two
> versions **only** when the second is `0.10.x` reachable solely through
> `crc-fast` ← `aws-smithy-checksums`; the gate fails on `digest 0.10` arriving by any other
> path.

Three things changed and each is deliberate:

1. **Per shipped binary, not per workspace.** What ships is a binary built with its own
   feature set. A workspace-wide resolution unifies `vf-migrate`'s `migrate` into the
   assertion and reports a duplicate that is in no artifact.
2. **`no-proc-macro`.** A compile-time hash on the build host is not a second
   implementation in the binary. Counting it makes the gate wrong in the direction that gets
   gates deleted.
3. **The `digest` exception is asserted by path, not waived.** "Two versions are fine" would
   let a second RustCrypto stack in through any new dependency. "Two versions are fine iff
   the second one is reachable only here" still fails on the thing A1 was built to catch.

**`STRICT_RUSTCRYPTO` is deleted.** A supply-chain assertion with an environment variable
that turns it off is an assertion that is off. Owner **Forge** to implement
(`ci/tls-provider-assertions.sh`), **Crucible** to run, under §25
`build/rust-supply-chain`.

**Revisit** when `aws-smithy-checksums` moves to `digest 0.11`, or when `crc-fast` makes its
`digest` dependency optional. At that point the `digest` exception is deleted and the
assertion becomes uniform.

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

### A2 — 2026-10-01 — pin corrections, OIDC pins, and A1 assertion 2 restated (§3.7)

**Raised by the work, not by a reviewer.** Forge wrote the manifests on VUL-6 and found that
§3's pins do not all build: four named a feature that does not exist or whose meaning is the
opposite of what §3 assumed, and one pair cannot resolve together. Forge escalated those four
on VUL-39 rather than deciding them, and applied two further corrections (`sqlx`, `kube`)
inside the manifest without flagging them — those two are the ones that would otherwise have
been re-litigated, so A2 writes them down.

A2 adds §3.7. It corrects `reqwest`, `aws-sdk-s3`, `aws-config`, `sqlx`, `kube` and
`testcontainers`; removes `sqlx/migrate` from the workspace pin and scopes it to the
migration-job crate; decides `k8s-openapi`'s version feature as `v1_32` with the reason;
records the `webpki-roots`-versus-internal-CA consequence as a required change on `vf-db`;
closes §3.6's last open pair by pinning `jsonwebtoken 11.1.0` with `aws_lc_rs` and **rejecting
`openidconnect`**; moves the CI tool pins into the ADR; withdraws `utoipa-swagger-ui`; names
the two accepted duplicates outside A1's set; and restates A1's second assertion so it is
evaluated per shipped binary, excludes proc-macro edges, and carries one path-asserted
`digest` exception instead of an environment-variable escape hatch.

**What it reverses.** §3.1's `utoipa-swagger-ui` pin is withdrawn, and §3.5's "rustls
everywhere" is corrected from a feature name to a *provider* requirement — on the AWS SDK the
feature named `rustls` is the `ring` one. §3.6's `STRICT_RUSTCRYPTO` escape hatch (introduced
in the VUL-6 implementation, never in this document) is deleted. §27 item 17 stays closed:
A2 corrects how the approved set is spelled and adds two crates to it; it does not reopen
whether the set is approved.

**What it does not settle.** Whether Aether's control plane is at or above Kubernetes 1.32.
If it is below, §3.2's `kube 4.2.0` / `k8s-openapi =0.28.0` pair is the wrong pin and §3.2
needs a new resolution — not a different feature. CEO owns the answer; §24.2 puts the
Kubernetes surfaces after the services, so it does not block the §24.1 skeleton.
