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

### 3.2 Constraint two — branch protection was not available on this plan, and is now

**This constraint has been resolved. It is kept here because the resolution changed the
disclosure posture of the whole project, and that is not a detail anyone should have to
reconstruct from a settings page.**

As first recorded, `vulcanflow` was on the **free** plan with all four active repositories
**private**, and GitHub gates both branch protection and rulesets behind a paid plan for
private repositories:

```
GET  /repos/vulcanflow/docs/branches/main/protection  → 403 "Upgrade to GitHub Pro or make
PUT  /repos/vulcanflow/docs/branches/main/protection  → 403  this repository public to
GET  /repos/vulcanflow/docs/rulesets                  → 403  enable this feature."
GET  /orgs/vulcanflow/rulesets                        → 403 "Resource not accessible by integration"
```

The `PUT` was attempted, not merely the `GET`, so this was a confirmed write refusal and not a
read permission artefact. GitHub **Actions**, by contrast, worked normally throughout:

```
GET /repos/vulcanflow/docs/actions/permissions → {"enabled": true, "allowed_actions": "all"}
```

So the gate could **run** and go red on every pull request, while GitHub would not *refuse the
merge* on it, nor refuse a force-push, nor refuse an admin bypass. That was a billing decision
rather than an engineering one, and it was escalated to the board. **The board chose option B
of §8: make the four active repositories public.** The three enforcement powers are now live.
§8 records the decision and what it cost.

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
4. **The gate itself is tested.** `gate-self-test` asserts 21 fixture verdicts
   (`ci/lane-gate-test.sh ci/lane-gate.sh` → `21 passed, 0 failed`), so a future edit that
   silently guts the gate fails the gate.
5. **The gate is live on `platform` only.** `docs` has no code; `infra` and `vf-api` are empty
   repositories whose `main` is an initial commit. The gate is added to each the moment it
   receives source; a Markdown-only repository has no lane to cross.
6. **`platform#1` is unaffected but should be checked.** It bundles `.github/workflows/**`,
   `ci/**`, `crates/*/src/**` and `Cargo.toml` — all PROD and NEUTRAL, no test files — so
   `lane-partition` passes it. T1 should confirm before merging that it introduces no
   `#[cfg(test)]` module, or `inline-test-modules` will fail it.
7. **This record and the gate merged without the gate.** See §9.
8. **Branch protection, force-push refusal and admin-bypass refusal are live on all four
   repositories.** `main` is no longer writable directly on any of them, by anyone, including
   the org owner. See §8 for the settings and the evidence, and §8.3 for what this cost.
9. **The four active repositories are public.** Anyone can read the source, the pinned
   dependency set, the infrastructure manifests and the design record — including the TDD and
   every ADR. That is a standing condition of the project now, not a phase. See §8.3.
10. **The eleven private repositories are unprotected, and stay that way until something is
    pushed into one.** The board deferred the fix on 2026-10-02 (§8.5). The first push into a
    `vf-*` or `scanners` repository creates a repository holding code whose `main` is directly
    writable, force-pushable and admin-bypassable, with no required checks — and, because §8.1's
    secret scanning and push protection are available only on a public repository, with no
    credential guard on the push either. It converts the deferral into an open decision; the daily
    audit is what raises it, within a day, whether or not the agent that pushed noticed. Nobody may
    treat that state as acceptable because it was deferred — deferral was granted for *empty*
    repositories.

---

## 8. Decided: the plan question, and what the decision cost

Deliverable 3 of VUL-2 — branch protection on every active repository with
`enforce_admins: true` and `allow_force_pushes: false` — could not be applied on the free plan
with private repositories (§3.2). Three ways forward were put to the board:

| | Option | Cost | Effect |
|---|---|---|---|
| **A** | Upgrade `vulcanflow` to **GitHub Team** | list price ≈ $4 per user per month; one seat is filled today | Unlocks branch protection and rulesets on private repositories. Repositories stay private. |
| **B** | Make the four repositories **public** | free, in money | Same enforcement unlocked, but publishes a pre-GA security product's source, its pinned dependency set and its infrastructure manifests. |
| **C** | Change nothing | free | The gate runs and goes red, visibly, but GitHub will not block the merge on it; Crucible's refusal (§6) is the only thing holding the gate, and `main` stays force-pushable. |

**The board chose B on 2026-10-01.** CEO applied it. `docs`, `platform`, `infra` and `vf-api`
are public; `private: false` on all four, confirmed by read-back. The eleven empty `vf-*` and
`scanners` repositories were **left private** — they hold no commits, so publishing them buys
nothing.

### 8.1 What was checked before publishing

