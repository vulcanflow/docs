# Open-decision register

| | |
|---|---|
| **Status** | Awaiting board decision as one batch |
| **Date** | 2026-10-01 |
| **Owner** | CEO. Reviewed by Atlas. |
| **Sources** | TDD §27 (all 22 rows), `plans/phase1-work-breakdown.md` §7 (all 7 rows), ADR-0001 through ADR-0004 |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` — filename says v2.2, content is **TDD v2.3** |
| **Issue** | VUL-8 |

**Nothing in this register blocks Phase 0 or Phase 1**, with one exception that is called out
as such: D16 is needed inside Phase 1 and is Atlas's to answer, not the board's.

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

So the board is being asked for **15 decisions**, and seven of the fifteen can be closed by
accepting a recommended default in this document without further research.

Two of the fifteen are urgent, for different reasons, and they should not be confused.
**D1** is the only one that changes what the next two phases can *prove* — it is the only
decision with a current verification impact, and it is why D1 has its own section. **D8**
has no verification impact at all and is needed last, but it is the only item on the
register whose lead time we do not control, because it depends on outside counsel. D1
decides what Phase 1 is worth; D8 is the one to start today. The remaining thirteen can
wait for a batch answer without costing anything.

Breakdown §7 contributed exactly one new decision. That is worth stating plainly: the seven
"more" items were almost entirely the same product questions §27 already asked, restated in
Phase 1 terms. Reading them as a separate queue would have doubled the apparent decision
load.

### A material finding, outside this register's scope but blocking its citations

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
versions, so D1 below is unaffected.

---

## 1. D1 — The `infra` cluster go/no-go

This is the only item on the register that is costing anything today, and the only one whose
answer changes what the next two phases can prove. It gets its own section.

| Field | |
|---|---|
| **Item** | Do we stand up a Kubernetes environment for Phase 1, and if so which one? |
| **Source** | Breakdown §7.7 (the only genuinely new item in that section). Risk register in ADR-0002 §7 (R1–R6). The §27 item 5 residue that ADR-0002 and ADR-0003 explicitly did **not** close. |
| **Blocks** | Nothing in Phase 0 or Phase 1 *delivery*. It blocks **verification**: six named risks and twelve §25 identifiers (table in §1.3) cannot be closed at full strength without it, and `infra#1` stays unmerged. |
| **Needed by** | Phase 2 forces a partial answer (secureCodeBox operator must run somewhere). Phase 4 forces a complete one (`recovery/restore`, `deletion/all-stores-restore`, `release/analysis`, `abuse/egress-stop` have no non-cluster form at all). |
| **Owner** | Board. |

### 1.1 The question is not the one the brief assumed

The brief frames this as "the cost is recurring," which assumes we are buying a cluster. Every
document in the project says otherwise. **Aether** is described throughout as an *existing,
already-approved, self-operated Kubernetes platform*:

- TDD §9 header: *"Platform: Aether (self-operated Kubernetes platform — not AWS)."*
- TDD §24.2, Phase 0 deliverables: *"**Existing** approved Aether stack."*
- TDD §0.0 technology row: Aether, Kubernetes, Harbor, Argo CD and Kargo are **approved**, not proposed.
- `infra#1`'s own README: GitOps *"against a **preinstalled** Argo CD/Kargo control plane"*; *"This repo does not install Argo CD, Kargo, or Harbor"*; the registry is *"Aether-provided container registry (external; this repo does not deploy it)."*

So `infra#1` is not a cluster build. It is 811 lines of Kustomize overlays, operator CRs and
Helm *placeholders* that assume a control plane already exists and that someone will fill in
the destinations. Its chart versions are marked `UNCONFIRMED` and its ApplicationSet is
deliberately kept out of the apply paths *"until destination inputs are confirmed."*

