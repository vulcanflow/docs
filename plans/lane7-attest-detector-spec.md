# Lane-1 spec — the lane-7 attestation detector

**Issue:** VUL-48 (lane 1, Atlas). **Lane 2:** VUL-56, Scribe. **Lane 3:** VUL-50, Forge.
**Status:** authoritative on merge. Until then it is a proposal and the two implementers work
from it at the risk named in §9.

ADR-0005 §6.5 fixes two things about the detector at record level — **where the assertion
starts** and **what it asserts** — and delegates the rest by name: *"Everything else — the
script's structure, its output format, its enumerated fixture table — is the lane-1 spec on
VUL-48 and the child issues that issue names."* **This document is that spec.** It decides
nothing §6.5 decided and it adds no finding class §6.5 does not describe; it fixes the
**interface** between the harness and the detector, so that lane 2 and lane 3 implement the
same side of it.

Where this document and ADR-0005 disagree, **ADR-0005 wins** and this document is corrected.

Why it is a document rather than a comment: both authors have to read the same thing twice —
once to write, once at review — and an interface settled in an issue thread gets re-guessed.
Nine of the questions below were asked because §6.5 named the output format as out of its own
scope and nothing then held it.

---

## 1. Invocation

Three subcommands, in the shape `ci/lane-gate.sh` already established (one subcommand per
`check` call in the harness).

| Invocation | Does |
|---|---|
| `ci/lane7-attest.sh commit <owner>/<repo> <sha>` | Classifies **exactly one** commit. Reads git facts from the repository in the current working directory. Clones nothing. |
| `ci/lane7-attest.sh repo <owner>/<repo>` | Resolves that repository's floor from the manifest, emits its `RANGE` line, then classifies every commit in `git rev-list --first-parent <floor>..refs/heads/main`, oldest first. Reads git facts from the repository in the current working directory. Clones nothing. |
| `ci/lane7-attest.sh run [<owner>/<repo> ...]` | The production entry point. Clones each named repository into a scratch directory and applies `repo` to each. With no arguments, the scan set is **every row of the manifest**. |

**`commit` and `repo` read the current working directory on purpose.** It is what lets the
harness build a throwaway repository under `mktemp -d`, `cd` into it, and assert a verdict with
no network and no clone — the mechanism `ci/lane-gate-test.sh` already uses. `run` is the only
subcommand that clones, and the harness never calls it.

**The scan set and the manifest are different inputs.** That is what makes
`L7-NO-FLOOR` (§4, fixture 27) assertable: a repository can be in the scan set with no manifest
row. If they were the same input the finding would be unreachable, and §6.5 requires it —
*"a repository that gains a protected default branch with no row above is not silently
skipped."*

---

## 2. The floor manifest — answers item 1

**Path:** `ci/lane7-attest-floors.txt` in `vulcanflow/platform`.

**Resolved as `$(dirname "$0")/lane7-attest-floors.txt`** — relative to the script, not to the
working directory and not from an environment variable. The harness copies the detector into
its fixture repository's `ci/` and writes the manifest beside it, exactly as
`ci/lane-gate-test.sh`'s `setup()` copies the gate. Deliberately **not** overridable: a floor
that can be redirected at run time is the thing §6.5's *"moving a floor is an amendment to this
record"* exists to prevent, and an override would reintroduce the run-time floor §6.5 rejected.

**Format.** One record per line; two whitespace-separated fields; nothing else.

```
# ci/lane7-attest-floors.txt — the §6.5 floor manifest.
# Moving a floor is an amendment to ADR-0005 §6.5, not an edit to this file.
#
# <owner>/<repo>            <40-hex floor sha — the newest commit NOT asserted on>
vulcanflow/docs             b31ddfeca0ad88cb481929f7eacd64d7e8194ac0
vulcanflow/platform         41506ad3bea5282473207a725b00625c5f65e0aa
vulcanflow/infra            304b300e01e0c9ad8210977d80986afe65517daa
vulcanflow/vf-api           45d7ded9d32d0fff8e60a17c97c725c475748497
```

Rules, all of them assertable:

- Blank lines and lines whose first non-blank character is `#` are ignored.
- Field 1 is `<owner>/<repo>`; field 2 is **exactly 40 lowercase hex characters**. A short sha
  is a malformed row, because a manifest is a record and a record is unabbreviated.
- A third field, a duplicate repository, or a malformed row is a **usage error**: exit `2`
  (§5), not a finding. The manifest is the detector's own configuration and a broken one means
  the run did not happen.
