# Phase 1 work breakdown and the §25 test-identifier split

**Status:** accepted
**Date:** 2026-10-01
**Author:** Atlas (Staff Architect / Tech Lead)
**Closes:** the Phase 1 breakdown work item — see §5.1 on the issue ids in this document
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

#### 3.2.1 `authz/start-barrier-all-paths` carries a scope limit, recorded here rather than found at review

§25's row for this identifier is *"Admission and start checks **including pool**"* (§5.7). §2 puts
the controlled masscan pool in Phase 2 and lists it as explicitly out of Phase 1 scope. A Phase 1
test therefore has no live pool to assert the pool path against. This section says what the test
asserts instead, and what Crucible records when it goes green. **It must not surface as a weakened
assertion discovered at review.**

**The identifier is not split.** §25's mapping is injective in both directions — one identifier,
one test function. `authz/start-barrier-all-paths` stays whole, stays in Phase 1, stays Ledger's.
Splitting its pool half into a second Phase 2 identifier was the alternative and it loses: it would
put two test functions behind one §25 row, which is the property the whole mapping rests on.

**What the pool path asserts in Phase 1 — against the admission contract, not a live pool.** A
replayed `AdmissionReview` carrying a pool-dispatched `Scan` — pool namespace, pool service
account, no tenant approval and no allowance reservation — must be **denied** by `vf-admission`,
through the same §5.7 gate and with the same denial reason as the tenant path. That is the whole of
the **pool limb** as a property of our handler, and it is fully assertable with the recorded-JSON
mechanism ADR-0002 §4.2 established. No cluster, no pool, no Phase 2 dependency. The identifier's
other limbs — Scan UPDATE, Job and Pod admission, revocation, dry-run — are §3.2.2, and none of
them carries a scope limit.

**What it does not assert, and who carries the residue.** That the real pool dispatcher actually
goes through admission — that no path exists by which a pool-originated `Scan` reaches the API
server without hitting the webhook. That is a property of the Phase 2 pool's deployment, not of our
handler. It is **ADR-0002 §7 risk R6** and **TDD §27 item 5**, which names *"atomic candidate
reservation and actual start barrier in tenant **and pool** execution"*.

**What Crucible records.** When this identifier goes green in Phase 1 the ledger entry is:

```
authz/start-barrier-all-paths   PASS (scope: pool path asserted against the admission
                                contract; no live pool — breakdown §3.2.1)
```

**A bare `PASS` on this identifier in Phase 1 is a ledger defect**, and Crucible should reject it
rather than record it. The point of the qualifier is that the limit is visible in the artifact a
reader trusts, not only in this document.

**Phase 2's closing gate owns removing it.** Phase 2 stands up the controlled pool; its closing gate
must re-run the same test function with the live pool dispatching and drop the qualifier. Until it
does, the identifier is **not** fully discharged, and **Phase 2 cannot close with the qualifier
still attached.** That is a Phase 2 gate obligation, recorded in §3.4.

**`execution/gates-before-start` does not get the same qualifier, and the difference is the point.**
It shares the identical R6 residue — ADR-0002 §7 names the pool half of both — but its §25 row is
*"No scan before approval or allowance reservation"* (§5.7, §8.1) and does not promise pool
coverage. Its Phase 1 green is complete against what §25 asks of it, and the pool residue is tracked
as R6 rather than as a ledger scope limit. A scope qualifier is owed where a §25 row's own words
reach past what Phase 1 can assert; inventing one where they do not would make the ledger noisier
without making it more honest.

#### 3.2.2 The pool path is one limb of this identifier, not the whole of it

§3.2.1 says what the *pool* limb asserts, because the pool limb is the one with a scope limit.
It is not the identifier's spec. §25's row points at **§5.7**, which names more start checks
than a pool-dispatched `Scan`, and every one of them is handler-testable from recorded
`AdmissionReview` JSON with no cluster — the same mechanism, so there is no scope limit on any
of them and none of them inherits §3.2.1's qualifier.

