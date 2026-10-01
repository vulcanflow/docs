# ADR-0001 — Retire the Go scaffold in `vf-api` PR #1

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | TDD **§27 item 18** |
| **Does not close** | §27 items 16a, 17, 19, 20, 21 |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §0.0, §2.3, §2.5, §24 |

## Question

TDD v2.3 §0.0 confirms Rust as the primary implementation language for all VulcanFlow-owned
backend code, and withdraws the Go application frameworks named in v2.2 (chi, Huma,
controller-runtime, Connect-Go). §27 item 18 asks engineering to:

> Inventory any Go code already written against v2.0–v2.2 (API skeleton, operator, report
> pipeline, validator). Decide rewrite-now versus bounded coexistence; no long-lived
> mixed-language implementation of the same safety-relevant logic (scope, allowance, graph
> rules).

So: what happens to the Go that already exists, and is there a coexistence period?

## Inventory of pre-v2.3 Go code

Complete as of 2026-10-01. **One artifact:** `vulcanflow/vf-api` PR #1
(`factory/bootstrap-vf-api` → `main`) — open, mergeable, 10 files, 425 additions, 0 deletions.

| File | Lines | Content |
|---|---:|---|
| `LICENSE` | 201 | Apache-2.0 text — language-independent |
| `README.md` | 50 | Scaffold description; cites TDD v2.2; names chi / Huma / Connect-RPC |
| `go.sum` | 18 | Dependency hashes |
| `.github/workflows/ci.yml` | 19 | `go test ./...`, `go vet ./...` |
| `.gitignore` | 13 | Go build artifacts |
| `go.mod` | 8 | Module path, Go 1.26, chi v5.3.2, huma v2.39.1 |
| `cmd/vf-api/main.go` | 25 | Listener bootstrap, `VF_API_ADDR`, read-header timeout |
| `internal/server/mux.go` | 25 | chi router, `GET /healthz`, `humachi` OpenAPI hook |
| `internal/server/mux_test.go` | — | Asserts `/healthz` and OpenAPI 3.1 output |
| `internal/dispatcher/dispatcher.go` | 6 | Package doc comment only — no declarations |

**Safety-relevant logic present: none.** No scope matching, no allowance reservation or
settlement, no observation state, no role policy, no graph rules. `internal/dispatcher` is a
doc comment reserving a package name. Every executable line is HTTP plumbing, and the
scaffold's own README says so.

**Elsewhere in the org:** `main` is empty in all 14 repositories. The only other open PR is
`vulcanflow/infra#1`, which is YAML / Kustomize GitOps with no Go in it. No Go exists in
`vf-operator`, `vf-report`, or any validator. The §27 item 18 inventory is exactly one
scaffold PR — the "operator, report pipeline, validator" the item anticipated were never
written.

## Options considered

**1. Merge PR #1, rewrite in place later (bounded coexistence).**
Puts a chi/Huma service on `vf-api` `main` as the organisation's only application code, and
therefore as the de facto template every later bootstrap copies. Requires a Go toolchain, Go
CI, and a second supply-chain gate (Go module auditing alongside the cargo-deny / cargo-audit
/ SBOM gates in §21.3) to maintain 56 lines of HTTP plumbing. §24.1 requires the walking
skeleton to be "all VulcanFlow-owned components in Rust from the first commit"; merging
contradicts the Phase 1 gate on day one. And `internal/dispatcher` is precisely where
allowance reservation lands (§2.3) — coexistence there is the thing item 18 forbids outright.

**2. Convert PR #1 in place to Rust on the same branch.**
Nothing in the diff survives translation. `go.mod`, `go.sum`, `ci.yml` and `.gitignore` are
toolchain-specific; the chi + `humachi` mux has no Rust analogue to port; `main.go` is 25
lines of listener setup. Only `LICENSE` carries over. A "conversion" would be a
delete-everything-and-start-over commit wearing a conversion label, leaving a review history
that misrepresents what happened. It also presupposes the per-repository Go-module layout
that §2.5.2 replaces: `vf-api` is a **binary crate inside the shared Cargo workspace**, not a
standalone module.

**3. Close PR #1 unmerged; Rust lands through the Phase 0 workspace scaffold.**
Keeps every default branch empty until the approved Rust workspace exists, which is what
§24.2 Phase 0 orders. Cost: the 56 executable lines and the one test are discarded. Both are
reproducible in minutes.

## Decision

**Close PR #1 unmerged.** Option 3.

VulcanFlow writes no new Go, and no Go is carried forward. **There is no coexistence period,
bounded or otherwise** — because there is nothing to coexist with. "Rewrite now versus bounded
coexistence" is a question about sunk cost, and here the sunk cost is 56 lines of HTTP
plumbing with zero safety-relevant logic. That is below the cost of maintaining a second
toolchain and a second supply-chain gate for one `/healthz` handler.

