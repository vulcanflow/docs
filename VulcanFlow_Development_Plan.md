# VulcanFlow development plan

Status: proposed for owner review. This is a plan-only deliverable. Acceptance retains this plan; implementation, task creation, hiring, spending, and deployment require separate authorization.

## Goal and source baseline

Deliver the customer loop: register an authorized target → run a pipeline → inspect findings → follow guidance → verify by rescan → generate and schedule a report. Build a usable product with AI disabled.

Reviewed on 2026-10-05: both files in the live repository tree at commit `4f8dda2623b23388b1a7aeefc60cc2401baeeb6a`:

- [Technical design](https://github.com/vulcanflow/docs/blob/4f8dda2623b23388b1a7aeefc60cc2401baeeb6a/VulcanFlow_Technical_Design_Document_v2.2.md): its filename says v2.2, but its contents identify **v2.3**, updated 2026-09-30, superseding v2.2. Section references below refer to this file.
- [README](https://github.com/vulcanflow/docs/blob/4f8dda2623b23388b1a7aeefc60cc2401baeeb6a/README.md): repository map and prior setup backlog.

The live source was read directly from public GitHub. Paperclip reports its GitHub connection ready, but authenticated tools fail because the configured credential cannot be resolved; the managed CLI also reports an incomplete identity. This does not block this public documentation review. Repair and verify authenticated access before private-repository work.

No application repositories, deployed infrastructure, existing backlog status, or test results were audited. The referenced PRD and change summary are absent from this source tree and were not independently reviewed. The TDD is an engineering draft, not evidence that its examples are implemented.

## Decisions this plan preserves

These are source requirements from §§0–5, 10, 15, 17 and 24, not new proposals:

- Rust is the primary language for owned backend code, controllers, admission, completion hooks, and shared native/WASM graph validation. TypeScript remains the UI language; upstream scanners and approved services retain their languages.
- Aether hosts the service. Execution is IPv4-only. Customer infrastructure credentials, exploitation, and automatic configuration changes are outside scope.
- No execution without live authorization, correct scope, reserved allowance, and recorded evidence. An apex expands to discovered descendants only when discovery is selected; a configured subdomain is exact-host only.
- A target is one domain/subdomain. Each successful scan work unit consumes one unit; failed units consume zero; infrastructure retries do not double-charge. Verification uses the same accounting.
- Every scan creates fresh observations. Only explicit, narrowly matched false-positive decisions carry forward. Historical fixes cannot hide later findings.
- GA includes guidance, verification, both report audiences and groupings, scheduling, and Phases 0–4 core controls. AI is later work. Reports must not claim compliance certification.

## Resolve source conflicts first

1. **Language:** README lists Go, chi/Huma, controller-runtime and Connect-RPC. TDD §0.0 explicitly withdraws these owned-backend choices. Use Rust and the TDD's REST/SSE direction; review and pin proposed crates before use.
2. **Repository structure:** README records an owner-selected polyrepo. TDD §2.5 proposes one Cargo workspace and lockfile. Preserve the recorded polyrepo decision until the owner approves a change. Engineering must propose either versioned shared crates across repos with contract CI, or an explicitly approved consolidation. Do not silently convert to a monorepo.
3. **Component mapping:** reconcile the README with `vf-admission`, `vf-graph`, `vf-hook-notify`, `vf-core`, and `vf-db`; keep dispatcher a module of the API. Do not create repositories until the mapping is agreed.
4. **Existing work:** reconcile README-linked GitHub issues #3–#14 and #16 and inspect any existing Go implementation before estimating migration or creating duplicate tasks. Their current completion is unverified.

## Delivery sequence and acceptance gates

Durations below are **planning allowances**, not source estimates or calendar commitments. They assume an available Aether environment, experienced Rust engineers, prompt decisions, and no major inherited-code rewrite. Re-estimate after M0. Security and recovery work starts in M0/M1 and continues throughout.

| Milestone | Deliverables and dependencies | Exit evidence | Proposed accountable role | Indicative duration |
|---|---|---|---|---|
| M0 — Baseline and foundation, TDD Phase 0 | Inventory code/platform/backlog; settle repo/workspace mapping; approve crate set and Rust review ownership; pin toolchain, SCB, charts, images and templates; validate arm64, S3 client, PgBouncer/sqlx, kube-rs admission, OpenAPI; establish GitOps, secrets, data services and CI. | Version/compatibility matrix and architecture decisions; signed arm64 image through dev/staging; S3 and DB conformance; no assumed SCB schema fields. | Technical lead + platform engineer | 2–4 weeks |
| M1 — Safe execution core, Phase 1 | After M0: OIDC/roles, tenant provisioning and isolation, Track A, canonical scope, durable outbox, graph translation, per-work reservations, admission/start barriers, ingest, observation identity, false-positive matching, SSE, cancellation. Use a controlled fixture fleet. | Unapproved and unreserved work cannot start through Scan/Job/Pod paths. Concurrent runs cannot overspend. Revocation stops pending/active work. Duplicate events cannot create duplicate observations or consumption. | Rust control-plane lead + security/execution engineer | 4–6 weeks |
| M2 — Walking skeleton and scanner conformance, Phases 1–2 plus thin 3a/3b slices | After M1: fixed subfinder → dnsx → httpx → nuclei pipeline; Rust adapters/hook; minimal findings view, small reviewed guidance set, verification action and one technical PDF. Pull these thin UI/report slices forward to test the full product loop. Expand tlsx/nmap and controlled masscan only after conformance. | One verified fixture target completes the full loop. Each actual SCB identifier maps to its logical work unit. Blocked/skipped checks yield inconclusive verification. Dynamic fan-out closes before final reports. Retry, malformed-output and zero-finding cases pass. Pool launch/kill paths pass before pool enablement. | Execution engineer + full-stack engineer + QA | 3–5 weeks |
| M3 — Findings and verification product, Phase 3a | After skeleton/data contracts: template activation, builder, native/WASM parity, topology, filters, projects, triage, guidance versions and fallbacks, narrow false-positive persistence, verification outcomes, usage visibility. | Later positive scan creates a new observation after a historical fix. Guidance works with AI off. Core flows pass accessibility and UI performance gates. User can see coverage and allowance skips. | Frontend/full-stack lead + Rust lead + content reviewer | 3–5 weeks |
| M4 — Reports and automation, Phase 3b | After stable observation/verification contracts: all four audience/grouping combinations, HTML/PDF, immutable snapshots, cutoff/watermark consistency, scheduler, durable delivery, notifications and preferences. May overlap M3 after shared contracts freeze. | Identical scope/filter/cutoff gives equal counts across variants. Malicious text/logos remain inert; renderer has no network. Restart/queue loss recovers without duplicate delivery. DST, overlaps and missed windows are explicit. Approved disclaimer appears in both formats. | Reporting/backend engineer + full-stack lead + QA | 3–5 weeks |
| M5 — Paid launch and release readiness, Phase 4 | After M3/M4: approved packages and target-slot policy; Lago/Stripe clients, signatures and reconciliation; operational abuse/KYC boundaries, egress reputation, deletion/retention, recovery drills, load evidence and support runbooks. | Accounting reconciles across retries, period boundaries and downgrades. Isolation, kill, restore, deletion replay and release analysis pass. Owner authorizes the final GA cut and release. | Technical lead + platform/QA + product/operations | 3–5 weeks |
| M6 — Later capabilities, Phase 5 | After GA and separate scope decisions: Track B at scale, optional sharing, branding polish, AI assistance, later SDKs and marketplace. | Each feature has an approved boundary, entitlement and acceptance contract. AI cannot dispatch or silently export tenant data. | Product + relevant engineering lead | Estimate separately |

M2 is a controlled pilot, not GA. The sequential M0–M5 allowances total roughly 18–30 weeks before contingency; overlap may shorten that, while Rust training, platform gaps, content review or migration may extend it. Do not publish a launch date from this range. M0 must produce a capacity-based forecast and revised scope.

Critical dependency chain: baseline/SCB compatibility → authorization and reservation/start barriers → trustworthy observations and outcomes → verification and report snapshots → commercial/recovery evidence → release. Frontend prototypes, reviewed guidance, report layouts and legal copy can advance alongside backend work after implementation is separately authorized.

## Verification and release contract

Use TDD §§23–25 as the acceptance index. Each enabled requirement must link to actual CI/staging evidence; a named test alone is insufficient.

- **Execution and isolation:** `authz/start-barrier-all-paths`, `execution/gates-before-start`, `execution/late-cascade-barrier`, `isolation/all-stores`, `isolation/background-queries`, and `abuse/egress-stop`. Include tenant/pool paths, stale JWTs, failed controllers, redirects, private destinations, pooled connections and actual traffic cessation. Proposed stop budget: under 60 seconds.
- **Accounting and observations:** `usage/unit-definition`, `usage/concurrent-reservations`, `usage/failure-retry-settlement`, `usage/verification-normal`, `findings/new-per-scan`, `findings/replayed-artifact`, `findings/fp-only-persistence`, and `verify/applicable-check-evidence`. Port discovery plus three successful service checks must count four; one failed service check reduces it to three.
- **Reports and schedules:** `report/matrix`, `report/cross-consistency`, `report/xss-network-corpus`, `report/disclaimer-framing`, `report/worker-and-queue-loss`, and `schedule/occurrence-identity`. Test snapshot reuse, stale rollups, recipient consent and deduplicated delivery.
- **Performance:** TDD targets include findings filter/sort p75 <100 ms at 10k observations, SSE p75 <1 second, builder 60 fps at 200 nodes, initial LCP <2.5 seconds, and core-flow/HTML-report WCAG 2.2 AA. Proposed report budgets are p95 <60 seconds for 1,000-observation technical PDFs and <30 seconds for 12-month management reports. Establish a defined test environment and dataset before enforcing budgets.
- **Resilience and release:** 100 concurrent pipelines, 50 report jobs and a 24-hour soak; restore and deletion replay; signed images, pinned templates, Rust supply-chain checks, and no isolation-failure override. Proposed DR goals: Postgres RPO ≤5 minutes/RTO ≤1 hour. TDD staging analysis uses platform errors <1%; its proposed 30-minute/50-scan minimum must hold on insufficient evidence. Production retains manual sign-off.

## Decisions and risk owners

| Decision or risk | Owner | Due / effect |
|---|---|---|
| Rust capability, crate approval, existing Go migration and polyrepo/workspace conflict | Technical lead; owner for repository change | M0 exit; highest near-term schedule uncertainty (§27: 16a, 17, 18) |
| SCB selected release, identifiers, hook/parser contracts, candidate reservation and real start barrier | Execution + platform lead | Before automatic execution/cascades; fail closed until proven (§27: 5, 20) |
| GA cut, budget, team capacity and launch date | Organization owner/product | Forecast after M0; no calendar commitment now (§27: 1) |
| Target-slot lifecycle, package limits, period changes, mixed outcomes and corrections | Product + metering lead | Before production accounting/paid use; fixture values are not prices (§27: 2–4, 21) |
| Scanner-specific false-positive equivalence and DNS-risk evidence | Security/execution lead | Before persistence/risk labels; prevent unrelated suppression or takeover claims (§27: 6, 13) |
| Report selection/count semantics, disclaimer and permission wording | Product/data + legal reviewer | Before customer reports or Track B (§27: 9, 12) |
| Retention, object versions/backups and restored-data erasure | Product/data + platform | Before production data/deletion promises (§27: 10) |
| External mail provider and recipient consent; optional public links | Product/data | Before external delivery; sharing stays off until decided (§27: 11) |
| AAAA observations; direct IP/CIDR targets | Product | Omit unless explicitly approved; active execution remains IPv4 (§27: 7, 8) |
| Egress abuse ownership, rate limits and depeering response | Operations + product | Before external pilot; prove minimum kill controls in M1 (§27: 14) |
| AI data consent/runtime/entitlements, optional Amass/branding, later RPC | Product + engineering | Later feature gates; do not block core plan (§27: 15, 16, 19) |

## Proposed team and governance

MasterChief owns this plan, decision tracking, and progress reporting. Proposed implementation capacity: one Rust technical lead, one execution/security engineer, one platform engineer, one frontend/full-stack engineer, and one QA/reliability engineer, with named product, legal and remediation-content reviewers. These are responsibilities and staffing assumptions, not hires or assignments. Add a reporting/backend engineer or extend the forecast if the lead cannot cover M4. Start guidance review early; the TDD's top-200 curated-class target is a proposal whose effort requires a content inventory.

At each milestone, present a short demo, evidence links, open risks, and revised forecast. At M0, reconcile existing work before proposing an assigned task graph. Preserve separate approval for implementation and hiring.

## Done means and next step

For this task: a source-linked plan is saved with dependencies, milestone deliverables, acceptance evidence, proposed roles, estimates labelled as assumptions, and unresolved owner decisions. The user reviews this revision and either accepts it as the planning deliverable or requests changes. Plan acceptance alone does not start implementation.
