# ADR-0003 — Streaming RPC, secureCodeBox parser/hook language, and billing clients

| | |
|---|---|
| **Status** | Accepted — amended **A1** |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | TDD **§27 items 19, 20 and 21** |
| **Amendments** | **A1** (2026-10-01) — three corrections, none of which reverses a decision: §3.2's custom-parser claim and §3.5's severity remedy (both factual, from re-reading the v5.9.0 scanner tree), and §3.3's mixed `argv` index bases (presentational). See §7. |
| **Depends on** | [ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md) — the crate set and the secureCodeBox v5.9.0 pin |
| **Does not close** | §27 items 16a, and the Product halves of items 2, 4 and 11 that bear on paid use |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §2.5.2, §2.5.3, §9, §17.5, §21.3, §24.2–24.4 |
| **Issue** | VUL-3 |

These three items are grouped into one record because each is a *language or client-shape*
question about an integration boundary, each is answered by reading the upstream contract,
and the probes were run together. They are decided independently below and can be revisited
independently.

---

## 1. Summary of the three decisions

| §27 item | Question | Decision |
|---|---|---|
| 19 | Typed streaming RPC beyond REST + SSE? | **No channel is adopted.** REST + SSE only, at GA and after. If post-GA SDK demand appears, the choice is pre-decided by the criteria in §2.4 — and the TDD's stated premise for ruling Connect out is now factually stale (§2.3). |
| 20 | SCB parser and hook language | **Parsers stay on the upstream JavaScript parser SDK**, including custom parsers. **Hooks are Rust**; `vf-hook-notify` implements the contract recorded verbatim in §3.3. |
| 21 | Lago / Stripe client approach in Rust | **Thin hand-written typed clients**, scoped to the endpoints §17.5 needs, with the providers' own OpenAPI specs pinned as the source of truth and a drift test. **Webhook signature verification is implemented in-house** with RustCrypto — not delegated to a vendor SDK — to the schemes recorded in §4.4. |

---

## 2. §27 item 19 — typed streaming RPC channel

> **§27 item 19** (Product/engineering, blocks Post-GA SDKs): *Is a typed streaming RPC
> channel needed beyond REST + SSE? If yes: tonic gRPC with gRPC-Web, or a
> Connect-compatible Rust implementation after maturity review.*

### 2.1 Decision

**No typed streaming RPC channel is adopted.** REST + SSE is the complete API surface at GA
and nothing in Phases 0–4 introduces a second transport. §2.5.2 already records "RPC: Not at
GA (REST only)"; this ADR converts that from a proposal into a decision and gives it a
revisit trigger.

### 2.2 Why — the design does not contain the need

- **The only streaming requirement in the design is server → client, and SSE serves it.**
  §9 and §2.4 make the live scan view a *view*: §21.4 requires that "in-flight scan survives
  browser/session loss — always", and §2.4 puts scan state server-side. The stream carries
  no commands, so bidirectional typed streaming buys nothing. A client that reconnects and
  re-reads state is the required behaviour either way.
- **A second surface doubles the §25 conformance burden.** `api/openapi-3_1-conformance`
  (ADR-0002 §4.1) would need an RPC twin, and every authorization and role-policy check in
  §4.2 would need enforcing on two transports. Two enforcement paths for one authorization
  concern is the shape of defect §27 item 18 exists to prevent; the fact that it would be
  two Rust paths rather than Rust-and-Go makes it cheaper, not safe.
- **gRPC-Web in a browser needs a translating proxy.** Aether is IPv4-only and
  self-operated, so that is an ingress component we would be adding and operating for a
  channel with no consumer yet.
- **The consumer does not exist.** §24.3 puts SDKs other than TypeScript in post-GA
  fast-follow, and the TypeScript SPA talks REST + SSE. Choosing a transport before there is
  a client to carry is how a project acquires an unused surface it must keep working.

### 2.3 A stale premise in §2.5.2, corrected

§2.5.2 justifies dropping Connect-RPC with: *"no first-party Connect-RPC Rust
implementation is assumed."*

**That is no longer true.** Observed on 2026-10-01: `connectrpc` on crates.io is published
from **`github.com/connectrpc/connect-rust`** — the ConnectRPC organisation itself, not a
third party. Repository created 2026-03-04, 521 stars, last pushed 2026-09-28, described as
*"An implementation of the ConnectRPC, gRPC, and gRPC-Web for Rust"*, and the crate
describes itself as *"A Tower-based Rust implementation of the ConnectRPC protocol"*.
Current release **v0.9.1** (2026-09-21), with a steady release cadence through 0.6 → 0.9
across 2026.

This does not change the decision — we are not adopting any RPC channel — but it does change
the *reasoning the TDD records*, and a stale premise left in place becomes the argument a
future reader relies on. §2.5.2's parenthetical should be corrected when the TDD is next
revised (that revision is tracked as the docs-correction issue, not here).