The §2.5.3 exceptions are unaffected, and none of them is Go that we own: upstream scanners
(subfinder, dnsx, httpx, tlsx, nuclei, Amass in Go; masscan in C), the secureCodeBox
operator / lurker / stock hooks, and a post-GA Terraform provider. These are wrapped, pinned
and conformance-tested, never rewritten, and none of them implements scope, allowance or
graph rules.

## Consequences

- `vf-api` PR #1 is closed unmerged. Branch `factory/bootstrap-vf-api` is retained for
  provenance, not for merge.
- All 14 default branches stay empty until the Cargo workspace scaffold lands. This is the
  intended Phase 0 state, not drift.
- **The per-repository Go bootstrap issues do not translate one-for-one into per-repository
  Rust bootstraps.** §2.5.2 and the Appendix C glossary define the Cargo workspace as "the single
  Rust build unit containing all VulcanFlow crates, with one pinned toolchain and lockfile."
  Six independent Rust bootstraps would produce six lockfiles and six toolchains, and would
  reintroduce `vf-core` version skew across exactly the crates that hold scope, allowance and
  finding state. They are closed in favour of the one workspace scaffold (appendix below).
- **No crate is approved by this ADR.** §27 item 17 stays open, and §24.4 still gates
  execution-path code on it. The §2.5.2 crate table remains `[PROPOSED]`.
- Dispatcher allowance reservation stays unimplemented until it can be written once, in Rust,
  in a pure library crate, with its §25 test identifiers attached.

## What would make me revisit this

- **A §27 item 17 probe failure that has a mature Go answer and no Rust one** — utoipa cannot
  emit conformant OpenAPI 3.1 with Problem Details modelling; kube-rs cannot cover the §5.7
  admission surface; sqlx misbehaves behind PgBouncer transaction pooling; or no S3 client
  passes Ceph RGW / RustFS conformance including path-style addressing and default checksum
  behaviour. A failure there is an argument about the *stack*, and it reopens §0.0 — not this
  ADR. While §0.0 stands, this decision stands.
- **An owner reversal of the §0.0 Implementation language row.**
- A §27 item 20 outcome that keeps a VulcanFlow-maintained secureCodeBox parser in upstream
  JavaScript. That is permitted by §2.5.3 and is not a reversal of this ADR.

**Not a reason to revisit:** schedule pressure on the Phase 1 skeleton. §27 item 16a tracks
Rust capability and the schedule impact of the language change. The answer to a schedule
problem is sequencing or enablement, not a second language in the control plane.

## Appendix — disposition of the `docs` bootstrap issues

All of `docs` issues #5–#16 were written as Go bootstrap tasks against TDD v2.2. (#15 is a
pull request, not an issue; the issue set is the eleven below.)

**Closed as superseded** — each was a per-repository Go-module bootstrap, replaced by the
single Cargo workspace per §2.5.2:

| Issue | Was | v2.3 position |
|---|---|---|
| #5 | `vf-api` Go service: chi / Huma / Connect-RPC skeleton + dispatcher | Binary crate `vf-api`; axum + utoipa `[PROPOSED]`; RPC dropped at GA (§27 item 19) |
| #6 | `vf-authz` **Go service** | **Library crate**, not a service (§2.3, §2.5.2) — used by `vf-api` and workers |
| #7 | `vf-operator` with controller-runtime / kubebuilder | Binary crate on kube-rs; `vf-admission` split out as its own crate (§2.3) |
| #8 | `vf-translator` Go library | Library crate; graph *validation* splits into the new `vf-graph` crate, native + WASM (§7.3) |
| #9 | `vf-ingest` Go service | Binary crate `vf-ingest` |
| #16 | `vf-meter` Go service | Binary crate `vf-meter`; Phase 1 core accounting unchanged |

**Rewritten for v2.3** — still needed, language references corrected:

| Issue | Change |
|---|---|
| #10 | scanners: upstream tools stay Go/C and are *not* rewritten (§2.5.3); VulcanFlow input adapters and the completion hook (`vf-hook-notify`) are Rust; parsers deferred to §27 item 20 |
| #11 | `vf-web` stays TypeScript/React (§14), but the graph validator is **not** written in TypeScript — it is `vf-graph` compiled to WASM (§7.3, §14.1, §14.3) |
| #12 | Later-phase READMEs: Rust for `vf-remediation`, `vf-report`, `vf-abuse`; `vf-aigw` stays LiteLLM/Python as an approved third-party component |
| #13 | Org CI conventions: Go `test`/`vet`/golangci-lint replaced by pinned `rust-toolchain.toml`, committed `Cargo.lock`, `cargo fmt`/`clippy -D warnings`/`test`, cargo-deny, cargo-audit, SBOM, arm64 |
| #14 | Factory repository wiring: v2.3 references; repository list follows the workspace topology |