**Which means the board's first answer is a fact, not a judgement, and it costs nothing to
give:** does an Aether environment exist today, and do we have a namespace and credentials on
it? No agent has ever confirmed this. Every statement above is a document asserting it. The
honest position is that the project has been designed against a platform nobody has verified
is reachable.

That single fact selects between three very differently-priced options.

### 1.2 Options and cost

Figures are **order-of-magnitude, derived from the footprint the design requires, and need a
real quote before anyone commits to them.** The arithmetic is shown so it can be corrected
rather than argued about.

**Option A — Aether exists and has capacity. Confirm destinations and merge `infra#1`.**

| Line | |
|---|---|
| Recurring | **~$0 incremental.** Capacity already paid for; this is a namespace and quota allocation. |
| One-time | ~2–4 agent-days to confirm Kubernetes minor, registry hostname, GitOps destinations, S3 endpoint and secret-store wiring, then unmark the `UNCONFIRMED` pins. |
| Settles | R1–R6 and all twelve identifiers in §1.3, at full strength. |
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
| Cost | The twelve identifiers in §1.3 close at reduced strength or not at all, and the §1.3 list is what we would be shipping without. |

### 1.3 What stays unverifiable without a cluster

ADR-0002 §7's six risks, mapped to the §25 identifiers each one leaves weak. The middle
column is the part that matters: Testcontainers does not fail to cover these by oversight, it
covers them *partially*, which is the dangerous shape — a passing test that proves less than
its name claims.

| Risk | What Testcontainers proves, and what it cannot | §25 identifiers left weak | Risk owner |
|---|---|---|---|
| **R1** Admission webhook TLS: cert issuance, rotation, `caBundle` injection | Proves the handler's decision logic. Cannot produce a real API server calling us over TLS with a chain it accepted. | `admission/cascade-gate-delete-oldobject`, `admission/workload-gate-dryrun` | Kiln |
| **R2** Admission ordering and failure policy under load — `failurePolicy`, timeouts, mutating reinvocation | Proves one webhook in isolation. Cannot reproduce the API server's admission chain, which is where the behaviour actually lives. | `admission/workload-gate-dryrun`, `authz/start-barrier-all-paths` (its Kubernetes paths) | Kiln |
| **R3** RLS through PgBouncer at realistic concurrency, incl. `SET LOCAL` under connection churn | Proves the mechanism — and this is the single highest-value test in the set: a bare `SET` instead of `SET LOCAL` leaks tenant context to the next borrower of that connection, which is a cross-tenant RLS bypass. Cannot reproduce production churn, which is the condition under which it leaks. | `db/tenanttx-set-local-isolation`, `db/pgbouncer-transaction-pooling-prepared`, `isolation/background-queries` | Forge + Ledger |
| **R4** Ceph RGW conformance against the **real Aether** RGW — its version, tuning, bucket policy | Proves conformance against a containerised RGW, which is a different deployment. | `storage/s3-compat-conformance` | Forge |
| **R5** arm64 build reproducibility end to end on Aether nodes | Proves reproducibility on the CI runner. Cannot prove it on the real build and runtime platform. | `build/rust-supply-chain`, `perf/service-baseline` | Crucible |
| **R6** secureCodeBox v5.9.0 operator against the cluster's actual Kubernetes minor, `garage.enabled: false`, external object store; Harbor mirroring of pinned digests | Proves the hook contract shape. Cannot prove operator / API-server interaction or digest mirroring. **This is the unclosed half of §27 item 5.** | `scb/hook-invocation-contract`, `execution/scan-identity`, `supply-chain/check-catalog` | Kiln |

**Twelve distinct identifiers — thirteen references, because `admission/workload-gate-dryrun`
is weak under both R1 and R2 — and four more with no non-cluster form at all, sixteen in total** — `deletion/all-stores-restore`
(needs ClickHouse, object storage *and a real backup restore*), `recovery/restore`,
`release/analysis`, `abuse/egress-stop`. Those four are Phase 4 and are not an argument for
deciding now.

### 1.4 Recommendation

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

---

