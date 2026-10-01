# Agent workflow

**Read this before you touch a file.** It is the operational form of
[ADR-0005](../decisions/ADR-0005-delivery-pipeline-and-lane-enforcement.md). The ADR explains
why; this tells you what to do.

If you only read one thing: **a pull request may contain production source, or tests, but never
both.** CI refuses the combination. Find your lane below and stay inside it.

---

## 1. Find your lane

| You are | Your lane | You change | You never touch |
|---|---|---|---|
| **Atlas** | 1 — Spec | TDD, ADRs, plans, issue specs | production source, tests |
| **Scribe** | 2 — Tests (unit, property, fuzz) | `crates/*/tests/**`, `fuzz/**`, fixtures, goldens | production source; and you never *run* a suite |
| **Ledger** | 2 — Tests (integration, conformance, e2e) | `tests/**`, `crates/*/tests/**`, `conformance/**`, fixtures, goldens | production source; and you never *run* a suite |
| **Forge** | 3 / 5 — Code (pure library crates) | `crates/*/src/**`, `Cargo.toml`, `Cargo.lock`; a NEUTRAL `ci/**` script that is pure computation | any test file, any fixture, any assertion, any `#[cfg(test)]` block |
| **Anvil** | 3 / 5 — Code (service binaries) | `crates/*/src/**`, `Cargo.toml`, `Cargo.lock`; a NEUTRAL `ci/**` script that wraps a service binary | as Forge |
| **Kiln** | 3 / 5 — Code (Kubernetes) | `crates/*/src/**`, `infra` manifests; a NEUTRAL `ci/**` script that touches cluster state | as Forge |
| **Crucible** | 4 / 7 — Run and merge | the ledger, release artefacts | production source, tests |
| **Assay** | 6 — Review by hand | review comments, verdicts | production source, tests |
| **Warren** | 6 — Automated review | the CodeRabbit App's review, triaged into verdicts | production source, tests |

**"Lane 5.5" appears under lane 6 below and is deliberately not a row here.** It is the author's
pre-pull-request CodeRabbit **CLI** pre-flight, owned by whoever opens the pull request whatever
lane they hold. It **gates nothing**: it produces no verdict, it is not one of §6.3's four merge
conditions, and skipping it is not a lane violation. Because it gates nothing, running it on your
own change is not an adjacent-lane violation whatever lane you hold — ADR-0005 §2 scopes that rule
to the seven numbered lanes. The CLI lives on the agent runner, not in any repository. Its
mechanics are still being codified (VUL-28). ADR-0005 §2 and §6.1.

**Writing a `ci/` script? Read ADR-0005 §4.4 first.** Three paths — `ci/lane-gate.sh`,
`ci/lane-gate-test.sh`, `.github/workflows/lane-gate.yml` — are GATE and no one touches them
alongside source or tests. **Everything else under `ci/` is NEUTRAL**, which means the gate lets
it through next to anything and used to mean nobody above claimed it. It is lane-3 work now, and
four things about it are not obvious:

- **It still takes all seven lanes.** Spec, then fixtures, then code, then a run, then two
  verdicts, then a merge. A shell script is not a shortcut around the pipeline.
- **Four agents may not author one, and the table above does not say so** — it is a lane table,
  not a path table, and the exclusions turn on the script's *subject* rather than on anyone's
  lane. **Crucible** may not author a script that audits lane 4 or lane 7; **Atlas** may not
  author one at all; **Assay** and **Warren** may not either. ADR-0005 §4.4 gives each reason.
  Those four rows are deliberately not annotated above, because a partial copy of §4.4 in a
  lane table is how the rule gets cited in the wrong shape.
- **It gets no §25 identifier**, so its ledger line reads `n/a (no §25 identifier in scope)` and
  the acceptance statement in the spec is what it is judged against. Do not invent an identifier.
- **Its fixture harness is Scribe's**, written before the script, from a fixture list Atlas
  enumerates in the spec — each fixture with the outcome it must produce. The harness is NEUTRAL
  too, so no check stops a coding agent from touching it. That is a temporal control, not a
  structural one: nothing mechanical catches you, which is exactly why you do not.

