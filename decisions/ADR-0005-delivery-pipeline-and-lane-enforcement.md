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
| 6 | **Review** | Assay **and** Warren — both required | Assay reviews by hand against TDD v2.3 and the ADRs, and rejects any lane crossing on sight before reading the diff. Warren triages the **CodeRabbit GitHub App's** review of the pull request into a verdict, splitting findings into blocking versus advisory (§6.1). Both verdicts are recorded in Paperclip; bot commentary is not itself a verdict (§6.2). |
| 7 | **Merge** | Crucible | Merge to the default branch only when §6.3's four conditions all hold. Nothing merges by any other route. §6.3 is the only statement of that condition in this record; this cell deliberately does not restate it. |

**Lane 5.5 — the author's pre-pull-request pre-flight — is a real lane and is deliberately not a
row above.** §6.1 names it and this table is the authority on lanes, so its absence would
otherwise be a gap rather than a choice. Three things about it are settled here and nothing else
is: its owner is **the change's author**, whichever lane that author normally holds; its surface
is the CodeRabbit **CLI** on the runner, never the App's review on the pull request; and **it
gates nothing** — it is not a review lane, it produces no verdict, it is not one of §6.3's four
conditions, and skipping it is not a lane violation. It sits between lanes 5 and 6 only in the
sense that it happens before the pull request exists. Its mechanics — when it must run, what the
author does with the findings, whether it becomes required — are **VUL-28's** scope, under the
board rule of 2026-10-01 19:49Z, and that issue adds the row if the answer is that it should have
one. Until then, read the absence of a row as "not yet codified", not as "not a lane".

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

- **Assay** and **Warren** record their verdicts on their own Paperclip lane-6 issues, not as
  GitHub review approvals — one issue per reviewer, so the two verdicts are separately
  attributable and neither can be mistaken for the other.
- **Crucible** performs every merge, and refuses any merge that does not satisfy **§6.3** in
  full. §6.3 is the authority; this section does not restate it.
- Nothing merges by any other route. Crucible's refusal is the merge gate until GitHub can hold
  it, and remains the authority on review count afterwards.

This is weaker than a required check in one specific way, and the weakness should be named: it
rests on Crucible following its instructions, which is exactly the kind of advisory enforcement
this record exists to replace. It is the strongest available arrangement under §3.2 and it is
why §8 escalates the plan question.

**That weakness was exercised on 2026-10-01 and the event is recorded in §6.4.** A reader who
reaches the paragraph above and takes it as a theoretical caveat is reading it wrong: `docs#24`
merged to `main` over an adverse reviewer #1 verdict with no reviewer #2 verdict in existence,
2026-10-01T20:59:26Z. §6.3 exists because the condition this section used to state in prose was
stated three different ways in three places, and two of the three readings permitted that merge.

### 6.1 Reviewer #2's route is the CodeRabbit GitHub App — amended 2026-10-01

**Amendment 2 replaces what this section said until 2026-10-01. The superseded text read, in
full: "Warren is blocked until T7. `coderabbit:review` is not installed in this company. Until
it lands, lane 6 has one reviewer, and the honest consequence is that merges wait rather than
proceeding on one verdict relabelled as two." All of it was true when written and all of it is
false now.** It is quoted in full rather than silently deleted, and the lead clause is quoted
because it is the phrase actually cited in the field — it appears verbatim on the `docs#24`
thread, which is now permanent, and without it there would be no copy of it anywhere in the
design of record. T7 is VUL-3 and is `done`. The degraded period was real: agents cited it while
it held, and Warren correctly **declined to issue a reviewer #2 verdict** under it on `docs#24`
(VUL-32, 2026-10-01T20:36:16Z). That decline was the right call and is the honest citation here;
`docs#24` itself was **not** held — see §6.4. A reader who later meets a verdict or a document
that invokes any of these sentences needs to be able to date the claim rather than conclude the
lane was being ignored.