Visibility is effectively irreversible: a public repository is cloned, forked and indexed by
third parties within minutes, and flipping it back private does not retrieve those copies.
So the full history of all four repositories — 39, 20, 6 and 2 commits respectively, every
reachable ref, not just `main` — was scanned before the flip for credential material:
GitHub tokens in all five prefixes, AWS access-key IDs, PEM private-key blocks, Slack tokens,
OpenAI and Anthropic keys, GitLab PATs, Google API keys, JWTs, and assignment-shaped
`password`/`secret`/`api_key`/`token`/`credential` values. **No match in any repository.**
`infra/docs/secrets-and-signing.md` appears in history but not on any live ref, and every
`values.yaml` under `infra/platform/**` was checked for `adminPassword`, `clientSecret`,
`privateKey` and `bootstrapToken` — clean.

**Secret scanning and push protection were then enabled on all four repositories**, which the
free plan allows once a repository is public. Push protection is the part that matters going
forward: it refuses a push containing a recognised credential, which is a guard the project did
not have while the repositories were private and which it now needs more.

### 8.2 The settings applied

Identical on all four repositories, `PUT /repos/vulcanflow/{repo}/branches/main/protection`:

| Setting | Value | Buys |
|---|---|---|
| `enforce_admins` | `true` | The shared identity (§3.1) is also the org owner. Without this, every rule below is advisory again. |
| `required_pull_request_reviews.dismiss_stale_reviews` | `true` | A new commit voids prior approval. No approving a diff then changing it. |
| `required_pull_request_reviews.required_approving_review_count` | `0` | Deliberate, and not a weakening — see below. |
| `allow_force_pushes` | `false` | History on `main` cannot be rewritten. |
| `allow_deletions` | `false` | `main` cannot be deleted. |
| `restrictions` | `null` | No push allowlist; the pull-request requirement does the work. |

On `platform` only, additionally:

| Setting | Value |
|---|---|
| `required_status_checks.contexts` | `lane-partition`, `test-erosion`, `inline-test-modules`, `gate-self-test` |
| `required_status_checks.strict` | `true` — the branch must be current with `main` before merging |

`docs`, `infra` and `vf-api` have no workflow on `main`, so they have no required checks. A
required check that no workflow produces blocks every merge forever; the checks are wired per
repository as each gains CI, which is the same rule as §7 item 5.

**The two tables above are a record of what is applied, and §8.2.1 is the standard they are
held to.** They are not the same statement and must not be read as one: a snapshot dated
2026-10-01 cannot be the thing a later audit cross-references, because a snapshot has no
opinion about a repository that did not exist when it was taken.

**Why zero required approvals.** GitHub refuses a pull request author's own approval, and §3.1
established that there is exactly one identity. A requirement of one approval would therefore
be unsatisfiable — nothing could ever merge. Zero keeps the part that is satisfiable: no direct
push to `main`, every change arrives as a pull request, and the required checks run on it.
Review authority is not abandoned; it lives in Paperclip (§6), and R1 in §10 is what moves it
back into GitHub if per-agent identities ever appear.

### 8.2.1 Which contexts a protected repository must require

§8.2 said *how many* required checks `platform` has and said nothing about *which*. That gap is
not cosmetic. `ci/repo-protection-audit.sh` enforces R2 by asserting
`required_status_checks.contexts | length > 0`, so a repository requiring four checks named
anything at all — four checks that block nothing this organisation cares about — reads as
`ok, 4 required check(s), admins included` and exits 0. `platform` is that line today and is
correct only by coincidence. The audit cannot be sharpened first: a script cannot check a
required set that no record states, and a list invented inside `ci/` would put the policy in the
enforcement and leave the standard equal to whatever the script's author last typed. So the set
is named here, and the audit cross-references it afterwards.

**The names.** `NAMED_FLOOR` is these four:

| Context | Produced by |
|---|---|
| `lane-partition` | `.github/workflows/lane-gate.yml`, job `lane-partition` |
| `test-erosion` | same workflow, job `test-erosion` |
| `inline-test-modules` | same workflow, job `inline-test-modules` |
| `gate-self-test` | same workflow, job `gate-self-test` |

Read from `.github/workflows/lane-gate.yml` at `vulcanflow/platform@origin/main` on 2026-10-02,
and matching §4's table. Four names, not five: `audit-self-test` is defined on platform#8 and
reaches `main` only when that merges — see the derivation below, which is what admits it, and
§8.5's proposal 1, which is what wires it.

**The standard.** Three clauses, each decidable from the repository's default branch and its
protection object, and nothing else:

| | Condition | Requirement |
|---|---|---|
| **(a)** | the repository defines any job in `.github/workflows/lane-gate.yml` on its default branch | `required_status_checks.contexts` ⊇ `NAMED_FLOOR` ∪ *(every job name that file defines)* |
| **(b)** | the repository's default branch contains any path in §4.2's **PROD** class | it must define `.github/workflows/lane-gate.yml` — a repository holding production source with no lane gate is a gap whatever its required-check count |
| **(c)** | neither (a) nor (b) | the required floor is **empty**; `contexts: null` is conformant |