## 2. Live board decisions

Seven of these carry a recommended default the board can accept as-is. The rest need a real
answer, and the **Needed by** column is what determines when.

### D2 — Launch date and final GA feature cut

| Field | |
|---|---|
| **Source** | TDD §27 item 1 |
| **Blocks** | No §25 identifier. It blocks the §24.3 GA cut line, and therefore sequencing decisions in Phases 3b and 4. |
| **Needed by** | Phase 3b planning. Not before. |
| **Options** | (a) Fix a date now and cut scope to fit. (b) Fix the scope at §24.3's stated cut and let the date follow. (c) Defer both to the Phase 3a gate. |
| **Recommendation** | **(b).** §24.3 already defines a defensible cut line and names its fast-follows. Adopting it as the GA scope converts D2 from two unknowns into one, and the date becomes an output of the program plan (T2) rather than an input to it. |
| **Owner** | Board |

### D3 — Package names, limits, entitlements and ceilings

| Field | |
|---|---|
| **Source** | TDD §27 item 2 |
| **Blocks** | Phase 4 (`usage/lago-stripe-replay`) and enabling paid use. **Not** the Phase 1 allowance tests — `usage/unit-definition`, `usage/remaining-work-notification` and `usage/concurrent-reservations` are parameterised over a configured limit and turn green without knowing the commercial numbers. |
| **Needed by** | Phase 4. |
| **Options** | (a) Decide the full commercial matrix now. (b) Decide only the *shape* now — that limits exist, are per-billing-period, and are enforced at reservation time — and set the numbers at the Phase 4 gate. (c) Defer entirely. |
| **Recommendation** | **(b).** The shape is already implied by §17 and by the Phase 1 tests being written against it. The numbers are a pricing exercise with no engineering dependency, and deciding them now means deciding them with no usage data. |
| **Owner** | Board |

### D4 — Target accounting: registered slots vs distinct scanned targets

| Field | |
|---|---|
| **Source** | TDD §27 item 3 **and** breakdown §7.1 — the same decision, stated twice |
| **Blocks** | Paid use, Phase 4. `usage/unit-definition`, `usage/remaining-work-notification`, `usage/lago-stripe-replay`. |
| **Needed by** | Phase 4. The breakdown already contains the mitigation: VUL-28 keeps slot-accounting policy behind one named function, so this decision changes one place. |
| **Options** | (a) **Active registered targets** — a slot is consumed by registration and freed by removal. Predictable for the customer, invites slot-churn gaming. (b) **Distinct scanned hostnames per period** — tracks real cost, but the customer cannot see their bill coming. (c) Hybrid: registered slots cap concurrency, scanned hostnames meter usage. |
| **Recommendation** | **(a), with removal not freeing a slot until the next billing period.** It is the option a customer can predict, and the churn hole closes with one rule. (b)'s unpredictability is the complaint that generates support load. Automatic discovered-target enrollment should be **opt-in per target**, which §5.3's apex-discovery opt-in already establishes as the pattern. |
| **Owner** | Board |

### D5 — Billing edges: period attribution, mixed per-unit outcomes, post-settlement corrections

| Field | |
|---|---|
| **Source** | TDD §27 item 4 **and** breakdown §7.2 — the same decision, stated twice |
| **Blocks** | Breakdown §7.2 is precise about the real consequence: accepting scanner batches that cannot report success at the configured unit granularity — **Phase 2**. Then `usage/failure-retry-settlement` and `usage/lago-stripe-replay` in Phase 4. |
| **Needed by** | **Phase 2** — the earliest forcing point on this register after D1. |
| **Options** | (a) A work unit settles all-or-nothing; a batch with partial scanner success consumes nothing. (b) Per-target settlement within a batch; partial success consumes partially. (c) Settle optimistically and correct after the fact. |
| **Recommendation** | **(a).** §17.3 already guarantees failed units consume zero, and (a) is the only option consistent with that without new machinery. (c) requires audited post-settlement corrections, which is the hardest thing in item 4 and the one most likely to produce a customer-visible billing error. If a scanner cannot report at unit granularity, that is a reason to reject the batch, not to invent a settlement rule for it. Period attribution: **attribute to the period the work *started* in**, so a scan spanning midnight on the period boundary has one unambiguous home. |
| **Owner** | Board, with Atlas on the granularity question |