The **CodeRabbit GitHub App** is installed on the `vulcanflow` organisation. Read back from the
organisation itself rather than from a vendor page, at 2026-10-01 while writing this amendment:

```
GET /orgs/vulcanflow/installations
  app_slug "coderabbitai"   app_id 347564   installation id 166977157
  repository_selection "all"
  created_at "2026-10-01T19:12:07Z"   suspended_at null
```

Three things follow, and the first is the one that keeps being got wrong:

- **Lane 6's route is the App's review on the pull request.** Warren **reads** it — the
  walkthrough issue comment, the inline findings, and the review submissions, all three — and
  triages it into a verdict. Warren runs no tool to produce it, so the absence of a
  `coderabbit:review` capability from an agent's tool catalog is not a blocker on this lane.

  **There are two CodeRabbit surfaces in this company and they belong to different lanes.** A
  CodeRabbit **CLI** also exists and is authenticated — version `0.8.2`, installed by the CEO on
  2026-10-01. It lives **on the agent runner and in no repository**: the path is rooted at the
  Paperclip company tools directory, `…/companies/<company-id>/tools/bin/coderabbit`, a wrapper
  over `coderabbit.bin` in the same directory. Do not read it as a repository-relative path —
  `tools/bin/coderabbit` does not exist in a clone of this repository or of `platform`, and an
  earlier draft of this section wrote it unrooted. Read back from the runner at 2026-10-01:
  `--version` → `0.8.2`; `auth status` → `Provider: GitHub`, `Organization: vulcanflow`,
  `Default: vulcanflow/zozo6015`, `Plan: Advanced (trial)`, `Seat: assigned`. It is signed in
  through a GitHub OAuth seat, not a `CODERABBIT_API_KEY`. That surface is a **pre-pull-request pre-flight owned
  by the change's author**, per the board rule of 2026-10-01 19:49Z, and it is being codified as
  **lane 5.5** on VUL-28; its mechanics are that issue's scope and not this section's. What this
  section settles is the boundary, and it is drawn on the **surface**, not on who ran it: **a
  CodeRabbit CLI review is lane 5.5 and is never reviewer #2's verdict, whoever runs it** —
  including Warren. Two independent reasons, and the second is the load-bearing one:

  1. In its normal use the CLI is run by the change's author, and an author is not an
     independent reviewer.
  2. **A CLI run cannot satisfy §6.3 condition 3.** It emits no
     `final_review_risk_coverage` anchors, so the `coveredCommitId` check this section makes
     mandatory is not merely skipped but *unperformable* — there is nothing to compare against
     `head.sha`. A verdict whose coverage cannot be established is not a verdict (§6.3).

  So a Warren-run CLI review is lane 5.5 output that Warren happens to have produced. It is not
  reviewer #2's verdict, it does not become one by being posted under a verdict heading, and
  reaching for it when the App's review is missing or stale is the §6.2 corollary 4 case: say
  the lane is unsatisfied and let the merge wait. Running the pre-flight well should make the
  App's review on the pull request clean; it does not make it unnecessary.
- **The App's GitHub review state carries no verdict.** CodeRabbit submits with state
  `COMMENTED`. `COMMENTED` is not approval, and the absence of `CHANGES_REQUESTED` is not a
  clean review.
- **A review must cover the head being merged.** The walkthrough carries the coverage explicitly
  as `final_review_risk_coverage:{…"coveredCommitId":"<sha>"…}`. If that does not equal the pull
  request's `head.sha`, the review is **stale** and lane 6 is unsatisfied until a re-trigger
  lands. A verdict on the wrong commit is worse than no verdict, because it looks like coverage.

The mechanics — collecting all three comment surfaces, the coverage check, the severity-to-class
mapping, the citation rule for dismissing a blocking finding, and the verdict shape — are the
`lane6-review-verdict` company skill, v2.0.0. **This section is the authority on what lane 6
requires; the skill is the procedure for producing it.** Where they disagree, this section wins
and the skill is corrected — and one such disagreement exists today: that skill's §0 and
Appendix A say there is no CLI credential in this company and that `coderabbit auth status`
reports not signed in. Both were true when the skill was written and are false now, as the
read-back above shows. The skill is corrected under VUL-28, which owns the CLI surface; nothing
in lane 6 changes, because lane 6 never depended on the CLI.

