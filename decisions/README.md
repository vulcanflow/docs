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
| [ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md) | The delivery pipeline and how its lanes are enforced | Accepted — amended **1**, **2**, **3**, **4**, **5** | No §27 item — supersedes the enforcement claim in the VUL-1 plan §4 |
| [ADR-0007](./ADR-0007-repo-level-agent-process-bootstrap.md) | The agent process bootstrap is checked into `vulcanflow/platform` | Accepted | No §27 item — process. Weighs against ADR-0004's definition of what `platform` holds |

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
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |

**Closed 2026-10-01 — the GitHub plan decision.** The board took option B of ADR-0005 §8: the
four active repositories (`docs`, `platform`, `infra`, `vf-api`) are **public**, which unlocks
branch protection on the free plan. `main` is protected on all four with `enforce_admins: true`,
`allow_force_pushes: false` and `allow_deletions: false`, and `platform`'s four lane-gate checks
are required. Recorded as ADR-0005 amendment 1; the disclosure cost is §8.3 and the route back to
private is R7.

The §2.5.2 crate table is superseded by ADR-0002 §3 and is no longer `[PROPOSED]`. §24.4's
gate on "approval of the Rust crate set used on the execution path" is cleared.

**Amendments.** An ADR here is amended in place with an entry in its own amendment-history
section, never silently edited. An amendment may add to a closed decision; reversing one needs a
new ADR that supersedes it. The status column above carries the marker, so a stale row is itself a
defect — this one was stale for ADR-0005 amendment 1 until 2026-10-01. **The marker is whatever
label the record itself uses**, quoted rather than normalised: `A1` in ADR-0002's own amendment
history, bare numbers in ADR-0005's. Two conventions in one column is accurate rather than untidy
— the index exists to be checked against each record, and renumbering here would break exactly
that check.

| ADR | Amendment | What it changed |
|---|---|---|
| ADR-0002 | **A1** | Crypto, TLS and encoding pins — §3.6, history in §10 |
| ADR-0005 | **1** | The §8 GitHub plan question decided: option B, repositories public, protection live — §3.2, §7 items 8–9, §8, R2, R7 |
| ADR-0005 | **2** | Reviewer #2's route is the CodeRabbit **App**, triaged by Warren; bot commentary is not a verdict; lane 5.5 named — §2, §6.1, §6.2, R6, plus `process/agent-workflow.md` and `plans/open-decisions.md` D17 |
| ADR-0005 | **3** | Lane 7's merge condition has one statement, the `docs#24` merge is recorded under R5b, and every merge now owes an attestation — §2, §2.3, §6.1–§6.4, §7 item 10, §9, R5a/R5b, plus `process/agent-workflow.md` and this index |
| ADR-0005 | **4** | NEUTRAL `ci/**` is lane-3 work with four agents excluded by name, and the attestation detector's floor is four named commits — §2, §4.4, §6.1, §6.5, §7 items 10–11, R5b, R6, R8, plus `process/agent-workflow.md` |
| ADR-0005 | **5** | The gate's own weakening split into three limbs: deleting a check was already closed by branch protection, the classifier-with-its-own-harness diff is closed by monotonicity plus an assertion floor, and the workflow file's job **bodies** are left to review — unmechanised under `on: pull_request`, and **not** unmechanisable, which is the correction the CEO's R9 answer forced before merge: `on: pull_request_target` runs `main`'s workflow and `main`'s checkout, and is rejected on three reasons (it still lets the hollowing diff merge; it puts `gate-self-test` in violation of GitHub's never-execute rule; and GitHub blocks the trigger by default on public repositories from **2026-11-02**). **R9 is answered, negative** — no ruleset `workflows` rule on this plan — and new **R11** carries the Actions event policy with its expiry date — §4 and its check table, §4.1, §4.2, §4.4, §4.5, §4.6, §7 items 4 and 12, R9, R10, R11, plus `process/agent-workflow.md` |

Each row is a **pointer into that record's own amendment history**, which is the authority on what
an amendment changed. It is deliberately not a summary: a summary here would be a second statement
of the record, which is the defect ADR-0005 §6.3 exists to end.

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

## Phase 1 risks that only a real cluster can settle

ADR-0002 §7 lists risks **R1–R6** with named owners, recorded per §24.4. None of them
justifies standing up a cluster now: the `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is a CEO decision tracked separately.
