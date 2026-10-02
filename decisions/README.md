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

**Closed 2026-10-02 — the other eleven repositories.** The ten `vf-*` service repositories and
`scanners` are private, and the Free plan refuses branch protection on a private repository. The
board **deferred** the GitHub Team upgrade, on the ground that all eleven are empty and so have
nothing to protect. The deferral is not left to memory: `ci/repo-protection-audit.sh` on
`platform` fails if any repository in the organisation holds code without protection, and a daily
Paperclip routine assigned to CEO runs it and puts a finding back on the decisions desk. §8.5 names
the routine and records that the audit tests three of §8.2's six settings, which is a defect in the
script and not a narrowing of §8.2.
Recorded in ADR-0005 **§8.5**, with a revisit trigger whose identifier is allocated at merge —
see that amendment's history row for why it is not guessed here. §8.5 also records what the
deferral does **not** buy, including the two things easiest to misread: the audit bounds how long
a gap goes *unnoticed*, not how long it goes *unfixed*, and the eleven private repositories have
no secret scanning or push protection either, which no part of this decision addresses.

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
