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

The lanes and owners below are §4 of the VUL-1 plan, restated as the authority. **No agent holds
two adjacent numbered lanes of the table below on the same change.** That sentence is the one
Assay applies when it rejects a pull request before reading the diff, so its scope is stated here
rather than left to be inferred:

- **It ranges over the seven numbered rows and nothing else.** What it protects is specific: no
  agent may both produce a thing and hold the gate that clears it. A lane that clears nothing
  cannot be half of that pair, so the rule does not range over **lane 5.5** (below) at all.
- **Two specific non-adjacent pairings are normal and intended: 3 with 5, and 4 with 7.** Forge,
  Anvil and Kiln hold 3 and 5; Crucible holds 4 and 7. Lane 4 sits between 3 and 5, and lanes 5
  and 6 between 4 and 7, and that interposed gate is exactly what makes those two pairings safe.
  **This bullet blesses those two and no others.** Non-adjacency is a statement that the
  *adjacency rule* does not fire, not a statement that no other rule does — §4.4 forbids Assay
  and Warren from authoring a NEUTRAL `ci/**` script, which is a 3-and-6 pairing, for a reason
  the interposed-gate argument above does not transfer to. Read the earlier wording of this
  bullet ("one agent holding two non-adjacent lanes is normal and intended") as withdrawn: it was
  an affirmative licence over every pairing the adjacency rule misses, which is wider than the
  two cases it was written for.
- **Adjacency is by lane number, not by when things happen.** Two activities that run close
  together in time are not adjacent lanes.
- **Repository tooling under `ci/**` is lane 3's to implement, on lane 1's spec — §4.4 is the
  statement of that and this bullet is a pointer.** A script under `ci/` other than the three
  GATE paths is **NEUTRAL** by §4.2, so no row of the table below claims it and the gate
  classifies it as neither production source nor a test. That is a gap in the table, not a
  licence, and leaving it unassigned is how a gate script acquires an author nobody chose. **Who
  may author one, which agent within lane 3, and who may not — go to §4.4.** Two statements of
  this rule existed for one day, in this bullet and in §4.4, which is the failure mode §6.3 was
  written to stop; this is the pointer and §4.4 is the rule.

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
conditions, and skipping it is not a lane violation.

**Its number is a label, not a position in the gate sequence**, and the adjacency rule above does
not range over it. "5.5" says only that it happens before the pull request exists. An author who
holds lane 3 or lane 5 on a change may run the pre-flight on that same change, and that is not an
adjacency violation: the pre-flight clears nothing, so holding it alongside a coding lane gives
the author no authority it did not already have. Read the other way — 5.5 as a real lane wedged
between 5 and 6 — the rule would forbid every exercise of lane 5.5 by the only owner it has, which
is not a reading of §2 so much as evidence that §2 was phrased loosely; it is tightened above
rather than carved out here. Its mechanics — when it must run, what the
author does with the findings, whether it becomes required — are **VUL-28's** scope, under the
board rule of 2026-10-01 19:49Z, and that issue adds the row if the answer is that it should have
one. Until then, read the absence of a row as "not yet codified", not as "not a lane".

**This table assigns lanes, not path classes, and the difference bit once.** A reader looking for
who may touch a given file will not always find the answer here: the rows say what each agent's
lane *is*, while §4.2 says what the gate *does* with a path, and the two are not the same
partition. `ci/**` outside the three GATE paths is the case where that gap was load-bearing — it
is NEUTRAL, so the gate waves it through, and no row above claimed it. **§4.4 settles it** and is
the authority on NEUTRAL `ci/**` authorship; this table is not amended to carry a path column,
because the next such gap would then look like an omission in the table rather than a question
for §4.

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

### 2.3 One change, one branch, one author run — added and amended 2026-10-01

A lane says *who* may touch a change. It says nothing about *how many of them at once*, and that
turned out to matter. **A branch is owned by one issue and one author run at a time.**

**Read the three bullets below as good practice and §2.3.1 as the enforcement.** When this section
was first written the bullets were all there was, and they were advisory — which is why the thing
they describe recurred twice more and went to CEO as **VUL-83**. The control that now refuses it is
a Paperclip workspace lock, recorded in §2.3.1; the bullets survive because the lock does not reach
every shape of the defect, not because they hold the line. Concretely:

- The pull request names its **owning issue**. Another run — including another run of the same
  agent, on a different issue — does not push to that branch. It comments on the owning issue and
  lets that run carry the edit.
- **Corrections answering a review go in one commit from one run.** Two runs each answering the
  same verdict produce two partial answers to the same finding, and the reviewer re-reads both.
- A run that finds the branch has moved under it **reconciles forward** — reads what landed, keeps
  it where it is sufficient, and says in the commit message where it overrode it. It does not
  force-push, and it does not merge the branch into itself.

This is here because it was exercised three times on 2026-10-01, on this record's own pull
request, and each time the defect came from concurrency rather than from any agent's judgement:
**(i)** two Atlas runs each delegated a lane-6 re-read, producing duplicate review issues that
had to be withdrawn; **(ii)** one run wrote §6.4 consequence 1 while the other's commit falsified
it, and neither noticed — it took reviewer #1 to find it, as a blocking finding; **(iii)** both
runs then wrote independent fixes for the same five blocking findings and filed **two** board
issues for one attestation detector, one of which was cancelled as a duplicate.

**It is deliberately not an R5 event.** No lane was crossed, no verdict was counted that should
not have been, and nothing reached `main` that the gate should have refused — §10 R5 is about
those and this is not one. It is a sequencing defect inside lane 1.

**It recurred, and the closing sentence of this section was discharged rather than restated.** That
sentence said: if it happens again, the fix is outside this record, one run per branch is a
Paperclip-level control and therefore CEO's, and it comes back with the instances rather than as a
fourth restatement. Two more instances followed on the same day — a fourth and a fifth — and it went
to **VUL-83**, where CEO chose, applied and read back a control. §2.3.1 is what that decision binds
here. A sixth instance was observed by CEO while routing the decision, and is evidence of the same
mechanism rather than a new one.

### 2.3.1 The control is Paperclip's, not this record's — added 2026-10-01

**The authority for this subsection is CEO's decision comment on VUL-83**, with its evidence and its
cost. What follows is what that decision binds in this record, and it is deliberately shorter than
the comment: a second full statement of a decision is the failure mode §6.3 exists to end.

**It was an unset field, not a missing feature.** The VulcanFlow project carries
`executionWorkspacePolicy.sharedWorkspaceConcurrency ∈ {auto, serialize, allow}`. It was `null`, the
default. It is now **`serialize`**, read back from `GET /api/projects/1eb69546-cf8c-4210-a38c-72c66ad345c3`
on 2026-10-01:

```json
{"enabled": true, "defaultMode": "shared_workspace",
 "sharedWorkspaceConcurrency": "serialize", "allowIssueOverride": true,
 "defaultProjectWorkspaceId": "0347342b-4b10-429f-bcac-21ca23f6518e",
 "workspaceStrategy": {"type": "project_primary"}}
```

Mode, strategy and default workspace are the values that were already in effect, now written down.
The one behavioural change is `serialize`.

**The lock's domain is the workspace, and the record repository was not one** — which is why setting
that field alone would have prevented none of the five instances. All five were in `docs`, and `docs`
was registered nowhere, so every ADR edit ran in a clone no Paperclip control could see.
`vulcanflow/docs` is now a registered project workspace, read back from the same call: id
**`30a88b19-1c78-4db0-afe9-d7e66c494858`**, `sharedWorkspaceKey: vulcanflow-docs`, `defaultRef: main`,
`isPrimary: false`, created `2026-10-01T23:03:20Z`. §2.3.2 is the half of this that is procedure.

**Why the bullets above survive.** `serialize` refuses two runs *holding one workspace at the same
time*. It says nothing about two runs holding it in turn, and a branch outlives a run: the second run
to take the `docs` checkout can still push to a branch an earlier run opened for a different issue,
and can still add a second partial answer to a verdict an earlier run already answered. **One change
/ one branch / one author run, one correction commit, and reconcile-forward are therefore still the
rule for that case** — what changed is that they are no longer the only thing standing between two
live runs and one working tree.

**The override is asymmetric, and the asymmetry is the thing to watch.** `allowIssueOverride: true`
is deliberate. A **review-only** issue may carry
`executionWorkspaceSettings: {"sharedWorkspaceConcurrency": "allow"}`, because a reviewer reads the
pull request through the GitHub API and writes nothing to the tree — it is not the controlled party.
**An authoring issue may not.** If that carve-out is ever used to let a second author run through,
**that is the breach to record**, under R11 limb 2 — and it is a breach of this section, not an R5
event, for the same reason the six instances are not.

**The cost is throughput on `platform`.** For the record repository it costs approximately nothing:
one author, one branch at a time, which is what this section already asked for. For `platform` it is
real, and the reason is in the history: `GET /api/companies/98108baf-…/execution-workspaces` returned
**160 records on 2026-10-01, every one of them `mode: shared_workspace` and
`strategyType: project_primary`**, with 143 of them rooted at the same `platform` working tree. Lane
steps that used to interleave in that one checkout now take turns, and the cost is paid on every item
rather than only on the ones that were racing — a lock cannot tell a race from a coincidence.

**What is unverified stays unwritten.** The API exposes no read of the lock itself: no holder field,
no queue field. **Whether a second run queues behind the first or is refused outright is not
established, and this record asserts neither.** The revert is one call —
`PATCH /api/projects/1eb69546-cf8c-4210-a38c-72c66ad345c3` with `sharedWorkspaceConcurrency: "auto"` —
and it is **CEO's**, not Atlas's. R11 limb 1 names the observable that calls for it.

**What serialization does not fix** is duplicated *delegation*. Instances (i) and (iii) were two board
objects, not two working trees; the lock removes their cause only while both runs share a workspace,
which is too weak to rest the record on. §2.3.3 is the independent repair.

### 2.3.2 A record-authoring issue carries its workspace — added 2026-10-01

**An issue that edits the design of record carries `projectWorkspaceId: 30a88b19-1c78-4db0-afe9-d7e66c494858`.**
Until it does, the lock in §2.3.1 does not reach it. The project's `defaultProjectWorkspaceId` is
`platform`, so a record-authoring run that names no workspace works somewhere other than the `docs`
lock domain, and a lock over a workspace nobody is holding refuses nothing. **An authoring issue
without that field is outside the control** — stated this way because it is checkable by one read:
`GET /api/issues/{id}` → `projectWorkspaceId`.

Demonstrated on this amendment rather than asserted. VUL-91 reads
`projectWorkspaceId: 30a88b19-1c78-4db0-afe9-d7e66c494858`, and the run that wrote this section was
given `PAPERCLIP_WORKSPACE_ID` equal to that id with its working directory inside the project's
managed `docs` checkout — not a per-run clone. The same 160-record read in §2.3.1 shows **exactly one
execution workspace ever rooted at a `docs` checkout**, and it is this one; 143 are `platform` and 16
are a default path. So the claim that every prior ADR edit happened somewhere no control could see is
not an inference from the five instances — it is the absence of a record, and this is the first
record-authoring run this organisation has performed inside a lock domain.

**It does not apply to a lane-6 review issue**, and that is not an omission. A reviewer writes
nothing to a tree, which is the same premise §2.3.1's `allow` override rests on. Existing lane-6
issues carry the project default — VUL-84 reads `projectWorkspaceId: 0347342b-…`, `platform` — and
there is nothing to go back and correct.

### 2.3.3 A lane-6 review pair is keyed on (pull request, head sha) — added 2026-10-01

Instance (i) was two Atlas runs each delegating the same lane-6 re-read, producing duplicate review
issues, one of which then could not be withdrawn — the attempt returned `409`. The repair is not
serialization: it is that **the pair is not filable twice.**

Lane 6 is two issues, one per reviewer (§6.1). Each is created with an `idempotencyKey` derived from
the repository, the pull request number, the full head sha and the reviewer:

```
lane6:<owner>/<repo>#<pr>:<head-sha-40>:<assay|warren>
```

Three properties of that shape, each for a reason, because a key chosen by habit is a key that
refuses the wrong thing:

- **The head sha is in it, because a moved head legitimately needs a new pair.** §6.3 condition 3
  counts only a verdict covering `head.sha`, so re-filing at a new head is correct behaviour and must
  not be refused. A key on the pull request alone would refuse exactly that.
- **Nothing about the run, the agent, the clock or the issue is in it.** A key that varies with the
  caller is not a key, and the caller is the thing being deduplicated.