### D6 — AAAA observation: adopt explicitly, or state an IPv4-only limitation

| Field | |
|---|---|
| **Source** | TDD §27 item 7 **and** breakdown §7.5 — the same decision, stated twice. §5.9 carries it as `[PROPOSED — owner review]`; §24.4 requires it be adopted explicitly or omitted with an honest coverage statement. |
| **Blocks** | Report coverage claims. `execution/ipv4-only` (its AAAA cases) and `report/matrix` coverage wording. The breakdown notes `execution/ipv4-only` encodes the current behaviour **either way**, so this does not block writing the test. |
| **Needed by** | Phase 3b (report coverage). Cheaper to decide now because it is one sentence. |
| **Options** | (a) **Observe AAAA, scan IPv4 only, disclose untested IPv6** — §5.9's proposal. (b) Do not observe AAAA at all; state a flat IPv4-only limitation. |
| **Recommendation** | **(a).** Aether is IPv4-only, so neither option adds scanning capability — the only difference is whether we can *tell the customer about the gap*. Under (b), a customer's IPv6-only host is silently never scanned and the report cannot say so. Under (a), the report states "resolved AAAA, not scanned," which is the honest artefact and costs one extra DNS record type and one report sentence. Honesty about coverage is the whole argument for naming this in §24.4. |
| **Owner** | Board |

### D7 — Standalone IP/CIDR registration

| Field | |
|---|---|
| **Source** | TDD §27 item 8 |
| **Blocks** | Standalone IP/CIDR features. `authz/configured-scope`, `usage/unit-definition`. Domain-derived IPv4 scanning is already approved and unaffected. |
| **Needed by** | Not on the GA path if dropped. Phase 1 if retained, because it changes what `authz/configured-scope` must enforce. |
| **Options** | (a) Retain direct IP/CIDR registration, and define target accounting and an authorization-attestation path for it first. (b) Drop it for GA; domain-derived IPv4 only. |
| **Recommendation** | **(b), and decide it now rather than later.** This is the one deferrable item where deferring is actively expensive: §27 item 8 says *"if yes, explicitly define target accounting/scope **before** enabling"* — so a late yes reopens D4 and `authz/configured-scope` after both are closed. The substantive problem is that ownership of a CIDR cannot be proven by the DNS-based challenge in §5.2; it needs a different attestation path, which is real Phase-4 work. Drop it, and let customer demand reopen it post-GA. |
| **Owner** | Board |

### D8 — Legal: disclaimer, ToS, permission-attestation wording; attestation expiry; manual-review service level

| Field | |
|---|---|
| **Source** | TDD §27 item 9, and the disclaimer half of breakdown §7.4 |
| **Blocks** | `report/disclaimer-framing` (Phase 3b — the breakdown names it as blocked on this decision specifically). The KYC half of `authz/approval-tracks`, i.e. Track B. §16.7's versioned disclaimer in the base template. |
| **Needed by** | Phase 3b for the disclaimer. Track B at GA per §24.3's fast-follow list. |
| **Options** | (a) Commission outside counsel now. (b) Ship a conservative internally-drafted placeholder and get it reviewed before GA. (c) Defer to Phase 3b. |
| **Recommendation** | **(a) — and this is the item to start today even though it is needed last.** It is the only decision on the register with a dependency outside the company and therefore the only one with a lead time we do not control. Everything else here is a judgement the board can make in an afternoon. A security scanner's permission-attestation wording is also the document that decides whether an unauthorised scan is our liability or the customer's, which is not a thing to draft internally and hope. §16.7 already puts the disclaimer under a version, so counsel's text drops into a slot that exists. |
| **Owner** | Board + outside counsel |

