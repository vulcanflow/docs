# Decision records

Recorded engineering decisions for VulcanFlow. A decision that exists only in a comment
thread gets re-litigated, so anything that settles an open item lands here.

Each record states the question, the options considered, the choice, the reason, the TDD
§27 item it closes, and what would make the owner revisit it.

| ADR | Title | Status | Closes |
|---|---|---|---|
| [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) | Retire the Go scaffold in `vf-api` PR #1 | Accepted | §27 item 18 |
| [ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md) | Approved Rust crate set, toolchain pin, and secureCodeBox pin | Accepted — amended **A1** | §27 item 17 (and the engineering half of item 5 / §21.3) |
| [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) | Streaming RPC, secureCodeBox parser/hook language, and billing clients | Accepted — amended **A1** | §27 items 19, 20, 21 |
| [ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md) | The delivery pipeline and how its lanes are enforced | Accepted | No §27 item — supersedes the enforcement claim in the VUL-1 plan §4 |
| [ADR-0006](./ADR-0006-false-positive-equivalence-and-alias-semantics.md) | False-positive equivalence fields, alias semantics, revocation, and scanner-version invalidation | Accepted | §27 item 6 |

**ADR-0004 and ADR-0006 are not on `main` yet.** ADR-0004 is the Cargo workspace repository
layout — docs#23 was closed unmerged and it is re-filed as PR docs#27. ADR-0006 is the
false-positive equivalence fields and alias semantics, PR docs#26. Both numbers are reserved;
the gaps here are not missing records.

ADR-0005's operational companion is [`process/agent-workflow.md`](../process/agent-workflow.md),
which is what an agent reads mid-task. ADR-0007's is `CLAUDE.md` at the root of
`vulcanflow/platform`, which every session rooted at that checkout reads at startup.

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 13 | Engineering | DNS risk labels — external-resolution evidence, and the stronger evidence a confirmed dangling-resource finding needs. ADR-0006 §4.3 covers the DNS *record* observation class only; item 13 **blocks** any `dnsx/dangling-*` check class, because ADR-0006 §11 forbids a new check class creating or receiving a suppression until its equivalence row exists |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |
| — (not §27) | Needs an owner | **The product-facing severity model.** `findings-schema.json` at secureCodeBox v5.9.0 restricts `severity` to `INFORMATIONAL \| LOW \| MEDIUM \| HIGH`, so no conformant parser can emit `CRITICAL`, and the v5.9.0 nuclei parser collapses `CRITICAL → HIGH` before the artifact is written. §10.3's `KEV → EPSS → CVSS` enrichment is the mechanism; what a customer sees is undecided. ADR-0003 A1 §7.2, ADR-0006 §8.4.5 |

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
history in §10). ADR-0003 carries **A1** (two factual corrections from re-reading the v5.9.0
scanner tree — history in §7; the corrected paragraphs in §3.2 and §3.5 are annotated in place,
not rewritten). An amendment may add to a closed decision; reversing one needs a new ADR that
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
- **§10.2**'s `[PROPOSED]` false-positive matching paragraphs and its *"Engineering must define
  scanner-specific match fields with fixture examples before enabling persistence (§27)"* clause
  are discharged by ADR-0006. §26's "False positives" row is no longer a proposal. The §10.2
  sentences themselves remain accurate; only their status changes.
- **§24.1** lists the walking-skeleton pipeline as `subfinder → dnsx → httpx → nuclei`. That is
  unchanged, but it is worth stating next to it that secureCodeBox v5.9.0 ships **no** `dnsx` and
  **no** `httpx` scanner, so two custom `ScanType`/`ParseDefinition`/parser sets are Phase 1 work.
  §21.3 already anticipated the custom *images*; it is the parsers that were assumed to exist.
  See ADR-0003 A1 §7.1 and ADR-0006 §4.1.
- **No TDD section specifies where ingest-time severity comes from** — that is the actual gap, and
  this entry previously mis-stated it as *"§10.1/§16.3 treat scanner severity as usable as-is"*.
  Neither section says that: §10.1 is three sentences on asset-graph mapping and never mentions
  severity, and §16.3 lists *"CVE/CWE/CVSS/EPSS/KEV snapshots"* without a claim about scanner
  severity. The correction stands on its own facts: no secureCodeBox finding can be `CRITICAL` (the
  `findings-schema.json` enum has no such value) and the v5.9.0 nuclei parser collapses
  `CRITICAL → HIGH` before the artifact is written, so severity must be derived at ingest from
  §10.3's `KEV → EPSS → CVSS` enrichment. The claim this corrects is **ADR-0003 §3.5**'s remedy, not
  a TDD section. See ADR-0003 A1 §7.2 and ADR-0006 §8.4.5.

## Phase 1 risks that only a real cluster can settle

ADR-0002 §7 lists risks **R1–R6** with named owners, recorded per §24.4. None of them
justifies standing up a cluster now: the `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is a CEO decision tracked separately.
