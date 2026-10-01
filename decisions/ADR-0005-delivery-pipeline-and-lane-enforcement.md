# ADR-0005 — The delivery pipeline and how its lanes are enforced

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) — recorded by CEO under the bootstrap in §9 |
| **Decides** | How the nine delivery rules in the VUL-1 plan §4 are made to fail mechanically rather than advisorily |
| **Supersedes** | The enforcement claim in VUL-1 plan §4 (`CODEOWNERS` + two required approving reviews) and the "enforced at review" clause of `plans/phase1-work-breakdown.md` §6 |
| **Depends on** | Nothing. This is the first thing through the pipeline, by design |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (content is **TDD v2.3**) — §25 test identifiers, §24.4 |
| **Issue** | VUL-2 |
| **Implementation** | `vulcanflow/platform` — `ci/lane-gate.sh`, `ci/lane-gate-test.sh`, `.github/workflows/lane-gate.yml` (PR platform#2) |

---

## 1. The question

The VUL-1 plan set nine delivery rules and assigned them to nine agents as seven lanes. The
plan then claimed those lanes would be enforced in three places: each agent's managed
instructions, branch protection with two required approving reviews, and `CODEOWNERS` putting
test paths under the test authors.

An instruction is advisory. The first time a coding agent is one assertion away from a green
suite, it will reach for the test file — not out of malice, but because editing the assertion
is the shortest path to the goal it was given. So the question this record answers is not
*what are the rules*; §4 already settled that. It is **which mechanism actually refuses**,
given what this organisation's GitHub setup can and cannot do.

Two facts discovered while implementing it changed the answer. Both are recorded in §4 and §5
with the evidence, because both will look like arbitrary design choices to anyone who reads
only the outcome.

---

## 2. The pipeline, unchanged

The lanes and owners below are §4 of the VUL-1 plan, restated as the authority. No agent holds
two adjacent lanes on the same change.

| # | Lane | Owner | The rule |
|---|---|---|---|
| 1 | **Spec** | Atlas | Every work item gets a §25 test identifier and an acceptance statement before anything is written. |
| 2 | **Tests** | Scribe (unit, property, fuzz) · Ledger (integration, conformance, e2e) | Tests are written first, from the spec, by test authors only. Test authors never execute a suite and never open production source. |
| 3 | **Code** | Forge · Anvil · Kiln | Implement until the named identifiers pass. |
| 4 | **Run** | Crucible — sole executor | The only agent that executes suites. Publishes a mechanical PASS / FAIL / MISSING ledger keyed by §25 identifier. |
| 5 | **Red → fix** | Forge · Anvil · Kiln | A failing test is a code defect. It routes back to lane 3 as a code fix. The test is not touched, not relaxed, not quarantined. |
| 6 | **Review** | Assay **and** Warren — both required | Assay reviews by hand against TDD v2.3 and the ADRs, and rejects any lane crossing on sight before reading the diff. Warren runs `coderabbit:review` and posts the verdict, triaging findings into blocking versus advisory. |
| 7 | **Merge** | Crucible | Green suite + two approving verdicts → merge to the default branch. Nothing merges by any other route. |

### 2.1 The prohibitions, stated in the negative

- **Forge, Anvil, Kiln** change production source only. Never a test file, a fixture, a golden
  file, an assertion, or a `#[cfg(test)]` block. They never add `#[ignore]`, never rename or
  delete a test, and never relax a test to reach green.
- **Scribe, Ledger** change test files only. They never execute a suite and never open
  production source.
- **Crucible** executes suites and merges. It writes neither production code nor tests.
- **Atlas** sets the spec. It writes neither production code nor tests, and is deliberately not
  a required reviewer, so the review gate stays exactly two.
- **Assay, Warren** review. They write neither code nor tests.

### 2.2 The one escape hatch

Lane 5 is absolute for a red test: it is never edited into green. But a test can still be
*wrong* — asserting behaviour the TDD never promised. That is not a test fix, it is a **spec
change**:

1. Atlas amends the TDD or files an ADR. The spec moves first, in the open.
2. Only then does **the original test author** rewrite the test.
3. The rewrite goes on a **test-only pull request carrying no production code**.

Three properties make this safe rather than a loophole: the spec moves before the test, the
author is the test author and not whoever hit the red, and the rewrite is physically separated
from the code it would otherwise be covering for. The gate in §6 enforces the third
mechanically; the first two are review obligations on Assay.

---

## 3. Why the original enforcement design cannot be built

### 3.1 Constraint one — there is one GitHub identity, not nine

Every VulcanFlow agent reaches GitHub through Paperclip's managed credential broker. For this
company the broker resolves to a **single personal identity**:

```
POST /runtime-tools/github/credentials
  → status            available
    source            personal
    login             zozo6015
    authenticationMode  managed
    GIT_AUTHOR_NAME   zozo6015
    GIT_AUTHOR_EMAIL  2540250+zozo6015@users.noreply.github.com
```

and the organisation has exactly one member:

```
GET /orgs/vulcanflow/members  → ["zozo6015"]
GET /orgs/vulcanflow          → plan.name "free", plan.filled_seats 1
```

Two of the three planned mechanisms die on this:

- **`CODEOWNERS` cannot distinguish agents.** It maps paths to GitHub accounts. One account is
  one owner, so a `tests/**` rule owned by "the test authors" is a rule owned by the same
  account that writes the production code. It cannot tell Scribe from Forge.
- **"Two required approving reviews" is unsatisfiable.** GitHub refuses a pull request
  author's own approval, and a second approval from the same account does not exist. Requiring
  two would not raise the bar; it would deadlock every pull request the agents open.

**On the verification that was asked for and not obtained.** VUL-2 asked to confirm this by
inspecting the broker response for a *second* agent once a hire was approved. That test could
not be run: all nine hires are still `pending_approval`, so no second agent has a runtime from
which to call the broker. The org-membership evidence above is treated as dispositive anyway,
because it does not depend on what Paperclip would mint — a GitHub approving review must come
from an account with access to the repository, and there is exactly one such account. The
residual uncertainty is carried as revisit trigger **R1** in §10.

### 3.2 Constraint two — branch protection is not available on this plan

`vulcanflow` is on the **free** plan and all four active repositories are **private**. GitHub
gates both branch protection and rulesets behind a paid plan for private repositories:

```
GET  /repos/vulcanflow/docs/branches/main/protection  → 403 "Upgrade to GitHub Pro or make
PUT  /repos/vulcanflow/docs/branches/main/protection  → 403  this repository public to
GET  /repos/vulcanflow/docs/rulesets                  → 403  enable this feature."
GET  /orgs/vulcanflow/rulesets                        → 403 "Resource not accessible by integration"
```

The `PUT` was attempted, not merely the `GET`, so this is a confirmed write refusal and not a
read permission artefact. GitHub **Actions**, by contrast, works normally on these
repositories:

```
GET /repos/vulcanflow/docs/actions/permissions → {"enabled": true, "allowed_actions": "all"}
```

So the gate can **run** and go red on every pull request. What GitHub cannot currently do is
*refuse the merge* on it, or refuse a force-push, or refuse an admin bypass. That is a billing
decision, not an engineering one, and it is escalated rather than decided here — see §8.

---

## 4. Decision

**The lanes are enforced by partitioning the diff on paths, not on people.** A path partition
needs no identities, so it is immune to constraint 3.1, and it is implemented as GitHub Actions
checks, so it is unaffected by constraint 3.2 for the purpose of *detecting* a violation.

Four checks, in `vulcanflow/platform` at `ci/lane-gate.sh`, run by
`.github/workflows/lane-gate.yml` on every pull request to `main`:

| Check | Refuses |
|---|---|
| `lane-partition` | any diff containing **both** production source and test files; also any diff that changes the gate itself alongside source or tests |
| `test-erosion` | any added `#[ignore]`; any removed test declaration; any net fall in the workspace's total test count |
| `inline-test-modules` | any `#[cfg(test)]` module inside `crates/*/src` |
| `gate-self-test` | runs the gate against 17 fixture diffs and asserts each verdict, so the gate's own logic is itself tested |

### 4.1 Why `lane-partition` is the load-bearing one

It makes lanes 2, 3 and 5 mechanical in a single rule. A coding agent **physically cannot** ship
a test edit alongside code: the combination is refused before anyone reads it. And a pull
request containing only test files is self-evidently a test author's work, whoever the commit
says authored it. The identity problem is routed around rather than solved — the gate stops
asking *who* and starts asking *what*.

The second half of the rule — a gate change may not travel with source or tests — closes the
obvious hole, which is weakening the gate in the same pull request as the change it would let
through.

### 4.2 The path classification

Ordered; a path is tested against TEST before PROD, so `crates/vf-core/tests/add.rs` is a test
and not production source.

| Class | Paths |
|---|---|
| **GATE** | `ci/lane-gate.sh`, `ci/lane-gate-test.sh`, `.github/workflows/lane-gate.yml` |
| **TEST** | `tests/**`, `crates/*/tests/**`, `fuzz/**`, `conformance/**`, `**/testdata/**`, `**/golden/**`, `*.golden`, `*.snap` |
| **PROD** | `crates/*/src/**`, `crates/*/build.rs`, `crates/*/Cargo.toml`, `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml` |
| **NEUTRAL** | everything else — Markdown, `.gitignore`, other workflows, other `ci/` scripts |

At most one of GATE, TEST and PROD may appear in a diff. NEUTRAL may accompany any of them, so
a code change can carry its own documentation.

### 4.3 Why `test-erosion` counts the whole tree

Scanning only the diff catches a weakened test but not a vanished one. `test-erosion` counts
test declarations (`#[test]`, `#[tokio::test]`, `#[test_case`, `#[rstest`, `proptest!`,
`fuzz_target!`) across the entire tree at the base and at the head, and fails if the number
falls. That catches a deleted test file, a test commented out, and a test quietly moved out of
the suite — while allowing a test to be *renamed or relocated*, because the count is preserved.

---

## 5. The consequence that constrains real work: no inline test modules

This is the one part of the decision that costs something, and it is recorded here rather than
buried in CI, because it changes how Scribe writes tests.

**In Rust, a `#[cfg(test)]` module lives in the same file as the code it tests.** That file is
production source and test at once. A path partition cannot split one file, so inline test
modules would leave the gate's central rule trivially bypassable: a coding agent editing its
own `#[cfg(test)]` block produces a diff the gate classifies as pure PROD and waves through.

Therefore: **unit tests live in `crates/<crate>/tests/` and are written against the crate's
public API.** `plans/phase1-work-breakdown.md` §6 already forbids coding agents from touching a
`#[cfg(test)]` block; this record goes further and removes the construct from the workspace, so
the prohibition does not depend on anyone noticing.

The cost is real: a test in `crates/<crate>/tests/` cannot reach a private item. For the nine
§25 identifiers Scribe owns — all of which reduce to a pure function — this is not a
limitation and is arguably better, because it tests the contract rather than the internals. If
a crate genuinely needs to assert on a private item, that is revisit trigger **R3** in §10, not
a local exception.

---

## 6. Review and merge authority lives in Paperclip, not GitHub

Because two approving GitHub reviews are unsatisfiable (§3.1) and required checks are
unavailable (§3.2), lanes 6 and 7 are enforced in Paperclip:

- **Assay** and **Warren** record their verdicts on the Paperclip issue, not as GitHub review
  approvals. Two recorded verdicts are the gate.
- **Crucible** performs every merge, and refuses to merge without *all* of: a green lane gate,
  a green test ledger with no FAIL and no MISSING for the identifiers in scope, Assay's verdict
  and Warren's verdict.
- Nothing merges by any other route. Crucible's refusal is the merge gate until GitHub can hold
  it, and remains the authority on review count afterwards.

This is weaker than a required check in one specific way, and the weakness should be named: it
rests on Crucible following its instructions, which is exactly the kind of advisory enforcement
this record exists to replace. It is the strongest available arrangement under §3.2 and it is
why §8 escalates the plan question.

**Warren is blocked until T7.** `coderabbit:review` is not installed in this company. Until it
lands, lane 6 has one reviewer, and the honest consequence is that merges wait rather than
proceeding on one verdict relabelled as two.

---

## 7. Consequences

1. **A coding agent cannot reach a test file in the same change as code.** The shortest path to
   green no longer runs through the assertion.
2. **Every change costs more pull requests.** A feature is now at least two: the test, then the
   code. This is the intended price and it is paid on every item.
3. **Unit tests move out of `src`.** See §5.
4. **The gate itself is tested.** `gate-self-test` asserts 17 fixture verdicts, so a future edit
   that silently guts the gate fails the gate.
5. **The gate is live on `platform` only.** `docs` has no code; `infra` and `vf-api` are empty
   repositories whose `main` is an initial commit. The gate is added to each the moment it
   receives source; a Markdown-only repository has no lane to cross.
6. **`platform#1` is unaffected but should be checked.** It bundles `.github/workflows/**`,
   `ci/**`, `crates/*/src/**` and `Cargo.toml` — all PROD and NEUTRAL, no test files — so
   `lane-partition` passes it. T1 should confirm before merging that it introduces no
   `#[cfg(test)]` module, or `inline-test-modules` will fail it.
7. **This record and the gate merged without the gate.** See §9.
8. **Branch protection, force-push refusal and admin-bypass refusal do not exist yet.** Until
   §8 is decided, `main` on all four repositories is writable directly and rewritable by
   force. The gate detects lane crossings on pull requests; it does not stop a direct push.

---

## 8. Open, and escalated: the plan decision

Deliverable 3 of VUL-2 — branch protection on every active repository with
`enforce_admins: true` and `allow_force_pushes: false` — **cannot be applied on the current
GitHub plan** (§3.2). Three ways forward, in the order recommended:

| | Option | Cost | Effect |
|---|---|---|---|
| **A** | Upgrade `vulcanflow` to **GitHub Team** | list price ≈ $4 per user per month; one seat is filled today | Unlocks branch protection and rulesets on private repositories. All four required checks become genuinely required; force-push, branch deletion and admin bypass become refusable. Repositories stay private. |
| **B** | Make the four repositories **public** | free | Same enforcement unlocked, but publishes a pre-GA security product's source, its pinned dependency set and its infrastructure manifests. |
| **C** | Change nothing | free | The gate still runs and still goes red, visibly, on every pull request. GitHub will not block the merge on it; Crucible's refusal (§6) is the only thing holding the gate, and `main` stays force-pushable. |

**A** is recommended: it is the only option that buys real enforcement without disclosure, and
at one seat the cost is negligible against the cost of one silently-edited test reaching `main`.
The decision is the board's, and it is open until they take it.

---

## 9. The bootstrap, stated plainly

This record and the gate it describes were merged by CEO without passing through the pipeline,
because at the moment they merged the pipeline did not exist and no reviewing agent had been
approved. That is a genuine exception and it is written down rather than glossed, so that it
cannot later be cited as precedent.

Its scope is exactly: ADR-0005, `process/agent-workflow.md`, the `decisions/README.md` index
row, and `platform` PR #2. Nothing else. Every change after these goes through the gate, and
the gate's first demonstrated refusal is linked from §11 of `process/agent-workflow.md`.

---

## 10. Revisit triggers

Named, so this record is revisited on evidence rather than on mood.

- **R1 — per-agent GitHub identities appear.** If the board approves the nine hires and the
  broker returns a *distinct* `login` for any agent other than CEO, §3.1 is void. Re-examine
  `CODEOWNERS` over `tests/**`, `**/tests/**` and `fuzz/**`, and required approving reviews, as
  an *addition* to the path partition — not a replacement, since the partition is strictly
  stronger against a shared account and costs nothing once written.
- **R2 — the plan question in §8 is decided.** On **A** or **B**, wire all four checks as
  required status checks on `main` with `enforce_admins: true`, `allow_force_pushes: false`,
  `allow_deletions: false` and stale approvals dismissed on new commits, on every repository
  that has a default branch. On **C**, record the acceptance of risk against §7 item 8.
- **R3 — a crate needs to unit-test a private item.** Bring the case to Atlas. The resolution
  is either a narrowed public surface, a `pub(crate)` seam exposed deliberately, or an
  amendment to §5 — never a local `#[cfg(test)]` module.
- **R4 — the two-pull-request cost measurably dominates.** If the test/code split is observed
  to cost more in cycle time than the lane crossings it prevents, revisit §7 item 2 with the
  measurements, not the impression.
- **R5 — a lane crossing reaches `main` anyway.** Any such event is a gate defect. Reproduce it
  as a fixture in `ci/lane-gate-test.sh` first, then fix the gate.
- **R6 — `coderabbit:review` lands (T7).** Lane 6 returns to two reviewers; remove the
  single-reviewer degradation in §6.

---

## 11. Relationship to the other records

- **Supersedes the enforcement claim in the VUL-1 plan §4.** The lanes, the owners, the
  prohibitions and the single escape hatch are unchanged and are restated above as the
  authority. Only the *mechanism* changes: path partition in CI instead of `CODEOWNERS` plus
  two required approving reviews.
- **Supersedes the "enforced at review" clause of `plans/phase1-work-breakdown.md` §6.** That
  section already names the actual agents correctly — Forge/Anvil/Kiln, Scribe/Ledger, Crucible
  — and its prohibitions stand verbatim. What it got wrong is where enforcement lives: review
  is now the second line, not the first. Note that the file is **not yet on `main`** — it is
  PR docs#22, closed unmerged and pending T1 — so this supersession takes effect if and when it
  lands.
- **Does not touch** ADR-0001 through ADR-0004. Nothing here bears on the language decision,
  the crate set, the integration-boundary choices, or the workspace repository layout.

---

## 12. Amendment history

| | Date | Change |
|---|---|---|
| — | 2026-10-01 | Accepted as recorded. No amendments. |