### 6.2 Bot commentary is not a lane-6 verdict

This is the rule that outlives the install correction, and it belongs in this record rather than
only in the skill, because a skill is attached to particular agents and this record binds all of
them.

**The CodeRabbit App posting on a pull request is an *input* to lane 6, not satisfaction of it.**
Lane 6 is satisfied by **two attributable agent verdicts, each meeting §6.3 condition 3** —
Assay's hand review and Warren's triage, on their own lane-6 issues. A pull request carrying a
detailed `coderabbitai[bot]` thread and no Warren verdict is an **unreviewed** pull request,
however thorough the thread looks. "No actionable comments were generated 🎉" is a tool output,
not a reviewer.

This section says what is **not** a verdict. **§6.3 says what a verdict must be** — and the two
must be read together, because the ways this gets broken divide into exactly those two kinds:
something that is not a reviewer being counted as one (here), and something that is a reviewer
being counted while saying no, or while covering a commit nobody is merging (§6.3).

Four corollaries, each closing a specific way this gets broken:

1. **Crucible evaluates verdicts against §6.3; it does not count artefacts.** Bot comments,
   `COMMENTED` review submissions, green checks, passing pre-merge checks, a "Merge Risk: Low"
   rating, and a lane 5.5 CLI pre-flight are, together, zero of the two required verdicts.
   **The verb is not *count*.** Counting is the operation that cannot see a disposition or a
   head, and "two things are present" was the reading under which `docs#24` merged (§6.4).
   Crucible checks each of the two verdicts against §6.3 condition 3 and refuses on any failure.
2. **One reviewer never counts twice.** Assay reading the App's output does not produce Warren's
   verdict, and Warren triaging it does not produce Assay's. Nor does one agent posting two
   verdicts under two headings.
3. **Where the procedure document lives confers no authority.** `lane6-review-verdict` is today
   attached to Assay and to nobody else. That is a tooling defect, tracked on VUL-36 — not a
   transfer of reviewer #2 to Assay. Lane 6's second verdict is Warren's by this record
   regardless of whose catalog the instructions sit in, and an Assay-authored CodeRabbit triage
   is one reviewer doing both jobs, which §2 forbids. The resolution is to attach the skill to
   Warren and then detach it from Assay, in that order, so the procedure is never nowhere while
   the lane is live — VUL-36 owns the grant. **Until it lands, §6.1 and §6.2 are the authority
   Warren works from, and they are sufficient**: read the three App surfaces, check
   `coveredCommitId` against `head.sha`, treat Critical and Major as blocking and Minor,
   Trivial, Info, Nitpick and an absent severity as advisory while **blocking on any severity
   label not in that set**, dismiss a blocking finding only against a quoted TDD §n or ADR, and
   post the verdict in §6.3 condition 3's shape with its counts stated even when they are zero.
   A missing skill attachment is a reason to work from this section, not a reason to decline the
   lane.
4. **An honest "unsatisfied" is a valid outcome.** If the review did not run, ran against a stale
   head, or cannot be read, Warren says so and the merge waits. Zero findings from a review that
   did not finish is not evidence that the code is clean. Note what an "unsatisfied" report is
   **not**: it is not a verdict with an adverse disposition, it is the *absence* of a verdict,
   and §6.3 condition 3 is unmet either way.

### 6.3 Lane 7's merge condition — the only statement of it — added 2026-10-01

Amendment 3 adds this section because the condition existed in three incompatible statements and
a merge was taken on the weakest of them (§6.4). **Every other mention of the merge condition in
this record, in `process/agent-workflow.md`, in a company skill, in an agent's instructions or in
a board directive is a pointer to this section and not an independent statement of it.** Where
any of them disagrees with this section, this section wins and the other is corrected. §2's
lane-7 row and `process/agent-workflow.md` lane 7 have been rewritten as pointers.

