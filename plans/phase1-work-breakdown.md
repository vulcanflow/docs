# Phase 1 work breakdown and the §25 test-identifier split

**Status:** accepted
**Date:** 2026-10-01
**Author:** Atlas (Staff Architect / Tech Lead)
**Closes:** VUL-5
**Design of record:** TDD v2.3 (`VulcanFlow_Technical_Design_Document_v2.2.md`), §24 and §25
**Depends on:** ADR-0001 (Go scaffold retired), ADR-0002 (crate set pinned), ADR-0003 (RPC, SCB parser, billing clients)

---

## 1. What this document decides

Three things, so that none of them get re-litigated in a comment thread:

1. **Where the Phase 1 line is** — which TDD sections and which repos are in scope now, and
   which are deliberately not.
2. **Which of the 45 §25 test identifiers are in Phase 1 scope**, and for each in-scope
   identifier, the single issue that owns the test.
3. **The dependency order** — expressed as real `blockedByIssueIds` edges on the board, not
   as prose.

It does not decide product, pricing, legal, retention, disclaimer or infrastructure
questions. Those are CEO's and are listed in §7 below with the work they actually block.

---

## 2. Where the Phase 1 line is

Phase 1 is **§24.2's "Control plane" row**, read together with §24.1's standing mandate that
*minimum authorization, usage, audit and cancellation must exist with the first execution
path*. That mandate is why allowance accounting and the audit tables are Phase 1 work rather
than commercial hardening: Phase 1 can already scan public targets.

**In scope — repos and crates**

| Crate | Kind | TDD | Owner | Issue |
|---|---|---|---|---|
| `vf-core` | library, no I/O | §5.3, §5.9, §6.2, §10.4 | Forge | VUL-24 |
| `vf-authz` | library, no I/O | §4.2, §5.1–5.8 | Forge | VUL-25 |
| `vf-graph` | library, no I/O, native + wasm | §7.1–7.3, §14.3 | Forge | VUL-26 |
| `vf-translator` | library, no I/O | §7.4, §7.5, §5.7 | Forge | VUL-29 |
| `vf-db` (schema, RLS, append-only tables) | migrations | §3.5, §5.6, §6.3, §17.3, §21.2 | Forge | VUL-27 |
| `vf-db` (`TenantTx`) | library | §3.5, ADR-0002 §4.3 | Forge | VUL-13 |
| storage wrapper (single S3 constructor) | library | ADR-0002 §4.4 | Forge | VUL-14 |
| `vf-meter` (pure `Reservation`, unit arithmetic) | library | §17.1–17.3 | Forge | VUL-28 |
| `vf-api` (auth, role layer) | service | §4.1, §4.2, §13.3, §13.4 | Anvil | VUL-30 |
| `vf-api` (Track A targets, evidence) | service | §5.1, §5.2, §5.6, §13.2 | Anvil | VUL-31 |
| `vf-api` (dispatcher, outbox) | service | §8.1, §8.2, §8.3, §13.2 | Anvil | VUL-32 |
| `vf-api` (SSE) | service | §9.1–9.3 | Anvil | VUL-33 |
| `vf-api` (OpenAPI emission) | service | ADR-0002 §4.1 | Anvil | VUL-11 |
| `vf-meter` (reservation transaction) | service | §17.3–17.5 | Anvil | VUL-34 |
| `vf-ingest` | service | §8.4, §10.1–10.4 | Anvil | VUL-35 |
| `vf-hook-notify` | binary | §8.4, ADR-0003 §3.3 | Anvil | VUL-16 |
| `vf-operator` | controller | §8.1–8.3, §11.1, §3.3 | Kiln | VUL-36 |
| `vf-admission` | webhook | §5.7, ADR-0002 §4.2 | Kiln | VUL-12 |
| SCB v5.9.0 CRD types + drift gate | generated | ADR-0002 §6.3 | Kiln | VUL-15 |
| workspace, pins, CI gates | build | §2.5.2, §21.3 | Forge | VUL-6 |