Clause (c) is a decision and not an omission. `docs`, `infra` and `vf-api` are in it today —
`contexts: null` on all three, read back 2026-10-02 — and they are conformant, because a
required context no workflow produces blocks every merge forever (§8.2) and a Markdown-only
repository has no lane to cross (§4.2 **NEUTRAL**, §7 item 5). This is per repository and not
per repository *class*: a class is a label somebody chooses, and the audit would then be
checking the label. The three clauses read the repository.

**Evaluated against the organisation as it stands, this standard produces no gap** — which is
stated because a standard that indicts something on the day it is written is usually describing
the wrong thing. `platform` is in clause (a): `lane-gate.yml` on `main` defines exactly the four
jobs above and protection requires exactly those four contexts, so the floor is complete and
there is no surplus. `docs`, `infra` and `vf-api` are in clause (c) on both limbs — no
`lane-gate.yml` and no `PROD` path on any of their default branches — so `contexts: null` is
conformant rather than tolerated. The eleven private repositories have no default branch at all
and stay `watch` under §8.5. All of it read back on 2026-10-02. The first gap this standard will
raise is `platform`'s own, on the day platform#8 merges and before CEO's `PUT` lands, and that is
the point of it.

**Minimum, not exact.** Clause (a) is `⊇`, deliberately, and the choice is material because it
is the whole difference in what a script refuses:

- **A missing floor member is a gap, unconditionally.** This is the case the count test cannot
  see, in either of its two shapes: *wrong names, right count*, and *right names, one missing*.
- **A context above the floor is conformant.** ADR-0002 already names CI gates that do not exist
  yet and will land on this branch as required checks: the single-crypto-provider confirmation
  (§3.6 — `cargo tree -i ring` empty and `cargo tree -d` showing no duplicated `rustls`, `hmac`,
  `sha2` or `digest`, to be wired into the workspace CI gate as part of
  `build/rust-supply-chain`) and the CRD drift gate (§6.3 — a job that regenerates the kopium
  types and fails on a difference). An exact-set rule makes each of those an amendment to this
  record before it can be wired, and a standard that makes routine work need an amendment is a
  standard that gets routed around.
- **A context above the floor is nevertheless *reported*.** `ok, 5 required (floor complete), 1
  above floor: cargo-deny` is a legitimate line; silence is not. An unrecognised required
  context is either a gate nobody wrote down or a decoy, and printing it is the only cheap
  defence against the second.

**Why `NAMED_FLOOR` is a lower bound on the derivation and not replaced by it.** The derived
half — every job the repository's own `lane-gate.yml` defines — is what makes the standard
survive growth: `audit-self-test` enters `platform`'s floor the moment platform#8 merges, with
no edit to this record. On its own it would also make the standard shrink on command, because
*deleting* a job from `lane-gate.yml` would legitimately remove its context from the floor. That
is the gate-self-weakening move in one commit to one GATE-class file, and `gate-self-test` does
not catch it: that harness asserts `ci/lane-gate.sh`'s verdicts, not the workflow's job list.
Under the union, deleting one of the four does not shrink the floor — the context stays required,
nothing reports it, and the repository jams shut. **Jamming `main` shut is the correct response
to deleting a lane gate.**

**What a rename actually costs, and the one move that is refused.** A renamed job is often
described as silently emptying the required set. It does not, and the correction matters because
the real hazard is one step further on. Rename the `test-erosion` job without the matching `PUT`
and the old context stays required and never reports: every pull request blocks forever, which
is loud, and which §8.2 already names. The danger is the fix. The shortest way to unjam a jammed
repository is to *delete* the required context — and that does silently empty the set, by a hand
that believes it is doing housekeeping. So:

1. **Renaming a job in `lane-gate.yml` is unfinished until §4's table, `NAMED_FLOOR` above, and
   `required_status_checks.contexts` all carry the new name.** One owner, one sitting — the same
   shape as §8.5's `PUT` obligation, and for the same reason: a change that is correct only after
   its second half lands is a change with a wrong state in the middle.
2. **A jammed repository is never unjammed by removing a required context.** It is unjammed by
   restoring the job name. This is written as a refusal rather than left to judgement precisely
   because it is the plausible-looking move under time pressure.
3. **The duplication between §4's table and `NAMED_FLOOR` is real and is not pretended away.**
   §8.2.1 is the source. The follow-up audit change mirrors these four names into exactly one
   GATE-class artifact on `platform`, one context per line, because a script cannot read this
   table — and a mirror that drifts from this record is caught by review against a published
   standard rather than by a script. That is weaker than mechanical, and it is still strictly
   better than today, where there is no published standard to review a drift against.

**What this does not close.** Two residual holes, named rather than left to be discovered:

- **The floor is a set of names, and a name does not identify a producer.** The legacy
  `contexts` array carries names only; the newer `checks` form carries an `app_id` alongside each
  — `15368`, GitHub Actions, on all four of `platform`'s contexts, read back 2026-10-02. An audit
  matching names alone therefore accepts a check of the right name from the wrong app. Pinning
  the producer is available and is **not** required here, because it needs a decision about which
  app ids are acceptable in a repository that may later legitimately use a second app, and no
  case is driving that decision yet.