Two things follow. First, "Connect is impossible in Rust" is not a valid future argument.
Second, it is still **0.x**: pre-1.0, seven months old as a repository, and not yet something
to stake an external API contract on. "After maturity review", as item 19 puts it, remains
the right posture — and §2.4 states the bar so the review is not re-argued from scratch.

### 2.4 Pre-decided choice, if demand appears post-GA

The *whether* is a product question about post-GA SDK demand and belongs to CEO; it blocks
Post-GA SDKs, not Phase 0, so it is not a schedule risk now. The *which* is engineering's,
and is settled here so that a future demand signal does not reopen a technology debate:

- **Default: `tonic` 0.14.x + `tonic-web`.** Both at 0.14.6, with `tonic` at ~409M
  downloads and an optional `axum ^0.8` integration that matches the `axum 0.8.9` pin in
  ADR-0002 §3.1. It is the conservative choice and composes with the stack we already have.
- **Prefer `connectrpc` instead, if and only if, at the time the question is asked:**
  (a) it has reached **≥ 1.0** with a published stability commitment; (b) it is still
  published by the `connectrpc` organisation; and (c) it mounts as a `tower::Service` in our
  existing axum router without a parallel server. If all three hold it is the better choice,
  because it is the same wire protocol as the withdrawn Connect-Go surface and removes the
  gRPC-Web proxy entirely.
- **Rejected in advance:** a hand-rolled streaming protocol, and adding RPC "for internal
  service-to-service calls". Internal calls in this design go through Postgres, the outbox,
  and the Kubernetes API, not through a service mesh of RPC endpoints.

### 2.5 Revisit trigger

A concrete post-GA SDK requirement from a named customer or a signed commitment, **or** a
streaming requirement that SSE provably cannot meet — meaning client → server streaming, or
a typed schema obligation an OpenAPI 3.1 contract cannot express. Browser reconnection cost
is not such a requirement. On either trigger, re-run §2.4's three-part check and record the
outcome as a new ADR.

---

## 3. §27 item 20 — secureCodeBox parser and hook language

> **§27 item 20** (Engineering, blocks Phase 2): *SCB parser and hook language: keep
> upstream JavaScript parser SDK for custom parsers, or implement the parser contract in
> Rust; confirm the hook invocation contract the Rust `vf-hook-notify` must satisfy for the
> selected release.*

Both halves were probed against **secureCodeBox v5.9.0**, the release pinned in ADR-0002
§6.1. Observing the actual contract rather than the documentation is the whole point: these
are process-invocation ABIs, and they are specified by what the operator passes to the
container.

### 3.1 Decision — parsers stay JavaScript

**Keep the upstream JavaScript parser SDK for all parsers, including custom ones, through
GA.** No Rust parser SDK is built. This confirms §2.5.3's default and converts it from a
default into a decision.

### 3.2 Why — the contract is wide, and the parser is not safety-critical

**Observed.** `parser-sdk/nodejs/parser-wrapper.ts` at tag `v5.9.0`, and the SDK directory
listing at the same tag.

The first observation is about what upstream ships. At `v5.9.0`:

| SDK | Languages shipped upstream |
|---|---|
| `hook-sdk/` | **`golang`, `nodejs`** |
| `parser-sdk/` | **`nodejs` only** |

Hooks have a second-language SDK upstream; parsers do not. That asymmetry is itself
evidence: the hook contract has an upstream precedent for reimplementation and a maintained
second implementation to conform against, and the parser contract has neither.

The second observation is that the parser contract is not the narrow stdin → stdout
transform one might assume. `parser-wrapper.ts` does all of this, in order:

1. Requires env `SCAN_NAME` and `NAMESPACE`; honours `CRASH_ON_FAILED_VALIDATION`.
2. **Reads the Kubernetes API** — `GET` the `Scan` (`execution.securecodebox.io/v1`,
   plural `scans`), then `GET` the `ParseDefinition` named by `scan.status.rawResultType`.
3. Takes the raw-result download URL from `process.argv[2]` and the findings upload URL from
   `process.argv[3]`, and fetches the raw result as **text, or as binary if
   `ParseDefinition.spec.contentType == "Binary"`**.
4. Calls the parser, then **`addIdsAndDates`** (injects UUIDs and dates) and
   **`addScanMetadata`** (injects scan metadata) — both upstream-defined shapes.
5. Validates the result against the SDK's bundled `findings-schema.json`.
6. **`PATCH`es `scans/{name}/status`** with a merge patch carrying
   `status.findings.{count, severities{informational,low,medium,high}, categories{…}}`.
7. **`PUT`s** the findings JSON to the upload URL with header **`content-type: ""`** — an
   empty content type, which matters because the presigned URL's signature was computed for
   it.

