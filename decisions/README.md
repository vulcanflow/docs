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

**ADR-0004 is not on `main` yet.** It is PR docs#23, closed unmerged and pending T1. The number
is reserved for it; the gap here is not a missing record.

ADR-0005's operational companion is [`process/agent-workflow.md`](../process/agent-workflow.md),
which is what an agent reads mid-task.

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| — (not §27) | Board | **The GitHub plan decision.** `vulcanflow` is on the free plan with private repositories, so branch protection and rulesets are unavailable and the lane gate cannot be made a *required* check. ADR-0005 §8 sets out the three options and recommends upgrading to GitHub Team. Until it is taken, `main` is directly writable and force-pushable on all four repositories |
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 13 | Engineering | DNS risk labels — external-resolution evidence, and the stronger evidence a confirmed dangling-resource finding needs. ADR-0006 §4.3 covers the DNS *record* observation class only; item 13 **blocks** any `dnsx/dangling-*` check class, because ADR-0006 §11 forbids a new check class creating or receiving a suppression until its equivalence row exists |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |
| — (not §27) | Needs an owner | **The product-facing severity model.** `findings-schema.json` at secureCodeBox v5.9.0 restricts `severity` to `INFORMATIONAL \| LOW \| MEDIUM \| HIGH`, so no conformant parser can emit `CRITICAL`, and the v5.9.0 nuclei parser collapses `CRITICAL → HIGH` before the artifact is written. §10.3's `KEV → EPSS → CVSS` enrichment is the mechanism; what a customer sees is undecided. ADR-0003 A1 §7.2, ADR-0006 §8.4.5 |

The §2.5.2 crate table is superseded by ADR-0002 §3 and is no longer `[PROPOSED]`. §24.4's
gate on "approval of the Rust crate set used on the execution path" is cleared.

**Amendments.** An ADR here is amended in place with an entry in its own amendment-history
section, never silently edited. ADR-0002 carries **A1** (crypto, TLS and encoding pins — §3.6,
history in §10). ADR-0003 carries **A1** — one amendment with four entries, history in §7: two
factual corrections from re-reading the v5.9.0 scanner tree (§3.2 and §3.5, annotated in place,
not rewritten); one presentational correction to §3.3, which mixed slice-relative and absolute
`argv` index bases in adjacent rows of the very table that exists to prevent an off-by-one; and
one citation correction to §3.4, which sourced the scan-fingerprint field to a reading ADR-0002
does not contain — the field choice (`metadata.uid`) is now stated in ADR-0003's own voice and
closes §27 item 5's "exact scan-ID field" limb.
**§3.3 is the one place the annotate-in-place rule is deliberately not followed**, because there
the presentation *is* the defect: the before-state is quoted verbatim in §7.3 instead. An
amendment may add to a closed decision; reversing one needs a new ADR that supersedes it.

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
- **§10.1/§16.3** treat scanner severity as usable as-is. No secureCodeBox finding can be
  `CRITICAL` (schema enum), and the nuclei parser collapses `CRITICAL → HIGH`. Severity must be
  derived at ingest from §10.3 enrichment. See ADR-0003 A1 §7.2.

## Phase 1 risks that only a real cluster can settle

ADR-0002 §7 lists risks **R1–R6** with named owners, recorded per §24.4. None of them
justifies standing up a cluster now: the `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is a CEO decision tracked separately.
