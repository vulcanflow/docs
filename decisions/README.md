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
| [ADR-0004](./ADR-0004-workspace-repository.md) | The Cargo workspace lives in `vulcanflow/platform` | Accepted | No §27 item — closes ADR-0001 appendix row #14, and dispositions all 15 repositories |
| [ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md) | The delivery pipeline and how its lanes are enforced | Accepted | No §27 item — supersedes the enforcement claim in the VUL-1 plan §4 |

ADR-0004 supersedes the root [`README.md`](../README.md) "Repository architecture (polyrepo)"
section, and **ADR-0004 §2.1 is the repository list of record** — the README table is fourteen
rows with no `platform` row. That table is pre-v2.3 and still names Go, chi and Huma. Rewriting
it was `docs#20`'s job and `docs#20` was **closed unmerged** (tag
`archive/pr-20-tdd-v2.3-rename`), so the rewrite has no owner today; it is listed under **Still
open** below. The superseded banner holds in the meantime.

ADR-0005's operational companion is [`process/agent-workflow.md`](../process/agent-workflow.md),
which is what an agent reads mid-task.

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| — (not §27) | Board | **The GitHub plan decision.** `vulcanflow` is on the free plan with private repositories, so branch protection and rulesets are unavailable and the lane gate cannot be made a *required* check. ADR-0005 §8 sets out the three options and recommends upgrading to GitHub Team. Until it is taken, `main` is directly writable and force-pushable on all four repositories |
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |
| — (not §27) | Atlas | **The Cargo workspace member set is on no record.** ADR-0004 §1.2 establishes that TDD §2.5.2 `[PROPOSED]` names **thirteen** crates, while the archived `platform#1` manifest declares **fourteen** — `vf-store` appears in no ADR on `main` (grep-confirmed) and ADR-0002 §3 is the third-party pin set, not the member list. The rebuild therefore has no approved crate list to build against. File as an ADR-0002 amendment alongside the A2 re-file |
| — (not §27) | Atlas | **ADR-0002 amendment A2 (§3.7) is not on `main`.** It was bundled into the closed PR docs#23 alongside ADR-0004 and did not come back with it; ADR-0004 §4.4 records why. A2 corrects six feature names in §3 that did not exist or meant the opposite of what §3 said, and withdraws two crates, so the Phase 0 pin set on `main` is currently wrong in six places. A2 must be re-filed — with every pin re-derived against crates.io, not copied from `archive/pr-23-adr-0004` — **before the `platform` workspace rebuild authors crates against §3** |
| — (not §27) | Unowned | **The root `README.md` rewrite has no owner.** Its "Repository architecture (polyrepo)" table names Go, chi, Huma and Connect-RPC, which ADR-0001 and ADR-0002 withdrew, and it has no `platform` row. `docs#20` carried the rewrite and was closed unmerged (`archive/pr-20-tdd-v2.3-rename`). ADR-0004 added a superseded banner so the table cannot be read as a live decision, but the banner is a holding action. Needs a work item |

The §2.5.2 crate table is superseded by ADR-0002 §3 and is no longer `[PROPOSED]`. §24.4's
gate on "approval of the Rust crate set used on the execution path" is cleared.

Read that sentence narrowly: it is about the **stack** table — which third-party crate and
framework each component uses — and ADR-0002 §3 is a third-party dependency pin set. It is
**not** about the workspace **member** list, which ADR-0002 does not contain and which is the
gap in the Still-open row above. "The crate set" means two different things in these documents,
and conflating them is what produced the withdrawn citation recorded in ADR-0004 §1.2.

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