**One run per working tree is now also a platform setting — and what the setting does has not been
read back, so it does not replace your discipline.** The VulcanFlow project is set to
`sharedWorkspaceConcurrency: serialize`, whose subject is two runs holding one workspace at the same
time. Five things follow for you, and **ADR-0005 §2.3 is the rule — this is the pointer**:

- **On `platform`, expect lane steps to take turns.** A run that would previously have started
  alongside another may not. That is the intended cost, not a fault to work around, and nothing
  about it is yours to change: the policy and its revert are CEO's.
- **Behave as though it were not there.** The lock exposes no holder and no queue, the first
  observation taken with it in force showed three runs in one `docs` tree (ADR-0005 §2.3.1), and
  nothing covers a branch *across time* in any case — the next run can still push to a branch an
  earlier one opened. So one change / one branch / one author run, one correction commit answering a
  verdict, and **reconcile forward, never force-push** are binding, and a violation is a blocking
  finding in lane 6.
- **Editing the design of record? The issue must carry `projectWorkspaceId`** — the `docs` workspace,
  `30a88b19-1c78-4db0-afe9-d7e66c494858`. Without it the run works outside the lock domain and the
  control does not reach it. Atlas sets it when filing; if you are handed a record-authoring issue
  without it, say so before you start. ADR-0005 §2.3.2.
- **A lane-6 pair filed from such an issue inherits that field and must be moved back off it** —
  one `PATCH /api/issues/{id}` after filing, by whoever filed it. A reviewer needs no checkout of
  the record repository, and leaving the pair in that domain is what produced the three-run tree
  above. ADR-0005 §2.3.2 gives the two field values.
- **If your wake banner says another run concurrently holds your workspace, that is R11 limb 5 and
  it is reportable** — quote the banner, both run ids and the workspace id to CEO. Do not infer from
  it that the lock is broken, and do not infer from a quiet banner that it is working.

---

## 2. What a work item looks like, start to finish

### Lane 1 — Atlas writes the spec

Nothing starts without a spec. Atlas puts on the issue:

- the **§25 test identifier(s)** the item is answerable by, named exactly as §25 names them;
- an **acceptance statement** — what is true when this is done, in terms a test can assert;
- the crate and module the work belongs in, and any ADR that constrains it.

Then the issue goes to the test author. Not to a coding agent.

### Lane 2 — Scribe or Ledger writes the test, first

Write the test from the spec. You have not seen the implementation and you do not need it.

- Unit, property and fuzz tests → **Scribe**.
- Integration, conformance and e2e → **Ledger**.
- Unit tests go in `crates/<crate>/tests/`, against the crate's **public API**. There are no
  `#[cfg(test)]` modules in this workspace — CI refuses them, and ADR-0005 §5 says why.
- Name the test function after the §25 identifier so the ledger can key on it.

**You do not run it.** A test that exists and fails is the correct state to hand over — it is
what proves the test is actually testing something. Open a pull request containing **only test
files**, note the §25 identifier in the description, and hand off (§3).

### Lane 3 — Forge, Anvil or Kiln writes the code

You are given the §25 identifier and a failing test. Implement until it passes.

- Production source only. If you find yourself opening a file under `tests/` or `fuzz/`, stop —
  you are out of your lane and CI will refuse the pull request.
- You cannot run the suite either. Crucible runs it. Write the code, reason about why it
  satisfies the assertion, hand off.
- Open a pull request containing **only production source** (plus any docs you want to add).

### Lane 4 — Crucible runs the suites and publishes the ledger

Crucible is the only agent that executes anything. It publishes the ledger (§4) keyed by §25
identifier.

### Lane 5 — red goes back to lane 3

**A failing test is a defect in the code.** Not in the test.

It routes back to the coding agent as a code fix. The test is not edited, not relaxed, not
`#[ignore]`d, not deleted, not quarantined, and its assertion is not loosened. If you are
holding a red test and your instinct is to change it, that instinct is the thing this process
exists to stop.

The single exception is §5 below, and it does not start with the test.

### Lane 6 — Assay and Warren both review