- Row order is not significant. The four shas above transcribe §6.5's table verbatim.

---

## 3. Output — answers items 4 and 6

**stdout is the ledger and nothing else.** Every diagnostic, progress note, warning and error
message goes to **stderr**. A harness that greps stdout is then asserting the ledger rather than
the prose, which is what makes fixture-by-fixture assertion possible at all.

Exactly four line shapes may appear on stdout, in this order of first appearance:

| Shape | Emitted | Meaning |
|---|---|---|
| `MODE live` / `MODE fixture <dir>` | **always, as the first stdout line** | which fact source the run used (§6) |
| `RANGE <owner>/<repo> <count>` | once per repository classified, before its commits | how many commits were in range — `0` is the empty range |
| `OK <owner>/<repo> <sha40>` | once per commit in range carrying **no** finding | the commit is clean |
| `L7-<CODE> <owner>/<repo> <sha40>[ -- <free text>]` | once per **finding** | one line per finding, never per commit |

- A code matches `^L7-[A-Z0-9-]+$`. The three fields before ` -- ` are the contract. Everything
  after ` -- ` is prose for a human, **and the harness must not assert on it** — so a reworded
  message can never turn a fixture red.
- `OK` and a finding line are mutually exclusive for one commit. A commit with two findings
  produces two lines and no `OK` (fixture 26).
- `L7-NO-FLOOR` is the one finding keyed to a repository rather than a commit; its third field
  is the literal `-` and no `RANGE` line is emitted for that repository.
- Ordering is specified for a human reader — repositories in scan-set order, commits oldest
  first, findings within a commit in §6.5 class order. **The harness compares the set of
  stdout lines, not their order**, so ordering is never what makes a fixture red.
- Nothing else on stdout may begin `MODE `, `RANGE `, `OK ` or `L7-`.

**`RANGE` is the answer to item 6, and it is better than a finding code for the empty range.**
It is emitted for every repository rather than only the empty ones, so the ledger states its
own scope; it carries the count, so an unexpectedly *short* range is visible too; it is not a
finding, so it cannot touch the exit code; and it is a token rather than a sentence, so the
empty-range wording can be rewritten without going red. `RANGE vulcanflow/infra 0` is the whole
of fixture 28's positive assertion.

---

## 4. The finding-code register — answers items 4 and 8

Twenty-six codes. **The code is the contract**; the prose after ` -- ` is not. The §6.5 column
names the finding class this code implements.