So a Rust parser SDK means reimplementing: a Kubernetes client with RBAC for two CRD reads
and one status subresource patch, a presigned-URL download that switches on a CRD field, an
upstream-defined metadata and id/date injection shape, a JSON Schema validation against an
upstream schema file, and a `PUT` whose exact header set is signature-relevant. Each is a
place a reimplementation diverges silently, and divergence surfaces as missing or malformed
findings in the primary product data path. All of it is **coupled to the SCB release** —
`findings-schema.json` and the metadata shape move with the version — so it becomes a
permanent tracking cost against a dependency we pinned precisely to avoid tracking costs.

Against that: **the parser is not safety-critical in the §2.5.1 sense.** It performs no
scope matching, no allowance accounting, no role policy, and no graph rule evaluation. The
Rust argument that does apply is memory safety while parsing attacker-influenced scanner
output — and that is real, but it is answered at a better boundary:

- the parser runs as a one-shot Job in the scan namespace, holding no tenant data and no
  credentials beyond its own ServiceAccount; and
- **`vf-ingest` re-validates and re-canonicalises every finding before anything persists**,
  which is VulcanFlow-owned Rust and is where §25's `fuzz/untrusted-input` and
  `findings/replayed-artifact` apply.

The safety boundary we need is at ingest, which we own. Moving it into the parser would not
remove the need for it at ingest, because a parser is an untrusted producer regardless of
the language it is written in.

**The decision also costs nothing now.** The Phase 1 skeleton is `subfinder → dnsx → httpx →
nuclei` (§24.1), and upstream ships parsers for all four. **Phase 1 requires zero custom
parsers.** Deciding to stay on the upstream SDK defers no work and blocks nothing.

> **Corrected by amendment A1 (§7.1): the two sentences above are wrong.** secureCodeBox v5.9.0
> ships no `dnsx` and no `httpx` scanner, so Phase 1 requires **two** custom parsers. The decision
> to stay on the upstream JavaScript SDK is unchanged and is if anything better supported; what is
> wrong is the claim that it costs nothing. See
> [ADR-0006](./ADR-0006-false-positive-equivalence-and-alias-semantics.md) §4.1 for the evidence.

**Rejected alternatives.** A Rust parser SDK (above). A Go parser SDK — the language is
withdrawn by §0.0 and ADR-0001, and it would be a mixed-language implementation with no
compensating benefit. Parsing inside `vf-ingest` instead of in a parser container — this
abandons the SCB `ParseDefinition` model, loses the per-scanner upstream parsers we get for
free, and puts untrusted scanner-output parsing inside a service that holds tenant
credentials, which is strictly worse.

### 3.3 The hook invocation contract `vf-hook-notify` must satisfy — recorded verbatim

**Observed.** `hook-sdk/golang/sdk.go` at tag `v5.9.0`. This is the authoritative statement
of the contract, because it is a maintained upstream implementation of it in a language that
is not JavaScript.

**A hook is a one-shot process, not a server.** This is the single most important fact, and
getting it wrong would mean building `vf-hook-notify` as an HTTP listener that is never
called.

**One index base, stated once, used everywhere below.** Three index bases exist upstream for
the same four URLs, and mixing them is precisely the off-by-one this section exists to
prevent. **Every `argv[n]` in this section is absolute — the index into the process's own
argument vector, with the binary's path at `argv[0]`.** That is `std::env::args().nth(n)` in
Rust and `os.Args[n]` in Go. Converting between the bases, once:

| | Index of the **first** presigned URL | General rule |
|---|---|---|
| **This section (absolute)** | `argv[1]` | — |
| Rust, `std::env::args()` | `.nth(1)` | `.nth(n)` |
| Go SDK, which slices `os.Args[1:]` and then indexes the slice | slice position `0` | slice position **`n − 1`** |
| Node parser wrapper, where `process.argv[0]` is the interpreter and `[1]` the script | `process.argv[2]` | **`process.argv[n + 1]`** |

Nothing below uses a slice-relative or a Node index. Where upstream's own wording is quoted
it is converted to the absolute base, and the quote says so.

| Channel | Contract |
|---|---|
| Env `SCAN_NAME` | **Required.** The Go SDK's `NewClient` errors if empty. |
| Env `NAMESPACE` | **Required.** Same. |
| `argv[1]` … `argv[4]` | **Positional presigned URLs, in this exact order:** `argv[1]` raw-results **download**, `argv[2]` findings **download**, `argv[3]` raw-results **upload**, `argv[4]` findings **upload**. The Go SDK documents the same list over `os.Args[1:]` as "(rawResults, findings, rawResultsUpload, findingsUpload), in that order" — slice positions `0`–`3`, which are absolute `argv[1]`–`argv[4]`. |
| ReadOnly vs ReadAndWrite | **Distinguished only by how many URLs are passed.** A `ReadOnly` hook receives **2** — `argv[1]` and `argv[2]` are present, `argv[3]` and `argv[4]` are absent. The SDK's `urlAt(index)` returns `""` past the end and `UpdateRawResults` then fails with "cannot update raw results in a ReadOnly hook". There is no mode flag to read. |
| Raw-results download | `argv[1]` → `GET`, as text. |
| Findings download | `argv[2]` → `GET`, parse as a **JSON array** of `Finding`, each element validated. |
| Writes (ReadAndWrite only) | `PUT` to the upload URL — `argv[3]` for raw results, `argv[4]` for findings. `UpdateFindings` additionally **patches the `Scan` status** with the new finding statistics via the Kubernetes API. |
| Exit | Non-zero exit is hook failure. There is no response body. |