**Not in scope, and why.** `vf-report`, `vf-abuse`, the TypeScript SPA, the scanner images,
the controlled masscan pool, the builder UI, remediation guidance content, verification
plans, scheduling, billing sync and AI. §24.2 sequences them into Phases 2–5, and §24.1 is
explicit that breadth before depth loses the thread. Note that §24.1's walking-skeleton
sentence reaches past the Phase 1 row — it also names a curated guidance set,
normal-consumption verification and one technical PDF report. Those three are **VUL-8's**
decomposition (the walking-skeleton epic), not this document's; VUL-8 is now blocked on all
23 Phase 1 issues and will be decomposed when they resolve.

---

## 3. The §25 split — 27 of 45 identifiers are in Phase 1 scope

The rule: **one identifier, one test-authoring issue.** Implementation issues *list* the
identifiers they must turn green; they do not own the test. Scribe owns unit, property and
fuzz. Ledger owns integration, conformance and e2e. Neither runs anything.

### 3.1 Scribe — 9 identifiers (VUL-7)

`authz/approval-tracks` · `authz/all-types-same-approval` · `authz/configured-scope` ·
`authz/psl-exact-root` · `execution/ipv4-only` · `graph/native-wasm-parity` ·
`usage/unit-definition` · `usage/remaining-work-notification` · `fuzz/untrusted-input`

These are the ones that reduce to a pure function over a value. That is not a coincidence: it
is the §2.5.1 argument for Rust and the reason scope matching, allowance arithmetic, graph
rules and observation state live in library crates with no I/O. A property test needs a pure
function to be a property *of*.

### 3.2 Ledger — 18 identifiers (VUL-22)

`auth/oidc-jwt-roles` · `auth/suspended-token` · `audit/authorization-history` ·
`authz/start-barrier-all-paths` · `execution/gates-before-start` · `isolation/all-stores` ·
`isolation/background-queries` · `graph/validator-translator-conformance` ·
`execution/scan-identity` · `execution/browser-disconnect` · `execution/late-cascade-barrier` ·
`findings/new-per-scan` · `findings/replayed-artifact` · `findings/fp-only-persistence` ·
`usage/failure-retry-settlement` · `usage/concurrent-reservations` ·
`build/rust-supply-chain` · `perf/service-baseline`

### 3.3 The seven ADR-derived identifiers (VUL-23, Ledger)

ADR-0002 and ADR-0003 approved the crate set *conditionally*, on executable confirmations
that were never filed as issues. Every one of VUL-11, VUL-12, VUL-13, VUL-14 and VUL-16 had a
production-source issue and no test counterpart. VUL-23 closes that gap:

`api/openapi-3_1-conformance` · `admission/cascade-gate-delete-oldobject` ·
`admission/workload-gate-dryrun` · `db/pgbouncer-transaction-pooling-prepared` ·
`db/tenanttx-set-local-isolation` · `storage/s3-compat-conformance` ·
`scb/hook-invocation-contract`

`db/tenanttx-set-local-isolation` is the single highest-value test in the set. A bare `SET`
instead of `SET LOCAL` under PgBouncer transaction pooling leaks tenant context to the next
borrower of that connection — a cross-tenant RLS bypass. That test failing means tenant data
is cross-visible in production.

### 3.4 Deferred — 18 identifiers, with the phase that owns each

