# Decision records

Recorded engineering decisions for VulcanFlow. A decision that exists only in a comment
thread gets re-litigated, so anything that settles an open item lands here.

Each record states the question, the options considered, the choice, the reason, the TDD
§27 item it closes, and what would make the owner revisit it.

| ADR | Title | Status | Closes |
|---|---|---|---|
| [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) | Retire the Go scaffold in `vf-api` PR #1 | Accepted | §27 item 18 |
| [ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md) | Approved Rust crate set, toolchain pin, and secureCodeBox pin | Accepted — amended **A1** | §27 item 17 (and the engineering half of item 5 / §21.3) |
| [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) | Streaming RPC, secureCodeBox parser/hook language, and billing clients | Accepted | §27 items 19, 20, 21 |
| [ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md) | The delivery pipeline and how its lanes are enforced | Accepted | No §27 item — supersedes the enforcement claim in the VUL-1 plan §4 |
| [ADR-0007](./ADR-0007-repo-level-agent-process-bootstrap.md) | The agent process bootstrap is checked into `vulcanflow/platform` | Accepted | No §27 item — process. Weighs against ADR-0004's definition of what `platform` holds |
| [ADR-0008](./ADR-0008-product-facing-severity-model.md) | The product-facing severity model — what a VulcanFlow report asserts | Accepted | No §27 item — the severity model was **open by omission**; see the §27 correction below |

**ADR-0004 and ADR-0006 are not on `main` yet.** ADR-0004 is the Cargo workspace repository
layout — docs#23 was closed unmerged and it is re-filed as PR docs#27. ADR-0006 is the
false-positive equivalence fields and alias semantics, PR docs#26. Both numbers are reserved;
the gaps here are not missing records. ADR-0008 is in the table above because this commit is
what puts it on `main`; it depends on facts recorded in ADR-0003 **A1 §7.2** and ADR-0006
**§8.4.5**, both of which are on docs#26 — the facts themselves were re-verified first-hand at
secureCodeBox `v5.9.0` for ADR-0008 §1.2 and do not wait on that merge.

ADR-0005's operational companion is [`process/agent-workflow.md`](../process/agent-workflow.md),
which is what an agent reads mid-task. ADR-0007's is `CLAUDE.md` at the root of
`vulcanflow/platform`, which every session rooted at that checkout reads at startup.

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |

**Closed 2026-10-02 — the product-facing severity model.** The board answered the VUL-139 card
on 2026-10-02T09:40:24Z — severity is **derived at ingest**, option 1 — and delegated the non-CVE
fallback and the parameters to CEO, who fixed them. Recorded as ADR-0008, which is the design of
record for the question. **This is a cross-branch deletion, named so it is not lost:** docs#26's
`Still open` table carries a *"Needs an owner — decided, record pending"* row for the severity
model, which does not exist on `main` and so cannot be deleted in ADR-0008's own diff. **Whichever
of docs#26 and ADR-0008 lands second deletes that row.** Item 13 is a separate row on docs#26 and
stays with its owner — ADR-0008 §8.4 names it as the first candidate for a `critical`-capable
first-party class and does not resolve it.

**Closed 2026-10-01 — the GitHub plan decision.** The board took option B of ADR-0005 §8: the
four active repositories (`docs`, `platform`, `infra`, `vf-api`) are **public**, which unlocks
branch protection on the free plan. `main` is protected on all four with `enforce_admins: true`,
`allow_force_pushes: false` and `allow_deletions: false`, and `platform`'s four lane-gate checks
are required. Recorded as ADR-0005 amendment 1; the disclosure cost is §8.3 and the route back to
private is R7.

The §2.5.2 crate table is superseded by ADR-0002 §3 and is no longer `[PROPOSED]`. §24.4's
gate on "approval of the Rust crate set used on the execution path" is cleared.

**Amendments.** An ADR here is amended in place with an entry in its own amendment-history
section, never silently edited. ADR-0002 carries **A1** (crypto, TLS and encoding pins — §3.6,
history in §10). An amendment may add to a closed decision; reversing one needs a new ADR that
supersedes it.

## Corrections to TDD v2.3 that fall out of these records

To be applied when the TDD is next revised; recorded here so they are not lost in the
meantime.

- **§2.5.2** justifies dropping Connect-RPC with "no first-party Connect-RPC Rust
  implementation is assumed". A first-party implementation now exists
  (`connectrpc/connect-rust`, v0.9.1, published by the ConnectRPC organisation). The
  decision not to adopt an RPC channel is unchanged; the stated reason is stale. See
  ADR-0003 §2.3.
- **§21.3** names secureCodeBox 5.8.0 as the published upstream release. v5.9.0 is now
  published and is the selected release. See ADR-0002 §6.
- **§10.3**'s `KEV → EPSS → CVSS` chain is **the authority for the asserted severity**, not only a
  configurable sort order. The section currently states the chain as "risk ordering" and exposes it
  as a sort; ADR-0008 §3 makes it the derivation. Both readings stand and they are different things
  (ADR-0008 §8.2). The same section's `[PROPOSED]` daily mirror of the KEV, EPSS and CVE/CVSS feeds
  into Postgres is **promoted from proposed to required for Phase 1** — ADR-0008 §7.3 is the
  consequence the board accepted when it took option 1.
- **§6.3 `findings`** gains five columns, one constraint, and a named payload: `scanner_severity`,
  `severity_source`, `severity_model_version`, `cvss_version`, `unenriched`; a `CHECK` on `severity`
  over the five product values `informational | low | medium | high | critical`, in the style §6.3
  already uses for `finding_states.state`; and the contents of `enrichment_snapshot` named rather
  than left open — KEV catalogue as-of date, EPSS model date and score, CVSS version/vector/score,
  which CVE drove the outcome, and the EPSS threshold in force at ingest. ADR-0008 §7.1 is the
  single list. **Note the section number:** the `findings` table is in **§6.3**, not §8; the VUL-139
  decision document cites §8 throughout and ADR-0008 §1.5 records the correction.
- **§16.2 / §16.3** — *"findings by severity"* is **five** buckets, not four. The §16.3 technical
  report lists the `severity_model_version`(s) present — plural, because §16.4 resolves a report to
  observations at a cutoff and observations ingested weeks apart can carry different versions — and
  flags `enrichment_stale` and `unenriched` findings. §16.4's `input-snapshot.json` already stores
  "enrichment and guidance versions"; the severity model version belongs in that set. ADR-0008 §6.2
  and §7.2.
- **§27 contains no item covering the severity model.** It was open by omission, which is a worse
  way for a question to be open than being listed, because nothing ever came up for review. Recorded
  here the way ADR-0005 and ADR-0007 both record having no §27 item, so the next revision closes the
  gap in the register rather than only in ADR-0008. §27 **item 13** is adjacent and unaffected: it
  stays with Engineering, and ADR-0008 §8.4 names it as the first candidate for a `critical`-capable
  first-party finding class without resolving it.
- **§25 contains no identifier for ingest-time severity derivation.** The matrix predates the
  question. ADR-0008 §7.4 names the five existing identifiers the record bears on
  (`findings/new-per-scan`, `findings/replayed-artifact`, `findings/fp-only-persistence`,
  `report/cross-consistency`, `report/matrix`) and deliberately mints no new one: a new identifier
  creates a new required test, and it belongs with the Phase 1 lane-1 spec for the enrichment path,
  one identifier to one test function.

## Phase 1 risks that only a real cluster can settle

ADR-0002 §7 lists risks **R1–R6** with named owners, recorded per §24.4. None of them
justifies standing up a cluster now: the `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is a CEO decision tracked separately.