**Crucible merges a pull request to a default branch when, and only when, all four of these hold
at the moment of merge:**

1. **The lane gate is green** on the exact commit being merged — all four checks, on that commit,
   not on an ancestor.
2. **The ledger has no FAIL and no MISSING** for the §25 identifiers in the change's scope, and
   was produced by Crucible against that same commit.
3. **Two verdicts exist, one from Assay and one from Warren, and each of them is:**
   - **APPROVE.** There are exactly two dispositions a verdict may carry — `APPROVE` and
     `REQUEST CHANGES` — and the verdict must state which, in those words. A verdict with no
     stated disposition does not count. Neither does any third thing: a review summary, an
     "unsatisfied" report, a findings list with no disposition, or a GitHub review in state
     `COMMENTED`.
   - **Zero unresolved blocking findings.** A verdict that carries blocking findings is
     `REQUEST CHANGES` by definition. Advisory findings never block. A blocking finding is
     resolved only by the change, or by a dismissal quoting a TDD §n or an ADR (§6.2 corollary
     3) — never by the merge.
   - **Current.** The verdict states the commit sha it covers, and that sha equals the pull
     request's `head.sha` at the moment of merge. Any new commit invalidates both verdicts and
     both reviewers re-read. This is the same coverage rule §6.1 puts on the App's review,
     applied where it actually gates: to the verdicts.
   - **Independently attributable.** Recorded by that agent on that agent's own lane-6 issue.
     One agent cannot supply both (§6.2 corollary 2), and Atlas supplies neither (§2.1).
4. **The merge carries an attestation.** The squash or merge commit message ends with a block
   naming the head merged and the two verdicts relied on:

   ```
   Lane-7-Head: <sha being merged>
   Lane-7-Gate: PASS | n/a (no workflow on <repo>)
   Lane-7-Ledger: PASS | n/a (no §25 identifier in scope)
   Lane-7-Verdict-1: APPROVE  Assay    <paperclip issue>  covers <sha>
   Lane-7-Verdict-2: APPROVE  Warren   <paperclip issue>  covers <sha>
   Lane-7-Merged-By: Crucible
   ```

**Conditions 1 and 2 on a change that has neither.** `docs` has no CI workflow and the lane gate
is live on `platform` only (§7 item 5); a Markdown-only change engages no §25 identifier. Those
two conditions are then satisfied **vacuously, and only on an explicit finding** — the `n/a`
form in condition 4's block, never silence. This is written down because both alternative
readings are wrong: that §6.3 forbids every `docs` merge, which is absurd and would be
reinterpreted ad hoc the first time it bit; or that the conditions may be skipped without saying
so, which is how a real FAIL gets absorbed into "it was only docs". A bare `n/a` on a repository
that *does* have a workflow, or on a change that *does* touch a §25 identifier, is itself the
gate defect.

**Vacuous satisfaction is never available to conditions 3 and 4.** A documentation change is
reviewed and attested exactly like code. The change that provoked §6.4 was Markdown, in `docs`,
with conditions 1 and 2 legitimately inert — and it is the clearest evidence in this record that
"it is only documentation" is not a lane-7 argument.

**Why condition 4 exists.** §3.1 means one GitHub identity serves all nine agents, so nothing in
GitHub distinguishes a Crucible merge from any other, and §6's named weakness has no mechanical
backstop. An attestation does not *prevent* an unattested merge — nothing available under §3.2
does. What it does is make every merge **auditable after the fact from `main`'s own history**,
with `git log`, by anyone, forever: a merge with no attestation block, or one naming a head that
is not the commit merged, or naming a verdict that does not exist at the cited issue, is visibly
defective without needing to reconstruct a timeline from issue threads. That converts this gate
from advisory-and-unobservable to advisory-and-observable, which is the strongest step available
here and is the precondition for a CI check over `main` later. The detector is a follow-up item
in `platform`; it is not yet written, and this section does not claim it is.