### D9 — Retention and deletion policy

| Field | |
|---|---|
| **Source** | TDD §27 item 10, and the retention half of breakdown §7.4 |
| **Blocks** | `deletion/all-stores-restore` and `recovery/restore` (both Phase 4). §3.6 defers the policy here explicitly; §24.2 puts full isolation and recovery evidence in Phase 4. |
| **Needed by** | Phase 4. |
| **Options** | (a) Accept §26's stated defaults — 30-day recovery window; raw output 30d, exports 7d, reports for account life. (b) Set shorter retention to reduce the breach surface. (c) Longer, for customer historical comparison. |
| **Recommendation** | **(a).** §26's defaults are already written down, are internally consistent with §3.6's recovery window, and the Phase 1 mechanism (`TenantTx`, RLS) is policy-agnostic — changing the numbers later changes configuration, not code. The two genuinely open sub-questions are **locked evidence vs immediate erasure** (recommend: erasure wins, with a documented exception only where a legal hold exists) and **restored-data re-erasure** (recommend: a restore must replay the deletion log before the data is reachable — which is precisely what `deletion/all-stores-restore` tests, and the test is already written to that shape). |
| **Owner** | Board |

### D10 — Public report sharing and external delivery

| Field | |
|---|---|
| **Source** | TDD §27 item 11, and the sharing half of breakdown §7.4 |
| **Blocks** | `report/branding-sharing` (Phase 3b/4 — the breakdown names it as blocked on this decision specifically). §16.9's share records. §21's transactional-email provider boundary. |
| **Needed by** | Phase 3b/4. §24.3 already lists public report sharing as a post-GA fast-follow. |
| **Options** | (a) Ship public sharing and an external email provider at GA. (b) **Authenticated download only at GA**; no external transactional-email provider; revisit post-GA. (c) Sharing on, email off. |
| **Recommendation** | **(b).** §26 already has public sharing *off by default* and §24.3 already lists it post-GA, so this mostly ratifies the design. The decisive point is §0.0 rule 8 and §18.1: *no tenant findings data leaves the Aether boundary without explicit per-tenant opt-in.* An external transactional-email provider is exactly that boundary crossing, and adopting one requires a vendor data-processing review we have not scoped. Authenticated download needs none. Oversized reports to non-account recipients then stops being a question. |
| **Owner** | Board |

### D11 — Report selection, count semantics, account-wide scope, and KPI targets

| Field | |
|---|---|
| **Source** | TDD §27 item 12 |
| **Blocks** | `report/matrix`, `report/cross-consistency` (both Phase 3b). |
| **Needed by** | Phase 3b. |
| **Options** | (a) Accept §26's defaults — 5–10 top risks, 5,000-observation render cap, explicit coverage statement, the §16.10 budgets (1,000-observation technical PDF p95 <60s; 12-month management report p95 <30s). (b) Set different counts and caps. (c) Defer. |
| **Recommendation** | **(a).** The numbers are illustrative in §26 but they are self-consistent and the budgets in §16.10 are already written as testable thresholds. The one part genuinely needing a board answer is **comparable coverage** — what we claim a report covers. Recommend tying it to D6: coverage is "resolved and scanned," with resolved-but-unscanned (IPv6, blocked, skipped) stated separately rather than folded in. That is the same honesty rule as D6 and `verify/applicable-check-evidence` already enforces it on the verification side. |
| **Owner** | Board |

### D12 — Egress service: depeering contingency, abuse ownership, rate limits, KYC operations