**Why the base is called out rather than left implicit.** Transcribing a Node index into Rust
yields an off-by-one that is a confusing runtime failure, not a compile error:
`std::env::args().nth(2)` in a Rust hook reads the *findings* download URL while the author
believes it is reading raw results, and both are valid presigned URLs, so the first symptom is
a parse error far from the cause. **A `vf-hook-notify` PR whose argv indices are not absolute,
or which does not read as if `argv[1]` is the first URL, is a required change at review.**

### 3.4 Decision — hooks are Rust

**`vf-hook-notify` is a Rust `ReadOnly` ScanCompletionHook** implementing §3.3 directly. No
hook SDK dependency is taken; the contract is a handful of environment variables, positional
arguments and HTTP GETs, and the Go SDK above is the conformance reference.

- **`ReadOnly`, deliberately.** Its job is to notify VulcanFlow that a scan finished and
  hand off the findings artifact to `vf-ingest`. It must not mutate findings or raw results:
  mutation in a hook would put finding transformation outside `vf-ingest` and outside the
  §25 tests that cover it. A PR that gives `vf-hook-notify` upload URLs is a required change.
- **It must therefore tolerate receiving exactly two argv URLs** and must not index past
  the end. The Go SDK's `urlAt` returning `""` rather than panicking is the behaviour to
  mirror; in Rust that is an `Option`, and `unwrap()` on it is a required change
  (§2.5.2 already forbids `unwrap` on external input paths).
- **Idempotency is required, not optional.** §8.4 requires ingest retries to be idempotent
  and §25 names `findings/replayed-artifact`. The hook can be invoked more than once for one
  scan; the scan fingerprint (§21.3, `metadata.uid` of the Scan object per ADR-0002's
  reading of §21.3) is the deduplication key, not the hook invocation.

**Executable confirmation.** Test ID `scb/hook-invocation-contract`: invoke the built
`vf-hook-notify` binary as a subprocess with `SCAN_NAME` and `NAMESPACE` set and a local
HTTP server standing in for the presigned URLs; assert that it reads `argv[1]`/`argv[2]`,
that it exits non-zero with a clear message when `SCAN_NAME` is absent, and that it does not
fail when only two URLs are supplied. **No cluster required.** Binary: **Anvil**. Test:
**Ledger**. Execution: **Crucible**.

A second test ID, `scb/parser-contract-conformance`, covers the upstream parsers we consume
rather than any we write: assert that the findings artifact a stock v5.9.0 parser produces
deserializes into `vf-ingest`'s types, including the id/date and scan-metadata fields the
wrapper injects. This is the test that catches a secureCodeBox bump changing the findings
shape. Authored by **Ledger** against the §25 entry `execution/scan-identity`.

### 3.5 One upstream quirk worth knowing before it is mistaken for our bug

The parser wrapper's `updateScanStatus` counts severities into exactly four buckets —
`informational`, `low`, `medium`, `high`. **There is no `critical` bucket in the Scan
status.** VulcanFlow's own severity model must not be derived from `Scan.status.findings`;
read severities from the findings artifact, which carries the parser's actual severity
values. Treating the CRD status as the severity source of truth would silently collapse or
drop critical findings, which in a security product is a reporting defect with customer
consequences.

> **Corrected by amendment A1 (§7.2): the remedy in the second sentence does not work.** The
> findings artifact cannot carry `CRITICAL` either — `parser-sdk/nodejs/findings-schema.json` at
> `v5.9.0` restricts `severity` to `INFORMATIONAL | LOW | MEDIUM | HIGH`, so no schema-conformant
> parser can emit it, and the v5.9.0 nuclei parser's `getAdjustedSeverity` maps `CRITICAL → HIGH`
> before the artifact is written. The warning stands; the fix is to derive severity at ingest from
> enrichment per §10.3 (`KEV → EPSS → CVSS`). See
> [ADR-0006](./ADR-0006-false-positive-equivalence-and-alias-semantics.md) §8.4.5.

### 3.6 Revisit trigger

Upstream publishing a non-JavaScript parser SDK — mirroring what `hook-sdk/golang` is for
hooks — would reopen §3.1, because the conformance reference we currently lack would then
exist. Also: a custom parser whose raw results are large enough that the Node runtime's
footprint breaches the scan-namespace limits, or an unpatched advisory in the parser SDK's
dependency tree. A change to the hook argv order or the `ReadOnly` URL-count convention in
any SCB release reopens §3.3 and is part of the ADR-0002 §6.3 bump checklist.

---

## 4. §27 item 21 — Lago and Stripe clients in Rust