- **Assay** reviews by hand against TDD v2.3 and the ADRs: correctness, crate boundaries,
  `unsafe` rules, supply-chain diffs. Assay also checks the lane **first**, before reading the
  diff, and rejects a crossing on sight.
- **Warren** reads the **CodeRabbit GitHub App's** review on the pull request, posts the
  verdict, and sorts findings into blocking versus advisory so the bot cannot gate a merge on
  style. Warren runs no tool to produce this: lane 6's route is the App's review on the pull
  request. The authenticated CodeRabbit **CLI** is a different surface — the pre-pull-request
  pre-flight, lane 5.5 — and **a CLI review is never reviewer #2's verdict, whoever runs it,
  including Warren**: it emits no coverage anchors, so §6.3's head-coverage check on the verdict
  cannot be performed at all (ADR-0005 §6.1).

Each verdict goes **on that reviewer's own lane-6 issue** — one issue per reviewer — and not as a
GitHub review approval, because there is one GitHub identity in this organisation and GitHub
approvals cannot represent two reviewers (ADR-0005 §3.1).

**What a verdict must contain to count is ADR-0005 §6.3 condition 3, and this section does not
restate it** — the same rule lane 7 below is written under, for the same reason. Read it before
you post, every time. It is four clauses, and one of them is why a verdict you have already
posted can stop counting without anyone editing it.

> **The CodeRabbit App is installed** — `coderabbitai`, app id `347564`, on the organisation
> since `2026-10-01T19:12:07Z`. Lane 6 has both reviewers again, and the single-reviewer
> degradation notice that stood here until 2026-10-01 is **withdrawn**. ADR-0005 §6.1.
>
> **But the bot is not the reviewer.** `coderabbitai[bot]` commenting on a pull request is
> *input* to Warren's verdict, not the verdict. A pull request with a detailed bot thread and no
> Warren verdict recorded in Paperclip is **unreviewed**, and Crucible does not merge it.
> CodeRabbit submits with state `COMMENTED`, which is not an approval, and its review must cover
> the current head to count at all. ADR-0005 §6.2.

### Lane 7 — Crucible merges

**The merge condition has exactly one statement and it is ADR-0005 §6.3.** This section does not
restate it, summarise it, or add to it — it tells you where to read it and what the four
conditions are called, because a second phrasing is how `docs#24` came to merge over a
`REQUEST CHANGES` with no reviewer #2 verdict at all (ADR-0005 §6.4). Read §6.3 before every
merge.

The four conditions are called, in order: **(1)** the **lane gate**, **(2)** the **ledger**,
**(3)** the **two reviewer verdicts**, **(4)** the **merge attestation**. What each one requires
is ADR-0005 §6.3 and is not reproduced here — the paragraph above forbids a second phrasing, and
a list of names that looked close enough to a summary is how the second phrasing gets back in.

Missing any one of those, Crucible refuses and says which one. Nothing merges by any other
route — including by whoever has admin. An unattested merge on `main` is a recorded gate defect
under ADR-0005 §10 R5b, not a judgement call that turned out differently.

---

## 3. What a handoff looks like

A handoff is a comment on the Paperclip issue. It carries five things, and it is not a handoff
if it is missing one:

```
Lane:        2 → 3  (tests written, ready for implementation)
Identifier:  core/scope-token-parse        (§25)
Acceptance:  a malformed scope token is rejected without panicking, and the
             error names the offending segment
Artefact:    vulcanflow/platform PR #41 — crates/vf-core/tests/scope_token.rs
State:       test written, NOT run (lane 2 does not execute)
Next owner:  Forge
```

The lane transitions that exist:

| From → to | Meaning | Who hands off | Who picks up |
|---|---|---|---|
| 1 → 2 | spec ready | Atlas | Scribe / Ledger |
| 2 → 4 | test written, needs a first run to confirm it fails | Scribe / Ledger | Crucible |
| 4 → 3 | test confirmed red; implement | Crucible | Forge / Anvil / Kiln |
| 3 → 4 | code written, needs a run | Forge / Anvil / Kiln | Crucible |
| 4 → 6 | suite green; review | Crucible | Assay **and** Warren |
| 4 → 3 | suite red; code defect | Crucible | Forge / Anvil / Kiln |
| 6 → 7 | both verdicts satisfy ADR-0005 §6.3 condition 3 | Assay / Warren | Crucible |
| 6 → 3 | review found a defect | Assay / Warren | Forge / Anvil / Kiln |
| 2 → 1 | the test may be asserting something unspecified | Scribe / Ledger | Atlas (see §5) |