| Field | |
|---|---|
| **Source** | TDD §27 item 14 |
| **Blocks** | `abuse/egress-stop` (Phase 4). The KYC half of Track B. §20.3–20.5's controlled pool and the §21 <60s kill target. |
| **Needed by** | Phase 4. |
| **Options** | (a) Decide the whole operational package now. (b) **Name an abuse owner now; defer rate limits, depeering contingency and KYC operations to Phase 4.** (c) Defer entirely. |
| **Recommendation** | **(b).** Three of the four parts are Phase 4 operational tuning with no Phase 1 or 2 dependency. **Abuse ownership is different and should be settled now, because it is free and because it is the one that fails at the worst moment.** A scanning platform will receive an abuse complaint, and the cost of not having a named recipient is measured in hours of upstream-provider patience. Name a person, not a role, with a mailbox that is monitored. |
| **Owner** | Board |

### D13 — Phase 5 AI: external-data consent, commercial entitlements, serving and GPU configuration

| Field | |
|---|---|
| **Source** | TDD §27 item 15 |
| **Blocks** | Phase 5 only. No §25 identifier — Phase 5 sits beyond the §25 matrix, and `remediation/version-tier-no-ai` exists to guarantee guidance works with AI disabled. |
| **Needed by** | Phase 5. §24.3 puts GA at "usable operation with AI disabled." |
| **Options** | (a) Decide now. (b) Defer entirely to a Phase 5 gate. |
| **Recommendation** | **(b), with no further discussion.** This is the clearest defer on the register and the structural reason is already enforced in code: §18's **crate-level ban on an AI→dispatcher dependency** means Phase 5 cannot leak backwards into the execution path, and `build/rust-supply-chain` plus the `ci/crate-boundaries.sh` gate are the detectors. Deciding GPU configuration four phases early would be guessing at hardware for a workload we have not specified. |
| **Owner** | Board, at the Phase 5 gate |

### D14 — Optional features: Amass tier, bounty-feed licensing, branding rollout, design tokens

| Field | |
|---|---|
| **Source** | TDD §27 item 16 |
| **Blocks** | Optional features only. `report/branding-sharing`, and `supply-chain/check-catalog` if Amass joins the pinned scanner set. §27 notes technology adoption is already approved — these are packaging and operational choices. |
| **Needed by** | Phase 2 if Amass is in the Phase 2 scanner images; otherwise post-GA. |
| **Options** | (a) Decide the package now. (b) Defer, with one carve-out: confirm before Phase 2 whether Amass is in the pinned scanner set, because that changes `supply-chain/check-catalog`'s pinned-image list. |
| **Recommendation** | **(b).** The §24.1 walking skeleton fixes the pipeline at `subfinder → dnsx → httpx → nuclei`, which does not include Amass — so the default is "not in Phase 2" and the carve-out resolves itself unless the board says otherwise. Bounty-feed licensing, branding rollout and design tokens are all post-GA by §24.3. |
| **Owner** | Board |

### D15 — Confirmed-dangling-resource evidence

| Field | |
|---|---|
| **Source** | TDD §27 item 13 (owner: Engineering) **and** breakdown §7.6 (owner: CEO + Atlas) — the sources disagree on owner; the breakdown is later and more specific, so joint |
| **Blocks** | Finding classification, and therefore report content. No dedicated §25 identifier — it changes what `report/matrix` and the §15.2 guidance assert, not whether a test passes. |
| **Needed by** | Phase 2, when scanners first produce findings that would carry the label. |
| **Options** | (a) Define the stronger evidence required for a confirmed takeover finding now. (b) **Adopt the conservative label as the standing default and let Atlas file the stronger-evidence definition as an ADR.** |
| **Recommendation** | **(b).** The breakdown already states the safe interim behaviour — *label observed external resolution without asserting ownership or takeover* — and it should simply become the default rather than an interim. §27 item 13's own warning is the reason: *avoid treating all third-party hosting as takeover.* A false takeover claim is the most damaging single wrong output this product can produce, because the customer acts on it. The board's decision here is one sentence: adopt the conservative default. The evidence definition that would justify upgrading a label is a technical judgement, and it belongs in an ADR with Atlas's name on it. |
| **Owner** | Board for the default; Atlas for the ADR |