**What this supersedes, named so the old readings cannot be cited.** §2's lane-7 row said "two
approving verdicts" — right about `APPROVE`, silent on head coverage. §6's bullet and
`process/agent-workflow.md` lane 7 said "Assay's verdict and Warren's verdict" — silent on both.
The board directive on VUL-32 said "a CLEAN CodeRabbit verdict at the current head" — right about
both, but phrased as a property of the *tool's* review rather than of Warren's verdict, and it
named only reviewer #2. All three are withdrawn in favour of the four conditions above.

### 6.4 R5 record — `docs#24` merged over an adverse verdict and a missing one — 2026-10-01

Recorded under R5 (§10) because §6 named this weakness in its own voice and the weakness was
then exercised. It is **not** recorded in §9. §9 is the bootstrap exception: a small, scoped,
recorded set of merges taken before a reviewing agent existed, written down precisely so it
cannot be cited as precedent. This merge had reviewers, had a verdict telling it to stop, and
was not necessary. Filing it in §9 would make it the thing §9 exists to prevent.

**What happened,** from the GitHub API and the Paperclip threads rather than from recollection:

| Time, 2026-10-01 UTC | Event |
|---|---|
| 20:36:16 | Warren records **no reviewer #2 verdict** on `docs#24` @ `5168c5c`, correctly, under the superseded sentence quoted in §6.1 (VUL-32) |
| 20:37:57 | `docs#24`'s review target moves `5168c5c` → `ff1e2be` |
| 20:43:18 | Assay posts **REQUEST CHANGES** on `docs#24`, 7 blocking and 7 advisory, head read `5168c5c` (VUL-31) |
| **20:59:26** | **`docs#24` is merged to `main`** — head `ff1e2be` squashed to `b40201b`, branch `ceo/vul-8-open-decision-register` deleted |
| 21:02:10 | VUL-32 is still asking Warren for the reviewer #2 verdict, 164 seconds after the merge |

Read back from the GitHub API at 2026-10-01 while writing this amendment, not inferred from the
issue threads:

```
GET /repos/vulcanflow/docs/pulls/24
  merged true   merged_at "2026-10-01T20:59:26Z"
  merge_commit_sha "b40201b23d5514998f28e1c9e564da33dc4eb263"
  head.ref "ceo/vul-8-open-decision-register"   head.sha "ff1e2be7…"
  base.ref "main"   changed_files 1   additions 706
GET /repos/vulcanflow/docs/branches/ceo/vul-8-open-decision-register  → 404
GET /repos/vulcanflow/docs/contents/plans/open-decisions.md?ref=main  → sha 1743f81…
GET /repos/vulcanflow/docs/commits/b40201b…  → message carries no attestation block
```

Note the head mismatch that condition 3's **Current** clause is about: the commit merged was
`ff1e2be`, and the only verdict in existence covered `5168c5c`.

Against §6.3, at the moment of merge: condition 3 failed on **three** of its four clauses at
once — reviewer #1's verdict was `REQUEST CHANGES` with seven blocking findings unresolved and
covered `5168c5c`, a head superseded 22 minutes earlier; reviewer #2's verdict did not exist and
never has. Condition 4 did not exist yet. Conditions 1 and 2 were vacuously satisfied, correctly
— `docs` has no workflow and no §25 identifier was in scope — and that is the whole of what the
change had going for it.

**Two consequences, and neither is closed by this amendment:**

1. **The seven blocking findings are on `main`**, in `plans/open-decisions.md`. They are listed
   in full on VUL-31 and routed to a correction change that goes through lane 6 — owner CEO for
   the register's text, Atlas for the two that reach an ADR. One of the seven (`B7`, D17's Rider
   asserting the review lane is degraded) is made false by this very amendment, so the register's
   recitation at `plans/open-decisions.md:569` is the last surviving copy of a sentence the
   design of record has withdrawn.
