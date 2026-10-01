# Decision records

Recorded engineering decisions for VulcanFlow. A decision that exists only in a comment
thread gets re-litigated, so anything that settles an open item lands here.

Each record states the question, the options considered, the choice, the reason, the TDD
§27 item it closes, and what would make the owner revisit it.

| ADR | Title | Status | Closes |
|---|---|---|---|
| [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) | Retire the Go scaffold in `vf-api` PR #1 | Accepted | §27 item 18 |

## Still open

| §27 item | Owner | Blocks |
|---|---|---|
| 16a | Engineering leadership | Phase 0 schedule — team Rust capability and schedule impact |
| 17 | Engineering | Phase 0 — approve and pin the §2.5.2 crate set |
| 19 | Product / engineering | Post-GA SDKs — typed streaming RPC beyond REST + SSE? |
| 20 | Engineering | Phase 2 — secureCodeBox parser and hook language |
| 21 | Engineering | Phase 4 — Lago and Stripe client approach in Rust |

The §2.5.2 crate table stays `[PROPOSED]` until item 17 closes. Per §24.4, no crate on the
execution path is settled before then.