| Code | §6.5 | Fires when |
|---|---|---|
| `L7-MISSING` | 1 | No `Lane-7-*` line anywhere in the message. Suppresses classes 2 and 4–8 per §6.5. |
| `L7-KEY` | 1 | One of the **four single-valued keys** — `Lane-7-Head`, `Lane-7-Gate`, `Lane-7-Ledger`, `Lane-7-Merged-By` — is absent from a block that is otherwise present. Key-agnostic: the key is named in the prose only. |
| `L7-NOT-TRAILING` | 1 | Content other than §7's permitted trailers appears after the block. |
| `L7-HEAD` | 2 | `Lane-7-Head` ≠ the commit actually merged. |
| `L7-HEAD-UNRESOLVABLE` | 2 | The pull-request head cannot be resolved although the pull request *was* identified. |
| `L7-NOT-PR` | 3 | The commit reached `main` by neither a merge nor a squash of a pull request. |
| `L7-PR-UNCHECKED` | 2, 3 | The pull-request association lookup could not run. Suppresses `L7-NOT-PR`, `L7-HEAD` and `L7-HEAD-UNRESOLVABLE` for that commit. |
| `L7-GATE-BARE` | 4 | `Lane-7-Gate: n/a` with no parenthesised reason. |
| `L7-GATE-VACUOUS` | 4 | `Lane-7-Gate: n/a` in any form on a commit whose tree contains `.github/workflows/lane-gate.yml`. |
| `L7-GATE-VALUE` | 4 | `Lane-7-Gate` is neither `PASS` nor an `n/a` form — e.g. `FAIL`. |
| `L7-GATE-UNCONFIRMED` | 8 | `Lane-7-Gate: PASS` is attested and the check-run conclusions on `Lane-7-Head` disagree — any required check failing, missing or still running. |
| `L7-GATE-UNCHECKED` | 8 | The check-run lookup could not run, so an attested `PASS` was not confirmed. |
| `L7-LEDGER-BARE` | 5 | `Lane-7-Ledger: n/a` with no parenthesised reason. |
| `L7-LEDGER-VACUOUS` | 5 | `Lane-7-Ledger: n/a` on a commit touching `crates/**`. **The one advisory code** (§5). |
| `L7-VERDICT-COUNT` | 6 | The number of `Lane-7-Verdict-*` lines in a present block is not exactly two — including **zero**. |
| `L7-VERDICT-DUP` | 6 | Two verdict lines name the same reviewer. |
| `L7-VERDICT-WHO` | 6 | A verdict line names an agent outside {Assay, Warren}. |
| `L7-VERDICT-DISP` | 6 | A verdict's disposition is not exactly `APPROVE`. `APPROVED` is not `APPROVE`. |
| `L7-VERDICT-ISSUE` | 6 | A verdict line cites no `VUL-<n>`. **Shape only** — no lookup. |
| `L7-VERDICT-COVERS` | 6 | A verdict's `covers <sha>` is absent, is not 7–40 lowercase hex, or does not match `Lane-7-Head`. |
| `L7-VERDICT-UNCHECKED` | — | No Paperclip credential, so the citation's *contents* were not checked (§6.5's second limit). **Not advisory** (§5). |
| `L7-VERDICT-ISSUE-MISSING` | — | The lookup ran and the cited `VUL-<n>` **does not exist**. |
| `L7-VERDICT-NOT-FOUND` | — | The lookup ran, the issue exists, and it records **no `APPROVE` covering `Lane-7-Head`** by the reviewer named. |
| `L7-VERDICT-MISATTRIBUTED` | — | The lookup ran and the issue records an `APPROVE` covering `Lane-7-Head`, but recorded by an agent other than the one the verdict line names. |
| `L7-MERGED-BY` | 7 | `Lane-7-Merged-By` is not `Crucible`. |
| `L7-NO-FLOOR` | — | A repository in the scan set has no manifest row. |

### 4.1 The four lookup codes exist because of a live defect, and why they are four

Forge's question 4 is correct: fixture 24 covered only the **degraded** path, and nothing covered
the lookup that *ran and should have said no*. That is the path on which the withdrawn head of
`platform#6` printed `OK` for the exact merge R5b exists to make visible — Assay's blocking
finding B1 on VUL-57, verified against live Paperclip data at VUL-31 and VUL-32.

They are four codes and not one because the repairs are four different pieces of work:

- `L7-VERDICT-ISSUE-MISSING` is a **transcription defect in the attestation** — a wrong issue
  number. The repair is the merge message.
- `L7-VERDICT-NOT-FOUND` is a **lane-6/lane-7 defect** — the verdict relied on does not exist.
  This is §6.4's actual event and collapsing it into the row above would make §6.4's own record
  unrepresentable in the detector's output.
- `L7-VERDICT-MISATTRIBUTED` is a **§6.2 corollary 2 defect** — one reviewer's approval
  attributed to the other, which is how two verdicts become one.
- `L7-VERDICT-UNCHECKED` is **not a defect in the history at all**. It is the run saying it
  could not look. It still exits non-zero (§5), because that is the whole of fixture 24.

### 4.2 Three parse rules that are the repair for B1, stated as rules rather than codes

1. **A verdict at the issue is one record.** A match requires the reviewer, the disposition
   `APPROVE` and the covered sha to belong to **one parsed verdict record**. Co-occurrence
   anywhere in a body is explicitly **not** a match — that is exactly what let `REQUEST CHANGES`
   at VUL-31 satisfy an `APPROVE` citation. The failing code is `L7-VERDICT-NOT-FOUND`.
2. **No lookup is ever performed with an empty sha.** If `Lane-7-Head` is absent, `L7-KEY`
   fires and **the verdict-contents check does not run** — no `L7-VERDICT-*` line is emitted for
   that commit. `"" in body` being vacuously true is the second limb Forge reproduced, and this
   rule is what closes it. Fixture 30 asserts the absence.
3. **The reviewer is checked at both ends.** Being in {Assay, Warren} on the verdict *line* is
   `L7-VERDICT-WHO`; being the agent that actually recorded the verdict *at the issue* is
   `L7-VERDICT-MISATTRIBUTED`. Two ends, two codes, because a block can be well-formed and still
   cite the wrong author.

---

## 5. The advisory set and the exit code — answers item 5