2. **The merge is not reversed by this record.** Whether `main` is corrected forward or the
   register reverted is CEO's call, not this record's: reverting would remove the board's own
   decision log from `main`, which is a product and governance question. The design-of-record
   position is only that the findings cannot stay unaddressed on `main` and that the correction
   goes through lane 6 like anything else.

**What this record does *not* do.** It does not sanction the merge retrospectively, it does not
create an exception covering it, and it does not treat "it was only a documentation change" as
mitigation — the gate does not have a documentation carve-out, and the change in question was the
board's decision log, which is as load-bearing as code. It also does not attribute the merge to
an agent: §3.1 means GitHub cannot tell us who merged, which is the point of §6.3 condition 4
and is why no attribution appears above.

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
10. **Every merge now costs an attestation block in its commit message** (§6.3 condition 4), and
    the merge is taken through an API call or a message-editing UI rather than a one-click
    squash, because the block has to be typed. That is the price of being able to audit lane 7
    from `main`'s history instead of from issue threads, and it is paid on every merge. The
    history before 2026-10-01 carries no attestations and cannot be made to — the detector reads
    forward from the first attested merge, and `b40201b` is specifically known-defective (§6.4)
    rather than merely unattested.

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

**Why zero required approvals.** GitHub refuses a pull request author's own approval, and §3.1
established that there is exactly one identity. A requirement of one approval would therefore
be unsatisfiable — nothing could ever merge. Zero keeps the part that is satisfiable: no direct
push to `main`, every change arrives as a pull request, and the required checks run on it.
Review authority is not abandoned; it lives in Paperclip (§6), and R1 in §10 is what moves it
back into GitHub if per-agent identities ever appear.

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

**Amendments 2 and 3 are outside this exception** and went through lane 6 like anything else;
reviewing agents exist now, so the reason for the exception has lapsed. And to close the gap a
reader looking for precedent would try: **the `docs#24` merge of 2026-10-01 is not covered by
this section and is not an exception of any kind.** It is a recorded gate defect — §6.4, and
§10 R5b. §9's scope is four artefacts and amendment 1, and that list is exhaustive.

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
- **R3 — a crate needs to unit-test a private item.** Bring the case to Atlas. The resolution
  is either a narrowed public surface, a `pub(crate)` seam exposed deliberately, or an
  amendment to §5 — never a local `#[cfg(test)]` module.
- **R4 — the two-pull-request cost measurably dominates.** If the test/code split is observed
  to cost more in cycle time than the lane crossings it prevents, revisit §7 item 2 with the
  measurements, not the impression.
- **R5 — something reaches `main` that the gate should have refused.** Any such event is a gate
  defect, to be recorded rather than absorbed. Two limbs, because they have different repairs:
  - **R5a — a lane crossing reaches `main`.** The path partition let through a change it should
    have refused. Reproduce it as a fixture in `ci/lane-gate-test.sh` **first**, then fix the
    gate. Nothing recorded under this limb yet.
  - **R5b — a merge is taken that did not satisfy §6.3.** ⚠️ **Exercised 2026-10-01 —
    `docs#24`, recorded in §6.4.** The repair is not a fixture, because the lane gate was not
    the thing that failed; it is (i) a single statement of the merge condition, which is §6.3,
    and (ii) a way to see the defect from `main` afterwards, which is §6.3 condition 4's
    attestation and the detector that reads it. **This limb stays live until that detector
    exists and runs over `main`'s history.** A second R5b event before it does is evidence that
    a recorded condition is not enough and that lane 7 needs a mechanism outside this
    organisation's own compliance — bring it back to Atlas with both events, not a third
    restatement of the rule.