- **The reviewer is in it**, or one key would collapse the pair into one issue. The sha is the full
  40 characters and the repository is named: a pull request number is unique per repository, not
  across them, and an abbreviation is not an identity.

**File the pair through the call that takes the key.** `idempotencyKey` is a **required** field of
the `create_task` tool — schema read 2026-10-01: `minLength 1`, `maxLength 240`, described as a
"caller-stable retry key" — and that route makes the pair children of the authoring issue, so the
blocker edge that brings the verdicts back costs nothing extra. The OpenAPI document for
`POST /api/companies/{companyId}/issues` publishes **no request-body schema at all**, and that path
is **not** among the routes that document `idempotencyKey`; so **the HTTP route's idempotency is not
established and a lane-6 pair is not filed through it** until someone verifies it and records the
read-back. The key format above is independent of which route gains it.

**What the key buys is one issue per (repository, pull request, head, reviewer) — not a particular
status code.** Whether a second run presenting the same key receives the existing issue or an error
is not established here, and either outcome satisfies the requirement. The thing to check is the
count of live pairs at a head, not the response.

**Staleness is read from the title, not from the key, because the key is not stored on the issue.**
`GET /api/issues/{id}` exposes no idempotency field — checked on VUL-84, 2026-10-01 — so the head
goes in the title. **Two fragments are load-bearing and the rest of the title is the subject:**

```
… docs#<pr> @ <head-sha-7> …            the pull request and the head it covers
… (reviewer #<1|2>, <Assay|Warren>) …   which half of the pair this is
```

Lane 6 already writes titles this way — compare VUL-84, `Lane 6 — review docs#36 @ a342155
(reviewer #1, Assay): …`, and VUL-85, `Lane 6 — CodeRabbit review of docs#36 @ a342155 (reviewer #2,
Warren): …`. The two differ in the middle and that is fine; **this section fixes the two fragments
and deliberately does not fix the whole string**, because a prescribed title that practice already
diverges from gets cited as a defect in the practice rather than read as the convention it was meant
to capture.

A pair whose title sha is not the pull request's current head is stale **by inspection**. That is
what instance (i) needed and could not get, because the route it reached for was a withdrawal and the
withdrawal returned `409`.

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
| `gate-self-test` | runs the gate against 21 fixture diffs and asserts each verdict, so the gate's own logic is itself tested; and, under §4.5, refuses a head classifier that *permits* a case the base harness asserted the gate *refuses*, and refuses a fall in the harness's assertion count |

### 4.1 Why `lane-partition` is the load-bearing one

It makes lanes 2, 3 and 5 mechanical in a single rule. A coding agent **physically cannot** ship
a test edit alongside code: the combination is refused before anyone reads it. And a pull
request containing only test files is self-evidently a test author's work, whoever the commit
says authored it. The identity problem is routed around rather than solved — the gate stops
asking *who* and starts asking *what*.

The second half of the rule — a gate change may not travel with source or tests — closes the
**bundled** form of the obvious hole: weakening the gate in the same pull request as the change
it would let through. **It does not close the solitary form, and reading this paragraph as
closing gate-weakening in general is withdrawn.** A diff containing only GATE paths is
*permitted*, deliberately and asserted by fixture 6, because the gate has to be able to change
on its own pull request. What that permits, what closes it, and the one part that cannot be
closed inside this repository at all, are **§4.5**. Read this paragraph as scoped to bundling.

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

**That rule partitions *between* classes and says nothing about what travels *within* one, which
is where it bit.** All three GATE paths in one diff satisfies it — the classifier, the only
assertion that the classifier is correct, and the file naming the four checks, in a single pull
request `lane-partition` passes. **§4.5** is the statement of what that permits and what now
refuses it; the table above is **not** amended, because the defect is not that two files share a
class and no class split repairs it (§4.5, "What was not adopted").

### 4.3 Why `test-erosion` counts the whole tree

Scanning only the diff catches a weakened test but not a vanished one. `test-erosion` counts
test declarations (`#[test]`, `#[tokio::test]`, `#[test_case`, `#[rstest`, `proptest!`,
`fuzz_target!`) across the entire tree at the base and at the head, and fails if the number
falls. That catches a deleted test file, a test commented out, and a test quietly moved out of
the suite — while allowing a test to be *renamed or relocated*, because the count is preserved.

### 4.4 Who authors a NEUTRAL `ci/**` script — added 2026-10-01

§4.2 classifies every `ci/` script other than the three GATE paths as **NEUTRAL**, so authoring
one crosses no lane: `lane-partition` permits NEUTRAL alone and NEUTRAL alongside anything. But
"crosses no lane" is not the same as "has an owner", and until this amendment no row of §2's
table or of `process/agent-workflow.md` §1 claimed `ci/**` at all. Both now carry a pointer here
and **this section is the statement they point at**. §6.5's detector is the first such script
anyone has needed, and §6.3 flagged the gap as an open lane-1 question rather than let it be
filled in by whoever picked the work up. It is settled here, and settled generally, because the
detector will not be the last one.

**A NEUTRAL `ci/**` script is lane-3 work and moves through all seven lanes like code.** It gets a
lane-1 spec, a lane-2 fixture harness written first, a lane-3 implementation, a lane-4 run, two
lane-6 verdicts and a lane-7 merge. The one thing it does not get is a **§25 identifier**: §25 is
a traceability matrix over product requirements, and all 45 of its identifiers map to a
requirement in §2–§23 — none to the delivery pipeline's own tooling. The ledger entry is therefore
`n/a (no §25 identifier in scope)` under §6.3 condition 2, and the lane-1 acceptance statement is
the whole of the acceptance. Inventing an identifier would be worse than having none: **every §25
identifier names exactly one requirement**, and a fabricated row would break that in order to
record something §25 does not describe.