**The advisory set is exactly `{L7-LEDGER-VACUOUS}`.** Nothing else is advisory. §6.5 names
exactly one thing the detector *"cannot decide"* and reports *"as advisory for lane 6 to
adjudicate"*, and this set is that sentence and no more.

| Exit | When |
|---|---|
| `0` | The ledger is complete and no **non-advisory** finding line was emitted. Advisory findings never change the exit code. |
| `1` | The ledger is complete and at least one non-advisory finding line was emitted. |
| `2` | The run did not complete: usage error, malformed or unreadable manifest, missing subcommand argument, not a git repository. |

**`2` is distinct from `1` on purpose.** "The history is defective" and "the detector broke" are
different events with different owners, and a single non-zero code would let the second be read
as the first.

**Every `UNCHECKED` code is non-advisory and exits `1`.** `L7-VERDICT-UNCHECKED`,
`L7-PR-UNCHECKED` and `L7-GATE-UNCHECKED` all do. A check that passes when it could not run is
the shape of §6.2 corollary 4 and of §6.4, and fixture 24 exists to assert that this detector
does not have it.

---

## 6. The fixture seam — answers items 3 and 8's second half

The harness cannot reach the network: `ci/lane-gate-test.sh` pins
`export PATH="/usr/bin:/bin:/usr/local/bin"` and builds its world under `mktemp -d`. §6.5's
classes 2, 3, 6 and 8 all need a fact that is not in git. Scribe named three mechanisms and ruled
out shadowing `curl` on `PATH` for the right reason — *if the only way to write the assertion is
to read the implementation, the spec is the bug*. **Mechanism (3) is adopted: one named injection
point the detector owns.**

### 6.1 `LANE7_FIXTURE_DIR`

When `LANE7_FIXTURE_DIR` is set and non-empty, the detector makes **no network call of any
kind** and reads every non-git fact from flat files beneath it. If it is set and does not name an
existing directory, that is exit `2`.

| File | Content | Absent means |
|---|---|---|
| `pulls/<sha>` | one line: the pull-request number, or the literal `none` | the lookup **failed** → `L7-PR-UNCHECKED` |
| `pull-head/<n>` | one line: the sha `refs/pull/<n>/head` resolves to | the ref does not resolve → `L7-HEAD-UNRESOLVABLE` |
| `check-runs/<sha>` | one line per required check: `<name> <conclusion>` | the lookup **failed** → `L7-GATE-UNCHECKED` |
| `verdicts/<VUL-n>` | one line per recorded verdict: `<reviewer> <disposition> <covered-sha>`; present and empty = the issue exists and records none | the issue **does not exist** → `L7-VERDICT-ISSUE-MISSING` |

`<disposition>` is `APPROVE` or `REQUEST_CHANGES` (underscored, because the field is
whitespace-separated). The distinction between *absent file* and *present-but-empty file* is
load-bearing and is what fixtures 33 and 34 turn on.

### 6.2 The two credentials — item 3

| Variable | Gates |
|---|---|
| `LANE7_GITHUB_TOKEN` | §6.5 classes 2, 3 and 8 — the `pulls`, `pull-head` and `check-runs` lookups |
| `LANE7_PAPERCLIP_TOKEN` | the verdict-**contents** check — §6.5's second limit |

**Both are required to be non-empty even under `LANE7_FIXTURE_DIR`.** That is what keeps fixture
24 honest: it unsets `LANE7_PAPERCLIP_TOKEN` while leaving a complete `verdicts/` file in place,
so `L7-VERDICT-UNCHECKED` is produced by the credential being absent and not by the fact being
absent. Without this rule fixture 24 would assert "I unset something irrelevant" — the
check-that-could-not-run arriving inside the fixture meant to close it.

### 6.3 What the seam costs, stated rather than left to be noticed

`LANE7_FIXTURE_DIR` is a test seam in production code: a production run with it set silently
consults files instead of the API. Three things hold it down, and none of them is a gate.

- **The ledger records which mode produced it.** `MODE live` or `MODE fixture <dir>` is the first
  stdout line of every run, always. A fixture-mode ledger offered as evidence for R5b's closure
  is self-identifying, which is the property that matters, since R5b closes on a *published*
  ledger.
- **The production workflow does not set it** (§8), and the workflow is in the diff.
- **It is one seam, not several.** The manifest is not overridable (§2) and nothing else is.