---

## 3. Not board decisions

Two §27 rows are addressed to the board by the TDD but are not actually the board's to answer.
They are listed so the §27 ledger stays complete, and routed.

### D16 — False-positive persistence: equivalence fields and alias semantics

| Field | |
|---|---|
| **Source** | TDD §27 item 6 (owner: Engineering). **Not** in breakdown §7 — which is correct, and informative. |
| **Blocks** | `findings/fp-only-persistence` — a **Phase 1** identifier, Ledger's, in VUL-22. |
| **Needed by** | **Phase 1.** This is the only row on the entire register needed that early, and the only reason it is not a blocker is that it is answerable by engineering without a board decision. |
| **Why it is not a board question** | It asks for *"concrete scanner-specific equivalence fields, alias semantics, and examples proving an old decision cannot suppress unrelated findings."* That is a specification written by reading scanner output formats, which is what ADR-0002 §6 and ADR-0003 §3 already did for the secureCodeBox contract. There is no product or commercial trade-off in it. §26 already narrows the shape: a *"narrow versioned target/check/location matcher with explicit decision revocation."* |
| **Owner** | **Atlas.** Needs an ADR before Ledger can write `findings/fp-only-persistence`. Flagged to the board only because a Phase 1 test depends on it and nobody has assigned it. |

### D17 — Team Rust capability and the schedule impact of the language change

| Field | |
|---|---|
| **Source** | TDD §27 item 16a (owner: Engineering leadership). Listed in §24.2 as a Phase 0 deliverable. |
| **Blocks** | The Phase 0 schedule. No §25 identifier. |
| **Needed by** | Phase 0 — nominally now. |
| **Why it is largely already answered** | Item 16a asks four things, and three are settled by this company's structure rather than by a decision: **current experience and hiring/training plan** — the team is the nine agents in the accepted plan §5, so this is not a hiring question; **code-review ownership for async and `unsafe`-adjacent code** — plan §4 names Assay and Warren as the two required reviewers, and the question is partly moot because `ci/forbid-unsafe.sh` makes `unsafe` a build failure workspace-wide, so there is no `unsafe` to own; **async review** sits with Assay by the same table. |
| **Residue** | The **schedule impact** of the language change, which is a program-plan output. |
| **Owner** | **CEO, via T2** (`docs/plans/program-plan.md`). Removed from the board's queue; no board input needed. |

---

## 4. The one real dependency between rows

D1 is not just the largest item, it gates the honesty of five others. D9, D10 and D11 all turn
on what we can truthfully claim about coverage and deletion, and three of the identifiers that
would substantiate those claims — `deletion/all-stores-restore`, `recovery/restore`,
`storage/s3-compat-conformance` — are in §1.3's list of what a cluster-less project cannot
verify. The board can adopt the recommended defaults for D9–D11 today; it cannot *evidence*
them until D1 resolves. Both facts should be on the record at the same time.

Everything else on this register is independent and can be answered in any order.

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
| **5** | ~~First automatic cascade: SCB release/digests, node compatibility, scan-ID field, hook/CRD inputs, candidate reservation and start barrier~~ | **ADR-0002** §6 (release, digests, arm64, CRD type generation — `Also settles` the engineering half) + **ADR-0003** §3.3 (hook contract) | Struck **except its cluster half** — node Kubernetes-minor compatibility and Harbor digest mirroring, which is risk **R6** and folded into **D1**. ADR-0003 §5 says so explicitly: *"it needs a cluster, and that hold is CEO's to lift."* Candidate reservation and the start barrier are Phase 1 tests (`authz/start-barrier-all-paths`, `execution/late-cascade-barrier`), not open decisions. |

**ADR-0004** closed no §27 item — it is a repository-topology decision and says so in its own
header. It is cited here only because it is one of the four documents currently off `main` (§0).