> **§27 item 21** (Engineering, blocks Phase 4): *Lago and Stripe client approach in Rust:
> generated from provider OpenAPI specs, thin hand-written clients, or reviewed community
> crates; webhook signature verification implementation.*

§24.4 lists "usage reconciliation (including the Lago/Stripe client decision)" as blocking
**paid use**, not Phase 0 or Phase 1. This decision is made now so Phase 4 starts without a
technology debate, and it costs nothing to make early.

**Standing assumption.** This decides *how* we talk to Lago and Stripe, taking as given that
they are the providers. Whether they remain so is commercial and belongs to CEO (§27 items
2 and 4); a provider change is the revisit trigger in §4.6.

### 4.1 Decision — thin hand-written typed clients

**Hand-written typed clients, scoped to the endpoints §17.5 actually needs, with the
providers' own OpenAPI specs pinned as the source of truth for request and response
shapes.** Not full code generation; not a community crate.

### 4.2 Why not full generation from the provider specs

Both specs exist and are live, which was worth confirming rather than assuming:

- **Stripe:** `stripe/openapi` is actively released — latest release **v2526**, published
  **2026-09-30**, pushed the same day.
- **Lago:** `getlago/lago-openapi` holds `openapi.yaml` at the repository root with Spectral
  linting and generator templates, last pushed **2026-09-28**.

So generation is *available*. It is still the wrong default here:

- **Stripe's spec covers the whole Stripe API.** §17.5 needs usage records, subscription
  reads and a small number of customer operations. Generating the full surface means a very
  large crate, long compile times on every build, and `cargo-deny`/SBOM review over
  thousands of types nobody calls.
- **Stripe's spec regenerates roughly weekly.** A generated client turns provider release
  cadence into our diff cadence — weekly churn in a crate whose used surface did not change.
  That directly contradicts the "pinned and reproducible" posture in §21.3 and ADR-0002 §2.
- A generated client still needs a hand-written layer for idempotency keys, retry policy,
  pagination and error mapping, so generation does not remove the hand-written client — it
  adds a large dependency underneath one.

**The specs are still load-bearing**, just as reference rather than as build input: they are
the authority for field names, types and enum values, pinned by release tag (Stripe `v2526`)
and commit (Lago), and checked by a drift test. Where generation *is* worth it is one
narrow case, allowed explicitly: generating **types only** (no client, no transport) for a
single large response body, committed like the SCB CRD types in ADR-0002 §6.3 with the same
drift gate.

### 4.3 Why not the community crates — probed, and the answer differs per provider

| Crate | Version | Downloads | Verdict |
|---|---|---|---|
| `async-stripe` | stable **0.41.0**; **1.0.0-rc.9** (2026-09-11) | ~5.5M | **Rejected.** Widely used, but the 1.0 line has been in release-candidate since at least 1.0.0-rc.2 (2026-02) through rc.9 (2026-09) — eight RCs in seven months. The stable 0.41.0 line is the *older* API. Taking rc.9 means depending on a pre-release for billing; taking 0.41.0 means adopting an API its own maintainers have superseded. Neither is a good position for the code that reconciles money. |
| `stripe-rust` | 0.12.3 (**2020-05-17**) | ~73k | **Rejected.** Unmaintained for over six years. |
| `lago` / `lago-api` | 0.3.0 (2026-04) | **135 / 1 089** | **Rejected.** Effectively unused. |
| `getlago/lago-rust-client` | workspace `0.1.0` | 11 GitHub stars | **Rejected.** It is first-party, which is a point in its favour, and it is hand-written (a `lago-types` + `lago-client` workspace) rather than generated. But at v0.1.0 and 11 stars it is not a dependency to put under billing reconciliation — and its own `Cargo.toml` pins **`reqwest 0.12`** and **`chrono 0.4`**, against ADR-0002's `reqwest 0.13.5` and `jiff`. Adopting it would reintroduce `chrono` and a second HTTP stack for one integration. |

The honest summary: for Stripe there is a popular crate stuck mid-major-version, and for
Lago there is no viable crate at all. A thin client we own, over the `reqwest 0.13.5` already
pinned in ADR-0002, is both smaller than the alternatives and consistent with the rest of
the workspace.

**Scope discipline that makes this cheap.** The client covers only what §17.5 requires:
idempotent usage-record submission, subscription/entitlement reads, and webhook ingestion.
It is **not** a general-purpose SDK, and a PR that adds an endpoint with no §17.5 caller is a
required change. Every mutating call carries an idempotency key derived from our own ledger
row, because §17.5 requires idempotent outbox delivery and a retried usage push must not
double-count.

### 4.4 Webhook signature verification — implemented in-house, to these observed schemes