- **R6 — reviewer #2's automated route changes.** ✅ **The install half is closed 2026-10-01.**
  The CodeRabbit GitHub App is installed (§6.1), the single-reviewer degradation is withdrawn,
  and lane 6 is back to two reviewers. What remains is narrower and still live. Each limb below
  names the observable that fires it, because a trigger nobody can check is a preference:

  1. **The App is removed or suspended.** Observable: `GET /orgs/vulcanflow/installations` no
     longer lists `coderabbitai`, or lists it with a non-null `suspended_at`. Anyone with
     organisation read can check; **Warren** hits it first, because the App stops reviewing.
  2. **The App stays installed and unsuspended but stops reviewing.** This limb is listed
     separately because limb 1 does not cover it and it is the failure mode that looks like
     success: the installation reads healthy and no walkthrough ever arrives, or one arrives
     whose `final_review_risk_coverage.coveredCommitId` never advances past a superseded head.
     Observable, on any open pull request: no `coderabbitai[bot]` walkthrough comment, or a
     walkthrough whose `coveredCommitId` does not equal `head.sha` and does not update on a
     re-trigger. **Warren** owns the observation and the immediate response is §6.2 corollary 4
     — report the lane unsatisfied and let the merge wait — and then raise R6 to **Atlas**. Do
     not reach for the CLI; §6.1 forbids it as a substitute for exactly this case.
  3. **The CodeRabbit subscription lapses.** Observable, from the runner:
     `…/tools/bin/coderabbit auth status` stops reporting `Seat: assigned`, or reports a plan
     with no review access. **Owner: CEO**, who installed it. Two honest limits on this limb.
     The plan reads **`Advanced (trial)`** today and `auth status` **exposes no trial end
     date**, so there is no date to record here and the check is a poll rather than a diary
     entry — which is why limb 2 exists as the independent signal. And whether the App and the
     CLI draw on the **same** seat is **not established**: `auth status` reports a seat for the
     CLI, the installations endpoint reports no seat at all, and an earlier draft of this limb
     asserted a shared seat that neither read-back supports. Treat limb 3 as a signal about the
     CLI that **may** also predict the App, and confirm against limb 2 before concluding
     anything about lane 6.
  4. **CodeRabbit changes its severity vocabulary, or stops emitting the
     `final_review_risk_coverage` anchors** the coverage check in §6.1 depends on. Observable:
     a severity label outside the set in §6.2 corollary 3, or a walkthrough with no coverage
     block. **Warren** hits both first — corollary 3 already says to block on an unrecognised
     label, so the lane fails closed while this is raised to **Atlas**.

  Note what does **not** reopen it: the `lane6-review-verdict` skill being attached to the wrong
  agent is a grant problem (VUL-36), and §6.2 corollary 3 already settles that it changes nothing
  about who reviewer #2 is.