- **Rulesets stay out of scope, unchanged.** §8.5 records that the audit deliberately does not
  evaluate a ruleset-governed branch against §8.2 and says *check by hand* instead. Nothing above
  restates this standard in ruleset terms; doing so would be a further amendment, not a line in a
  script.

**Relation to §8.5's proposal 1.** Clause (a) admits `audit-self-test` to `platform`'s floor
automatically on platform#8's merge, so from that moment `platform` requiring four contexts is a
gap the daily audit raises within a day. The obligation in §8.5 stands unchanged — §4's table
still has to list five checks and widen its lead sentence, since the fifth check lives in
`ci/repo-protection-audit-test.sh` and not in `ci/lane-gate.sh`, and §8.2's wiring table still
has to show five contexts — because those sections describe the checks and record what is
applied, which is not what §8.2.1 does. What changes is only that forgetting the `PUT` is now
detected rather than hoped against.

**The follow-up this record authorises, and its boundary.** `ci/repo-protection-audit.sh`
replaces its count test with a cross-reference against the three clauses above, on its own
pull request, as a GATE-class diff with its own lane 5.5 run. Acceptance, in one sentence: *the
audit reports a gap for a repository whose required contexts omit any member of the floor,
including the case where the count is right and every name is wrong, and reports a surplus
context without failing on it.* It earns a fixture before it earns a line of fix, per R-next —
at minimum *wrong names, right count*, *right names, one missing*, *floor complete plus a
surplus context* → `ok` with the surplus printed, and *a floor member required that no job
defines* → gap. **No §25 identifier attaches to any of this.** §25 is the product's traceability
matrix; a `ci/**` harness assertion is not in the 45 and inventing one for it would corrupt the
mapping's injectivity. The fixture names in `ci/repo-protection-audit-test.sh` are the acceptance
evidence instead.

### 8.3 What the decision cost, stated plainly

Option B bought enforcement with disclosure rather than with money. The VulcanFlow TDD, all six
ADRs, the open-decision register, the Rust workspace with its pinned toolchain and dependency
set, and the complete GitOps manifests for the platform — ArgoCD, Kargo, Keycloak, Harbor,
CloudNativePG, ClickHouse, SecureCodeBox — are now world-readable. For a pre-GA security
product this is a real cost in two ways: it hands a competitor the design, and it hands an
attacker a map of the infrastructure the product will run on. Nothing published contains a
credential (§8.1), so the exposure is of design, not of access.

This is recorded as the board's decision, not as a neutral default, so that a later reader does
not mistake an open repository for an absence of thought about it. **Option A remains available
for roughly $4 per month and is the only way back to private with enforcement intact** — see
R7 in §10.

### 8.4 The evidence that it works

Branch protection that nobody has watched refuse something is a settings screenshot, not a
control. Two refusals were observed, both as the org owner and repository admin:

**A direct push to `main` on `platform`:**

```
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote: - Changes must be made through a pull request.
remote: - 4 of 4 required status checks are expected.
 ! [remote rejected] main -> main (protected branch hook declined)
```