**Decision: verify signatures in VulcanFlow with RustCrypto (`hmac`, `sha2`), not through a
vendor SDK.** Both are pinned in [ADR-0002 §3.6](./ADR-0002-rust-crate-set-and-phase0-pins.md)
— an earlier revision of this paragraph said "already pinned in §2.5.2", which was wrong: §2.5.2
names them but pins nothing, and ADR-0002 §3 did not list them until amendment A1 added §3.6.
The constant-time comparison requirement and the base64/hex encoding details below are pinned
there too. Both schemes are small, both were read from the
providers' own source, and a signature verifier is exactly the kind of code that should be
ours, property-tested, and not reached through a dependency whose stable line we already
rejected.

**Stripe.** Observed in `stripe/stripe-ruby`, `lib/stripe/webhook.rb` (`master`):

- Header `Stripe-Signature`, formatted `t=<unix-seconds>,v1=<hex>`, comma-separated items.
- Signed material is **`"{timestamp}.{raw_body}"`**; signature is
  `HMAC-SHA256(secret, material)` **hex**-encoded, with the `whsec_…` endpoint secret.
- **Multiple `v1` values may be present** and verification succeeds if *any* matches — this
  is how secret rotation works. A verifier that reads only the first `v1` breaks during
  rotation.
- Comparison is **constant-time** (`Util.secure_compare`).
- Timestamp tolerance defaults to **300 s**; a timestamp older than the tolerance is
  rejected. This is the replay window.
- Upstream's own comment states the rule plainly: *"It's a good idea to parse the payload
  only after verifying it."*

  **Consequence for our handler:** the axum route must take the raw body as `Bytes` and
  verify before deserializing. A handler signature of `Json<StripeEvent>` has already
  parsed unverified attacker input and cannot recover the exact bytes that were signed —
  re-serializing is not byte-identical. **`Json<T>` on a webhook route is a required change
  at review.**

**Lago.** Observed in `getlago/lago-api`: `app/models/webhook.rb` (`generate_headers`,
`jwt_signature`, `hmac_signature`) and `app/models/webhook_endpoint.rb`
(`SIGNATURE_ALGOS = [:jwt, :hmac]`, DB default `jwt`).

- Headers: **`X-Lago-Signature`**, **`X-Lago-Signature-Algorithm`** (`"jwt"` or `"hmac"`),
  **`X-Lago-Unique-Key`** (the Lago webhook record id).
- `hmac` algorithm: `Base64.strict_encode64(OpenSSL::HMAC.digest("sha-256",
  organization.hmac_key, payload.to_json))` — HMAC-SHA256, **standard base64 with padding,
  no line breaks** (not hex, unlike Stripe, and not base64url).
- `jwt` algorithm: `JWT.encode({data: payload.to_json, iss: LAGO_API_URL}, RsaPrivateKey,
  "RS256")` — RS256, verified against Lago's RSA public key.

**Two findings here are security-relevant, not stylistic.**

1. **Lago's signature covers no timestamp.** Neither algorithm includes one in the signed
   material, so there is no tolerance window to check and **no replay protection in the
   signature at all** — unlike Stripe. Replay protection must come from deduplicating
   **`X-Lago-Unique-Key`** in a persisted inbox. §17.5 already requires idempotent delivery
   and §25 names `usage/lago-stripe-replay`; this makes the unique key the mechanism rather
   than an optimisation. A Lago webhook handler without persistent unique-key deduplication
   is accepting replays.
2. **Under the `jwt` algorithm the authentic payload is the `data` claim, not the HTTP
   body.** The body is not itself signed; a JWT whose `data` claim differs from the body is
   perfectly valid. A verifier that validates the JWT and then processes the request body is
   **processing unsigned input** while appearing to verify. This is a subtle and complete
   authentication bypass.

   **Therefore: configure the Lago webhook endpoint with `signature_algo: hmac`, not the
   `jwt` default.** With `hmac` the signature covers the body directly, there is no
   public-key fetch or key-rotation path to operate, and the body/claim ambiguity does not
   exist. If `jwt` is ever required, the handler must take the `data` claim as the payload
   and ignore the body entirely — and must also verify `iss`.

Both verifiers reject on: missing header, unparseable header, unknown algorithm, length
mismatch, and — for Stripe — timestamp outside tolerance. Verification failure returns 401
and does **not** echo the body or the computed signature into logs or the response.

**Executable confirmation.** Test IDs `billing/stripe-signature-verify` and
`billing/lago-signature-verify`, each a table-driven test over: a valid signature; a
tampered body; a tampered signature; a second rotated `v1` value (Stripe); an expired
timestamp (Stripe); a replayed `X-Lago-Unique-Key` (Lago); a wrong algorithm header; and a
truncated header. Plus a proptest asserting that no input causes a panic — these handlers
take attacker-controlled bytes on an unauthenticated route. Verifier: **Forge** (pure
library code in `vf-core`, no I/O, so it is property-testable per §2.5.2). Handler:
**Anvil**. Unit and property tests: **Scribe**. Integration tests: **Ledger**. These feed
§25's `usage/lago-stripe-replay`.

### 4.5 Where this code lives