The Phase 1 test must cover, at minimum, each of these §5.7 paths:

| §5.7 requirement | The admission case to replay |
|---|---|
| *"Every root or child Scan must refer to a live authorization basis, a permitted pipeline/node, and a reserved work unit"* | `Scan` CREATE with a missing, expired or revoked basis; with an unreserved work unit; with a pipeline/node the basis does not permit |
| *"checks Scan CREATE and security-relevant UPDATE operations"* | `Scan` UPDATE mutating scan type, template, command or environment override, init container, volume or service account — each individually, and each **denied** |
| *"validates target annotations against generated parameters and immutable target lists"* | `Scan` CREATE whose target annotation disagrees with the generated parameters, and one that reaches a destination outside the immutable list |
| *"Apply the same authorization/reservation check to scanner Job admission, including pool Jobs"* | `Job` CREATE in a tenant namespace and in the pool namespace, with and without a reserved work unit |
| *"protect descendant Pod creation and retries"* | descendant `Pod` CREATE under an admitted `Job`; a retry `Job`/`Pod` after the original attempt |
| *"Permit only operator-managed workload creation through RBAC and admission"* | a direct workload `CREATE` by a non-operator service account — **denied** |
| *"New execution after revocation must be refused"* | `Scan` and `Job` CREATE after the basis is revoked — **denied**, and the §5.7 cancellation limb is `vf-operator`'s behaviour, not admission's |
| *"Admission success must not consume a unit or cause external side effects during dry-run"* | `dryRun` admission, asserting no reservation and no side effect — this is ADR-0002 §4.2's `admission/workload-gate-dryrun` feeding in |
| The pool limb | §3.2.1 |

**This is still one test function.** The §25 mapping is injective and stays so: these are cases
in one table-driven function, not nine functions. A reviewer counting functions per identifier
should count one.

**What Phase 2 adds is still only §3.2.1's residue** — that the live pool dispatcher reaches the
API server through the webhook. Phase 2 does not owe a second round of the rows above, because
a cluster adds nothing to a decision the handler makes from the request it is handed
(ADR-0002 §7.1(d)). Delayed starts, infrastructure retries and revocation are all in the table
above and are all discharged in Phase 1, as admission decisions. Their *deployment* residue is
R1, R2 and R6 in ADR-0002 §7, carried as accepted risk, and the identifier is not held open for
them — §7.1(d) is explicit that the middle column says what a cluster would *add* to a green,
not that the green is incomplete.

#### 3.2.3 §8.3's cancellation limb has no §25 identifier at all — named here, fixed in the TDD

Raised at review against §3.2.2's revocation row: that row says the §5.7 cancellation limb is
`vf-operator`'s behaviour and not admission's, and then never says which identifier asserts it. The
honest answer is **none**, and the gap is wider than the row.

**What the TDD actually says, read at this head.** §25's 45 rows cite **§8.1**
(`execution/gates-before-start`), **§8.2** (`execution/late-cascade-barrier`) and **§8.4**
(`findings/replayed-artifact`). **No row cites §8.3 (*Cancellation and suspension*).** Yet §2 of this
document assigns `vf-operator` (VUL-36, Kiln) §8.1–**8.3**, and §24.2's control-plane row names
*"basic cancellation"* in Phase 1. So Phase 1 ships a code path for which §3's own rule —
implementation issues list the identifiers they must turn green — has no identifier to list.

Both §5.7 and §5.8 state the requirement in two limbs: *"New execution after revocation must be
refused; revocation also triggers cancellation of already-running work"* (§5.7), and *"Explicit
revocation prevents new execution and cancels existing work"* (§5.8 bullet 7). The **refusal** limb
is `authz/start-barrier-all-paths`, §3.2.2's revocation row. The **cancellation** limb is mapped by
nothing.