Note `4 → 3` appears twice. Confirming a test is red and reporting a regression are the same
transition; both hand a coding agent a named failing identifier.

**The `4 → 6` handoff is two issues and they are keyed, so the pair cannot be filed twice.** One
issue per reviewer (ADR-0005 §6.1). Each carries the head in its title and an `idempotencyKey`
carrying the head too:

```
title: … <repo>#<pr> @ <head-sha-7> … (reviewer #<1|2>, <Assay|Warren>) …
key:   lane6:<owner>/<repo>#<pr>:<head-sha-40>:<assay|warren>
```

Those two title fragments are what is required; the rest of the title is the subject. **A review
issue whose `@ sha` is not the pull request's current head is stale by inspection** — that is what
the title fragment is for, and it is the half that works without the key working.

**ADR-0005 §2.3.3 is the rule**: which call to file through, why the other route is excluded, why
each limb is in the key, and what the key is and is not claimed to do. Two steps that are easy to
miss and are not optional: **re-file the pair when the head moves** (the key changes with it, by
design), and **PATCH the new pair off the record workspace** it inherits (§2.3.2).

---

## 4. What the ledger looks like

Crucible publishes this after every run, on the issue. It is mechanical: it reports what the
runner said, with no interpretation.

```
Ledger — vulcanflow/platform @ 9f2c1ab — 2026-10-01T14:02Z
Suite: cargo test --workspace --all-targets   (aarch64-unknown-linux-gnu)

§25 identifier                        Status   Test function
------------------------------------  -------  ----------------------------------------
core/scope-token-parse                PASS     scope_token::rejects_malformed
authz/role-layer-denies-cross-tenant  PASS     role_layer::denies_cross_tenant
db/tenanttx-set-local-isolation        FAIL     tenanttx::set_local_is_transaction_scoped
graph/cycle-detection                 MISSING  —

Summary: 2 PASS, 1 FAIL, 1 MISSING  →  NOT MERGEABLE
FAIL   db/tenanttx-set-local-isolation
       assertion failed: expected SET LOCAL to be rolled back with the transaction
       crates/vf-db/tests/tenanttx.rs:84
       → routes to lane 3 (Anvil) as a code defect
MISSING graph/cycle-detection
       no test function found for this identifier
       → routes to lane 2 (Scribe); the spec names it and nothing asserts it
```

Three statuses and nothing else:

- **PASS** — a test exists for the identifier and it passed.
- **FAIL** — a test exists and it failed. **Always** a code defect; routes to lane 3.
- **MISSING** — the spec names the identifier and no test asserts it. Routes to lane 2. This is
  the status that stops an item being declared done because nobody wrote the test.

A FAIL is never reported as "flaky", "known" or "pre-existing". If a test is genuinely
non-deterministic, that is a defect report against the test, raised to Atlas under §5 — not a
line in the ledger that gets skipped past.

---

## 5. The one escape hatch

Use this only when a test asserts behaviour **the TDD never promised** — not when a test is
inconvenient, and not when a test is red.

1. **Stop.** Do not edit the test. Raise it to Atlas with the identifier and what the test
   asserts that the spec does not.
2. **Atlas moves the spec first** — amends the TDD or files an ADR. In the open, before any test
   changes.
3. **The original test author** — the one who wrote it, not whoever hit the red — rewrites the
   test.
4. The rewrite goes on a **test-only pull request containing no production code.**

If you are a coding agent, your part of this ends at step 1. Steps 2–4 are not yours, and the
gate will refuse your attempt at step 4 anyway.

---

## 6. The gate, and what it will say when you cross a lane

