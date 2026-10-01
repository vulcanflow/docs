# Decision records

Recorded engineering decisions for VulcanFlow. A decision that exists only in a comment
thread gets re-litigated, so anything that settles an open item lands here.

Each record states the question, the options considered, the choice, the reason, the TDD
§27 item it closes, and what would make the owner revisit it.

| ADR | Title | Status | Closes |
|---|---|---|---|
| [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) | Retire the Go scaffold in `vf-api` PR #1 | Accepted | §27 item 18 |
| [ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md) | Approved Rust crate set, toolchain pin, and secureCodeBox pin | Accepted — amended **A1**, **A2** | §27 item 17 (and the engineering half of item 5 / §21.3) |
| [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) | Streaming RPC, secureCodeBox parser/hook language, and billing clients | Accepted | §27 items 19, 20, 21 |
| [ADR-0004](./ADR-0004-workspace-repository.md) | The Cargo workspace lives in `vulcanflow/platform` | Accepted | the repository question left open by ADR-0001's appendix |

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| 5 (cluster half) | Engineering | First automatic cascade — Harbor artifacts and node Kubernetes compatibility for the pinned secureCodeBox v5.9.0. Needs a cluster; ADR-0002 §7 risk R6 |
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |

One more open question that is not a §27 item but gates the same code:

| Question | Owner | Blocks |
|---|---|---|
| Aether's Kubernetes control-plane minor | CEO | ADR-0002 §3.7.4. `k8s-openapi 0.28.0` offers `v1_32`–`v1_36` and is pinned at `v1_32`. If Aether is **below** 1.32, `kube 4.2.0` / `k8s-openapi =0.28.0` is the wrong pair. Does not block §24.1 |

The §2.5.2 crate table is superseded by ADR-0002 §3 and is no longer `[PROPOSED]`. §24.4's
gate on "approval of the Rust crate set used on the execution path" is cleared.

**Amendments.** An ADR here is amended in place with an entry in its own amendment-history
section, never silently edited. ADR-0002 carries **A1** (crypto, TLS and encoding pins — §3.6)
and **A2** (pin and feature corrections found by building the workspace, the OIDC pins, and
A1 assertion 2 restated — §3.7); history for both is in its §10. An amendment may add to a
closed decision; reversing one needs a new ADR that supersedes it.

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