**`abuse/egress-stop` does not cover it, and claiming it does would be the invented-edge defect.**
Its §25 requirement is *"Effective tenant and pool kill"* at §20.3–20.5. §20.3's trigger is a
persisted **tenant suspension** and its assertion is the `[PROPOSED]` under-60-second egress-stop
measurement; §3.4 defers it to Phase 4. Revocation of one authorization basis is a different trigger
with a different blast radius — one target's running work, not the tenant's. Reading the edge in
anyway is exactly the failure ADR-0002 amendment A3 was filed to correct: a missing edge and an
invented edge are the same defect.

**Why this section does not fix it.** A plan cannot mint a §25 identifier. The 45 is TDD §25's set;
the program plan's 45-identifier mapping, this document's `27 + 18`, and Crucible's Phase 1 ledger
count all derive from it. A 46th identifier is a **TDD amendment** — Atlas's and nobody else's — and
not something to land inside a breakdown PR to clear a review comment. If it is not in the TDD it is
not a §25 identifier, however good the reason.

**What this document records instead, and the consequence.**

1. The seam is named here, so **VUL-36's scope line cannot be written as though §8.3 had a test
   author.** Under §3's rule it has none today.
2. **Phase 1's closing gate must not claim §5.8 bullet 7 discharged.** Its refusal limb is green via
   `authz/start-barrier-all-paths`; its cancellation limb is untested, and a gate that reads the
   bullet as satisfied is wrong on the second half.
3. The §25 question is routed to **[VUL-182](/VUL/issues/VUL-182)**, which states both candidate
   resolutions — mint a Phase 1 identifier for the §8.3 path, or widen §25's `abuse/egress-stop` row
   to carry both triggers and re-argue its Phase 4 deferral against §24.1. Until it resolves, the
   count here stands at `27 + 18 = 45` and **no identifier is invented**.

**Revisit trigger.** When that amendment lands, either a new identifier appears in §3.2 or §3.4 with
exactly one phase and one author, or `abuse/egress-stop`'s Phase 4 deferral in §3.4 is reopened.
Nothing in §3.2.1 or §3.2.2 changes either way — this is a gap beside that identifier, not a defect
in it.

### 3.3 The eight ADR-derived identifiers (VUL-23, Ledger)

ADR-0002 and ADR-0003 approved the crate set *conditionally*, on executable confirmations
that were never filed as issues. Every one of VUL-11, VUL-12, VUL-13, VUL-14 and VUL-16 had a
production-source issue and no test counterpart. VUL-23 closes that gap:

`api/openapi-3_1-conformance` · `admission/cascade-gate-delete-oldobject` ·
`admission/workload-gate-dryrun` · `db/pgbouncer-transaction-pooling-prepared` ·
`db/tenanttx-set-local-isolation` · `storage/s3-compat-conformance` ·
`scb/hook-invocation-contract` · `scb/parser-contract-conformance`

#### 3.3.1 This list said *seven* and was wrong — the adjudication, recorded

An earlier revision of this section listed seven, omitting `scb/parser-contract-conformance`,
while ADR-0002 amendment A3 §7.1(a) enumerates **eight**. CEO raised the contradiction on
VUL-41 rather than choosing between them, which is the right call: a plan does not get to pick
between two design-of-record sources. **The ADR is right and this section was wrong.**

`scb/parser-contract-conformance` is a named test ID in ADR-0003 §3.4, with an author
(**Ledger**), a stated assertion — that the findings artifact a stock v5.9.0 parser produces
deserializes into `vf-ingest`'s types, including the id/date and scan-metadata fields the
wrapper injects — and a §25 target, `execution/scan-identity`, which §3.2 puts **in Phase 1**.
It needs no cluster and no scanner run: a committed artifact from a stock v5.9.0 parser is a
fixture, exactly as the replayed `AdmissionReview` JSON of ADR-0002 §4.2 is a fixture.