| Identifier | Deferred to | Why |
|---|---|---|
| `remediation/version-tier-no-ai` | 3a | Guidance content is §15.2–15.3, Phase 3a. |
| `verify/applicable-check-evidence` | 3a | Verification plans are §15.4, Phase 3a. |
| `findings/new-after-fix` | 3a | Requires a verification outcome to exist first. |
| `usage/verification-normal` | 3a | §15.6 — needs the verification path. |
| `ui/performance-accessibility` | 3a | §14, the SPA. Not rewritten in Rust (§2.5.3). |
| `schedule/occurrence-identity` | 3b | §11 scheduling. |
| `report/matrix` | 3b | §16.1–16.4. |
| `report/cross-consistency` | 3b | §16.4, §16.8. |
| `report/xss-network-corpus` | 3b | §16.4 renderer. |
| `report/worker-and-queue-loss` | 3b | §16.4–16.5. |
| `report/disclaimer-framing` | 3b | §7.7, §16.7 — **and** blocked on CEO's disclaimer-wording decision (§24.4). |
| `report/branding-sharing` | 3b/4 | §16.6, §16.9 — blocked on CEO's public-sharing decision (§24.4). |
| `supply-chain/check-catalog` | 2 | §21.3 — pinned scanner images and nuclei templates. Needs the Phase 2 images. |
| `usage/lago-stripe-replay` | 4 | §17.5 — blocked on CEO's Lago/Stripe client decision (§24.4). |
| `abuse/egress-stop` | 4 | §20.3–20.5 — the controlled pool and egress operations. |
| `deletion/all-stores-restore` | 4 | §3.6. The *mechanism* is Phase 1 (`TenantTx`, RLS), but the test needs ClickHouse, object storage **and a real backup restore**, and §3.6 defers the retention policy to §27 (CEO). §24.2 puts full isolation and recovery evidence in Phase 4. |
| `release/analysis` | 4 | §21 promotion gate. |
| `recovery/restore` | 4 | §21.4 DR. Needs real infrastructure. |

27 in scope + 18 deferred = 45. The `[PROPOSED — owner review]` AAAA cases in §5.9 are not a
named identifier; the behaviour they describe is covered by `execution/ipv4-only`, and §24.4's
requirement to adopt the AAAA proposal explicitly or state an honest IPv4-only coverage
limitation remains open for CEO.

---

## 4. The binding test naming convention

An identifier maps to **exactly one** test function, with `/` and `-` replaced by `_`:

| Identifier | Test function |
|---|---|
| `authz/configured-scope` | `fn authz_configured_scope()` |
| `usage/failure-retry-settlement` | `fn usage_failure_retry_settlement()` |
| `db/tenanttx-set-local-isolation` | `fn db_tenanttx_set_local_isolation()` |

The file holding it carries a module doc comment `//! §25: <identifier>`, or
`//! ADR: <identifier>` for the seven in §3.3.

This exists so that §25 traceability is a **grep**, not a judgement. It is what lets Crucible
(VUL-37) publish a mechanical `PASS` / `FAIL` / `MISSING` ledger over all 34 identifiers
instead of an opinion about how the run went. An identifier whose test is named anything else
reads as `MISSING`, which is a finding against its test-author issue.

---

## 5. Dependency order

Three rules, all enforced as `blockedByIssueIds` on the board.

1. **VUL-6 first.** The workspace, the pinned toolchain and the committed `Cargo.lock` block
   everything, test and code alike. Nothing has anywhere to land until it exists.
2. **Tests land ahead of the code they cover.** Each implementation issue is blocked by the
   test-authoring issue that owns its identifiers. TDD §23.2 makes this safe: a listed
   identifier is a required test, **not** an assertion that it passes, so a named test that
   exists and fails is the correct early state. Test authors are never waiting on coding
   agents.
3. **Code lands bottom-up.** Pure library crates (`vf-core`, `vf-authz`, `vf-graph`,
   `vf-translator`, the `vf-meter` reservation value) and the schema before the services that
   consume them; services before the Kubernetes surfaces.