Four required checks run on every pull request to `main`, from
[`ci/lane-gate.sh`](https://github.com/vulcanflow/platform/blob/main/ci/lane-gate.sh):

| Check | Fails when |
|---|---|
| `lane-partition` | the diff has both production source and test files — or changes the gate alongside either |
| `test-erosion` | an `#[ignore]` was added, a test declaration was removed, or the workspace test count fell |
| `inline-test-modules` | a `#[cfg(test)]` module exists under `crates/*/src` |
| `gate-self-test` | the gate's own fixture assertions fail; or the head classifier **permits** a case the base harness asserts the gate **refuses**; or the harness's assertion count fell (ADR-0005 §4.5) |

### Which paths are which

| Class | Paths |
|---|---|
| **GATE** | `ci/lane-gate.sh`, `ci/lane-gate-test.sh`, `.github/workflows/lane-gate.yml` |
| **TEST** | `tests/**`, `crates/*/tests/**`, `fuzz/**`, `conformance/**`, `**/testdata/**`, `**/golden/**`, `*.golden`, `*.snap` |
| **PROD** | `crates/*/src/**`, `crates/*/build.rs`, `crates/*/Cargo.toml`, `Cargo.toml`, `Cargo.lock`, `rust-toolchain.toml` |
| **NEUTRAL** | everything else — Markdown, `.gitignore`, other workflows, other `ci/` scripts |

At most one of GATE, TEST, PROD per pull request. NEUTRAL rides along with any of them, so your
code change can carry its own docs.

**That rule is about what travels *between* classes. It says nothing about what travels within
one, and a GATE-only diff is permitted on purpose** — the gate has to be able to change. What
guards a GATE diff instead is ADR-0005 **§4.5** (the `gate-self-test` row above) and **§4.6**, the
five-item review obligation that applies to every pull request touching a GATE path. If you are
touching one, read both before you open it.

### Run it yourself before you push

The gate is a shell script with no dependencies. Checking is cheaper than a red CI run:

```sh
ci/lane-gate.sh partition            # against origin/main...HEAD
ci/lane-gate.sh all origin/main HEAD
```

### When it fails, do not work around it

A red `lane-partition` means split the pull request — not add a path to the NEUTRAL list. The
gate only changes through its own pull request, with a new fixture in `ci/lane-gate-test.sh`
asserting the new behaviour, and under an ADR-0005 revisit trigger. A gate edit bundled with the
change it would permit is itself refused — **that is the bundled form only, and the solitary form
is permitted; ADR-0005 §4.5 is what guards it.**

**Changing what the gate does takes three pull requests, in this order, and this is the one place
in the pipeline where lane 2 does not precede lane 3.** `gate-self-test` is a required check, so a
fixture asserting behaviour the classifier does not yet have is a red required check and cannot
merge at all (ADR-0005 **R10**). So: **(1)** Scribe lands fixtures for the behaviour the gate
*already* has, green; **(2)** Forge lands the change to `ci/lane-gate.sh`, green because it is
additive; **(3)** Scribe lands fixtures for the new behaviour, green because it now exists. Nobody
crosses a lane and nothing is ever red. Only step 3 is inverted, it applies to GATE paths and
nothing else, and §4.5's monotonicity sub-check is why the inversion is not a weakening: what your
pull request must survive is the fixture set already on `main`. Step 3's fixtures are enumerated
**with their outcomes** in the lane-1 spec before step 2 is written — that enumeration and §4.6
item 4 are the whole control, because there is no mechanism for it.

---

## 7. What GitHub now enforces, and what it still does not

**`main` is protected on `docs`, `platform`, `infra` and `vf-api`.** Do not plan around a direct
push; there isn't one.

- **You cannot push to `main`.** Not with a merge commit, not with an empty commit, not as
  admin. `enforce_admins` is on. Every change arrives as a pull request.
- **You cannot force-push or delete `main`.** History on the default branch is append-only.
- **On `platform`, a red lane-gate check blocks the merge.** All four checks are required, and
  the merge API returns `405` while any one of them is failing. The branch must also be current
  with `main` before it can merge — use "Update branch", do not force-push your own branch over
  a reviewed diff.
- **Nobody can approve your pull request, and nothing requires them to.** There is one GitHub
  identity for all agents, and GitHub refuses an author's own approval, so required approvals
  are set to zero (ADR-0005 §8.2). This is not permission to skip review: **Assay's and
  Warren's verdicts live on their own lane-6 Paperclip issues and Crucible checks each of them
  against ADR-0005 §6.3.** A green GitHub merge button is not an approval.

Still gaps, and still not permission:

- **Nothing in GitHub stops a merge that ignores the verdicts, and this has happened.** The
  merge button is green whenever protection and required checks are satisfied; it knows nothing
  about `REQUEST CHANGES`, about a missing reviewer #2, or about a verdict covering a commit
  that is no longer the head. `docs#24` merged on 2026-10-01 over all three at once — ADR-0005
  §6.4 records it under R5b. Until the attestation detector in §6.3 condition 4 exists, the only
  thing between a verdict and `main` is whoever is at the keyboard reading §6.3. Read it.

- **`docs`, `infra` and `vf-api` have no required checks** — they have no CI workflow on `main`
  yet. They are protected, so pull requests are still mandatory. If you add the first workflow
  to one of them, you also wire its checks as required (ADR-0005 R2).
- **The lane gate itself is on `platform` only.** `docs` has no code; `infra` and `vf-api` are
  empty. The gate goes in the moment a repository receives source.
- **A lane crossing split across two pull requests passes both.** The gate partitions paths, not
  people. VUL-9 is what closes that.
- **The gate cannot fully guard the file that decides whether it runs.** Branch protection holds
  the four check **names**, not the four job **bodies**. A pull request touching
  `.github/workflows/lane-gate.yml` and nothing else is GATE-only, so `lane-partition` permits it,
  and a `run:` step rewritten to exit zero leaves all four required contexts green while none of
  the four checks has run. Deleting or renaming a job is **not** this — a required context that
  never reports blocks the merge, which is the mechanism behind the `4 of 4 required status checks
  are expected` push refusal in §8. The weakening that passes is the one that keeps the names and
  changes the bodies. ADR-0005 §4.5 limb 3 records why no path rule reaches it, §4.6 is the review
  obligation that stands in for a mechanism, and **R9** is the escalation. Treat that file as the
  highest-consequence path in the repository: if you are reviewing a diff that touches it, §4.6
  item 2 is a blocking finding on its own.
- **A bot thread can be mistaken for reviewer #2.** The CodeRabbit App is installed and posts on
  every pull request, but nothing in GitHub distinguishes its commentary from a review verdict.
  Lane 6 is **two Paperclip verdicts that each meet ADR-0005 §6.3 condition 3** — not two
  comments, and not two of anything merely being present (ADR-0005 §6.2 corollary 1). The
  `lane6-review-verdict` procedure is also attached to Assay rather than Warren today (VUL-36);
  that is a grant defect and does not move reviewer #2 to Assay.
- **The four repositories are public.** Anything you commit is world-readable the moment it is
  pushed, including on a branch you later delete. Secret scanning and push protection are on,
  but they only catch *recognised* credential shapes — they will not catch a customer name, an
  internal hostname or an unreleased detail you did not mean to publish.
- **GitHub does not know two of your runs are in the same tree, and it never will.** Nothing in
  this section is what stops that. The control is Paperclip's workspace lock (§1 above, ADR-0005
  §2.3.1) and it lives outside both GitHub and this repository — so if a push does something you
  did not expect, "protection is on" is not the answer to look for. Check whether another run of
  yours was there first.

---

## 8. The gate, demonstrated

A gate nobody has seen fail is a claim. Two refusals have been observed and both are linked from
the VUL-2 issue:

- **`platform` PR #3** — the lane-partition check going red on a deliberate crossing, back when
  red had no consequence.
- **`platform` PR [#4](https://github.com/vulcanflow/platform/pull/4)** — the same crossing after
  protection landed: `lane-partition` ❌, the other three ✅, `mergeable_state: blocked`, and
  `PUT .../merge` → `405 Required status check "lane-partition" is failing`. A direct push to
  `main` was refused in the same session as repository admin (`GH006`).

Details and exact output in ADR-0005 §8.4.

---

*Authority: [ADR-0005](../decisions/ADR-0005-delivery-pipeline-and-lane-enforcement.md). Where
this document and the ADR disagree, the ADR wins and this document is the bug.*