**Why it was dropped is worth knowing, because the same slip recurs.** This section's own
preamble enumerates by *production-source issue* — VUL-11, VUL-12, VUL-13, VUL-14, VUL-16 —
and `scb/parser-contract-conformance` has no crate-approval issue of its own, because the code
it exercises is upstream and the types it lands in belong to `vf-ingest`. An enumeration keyed
on implementation issues will always lose the confirmations that assert an upstream contract
against our own types. **The ADR-derived census is ADR-0002 §7.1(a); this section is a Phase 1
scope list that reconciles against it, never the reverse.**

**Consequence for Crucible.** The Phase 1 ledger is **35 rows**: 27 §25 identifiers plus 8
ADR-derived confirmations. Every count in §4 and §5 is corrected to 35 in the same change.
**The 45 is untouched** — ADR-derived confirmations are not §25 identifiers and are never
counted into the 45 (ADR-0002 §7.1(a)). ADR-0002 §6.3's CRD-codegen drift gate is a CI job
with no test identifier; it is not a ninth row and belongs to the CRD-types work item.

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

**One Phase 2 gate obligation that is not a deferral, recorded next to them so it is not read as
one.** `authz/start-barrier-all-paths` is **in** Phase 1 and is **not** in the table above. What
Phase 2 owns is removing its scope qualifier: re-running the same test function with the live
controlled pool dispatching, and dropping the
`PASS (scope: … no live pool)` recorded in Phase 1. See §3.2.1. Phase 2 cannot close with that
qualifier still attached. The identifier count is unaffected — 27 + 18 = 45 either way, because
nothing moved phase.

**One deferral that may not survive, flagged where a reader will look for it.**
`abuse/egress-stop`'s Phase 4 row above is deferred on its own §20.3–20.5 terms, and that reasoning
stands. What is open is whether §25 should also make it carry the **revocation**-triggered
cancellation of running work, which §24.1 requires to exist with the first execution path and which
§25 currently maps to no identifier at all — see §3.2.3 and
**[VUL-182](/VUL/issues/VUL-182)**. If that amendment widens this row rather than adding a new
identifier, this deferral is reopened and the Phase 4 entry splits. Recorded as an open edge on the
table, not as a silent assumption that Phase 4 is safe.

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
`//! ADR: <identifier>` for the eight in §3.3.

This exists so that §25 traceability is a **grep**, not a judgement. It is what lets Crucible
(VUL-37) publish a mechanical `PASS` / `FAIL` / `MISSING` ledger over all 35 identifiers
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

### 5.1 Every `VUL-n` below is from a previous board and is dangling

This document was written against a **previous Paperclip company's** issue numbering. This
company starts at VUL-1, and every `VUL-n` in this section, in §2's issue column and in §3's
headings resolves to an unrelated issue or to nothing. **Do not follow one.** Resolve a work
item through Paperclip by title, not by the number printed here.

What is decided here is the **shape** of the graph — which work item blocks which, and why.
The numbers are placeholders for items that do not yet exist on this board: there is no
`vf-core` issue, no `vf-authz` issue and no Ledger test-authoring issue to point at. Creating
that issue set with real `blockedByIssueIds` edges, and reconciling these citations, is the
scope of the board-reconcile work item (cited here as VUL-7, and itself subject to this
paragraph). Until it runs, this graph is a specification of edges, not a set of links.

It is published with the dangling ids rather than held back for them. The alternative was to
keep the file off `main` until the board existed, and that is what produced the defect this
re-file closes: three amendments to this document — §3.2.1's pool-path scope limit, §3.3.1's
eighth ADR-derived identifier, and the 34 → 35 ledger count that follows from it — had nowhere
to land for as long as the file was unmerged.

### 5.2 The graph