The signature verifiers are **pure functions in a library crate** — bytes and a secret in, a
verdict out, no I/O — so they sit with the other property-testable logic per §2.5.2. The HTTP
clients live in `vf-meter`, which §2.3 already owns for usage sync. Keeping the verifier out
of the service crate is what lets Scribe fuzz it without standing anything up.

### 4.6 Revisit trigger

A provider change (CEO's call, §27 items 2 and 4). `async-stripe` **1.0.0 going stable** with
an API that matches our scoped usage — that is worth re-pricing, since the maintenance
argument flips once the crate is no longer mid-major-version. Either provider changing its
signature scheme, which the signature tests would catch as a hard failure. Our used endpoint
surface growing past roughly a dozen endpoints per provider, at which point types-only
generation (§4.2) becomes the better trade.

---

## 5. Items this record does not touch

- **§27 item 16a** — team Rust capability and the schedule impact of the language change.
  Engineering leadership, and a staffing question rather than a technical one.
- **§27 item 5's cluster half** — Harbor artifacts and node Kubernetes compatibility for the
  pinned secureCodeBox release. Risk R6 in ADR-0002 §7; it needs a cluster, and that hold is
  CEO's to lift.
- **The Product halves of items 2, 4 and 11** — package limits, billing-period edges and
  delivery consent. These block paid use alongside §4 above; this ADR decides only the
  client shape, not the commercial policy it will carry.

---

## 6. Provenance

Every version, download count, header name, default and quoted line above was read on
**2026-10-01** from: the crates.io API; the GitHub repository and contents APIs; and
`raw.githubusercontent.com` at the exact refs named —
`secureCodeBox/secureCodeBox` at tag **`v5.9.0`** (`hook-sdk/golang/sdk.go`,
`parser-sdk/nodejs/parser-wrapper.ts`, and the `hook-sdk`/`parser-sdk` directory listings),
`getlago/lago-api` at `main` (`app/models/webhook.rb`, `app/models/webhook_endpoint.rb`,
`app/graphql/types/webhook_endpoints/signature_algo_enum.rb`,
`app/services/webhooks/send_http_service.rb`), and `stripe/stripe-ruby` at `master`
(`lib/stripe/webhook.rb`).

These are **source-level probes of upstream contracts, not executions.** That is the right
form of evidence for this record: items 19, 20 and 21 ask what a contract *is* and which
implementation to choose, and those are answered by reading the authoritative source. Every
claim whose truth depends on our own code running instead carries a named test ID and an
owner above.

---

## 7. Amendment history

Amendments are recorded here rather than silently edited in, so a reader who reviewed an earlier
revision can see what moved. §7.1 and §7.2 are **factual corrections**; §7.3 is a **presentational
correction** — it changes how §3.3 states a contract, not what the contract is. None of the three
reverses a decision, and reversing one would need a new ADR that supersedes this record.

All three are entries under the single amendment **A1**. There is one amendment history on this
record, not one per correction.

### 7.1 A1 §1 — §3.2's custom-parser claim is wrong (2026-10-01)

§3.2 asserted that *"upstream ships parsers for all four"* scanners of the §24.1 pipeline and that
*"Phase 1 requires zero custom parsers."* Both are false.

**Observed.** The `scanners/` directory at `secureCodeBox/secureCodeBox` tag `v5.9.0` contains
`ffuf`, `git-repo-scanner`, `gitleaks`, `kube-hunter`, `ncrack`, `nikto`, `nmap`, `nuclei`,
`screenshooter`, `semgrep`, `ssh-audit`, `sslyze`, `subfinder`, `test-scan`, `trivy`, `trivy-sbom`,
`whatweb`, `wpscan`, `zap-automation-framework`. **There is no `dnsx` and no `httpx`.** Confirmed
a second way: `scanners/subfinder/parser/` and `scanners/nuclei/parser/` both contain `parser.js`,
and the equivalent paths for `dnsx` and `httpx` do not exist.

TDD §21.3 already anticipated the *images* — *"Build and validate the existing custom arm64 images
for dnsx, httpx, tlsx, masscan, and optional Amass"* — so the gap is not in the TDD. What §3.2 got
wrong is assuming a custom image comes with a parser. It does not: it needs a custom `ScanType`
**and** a custom `ParseDefinition` with a parser behind it.

**The decision in §3.1 is unchanged, and this strengthens rather than weakens it.** Once we are
certainly writing parsers, the choice between the upstream JavaScript SDK and a Rust
reimplementation matters more, and §3.2's seven-step reading of `parser-wrapper.ts` is the reason
to stay on the SDK: a Rust path would mean reimplementing the Kubernetes reads, the presigned
download that switches on `ParseDefinition.spec.contentType`, `addIdsAndDates`, `addScanMetadata`,
the schema validation, and the signature-relevant empty `content-type` on the `PUT` — for two
scanners, in Phase 1.

**What changes is the cost estimate.** Phase 1 carries two custom parsers (`dnsx`, `httpx`) plus
their `ScanType` and `ParseDefinition` manifests, not zero. Their field mapping is specified in
[ADR-0006](./ADR-0006-false-positive-equivalence-and-alias-semantics.md) §4.3–4.4, read from the
upstream tools' own output structs at `projectdiscovery/dnsx` `v1.3.1` (via
`projectdiscovery/retryabledns` `v1.0.116`) and `projectdiscovery/httpx` `v1.12.0`.