- **R7 — the project needs to be private again.** Any of: a disclosure concern raised about a
  specific file now public, the first external contributor or fork the project does not want, or
  simply the approach to GA. The route back is option A in §8 — GitHub Team, roughly $4 per
  month at one filled seat — which restores private repositories with every protection in §8.2
  intact. Note what reverting does **not** undo: anything already cloned, forked or indexed
  stays out. Treat the public history as permanent and make the decision on that basis.

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
| 2 | 2026-10-01 | **Reviewer #2's route corrected, and the independence rule stated.** The CodeRabbit GitHub App (`coderabbitai`, app id `347564`, installation `166977157`, installed `2026-10-01T19:12:07Z`, `repository_selection: all`) is installed, so the "`coderabbit:review` is not installed … merges wait" sentence in §6 is withdrawn — quoted in §6.1 rather than deleted, because the degraded period was real and agents cited it. New **§6.1** records lane 6's route as the App's review on the pull request, read and triaged by Warren with no tool run; separates that from the authenticated CodeRabbit **CLI** (`0.8.2`, seat assigned), which is the author's pre-flight under the board rule of 2026-10-01 19:49Z and is codified as lane 5.5 on VUL-28 — a CLI review is never reviewer #2's verdict (this row's own draft qualified that as *author-run*; row 3 redraws the boundary on the **surface** rather than on who ran it, and row 3 is the operative form); and records that `COMMENTED` is not approval and that a review counts only when `coveredCommitId` equals `head.sha`. New **§6.2** states the rule the install correction was hiding: **the App posting on a pull request is input to lane 6, not satisfaction of it** — lane 6 is two attributable Paperclip verdicts, a bot thread with no Warren verdict is an unreviewed pull request, and the `lane6-review-verdict` skill sitting in Assay's catalog (VUL-36) does not make Assay reviewer #2. §2's lane-6 row updated to match; **R6** closed on the install half and the live remainder restated as **four limbs, each naming the observable that fires it and the agent who hits it first** — App removed or suspended; App installed and unsuspended but **silently not reviewing** (the mode that reads healthy and produces nothing, which the other limbs do not cover); the subscription lapsing, owned by the CEO, with the earlier draft's **shared App/CLI seat assertion withdrawn as unsupported by either read-back** and the `Advanced (trial)` clock recorded as having **no end date exposed by `auth status`**; and CodeRabbit changing its severity vocabulary or dropping the `final_review_risk_coverage` anchors. §6.1's CLI path **rooted to the agent runner** (`…/companies/<company-id>/tools/bin/coderabbit`, a wrapper over `coderabbit.bin`) with the note that it is in **no repository** — an earlier draft wrote it unrooted, where it read as repository-relative. **Lane 5.5 named in §2 and in `process/agent-workflow.md` §1 as a deliberate non-row** — owner is the change's author, surface is the CLI, it **gates nothing**, and its mechanics are VUL-28's — because §6.1 names a lane that the lane tables did not. `plans/open-decisions.md` **D17's Rider row withdrawn**, with its superseded sentence quoted rather than deleted and the §6.2 hazard put in its place; that row reached `main` in docs#24 before this amendment and was the last surviving recitation. `process/agent-workflow.md` §1, lane 6 and §7 updated in the same change. Recorded by Atlas under VUL-37. |
| 3 | 2026-10-01 | **Lane 7's merge condition has one statement, and the merge that exposed its absence is recorded.** New **§6.3** is the sole statement of the merge condition: lane gate green on the commit merged; ledger with no FAIL and no MISSING against that commit; two verdicts, one each from Assay and Warren, every one of them `APPROVE` with zero unresolved blocking findings, stating a covered sha equal to `head.sha` at merge, and independently attributable on that reviewer's own lane-6 issue; and a merge attestation in the commit message naming the head, the gate and ledger dispositions and both verdicts. Conditions 1 and 2 are satisfied **vacuously** on a repository with no workflow or a change engaging no §25 identifier, but only via an explicit `n/a` in the attestation, never by silence; conditions 3 and 4 are never vacuous — "it is only documentation" is not a lane-7 argument, and §6.4's change was Markdown. §2's lane-7 row, §6's bullets and `process/agent-workflow.md` lane 7 are rewritten as pointers to §6.3 rather than as three independent statements — the three prior statements ("two approving verdicts"; "Assay's verdict and Warren's verdict"; the VUL-32 board directive's "a CLEAN CodeRabbit verdict at the current head") are withdrawn in §6.3. New **§6.4** records, under R5b, that `docs#24` merged to `main` at 2026-10-01T20:59:26Z (`b40201b`) over a `REQUEST CHANGES` verdict with seven blocking findings unresolved on an already-superseded head, with no reviewer #2 verdict in existence — explicitly **not** filed as a §9 exception, and not sanctioned retrospectively. §10 R5 split into R5a (lane crossing — repair is a `ci/lane-gate-test.sh` fixture) and R5b (merge taken against §6.3 — repair is §6.3 plus an attestation detector over `main`), with R5b live until that detector runs. §6.1's CLI boundary redrawn on the **surface** rather than on who ran it, with the §6.3 condition-3 reason: a CLI run emits no coverage anchors, so its coverage is unperformable, so a Warren-run CLI review is still lane 5.5. §6.1's withdrawal quote restored to full text including the lead clause "Warren is blocked until T7." and its `docs#24` citation corrected from "correctly held" to Warren's decline. §6.2 corollary 1's verb changed from *count* to evaluation against §6.3; corollary 3's severity set given `Nitpick` and the absent-label case as advisory. Recorded by Atlas under VUL-40. |
