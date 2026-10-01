# Open-decision register

| | |
|---|---|
| **Status** | **9 of 15 board rows decided 2026-10-01** — see §0.1. Six remain open: **D7** is forced inside Phase 1 *if* it is answered "retain"; the other five are not forced before the Phase 2 gate, where **D5** and **D15** are the earliest. |
| **Date** | Built 2026-10-01. Board answers recorded 2026-10-01. |
| **Owner** | CEO. Reviewed by Atlas; F1–F3 and P1–P6 from that review are applied. |
| **Sources** | TDD §27 (all 22 rows), `plans/phase1-work-breakdown.md` §7 (all 7 rows), ADR-0001 through ADR-0005 |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` — filename says v2.2, content is **TDD v2.3** |
| **Issue** | VUL-8 |

**Nothing in this register blocks Phase 0 or Phase 1**, with two exceptions, both called out as
such: **D16** is needed inside Phase 1 and is Atlas's to answer, not the board's; and **D7** is
needed inside Phase 1 *if it is answered "retain"*, because that rewrites what
`authz/configured-scope` must enforce — which is the reason D7's own row argues for deciding it
early rather than deferring.

---

## 0. What this register found

The brief described "15 product and legal items plus seven more." The actual arithmetic is
better than that, and the difference is the point of gathering them:

| | Count |
|---|---|
| TDD §27 rows | 22 |
| Already settled by an accepted ADR — **struck**, §5 below | 6 |
| Live rows carried forward from §27 | 16 |
| Breakdown §7 rows | 7 |
| …of which duplicate a §27 row already counted | 5 |
| …of which an ADR already settled | 1 |
| …genuinely new | **1** (the `infra` cluster hold) |
| **Live register rows** | **17** |
| …of which the **board** owns | 15 |
| …of which an **agent** owns outright — engineering questions in disguise | 2 |

**D5** and **D15** are joint — the board answers the default, Atlas files the supporting
ADR — and are counted with the board. Only **D16** (Atlas) and **D17** (CEO via T2) leave
the board's queue entirely.

The board was asked for **15 decisions** as a single batch, with seven of the fifteen closable by
accepting a recommended default in this document without further research. **Nine were answered on
2026-10-01**: D1, the seven defaults as a block, and D8. Six remain open — D3, D4, D5, D7, D12,
D15 — and the earliest forcing point among them is the **Phase 2 gate** (D5 and D15). They are
listed with their forcing points in §2.1.

Breakdown §7 contributed exactly one new decision. That is worth stating plainly: the seven
"more" items were almost entirely the same product questions §27 already asked, restated in
Phase 1 terms. Reading them as a separate queue would have doubled the apparent decision
load.

## 0.1 Decision log — board answers of 2026-10-01

Recorded verbatim in substance from the board's answer to the VUL-8 decision batch. Each row's
full reasoning stays in the numbered section it belongs to; this is the index.

| # | Decision | Effect |
|---|---|---|
| **D1** | **No cluster.** *"The Aether cluster does not exist and it makes no sense to create one before having any deliverables."* **Option D adopted.** | The §1.1 question is answered **negatively**: there is no Aether environment. `infra#1` stays unmerged. R1–R6 are carried as named risk with the owners in §1.3. Re-decided at the **Phase 2 gate**, not before. Consequences in §1.6. |
| **D2** | §24.3's stated cut line **is** the GA scope. The date is an output of the program plan, not an input. | T2 (`plans/program-plan.md`) derives the date. No separate GA-scope decision remains. |
| **D6** | **Observe AAAA, scan IPv4 only, disclose untested IPv6.** §5.9 moves from `[PROPOSED — owner review]` to adopted. | `report/matrix` coverage wording must say "resolved AAAA, not scanned". §24.4's "adopt explicitly or state the limitation" is discharged by adopting. |
| **D9** | **§26's retention defaults stand** — 30-day recovery window, raw output 30d, exports 7d, reports for account life. Erasure beats locked evidence except under a documented legal hold. A restore must replay the deletion log before restored data is reachable. | `deletion/all-stores-restore` and `recovery/restore` keep the shape they were already written to. Policy is configuration, not code. |
| **D10** | **Authenticated download only at GA.** No external transactional-email provider. Public report sharing stays off by default and post-GA. | Ratifies §26 and §24.3. No vendor data-processing review is needed for GA. §0.0 rule 8 / §18.1 boundary is not crossed. |
| **D11** | **§26's report defaults and §16.10's budgets stand.** Comparable coverage is defined as **"resolved and scanned"**, with resolved-but-unscanned — IPv6, blocked, skipped — stated separately rather than folded in. | Ties to D6 by construction. `report/matrix` and `report/cross-consistency` can be written against fixed numbers. |
| **D13** | **Phase 5 AI deferred entirely** to the Phase 5 gate. | GA is "usable operation with AI disabled" per §24.3. The §18 crate-level AI→dispatcher ban and `ci/crate-boundaries.sh` are what make the deferral safe. |
| **D14** | **Optional-feature package deferred.** The carve-out resolves to its default: **Amass is not in the Phase 2 pinned scanner set.** | §24.1's walking skeleton stays `subfinder → dnsx → httpx → nuclei`. `supply-chain/check-catalog`'s pinned-image list is fixed for Phase 2. |
| **D8** | **Option (b), not the recommended (a): ship a conservative internally-drafted disclaimer / ToS / permission-attestation text and get it reviewed before GA.** Outside counsel is not commissioned now. | Removes the only external lead time on the register. The residual is recorded in D8's row: the pre-GA review is now the single control standing between an internal draft and a liability position, so it is a gate, not a courtesy. |

**Still open: D3, D4, D5, D7, D12, D15.** Six rows, all board-owned, all pricing, billing-edge,
scope or liability-posture judgements where this register deliberately offered no default. §2.1
lists them with the phase that forces each. Nothing in that set blocks Phase 0 or Phase 1 except
D7-if-retained.

---

## 0.2 A material finding, outside this register's scope but blocking its citations

Four documents this register cites are **not on `main`**. They are in closed pull requests
whose branches have been deleted:

| Document | Lives in | State |
|---|---|---|
| `plans/phase1-work-breakdown.md` — the §7 cited throughout | [`docs#22`](https://github.com/vulcanflow/docs/pull/22) | closed, unmerged, branch deleted |
| `ADR-0004` — workspace repository | [`docs#23`](https://github.com/vulcanflow/docs/pull/23) | closed, unmerged, branch deleted |
| `ADR-0002` amendment **A2** — §3.7 pin corrections, OIDC pins | [`docs#23`](https://github.com/vulcanflow/docs/pull/23) | closed, unmerged, branch deleted |
| TDD v2.3 filename rename + hub README | [`docs#20`](https://github.com/vulcanflow/docs/pull/20) | closed, unmerged, branch deleted |

The content is recoverable — GitHub retains the commit objects at each PR's head SHA
(`93c79f8`, `cee1edc`, `5b3405c`) and this register was written from them. It is not lost. But
until those PRs are restored, every `§7.x` and `ADR-0004` citation below points at something a
reader cannot open, and **T1 cannot do what it was asked to do**: three of its four landable
PRs no longer have branches. Restoring them belongs to T1, not here; flagged there too.

`ADR-0002` on `main` is the **pre-A2** text. Its §7 risk table R1–R6 is identical in both
versions, so D1 below is unaffected — verified twice, independently, by extracting §7 from both
revisions.

The consequence beyond citations is worth stating, because it is larger than a broken link:
**§3.7 is not currently in the design of record, and the workspace is already built against it.**
`platform#1`'s root `Cargo.toml` takes its pins from §3.7 and declares *"where §3.1–§3.2 and §3.7
disagree, §3.7 is the authority"* — an authority that exists only in an unmerged amendment. T1's to
restore; recorded here as a consequence, not as a decision.

---

## 1. D1 — The `infra` cluster go/no-go

This is the only item on the register that is costing anything today, and the only one whose
answer changes what the next two phases can prove. It gets its own section.

| Field | |
|---|---|
| **Item** | Do we stand up a Kubernetes environment for Phase 1, and if so which one? |
| **Source** | Breakdown §7.7 (the only genuinely new item in that section). Risk register in ADR-0002 §7 (R1–R6). The §27 item 5 residue that ADR-0002 and ADR-0003 explicitly did **not** close. |
| **Blocks** | Nothing in Phase 0 or Phase 1 *delivery*. It blocks **verification**: six named risks, **nine §25 identifiers and two ADR-derived confirmations**, plus three areas that have no test at all (table in §1.3). `infra#1` stays unmerged. |
| **Needed by** | Phase 2 forces a partial answer (secureCodeBox operator must run somewhere). Phase 4 forces a complete one (`recovery/restore`, `deletion/all-stores-restore`, `release/analysis`, `abuse/egress-stop` have no non-cluster form at all). |
| **Owner** | Board. |
| **DECIDED 2026-10-01** | **No cluster — Option D.** *"The Aether cluster does not exist and it makes no sense to create one before having any deliverables."* The existence question in §1.1 is answered **no**. Re-decided at the Phase 2 gate. Consequences in §1.6. |

### 1.1 The question is not the one the brief assumed — and the answer is "it does not exist"

The brief frames this as "the cost is recurring," which assumes we are buying a cluster. Two
documents in the project assert otherwise — that **Aether** is an *existing, already-approved,
self-operated Kubernetes platform*:

- TDD §24.2, Phase 0 deliverables: *"**Existing** approved Aether stack."*
- TDD Appendix C: *"Approved self-operated Kubernetes platform."*

Two others are weaker than they look and should not be read as confirmation. TDD §0.0's
technology row approves Aether, Kubernetes, Harbor, Argo CD and Kargo, but it is a **technology**
approval and says so in the same cell: *"Deployment versions, configuration, and compatibility
still require engineering verification."* The §9 header — *"Platform: Aether (self-operated
Kubernetes platform — not AWS)"* — states an architecture, not an installation.

And the ADRs disclaim existence explicitly, which is stronger than silence:

- **ADR-0002 §6**: *"It does **not** claim anything is installed; Harbor mirroring and cluster compatibility are cluster work and appear as risk R6 in §7."*
- **ADR-0002 §4** scope: the probe work had *"no `cargo`, no `rustc`, and no container runtime."* No container runtime means no cluster.
- **TDD §21.3**, quoted in ADR-0002 §6: *"Do not treat upstream release existence as proof of installation."*
- `infra#1`'s own README: GitOps *"against a **preinstalled** Argo CD/Kargo control plane"*, and — the real sentence, which never names Harbor — *"Argo CD, Kargo, and the container registry are Aether-provided and are not installed from this repository."*

So `infra#1` is not a cluster build. It is 811 lines of Kustomize overlays, operator CRs and
Helm *placeholders* that assume a control plane already exists and that someone will fill in
the destinations. Its chart versions are marked `UNCONFIRMED` and its ApplicationSet is
deliberately kept out of the apply paths *"until destination inputs are confirmed."*

**Which meant the board's first answer was a fact, not a judgement, and it cost nothing to
give:** does an Aether environment exist today, and do we have a namespace and credentials on
it? No agent had ever confirmed it, and no document in the design of record does.

> **Answered 2026-10-01: it does not exist.** The project has been designed against a platform
> that is not there.

There is already a live instance of this exposure compiled into the workspace, which is what makes
it concrete rather than a phasing argument. `platform#1`'s root `Cargo.toml` pins
`k8s-openapi = "=0.28.0"` with `features = ["v1_32"]`, under the manifest's own comment: *"if
Aether's control plane is older than 1.32, this pair is wrong outright — that is CEO's to
confirm."* With no control plane to confirm against, that pin is now an **assumption with no
owner who can discharge it** rather than a question awaiting an answer. §1.6 routes it.

That single fact selected between the four options below. It eliminated A and B outright.

### 1.2 Options and cost

Figures are **order-of-magnitude, derived from the footprint the design requires, and need a
real quote before anyone commits to them.** The arithmetic is shown so it can be corrected
rather than argued about.

**Option A — Aether exists and has capacity. Confirm destinations and merge `infra#1`.**

| Line | |
|---|---|
| Recurring | **~$0 incremental.** Capacity already paid for; this is a namespace and quota allocation. |
| One-time | ~2–4 agent-days to confirm Kubernetes minor, registry hostname, GitOps destinations, S3 endpoint and secret-store wiring, then unmark the `UNCONFIRMED` pins. |
| Settles | R1–R6 and everything in §1.3, at full strength. |
| Risk | The Kubernetes minor may not be in secureCodeBox v5.9.0's compatibility window (that *is* R6). Finding out is the cheapest thing on this list. |

**Option B — Aether exists but has no spare capacity for a dev environment.**

| Line | |
|---|---|
| Recurring | 3–4 additional arm64 worker nodes at ~8 vCPU / 32 GB: **~$400–550/month** (~$5–7k/year), plus ~1 TB block storage for CNPG's 3-instance HA and ClickHouse at ~$100/month. **≈ $500–650/month (~$6–8k/year).** |
| One-time | As Option A, plus node provisioning. |
| Settles | Same as Option A. R5 (arm64 reproducibility *on Aether nodes*) is settled only if the new nodes are the same platform. |

**Option C — No Aether environment exists. Provision a Phase-1 `dev` cluster.**

| Line | Monthly |
|---|---|
| 3 × control plane, 4 vCPU / 8 GB, arm64 | ~$135 |
| 4 × worker, 8 vCPU / 32 GB, arm64 | ~$520 |
| Block storage, ~1 TB (CNPG HA ×3, ClickHouse, Valkey) | ~$100 |
| S3-compatible object storage, ~500 GB + egress | ~$15 |
| Load balancer / ingress | ~$15 |
| **Total** | **≈ $785/month ≈ $9.4k/year** |

Plus a one-time ~2–3 agent-weeks to install what Aether was assumed to provide and
`infra#1` explicitly does not: Argo CD, Kargo, a signing-capable registry, and Ceph RGW.
And a standing correctness problem: **R4 and R5 become unanswerable by construction.** R4 asks
for conformance against *"the real Aether RGW deployment — its version, tuning and bucket
policy."* A cluster that is not Aether cannot answer that; it only moves the risk from
"untested" to "tested against the wrong thing," which is worse because it looks green.

`dev` only. Staging and production roughly triple it (~$2.4k/month, ~$29k/year) and are a
Phase 4 decision, not this one.

**Option D — No cluster. Keep the hold. Ship Phase 1 on Testcontainers and carry R1–R6 as named risk.**

| Line | |
|---|---|
| Recurring | $0. |
| Settles | Nothing. This is the status quo, and it is a legitimate choice through Phase 1. |
| Cost | The nine §25 identifiers and two ADR-derived confirmations in §1.3 close at reduced strength, three areas stay with no test at all, and §1.3 is the list of what we ship without. |
| **Adopted** | **Yes — this is the board's decision of 2026-10-01.** |

### 1.3 What stays unverifiable without a cluster

ADR-0002 §7's six risks, mapped to what each one actually leaves weak. **This table was wrong in
the first revision of this register and was rebuilt from Atlas's review (F1–F3).** The errors
pointed in both directions at once — they over-valued a cluster on R1/R2/R3 and buried the one
genuinely strong argument for it — so the corrected shape matters more than the correction.

Two distinctions the first version collapsed and this one keeps:

1. **§25 identifiers and ADR-derived confirmations are disjoint sets.** Breakdown §3 splits §25
   into 27 in-scope + 18 deferred = 45; §3.3 then adds **seven ADR-derived identifiers** as a
   separate family, which is why Crucible's ledger is 34 and not 27, and why module doc comments
   read `//! §25: <id>` *or* `//! ADR: <id>`. They are not interchangeable when pricing a cluster:
   the ADR-derived set is the conditional half of the crate approval, the §25 set is product
   evidence.
2. **"The test is weak" and "there is no test at all" are different costs.** The second is worse,
   and R1 is the case of it.

| Risk | §25 identifiers left weak | ADR-derived confirmations left weak | Has no test at all | Owner |
|---|---|---|---|---|
| **R1** Admission webhook TLS — cert issuance, rotation, `caBundle` injection | — (indirect only: `authz/start-barrier-all-paths` and `execution/gates-before-start` assert gate *decisions*, not TLS plumbing) | — | **cert issuance, rotation, `caBundle` injection** | Kiln |
| **R2** Admission chain ordering and failure policy — `failurePolicy`, timeouts, mutating reinvocation | `authz/start-barrier-all-paths` (its Kubernetes paths), `execution/gates-before-start` | — | admission-chain ordering, `failurePolicy`, mutating reinvocation | Kiln |
| **R3** RLS *at realistic concurrency* through PgBouncer | `isolation/background-queries`, `isolation/all-stores` | — | — | Forge + Ledger |
| **R4** Ceph RGW conformance against the **real Aether** RGW — version, tuning, bucket policy | `isolation/all-stores`, `deletion/all-stores-restore` | `storage/s3-compat-conformance` (containerised RGW ≠ Aether RGW) | — | Forge |
| **R5** arm64 build reproducibility end to end on Aether nodes | `build/rust-supply-chain`, `perf/service-baseline` | — | — | Crucible |
| **R6** secureCodeBox v5.9.0 operator against the cluster's actual Kubernetes minor; Harbor mirroring of pinned digests. **The unclosed half of §27 item 5.** | `execution/scan-identity` (**Phase 1**), `supply-chain/check-catalog` (Phase 2) | `scb/hook-invocation-contract` | Harbor digest mirroring; operator / API-server interaction | Kiln |

**Nine distinct §25 identifiers, two ADR-derived confirmations, and three areas with no test at
all.** Plus three further Phase 4 identifiers with no non-cluster form — `recovery/restore`,
`release/analysis`, `abuse/egress-stop`; `deletion/all-stores-restore` is already counted under R4.

Three corrections worth stating in full, because the first version of this table would have led the
board to buy the wrong thing:

- **The two `admission/*` tests are not weakened by R1 or R2.** ADR-0002 §4.2 says so in terms:
  `admission/cascade-gate-delete-oldobject` and `admission/workload-gate-dryrun` are driven by
  replaying recorded `AdmissionReview` JSON through the handler — *"no cluster required, which is
  why these are not in §7."* A cluster strengthens neither. Listing them over-valued a cluster for
  coverage we already have, **and simultaneously under-stated R1**, whose real cost is that cert
  issuance, rotation and `caBundle` injection have **no named test at all**. That is why ADR-0002
  §7 carries R1 as a risk with an owner instead of as a test ID.
- **The `SET LOCAL` leak is deterministic, not churn-dependent — and its test is full strength
  without a cluster.** A bare `SET` under transaction pooling leaks as soon as two transactions
  share one server connection; two connections and a borrow cycle is sufficient, and ADR-0002 §4.3
  specifies exactly that: Testcontainers stands up Postgres + PgBouncer in transaction mode and
  asserts a tenant GUC set inside `TenantTx` is *not* observable on a subsequently borrowed
  connection. So `db/tenanttx-set-local-isolation` and
  `db/pgbouncer-transaction-pooling-prepared` are **not** in the weak column. Churn changes the
  leak's frequency and blast radius and may expose additional paths — `server_lifetime` recycling,
  pool exhaustion, `DISCARD ALL` timing, re-preparation under `max_prepared_statements = 200` —
  but it is not what makes the bug appear. The first version told the board that the
  highest-consequence test in the product needed a cluster to be trusted. It does not.
- **R6 is the only risk that degrades a Phase 1 §25 identifier** — `execution/scan-identity`,
  Ledger's, in Phase 1 per breakdown §3.2. That is the single strongest cluster argument on the
  register, and the first version buried it among entries that did not belong.

**Net effect on D1:** the corrected table makes the "no cluster" decision *more* defensible, not
less. The things a cluster would have bought are mostly Phase 2 and Phase 4, and the two tests the
board would most want protected are already at full strength on Testcontainers — but the claim has
to be stated at the resolution of the tests, not of the topics, or it over-reads its own table:

- **Full strength without a cluster:** the `SET LOCAL` leak itself
  (`db/tenanttx-set-local-isolation`, `db/pgbouncer-transaction-pooling-prepared`) and the two
  admission-gate *decision* tests (`admission/cascade-gate-delete-oldobject`,
  `admission/workload-gate-dryrun`).
- **Still weak, per R3 above:** cross-tenant RLS isolation *at realistic PgBouncer concurrency* —
  `isolation/background-queries` and `isolation/all-stores`. The mechanism is proven; the
  concurrency conditions that govern the leak's frequency, blast radius and additional paths are
  not reproduced. "Cross-tenant RLS isolation is covered at full strength" would therefore be too
  broad a reading of a narrower, correct result.

### 1.4 Recommendation as put to the board

Retained as written, because the decision in §1.5 should be readable against what was actually
recommended rather than against a tidied-up version of it.

**Answer the fact first, then pick.** Specifically:

1. **Tell us whether an Aether environment exists and whether we have credentials for it.** This
   is free, it is the highest-information action available on this entire register, and no
   agent can do it. If the answer is no, a large amount of accepted design — the IPv4-only
   execution model, Ceph RGW conformance, Harbor signing, the Argo CD/Kargo promotion
   pipeline — rests on a platform that does not exist, and that is a bigger finding than this
   task.
2. **If Aether exists: Option A.** Authorise confirming destinations and merging `infra#1` at
   roughly zero recurring cost. There is no argument for Option D over Option A if A is free.
3. **If Aether does not exist: Option D through Phase 1, and decide C at the Phase 2 gate.** Do
   not spend ~$9.4k/year now to buy partial answers to R4 and R5 that a non-Aether cluster
   cannot give honestly. Phase 1 genuinely does not need it — Atlas was right to record
   *"none of these justifies standing up a cluster now"* — and Phase 2 is the first point
   where the secureCodeBox operator must run somewhere, which is a concrete trigger rather
   than a guess.

Either way `infra#1` stays unmerged until the board answers, and R1–R6 stay open with the
owners named above. That is already the recorded position in ADR-0002 §7 and T3's scope; this
register does not change it, it prices it.

### 1.5 Decision

**Taken 2026-10-01. Option D — no cluster.** In the board's words: *"The Aether cluster does not
exist and it makes no sense to create one before having any deliverables."*

That is both halves of the question answered in one sentence. The fact: **there is no Aether
environment.** The judgement: **do not buy a substitute for one until there is something to run on
it.** Recommendation 3 above anticipated the second half; the first half it could only ask for.

The position is defensible on this register's own evidence, and the corrected §1.3 strengthens it:
Option C's ~$9.4k/year buys a cluster that *cannot* answer R4 or R5 honestly, and the two highest-
consequence tests a board would want protected — the `SET LOCAL` tenant-context leak and the
admission-gate decision tests — are already at full strength on Testcontainers. RLS isolation *at
realistic PgBouncer concurrency* is the part that stays weak (R3), and it is carried as named risk
rather than claimed as covered. The forcing point is **Phase 2**, where
the secureCodeBox operator has to run somewhere. "Deliverables first" and "decide at the Phase 2
gate" are the same instruction.

### 1.6 Consequences of the decision, and who owns each

The decision is clean. Its consequences are not all inside this register, and they should not be
discovered later one at a time.

| Consequence | Owner |
|---|---|
| **`infra#1` stays unmerged** and is now held by a recorded decision rather than an open question. Its `UNCONFIRMED` pins and withheld ApplicationSet are correct as they stand — there are no destinations to confirm. | CEO / T3 |
| **R1–R6 are carried as accepted named risk**, not as pending work, with the owners in §1.3. The Phase 2 gate inherits them. | Atlas, in ADR-0002 §7 |
| **Two TDD statements are now known to be false.** §24.2's *"Existing approved Aether stack"* and Appendix C's *"Approved self-operated Kubernetes platform"* assert an installation that does not exist. The design of record should say what is true: Aether is the approved *target* platform, not a running one. | Atlas — TDD correction |
| **`platform#1` pins `k8s-openapi "=0.28.0"` / `v1_32` against a control plane that does not exist.** The manifest comment makes it CEO's to confirm; with no cluster there is nothing to confirm against. It is not wrong today — nothing runs against an API server — but it is an undischargeable assumption sitting in a dependency pin, and the first cluster we ever stand up is where it bites. It should be labelled as an assumption in the manifest rather than as a pending question. | Atlas / Crucible |
| **Phase 0's "existing approved Aether stack" deliverable cannot be delivered as written.** T3's Phase 0 scope needs its Aether line restated as "deferred to the Phase 2 gate", or Phase 0 cannot close. | CEO / T3 |
| **The Phase 2 gate acquires a mandatory agenda item**: re-decide D1 with Option C's price and §1.3's corrected list in front of it, including whether a non-Aether cluster is acceptable given that it cannot answer R4 or R5. | CEO, at the Phase 2 gate |

None of these is a reason to revisit the decision. They are the bill for it, and it is a smaller
bill than ~$9.4k/year for answers the cluster could not give.

---

## 2. Live board decisions

Seven of these carried a recommended default the board could accept as-is; **all seven were
accepted as a block on 2026-10-01** and each row below records it. The remaining rows need a real
answer, and the **Needed by** column is what determines when.

### 2.1 What is still open, and when it is forced

Six rows. All board-owned. All are pricing, billing-edge, scope or liability-posture judgements
where this register deliberately offered no default, because a recommendation on any of them would
have been a guess dressed up as advice.

| # | Open question | Forced by | Cost of waiting |
|---|---|---|---|
| **D5** | Billing edges — does a work unit settle all-or-nothing, and which period does a boundary-spanning scan belong to? | **Phase 2** | Highest of the six. Breakdown §7.2 names the real consequence: without it we may accept scanner batches that cannot report success at the configured unit granularity. |
| **D15** | Is VulcanFlow willing to publish a *"confirmed takeover"* label at all before GA? | **Phase 2** | Blocks VUL-14 (Atlas's evidence-bar ADR). The conservative default is already in force either way — see D15's row; this is a ratification, not a vote. |
| **D7** | Standalone IP/CIDR registration — retain or drop for GA? | Phase 1 **if retained** | Deferring is actively expensive, and this is the one row where that is true. A late "retain" reopens D4 and rewrites `authz/configured-scope` after both have closed. |
| **D4** | Target accounting — registered slots or distinct scanned targets? | Phase 4 | Low. Slot-accounting policy sits behind one named function by design, so the change lands in one place. |
| **D3** | Package names, limits, entitlements, ceilings | Phase 4 | Low. Phase 1's allowance tests are parameterised over a configured limit and pass without the commercial numbers. |
| **D12** | Egress service — abuse ownership, rate limits, depeering contingency, KYC operations | Phase 4, **except abuse ownership** | Naming an abuse owner is free and is the part that fails at the worst moment. The rest is Phase 4 operational tuning. |

### D2 — Launch date and final GA feature cut

| Field | |
|---|---|
| **Source** | TDD §27 item 1 |
| **Blocks** | No §25 identifier. It blocks the §24.3 GA cut line, and therefore sequencing decisions in Phases 3b and 4. |
| **Needed by** | Phase 3b planning. Not before. |
| **Options** | (a) Fix a date now and cut scope to fit. (b) Fix the scope at §24.3's stated cut and let the date follow. (c) Defer both to the Phase 3a gate. |
| **Recommendation** | **(b).** §24.3 already defines a defensible cut line and names its fast-follows. Adopting it as the GA scope converts D2 from two unknowns into one, and the date becomes an output of the program plan (T2) rather than an input to it. |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(b) accepted.** §24.3's cut line is the GA scope. The launch date is a T2 output derived from it, not a board input. |

### D3 — Package names, limits, entitlements and ceilings

| Field | |
|---|---|
| **Source** | TDD §27 item 2 |
| **Blocks** | Phase 4 (`usage/lago-stripe-replay`) and enabling paid use. **Not** the Phase 1 allowance tests — `usage/unit-definition`, `usage/remaining-work-notification` and `usage/concurrent-reservations` are parameterised over a configured limit and turn green without knowing the commercial numbers. |
| **Needed by** | Phase 4. |
| **Options** | (a) Decide the full commercial matrix now. (b) Decide only the *shape* now — that limits exist, are per-billing-period, and are enforced at reservation time — and set the numbers at the Phase 4 gate. (c) Defer entirely. |
| **Recommendation** | **(b).** The shape is already implied by §17 and by the Phase 1 tests being written against it. The numbers are a pricing exercise with no engineering dependency, and deciding them now means deciding them with no usage data. |
| **Owner** | Board |
| **Status** | **Open.** Not in the 2026-10-01 batch — no default offered. Forced at Phase 4. |

### D4 — Target accounting: registered slots vs distinct scanned targets

| Field | |
|---|---|
| **Source** | TDD §27 item 3 **and** breakdown §7.1 — the same decision, stated twice |
| **Blocks** | Paid use, Phase 4. `usage/unit-definition`, `usage/remaining-work-notification`, `usage/lago-stripe-replay`. |
| **Needed by** | Phase 4. The breakdown already contains the mitigation: slot-accounting policy sits behind one named function (breakdown §7.1's own note), so this decision changes one place. |
| **Options** | (a) **Active registered targets** — a slot is consumed by registration and freed by removal. Predictable for the customer, invites slot-churn gaming. (b) **Distinct scanned hostnames per period** — tracks real cost, but the customer cannot see their bill coming. (c) Hybrid: registered slots cap concurrency, scanned hostnames meter usage. |
| **Recommendation** | **(a), with removal not freeing a slot until the next billing period.** It is the option a customer can predict, and the churn hole closes with one rule. (b)'s unpredictability is the complaint that generates support load. Automatic discovered-target enrollment should be **opt-in per target**, which §5.3's apex-discovery opt-in already establishes as the pattern. |
| **Owner** | Board |
| **Status** | **Open.** Not in the 2026-10-01 batch. Forced at Phase 4. |

### D5 — Billing edges: period attribution, mixed per-unit outcomes, post-settlement corrections

| Field | |
|---|---|
| **Source** | TDD §27 item 4 **and** breakdown §7.2 — the same decision, stated twice |
| **Blocks** | Breakdown §7.2 is precise about the real consequence: accepting scanner batches that cannot report success at the configured unit granularity — **Phase 2**. Then `usage/failure-retry-settlement` and `usage/lago-stripe-replay` in Phase 4. |
| **Needed by** | **Phase 2** — the earliest forcing point on this register after D1. |
| **Options** | (a) A work unit settles all-or-nothing; a batch with partial scanner success consumes nothing. (b) Per-target settlement within a batch; partial success consumes partially. (c) Settle optimistically and correct after the fact. |
| **Recommendation** | **(a).** §17.3 already guarantees failed units consume zero, and (a) is the only option consistent with that without new machinery. (c) requires audited post-settlement corrections, which is the hardest thing in item 4 and the one most likely to produce a customer-visible billing error. If a scanner cannot report at unit granularity, that is a reason to reject the batch, not to invent a settlement rule for it. Period attribution: **attribute to the period the work *started* in**, so a scan spanning midnight on the period boundary has one unambiguous home. |
| **Owner** | Board, with Atlas on the granularity question |
| **Status** | **Open, and the most urgent of the six.** Not in the 2026-10-01 batch. Forced at the **Phase 2 gate**. |

### D6 — AAAA observation: adopt explicitly, or state an IPv4-only limitation

| Field | |
|---|---|
| **Source** | TDD §27 item 7 **and** breakdown §7.5 — the same decision, stated twice. §5.9 carries it as `[PROPOSED — owner review]`; §24.4 requires it be adopted explicitly or omitted with an honest coverage statement. |
| **Blocks** | Report coverage claims. `execution/ipv4-only` (its AAAA cases) and `report/matrix` coverage wording. The breakdown notes `execution/ipv4-only` encodes the current behaviour **either way**, so this does not block writing the test. |
| **Needed by** | Phase 3b (report coverage). Cheaper to decide now because it is one sentence. |
| **Options** | (a) **Observe AAAA, scan IPv4 only, disclose untested IPv6** — §5.9's proposal. (b) Do not observe AAAA at all; state a flat IPv4-only limitation. |
| **Recommendation** | **(a).** Aether is IPv4-only, so neither option adds scanning capability — the only difference is whether we can *tell the customer about the gap*. Under (b), a customer's IPv6-only host is silently never scanned and the report cannot say so. Under (a), the report states "resolved AAAA, not scanned," which is the honest artefact and costs one extra DNS record type and one report sentence. Honesty about coverage is the whole argument for naming this in §24.4. |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(a) accepted.** Observe AAAA, scan IPv4 only, disclose untested IPv6. §5.9 moves from `[PROPOSED — owner review]` to adopted, and §24.4's "adopt explicitly or state the limitation" is discharged by adopting. Reports must say *"resolved AAAA, not scanned."* Note after D1: *"Aether is IPv4-only"* was the premise, and Aether does not exist — but the decision is unaffected, because (a) is the more honest artefact on any platform and (b)'s only merit was saving one DNS record type. |

### D7 — Standalone IP/CIDR registration

| Field | |
|---|---|
| **Source** | TDD §27 item 8 |
| **Blocks** | Standalone IP/CIDR features. `authz/configured-scope`, `usage/unit-definition`. Domain-derived IPv4 scanning is already approved and unaffected. |
| **Needed by** | Not on the GA path if dropped. Phase 1 if retained, because it changes what `authz/configured-scope` must enforce. |
| **Options** | (a) Retain direct IP/CIDR registration, and define target accounting and an authorization-attestation path for it first. (b) Drop it for GA; domain-derived IPv4 only. |
| **Recommendation** | **(b), and decide it now rather than later.** This is the one deferrable item where deferring is actively expensive: §27 item 8 says *"if yes, explicitly define target accounting/scope **before** enabling"* — so a late yes reopens D4 and `authz/configured-scope` after both are closed. The substantive problem is that ownership of a CIDR cannot be proven by the DNS-based challenge in §5.2; it needs a different attestation path, which is real Phase-4 work. Drop it, and let customer demand reopen it post-GA. |
| **Owner** | Board |
| **Status** | **Open.** Not in the 2026-10-01 batch — a scope decision, not a default to ratify. **This is the one open row with a Phase 1 cost**: Scribe is writing `authz/configured-scope` now, and a late "retain" rewrites it on a test-only PR. Cheapest to answer next. |

### D8 — Legal: disclaimer, ToS, permission-attestation wording; attestation expiry; manual-review service level

| Field | |
|---|---|
| **Source** | TDD §27 item 9, and the disclaimer half of breakdown §7.4 |
| **Blocks** | `report/disclaimer-framing` (Phase 3b — the breakdown names it as blocked on this decision specifically). The KYC half of `authz/approval-tracks`, i.e. Track B. §16.7's versioned disclaimer in the base template. |
| **Needed by** | Phase 3b for the disclaimer. Track B at GA per §24.3's fast-follow list. |
| **Options** | (a) Commission outside counsel now. (b) Ship a conservative internally-drafted placeholder and get it reviewed before GA. (c) Defer to Phase 3b. |
| **Recommendation** | **(a) — and this is the item to start today even though it is needed last.** It is the only decision on the register with a dependency outside the company and therefore the only one with a lead time we do not control. Everything else here is a judgement the board can make in an afternoon. A security scanner's permission-attestation wording is also the document that decides whether an unauthorised scan is our liability or the customer's, which is not a thing to draft internally and hope. §16.7 already puts the disclaimer under a version, so counsel's text drops into a slot that exists. |
| **Owner** | Board + outside counsel |
| **DECIDED 2026-10-01** | **(b), against the recommendation: ship a conservative internally-drafted text and have it reviewed before GA.** Outside counsel is not commissioned now. Recorded as a deliberate choice, not an oversight — the recommendation for (a) was argued and declined. |
| **What the decision changes** | Two things, and they are the whole residue. **(1) The pre-GA review becomes a release gate, not a courtesy.** Under (a) the liability position was counsel's work product; under (b) it is an internal draft plus one review, so that review is the only control standing between us and the posture, and it must be *named, scheduled and blocking* rather than assumed. **(2) The lead time we did not control is gone** — which is the real gain, and the reason this is a reasonable call: D8 leaves the critical path entirely and nothing on the register now depends on an external party. §16.7's versioned disclaimer slot means a later counsel review replaces the text without a schema change, so (b) is reversible into (a) at any point before GA at no engineering cost. |
| **Scope of the draft** | Four artefacts, all conservative: the report disclaimer (§16.7, versioned); ToS; the permission-attestation wording customers accept before a scan; and attestation expiry. Manual-review service level for Track B stays with §24.3's fast-follow list. The drafting standard is **"assume it will be read adversarially after an unauthorised scan"** — that is the scenario the wording exists for. |

### D9 — Retention and deletion policy

| Field | |
|---|---|
| **Source** | TDD §27 item 10, and the retention half of breakdown §7.4 |
| **Blocks** | `deletion/all-stores-restore` and `recovery/restore` (both Phase 4). §3.6 defers the policy here explicitly; §24.2 puts full isolation and recovery evidence in Phase 4. |
| **Needed by** | Phase 4. |
| **Options** | (a) Accept §26's stated defaults — 30-day recovery window; raw output 30d, exports 7d, reports for account life. (b) Set shorter retention to reduce the breach surface. (c) Longer, for customer historical comparison. |
| **Recommendation** | **(a).** §26's defaults are already written down, are internally consistent with §3.6's recovery window, and the Phase 1 mechanism (`TenantTx`, RLS) is policy-agnostic — changing the numbers later changes configuration, not code. The two genuinely open sub-questions are **locked evidence vs immediate erasure** (recommend: erasure wins, with a documented exception only where a legal hold exists) and **restored-data re-erasure** (recommend: a restore must replay the deletion log before the data is reachable — which is precisely what `deletion/all-stores-restore` tests, and the test is already written to that shape). |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(a) accepted, including both sub-questions as recommended.** §26's defaults stand: 30-day recovery window, raw output 30d, exports 7d, reports for account life. **Erasure beats locked evidence**, with a documented exception only under a legal hold. **A restore must replay the deletion log before restored data is reachable** — which is the shape `deletion/all-stores-restore` is already written to. Note after D1: that test is in §1.3's R4 row and cannot be evidenced without a cluster, so the policy is decided and its proof is deferred to the Phase 2 gate. |

### D10 — Public report sharing and external delivery

| Field | |
|---|---|
| **Source** | TDD §27 item 11, and the sharing half of breakdown §7.4 |
| **Blocks** | `report/branding-sharing` (Phase 3b/4 — the breakdown names it as blocked on this decision specifically). §16.9's share records. §21's transactional-email provider boundary. |
| **Needed by** | Phase 3b/4. §24.3 already lists public report sharing as a post-GA fast-follow. |
| **Options** | (a) Ship public sharing and an external email provider at GA. (b) **Authenticated download only at GA**; no external transactional-email provider; revisit post-GA. (c) Sharing on, email off. |
| **Recommendation** | **(b).** §26 already has public sharing *off by default* and §24.3 already lists it post-GA, so this mostly ratifies the design. The decisive point is §0.0 rule 8 and §18.1: *no tenant findings data leaves the Aether boundary without explicit per-tenant opt-in.* An external transactional-email provider is exactly that boundary crossing, and adopting one requires a vendor data-processing review we have not scoped. Authenticated download needs none. Oversized reports to non-account recipients then stops being a question. |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(b) accepted.** Authenticated download only at GA. No external transactional-email provider — so no vendor data-processing review is on the GA path. Public report sharing stays off by default and post-GA. §0.0 rule 8 / §18.1's boundary is not crossed at GA. |

### D11 — Report selection, count semantics, account-wide scope, and KPI targets

| Field | |
|---|---|
| **Source** | TDD §27 item 12 |
| **Blocks** | `report/matrix`, `report/cross-consistency` (both Phase 3b). |
| **Needed by** | Phase 3b. |
| **Options** | (a) Accept §26's defaults — 5–10 top risks, 5,000-observation render cap, explicit coverage statement, the §16.10 budgets (1,000-observation technical PDF p95 <60s; 12-month management report p95 <30s). (b) Set different counts and caps. (c) Defer. |
| **Recommendation** | **(a).** The numbers are illustrative in §26 but they are self-consistent and the budgets in §16.10 are already written as testable thresholds. The one part genuinely needing a board answer is **comparable coverage** — what we claim a report covers. Recommend tying it to D6: coverage is "resolved and scanned," with resolved-but-unscanned (IPv6, blocked, skipped) stated separately rather than folded in. That is the same honesty rule as D6 and `verify/applicable-check-evidence` already enforces it on the verification side. |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(a) accepted, including the coverage definition.** §26's defaults stand — 5–10 top risks, 5,000-observation render cap, explicit coverage statement — and §16.10's budgets become the testable thresholds for `report/matrix` and `report/cross-consistency`. **Comparable coverage means "resolved and scanned"**; resolved-but-unscanned (IPv6 per D6, blocked, skipped) is stated separately and never folded in. |

### D12 — Egress service: depeering contingency, abuse ownership, rate limits, KYC operations

| Field | |
|---|---|
| **Source** | TDD §27 item 14 |
| **Blocks** | `abuse/egress-stop` (Phase 4). The KYC half of Track B. §20.3–20.5's controlled pool and the §21 <60s kill target. |
| **Needed by** | Phase 4. |
| **Options** | (a) Decide the whole operational package now. (b) **Name an abuse owner now; defer rate limits, depeering contingency and KYC operations to Phase 4.** (c) Defer entirely. |
| **Recommendation** | **(b).** Three of the four parts are Phase 4 operational tuning with no Phase 1 or 2 dependency. **Abuse ownership is different and should be settled now, because it is free and because it is the one that fails at the worst moment.** A scanning platform will receive an abuse complaint, and the cost of not having a named recipient is measured in hours of upstream-provider patience. Name a person, not a role, with a mailbox that is monitored. |
| **Owner** | Board |
| **Status** | **Open.** Not in the 2026-10-01 batch — naming a person is the board's, not a default to ratify. The three Phase 4 parts can wait; **abuse ownership should not**, and it is one line. |

### D13 — Phase 5 AI: external-data consent, commercial entitlements, serving and GPU configuration

| Field | |
|---|---|
| **Source** | TDD §27 item 15 |
| **Blocks** | Phase 5 only. No §25 identifier — Phase 5 sits beyond the §25 matrix, and `remediation/version-tier-no-ai` exists to guarantee guidance works with AI disabled. |
| **Needed by** | Phase 5. §24.3 puts GA at "usable operation with AI disabled." |
| **Options** | (a) Decide now. (b) Defer entirely to a Phase 5 gate. |
| **Recommendation** | **(b), with no further discussion.** This is the clearest defer on the register and the structural reason is already enforced in code: §18's **crate-level ban on an AI→dispatcher dependency** means Phase 5 cannot leak backwards into the execution path, and `build/rust-supply-chain` plus the `ci/crate-boundaries.sh` gate are the detectors. Deciding GPU configuration four phases early would be guessing at hardware for a workload we have not specified. |
| **Owner** | Board, at the Phase 5 gate |
| **DECIDED 2026-10-01** | **(b) accepted.** Phase 5 AI deferred entirely to the Phase 5 gate. GA is "usable operation with AI disabled" per §24.3, and §18's crate-level AI→dispatcher ban plus `ci/crate-boundaries.sh` are what keep the deferral from leaking into the execution path. |

### D14 — Optional features: Amass tier, bounty-feed licensing, branding rollout, design tokens

| Field | |
|---|---|
| **Source** | TDD §27 item 16 |
| **Blocks** | Optional features only. `report/branding-sharing`, and `supply-chain/check-catalog` if Amass joins the pinned scanner set. §27 notes technology adoption is already approved — these are packaging and operational choices. |
| **Needed by** | Phase 2 if Amass is in the Phase 2 scanner images; otherwise post-GA. |
| **Options** | (a) Decide the package now. (b) Defer, with one carve-out: confirm before Phase 2 whether Amass is in the pinned scanner set, because that changes `supply-chain/check-catalog`'s pinned-image list. |
| **Recommendation** | **(b).** The §24.1 walking skeleton fixes the pipeline at `subfinder → dnsx → httpx → nuclei`, which does not include Amass — so the default is "not in Phase 2" and the carve-out resolves itself unless the board says otherwise. Bounty-feed licensing, branding rollout and design tokens are all post-GA by §24.3. |
| **Owner** | Board |
| **DECIDED 2026-10-01** | **(b) accepted, and the carve-out resolves to its default: Amass is *not* in the Phase 2 pinned scanner set.** §24.1's walking skeleton stays `subfinder → dnsx → httpx → nuclei`, so `supply-chain/check-catalog`'s pinned-image list is fixed for Phase 2. Bounty-feed licensing, branding rollout and design tokens are post-GA. |

### D15 — Confirmed-dangling-resource evidence

| Field | |
|---|---|
| **Source** | TDD §27 item 13 (owner: Engineering) **and** breakdown §7.6 (owner: CEO + Atlas) — the sources disagree on owner; the breakdown is later and more specific, so joint |
| **Blocks** | Finding classification, and therefore report content. No dedicated §25 identifier — it changes what `report/matrix` and the §15.2 guidance assert, not whether a test passes. |
| **Needed by** | Phase 2, when scanners first produce findings that would carry the label. |
| **The conservative default is already in force** | Not a board option — a given. §27 item 13 already mandates it (*"avoid treating all third-party hosting as takeover"*) and breakdown §7.6 already records it as standing behaviour: **label observed external resolution without asserting ownership or takeover.** This register's first revision put that on the board's queue as a decision; Atlas's review correctly pointed out it has no alternative branch. It is adopted, and it needs no vote. |
| **The actual board question** | Narrowed to the one thing that is genuinely the board's: **is VulcanFlow willing to publish a "confirmed takeover" label at all before GA, given that a false one is the most damaging single output this product can produce?** If **no**, Atlas's ADR has nothing to define yet and D15 leaves the queue entirely. If **yes**, the ADR defines the evidence bar and the board has said what it is buying. |
| **Owner** | Board for the one question above; Atlas for the evidence-bar ADR (VUL-14, blocked on it) |
| **Status** | **Open.** Not in the 2026-10-01 batch. Forced at the **Phase 2 gate**, when scanners first produce findings that would carry the label. |

---

## 3. Not board decisions

Two §27 rows are addressed to the board by the TDD but are not actually the board's to answer.
They are listed so the §27 ledger stays complete, and routed.

### D16 — False-positive persistence: equivalence fields and alias semantics

| Field | |
|---|---|
| **Source** | TDD §27 item 6 (owner: Engineering). **Not** in breakdown §7 — which is correct, and informative. |
| **Blocks** | `findings/fp-only-persistence` — a **Phase 1** identifier, Ledger's. |
| **Needed by** | **Phase 1.** The only row on this register needed that early *unconditionally* (D7 joins it if answered "retain"), and the only reason it is not a blocker is that it is answerable by engineering without a board decision. |
| **Why it is not a board question** | It asks for *"concrete scanner-specific equivalence fields, alias semantics, and examples proving an old decision cannot suppress unrelated findings."* That is a specification written by reading scanner output formats, which is what ADR-0002 §6 and ADR-0003 §3 already did for the secureCodeBox contract. There is no product or commercial trade-off in it. §26 already narrows the shape: a *"narrow versioned target/check/location matcher with explicit decision revocation."* |
| **Owner** | **Atlas.** Needs an ADR before Ledger can write `findings/fp-only-persistence`. Flagged to the board only because a Phase 1 test depended on it and nobody had assigned it. |
| **Status** | **Routed and in progress.** Atlas accepted it; the ADR is tracked on its own issue, with the dependent implementation and test work fanned out to Forge, Anvil, Scribe and Ledger behind it. Off the board's queue. |

### D17 — Team Rust capability and the schedule impact of the language change

| Field | |
|---|---|
| **Source** | TDD §27 item 16a (owner: Engineering leadership). Listed in §24.2 as a Phase 0 deliverable. |
| **Blocks** | The Phase 0 schedule. No §25 identifier. |
| **Needed by** | Phase 0 — nominally now. |
| **Why it is largely already answered** | Item 16a asks four things and three are settled by structure rather than by a decision. **Current experience and hiring/training plan** — the team is the nine agents, so this is not a hiring question. **Code-review ownership for async and `unsafe`-adjacent code** — **ADR-0005 §6** is the design of record and its lane table says *"Assay **and** Warren — both required."* (Not the Paperclip plan: a plan is not a decision, an ADR is.) **Async review** sits with Assay by the same table. |
| **`unsafe` — relocated, not removed** | What makes `unsafe` a build failure workspace-wide is `[workspace.lints.rust] unsafe_code = "forbid"` in the root `Cargo.toml`, inherited by all 14 member crates via `[lints] workspace = true`; `forbid` cannot be downgraded by an inner `allow`. `ci/forbid-unsafe.sh` is belt-and-braces by its own header — it greps crate roots and compiles nothing, so citing it as the mechanism was wrong even though the conclusion holds. More important for §27 item 16a, which asks about `unsafe`-***adjacent*** code: `forbid(unsafe_code)` binds **our** crates only. Every dependency ships `unsafe`, and the pinned TLS provider is `aws-lc-rs` — an FFI binding over a C crypto library. So "no `unsafe` to own" is true of code we write and false of code we ship: ownership is **relocated** to the supply-chain gate (`cargo-deny`, `cargo-audit`, SBOM, `build/rust-supply-chain`), not eliminated. |
| **Rider** | **Withdrawn 2026-10-01.** This row used to read: *"ADR-0005 §6 also records that `coderabbit:review` is not installed, so the review lane currently has **one** reviewer and merges wait. Review ownership is answered but degraded."* That was true when written and is false now — the CodeRabbit GitHub App (`coderabbitai`, app id `347564`) has been installed on the organisation since `2026-10-01T19:12:07Z`, so lane 6 has **both** reviewers: Assay by hand, and Warren triaging the App's review of the pull request. ADR-0005 §6.1, amendment 2. Review ownership is answered and **not** degraded. Note what replaces the degradation as the live hazard, because it is the one this row would otherwise leave a reader unprepared for: a `coderabbitai[bot]` thread is *input* to lane 6, not satisfaction of it, so a pull request carrying a detailed bot thread and no recorded Warren verdict is **unreviewed** (ADR-0005 §6.2). |
| **Residue** | The **schedule impact** of the language change, which is a program-plan output. |
| **Owner** | **CEO, via T2** (`plans/program-plan.md`). Removed from the board's queue; no board input needed. |

---

## 4. The one real dependency between rows — and how the decisions landed on it

D1 is not just the largest item, it gates the honesty of three others. D9, D10 and D11 all turn
on what we can truthfully claim about coverage and deletion, and two of the identifiers that
would substantiate those claims — `deletion/all-stores-restore` and
`storage/s3-compat-conformance` — are in §1.3, with `recovery/restore` among the Phase 4 three
that have no non-cluster form at all. (`storage/s3-compat-conformance` is an ADR-derived
confirmation rather than a §25 identifier; see §1.3's first distinction.)

**Both halves happened on 2026-10-01, and the register should be read with that in mind:** the
board adopted the recommended defaults for D9, D10 and D11, *and* declined the cluster that would
evidence them. That is coherent — the policies are decided and their proof is deferred to the
Phase 2 gate — but it means **three GA-relevant claims now rest on documented intent rather than
on a passing test.** D10 is the one that makes this comfortable rather than uncomfortable: by
choosing authenticated download only, with no external email provider and no public sharing, the
board shrank the surface of claims that need evidencing at GA to roughly the set we can evidence
without a cluster.

Everything else on this register is independent and could be answered in any order.

---

## 5. Struck — settled by an accepted ADR

These six §27 rows are closed. They are recorded here with the ADR that closed each so the
board is not asked a question it has already answered, and so a reader of §27 — which still
lists them as open — can see why.

| §27 | ~~Question~~ | Settled by | Decision |
|---|---|---|---|
| **17** | ~~Approve the §2.5.2 crate set and pin versions; utoipa OpenAPI 3.1; kube-rs admission coverage; sqlx behind PgBouncer; S3 client vs Ceph RGW/RustFS~~ | **ADR-0002** (`Closes` §27 item 17), amended A1 (§3.6 crypto/TLS/encoding pins) and A2 (§3.7 feature corrections, `k8s-openapi v1_32`, `jsonwebtoken` pinned, `openidconnect` rejected, `utoipa-swagger-ui` withdrawn) | Crate set approved and pinned. Supersedes the `[PROPOSED]` table in §2.5.2. The four probes in §4 became required Phase 1 tests with named owners, not preconditions. |
| **18** | ~~Inventory Go code written against v2.0–v2.2; rewrite-now vs bounded coexistence~~ | **ADR-0001** (`Closes` §27 item 18) | `vf-api` PR #1 closed unmerged. 10 files, 425 additions, 56 executable lines, zero safety-relevant logic. No coexistence period, no Go on any default branch. |
| **19** | ~~Is a typed streaming RPC channel needed beyond REST + SSE?~~ | **ADR-0003** §2 | **No channel adopted.** REST + SSE is the complete GA surface. Post-GA choice pre-decided by §2.4's criteria if demand appears; revisit trigger recorded. |
| **20** | ~~SCB parser and hook language; the hook invocation contract~~ | **ADR-0003** §3 | Parsers stay on the upstream JavaScript parser SDK, including custom ones. **Hooks are Rust**; the contract `vf-hook-notify` must satisfy is recorded verbatim in §3.3 from secureCodeBox `v5.9.0`. |
| **21** | ~~Lago and Stripe client approach in Rust~~ | **ADR-0003** §4 | **Thin hand-written typed clients** scoped to §17.5's endpoints, provider OpenAPI specs pinned as source of truth with a drift test. Webhook signature verification in-house with RustCrypto, to the schemes in §4.4. |
| **5** | ~~First automatic cascade: SCB release/digests, node compatibility, scan-ID field, hook/CRD inputs, candidate reservation and start barrier~~ | **ADR-0002** §6 (release, digests, arm64, CRD type generation — `Also settles` the engineering half) + **ADR-0003** (hook contract in §3.3, **and the exact scan-ID field: the scan fingerprint is `metadata.uid` of the `Scan` object**) | Struck **except its cluster half** — node Kubernetes-minor compatibility and Harbor digest mirroring, which is risk **R6** and folded into **D1**. ADR-0003 §5 says so explicitly: *"it needs a cluster, and that hold is CEO's to lift."* Candidate reservation and the start barrier are Phase 1 tests (`authz/start-barrier-all-paths`, `execution/late-cascade-barrier`), not open decisions. Limb by limb: release/digests ✅, scan-ID field ✅, hook/CRD inputs ✅, node K8s-minor **open → R6**, Harbor mirroring **open → R6**, reservation/start barrier ✅ discharged by a named test. |

**ADR-0004** closed no §27 item — it is a repository-topology decision and says so in its own
header. It is cited here only because it is one of the four documents currently off `main` (§0).

Breakdown **§7.3** — *"Lago/Stripe client decision (§27 item 21, ADR-0003)"* — is likewise
**struck**: ADR-0003 §4 decided the client shape. What remains of that row is commercial
policy, which is D3 and D5. The row was stale when written.

---

## 6. Coverage ledger

Every source row, accounted for exactly once, with its state as of 2026-10-01. This table is the
acceptance criterion.

### TDD §27 — 22 rows

| §27 | Disposition | State |
|---|---|---|
| 1 | D2 | **Decided** |
| 2 | D3 | Open — Phase 4 |
| 3 | D4 (merged with breakdown §7.1) | Open — Phase 4 |
| 4 | D5 (merged with breakdown §7.2) | Open — **Phase 2** |
| 5 | **Struck** — ADR-0002 §6 + ADR-0003; cluster residue (R6) → D1 | Struck |
| 6 | D16 — Atlas, not board | Routed, in progress |
| 7 | D6 (merged with breakdown §7.5) | **Decided** |
| 8 | D7 | Open — Phase 1 if retained |
| 9 | D8 (merged with breakdown §7.4, disclaimer half) | **Decided** — internal draft, reviewed before GA |
| 10 | D9 (merged with breakdown §7.4, retention half) | **Decided** |
| 11 | D10 (merged with breakdown §7.4, sharing half) | **Decided** |
| 12 | D11 | **Decided** |
| 13 | D15 (merged with breakdown §7.6) | Default in force; narrowed question open — **Phase 2** |
| 14 | D12 | Open — Phase 4, except abuse ownership |
| 15 | D13 | **Decided** — deferred to the Phase 5 gate |
| 16 | D14 | **Decided** — deferred; Amass not in Phase 2 |
| 16a | D17 — CEO via T2, not board | Routed |
| 17 | **Struck** — ADR-0002 | Struck |
| 18 | **Struck** — ADR-0001 | Struck |
| 19 | **Struck** — ADR-0003 §2 | Struck |
| 20 | **Struck** — ADR-0003 §3 | Struck |
| 21 | **Struck** — ADR-0003 §4 | Struck |

16 live, 6 struck, 22 total. ✓ Of the 16 live: **9 decided** (D1, D2, D6, D8, D9, D10, D11, D13,
D14), **5 open** (D3, D4, D5, D7, D12), 1 part-decided (D15 — default in force, one question open),
2 routed to agents (D16, D17). That is 9 + 5 + 1 + 2 = 17 register rows against 16 live §27 rows,
because D1 is the breakdown's new row rather than a §27 one.

### Breakdown §7 — 7 rows

| §7 row | Disposition |
|---|---|
| Target-slot semantics (§17.2) | **Duplicate** of §27 item 3 → D4 |
| Post-settlement corrections, mixed per-unit outcomes (§17.3, §8.2) | **Duplicate** of §27 item 4 → D5 |
| Lago/Stripe client decision | **Struck** — ADR-0003 §4 |
| Disclaimer wording, public sharing, retention (§24.4, §3.6) | **Duplicate** of §27 items 9, 10, 11 → D8, D9, D10 |
| AAAA handling (§5.9, §24.4) | **Duplicate** of §27 item 7 → D6 |
| Confirmed-dangling-resource evidence (§5.3) | **Duplicate** of §27 item 13 → D15 |
| Lifting the `infra` `factory/phase0-foundation` hold | **New** → **D1** — **decided: hold stands, no cluster** |

1 new, 1 struck, 5 duplicates, 7 total. ✓

### Register total

17 live rows (D1–D17): 15 board-owned (D5 and D15 jointly with Atlas), 2 routed to agents outright.
7 struck items across both sources. 16 + 1 new = 17 live. ✓
**9 of the 15 board rows decided 2026-10-01; 5 fully open; 1 part-decided.**

---

## 7. Provenance

Written 2026-10-01 from the following exact sources, all read rather than recalled:

- `VulcanFlow_Technical_Design_Document_v2.2.md` at `vulcanflow/docs@main` — §0.0, §5.9, §9, §17, §21, §21.3, §24.1–24.4, §25, §26, §27, Appendix C. Content is TDD v2.3.
- `decisions/ADR-0001-retire-go-scaffold-vf-api.md`, `ADR-0002-rust-crate-set-and-phase0-pins.md`, `ADR-0003-rpc-scb-parser-and-billing-clients.md`, `ADR-0005` (delivery pipeline, §6 lane table) at `vulcanflow/docs@main`.
- `plans/phase1-work-breakdown.md` at `vulcanflow/docs` commit **`93c79f8`** — the head of closed PR #22, branch deleted. §2, §3, §3.2, §3.3, §4, §7.
- `decisions/ADR-0004-workspace-repository.md` and `ADR-0002` **with amendment A2** at `vulcanflow/docs` commit **`cee1edc`** — the head of closed PR #23, branch deleted. ADR-0002 §7's R1–R6 table is byte-identical to the `main` version (verified twice, independently).
- `vulcanflow/infra` PR #1 at `ed1b42c` — description and `README.md`, for what it does and does not install.
- `vulcanflow/platform` PR #1 at `66f4a91` — root `Cargo.toml` (workspace lint table, `k8s-openapi` pin and its comment) and `ci/forbid-unsafe.sh`.

The off-`main` documents in §0 are reachable without the deleted branches:
`git fetch origin 'refs/pull/*/head:refs/remotes/pr/*'` restores `93c79f8` (#22), `cee1edc` (#23)
and `5b3405c` (#20).

### Revision history

| Rev | Change |
|---|---|
| 1 | Register built. |
| 2 | Arithmetic corrections; Option B total; D1's urgency separated from D8's (CodeRabbit, lane 6). |
| 3 | **Atlas's review applied** — F1–F3 (§1.3 rebuilt: §25 vs ADR-derived sets separated, R1/R2's `admission/*` entries removed and R1 restated as "no test at all", R3's leak corrected to deterministic, `execution/gates-before-start` and `isolation/all-stores` added, `execution/scan-identity` surfaced as the only Phase 1 degradation) and P1–P6 (stale `VUL-*` ids dropped, 15/2 split, D7 named as the second Phase 1 exception, `infra#1` README quoted correctly, §4 scoped to §1.3, §27 item 5's scan-ID limb added). §1.1 narrowed to the two documents that actually assert Aether's existence, with §0.0's own verification caveat quoted against them. D15 narrowed to a ratification. D17 recited to ADR-0005 §6, with `unsafe` ownership described as relocated rather than removed. **Board answers of 2026-10-01 recorded** — §0.1, §1.5, §1.6, §2.1 and the per-row `DECIDED` entries. |
| 4 | **Two self-contradictions closed (CodeRabbit on `5168c5c`, lane 6).** The status row said "none before the Phase 2 gate," which the paragraph eight lines below it already contradicted on D7 — corrected to name D7's Phase-1-if-retained forcing point and D5/D15 as the earliest of the rest. And §1.3/§1.5 claimed "cross-tenant RLS isolation … at full strength on Testcontainers," which over-read §1.3's own R3 row: full strength belongs to the `SET LOCAL` leak tests and the admission-gate decision tests, while `isolation/background-queries` and `isolation/all-stores` stay weak at realistic PgBouncer concurrency. Counts and the set of open rows unchanged. |

Cost figures in §1.2 are derived from the footprint the design requires and are
order-of-magnitude only. **They are not quotes and no commitment should be made on them.** They are
retained after D1 because the Phase 2 gate will need them again.