### 7.2 A1 §2 — §3.5's severity remedy does not work (2026-10-01)

§3.5 correctly observes that `Scan.status.findings` has only four severity buckets and that
VulcanFlow's severity model must not be derived from it. Its remedy — *"read severities from the
findings artifact, which carries the parser's actual severity values"* — does not hold.

**Observed.** `parser-sdk/nodejs/findings-schema.json` at `v5.9.0` constrains `severity` to
`enum: ["INFORMATIONAL", "LOW", "MEDIUM", "HIGH"]`. The four-bucket limit is therefore a property
of the **findings envelope**, not only of the CRD status, and no schema-conformant secureCodeBox
parser can emit `CRITICAL`. `scanners/nuclei/parser/parser.js` at the same tag confirms it in
practice: `getAdjustedSeverity` maps `CRITICAL → HIGH`, `INFO → INFORMATIONAL`, `UNKNOWN → LOW`,
and the original value is not preserved anywhere in `attributes`.

**The warning in §3.5 stands and the consequence is larger than it stated:** a critical finding
appears as `HIGH` everywhere downstream, and reading the artifact instead of the CRD status does
not recover it.

**The fix is already in the design of record.** §10.3 prescribes deterministic enrichment with the
risk ordering `KEV → EPSS → CVSS`, explicitly *not* an AI feature. VulcanFlow's severity must be
derived at ingest from that enrichment, with the scanner's severity retained as an input rather
than as the answer. The product-facing severity model — what a customer sees, and whether a
`CRITICAL` band exists at all — is a separate decision that needs an owner; it is named in
ADR-0006's "Does not close" row for that reason.

### 7.3 A1 §3 — §3.3 mixed two `argv` index bases in adjacent rows (2026-10-01)

**Raised by me** in review of the open-decision register (docs#24). §3.3 exists for exactly one
reason: to stop a Rust author transcribing a Node `argv` index and producing a silent off-by-one
in `vf-hook-notify`'s presigned-URL handling. It mixed index bases while doing it.

**What was there.** The `argv[1..]` row numbered the four URLs **slice-relative**:

> `argv[1..]` | **Positional presigned URLs, in this exact order:** `[0]` raw-results
> **download**, `[1]` findings **download**, `[2]` raw-results **upload**, `[3]` findings
> **upload**.

Two rows below, the download rows cited the same URLs **absolute** — *"Findings download |
`argv[2]`"* and *"Raw-results download | `argv[1]`"*.

**Both statements were correct.** Slice position `0` *is* absolute `argv[1]`, and slice position
`1` *is* absolute `argv[2]`. Nothing was wrong; the two bases simply sat four lines apart with
nothing saying they were different bases. A reader who took the first row's `[1]` as the findings
download — which is what that row literally says, in its own base — and then wrote
`std::env::args().nth(1)` would read the **raw-results** URL believing it was findings. That is
the exact failure §3.3 was written to prevent, reachable by reading §3.3 carefully.

**What changed.** §3.3 now opens with a conversion table that fixes one base — absolute, the
process's own argument vector with the binary at `argv[0]` — and gives the general rule for the
Go SDK's slice (`n − 1`) and the Node wrapper (`n + 1`) once. Every row below uses absolute
indices only; the Go SDK's quoted wording is retained and marked as converted. The raw-results
and findings download rows were also reordered to match index order, and the writes row now names
`argv[3]` and `argv[4]` explicitly instead of saying "the upload URL".

**Nothing about the contract moved.** The four URLs, their order, the two-URL `ReadOnly` case,
`urlAt` returning `""` past the end, and the `std::env::args().nth(1)` conclusion are all
unchanged, and the review rule is unchanged in substance — it is restated in terms of the fixed
base so it can be applied without re-deriving it.

**Why this one is rewritten in place rather than annotated.** `decisions/README.md` requires an
amended paragraph to be annotated rather than rewritten, and §7.1 and §7.2 follow that rule. This
entry does not, deliberately: the defect *is* the presentation, so leaving the mixed-base table in
place under a blockquote saying "these two rows use different bases" would preserve the trap in
the one section whose purpose is to close it. The before-state is quoted verbatim above, which is
what the annotate-in-place rule is for — a reader who reviewed the earlier revision can see
exactly what moved.

**§25 and test impact: none.** `scb/hook-invocation-contract` (§3.4) already asserts that the
binary reads `argv[1]` and `argv[2]`, in the absolute base, and that it tolerates receiving only
two URLs. The test is unchanged and no test author needs to act on this amendment. It is an
input to the implementer and to the reviewer of `vf-hook-notify`, not to the test.