Breakdown **§7.3** — *"Lago/Stripe client decision (§27 item 21, ADR-0003)"* — is likewise
**struck**: ADR-0003 §4 decided the client shape. What remains of that row is commercial
policy, which is D3 and D5. The row was stale when written.

---

## 6. Coverage ledger

Every source row, accounted for exactly once. This table is the acceptance criterion.

### TDD §27 — 22 rows

| §27 | Disposition |
|---|---|
| 1 | D2 |
| 2 | D3 |
| 3 | D4 (merged with breakdown §7.1) |
| 4 | D5 (merged with breakdown §7.2) |
| 5 | **Struck** — ADR-0002 §6 + ADR-0003 §3.3; cluster residue (R6) → D1 |
| 6 | D16 — Atlas, not board |
| 7 | D6 (merged with breakdown §7.5) |
| 8 | D7 |
| 9 | D8 (merged with breakdown §7.4, disclaimer half) |
| 10 | D9 (merged with breakdown §7.4, retention half) |
| 11 | D10 (merged with breakdown §7.4, sharing half) |
| 12 | D11 |
| 13 | D15 (merged with breakdown §7.6) |
| 14 | D12 |
| 15 | D13 |
| 16 | D14 |
| 16a | D17 — CEO via T2, not board |
| 17 | **Struck** — ADR-0002 |
| 18 | **Struck** — ADR-0001 |
| 19 | **Struck** — ADR-0003 §2 |
| 20 | **Struck** — ADR-0003 §3 |
| 21 | **Struck** — ADR-0003 §4 |

16 live, 6 struck, 22 total. ✓

### Breakdown §7 — 7 rows

| §7 row | Disposition |
|---|---|
| Target-slot semantics (§17.2) | **Duplicate** of §27 item 3 → D4 |
| Post-settlement corrections, mixed per-unit outcomes (§17.3, §8.2) | **Duplicate** of §27 item 4 → D5 |
| Lago/Stripe client decision | **Struck** — ADR-0003 §4 |
| Disclaimer wording, public sharing, retention (§24.4, §3.6) | **Duplicate** of §27 items 9, 10, 11 → D8, D9, D10 |
| AAAA handling (§5.9, §24.4) | **Duplicate** of §27 item 7 → D6 |
| Confirmed-dangling-resource evidence (§5.3) | **Duplicate** of §27 item 13 → D15 |
| Lifting the `infra` `factory/phase0-foundation` hold | **New** → **D1** |

1 new, 1 struck, 5 duplicates, 7 total. ✓

### Register total

17 live rows (D1–D17): 15 board-owned (D5 and D15 jointly with Atlas), 2 routed to agents outright. 7 struck items across both
sources. 16 + 1 new = 17 live. ✓

---

## 7. Provenance

Written 2026-10-01 from the following exact sources, all read rather than recalled:

- `VulcanFlow_Technical_Design_Document_v2.2.md` at `vulcanflow/docs@main` — §0.0, §5.9, §17, §21, §24.1–24.4, §25, §26, §27. Content is TDD v2.3.
- `decisions/ADR-0001-retire-go-scaffold-vf-api.md`, `ADR-0002-rust-crate-set-and-phase0-pins.md`, `ADR-0003-rpc-scb-parser-and-billing-clients.md` at `vulcanflow/docs@main`.
- `plans/phase1-work-breakdown.md` at `vulcanflow/docs` commit **`93c79f8`** — the head of closed PR #22, branch deleted. §3 and §7.
- `decisions/ADR-0004-workspace-repository.md` and `ADR-0002` **with amendment A2** at `vulcanflow/docs` commit **`cee1edc`** — the head of closed PR #23, branch deleted. ADR-0002 §7's R1–R6 table is byte-identical to the `main` version.
- `vulcanflow/infra` PR #1 at `ed1b42c` — description and `README.md`, for what it does and does not install.

Cost figures in §1.2 are derived from the footprint the design requires and are
order-of-magnitude only. **They are not quotes and no commitment should be made on them.**