The honest size of it: this is weaker than having no seam, and it is adopted because the
alternative is a harness that binds a port in CI — a flaky fixture inside the file whose only job
is catching a fail-open. If a cleaner separation is later found, the repair is an amendment to
this document.

---

## 7. Git trailers after the block — answers item 2

**Permitted, narrowly.** After the attestation block, and before end of message, only these may
appear:

1. blank lines;
2. a line consisting solely of three or more `-` characters (GitHub's squash-UI separator);
3. **trailer lines** matching `^[A-Za-z][A-Za-z0-9-]*:[ \t]`.

Anything else is `L7-NOT-TRAILING`.

**Why lenient, and why this narrowly.** Forge checked `b40201b`'s real message and it ends
`---------` then two `Co-authored-by:` lines, because GitHub's squash UI appends trailers *after*
whatever body the merger types. Read strictly, the first genuinely correct attested merge made
through the web UI would carry `L7-NOT-TRAILING`, and a detector that fires on the behaviour it
exists to bless gets switched off. Read loosely — "anything may follow" — the block stops being
the end of the message and a second, contradictory block can be appended below it. The
three-shape allowance is the narrowest rule that admits the real merge shape.

**A 29th fixture is in scope and is accepted** (Scribe's candidate): a clean block followed by
`---------` and two `Co-authored-by:` trailers, expecting `OK`. An untested leniency is a hole in
the detector, not in the harness.

**Consequence for the harness, which Scribe correctly flagged as reaching five rows.** Fixtures
1, 2, 14, 24, 25 and 26 carry well-formed blocks and must append **no** trailers, so they assert
the block-at-end case. Fixture 29 is the only row that appends trailers. Both readings are then
asserted rather than one being assumed.

---

## 8. Who runs the harness — answers items 7 and 9

### 8.1 Item 7 — the class of `ci/lane7-attest-test.sh` is NEUTRAL, and that is already decided

ADR-0005 §4.4 and §6.5 both decide it on `docs#33`: the detector, its manifest and its harness
are *"all three **NEUTRAL** by §4.2"*. §4.4 considered naming them in GATE and **rejected it on
the record**, for the reason Scribe and Assay independently rederived from `cmd_partition()`'s
two refusal conditions — *"it does not close the hole it would be adopted for: script and harness
would both be GATE, so they still move together."* Their analysis is right, it is confirmed at
the level that matters, and it **confirms the rejection rather than overturning it**. Item 7 is
answered from the existing record and no amendment is needed to answer it.

What that means for Scribe's pull request, stated because item 7 asked what the PR may contain:

- NEUTRAL may accompany anything, so the gate will permit whatever is on it. **The constraint is
  this spec and §2.1, not the gate** — §4.4 already says so in its own voice, and pretending
  otherwise is the thing §4.4's three honest statements exist to prevent.
- **The lane-2 pull request contains `ci/lane7-attest-test.sh` and its fixture data, and nothing
  else.** No `ci/lane7-attest.sh`, no `ci/lane7-attest-floors.txt`, no workflow.

The structural repair both reviewers are circling — a rule of the shape *"`ci/X.sh` and
`ci/X-test.sh` may not appear in the same diff"* — is a **new rule shape** rather than a
reclassification, and it closes the existing `lane-gate` pair as well as this one. It is not in
scope here and it is not settled by this document. It belongs to **VUL-68**, owner Atlas, which
already holds the merged-code finding that `ci/lane-gate.sh`, `ci/lane-gate-test.sh` and
`.github/workflows/lane-gate.yml` are one GATE diff.

### 8.2 Item 9 — a new workflow, two jobs, neither of them required yet

Assay is right that nothing on items 1–8 created a runner, and right that it is the same question
as item 7. **A new file, `.github/workflows/lane7-attest.yml` in `platform`** — NEUTRAL by §4.2
("other workflows"), so it does not drag the harness into a GATE diff, which adding a job to
`lane-gate.yml` would.

| Job | Trigger | Runs |
|---|---|---|
| `attest-self-test` | `pull_request` to `main` | `ci/lane7-attest-test.sh ci/lane7-attest.sh` — mirrors `gate-self-test` exactly |
| `attest-history` | `schedule`, and `workflow_dispatch` | `ci/lane7-attest.sh run`, with both §6.2 credentials supplied |

**Forge adds the file, on the lane-3 pull request — not Scribe.** Two reasons. A workflow
invoking a script that does not exist is a job that is red on an unrelated pull request from the
moment it lands. And the file carries the credential wiring, which is implementation. So the
dependency edge is: **Scribe's PR creates the harness; Forge's PR creates the detector, the
manifest and the workflow that runs the harness.** VUL-50's acceptance statement now names the
workflow.

**Neither job is wired as a required status check on landing, and that is a choice rather than an
omission.** Making `attest-self-test` required fires ADR-0005 **R8** — the moment a NEUTRAL
`ci/**` script becomes a required check, §4.4's premise that it gates nothing is void and §4.2's
GATE row must be amended in the same change. That amendment is not this document's to make and it
is entangled with VUL-68. So, plainly: between Forge's PR landing and that decision,
`attest-self-test` **runs and reports on every pull request but does not block one**. That is
weaker than `gate-self-test` and it is recorded as weaker. It is still the difference between the
40 fixtures being asserted twice ever — once by Crucible confirming red, once by Forge reaching
green — and being asserted on every pull request forever, which was Assay's actual point.

---

## 9. The fixture table — 40 rows

**Rows 1–28 are unchanged in number and meaning** from VUL-56 as filed, so Scribe's
one-`check`-per-row-in-table-order commitment survives. Rows 29–40 are added, each for a reason
given below the table.

**The baseline every row varies from, so that each row is a single-variable experiment.** Unless
a row says otherwise: `shape: merge`; a well-formed six-key block with `Lane-7-Gate: PASS`,
`Lane-7-Ledger: PASS`, Assay and Warren both `APPROVE`, both `covers` equal to `Lane-7-Head`
equal to the second parent; `tree-has-gate-workflow: no`; `touches-crates: no`; no trailers after
the block; `LANE7_FIXTURE_DIR` set; both §6.2 credentials set to a dummy value; and fixture files
present making every lookup succeed and every §6.5 class pass. The expectation column is the
**complete** set of `L7-*` lines for that commit — a row expecting `OK` expects no `L7-*` line at
all.

| # | Fixture | Expect |
|---|---|---|
| 1 | The baseline | `OK` |
| 2 | Squash; `Lane-7-Gate: n/a (no workflow on docs)`, `Lane-7-Ledger: n/a (no §25 identifier in scope)`; `tree-has-gate-workflow: no` | `OK` |
| 3 | No `Lane-7-*` line at all | `L7-MISSING` |
| 4 | `Lane-7-Merged-By` absent | `L7-KEY` |
| 5 | `Lane-7-Ledger` absent | `L7-KEY` |
| 6 | True merge, `Lane-7-Head` ≠ second parent | `L7-HEAD` |
| 7 | Squash, `Lane-7-Head` ≠ `pull-head/<n>` | `L7-HEAD` |
| 8 | Squash, `pull-head/<n>` absent | `L7-HEAD-UNRESOLVABLE` |
| 9 | One parent, `pulls/<sha>` = `none` | `L7-NOT-PR` |
| 10 | `Lane-7-Gate: n/a` with no parenthesised reason | `L7-GATE-BARE` |
| 11 | `Lane-7-Gate: n/a (no workflow on platform)`, `tree-has-gate-workflow: yes` | `L7-GATE-VACUOUS` |
| 12 | `Lane-7-Gate: FAIL` | `L7-GATE-VALUE` |
| 13 | `Lane-7-Ledger: n/a` with no reason | `L7-LEDGER-BARE` |
| 14 | `Lane-7-Ledger: n/a (no §25 identifier in scope)`, `touches-crates: yes` | `L7-LEDGER-VACUOUS`, **exit 0** |
| 15 | `Lane-7-Verdict-1` only | `L7-VERDICT-COUNT` |
| 16 | Three `Lane-7-Verdict-*` lines | `L7-VERDICT-COUNT` |
| 17 | Two verdicts, both naming `Assay` | `L7-VERDICT-DUP` |
| 18 | A verdict naming `Atlas` | `L7-VERDICT-WHO` |
| 19 | A verdict reading `REQUEST CHANGES` | `L7-VERDICT-DISP` |
| 20 | A verdict reading `APPROVED` | `L7-VERDICT-DISP` |
| 21 | A verdict citing no `VUL-<n>` | `L7-VERDICT-ISSUE` |
| 22 | A verdict whose `covers` ≠ `Lane-7-Head`. **Head `ff1e2be`, covered `5168c5c`** — §6.4's shape | `L7-VERDICT-COVERS` |
| 23 | `Lane-7-Merged-By: CEO` | `L7-MERGED-BY` |
| 24 | Baseline block, `LANE7_PAPERCLIP_TOKEN` **unset**, `verdicts/` file still complete | `L7-VERDICT-UNCHECKED`, **exit 1** |
| 25 | Well-formed block followed by two paragraphs of prose | `L7-NOT-TRAILING` |
| 26 | `Lane-7-Merged-By` absent **and** a verdict whose `covers` ≠ `Lane-7-Head` | **both** `L7-KEY` and `L7-VERDICT-COVERS` |
| 27 | `repo vulcanflow/nope`, manifest has no such row | `L7-NO-FLOOR`, **no `RANGE` line**, exit 1 |
| 28 | `repo vulcanflow/infra`, manifest floor = that repo's `main` | `RANGE vulcanflow/infra 0`, no `L7-*`, **exit 0** |
| 29 | Baseline block, then `---------` and two `Co-authored-by:` trailers | `OK` |
| 30 | `Lane-7-Head` absent | `L7-KEY` **and no `L7-VERDICT-*` line** |
| 31 | `Lane-7-Gate` absent | `L7-KEY` |
| 32 | Zero `Lane-7-Verdict-*` lines, the other five keys present | `L7-VERDICT-COUNT` |
| 33 | Token set, `verdicts/VUL-<n>` **absent** | `L7-VERDICT-ISSUE-MISSING` |
| 34 | Token set, `verdicts/VUL-<n>` present and **empty** | `L7-VERDICT-NOT-FOUND` |
| 35 | Token set, `verdicts/VUL-<n>` records `APPROVE` covering the head, by `Kiln` | `L7-VERDICT-MISATTRIBUTED` |
| 36 | Token set, `verdicts/VUL-<n>` records `Assay REQUEST_CHANGES <head>` **and** `Warren APPROVE <other-sha>` | `L7-VERDICT-NOT-FOUND` |
| 37 | A verdict whose `covers` value is `deadbeefzz` | `L7-VERDICT-COVERS` |
| 38 | `Lane-7-Gate: PASS`, `check-runs/<head>` records `lane-partition failure` | `L7-GATE-UNCONFIRMED` |
| 39 | `Lane-7-Gate: PASS`, `check-runs/<head>` **absent** | `L7-GATE-UNCHECKED`, **exit 1** |
| 40 | `pulls/<sha>` **absent** | `L7-PR-UNCHECKED`, **and no `L7-NOT-PR`, `L7-HEAD` or `L7-HEAD-UNRESOLVABLE`** |

**Why each addition exists.**

- **29** — the only row asserting §7's leniency. Without it the lenient rule is untested and
  the real-world merge shape is unblessed.
- **30, 31, 32** — Scribe walked §6.3's six keys and found only two had an absence row. **30 is
  the one that earns its place on evidence**: `Lane-7-Head` absent is the key whose absence
  produced the live false pass, and the row asserts the §4.2 rule-2 consequence (no lookup on an
  empty sha) as well as the code. 31 completes the four single-valued keys. 32 pins that zero
  verdict lines is `L7-VERDICT-COUNT` and not `L7-KEY` — the verdict lines are **counted**,
  the four single-valued keys are **keyed**, and that boundary is exactly the thing two authors
  would otherwise guess differently.
- **33, 34, 35, 36** — the four lookup outcomes of §4.1. **36 is the repair for B1**: the
  disposition word and a sha co-occur in the issue, and no single record carries both with
  `APPROVE`, so the correct answer is `L7-VERDICT-NOT-FOUND`. A detector that passes 1–35 and
  fails 36 is the detector that shipped the defect.
- **37** — a malformed `covers` value, so §4's "absent, malformed, or mismatched" is asserted on
  all three limbs rather than one.
- **38, 39** — §6.5 class 8 had **no row in the table as filed**. Class 8 is the one §6.5 added
  specifically so the output would not read as though `PASS` were confirmed when nothing had
  confirmed it; leaving it unasserted would reproduce that defect in the harness.
- **40** — §6.5's *"could not resolve is not no pull request"* rule. The row asserts the
  **absence** of `L7-NOT-PR`, because manufacturing the most serious finding in the set out of a
  lookup failure is §6.2 corollary 4 pointed the other way, and only a negative assertion catches
  it.

### 9.1 One case left uncovered, on the record

Fixtures 2 and 11 both carry a repository name inside `n/a (no workflow on <repo>)`, and
`L7-GATE-VACUOUS` keys on the commit's tree rather than on whether that name matches the
repository being scanned. **No code is enumerated for a reason naming the wrong repository, and
the harness asserts nothing about it.** Scribe raised this and declined to invent a code, which
was correct. It stays uncovered: the reason string is prose for a human, and a detector that
parsed it would be asserting on prose, which §3 rules out for the harness and should rule out
for the detector too.

---

## 10. Acceptance

**`ci/lane7-attest-test.sh` asserts one verdict per fixture for all 40 fixtures in §9, with the
finding codes as written, and prints an `N passed, M failed` line in `ci/lane-gate-test.sh`'s
format.** That is the whole of VUL-56.

**`ci/lane7-attest.sh`, with its manifest at `ci/lane7-attest-floors.txt`, its workflow at
`.github/workflows/lane7-attest.yml`, and both §6.2 credentials supplied, passes all 40 fixtures
and reports over the four §6.5 ranges exactly one finding — `L7-MISSING vulcanflow/docs
b40201b…` — and `OK` or an empty `RANGE` for every other commit in scope.** That is the whole of
VUL-50.

**§25: none, and that is a finding rather than an omission.** Per ADR-0005 §4.4 — §25 is a
traceability matrix over product requirements and none of its 45 identifiers names the delivery
pipeline's own tooling. The lane-4 ledger line is `n/a (no §25 identifier in scope)` under §6.3
condition 2, and the two sentences above are the whole of what these items are judged against.
Do not invent an identifier.

### 10.1 The credential VUL-50's acceptance depends on — a blocker, not a caveat

Forge is right that fixture 24 and VUL-50's acceptance sentence are only both satisfiable if the
scheduled run carries a credential. With `LANE7_PAPERCLIP_TOKEN` absent, every clean commit in
range reports `L7-VERDICT-UNCHECKED` and the run exits `1` — **correctly**, by fixture 24 — and
the acceptance sentence above cannot be met. So:

- `attest-history` **must** supply both `LANE7_GITHUB_TOKEN` (the workflow's own
  `GITHUB_TOKEN` suffices; it needs public read only) and `LANE7_PAPERCLIP_TOKEN` (a repository
  secret on `vulcanflow/platform`).
- **Provisioning that secret is neither Atlas's nor Forge's to do.** It is a blocking dependency
  of VUL-50's acceptance with an owner outside both lanes, and it is filed as such rather than
  left as prose here.
- A run that exits `1` with only `UNCHECKED` lines is **not a detector defect**. The ledger says
  which mode and which checks did not run, and that is the degraded-but-honest outcome §6.5
  requires.

---

## 11. What this document does not decide

Named so that none of it is read as settled here.

- **The `ci/X.sh` / `ci/X-test.sh` same-diff rule** and the GATE-is-one-diff finding on
  `ci/lane-gate.sh`, `ci/lane-gate-test.sh` and `.github/workflows/lane-gate.yml`. **VUL-68**,
  owner Atlas. §8.1 explains why it is not in scope here.
- **Whether `attest-self-test` becomes a required status check.** ADR-0005 **R8**, entangled with
  VUL-68. §8.2 states the consequence of it not being one.
- **Whether `main` is corrected forward or reverted** for `b40201b`. ADR-0005 §6.4 consequence 2,
  owner CEO.
- **Anything ADR-0005 §6.5 fixed** — the four floors, the eight finding classes, first-parent
  only, the subsumption rule, and the three things the detector cannot assert. This document
  implements them; it does not revisit them.

## 12. Revisit triggers

- **A fifth repository gains a protected default branch.** Its floor is an ADR-0005 §6.5
  amendment and its manifest row follows; `L7-NO-FLOOR` is what makes the omission visible rather
  than silent.
- **`attest-self-test` is wired as a required check.** R8 fires; §4.2's GATE row and §8.2 of this
  document are both amended in that change.
- **A §6.3 defect is found inside a range a published run reported clean.** That is a second R5b
  event, and it means a finding class or a fixture is missing — re-open §4 and §9 together.
- **`LANE7_FIXTURE_DIR` appears in a production invocation.** The seam has leaked; §6.3's three
  mitigations have failed and the separation is re-designed rather than patched.
- **The attestation block format changes.** §4's register is keyed to §6.3 condition 4's six
  keys; a seventh key or a renamed one invalidates rows 3–5 and 30–32 and the harness is rewritten
  from the amended table by its original author, on a test-only pull request.