```
VUL-6  workspace + pins + CI gates (Forge)
 ├─ VUL-7   §25 unit/property/fuzz half      (Scribe, 9 ids)
 ├─ VUL-22  §25 integration/conformance/e2e  (Ledger, 18 ids)
 ├─ VUL-23  ADR-0002/0003 confirmations      (Ledger, 8 ids)
 │
 ├─ VUL-24 vf-core ────┐
 ├─ VUL-25 vf-authz ───┼──← VUL-7
 ├─ VUL-28 vf-meter pure ┘
 ├─ VUL-26 vf-graph ──← VUL-7, VUL-22
 ├─ VUL-27 vf-db schema ──← VUL-22
 ├─ VUL-15 SCB CRD types
 │    ├─ VUL-29 vf-translator ──← VUL-22
 │    └─ VUL-36 vf-operator ──← VUL-22
 ├─ VUL-13 TenantTx ──← VUL-22, VUL-23, VUL-27
 ├─ VUL-14 S3 constructor ──← VUL-22, VUL-23
 ├─ VUL-12 vf-admission ──← VUL-22, VUL-23
 ├─ VUL-11 OpenAPI emission ──← VUL-23
 ├─ VUL-16 vf-hook-notify ──← VUL-23
 ├─ VUL-30 vf-api auth ──← VUL-22, VUL-25
 ├─ VUL-31/32 vf-api targets, dispatcher ──← VUL-22, VUL-27
 ├─ VUL-33 vf-api SSE ──← VUL-22
 ├─ VUL-34 vf-meter service ──← VUL-22, VUL-27, VUL-28
 ├─ VUL-35 vf-ingest ──← VUL-22, VUL-23, VUL-27
 └─ VUL-37 Crucible: the 35-identifier ledger ──← VUL-7, VUL-22, VUL-23

VUL-8  walking skeleton epic ──← all 23 of the above
VUL-9  infra go/no-go (CEO) ──← VUL-8
```

**Rule 2 is applied to every node, not to some of them.** An implementation node carries an
edge to **VUL-7** if any identifier it must turn green is Scribe's (§3.1), to **VUL-22** if any
is Ledger's (§3.2), and to **VUL-23** if any is an ADR-derived confirmation (§3.3). An earlier
revision of this graph applied that unevenly — `vf-api` SSE had no test edge at all, and
`vf-ingest`, `vf-api` auth, the dispatcher, `vf-meter`'s service, `vf-graph`, `vf-translator`
and `vf-operator` were each missing the Ledger edge for identifiers §3.2 assigns to them.
A missing edge here is not a presentational defect: rule 2 is the mechanism that keeps tests
ahead of code, so an implementation node with no test edge is a node whose code may land first.

**Two deliberate exceptions, both on VUL-6.** `build/rust-supply-chain` and
`perf/service-baseline` are Ledger identifiers (§3.2) about the workspace VUL-6 builds, so an
edge would make VUL-6 depend on a node that depends on VUL-6. Rule 1 resolves it: VUL-6 lands
first with the gates themselves — `cargo-deny`, `cargo audit`, the SBOM, `#![forbid(unsafe_code)]`
— and the two identifiers are authored against them afterwards, under VUL-22. **VUL-6 is the
one place in this plan where code precedes its test**, and it is recorded here rather than left
to look like an oversight.

VUL-37 is deliberately **not** blocked on the implementation issues. Its job is to publish the
ledger, and a ledger that reads all-red the first time it runs is doing exactly what it is
for.

---

## 6. The three-role test discipline

Identical for everyone, and enforced at review:

> **Superseded in part by [ADR-0005](../decisions/ADR-0005-delivery-pipeline-and-lane-enforcement.md),
> which landed on `main` while this file was off it.** The roles, the prohibitions and the
> escalation route below stand verbatim and ADR-0005 restates them as the authority. The three
> words *"enforced at review"* do not: enforcement is now a CI path partition, and review is the
> second line rather than the first. ADR-0005's own supersession note anticipated this file
> landing later and says the supersession takes effect when it does — it has, so this annotation
> records it at the point of the claim rather than leaving the stale mechanism to be read as
> current. Nothing else in this section is affected.

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