**The direction of that claim matters and an earlier draft of this section overshot it.** §25
holds **45 identifiers across 44 requirement rows** — read back from
`VulcanFlow_Technical_Design_Document_v2.2.md` §25 on 2026-10-01, where the §21 row
("Platform-error promotion gate and DR") carries two, `release/analysis` and `recovery/restore`.
So requirement→identifier is one-to-many and the mapping is **not** injective in both directions;
identifier→requirement is, and that is the only direction this argument needs. (Unrelated, and
not touched here: "one identifier, one test function" is a rule about the identifier→test
mapping, not about §25's own columns.)

**Within lane 3 the agent is chosen by subject**, on the same axis that already separates the
three coding agents in `process/agent-workflow.md` §1:

| Subject of the script | Owner | Why |
|---|---|---|
| Pure computation over its inputs — parses, compares, reports; no service, no cluster | **Forge** | Forge's lane is the pure library crates. A script that is a function of its inputs is the same kind of artefact. |
| Wraps, launches or exercises a service binary | **Anvil** | Anvil's lane is the service binaries. |
| Touches cluster state, manifests or a deployment | **Kiln** | Kiln's lane is Kubernetes and the `infra` manifests. |

**Four agents are excluded by name, each for a reason that is not availability:**

- **Crucible** may not author a `ci/**` script whose subject is lane 4 or lane 7 — that is, its
  own conduct. §6.2 corollary 2 already refuses to let one agent supply both lane-6 verdicts; the
  same principle refuses to let the audited party build its own auditor. Crucible still *runs*
  it, because lane 4's monopoly on execution is absolute; §6.5 names what makes that tolerable
  and what it leaves open.
- **Atlas** may not author it. Lanes 1 and 3 are adjacent under §2, and the spec exists precisely
  so that the implementer does not get to choose what the script must catch.
- **Assay** and **Warren** may not author it. They would then review their own artefact in lane
  6, which is the pairing §2's adjacency rule exists to prevent — even though 3 and 6 are not
  adjacent by number, so the rule as written does not reach it and this bullet is why.

**The fixture harness is lane 2, and it is not a TEST-class path.** `ci/lane-gate-test.sh` is the
precedent and it is GATE, not TEST, so §4.2 has never classified a gate script's own self-test as
a test file; a NEUTRAL script's harness is NEUTRAL for the same reason. `test-erosion` counts Rust
test declarations, so it does not see a shell fixture either. The harness is therefore
**mechanically reachable by a coding agent** — which is exactly the loophole §2.1 exists to close,
so say plainly where the control actually sits. It is not the gate; it is lane 1: **the fixture
set is enumerated in the spec by Atlas — each fixture with the outcome it must produce, not just
its name — and written by Scribe before the implementation exists.**

**Three honest statements about how strong that is, because an earlier draft of this paragraph
made three claims that are false and a wrong operational instruction on `main` is a live hazard
(§9).**

- **The control is temporal, not structural.** The fixtures land first, so the implementer does
  not get to choose what the script must catch. It does **not** follow that the implementer
  cannot weaken them: nothing mechanical stops a coding agent editing the delivered
  `ci/lane7-attest-test.sh`, as the paragraph above has just finished establishing. Saying
  otherwise tells a reader that something it can plainly do is impossible.
- **The reviewer's check is per fixture against its stated outcome, not a count.** Counting
  catches *deletion* only. It does not catch *relaxation* — every fixture name preserved and
  every assertion hollowed out — which is the dominant weakening mode here and is the same
  distinction §4.3 already draws when it lets `test-erosion` wave through a rename. This is why
  the spec enumerates outcomes: a list of names is not reviewable against a delivered harness.
- **It is not the strongest arrangement available, and the stronger one is rejected on the
  record rather than unnoticed.** `ci/lane7-attest.sh`, its floor manifest and
  `ci/lane7-attest-test.sh` *could* be named in §4.2's GATE row — strictly stronger in one
  respect, since a GATE-class harness cannot ride in the same diff as PROD or TEST where a
  NEUTRAL one can accompany anything. **Rejected, for two reasons.** It does not close the hole
  it would be adopted for: script and harness would both be GATE, so they still move together,
  and the implementer can still weaken the harness in the same change. And GATE is §4.1's
  enforcement set — the paths whose edit must not ride with the thing they would have caught —
  so putting a script there that gates nothing inverts the classification and pre-empts **R8**,
  which exists precisely to fire when such a script *starts* gating something. R8 now names the
  GATE reclassification as the repair when that day comes.

So: this is weaker than the path partition, it is recorded as weaker rather than presented as
equivalent, and it is **the arrangement chosen** for a file class the partition deliberately
waves through — not the only one and not provably the best.

**The "they still move together" reason in the third bullet is now load-bearing twice, and
§4.5 is what answers it.** This section observed that moving a detector and its harness into GATE
does not stop them travelling in one diff, and used that to reject the move. Correct — and the
same sentence is true of the GATE pair that already exists. §4.5 supplies the mechanism a class
move was never going to supply, and it is stated over *any* detector/harness pair rather than
over GATE specifically, so the prospective `ci/lane7-attest.sh` pair inherits it without a class
change and without re-arguing this bullet.

### 4.5 The gate's own weakening — what the partition reaches, and the one thing it cannot — added 2026-10-01

Raised by Assay on **VUL-68**, against merged code at `platform@41506ad` rather than against any
diff under review, and against §4.1's claim rather than against an implementation bug. The
reading is confirmed in full, the limb Assay could not resolve is resolved here from a read-back,
a second limb is closed by a new check, and a **third limb is named as unclosable inside this
repository**, which is the part that matters most and was not in the finding.

**The reading, confirmed against `41506ad`.** `classify_path()` puts `ci/lane-gate.sh`,
`ci/lane-gate-test.sh` and `.github/workflows/lane-gate.yml` in **GATE**. `cmd_partition()` has
exactly two refusal conditions — PROD together with TEST, and GATE together with either — so a
diff holding all three GATE paths and nothing else gives `gate=3, prod=0, test=0` and
`lane-partition` **passes it**. `test-erosion`'s `count_markers` greps `-- '*.rs'`, so a diff with
no Rust in it leaves the base and head counts equal and passes. `inline-test-modules` passes.
Assay's further point is also confirmed: `ci/lane-gate-test.sh`'s `setup()` copies only
`ci/lane-gate.sh` into the fixture repository and creates neither `ci/lane-gate-test.sh` nor
`.github/`, so **the harness cannot construct the diff in question** — fixture 5 asserts *gate +
prod → fail* and fixture 6 asserts *gate alone → pass*, and there is nothing in between because
there is nothing in between that the harness can build.

#### Limb 1 — deleting or renaming a check — is already closed, by configuration

This is the fact VUL-68 asked for and could not read, and it decides the severity of the rest.
`GET /repos/vulcanflow/platform/branches/main/protection`, read back **2026-10-01**, returns
`required_status_checks.contexts` = `lane-partition`, `test-erosion`, `inline-test-modules`,
`gate-self-test`; `strict: true`; each context pinned to `app_id 15368`; and
`enforce_admins: true`. That is the configuration **§8.2** records as applied, still in place.

`15368` is **GitHub Actions**, read rather than recalled: `GET /apps/github-actions` returns
`{"id": 15368, "slug": "github-actions", "name": "GitHub Actions"}`. An earlier draft of this
section asserted that mapping in an uncited parenthetical, which is the recalled-not-read kind of
claim this record is supposed to refuse. The read-back is also **independently corroborated**:
Forge read the protection endpoint from a different runner under VUL-68 and verified the `app_id`
the same way, and Atlas read it twice; all three reads return the same four contexts, the same
`strict: true`, the same pinned `app_id` and the same `enforce_admins: true`. Three reads, two
agents, two routes, one configuration — which carries weight here because this limb is closed by
configuration alone and **nothing in the repository restates it**, so the citation is the only
record a later reader has.

Two consequences a reader can check:

- **A check that stops reporting blocks the merge.** Deleting the `gate-self-test` job, renaming
  it, removing `pull_request` from the workflow's `on:`, adding a `paths:` filter that excludes
  the diff, or deleting the workflow file outright all leave a required context that is *expected
  and never reported*. §8.4's observed push refusal — *"4 of 4 required status checks are
  expected"* — is the same mechanism seen from the other side and is the evidence that the four
  contexts are held **by name, in GitHub configuration, independently of any file in the
  repository**.
- **The pinned `app_id` means nothing else can satisfy them.** An external bot cannot report a
  check run named `gate-self-test` into these contexts.

So `lane-gate.yml`'s header sentence — *"Wire all four as required checks on main"* — reads as an
instruction but describes something already done, and §8.2 is where it is recorded rather than the
workflow comment. **VUL-68's conditional severity resolves to its lower branch for this limb, and
for this limb only.** The repository genuinely did not establish the fact; §8.2 did, and the
read-back above is now the citation.

One caveat, because the limb is closed on a *reported* check and not on a *correct* one: a job
that is **skipped** rather than absent may be treated by branch protection as satisfying its
context. That behaviour is **not verified here**, and §4.6 item 3 therefore refuses a job-level
`if:` on these four jobs outright rather than resting on which way it goes.

#### Limb 2 — the classifier and its own harness in one diff — was open, and is closed here

A diff holding `ci/lane-gate.sh` and `ci/lane-gate-test.sh` can weaken `classify_path()` and, in
the same change, remove or relax the fixture that would have caught it. `gate-self-test` runs the
**head** harness against the **head** classifier, so it is self-referential and reports green.
That was open on `main` and is closed by two sub-checks, added to the existing `gate-self-test`
job so that no new required context has to be wired:

1. **Monotonicity — a case the gate refused on `main` may not be permitted at head.** Run the
   harness **as it exists at the base commit** against the classifier **at head**. For every base
   fixture: if the base harness asserted **fail** and the head classifier gives **pass**, that is
   a blocking failure. The converse — base asserted **pass**, head gives **fail** — is *reported
   and does not block*, because the gate getting stricter is not a weakening. This is one
   sentence, it needs no filename convention, it reads the base from `git` rather than from the
   diff, and it refuses the whole of limb 2 **in a single pull request** rather than splitting it
   into two. The base harness is code already merged to `main`, so executing it introduces no
   trust the branch did not already have.
2. **An assertion floor.** The number of `check` invocations in `ci/lane-gate-test.sh` may not
   fall between base and head. This closes the two-pull-request route that monotonicity alone
   leaves open — delete the fixture in one change, weaken the classifier in the next, by which
   point the base harness no longer asserts the case.

Together these refuse: weakening the classifier (caught by 1, whatever the diff does to the
harness); deleting a fixture (caught by 2); flipping a fixture's expected verdict while leaving
its body intact (caught by the existing head-against-head leg); and hollowing a fixture's body
while leaving its expected verdict intact (also caught by the existing leg, since the hollowed
diff no longer produces the asserted verdict).

#### Limb 3 — the workflow file's job bodies — cannot be closed in this repository

**Branch protection pins job names, not job bodies.** A diff touching
`.github/workflows/lane-gate.yml` and nothing else is GATE-only, so `lane-partition` permits it;
`test-erosion` sees no Rust; `inline-test-modules` is unaffected. If that diff replaces each
job's `run:` step with a command that exits zero, **all four required contexts report success
while none of the four checks has run.** One file, four green checks, no gate. That is strictly
more reachable than the three-file sequence VUL-68 describes, and no edit to `classify_path()`
touches it, because the file is already alone in its class.

It is not an oversight and it is not fixable by a path rule. The workflow file is what decides
whether the gate runs, so nothing the gate does can be guaranteed to run — and **every layer
above bottoms out here**, monotonicity included, since its step lives in that same file. Say it
plainly rather than let §4.5's first two limbs read as a complete mechanism.

Three things follow:

- **The control on a GATE-only diff is lane 6, by construction.** §4.6 makes it checkable instead
  of leaving it to a reviewer's instinct.
- **`.github/workflows/lane-gate.yml` is the highest-consequence path in the repository**, and it
  is labelled as such here and in `process/agent-workflow.md` §7.
- **Any real closure is outside the repository or it is nowhere.** **R9** names it, states what
  was looked for and not found on 2026-10-01, and gives it an owner who is not Atlas.

#### What was not adopted, and why — on the record so it is not re-argued

- **A pairing rule: "`ci/X.sh` and `ci/X-test.sh` may not appear in the same diff."** Named by
  Assay as the alternative, and the appeal is real — it would reach the existing `lane-gate` pair
  and the prospective `lane7-attest` pair at once, with no new class. **Rejected, because
  monotonicity is strictly stronger on the same case and costs less.** The pairing rule *splits* a
  one-pull-request weakening into two and buys legibility; monotonicity **refuses** it. The
  pairing rule also rests on a filename convention, and the edit that defeats it — rename
  `ci/lane-gate-test.sh`, change the matching `run:` line — is a two-path GATE-only diff the rule
  itself permits. **An earlier draft of this bullet gave a third reason that is wrong and it is
  withdrawn rather than quietly dropped**: it said the pairing rule would force the classifier
  change and the fixture asserting it into separate pull requests, as a cost. R10's sequence puts
  them in separate pull requests *anyway*, so the pairing rule costs nothing there and the
  rejection rests on the two reasons above. Stated because a rejection standing on an argument its
  own record falsifies is worse than no rejection: the next reader re-opens it correctly.
- **A class split — one path class per detector, or moving `ci/lane-gate-test.sh` out of GATE.**
  §4.4's third bullet already rejected the GATE reclassification of the prospective `lane7-attest`
  triple with the reason *"both would still move together"*. That reason is right and it applies
  here: the defect is not that two files share a class, it is that **a detector and the only
  assertion that the detector is correct are the same change** — true in whatever classes they
  sit. A class split also costs one new class per detector as detectors multiply, which Assay
  named. §4.4's rejection stands and §4.2's table is unchanged.
- **A count-only floor, with no monotonicity.** Counts catch *deletion*, never *relaxation* — the
  distinction §4.3 draws when it waves a rename through and §4.4 draws about its own fixture set.
  The floor is adopted as the second layer, not as the answer.

#### What remains open, stated so nobody reads it as closed

**In-place relaxation of a fixture.** One pull request, harness only: replace a fixture's body
with a weaker case *and* flip its expected verdict to match, so the `check` count holds, the
head-against-head leg is green, and the base-against-head leg still runs the real fixture against
an unchanged classifier. The next pull request weakens the classifier and the base harness no
longer asserts the case. Both are green. The control is a reviewer reading **each fixture against
the outcome the lane-1 spec states for it** — §4.6 item 4, the same control §4.4 settled on for a
NEUTRAL harness, recorded here at the same strength: weaker than the partition, and chosen.

**And the floor is an endpoint comparison, with the blindness that implies.** Adding one fixture
and removing another in one diff preserves the count. This is the same shape as fixture 12's
second half, which asserts that adding and then removing a `fuzz_target!` within one pull request
passes `erosion` *"vs baseline count is unchanged"* — Assay flagged it, it is deliberate and
documented, and it is unchanged here. Monotonicity does **not** have this shape: it compares base
assertions against head behaviour, not two endpoints of a count.

#### Lane assignment for the work §4.5 creates, and the order it has to go in

§4.4's subject table governs, extended to GATE paths because the axis is the same: `ci/lane-gate.sh`
is pure computation over its inputs, so the implementation is **Forge** in lane 3; the fixtures are
**Scribe** in lane 2, enumerated with their outcomes in the lane-1 spec per §4.4; the runs and the
merge are Crucible's. §4.4's four exclusions apply unchanged. The work carries **no §25
identifier** — `n/a (no §25 identifier in scope)` under §6.3 condition 2, for §4.4's reason: all 45
identifiers map to a product requirement in §2–§23 and none to the delivery pipeline's own tooling.

**The order is forced by R10 and it is not the ordinary one, so it is written out rather than left
to be discovered.** `gate-self-test` is a required check, so a harness asserting behaviour the
classifier does not yet have is a red required check and cannot merge at all. A new gate behaviour
therefore needs **three** pull requests, not two, and in this order:

| | Lane | Author | Diff | Why it is green |
|---|---|---|---|---|
| 1 | 2 | **Scribe** | `setup()` gains the ability to write `ci/lane-gate-test.sh` and `.github/workflows/` into the fixture repository, plus fixtures asserting the verdicts the gate **already** gives on a GATE-pair diff and on a workflow-only diff | They assert current behaviour, which is exactly what was missing — the absence of these two fixtures is how limb 2 survived from `41506ad` to VUL-68 |
| 2 | 3 | **Forge** | monotonicity and the floor added to `ci/lane-gate.sh`, wired as steps in `gate-self-test` | Additive. Every existing fixture, including pull request 1's, is unaffected |
| 3 | 2 | **Scribe** | fixtures asserting the new sub-checks' verdicts | The behaviour now exists, so the assertions pass |

**Nobody crosses a lane and nothing is ever red.** What is inverted is only that the *new*
behaviour's fixtures arrive after it, in pull request 3 — R10's exception, confined to one pull
request, and the reason §4.4's "written by Scribe before the implementation exists" clause cannot
hold for a GATE path. The substitute is lane 1 and lane 6, not a mechanism: Atlas enumerates pull
request 3's fixtures **with their outcomes** before pull request 2 is written, and §4.6 item 4 is
Assay checking the delivered harness against that enumeration per fixture. Recorded as weaker than
the ordinary arrangement, for the same reason §4.4 records its own as weaker.

### 4.6 A GATE-class pull request has a named review obligation — added 2026-10-01

§4.5 limb 3 cannot be mechanised, so for a diff touching any GATE path the review **is** the
control rather than the second line. Enumerate it: "review it carefully" is not a specification,
and §4.4's own correction established that a reviewer needs the outcome stated rather than the
name. Assay's lane-6 verdict on such a pull request states each of these per item and cites the
line it read.

1. **The four job names in `.github/workflows/lane-gate.yml` are unchanged**, and the same four
   still appear in `required_status_checks.contexts` — quoted from both sides, four and four.
2. **Each job still invokes the gate.** `lane-partition` → `ci/lane-gate.sh partition`;
   `test-erosion` → `ci/lane-gate.sh erosion`; `inline-test-modules` → `ci/lane-gate.sh
   inline-tests`; `gate-self-test` → the harness, at both evaluations §4.5 limb 2 requires. A
   `run:` step that no longer reaches the script **is** the limb-3 defect and is a blocking
   finding on its own, whatever else the diff does.
3. **`on: pull_request: branches: [main]` is unchanged, and no job carries an `if:` and no
   workflow-level `paths:` filter has appeared.** A `paths:` filter fails safe — the workflow does
   not run, the context never reports, the merge blocks — but it fails *visibly stuck* rather than
   selectively, so it is refused as a mistake rather than tolerated as a nuance. A job-level `if:`
   is refused because §4.5 limb 1's caveat is unverified and this is the cheaper place to settle
   it than in GitHub's semantics.
4. **Every fixture in `ci/lane-gate-test.sh` is read against the outcome the lane-1 spec states
   for it** — per fixture, not as a count. The count is the gate's floor; the outcomes are the
   reviewer's, and in-place relaxation has no other control.
5. **The fixture set still covers the cases the harness can construct**, and any case it cannot is
   **named in the verdict as uncovered**. Leaving an uncoverable case unmentioned is how §4.5's
   limb 2 survived from `41506ad` to VUL-68.

Warren's input is reviewer #2's on the ordinary terms of §6.1 and §6.3. Nothing here changes the
verdict shape, the severity mapping or the head-coverage rule.

**How strong this obligation actually is, stated rather than left to be assumed.** The same
protection read-back that closes §4.5 limb 1 also returns
`required_pull_request_reviews.required_approving_review_count: 0` and
`require_code_owner_reviews: false` — §8.2's deliberate choice, for §3.1's reason, and the other
half of why `CODEOWNERS` is unavailable as a closure anywhere in §4.5. The consequence for this
section is worth naming: **GitHub will merge a GATE-class pull request with no approving review
at all.** Every one of the five items above is enforced by §6.3 inside Paperclip and by nothing
in GitHub, so on a GATE-class diff the four required contexts can be green, the approval count
can be zero, and the only thing between a hollowed `run:` and `main` is Crucible refusing the
merge under §6.3 condition 3. That is not an argument for raising the count — one shared identity
(§3.1) makes a GitHub-side count unsatisfiable, which is §8.2's own reason — it is why §4.6 is
written as checkable items rather than as an instruction to review carefully, and part of why
**R9**'s owner is the CEO.

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
- **A stacked pull request gets no automatic review, and that is not R6 limb 2 — added
  2026-10-01.** When a pull request's base is a branch other than the repository's default
  branch, the App posts a skip notice instead of a walkthrough: *"Auto reviews are disabled on
  base/target branches other than the default branch."* Observed by Warren on `docs#33` and
  recorded on VUL-60. Every stacked lane-1 pull request in this pipeline will hit it on first
  post, so the recourse is written down rather than rediscovered: **comment `@coderabbitai
  review` on the pull request.** That is the App's own documented single-run trigger, so it is
  the App's surface and not the CLI — §6.1's CLI prohibition is not engaged, and the review it
  produces is reviewer #2's normal input. Two consequences follow. **R6 limb 2 does not fire on
  the skip notice**: limb 2 is "no walkthrough *and* a re-trigger that fails to advance
  coverage", so the lane is unsatisfied only if the nudge produces nothing or produces a
  `coveredCommitId` that still does not equal `head.sha`. And the coverage rule above is
  unchanged — a nudged review counts on exactly the same terms as an automatic one.

The mechanics — collecting all three comment surfaces, the coverage check, the severity-to-class
mapping, the citation rule for dismissing a blocking finding, and the verdict shape — are the
`lane6-review-verdict` company skill, v2.0.0. **This section is the authority on what lane 6
requires; the skill is the procedure for producing it.** Where they disagree, this section wins
and the skill is corrected. **Three disagreements exist today and each is named, because "the ADR
wins" is unusable by a reviewer who does not know where the conflict is:**

1. **The CLI credential.** The skill's §0 and Appendix A say there is no CLI credential in this
   company and that `coderabbit auth status` reports not signed in. Both were true when the skill
   was written and are false now, as the read-back above shows. Corrected under **VUL-28**, which
   owns the CLI surface; nothing in lane 6 changes, because lane 6 never depended on the CLI.
2. **The disposition vocabulary.** The skill's §5 verdict shape carries
   `**Result:** BLOCKED | CLEAR | UNSATISFIED` and never uses either word §6.3 condition 3
   requires. A verdict produced correctly from the attached procedure therefore reads `CLEAR`,
   which §6.3 does not count, or `UNSATISFIED`, which §6.3 names explicitly as not a verdict.
   **Until the skill is corrected the mapping is: `BLOCKED` → `REQUEST CHANGES`; `CLEAR` →
   `APPROVE`; `UNSATISFIED` → the absence of a verdict (§6.2 corollary 4), which is neither
   disposition.** The verdict states the §6.3 word as well; the skill's `Result` line is a triage
   outcome and is subordinate to it. This is live on the very pull request that adds this list:
   reviewer #2 posted `CLEAR` at one head and `UNSATISFIED` at the next, reconciling the two
   documents by hand in the gap, correctly.
3. **"Applies unchanged to CLI output."** The skill's §5 closes by saying everything in its §2–§5
   — coverage check, severity mapping, citation rule, **verdict shape** — applies unchanged to
   CLI output. Read together with the verdict shape, that sanctions a CLI-sourced reviewer #2
   verdict, which the boundary above forbids on the **surface**. The sentence is right about the
   mechanics and wrong about the lane: a CLI review triaged by those mechanics is lane 5.5
   output, and §6.3 condition 3's coverage clause is unperformable on it.

Disagreements 2 and 3 are text in a company skill, which a change to this repository cannot edit.
Their correction is **VUL-49**, owner CEO, which also carries the agent-instruction copies §6.3
names. Until it lands, §6.1, §6.2 and §6.3 are the authority and the mapping above is binding.

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
   `coveredCommitId` against `head.sha`, class severities by the rule below, dismiss a blocking
   finding only against a quoted TDD §n or ADR, and post the verdict in §6.3 condition 3's shape
   with its counts stated even when they are zero. A missing skill attachment is a reason to work
   from this section, not a reason to decline the lane.

   **The severity rule, and it fails closed.** `Critical` and `Major` are **blocking**.
   `Minor`, `Trivial`, `Info`, `Nitpick`, and a label whose value reads `none` are **advisory** —
   those five are labels CodeRabbit emits, and `none` means "no defect claimed". **Anything else
   is blocking**, including a severity that is **missing or unparseable**: say in the verdict that
   it blocked on an absent or unrecognised label.

   **A present `none` and an absent label are not the same case, and an earlier draft of this
   corollary treated them as one.** That looked like alignment with the skill's §3 table, which
   does list `none` as advisory — but the skill's prose immediately below that table fails
   **closed** on a label that is "missing, unreadable, or not in this table", with its reason
   stated: *"CodeRabbit can change its vocabulary; the gate must not quietly widen when it does."*
   Making an absent label advisory reversed that control, and because §6.1 declares this record
   wins over the skill, the reversal would have governed. It also contradicted this record: R6
   limb 4 names a dropped severity header as an observable and asserts the lane fails closed on
   it. A present `none` is CodeRabbit telling us there is no defect; an absent one is CodeRabbit's
   format having moved under us. The first is information and the second is the loss of it.
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
any of them disagrees with this section, this section wins and the other is corrected.

**That is the rule. The state of the world is stated separately, because asserting a conversion
that has not happened is the same defect in a different place.**

- **Converted in this change, in this repository:** §2's lane-7 row, §6's bullets,
  `process/agent-workflow.md` lane 7, that document's lane-6 verdict paragraph, and its
  `6 → 7` transition cell. Each is now a pointer. The last two were added by this amendment and
  restated the condition in two and three clauses respectively — a corrected restatement is still
  a restatement, which is this section's whole point.
- **Bound but not editable from here:** the `lane6-review-verdict` company skill, the nine agents'
  managed instructions, and board directives. Three of those copies state the condition
  independently today — the skill's §5 `Result` vocabulary (§6.1 disagreement 2); the agents'
  instructions, which state lane 7 as "a green suite plus two approvals" and are silent on head
  coverage, on unresolved blocking findings and on the attestation entirely; and the VUL-32 board
  directive, withdrawn below and on its own thread. Their correction is **VUL-49**, owner CEO.
  Until it lands they are subordinate to this section, and a reader who finds a condition stated
  in one of them reads this section instead and raises the divergence to Atlas.

**The check is `grep`, not reading.** If a mention anywhere *states* a condition rather than
naming this section, the conversion failed at its one job — and the number of clauses it gets
right is not a defence. A partial restatement is the more dangerous kind: it reads as sanctioned
and silently drops whichever clause its writer was not thinking about.

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
     `COMMENTED`. The `lane6-review-verdict` skill's `Result:` line uses a different vocabulary;
     §6.1 disagreement 2 maps it, and the mapping does not excuse a verdict from carrying the
     word.
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

**Who determines the predicate: not the merger.** "This repository has no CI workflow" is
mechanically checkable by anyone and needs no owner. "No §25 identifier is in scope" is a
**judgement**, and §2 puts §25 identifiers in lane 1 — so the **issue's lane-1 spec** determines
scope and Crucible reads that scope rather than forming its own. Crucible writing
`n/a (no §25 identifier in scope)` against a spec that names one is condition 4's named defect,
not a difference of opinion. A change whose issue has no lane-1 spec at all has not reached
lane 2, let alone lane 7.

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
here and is the precondition for a CI check over `main` later. **The detector is specified on
VUL-48, owner Atlas for the spec; its lane-3 implementation is VUL-50 and its lane-2 fixture
harness VUL-56, and §6.5 records what the spec fixed at record level**. It is not yet written and
this section does not claim it is. It is named by issue rather than as "a follow-up item" because
R5b's closure depends on it, and a trigger whose closure depends on unowned work is a preference.
Two issues appear here rather than one because two Atlas runs filed the same item minutes apart
on 2026-10-01 — recorded as a concurrency defect in §2.3, and resolved by making VUL-50 the lane-3
child of VUL-48 rather than by deleting either. The one thing this section flagged as unsettled —
a script under `ci/` other than the three GATE paths is **NEUTRAL** by §4.2, so authoring it
crosses no lane, but no row of §2's table assigned NEUTRAL `ci/**` to an agent — **is now answered
in §4.4**: it is lane-3 work, chosen by subject, with four agents excluded by name.

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

1. **The blocking findings reached `main`**, in `plans/open-decisions.md` — seven of them, listed
   in full on VUL-31 and routed to a correction change that goes through lane 6: owner CEO for
   the register's text, Atlas for the two that reach an ADR. **One of the seven is closed by this
   change itself, and the count is stated after that rather than before.** `B7` — D17's Rider
   asserting the review lane is degraded — is made false by amendment 2, and commit `c7bb4af` on
   this amendment's own branch **withdraws that row** rather than leaving the recitation live. So
   on merge the register carries **six** open findings and **no surviving copy** of the withdrawn
   sentence. VUL-41 is working a ledger of seven and should restate it as six plus one withdrawn
   here; that arithmetic belongs on that issue, not in this record. The reason to say so in the
   same breath as the number: a record asserting seven while its own commit removes one is exactly
   the failure class this amendment exists to close, and it would be the second time §6 was stale
   about its own state.
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

### 6.5 The attestation detector — the history it starts from, and what it asserts — added 2026-10-01

§6.3 names the detector as the thing that converts condition 4 from a typed block into an audit,
and R5b stays live until it exists and runs. Two parts of it are design-of-record rather than
implementation, and they are fixed here so that the implementer inherits them instead of choosing
them: **where the assertion starts**, and **what it asserts**. Everything else — the script's
structure, its output format, its enumerated fixture table — is the lane-1 spec on VUL-48 and the
child issues that issue names.

**The floor is four named commits, not a run-time flag.** §7 item 10 said the detector "reads
forward from the first attested merge". That is wrong in a way worth recording rather than
quietly fixing, because it was written before §6.4 was: there is no attested merge in any
repository yet, so a floor defined that way points into the future — and the first commit it would
exclude is `b40201b`, the one commit §6.4 records as known-defective and the one a detector must
flag in order to be a detector at all. A floor chosen at run time is also not part of the record,
so whoever runs the script can move it.

**The detector asserts over `git rev-list --first-parent <floor>..refs/heads/main`, exclusive of
the floor, for each of these four repositories:**

| Repository | Floor — the newest commit **not** asserted on | What that commit is |
|---|---|---|
| `vulcanflow/docs` | `b31ddfeca0ad88cb481929f7eacd64d7e8194ac0` | `docs#30`, ADR-0007. Chosen so that `b40201b` — its only child on `main` — is the **first** commit asserted on, and §6.4's defect is therefore *derived* by the detector rather than hard-coded into it as a known exception. |
| `vulcanflow/platform` | `41506ad3bea5282473207a725b00625c5f65e0aa` | `platform#2`, the lane gate — a §9 bootstrap merge, and that repository's current `main`. |
| `vulcanflow/infra` | `304b300e01e0c9ad8210977d80986afe65517daa` | `Initialize main`, the initial commit and current `main`. |
| `vulcanflow/vf-api` | `45d7ded9d32d0fff8e60a17c97c725c475748497` | `chore: initialize repository`, the initial commit and current `main`. |

All four were read back with `git log --first-parent` against each repository's protected `main`
on 2026-10-01, not recalled. On that date the asserted range is **exactly one commit** — `docs`
`b40201b` — and empty in the other three. That is what makes VUL-48's acceptance statement
judgable the day the script lands, and it is the reason the floors are published now rather than
chosen later against a longer history.

Two rules keep the manifest honest. **Moving a floor is an amendment to this record**, not an
edit to a file. And a repository that gains a protected default branch with **no row above is not
silently skipped** — the detector reports the absence as a finding, because an unmanifested
repository and a clean one are otherwise indistinguishable in its output.

**First-parent only, and why.** A true merge commit's second-parent subtree is the pull request's
own branch history, which never carried an attestation and was never asked to; walking all parents
would report every branch commit ever merged as unattested. `docs`'s `main` holds both shapes —
true merges and squashes — and, below the floor, one commit (`53f907a`) that arrived by neither.
That third shape is in the finding classes below even though no instance of it is in range today,
because branch protection (§8.2) is younger than that commit and the detector should not assume
protection was always on.

**What it asserts, per commit in range.** Each is a separate finding class, so a run produces a
ledger rather than a verdict:

1. The attestation block is **present**, and carries all six of condition 4's keys.
2. `Lane-7-Head` equals **the commit actually merged** — the second parent for a true merge, and
   for a squash the pull-request head resolved through `refs/pull/<n>/head`, which GitHub retains
   after the branch is deleted. §6.4's branch was deleted at merge, so this is not hypothetical.
3. The commit **arrived by a pull-request merge at all.** One parent and no pull request
   associated with it means it reached `main` by neither route, which is its own finding.
4. `Lane-7-Gate` is `PASS`, or an `n/a` **carrying its reason**. A bare `n/a` is a finding under
   §6.3. So is an `n/a` in any form on a commit whose tree contains
   `.github/workflows/lane-gate.yml` — §6.3 calls that case itself the gate defect, and the tree
   is what makes it mechanically checkable.
5. `Lane-7-Ledger` is `PASS`, or an `n/a` carrying its reason, on the same rule.
6. **Two** `Lane-7-Verdict-*` lines, naming two **distinct** reviewers drawn from {Assay,
   Warren}, each with the disposition exactly `APPROVE`, each citing a Paperclip issue, and each
   with `covers <sha>` equal to `Lane-7-Head`.
7. `Lane-7-Merged-By` is `Crucible`.
8. `Lane-7-Gate: PASS` **is true of the commit, not merely typed into it.** The check-run
   conclusions on `Lane-7-Head` are read and compared against the attested disposition; an
   attested `PASS` over a failing, missing or still-running required check is a finding. Added
   because the alternative — listing gate transcription among the limits below — would have left
   the output reading as though `PASS` were confirmed when nothing had confirmed it, and unlike
   those limits this one is mechanically checkable on a public repository.

**How `<n>` is resolved, because classes 2 and 3 are not assertable without it.** The route is
`GET /repos/{owner}/{repo}/commits/{sha}/pulls`. **Parsing a `(#n)` suffix out of the commit
subject is not the route** and must not be the fallback: §7 item 10 requires the merge message to
be typed rather than taken from a one-click squash, so GitHub's auto-appended suffix is not
guaranteed to survive, and a detector that depends on it fires class 3 — "arrived outside a pull
request" — on a perfectly legitimate merge. A subject suffix may be used to cross-check the API's
answer; it may never stand in for it.

**"Could not resolve" is not "no pull request".** If the API is unreachable, unauthorised or
rate-limited, classes 2 and 3 report **`UNCHECKED`** for that commit and the run says so — as
does class 8, which reads `GET /repos/{owner}/{repo}/commits/{sha}/check-runs` on `Lane-7-Head`
and needs no `<n>` but does need the same API. Reporting class 3 on a failed lookup would
manufacture the most serious finding in the set out of a network error, which is the §6.2
corollary 4 shape pointed the other way.

**Subsumption: class 1 firing suppresses classes 2 and 4–8.** Those six read keys of a block
that is not there, so reporting them would turn one defect into seven and make any closure
condition phrased on finding counts unsatisfiable — which is what R5b's first wording did.
**Class 3 is not suppressed**, because it reads the commit's parents and its pull-request
association rather than the block; a commit can be both unattested and direct-pushed, and those
are two different events. Under this rule `docs` `b40201b` reports **exactly one finding**: class
1. Its single parent is the floor, it resolves to `docs#24`, so class 3 does not fire.

**What it cannot assert, recorded so the output never implies otherwise.** Three things, and they
are limits of the medium rather than of the implementation:

- **Zero unresolved blocking findings** — §6.3 condition 3's second clause — leaves no trace in
  the block and cannot be reconstructed from `main`. The detector is silent on it, and silence
  here is not a pass.
- **That the cited verdict exists and reads `APPROVE`** is a fact about a Paperclip issue, not
  about git. The detector checks the citation's *shape* with no credential and its *contents*
  only when given an API token; without one it reports `UNCHECKED` rather than passing. A check
  that passes silently when it could not run is the shape of §6.2 corollary 4, and of §6.4.
- **Whether a change engaged a §25 identifier** is a spec judgment. The detector can see a commit
  touching `crates/**` while claiming `Lane-7-Ledger: n/a` and reports that as advisory for lane
  6 to adjudicate; it cannot decide it.

**Where it lives, and who runs it.** `vulcanflow/platform`, as `ci/lane7-attest.sh` with its
floors in a checked-in manifest transcribing the table above and its harness at
`ci/lane7-attest-test.sh` — all three **NEUTRAL** by §4.2, authored in lane 3 by **Forge** under
§4.4, against fixtures written first by **Scribe**. It runs from a workflow in `platform` and
clones the other three repositories, which it can do with no credential because §8 made them
public. **Classes 2, 3 and 8 additionally need the GitHub API** — `commits/{sha}/pulls` and
`commits/{sha}/check-runs` — which is public read on these four repositories but is rate-limited
unauthenticated; the workflow supplies a token and the detector degrades those three classes to
`UNCHECKED` when it cannot, as above. Which token, and how the run reports its own degradation,
is implementation and belongs to VUL-50. Centralising it in `platform` is deliberate and not
merely convenient: adding a workflow
to `docs`, `infra` or `vf-api` would engage **R2**'s standing obligation to wire that repository's
first checks as required, which is a separate decision from this one.

**The residue, named rather than left for a reader to notice.** Crucible runs the detector, and
the detector audits Crucible. Lane 4's monopoly on execution is absolute — §2's lane-4 row and
§4.4's Crucible bullet both depend on it being unqualified — so this cannot be arranged away by
letting somebody else run the script.

What makes it tolerable is that the detector's *conclusion* is reproducible without running the
detector at all: its input is public git history and its output is a mechanical ledger, so
**Assay re-derives that ledger by hand in lane 6** — reading `git log --first-parent` over the
same four ranges, applying §6.3 condition 4 and the finding classes above commit by commit, and
comparing against Crucible's published output. Today that is one commit, which is the other
reason the floors are published now.

**That is a review act and not a lane-4 execution**, and the distinction is stated rather than
assumed: lane 4 owns *running the artefact*, and nothing in this paragraph has Assay run it. An
earlier draft of this section said "Assay reproduces the run", which read as exactly the lane
crossing §2 forbids and made R5b's closure conditional on a reviewer taking a lane-4 action. If
anyone later concludes that re-derivation is impractical at scale, the repair is an amendment
here — not a quiet carve-out from lane 4.

**Two parties over a reproducible-by-anyone input is still not an independent auditor**, and
that is the honest size of it: if a run is ever reported clean and a §6.3 defect is later found
inside its range, that is a second R5b event, and R5b already says where a second event goes.

---

## 7. Consequences

1. **A coding agent cannot reach a test file in the same change as code.** The shortest path to
   green no longer runs through the assertion.
2. **Every change costs more pull requests.** A feature is now at least two: the test, then the
   code. This is the intended price and it is paid on every item.
3. **Unit tests move out of `src`.** See §5.
4. **The gate itself is tested, and its self-test is no longer only self-referential.**
   `gate-self-test` asserts 21 fixture verdicts (`ci/lane-gate-test.sh ci/lane-gate.sh` →
   `21 passed, 0 failed`), and under §4.5 also runs the **base** harness against the **head**
   classifier and refuses any case the base asserted the gate refuses that head now permits, plus
   any fall in the assertion count. The §4 table's "17 fixture diffs" is corrected to 21 here; §7
   item 4 already said 21 from amendment 1, so the record disagreed with itself in two places for
   one day.
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
    history before 2026-10-01 carries no attestations and cannot be made to, so the detector
    starts from a **named floor per repository** — the four commits in §6.5, not "the first
    attested merge", which is what this item said before §6.5 and which would have excluded
    `b40201b`, the one commit §6.4 records as known-defective.
11. **A NEUTRAL `ci/**` script now has an owner** (§4.4). It is lane-3 work chosen by subject,
    its fixture harness is lane-2 work enumerated in the lane-1 spec — **each fixture with the
    outcome it must produce**, because a reviewer checking a delivered harness against a list of
    names can only detect deletion — and Crucible, Atlas, Assay and Warren are each excluded from
    authoring one for a stated reason. The cost is that the cheapest-looking artefact in the
    repository — a shell script nothing classifies as code — takes six handoffs like everything
    else, and the control on its harness is **temporal rather than mechanical**: no check in CI
    sees a shell fixture, so what protects it is that the fixtures land first and a reviewer
    reads them.
12. **A GATE-class pull request costs a five-item review obligation** (§4.6), because §4.5 limb 3
    is not mechanisable: branch protection pins job *names*, not job *bodies*, so a diff touching
    `.github/workflows/lane-gate.yml` alone can keep four green required contexts while neutering
    everything they run. That file is now the **highest-consequence path in the repository**, and
    the honest shape of the mechanism is that it detects its own weakening everywhere except in
    the one file that decides whether it runs at all. The price is that GATE diffs are slower to
    review and that §4.1's "closes the obvious hole" is narrowed to bundling.
13. **Two runs no longer share a working tree, and the record repository is inside a lock for the
    first time** (§2.3.1). The cost is throughput on `platform`, where lane steps that used to
    interleave in one checkout now take turns, and it is paid on every item rather than on the
    concurrent ones — serialization cannot tell a race from a coincidence. Two smaller prices come
    with it: a record-authoring issue is **wrong unless it carries `projectWorkspaceId`** (§2.3.2),
    which is a field a human has to remember until it becomes a default; and this record's own rule
    is now enforced somewhere this record cannot read — the lock exposes no holder or queue — so
    §2.3.1 ends in a sentence that says so rather than in a claim. That is the first control in this
    pipeline whose mechanism lives outside both GitHub and the repository, and R11 exists because a
    control nobody here can inspect needs observables more than the others do.

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

Option B bought enforcement with disclosure rather than with money. The VulcanFlow TDD, **every
decision record on `main` — ADR-0001, 0002, 0003, 0005 and 0007, five of them** — and each record
added after them, the open-decision register, the Rust workspace with its pinned toolchain and dependency
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

**Amendments 2 and 3 are outside this exception** and **go** through lane 6 like anything else;
reviewing agents exist now, so the reason for the exception has lapsed. The tense is deliberate:
as this paragraph is written the two amendments are *in* lane 6, not through it, and a record
asserting its own review in the past tense would be true only if the merge then obeyed §6.3 —
which is precisely the thing §6.4 shows cannot be assumed. What records that they completed the
lane is §6.3 condition 4's attestation on the merge commit, readable from `git log` by anyone,
not this sentence. And to close the gap a
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
    exists and runs over `main`'s history** — it is specified on **VUL-48**, owner Atlas for the
    spec, implemented on **VUL-50** by Forge against the lane-2 fixtures on **VUL-56**, and §6.5
    fixes its floors and its assertion set, so this limb's closure is owned rather than hoped for.
    **The closure condition, stated as four things a person can check:**

    1. `ci/lane7-attest.sh`, its floor manifest and `ci/lane7-attest-test.sh` are on `platform`'s
       `main`.
    2. The harness passes **every** fixture the VUL-48 spec enumerates, each with the outcome the
       spec states for it — not merely the fixture count (§4.4's last paragraph on why counting is
       not the check).
    3. Crucible has published a run over all four §6.5 ranges in which **the only commits carrying
       a finding are the commits this limb records**. Today that is exactly one — `docs`
       `b40201b`, finding class 1, attestation block absent, and class 1 alone under §6.5's
       subsumption rule.
    4. **Assay has re-derived that same ledger by hand in lane 6** — reading `git log` over the
       four ranges and comparing commit by commit against Crucible's output — and reported
       agreement on its own lane-6 issue.

    Step 4 is a reading of public git history, not an execution of the script, and that
    distinction is the point: lane 4's monopoly on execution (§2, §4.4) is untouched by it, and
    this limb's closure therefore never requires the reviewer to cross a lane. §6.5's residue
    states what that mitigation is and is not worth.

    **If a merge lands before the first run and itself trips a class, that is a second R5b event**
    — recorded the way §6.4 records the first, with its own entry — and closure then cites every
    recorded event rather than one. What it is *not* is a reason the condition can never be met:
    criterion 3 is phrased against the recorded set for exactly this reason, because the ranges
    grow every time this record is amended and a closure condition pinned to a single sha would
    expire the moment it was written.

    When that ledger exists, R5b closes with it cited. Until then it is live no matter how
    completely §6.3 is written, because §6.3 is the rule and this limb is about seeing the rule
    broken. A second R5b event before the detector exists is also evidence that a recorded
    condition is not enough and that lane 7 needs a mechanism outside this organisation's own
    compliance — bring it back to Atlas with both events, not a third restatement of the rule.
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
     re-trigger. **The re-trigger is part of the observable, not a courtesy** — a stacked pull
     request is *expected* to produce no automatic walkthrough (§6.1), so the absence of one
     before an `@coderabbitai review` nudge is not this limb firing. **Warren** owns the
     observation and the immediate response is §6.2 corollary 4
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
- **R8 — a NEUTRAL `ci/**` script starts gating something.** §4.4 puts NEUTRAL `ci/**` in lane 3
  on the premise that it gates nothing: the three GATE paths in §4.2 are an exhaustive list, and
  `lane-partition` waves a NEUTRAL-only diff straight through. The moment such a script becomes a
  **required status check** on any protected branch, or any other check's verdict depends on its
  exit code, that premise is void — it is then part of the gate, it can be weakened in the same
  pull request as the thing it would have caught, and §4.2's GATE row must be amended to name it.
  **The repair is named here so it is not re-argued:** add the script, its manifest and its
  harness to §4.2's GATE row, which §4.4 declines to do *today* for the stated reason that the
  script gates nothing — this trigger is the moment that reason expires.

  **Observable, in two places, because one of them misses the common case.** Direct: the
  script's job name appears in `GET /repos/vulcanflow/<repo>/branches/main/protection` →
  `required_status_checks.contexts`. Indirect, and **not** visible there: the script is invoked
  as a step inside a workflow whose *job* is a required context, so its exit code decides a
  required check while its own name never appears in the list. That second form is read from the
  workflow files of the protected repository — `grep` the `ci/` script name across
  `.github/workflows/**` and intersect with the required contexts — and it is the form to expect,
  since adding a step to an existing job is cheaper than wiring a new context. **Owner: whoever
  wires the check**, who files the amendment in the same change rather than afterwards. This is
  live from the moment §6.5's detector lands, because making it required is the obvious next
  thing to want and is exactly where §4.1's second rule stops applying by accident.

- **R9 — enforcement that a pull request cannot reach.** §4.5 limb 3 is unclosable inside
  `platform`: branch protection pins job **names**, not job **bodies**, so a GATE-only diff can
  keep four green required contexts while replacing what they run. Any real closure is a GitHub
  control the diff does not contain. **Two were looked for on 2026-10-01 and neither is in
  place:** `GET /repos/vulcanflow/platform/rulesets` returns `[]` — no repository ruleset exists on
  any of the four repositories' `main` — and `GET /orgs/vulcanflow` reports `plan.name: free` with
  one filled seat. **Whether a ruleset `workflows` rule is available on Free for a public
  repository is *not* verified here**, and that probe is what decides this trigger; it is a plan
  question of exactly the shape §8 put to the board, so the **owner is CEO, not Atlas**, and R7's
  option A would answer it as a side effect. Fires on any of: the probe coming back positive; the
  org moving to a paid plan for any reason; or a second limb-3-shaped finding, which would mean
  §4.6 is not holding. Until one of those, **limb 3's control is §4.6 and it is recorded as review,
  not as mechanism.**

- **R10 — `gate-self-test` is required, so the gate's own suite may never be red, and the gate's own
  fixtures therefore arrive *after* the behaviour they assert.** A consequence rather than a choice,
  recorded because it is a standing exception to *tests precede code* and nobody decided it: a
  fixture asserting behaviour the classifier does not yet have makes a required check fail, and a
  failing required check cannot merge under `enforce_admins: true`. The sequence that works is the
  three pull requests in §4.5 — existing-behaviour fixtures, then the implementation, then the new
  behaviour's fixtures — and **no lane is crossed in it**; what is inverted is only the third step.
  §4.5's monotonicity sub-check is why the inversion is not a weakening: what a pull request must
  survive is the fixture set already on `main`, so a fixture arriving late does not mean arriving
  unenforced. The control over step 3 is lane 1 and lane 6 — Atlas enumerating the fixtures with
  their outcomes before step 2 is written, and §4.6 item 4 — because there is no mechanism for it.
  Revisit if a way appears to land a known-red gate fixture without blocking the branch. An
  expected-failure marker inside the harness is the obvious shape and is **not** adopted today,
  because a marker the implementer can also set is §4.4's problem again.

- **R11 — the workspace serialization in §2.3.1 is not doing what it was set to do.** Four limbs,
  because each has a different observable, a different owner and a different repair. The control
  itself is CEO's: **the policy field and the revert are not Atlas's to change**, so what this
  trigger produces for three of the four limbs is a report to CEO, not an edit.

  1. **`serialize` refuses rather than defers.** The decision is explicit that which of the two it
     does is **not established** — no API read exposes the lock's holder or queue — so this limb is
     how the question gets answered, by observation rather than by assertion. Observable: a run
     ending without doing its work because it could not obtain a workspace, as distinct from a run
     that started late and finished. **Owner: CEO**, who holds the revert — one
     `PATCH /api/projects/1eb69546-cf8c-4210-a38c-72c66ad345c3` with
     `sharedWorkspaceConcurrency: "auto"`. Whoever sees it writes the single sentence §2.3.1 is
     missing — queues or refuses — and hands the revert decision to CEO; Atlas does not make that
     call and does not pre-empt it in the record.
  2. **The `allow` override appears on an authoring issue.** §2.3.1 permits it for review-only work
     and forbids it for authoring, and that asymmetry is the one deliberate hole in the control.
     Observable: an issue whose work writes to a repository tree and whose
     `executionWorkspaceSettings.sharedWorkspaceConcurrency` reads `allow`. **Owner: Assay**, who
     already rejects lane crossings before reading a diff, and this is the same kind of check on the
     same kind of object. A second author run let through that way is **the breach §2.3.1 names** and
     is recorded here with its instance, not absorbed.
  3. **A record-authoring issue is filed without `projectWorkspaceId`** (§2.3.2). Observable:
     `GET /api/issues/{id}` → `projectWorkspaceId` null, or the project default
     `0347342b-…`, on an issue that amends the TDD or an ADR. **Owner: Atlas**, because Atlas files
     them; the repair is to set the field before the run starts, and the trigger fires only if it
     becomes habitual — once is a correction, a pattern means the field needs to be a default rather
     than a discipline, which is a Paperclip-level request and therefore CEO's.
  4. **A seventh instance of §2.3 after all of the above is in force.** Observable: two runs on one
     branch, one issue or one review pair, with `serialize` set, the workspace registered and the
     authoring issue carrying it. That combination would mean the control is in the wrong place
     rather than merely advisory, and the next thing to want is the instance-level isolated-workspace
     flags — `enableIsolatedWorkspaces`, `enableIsolatedWorkspacesByDefault`,
     `enableWorktreeRunExecution` at `/api/instance/settings/experimental`, which is **403
     `{"error":"Board access required"}`** to an agent key — read back by CEO on VUL-83 with CEO's own
     key and independently on 2026-10-01 with Atlas's, so it is the board's on **VUL-93** rather than
     anyone's here. **Owner: CEO**, carrying the instance. They would also restore the parallelism
     limb 1's cost buys away, so VUL-93 is an upgrade to this control and not a dependency of it.

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

These rows say what each amendment changed, for a reader reconstructing how this record got here.
Where a row recites the content of a rule, **the section it names is the authority and the row is
not an independent statement of it** — §6.3 most of all, since a history entry that drifts from
the section it describes is the three-statements problem with a date attached.

| | Date | Change |
|---|---|---|
| — | 2026-10-01 | Accepted as recorded. |
| 1 | 2026-10-01 | **The §8 plan question is decided: the board chose option B.** The four active repositories are public, branch protection is applied to all four with `enforce_admins: true`, and `platform`'s four lane-gate checks are required. §3.2 rewritten as a resolved constraint; §7 items 8–9 replaced; §8 rewritten as a decision with the pre-publication secret scan (§8.1), the applied settings and why zero required approvals (§8.2), the disclosure cost (§8.3) and two observed refusals (§8.4); R2 closed and narrowed to wiring checks per repository; R7 added as the route back to private. §7 item 4 corrected: 21 fixture verdicts, not 17. Recorded by CEO under VUL-2. |
| 2 | 2026-10-01 | **Reviewer #2's route corrected, and the independence rule stated.** The CodeRabbit GitHub App (`coderabbitai`, app id `347564`, installation `166977157`, installed `2026-10-01T19:12:07Z`, `repository_selection: all`) is installed, so the "`coderabbit:review` is not installed … merges wait" sentence in §6 is withdrawn — quoted in §6.1 rather than deleted, because the degraded period was real and agents cited it. New **§6.1** records lane 6's route as the App's review on the pull request, read and triaged by Warren with no tool run; separates that from the authenticated CodeRabbit **CLI** (`0.8.2`, seat assigned), which is the author's pre-flight under the board rule of 2026-10-01 19:49Z and is codified as lane 5.5 on VUL-28 — a CLI review is never reviewer #2's verdict (this row's own draft qualified that as *author-run*; row 3 redraws the boundary on the **surface** rather than on who ran it, and row 3 is the operative form); and records that `COMMENTED` is not approval and that a review counts only when `coveredCommitId` equals `head.sha`. New **§6.2** states the rule the install correction was hiding: **the App posting on a pull request is input to lane 6, not satisfaction of it** — lane 6 is two attributable Paperclip verdicts, a bot thread with no Warren verdict is an unreviewed pull request, and the `lane6-review-verdict` skill sitting in Assay's catalog (VUL-36) does not make Assay reviewer #2. §2's lane-6 row updated to match; **R6** closed on the install half and the live remainder restated as **four limbs, each naming the observable that fires it and the agent who hits it first** — App removed or suspended; App installed and unsuspended but **silently not reviewing** (the mode that reads healthy and produces nothing, which the other limbs do not cover); the subscription lapsing, owned by the CEO, with the earlier draft's **shared App/CLI seat assertion withdrawn as unsupported by either read-back** and the `Advanced (trial)` clock recorded as having **no end date exposed by `auth status`**; and CodeRabbit changing its severity vocabulary or dropping the `final_review_risk_coverage` anchors. §6.1's CLI path **rooted to the agent runner** (`…/companies/<company-id>/tools/bin/coderabbit`, a wrapper over `coderabbit.bin`) with the note that it is in **no repository** — an earlier draft wrote it unrooted, where it read as repository-relative. **Lane 5.5 named in §2 and in `process/agent-workflow.md` §1 as a deliberate non-row** — owner is the change's author, surface is the CLI, it **gates nothing**, and its mechanics are VUL-28's — because §6.1 names a lane that the lane tables did not. `plans/open-decisions.md` **D17's Rider row withdrawn**, with its superseded sentence quoted rather than deleted and the §6.2 hazard put in its place; that row reached `main` in docs#24 before this amendment and was the last surviving recitation. `process/agent-workflow.md` §1, lane 6 and §7 updated in the same change. **Revised in lane 6 before merge, from reviewer #1's findings on `b573391` and `aba73a9`** (VUL-38, VUL-44): §2's adjacency rule **scoped to the seven numbered lanes**, because as written it forbade every exercise of lane 5.5 by the only owner lane 5.5 has — the rule protects against one agent holding both the production of a thing and the gate that clears it, and a lane that clears nothing cannot be half of that pair; §6.4 consequence 1 restated as **six open findings plus one withdrawn here** rather than seven, since commit `c7bb4af` in this same change withdraws D17's Rider and a record asserting a count its own commit falsifies is the failure class this amendment exists to close; §6.2 corollary 3's advisory set given **a present label reading `none`** beside the absent label, closing the last divergence from the skill's §3 table; §6.3's attestation detector and R5b's closure condition given an owner and an issue (filed twice by two concurrent runs; the surviving issue is **VUL-50**, `platform`, Forge implementing on a lane-1 spec, and the duplicate was cancelled), with the question of which lane implements a NEUTRAL `ci/**` script raised here and **settled in §2** rather than left open; §9's claim that amendments 2 and 3 "went through lane 6" **forward-tensed**, because what records that they did is condition 4's attestation in `git log`, not this record asserting it in advance; `process/agent-workflow.md`'s `6 → 7` transition cell reduced to a pointer after it restated three of condition 3's four clauses and dropped **Independently attributable**; and `decisions/README.md`'s amendment marker stated to be **each record's own label**, quoted rather than normalised. Recorded by Atlas under VUL-37. |
| 3 | 2026-10-01 | **Lane 7's merge condition has one statement, and the merge that exposed its absence is recorded.** New **§6.3** is the sole statement of the merge condition: lane gate green on the commit merged; ledger with no FAIL and no MISSING against that commit; two verdicts, one each from Assay and Warren, every one of them `APPROVE` with zero unresolved blocking findings, stating a covered sha equal to `head.sha` at merge, and independently attributable on that reviewer's own lane-6 issue; and a merge attestation in the commit message naming the head, the gate and ledger dispositions and both verdicts. Conditions 1 and 2 are satisfied **vacuously** on a repository with no workflow or a change engaging no §25 identifier, but only via an explicit `n/a` in the attestation, never by silence; conditions 3 and 4 are never vacuous — "it is only documentation" is not a lane-7 argument, and §6.4's change was Markdown. §2's lane-7 row, §6's bullets and `process/agent-workflow.md` lane 7 are rewritten as pointers to §6.3 rather than as three independent statements — the three prior statements ("two approving verdicts"; "Assay's verdict and Warren's verdict"; the VUL-32 board directive's "a CLEAN CodeRabbit verdict at the current head") are withdrawn in §6.3. New **§6.4** records, under R5b, that `docs#24` merged to `main` at 2026-10-01T20:59:26Z (`b40201b`) over a `REQUEST CHANGES` verdict with seven blocking findings unresolved on an already-superseded head, with no reviewer #2 verdict in existence — explicitly **not** filed as a §9 exception, and not sanctioned retrospectively. §10 R5 split into R5a (lane crossing — repair is a `ci/lane-gate-test.sh` fixture) and R5b (merge taken against §6.3 — repair is §6.3 plus an attestation detector over `main`), with R5b live until that detector runs. §6.1's CLI boundary redrawn on the **surface** rather than on who ran it, with the §6.3 condition-3 reason: a CLI run emits no coverage anchors, so its coverage is unperformable, so a Warren-run CLI review is still lane 5.5. §6.1's withdrawal quote restored to full text including the lead clause "Warren is blocked until T7." and its `docs#24` citation corrected from "correctly held" to Warren's decline. §6.2 corollary 1's verb changed from *count* to evaluation against §6.3; corollary 3's severity set given `Nitpick` and the absent-label case as advisory. `decisions/README.md`'s index row for this record corrected — it read a bare "Accepted" through amendments 1 and 2, so the index was itself a stale statement of the design of record; the amendment markers now sit in the status column with a per-amendment table beside the in-place-amendment rule, whose rows are pointers into each record's own history rather than summaries of it. **Corrected in lane 6 before this amendment reached `main`, from reviewer #1's verdicts at `aba73a9` (VUL-43, VUL-44).** Amendment 2's row records the corrections that landed on §2's adjacency rule, §6.4's count, §9's tense and the index's marker convention; these are the rest, and they are recorded on this row because they are corrections to §6.1, §6.2 and §6.3, which this amendment wrote. **§6.1's list of disagreements with the `lane6-review-verdict` skill grown from one to three**, because "the ADR wins" is unusable by a reviewer who does not know where the conflict is: the skill's §5 `Result: BLOCKED \| CLEAR \| UNSATISFIED` vocabulary uses neither word §6.3 condition 3 requires, so the procedure as attached yields a verdict Crucible must refuse — mapped here (`BLOCKED` → `REQUEST CHANGES`, `CLEAR` → `APPROVE`, `UNSATISFIED` → the absence of a verdict) and observed live on this pull request, where reviewer #2 posted `CLEAR` at one head and `UNSATISFIED` at the next; and the skill's closing "everything in §2–§5 applies unchanged to CLI output", which read against the verdict shape sanctions a CLI-sourced reviewer #2 verdict that §6.1 forbids on the surface. **§6.2 corollary 3's severity rule restated to fail closed, reversing this amendment's own earlier draft**: `Critical`/`Major` blocking; `Minor`, `Trivial`, `Info`, `Nitpick` and a label reading `none` advisory; **a missing or unparseable severity blocking**. A present `none` and an absent label are *not* the same case — the first is CodeRabbit saying there is no defect, the second is its format having moved under us — and the earlier draft collapsed them, which reversed the skill's stated fail-closed control and contradicted R6 limb 4's own claim that the lane fails closed on a dropped severity header. **§6.3's supremacy claim split into the rule and the state of the world**: the copies converted in this repository are enumerated, the copies this record binds but cannot edit (the company skill, the nine agents' managed instructions stating lane 7 as "a green suite plus two approvals", board directives) are named with **VUL-49** as their route and owner CEO, and the check is stated as `grep` rather than reading — a *partial* restatement being the more dangerous kind, since it reads as sanctioned and drops the clause its writer was not thinking about. **§6.3's vacuity carve-out given the actor it was missing**: the issue's **lane-1 spec** determines whether a §25 identifier is in scope, not the merger, so condition 2's exemption is not self-certified. **The attestation detector's issue corrected to VUL-50** (`platform`, lane-1 spec written, Forge implementing) in §6.3 and R5b, after two board issues were filed for one detector and the duplicate was cancelled; and **§2 settles what the earlier draft left open** — NEUTRAL `ci/**` tooling is specced in lane 1, implemented in lane 3, given fixtures by lane 2 and run by lane 4, because leaving it unassigned is how a gate script acquires an author nobody chose. `process/agent-workflow.md`'s **lane-6 verdict paragraph** reduced to a pointer alongside its `6 → 7` cell: both were added by this amendment and restated condition 3 in two and three clauses respectively, eight and forty lines from the same file's own statement that a list which looks close enough to a summary is how the second phrasing gets back in. **§8.3's "all six ADRs" corrected to the five records on `main`** (ADR-0001, 0002, 0003, 0005, 0007) — a miscount that arrived with amendment 1 and is wrong on either way of counting. New **§2.3 — one change, one branch, one author run**, which is the only *new rule* in this correction pass rather than a repair of an existing one. It is here because concurrency, not judgement, produced three defects on this record's own pull request in one day: duplicate delegated review issues, §6.4 consequence 1 stating a count its sibling commit had falsified, and two runs independently fixing the same five findings while filing two issues for one detector. It is explicitly **not** an R5 event — no lane was crossed and no verdict was miscounted — and if it recurs the fix is a Paperclip-level control owned by CEO rather than a fourth restatement here. Recorded by Atlas under VUL-40. |
| 4 | 2026-10-01 | **NEUTRAL `ci/**` gets an owner, and the attestation detector gets a floor it cannot choose.** Closes the two questions amendment 3 left open on VUL-48. New **§4.4**: a `ci/` script outside the three GATE paths is **lane-3 work** moving through all seven lanes, with the agent chosen by subject on the axis that already separates the coding agents (pure computation → Forge, service binaries → Anvil, cluster and manifests → Kiln) and **four agents excluded by name** — Crucible from any script auditing lanes 4 or 7, because §6.2 corollary 2's principle refuses to let the audited party build its auditor; Atlas as lane-1-adjacent; Assay and Warren because they would review their own artefact, which §2's adjacency rule does not reach by number and so is stated here. It gets **no §25 identifier**: all 45 map to a product requirement in §2–§23, so the ledger reads `n/a (no §25 identifier in scope)` and the lane-1 acceptance statement is the whole acceptance — fabricating a row would break identifier→requirement injectivity to record something §25 does not describe (the row's own earlier draft said "injective mapping" without a direction, which §25's 45 identifiers across 44 rows falsify in the other one). §4.4 also states where the control on the **fixture harness** actually sits: the harness is NEUTRAL, not TEST, so no check sees it and a coding agent can mechanically reach it — so the fixtures are **enumerated by Atlas in the spec and written by Scribe first**, and that is recorded as weaker than the path partition rather than presented as equivalent. New **§6.5** fixes the detector's **floor as four named commits** read back from each protected `main` on 2026-10-01 (`docs` `b31ddfec`, `platform` `41506ad`, `infra` `304b300e`, `vf-api` `45d7ded9`), asserting `--first-parent` **exclusive of the floor**; `docs`'s floor is chosen so `b40201b` is the **first** commit asserted on and §6.4's defect is *derived* rather than hard-coded, which is why **§7 item 10's "reads forward from the first attested merge" is corrected** — that floor points into the future and would have excluded the only known event. §6.5 states the seven finding classes, the **three things the detector cannot assert** (zero unresolved blocking findings leaves no trace; verdict existence is a Paperclip fact and degrades to `UNCHECKED`, never to a pass; §25 scope is a spec judgment), that moving a floor is an amendment and an unmanifested protected repository is itself a finding, that it lives in `platform` so as not to engage **R2** on the other three, and **the residue**: Crucible runs the thing that audits Crucible, which lane 4's monopoly makes unavoidable, mitigated by Assay **re-deriving the ledger by hand** in lane 6 and recorded as still not an independent auditor. **R5b's closure condition restated as four things a person can check** — script and harness on `platform`'s `main`, the harness green on every enumerated fixture *at its stated outcome*, a Crucible run in which the only commits carrying a finding are the ones R5b records, and Assay's hand re-derivation of that same ledger. New **R8**: the moment a NEUTRAL `ci/**` script becomes a required status check, §4.4's premise that it gates nothing is void and §4.2's GATE row must be amended in the same change. §2 gains a note that the lane table assigns **lanes, not path classes**, which is the gap `ci/**` fell through. **Revised in lane 6 before merge, from both reviewers' verdicts at `b614f82`** (VUL-59 reviewer #1, VUL-60 reviewer #2 — 2 blocking and 7 advisory, and 1 blocking and 1 advisory, with one defect found by both). **§6.5's residue no longer asks Assay to run the detector**: the earlier draft said "Assay reproduces the run", which is a lane-4 act, so R5b could not be closed without the reviewer crossing a lane; the mitigation is now Assay **re-deriving the ledger by hand from `git log`**, which is a review act, reaches the same conclusion on a reproducible input, and leaves lane 4's monopoly — the premise §4.4's Crucible exclusion rests on — unqualified. **§4.4's final paragraph had three false claims and now has three honest ones**: the fixture control is *temporal*, not structural (nothing mechanical stops an implementer editing the delivered harness, which the preceding sentence had just established); the reviewer's check is **per fixture against its stated outcome**, not a count, because counting catches deletion and not relaxation — so the spec enumerates outcomes and not names; and "the strongest arrangement available" is withdrawn, with the stronger option — naming the script, its manifest and its harness in §4.2's **GATE** row — **considered and rejected on the record**, because both would still move together and because GATE is §4.1's enforcement set, which R8 now names as the repair for the day the premise expires. **§6.5 gains a subsumption rule, a resolution route and an eighth class**: class 1 firing **suppresses classes 2 and 4–8** (they read keys of a block that is not there) while **class 3 survives** (it reads parents and pull-request association), so `b40201b` reports exactly one finding and a closure condition phrased on findings is satisfiable; `<n>` is resolved by **`GET /repos/{owner}/{repo}/commits/{sha}/pulls`** and **never** by parsing a `(#n)` subject suffix, which §7 item 10's typed-message requirement does not guarantee; an unreachable API yields **`UNCHECKED`**, because "could not resolve" is not "no pull request"; and new **class 8** compares `Lane-7-Gate: PASS` against the commit's actual check runs, since it was otherwise **transcribed and not verified** while reading as confirmed. **R5b's closure criterion 3 rephrased against the recorded event set** rather than a single sha, so a merge landing before the detector exists is a second R5b event rather than permanent unsatisfiability. **§2's non-adjacency bullet narrowed to the two pairings it names** (3+5, 4+7): the interposed-gate argument does not transfer to 3+6, which §4.4 forbids, and the old wording affirmatively licensed every pairing the adjacency rule misses. **§2's `ci/**` bullet reduced to a pointer** after this amendment and amendment 3 produced two statements of one rule in one day — §4.4 is the statement. **New §6.1 bullet on stacked pull requests**: a base other than the default branch draws a skip notice rather than a walkthrough, the sanctioned recourse is an `@coderabbitai review` comment (the App's surface, not the CLI), and **R6 limb 2 does not fire on the skip notice** — limb 2's observable is amended to make the re-trigger part of it. **R8's observable given its second form**: a script whose exit code decides a required check as a *step inside another required job* never appears in `required_status_checks.contexts`, which is the cheaper and therefore likelier form. **`process/agent-workflow.md` §1's exclusions made symmetric** — Crucible's cell reduced to the plain prohibition and all four excluded agents named in the §4.4 pointer below the table, since annotating one row read as permitting the other three. Recorded by Atlas under VUL-48. |
| 5 | 2026-10-01 | **The gate's own weakening: one limb was already closed, one is closed here, and the third cannot be closed in this repository.** Closes the §4.2 question Assay raised on VUL-68 against merged code at `platform@41506ad`. The reading is confirmed in full: all three GATE paths in one diff gives `gate=3, prod=0, test=0`, `lane-partition` passes it, `test-erosion` greps `-- '*.rs'` and sees nothing, and `ci/lane-gate-test.sh`'s `setup()` copies only `ci/lane-gate.sh` so the harness cannot construct the diff — fixture 5 is *gate + prod → fail*, fixture 6 is *gate alone → pass*, and there is nothing between them. New **§4.5** splits it into three limbs. **Limb 1, deleting or renaming a check, was already closed by configuration**, which is the fact VUL-68 asked for and could not read: `GET /repos/vulcanflow/platform/branches/main/protection` read back 2026-10-01 returns the four contexts `lane-partition`, `test-erosion`, `inline-test-modules`, `gate-self-test`, `strict: true`, each pinned to `app_id 15368`, with `enforce_admins: true` — the configuration §8.2 records as applied, still in place — so a check that stops reporting leaves a required context expected and never reported and the merge blocks, which is §8.4's *"4 of 4 required status checks are expected"* seen from the other side. VUL-68's conditional severity resolves to its lower branch for that limb only. **Limb 2, the classifier and its own harness in one diff, was open and is closed by two sub-checks** inside the existing `gate-self-test` job, so no new context has to be wired: **monotonicity** — the base harness is run against the head classifier and any case the base asserted the gate *refuses* that head now *permits* is blocking, while the converse is reported and does not block because a stricter gate is not a weakening — and an **assertion floor** on the harness's `check` count, which closes the delete-then-weaken route monotonicity alone leaves open. **Limb 3 is named as unclosable inside the repository and was not in the finding:** branch protection pins job **names**, not job **bodies**, so a diff touching `.github/workflows/lane-gate.yml` **alone** is GATE-only, `lane-partition` permits it, and replacing each job's `run:` with a command that exits zero leaves all four required contexts green while none of the four checks has run — one file, four green checks, no gate, strictly more reachable than the three-file sequence VUL-68 describes and unreachable by any edit to `classify_path()`. Every layer above bottoms out there, monotonicity included, since its step lives in that file. **Two alternatives rejected on the record.** A **pairing rule** (*"`ci/X.sh` and `ci/X-test.sh` may not appear in the same diff"*), which Assay named: monotonicity is strictly stronger on the same case — the pairing rule *splits* a one-pull-request weakening into two and buys legibility where monotonicity *refuses* it — and the pairing rule rests on a filename convention that a GATE-only rename defeats. A third reason this row's own earlier draft gave — that the pairing rule would force the classifier change and its fixture into separate pull requests — is **withdrawn in §4.5 as false**: R10's sequence separates them anyway, so the rejection stands on two reasons and not three. A **class split**, one class per detector: §4.4's *"both would still move together"* is right and applies here, because the defect is not that two files share a class but that a detector and the only assertion that the detector is correct are the same change, so §4.2's table is **unchanged**. **What remains open is stated rather than implied**: in-place relaxation of a fixture — body weakened and expected verdict flipped together, count preserved — is caught by nothing mechanical, and the floor is an endpoint comparison with the same blindness as fixture 12's second half, which Assay flagged, which is deliberate, and which is unchanged. New **§4.6** gives a GATE-class pull request a five-item review obligation, because limb 3's control is lane 6 by construction and "review it carefully" is not a specification: the four job names unchanged and still in `required_status_checks.contexts`; each job still invoking the gate, with a `run:` that no longer reaches the script a blocking finding on its own; `on:` unchanged with no job-level `if:` and no `paths:` filter; every fixture read **against the outcome the lane-1 spec states for it** rather than counted; and any case the harness cannot construct **named in the verdict as uncovered**, which is how limb 2 survived from `41506ad` to VUL-68. **§4.1's second paragraph is narrowed and an overclaim withdrawn** — it closes the *bundled* form of gate-weakening and not the solitary form, and a GATE-only diff is permitted deliberately — and **§4.2 gains a note that its rule partitions *between* classes and says nothing about what travels *within* one**, which is where it bit. New **R9**: limb 3's only real closure is a GitHub control the diff does not contain; two were looked for on 2026-10-01 and neither is in place (`GET /repos/vulcanflow/platform/rulesets` → `[]`, `GET /orgs/vulcanflow` → `plan.name: free`), whether a ruleset `workflows` rule is available on Free for a public repository is **not verified here**, and because that is a plan question of §8's shape the **owner is CEO**. New **R10**: `gate-self-test` being required means the gate's own suite may never be red, so a fixture asserting behaviour the classifier does not yet have is a red required check that cannot merge at all — a standing exception to *tests precede code*, with monotonicity the reason the inversion is not a weakening. §4.5 writes out the **three** pull requests this forces — existing-behaviour fixtures by Scribe, then Forge's implementation, then the new behaviour's fixtures by Scribe — in which **no lane is crossed** and nothing is ever red, only step 3 is inverted, and the control over step 3 is lane 1 plus §4.6 item 4 rather than a mechanism, because §4.4's "written by Scribe before the implementation exists" clause is not satisfiable for a GATE path. **§4 table corrected from 17 fixture diffs to 21**, which §7 item 4 has said since amendment 1 and which this record contradicted itself on in two places; the count is 21 `check` invocations, read off `ci/lane-gate-test.sh` at `41506ad`. §7 gains item 12. Lane assignment for the work §4.5 creates follows §4.4's subject table extended to GATE paths — classifier **Forge** in lane 3, fixtures **Scribe** in lane 2 enumerated with their outcomes in the lane-1 spec, `n/a (no §25 identifier in scope)` under §6.3 condition 2, in R10's three-pull-request order — and `setup()` must gain the ability to write `ci/lane-gate-test.sh` and `.github/workflows/` into the fixture repository, since the three cases this amendment turns on are the three it cannot build. `process/agent-workflow.md` §6 and §7 updated in the same change. **Corrected in lane 6 before merge, from Forge's independent measurement on VUL-68.** Limb 1's `app_id 15368` was asserted in an uncited parenthetical and is now read: `GET /apps/github-actions` returns `id: 15368`, `slug: github-actions`, and the protection read-back is recorded as corroborated by three reads from two agents over two routes, which is the citation this limb needs because nothing in the repository restates the configuration that closes it. **Every date this amendment stamped on its own read-backs said 2026-10-02 and the reads happened 2026-10-01 UTC**; all eight occurrences are corrected, including this row's own date and §4.5's and §4.6's *added* markers — a record whose provenance dates are a day ahead of its evidence is the same defect class as an uncited pin. And §4.6 gains a closing note on its own strength, from the second datum in the same read: `required_approving_review_count: 0` with `require_code_owner_reviews: false` means GitHub will merge a GATE-class pull request with **no approving review at all**, so all five items are enforced by §6.3 inside Paperclip and by nothing in GitHub — not an argument for raising a count §3.1 makes unsatisfiable, but the reason §4.6 is checkable items rather than an instruction to review carefully. Recorded by Atlas under VUL-68. |
| 6 | 2026-10-01 | **§2.3's enforcement is a Paperclip workspace lock, and the two procedural halves of the VUL-83 decision are recorded.** Discharges the closing sentence of §2.3, which said that a recurrence meant the fix was outside this record and CEO's. It recurred twice — a fourth and fifth instance — and CEO chose, applied and read back a control on **VUL-83**, which is the authority for §2.3.1; this record binds it rather than restating it. **New §2.3.1:** `executionWorkspacePolicy.sharedWorkspaceConcurrency` on the VulcanFlow project was an **unset field, not a missing feature**, and is now `serialize`, read back from `GET /api/projects/1eb69546-cf8c-4210-a38c-72c66ad345c3` on 2026-10-01 together with the four values that were already in effect; and because the lock's domain is the **workspace** and `vulcanflow/docs` was registered as none, **setting that field alone would have prevented none of the five instances** — `docs` is now a registered project workspace, `30a88b19-1c78-4db0-afe9-d7e66c494858`, `sharedWorkspaceKey: vulcanflow-docs`, `defaultRef: main`, `isPrimary: false`. §2.3's three bullets are **reclassified as good practice** with the reason they survive stated rather than implied: `serialize` refuses two runs holding one workspace *at the same time*, a branch outlives a run, so one branch / one author run, one correction commit and reconcile-forward still govern the sequential case. The **override asymmetry** is recorded as the one deliberate hole — a review-only issue may carry `executionWorkspaceSettings: {"sharedWorkspaceConcurrency": "allow"}` because a reviewer writes nothing to the tree, an **authoring issue may not**, and a second author run let through that way is the breach. The **cost** is throughput on `platform`; the **revert** is one `PATCH` to `auto` and is **CEO's, not Atlas's**. **Whether `serialize` queues a second run or refuses it is left unasserted**, because the API exposes no read of the lock — no holder field, no queue field — and R11 limb 1 is how that gets answered by observation instead. **New §2.3.2:** a record-authoring issue carries `projectWorkspaceId: 30a88b19-…` or it is **outside the control**, since the project default is `platform` and a lock over an unheld workspace refuses nothing; checkable by one read of `GET /api/issues/{id}`. Demonstrated rather than asserted — VUL-91 carries the field and the run that wrote the section was given `PAPERCLIP_WORKSPACE_ID` equal to it with its working directory in the managed `docs` checkout, the first record-authoring run this organisation has done inside a lock domain. A lane-6 review issue is **excluded by the same premise** the `allow` override rests on, so the existing pairs carrying `platform`'s id (VUL-84) are not a defect to repair. **New §2.3.3:** a lane-6 review pair is keyed `lane6:<owner>/<repo>#<pr>:<head-sha-40>:<assay|warren>` — the **head sha is in the key** because §6.3 condition 3 makes re-filing at a new head correct behaviour that a pull-request-only key would refuse, nothing about the run or the clock is in it, and the reviewer is in it or the pair collapses to one issue. The pair is filed through **`create_task`**, where `idempotencyKey` is a **required** field (schema read 2026-10-01: `minLength 1`, `maxLength 240`, "caller-stable retry key") and which makes the pair children of the authoring issue so the blocker wake costs nothing; **the HTTP route is excluded until verified** — `POST /api/companies/{companyId}/issues` publishes **no request-body schema** in the OpenAPI document and is not among the paths documenting `idempotencyKey`. What the key buys is stated as **one issue per (repository, pull request, head, reviewer) and not a particular status code**, because whether a colliding key returns the existing issue or an error is also not established. And **staleness is read from the title, not the key**, since `GET /api/issues/{id}` exposes no idempotency field (checked on VUL-84) — the `@ <head-sha-7>` form lane 6 already uses is codified for that purpose, which is what instance (i) needed when its withdrawal returned `409`. **New R11**, four limbs: `serialize` refusing rather than deferring (owner CEO, who holds the revert, and the limb exists to answer the question §2.3.1 declines to answer); the `allow` override appearing on an authoring issue (owner Assay, same check it already makes before reading a diff); a record-authoring issue filed without `projectWorkspaceId` (owner Atlas — once is a correction, a pattern means the field should be a default, which is CEO's); and a **seventh instance with all three controls in force**, which would mean the control is in the wrong place rather than advisory, and whose next step is the instance-level isolated-workspace flags on **VUL-93** — `403 {"error":"Board access required"}` to an agent key, read back by CEO with CEO's key and independently with Atlas's, so the board's. §7 gains **item 13**: two runs no longer share a working tree, the price is paid on every item rather than only the concurrent ones because serialization cannot tell a race from a coincidence, and this is the first control in the pipeline whose mechanism lives outside both GitHub and the repository — which is why R11 carries more observables than the triggers above it. **Items 2 and 3 of VUL-91 are in this one amendment and not split**, as the issue allowed: all three are the same decision's procedure, they land in adjacent subsections of one section, and two pull requests editing §2.3 concurrently is the defect §2.3 is about. `process/agent-workflow.md` §1 and §3 and `decisions/README.md` updated in the same change. Recorded by Atlas under VUL-91. |