```
VUL-6  workspace + pins + CI gates (Forge)
 ├─ VUL-7   §25 unit/property/fuzz half      (Scribe, 9 ids)
 ├─ VUL-22  §25 integration/conformance/e2e  (Ledger, 18 ids)
 ├─ VUL-23  ADR-0002/0003 confirmations      (Ledger, 7 ids)
 │
 ├─ VUL-24 vf-core ────┐
 ├─ VUL-25 vf-authz ───┼──← VUL-7
 ├─ VUL-26 vf-graph ───┤
 ├─ VUL-28 vf-meter pure ┘
 ├─ VUL-27 vf-db schema ──← VUL-22
 ├─ VUL-15 SCB CRD types
 │    ├─ VUL-29 vf-translator
 │    └─ VUL-36 vf-operator
 ├─ VUL-13 TenantTx ──← VUL-22, VUL-23, VUL-27
 ├─ VUL-14 S3 constructor ──← VUL-23
 ├─ VUL-12 vf-admission ──← VUL-22, VUL-23
 ├─ VUL-11 OpenAPI emission ──← VUL-23
 ├─ VUL-16 vf-hook-notify ──← VUL-23
 ├─ VUL-30 vf-api auth ──← VUL-25
 ├─ VUL-31/32 vf-api targets, dispatcher ──← VUL-27
 ├─ VUL-33 vf-api SSE
 ├─ VUL-34 vf-meter service ──← VUL-27, VUL-28
 ├─ VUL-35 vf-ingest ──← VUL-27
 └─ VUL-37 Crucible: the 34-identifier ledger ──← VUL-7, VUL-22, VUL-23

VUL-8  walking skeleton epic ──← all 23 of the above
VUL-9  infra go/no-go (CEO) ──← VUL-8
```

VUL-37 is deliberately **not** blocked on the implementation issues. Its job is to publish the
ledger, and a ledger that reads all-red the first time it runs is doing exactly what it is
for.

---

## 6. The three-role test discipline

Identical for everyone, and enforced at review:

- **Forge, Anvil, Kiln** change production source only. Never a test file, never a
  `#[cfg(test)]` block.
- **Scribe, Ledger** change test files only. They never run them and never touch production
  source.
- **Crucible** runs suites and reports. It writes neither code nor tests.
- A failing test is a defect in the code. Nobody weakens, skips, deletes, `#[ignore]`s,
  loosens an assertion in, or rewrites a test to make a suite go green.
- If a test is genuinely wrong, that is an escalation to Atlas, who files a change request to
  a test author. It is never a silent edit by whoever hit the red.

Every issue in this breakdown carries this as a section. No issue asks a coding agent to write
a test, a test author to run one, or Crucible to write either.

---

## 7. What is deliberately still open, and what it blocks

Nothing here blocks the Phase 1 skeleton. All of it blocks something later, and all of it is
CEO's call rather than engineering's.

| Open item | Blocks | Owner |
|---|---|---|
| Target-slot semantics: active registered targets vs distinct scanned hostnames in the period (§17.2) | paid use. VUL-28 keeps the slot-accounting policy behind one named function so the decision changes one place. | CEO |
| Post-settlement corrections and mixed per-unit outcomes (§17.3, §8.2) | accepting scanner batches that cannot report success at the configured unit granularity — Phase 2. | CEO + Atlas |
| Lago/Stripe client decision (§27 item 21, ADR-0003) | `usage/lago-stripe-replay`, Phase 4. | CEO |
| Disclaimer wording, public report sharing, retention policy (§24.4, §3.6) | reports and `deletion/all-stores-restore`. | CEO |
| AAAA handling: adopt explicitly or state an honest IPv4-only coverage limitation (§5.9, §24.4) | report coverage claims. `execution/ipv4-only` encodes the current behaviour either way. | CEO |
| Confirmed-dangling-resource evidence (§5.3) | finding classification. Until then, label observed external resolution without asserting ownership or takeover. | CEO + Atlas |
| Lifting the `infra` `factory/phase0-foundation` hold | the residue Testcontainers cannot answer: sqlx behind real PgBouncer, S3 conformance against the real Aether RGW, and the Kubernetes paths in `authz/start-barrier-all-paths`. VUL-9, blocked on VUL-8. | CEO |

Everything in this breakdown is built and tested **without a cluster** — Testcontainers and
local runs only. Where a behaviour genuinely cannot be verified that way, the issue requires
it to be named as a residual risk in the PR rather than worked around. That residue is the
input to VUL-9.
