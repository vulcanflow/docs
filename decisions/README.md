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

**ADR-0004 is not on `main` yet.** It is PR docs#23, closed unmerged and pending T1. The number
is reserved for it; the gap here is not a missing record.

ADR-0005's operational companion is [`process/agent-workflow.md`](../process/agent-workflow.md),
which is what an agent reads mid-task.

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| — (not §27) | Board | **The GitHub plan decision.** `vulcanflow` is on the free plan with private repositories, so branch protection and rulesets are unavailable and the lane gate cannot be made a *required* check. ADR-0005 §8 sets out the three options and recommends upgrading to GitHub Team. Until it is taken, `main` is directly writable and force-pushable on all four repositories |
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |

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

## Phase 1 risks that only a real cluster can settle

ADR-0002 §7 lists risks **R1–R6** with named owners, recorded per §24.4. None of them
justifies standing up a cluster now: the `infra` repo's `factory/phase0-foundation` branch
stays unmerged, and lifting that hold is a CEO decision tracked separately.