**A merge of a deliberate lane crossing** — platform PR [#4](https://github.com/vulcanflow/platform/pull/4),
one production file and one test file in one diff. `lane-partition` ❌ failed; `test-erosion`,
`inline-test-modules` and `gate-self-test` ✅ passed, so the gate refused the specific violation
rather than the pull request wholesale. `mergeable_state` was `blocked`, and the merge API
refused outright:

```
PUT /repos/vulcanflow/platform/pulls/4/merge  →  405
{"message": "Required status check \"lane-partition\" is failing."}
```

PR #4 is closed unmerged and its branch deleted. It is the successor to PR #3, which showed the
check going red back when red had no consequence; #4 is the half that was missing.

### 8.5 Decided: the eleven private repositories stay on Free, and the deferral has a control

Option B covered four of the organisation's fifteen repositories. The other eleven — the ten
`vf-*` service repositories and `scanners` — stayed private, and on the Free plan a private
repository cannot be protected at all. The same three options applied to them, and they were put
to the board on 2026-10-02 with option A recommended.

**The board chose to defer on 2026-10-02.** The organisation stays on Free; the eleven stay
private and unprotected; the question is revisited at the first push rather than now.

The reasoning that makes deferring defensible is narrow, and worth stating exactly because it
expires: **all eleven repositories are empty.** Not "nearly empty" —
`GET /repos/vulcanflow/{repo}/commits` answers `409 Git Repository is empty` on every one of
them, and none has been touched since 2026-09-09. A repository with no commits has no history to
force-push, no `main` to bypass and no lane to cross. The protection those eleven lack is
protection of nothing.

That stops being true at the first push, and nothing about a push announces itself. A deferral
whose safety depends on somebody remembering to look is not a control, so the deferral was given
one:

- **`ci/repo-protection-audit.sh`** on `platform` asks the organisation one question — *is there
  a repository that holds code and is not protected?* — and exits non-zero if there is. An empty
  unprotectable private repository reads as `watch`; the same repository with one commit in it
  reads as `GAP`. It also reports a repository that has pull-request workflows and **zero**
  required status checks, which is the mechanical half of **R2**.
- It **cannot** run as a pull-request check: it reads every repository in the organisation, and
  the per-repository `GITHUB_TOKEN` in Actions cannot. It runs from the Paperclip routine **"Org
  repo protection audit"**, daily at 07:00 UTC, assigned to CEO, whose instructions put a `GAP`
  back on the decisions desk as the same A/B/C choice rather than letting whoever is on shift
  improvise a fix.
- **It has been seen to fail**, to the standard §8.4 sets. It cannot be demonstrated against the
  live organisation without creating the gap it looks for, so it is demonstrated against fixture
  organisations: `ci/repo-protection-audit-test.sh` drives it through a stub `gh` and asserts
  nineteen verdicts, among them *code pushed into an unprotectable private repository* → `GAP`,
  `enforce_admins: false` → `GAP`, a force-pushable `main` → `GAP`, workflows with zero required
  checks → `GAP`, and an **unreadable organisation → exit 2, never 0**. That harness is hermetic,
  so unlike the audit it does run in CI, as the `audit-self-test` job.
- **Six of those nineteen exist because the lane-6 review found them**, and four of the six are
  cases where the audit returned a *clean verdict having examined nothing*: a repository list that
  failed to parse, and a read of `.github/workflows` that failed with anything other than a 404,
  both reported `0 gap(s)` and exited 0 in the first draft. An audit that fails open is worse than
  no audit, because it also removes the suspicion that there is nothing watching. The other two
  are false-verdict cases — required checks reported in `required_status_checks.checks` rather than
  `contexts`, and a branch governed by a **ruleset** rather than by classic protection. On the
  second, the audit deliberately does **not** evaluate rulesets against §8.2: a ruleset's rules
  are not those fields, and mapping one onto the other is how a weak ruleset comes to read as
  protection. It fails closed and says *check by hand*. Restating §8.2 in ruleset terms would be
  an amendment, not a line in a script.
- Live verdict on the day of the decision: `4 ok, 11 watched (empty, unprotectable), 0 gap(s)`.

**Two things in this subsection are proposals and not decisions, because they fall inside
sections other open amendments are rewriting.** They are named here so they are reviewed rather
than absorbed:

1. **`audit-self-test` is a reporting check until someone makes it required, and §4's table is
   the only thing that says which it is.** Making a check required before its job exists on
   `main` blocks every open pull request whose branch predates it, on a check that can never
   report — so the `PUT` is CEO's and it comes after platform#8 merges. That is a sequence, and a
   record that dates itself to one point in a sequence is wrong at every other point: a sentence
   saying "five required checks" is false before the `PUT`, and a sentence saying "four" is false
   after it. Neither is written here. The obligation is written instead, and it is one sentence:
   **the `PUT` that adds `audit-self-test` to `required_status_checks.contexts` is not finished
   until §4's table lists five checks and §8.2's `platform` row lists five contexts.** Whoever
   executes it owns both edits in the same sitting. Two things to get right when the row goes in:
   §4's lead sentence says every check lives in `ci/lane-gate.sh` and the fifth does not — it is
   `ci/repo-protection-audit-test.sh`, so the sentence needs widening and not just a row — and §4
   is the section amendment 5 (docs#36) is already rewriting, so whichever of the two lands second
   carries the reconciliation. Until §4 says five, it is describing four required checks
   correctly, and `audit-self-test` reports without blocking.
2. Classifying `ci/repo-protection-audit.sh` and its harness as **GATE** is proposed in
   platform#8 and is **not** recorded in §4.2 here. It is the same question §4.4 (docs#33) and
   §4.5 (docs#36) are deciding for detector/harness pairs under `ci/**`, and §4.5's position is
   explicitly that the mechanism is stated over any detector/harness pair *without reclassifying
   anything*. Whoever lands those two owns whether the audit pair is GATE, NEUTRAL-with-a-named
   owner, or covered by §4.5's mechanism; platform#8's classifier change is reviewable on its own
   merits and is reversible in one line. The lane-gate harness's assertion count also moves with
   platform#8 — three new fixtures, three new asserted verdicts — and reconciling the numbers in
   §4 and §7 item 4 belongs to amendment 5, which is rewriting both.

**What this decision does not buy.** The eleven repositories are not protected, and the audit does
not protect them — it notices. Three things follow from that, and each of them is a limit on the
control rather than on the decision.

*It bounds detection. It does not bound the answer.* The routine runs daily at 07:00 UTC, so a push
becomes a card on the decisions desk within about twenty-four hours of landing. What happens after
that has no bound on it at all: it is the same A/B/C choice, which has been open since 2026-10-01
and has already been deferred once — by this amendment. So there are two windows and only the first
one is closed. Push → card is at most a day. Card → answer is open-ended, and for the whole of it
`main` in that repository is directly writable, force-pushable and admin-bypassable, and the lanes
there are advisory. Reading "we will see it within a day" as "it is closed within a day" is the
specific error this paragraph exists to prevent, and the shortest correction to it is that the audit
guarantees nobody can say they did not know.

*The eleven have no credential guard either, and that is the larger exposure.* §8.1 records that
secret scanning and push protection were enabled on the four public repositories — which the Free
plan allows only once a repository is public — and calls push protection "the part that matters
going forward". The eleven private ones therefore have neither. So the first push into one of them
does not merely land on an unprotected `main`: it lands with nothing inspecting it for a credential,
into a repository whose history then cannot be rewritten clean by anyone who respects §8.2's
`allow_force_pushes: false`. For a pre-GA security product that is a worse outcome than the
branch-protection gap, and it is the one the audit cannot see at all — the audit reads protection
settings, not blobs. Nothing in this decision addresses it. Both routes out of it are the same two
that close the protection gap, because both of them turn on the plan or on the visibility.

*The audit tests less of §8.2 than its own failure text claims.* It evaluates `enforce_admins`,
`allow_force_pushes`, `allow_deletions`, and whether a repository that has workflows has zero
required checks. It does **not** read `required_pull_request_reviews` — so a repository whose
protection object exists but requires no pull request at all reads `ok` — nor
`dismiss_stale_reviews`, nor `required_status_checks.strict`. That is three of §8.2's six settings
and neither of the two `platform`-only rows. This is recorded as a **defect in
`ci/repo-protection-audit.sh`, not a narrowing of §8.2**: §8.2 is unchanged and remains the
standard the organisation is held to. Closing it is CEO's, under R2's surviving mechanical half,
and each missing setting earns a fixture in `ci/repo-protection-audit-test.sh` before it earns a
line of fix — the same rule R-next states for any other audit defect. Until it is closed,
`0 gap(s)` means *no repository failed the four things this audit tests*. It does not mean *the
organisation conforms to §8.2*, and the two sentences must not be used interchangeably.

Closing the protection gap itself costs either roughly $4 a month (option A) or that repository's
privacy (option B), and the board has decided to pay neither until there is something to protect.

---

## 9. The bootstrap, stated plainly

This record and the gate it describes were merged by CEO without passing through the pipeline,
because at the moment they merged the pipeline did not exist and no reviewing agent had been
approved. That is a genuine exception and it is written down rather than glossed, so that it
cannot later be cited as precedent.

Its scope is exactly: ADR-0005, `process/agent-workflow.md`, the `decisions/README.md` index
row, and `platform` PR #2. Nothing else. Every change after these goes through the gate, and
the gate's demonstrated refusals are linked from §8 of `process/agent-workflow.md`.

**Amendment 1 is inside this exception, and the scope is extended rather than stretched
quietly.** It is these same three files and no others; no reviewing agent is approved yet, so
there was still nobody to review it; and leaving it unmerged was the worse option, because
`process/agent-workflow.md` §7 on `main` said in so many words that GitHub cannot block a merge
on these checks. That sentence became false the moment protection landed, and a false "GitHub
will not stop you" in the document agents read mid-task reads as permission. A wrong operational
instruction on `main` is a live hazard; an unreviewed correction to it is a recorded one. The
exception does not extend past this amendment, and Atlas reviews both in place on approval.

---

## 10. Revisit triggers

Named, so this record is revisited on evidence rather than on mood.

- **R1 — per-agent GitHub identities appear.** If the board approves the nine hires and the
  broker returns a *distinct* `login` for any agent other than CEO, §3.1 is void. Re-examine
  `CODEOWNERS` over `tests/**`, `**/tests/**` and `fuzz/**`, and required approving reviews, as
  an *addition* to the path partition — not a replacement, since the partition is strictly
  stronger against a shared account and costs nothing once written.
- **R2 — the plan question in §8 is decided.** ✅ **Closed 2026-10-01.** The board chose B;
  protection is applied and demonstrated on all four repositories (§8.2, §8.4). What remains of
  this trigger is narrower and still live: **a repository that gains its first CI workflow must
  have its checks wired as required at the same time.** `docs`, `infra` and `vf-api` currently
  have none, so they are protected but have nothing required. Whoever adds the first workflow to
  one of them owns the `PUT .../branches/main/protection` that makes it required.
  **Which checks is no longer that person's judgement.** §8.2.1 names the set, so the `PUT` is a
  derivation from this record rather than a choice at the keyboard, and the trigger now has a
  second limb that fires without any workflow being added at all: a repository whose default
  branch gains a §4.2 **PROD** path must gain the lane gate too (clause (b)), and a code
  repository with no `lane-gate.yml` is a gap however many checks it requires. No new `R` number
  is minted for this — it is a sharpening of what R2 already owns, and §10's identifier space has
  four unmerged allocations in it (see the amendment-history row).
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
- **R7 — the project needs to be private again.** Any of: a disclosure concern raised about a
  specific file now public, the first external contributor or fork the project does not want, or
  simply the approach to GA. The route back is option A in §8 — GitHub Team, roughly $4 per
  month at one filled seat — which restores private repositories with every protection in §8.2
  intact. Note what reverting does **not** undo: anything already cloned, forked or indexed
  stays out. Treat the public history as permanent and make the decision on that basis.
- **R-next — the first commit lands in one of the eleven private repositories.** The deferral in
  §8.5 rests entirely on those repositories being empty, so the trigger is the moment one of them
  is not. The daily "Org repo protection audit" routine fires it: its `GAP` goes back to the
  decisions desk as option A, option B for that one repository, or an explicitly owned accepted
  risk. Two responses are **not** acceptable — publishing the repository without the §8.1
  pre-publication check, and widening the audit's tolerance until it reads green. If the audit is
  wrong, that is a defect in `ci/repo-protection-audit.sh` and it earns a fixture in
  `ci/repo-protection-audit-test.sh` before it earns a fix. **The routine does not stand down
  when the gap closes.** It is the only thing that runs the audit — §8.5 records why the audit
  cannot be a pull-request check — so standing it down retires R2's mechanical half with it, which
  is the half R2 was just upgraded to from "whoever adds the first workflow remembers". When every
  repository holding code is protected, drop the *cadence* to weekly and record the drop here; the
  routine itself is retired only by R1 or R7, which change who administers protection and
  therefore what is worth asking. Equally, a `GAP` that has been answered is closed by changing
  the repository, never by standing the routine down to stop it being asked.
  *(Identifier deliberately unallocated — see the amendment-history row. Four amendments with
  unmerged `R`-number allocations are open; amendment 6's own row records what collided last time
  this was guessed. Whoever merges this assigns the next free number and updates the three
  references to it: here, §8.5, and `decisions/README.md`.)*

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
| — | 2026-10-01 | Accepted as recorded. |
| 1 | 2026-10-01 | **The §8 plan question is decided: the board chose option B.** The four active repositories are public, branch protection is applied to all four with `enforce_admins: true`, and `platform`'s four lane-gate checks are required. §3.2 rewritten as a resolved constraint; §7 items 8–9 replaced; §8 rewritten as a decision with the pre-publication secret scan (§8.1), the applied settings and why zero required approvals (§8.2), the disclosure cost (§8.3) and two observed refusals (§8.4); R2 closed and narrowed to wiring checks per repository; R7 added as the route back to private. §7 item 4 corrected: 21 fixture verdicts, not 17. Recorded by CEO under VUL-2. |
| **(number unallocated)** | 2026-10-02 | **The eleven private repositories are decided: the board deferred, and the deferral has a control.** New **§8.5** — the decision; why deferring is defensible while all eleven answer `409 Git Repository is empty`; exactly when that expires; `ci/repo-protection-audit.sh` and the daily "Org repo protection audit" routine as the thing that notices, demonstrated against nineteen asserted verdicts over fixture organisations because it cannot be demonstrated live without creating the gap it looks for, six of them added from the lane-6 review and four of those six cases where the first draft returned a clean verdict having examined nothing; and what the decision does **not** buy — the audit notices, it does not protect, and the window between a push and an answer is up to a day wide. New **§7 item 10**. One new revisit trigger, **identifier unallocated**. `decisions/README.md` gains the closed-decision paragraph. **Scope held deliberately narrow.** §4, §4.2 and §7 item 4 are **not touched**, although the work behind this amendment bears on all three: `audit-self-test` is a reporting check until CEO wires it after platform#8 merges, the GATE classification of the audit pair is the question §4.4 (docs#33) and §4.5 (docs#36) are deciding and §8.5 records it as a proposal rather than as a decision, and the lane-gate harness's assertion count moves with platform#8 but reconciling §4's and §7's numbers belongs to amendment 5, which is rewriting both. **The amendment number and the trigger identifier are left unallocated on purpose.** Amendments 2 through 6 are allocated on four unmerged pull requests (docs#32, #33, #36, #37) and a seventh is drafting on VUL-105; amendment 6's own row records what happened the last time two branches each guessed an `R` number. Whoever merges this allocates both and updates the references. Drafted by CEO under VUL-2, in a scratch clone rather than the managed `docs` workspace, by a run holding the `platform` workspace — which is outside the §2.3.2 control and is disclosed here rather than left to be found. The §9 bootstrap exception does **not** cover it: all nine hires are approved, so it goes through lane 6 and lane 7 like any other change, and Atlas owns the framing. **Rewritten by Atlas on the same pull request after the lane-6 review, rather than merged and amended, because the findings were against the framing and that is Atlas's to fix.** Four changes, all inside §8.5 and R-next, none of them to the decision: (a) §8.5's `audit-self-test` item no longer dates itself to a moment in the merge/`PUT` sequence — it states the obligation instead, that the `PUT` is unfinished until §4 lists five checks and §8.2's `platform` row lists five contexts, and it names the §4 lead sentence as needing widening and not just a row, since the fifth check does not live in `ci/lane-gate.sh`; (b) "what this decision does not buy" now separates the two windows, because the daily routine bounds *detection* at about a day and bounds the *answer* at nothing; (c) the same paragraph records that the eleven private repositories have no secret scanning and no push protection either, which §8.1 enabled only on the four public ones, and states plainly that this is the larger exposure and that the audit cannot see it; (d) the same paragraph records that the audit evaluates three of §8.2's six settings and neither `platform` row — it does not read `required_pull_request_reviews`, `dismiss_stale_reviews` or `required_status_checks.strict` — recorded as a defect in the script and explicitly **not** as a narrowing of §8.2, with the fixture-before-fix rule attached; and (e) R-next no longer says to stand the routine down when the gap closes, which would have retired R2's mechanical half with it — the cadence drops to weekly instead, and only R1 or R7 retire the routine. Counts in this row and in §8.5 were read from platform#8 at head `777cb91`, not from the head the review ran against. |
| **(number unallocated)** | 2026-10-02 | **§8.2 names *which* contexts a protected repository must require, not just how many.** New **§8.2.1**: `NAMED_FLOOR` — `lane-partition`, `test-erosion`, `inline-test-modules`, `gate-self-test`, read from `.github/workflows/lane-gate.yml` at `vulcanflow/platform@origin/main` on 2026-10-02 — and a three-clause standard decidable from a repository's default branch and its protection object alone: (a) a repository defining any `lane-gate.yml` job must require `NAMED_FLOOR` ∪ every job that file defines, (b) a repository whose default branch holds a §4.2 **PROD** path must define `lane-gate.yml` at all, (c) anything else has an empty floor and `contexts: null` is conformant. **Per repository, not per repository class**, because a class is a label somebody chooses and the audit would then be checking the label. **A minimum and not an exact set**, because ADR-0002 already names CI gates that do not exist yet and will land as required checks on this branch (§3.6's single-crypto-provider confirmation wired into `build/rust-supply-chain`, §6.3's CRD drift gate), and an exact-set rule makes each one an amendment before it can be wired — but a surplus context is *reported*, since an unrecorded required check is either a gate nobody wrote down or a decoy. `NAMED_FLOOR` is a lower bound on the derivation rather than replaced by it: the derived half admits `audit-self-test` automatically when platform#8 merges, while the named half means deleting a job from `lane-gate.yml` jams `main` shut instead of shrinking the floor — which `gate-self-test` cannot catch, because it asserts `ci/lane-gate.sh`'s verdicts and not the workflow's job list. **The rename premise is corrected rather than adopted**: a rename without the `PUT` does not silently empty the required set, it jams every pull request forever; the silent emptying is the *fix* — deleting the required context to clear the jam — and that move is refused here by name. Two residual holes named and not closed: the floor matches context *names* and the legacy `contexts` array does not identify a producer (the `checks` form carries `app_id` `15368`, GitHub Actions, on all four of `platform`'s contexts, read back 2026-10-02), and rulesets stay out of scope per §8.5. **R2's live remainder is sharpened in place and no new `R` number is minted** — which checks to wire is now a derivation from §8.2.1 rather than a judgement at the keyboard, and clause (b) gives R2 a limb that fires on a first `PROD` path rather than on a first workflow. §8.2's two tables are labelled as a record of what is applied, explicitly not as the standard. **Scope held deliberately narrow.** §4 is **not** touched: §8.5's proposal 1 obligation stands unchanged — §4's table still has to list five checks and widen its lead sentence, since the fifth lives in `ci/repo-protection-audit-test.sh` — and §4 is the section amendment 5 (docs#36) is rewriting. `ci/repo-protection-audit.sh` is **not** touched: the cross-reference that replaces its count test is a GATE-class diff on its own pull request with its own lane 5.5 run, authorised and bounded at the end of §8.2.1 with its acceptance sentence and its four minimum fixtures, and filed when this lands rather than now (§4.2 is why the two do not travel together). **No §25 identifier attaches**: §25 is the product traceability matrix and a `ci/**` harness assertion is not among the 45, so the fixture names are the acceptance evidence. **The amendment number is left unallocated on purpose**, on the same basis as the row above: amendments 2 through 6 are allocated on four unmerged pull requests (docs#32, #33, #36, #37) and a seventh is drafting. Whoever merges this allocates it and updates `decisions/README.md`. Raised by the CodeRabbit CLI in round 2 of the lane 5.5 gate on platform#8 and deferred there as lane 1; recorded by Atlas under VUL-164. |
