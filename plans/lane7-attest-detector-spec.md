# Lane-1 spec — the lane-7 attestation detector

**Issue:** VUL-48 (lane 1, Atlas). **Lane 2:** VUL-56, Scribe. **Lane 3:** VUL-50, Forge.
**Status:** authoritative on merge. Until then it is a proposal and the two implementers work
from it at the risk named in §9.
**Revision 5** — the lane-5.5 pre-flight on `ac27fa6` (VUL-143), two findings, both accepted and
both the same species: a lookup whose **key** this document named two different ways. (1) **Class
8's required-check inventory had no source**, so the *missing* limb — a declared applicable job with
no check run of that name — was unimplementable and unexercised; the seam gains
**`expected-checks/<sha>`** (§6.1) and the table gains **fixture 48** (§9.6). ADR-0005 §6.5 class 8
now scopes that inventory to the workflow triggers applicable to the merged commit, and this
document follows it. (2) **The verdict-contents lookup keys on `Lane-7-Head`, not on the verdict
line's own `covers` sha** — §4's register always said so and §4.3's input list did not, which left
rows 22, 26 and 37 with two readings and different fixture *contents* under each. The key is stated,
and **fixture 37 leaves the suppression table**: with `Lane-7-Head` as the key, a malformed `covers`
does not deprive the lookup of one. **No row's expected set moves, including 37's.** The table is
**48 rows**.

**Revision 4** — Assay's lane-6 review of revision 2, adjudicated. One blocking finding survives
revision 3 and is accepted: **`Lane-7-Ledger` had no value check**, so a block typing the one
word §6.3 condition 2 forbids was published as clean. The repair is a **27th code**,
`L7-LEDGER-VALUE` (**§4**), and **fixture 47** (§9.4). Assay's second blocking finding was
already closed by revision 3 — see §9.3 — so its proposed numbering is superseded and the new row
is **47, not 45**. Three advisories accepted (**§4.3** rule B's converse, **§8.2** reworded,
VUL-56's description corrected) and one declined on the record with a revisit trigger
(**§9.5**). The table is **47 rows** before revision 5, **48** after it.

**Revision 3** — Forge's two lane-1 items and one phrasing tension, raised on VUL-56 and
adjudicated here: rule A reaches a verdict line's own fields (**§4.3**, transcription — no
fixture moves), an absent credential is a finding and never exit 2 (**§6.2**), and
`LANE7_GITHUB_TOKEN` gains the rows it never had (**fixtures 45 and 46**, §9.3).

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

**Set, not multiset — answers Scribe's N5, and it is a rule about the detector before it is a
rule about the harness.** The comparison is a set in both directions: the harness asserts the
set of stdout lines, and **the detector emits each `(code, repo, sha)` triple at most once per
commit.** A code that two §6.5 classes could both reach — `L7-PR-UNCHECKED` arises from classes 2
and 3, `L7-KEY` from any of four keys — is **one finding and one line**, not one per class and
not one per key. So "`L7-PR-UNCHECKED` for classes 2 and 3" in VUL-56's fixture 31 describes
*why* the code fires, never *how many times*.

Stating it on the emitting side rather than only the comparing side matters: if it were only a
harness convention, a detector that printed the same finding twice would still be green, and the
duplicate would reach the published ledger R5b closes on. One triple, one line, and the harness
compares sets because that is then the whole truth rather than a tolerance.

**`RANGE` is the answer to item 6, and it is better than a finding code for the empty range.**
It is emitted for every repository rather than only the empty ones, so the ledger states its
own scope; it carries the count, so an unexpectedly *short* range is visible too; it is not a
finding, so it cannot touch the exit code; and it is a token rather than a sentence, so the
empty-range wording can be rewritten without going red. `RANGE vulcanflow/infra 0` is the whole
of fixture 28's positive assertion.

---

## 4. The finding-code register — answers items 4 and 8

Twenty-seven codes. **The code is the contract**; the prose after ` -- ` is not. The §6.5 column
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
| `L7-GATE-UNCONFIRMED` | 8 | `Lane-7-Gate: PASS` is attested and the check-run lookup **ran** without confirming it — any required check failing, still running, or **absent, including the commit having no check runs at all**. Formerly proposed as `L7-GATE-UNTRUE`; renamed, see §9.1. |
| `L7-GATE-UNCHECKED` | 8 | The check-run lookup could not run, so an attested `PASS` was not confirmed. |
| `L7-LEDGER-BARE` | 5 | `Lane-7-Ledger: n/a` with no parenthesised reason. |
| `L7-LEDGER-VACUOUS` | 5 | `Lane-7-Ledger: n/a` on a commit touching `crates/**`. **The one advisory code** (§5). |
| `L7-LEDGER-VALUE` | 5 | `Lane-7-Ledger` is neither `PASS` nor an `n/a` form — e.g. `FAIL`, or `MISSING`. Added by revision 4; see §4.4. |
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

### 4.3 Second-order suppression — answers Forge's item 9 and Scribe's item 10

ADR-0005 §6.5 defines subsumption for **class 1 only**: an absent block suppresses classes 2 and
4–8 and leaves class 3 alive. Both implementers found the same gap in that, from opposite sides,
and they are right that it is a spec question rather than either lane's judgment call —
**fixture 26 establishes that the detector reports a ledger and does not stop at the first
finding, so neither author may close the gap by short-circuiting.** Three rules, in the order the
detector applies them.

**Rule A — a check does not run on a value it does not have.** A finding is emitted only by a
check whose every input is present and well-formed. The consequence, stated per key so that no
expected set has to be inferred:

| Absent key | Fires | And suppresses |
|---|---|---|
| `Lane-7-Head` | `L7-KEY` | **everything that reads `Lane-7-Head`** — `L7-HEAD`, `L7-HEAD-UNRESOLVABLE`, `L7-VERDICT-COVERS`, all four `L7-VERDICT-*` lookup codes, and class 8's `L7-GATE-UNCONFIRMED` / `L7-GATE-UNCHECKED` |
| `Lane-7-Gate` | `L7-KEY` | the four class-4 codes and both class-8 codes |
| `Lane-7-Ledger` | `L7-KEY` | all **three** class-5 codes (`L7-LEDGER-BARE`, `L7-LEDGER-VACUOUS`, `L7-LEDGER-VALUE` — three since revision 4, §4.4) |
| `Lane-7-Merged-By` | `L7-KEY` | nothing — no other check reads it |

So **fixture 30 is `L7-KEY` alone** and **fixture 31 is `L7-KEY` alone**, and Forge's naive
reading — one missing `Lane-7-Head` line producing `L7-KEY` *and* `L7-HEAD` *and*
`L7-VERDICT-COVERS`, two of the three vacuous — is wrong by this rule rather than by taste. This
is §4.2's rule 2 generalised: that rule closed the empty-sha *lookup*, and the live false pass
Forge reproduced came from exactly the vacuity this rule forbids. Rule A does not conflict with
fixture 26, because fixture 26's two findings read two *different* keys, both of which are
present.

**Rule A also reaches the fields *inside* a verdict line — revision 3, and it was already the
ruling.** Forge is right that rule A's table enumerates the four single-valued keys and stops,
while the verdict-contents lookup reads two further inputs that live inside a
`Lane-7-Verdict-*` line: its **disposition** and — revision 4 said — its **`covers` sha**. The
ruling stands for the **disposition** and **fixture 19 settles it**: under the reading where the
lookup runs anyway, the harness must write `Assay REQUEST CHANGES <Lane-7-Head>` into
`verdicts/VUL-<n>`, that record is not an `APPROVE`, `L7-VERDICT-NOT-FOUND` joins the expected set,
and the row as written goes red. **The `covers` sha is struck from that list by revision 5** — the
lookup does not read it, so rule A has nothing to reach there, and the argument revision 3 ran from
fixture 37 is withdrawn below. So the table below is a transcription of what §9 already asserts, not
a new rule, and **it changes no row and no expected set.**

The verdict-contents lookup's inputs are the cited issue, the reviewer, the disposition `APPROVE`,
and **`Lane-7-Head`** — *not* the verdict line's own `covers` sha. §4's register is where that was
always stated (`L7-VERDICT-NOT-FOUND`: the issue "records **no `APPROVE` covering `Lane-7-Head`**
by the reviewer named"), and revision 4's input list here said `covers` instead, which is the one
substantive thing revision 5 repairs. The two are the same value on a well-formed block and differ
on exactly the rows that exist to make them differ, so the ambiguity was invisible in the register
and decisive in the harness: it decides what the fixture **writes**. A class-6 code that fires
because one of those four inputs is not in the shape the lookup needs **suppresses the lookup**:

| Firing code | Why the lookup has no well-formed input | Suppresses |
|---|---|---|
| `L7-VERDICT-COUNT` (15, 16, 32) | there is no pair of verdicts to look up | all four `L7-VERDICT-*` lookup codes |
| `L7-VERDICT-DUP` (17) | two records of one reviewer is not the pair the lookup asks about | all four |
| `L7-VERDICT-WHO` (18) | the reviewer named is not one the lookup can ask about | all four |
| `L7-VERDICT-DISP` (19, 20) | `APPROVE` is the only disposition the detector can confirm; anything else leaves the comparison without a left-hand side | all four |
| `L7-VERDICT-ISSUE` (21) | no `VUL-<n>` is a lookup with no key — this one already fell out of the general clause | all four |

**`L7-VERDICT-COVERS` is not in that table, and revision 4 had it there.** Revision 4 listed
fixture 37's malformed `covers` as suppressing all four lookup codes, on the reason that
*"`deadbeefzz` is not a sha, so nothing can be keyed on it."* That reason is false once the key is
`Lane-7-Head`: a malformed `covers` is a malformed **value being checked**, not a malformed key, and
`Lane-7-Head` is well-formed on both 22 and 37. The lookup runs on both.

**Fixtures 22 and 37 therefore share one mechanism, and revision 4 said they were the pair that
proved two.** Both fire `L7-VERDICT-COVERS` alone, and both do it through class 6's own check rather
than through rule A:

- **22** — `covers` is a *well-formed* sha that is simply **wrong** (`5168c5c` against head
  `ff1e2be`): it "does not match `Lane-7-Head`".
- **37** — `covers` is **malformed**: it "is not 7–40 lowercase hex".

Those are the two limbs of `L7-VERDICT-COVERS` in §4's register, and keeping both rows is still
worth it — an implementation that checks the match and forgets the format passes 22 and fails 37.
What the pair is **not** is a rule-A/rule-C boundary, and the claim that it was is withdrawn.

**The argument revision 3 made from fixture 37 goes with it.** That argument ran: under the reading
where the lookup runs anyway, the harness must write `Assay APPROVE deadbeefzz`, that record does
not cover the head, `L7-VERDICT-NOT-FOUND` joins the expected set, and row 37 goes red. Every step
is sound and the premise is not — the harness writes `Assay APPROVE <Lane-7-Head>`, because the
**key is the head**. So fixture 37 settles nothing about rule A's reach, and the conclusion rule A
reaches a verdict line's own fields survives on `L7-VERDICT-DISP` alone, which is where it is real:
a block whose disposition is not `APPROVE` leaves the comparison with no left-hand side no matter
what it is keyed on, and fixture 19 is unsatisfiable without that suppression. One of two arguments
for the same ruling was load-bearing; the other was keyed on the wrong field.

**Rule B — class 3 is never suppressed by anything except its own lookup failing.** §6.5 already
says class 1 does not suppress it. Nothing else does either: `L7-NOT-PR` is a statement about how
the commit reached `main`, which no key in the block can make true or false. The one thing that
suppresses it is `L7-PR-UNCHECKED` — the lookup that would have decided it did not run (§4,
fixture 40).

**Rule B's converse — class 3 firing leaves class 2 with no referent — revision 4, and it was
already forced.** Assay's advisory A1 is correct that fixtures 9, 42 and 43 had their class-2
expectation *inferred* rather than stated, which is the one thing §4.3 promises not to leave to
the reader. Stating it:

`L7-HEAD` compares `Lane-7-Head` against **the commit actually merged** — the second parent on a
merge, `pull-head/<n>` on a squash. On a commit that reached `main` by neither, which is the
case `L7-NOT-PR` names, **that referent does not exist**, so by rule A's lead sentence the class-2
comparison does not run: `L7-HEAD` is silent whatever the block says, and
`L7-HEAD-UNRESOLVABLE` is unreachable by its own wording in §4, which requires that the pull
request *was* identified. So **fixtures 9 and 42 are `L7-NOT-PR` alone** and **43 is `L7-MISSING`
and `L7-NOT-PR`**, exactly as §9 states them.

This also settles the fixture *data* on those rows, which is the half that would otherwise be
guessed: §9.0's baseline defines `Lane-7-Head` as the second parent, and a one-parent commit has
none. On a row with no referent, **`Lane-7-Head` may carry any well-formed sha and is not
load-bearing** — the `verdicts/` and `check-runs/` files are keyed on whatever it names, per
§9.0's last clause, so they still agree and still report nothing.

Note the asymmetry against rule C, because it is the distinction an implementation gets wrong:
rule C is about a `Lane-7-Head` that is **present and wrong** against a referent that **exists**,
so the comparison runs and fails. This converse is about a referent that does **not exist**, so
there is no comparison to run. Fixture 6 and fixture 9 are the pair.

**Rule C — a wrong `Lane-7-Head` is present, so it is used.** Rule A turns on *absence*, not on
*incorrectness*. Fixtures 6, 7, 22 and 26 carry a `Lane-7-Head` that is well-formed and wrong, so
every check that reads it **does run, against the sha the block names.** This answers Scribe's
N3, and it answers it in the direction that keeps the detector honest: suppressing class 8
whenever class 2 fires would mean a commit could attest a false `PASS` *and* a false `Head` and
have the first go unreported, which is the §6.4 shape twice over.

N3's sharpest limb — fixture 22's `Head` is `ff1e2be`, a `docs` sha that cannot exist in a
`mktemp -d` fixture repository — **dissolves in the seam rather than needing a rule.** Under
`LANE7_FIXTURE_DIR` (§6.1) the check-run and pull-request facts are read from files keyed by sha,
not from git, so the harness writes `check-runs/ff1e2be…` and the lookup succeeds against a sha
no object store holds. That is the §9 baseline doing its job, and it is why the baseline is
phrased as *the lookups the detector performs* rather than *the lookups on the real head*.

**What this does not add.** No finding class, and no code. Rules A–C and rule B's converse are
an application order over §6.5's eight classes; §6.5's own subsumption rule is untouched and
still wins where they meet.

### 4.4 `L7-LEDGER-VALUE` — the asymmetry that published a `FAIL` as clean

**Assay's blocking finding B1 is accepted in full, and the defect is mine.** Class 4 yields three
codes — `L7-GATE-BARE`, `L7-GATE-VACUOUS` and `L7-GATE-VALUE`, the last for a value that is
*"neither `PASS` nor an `n/a` form — e.g. `FAIL`"*. ADR-0005 §6.5 says class 5 follows **the same
rule**. Until revision 4 it yielded **two**: `L7-LEDGER-BARE` and `L7-LEDGER-VACUOUS`. There was
no `L7-LEDGER-VALUE`.

Trace the baseline with `Lane-7-Ledger: FAIL` through the register as it stood:

- `L7-KEY` does not fire — the key is present.
- `L7-LEDGER-BARE` does not fire — `FAIL` is not `n/a`.
- `L7-LEDGER-VACUOUS` does not fire — the commit touches no `crates/**`, and it is advisory in
  any case, so even firing it would leave exit `0`.
- Nothing else reads the key (§4.3 rule A, `Lane-7-Merged-By` row's reasoning applied one key
  over).

The commit prints `OK` and the run exits `0`. **§6.3 condition 2 is literally "the ledger has no
`FAIL` and no `MISSING`"** — so a merge that types the one word the condition forbids, into the
field that names it, was published as clean by the instrument built to publish exactly that. §3
makes stdout the whole ledger and §4 makes the code the contract, so the omission is not
recoverable by a reader: there is no line to read. That is §6.4's shape occurring inside §6.4's
own repair, which is the second time this document has had to record that (§4.1 is the first).

**The repair, and it is the minimum one.** A 27th code, `L7-LEDGER-VALUE`, worded as
`L7-GATE-VALUE`'s parallel and non-advisory, so it exits `1` (§5's advisory set is unchanged and
is still exactly `{L7-LEDGER-VACUOUS}`). Rule A's class-5 suppression count goes from two to
three. **Fixture 47** asserts it, parallel to fixture 12 field for field.

**One code and not two, deliberately.** §6.3 condition 2 names both `FAIL` and `MISSING`, and
both are the same branch: the value is not `PASS` and not an `n/a` form. §9.4's fixture uses
`FAIL` because it is the value that makes the published-as-clean trace concrete and because it
makes 47 and 12 the same experiment twice; a detector that handles `FAIL` and mishandles
`MISSING` is not a reachable implementation of the rule as written. Enumerating a second code
would also break §4.1's own test — four codes exist there because four *repairs* differ, and
`FAIL` and `MISSING` have one repair.

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
| `check-runs/<sha>` | one line per check run **observed** on that sha: `<name> <conclusion>`; present and empty = **the lookup ran and the commit has no check runs at all** | the lookup **failed** → `L7-GATE-UNCHECKED` |
| `expected-checks/<sha>` | one line per **expected** job name, no conclusion; present and empty = the tree declares no applicable job | **not** a lookup failure — defaults from `tree-has-gate-workflow` (§6.1.1) |
| `verdicts/<VUL-n>` | one line per recorded verdict: `<reviewer> <disposition> <covered-sha>`; present and empty = the issue exists and records none | the issue **does not exist** → `L7-VERDICT-ISSUE-MISSING` |

`<disposition>` is `APPROVE` or `REQUEST_CHANGES` (underscored, because the field is
whitespace-separated).

**The distinction between an absent file and a present-but-empty file is load-bearing in two
places, not one.** It is the whole of the difference between *the lookup did not happen* and *the
lookup happened and the answer was nothing* — which is the §6.2-corollary-4 distinction this
detector exists to keep, expressed in the seam:

| | `verdicts/<VUL-n>` | `check-runs/<sha>` |
|---|---|---|
| **absent** — lookup failed | `L7-VERDICT-ISSUE-MISSING` (fixture 33) | `L7-GATE-UNCHECKED` (fixture 39) |
| **present, empty** — ran, found nothing | `L7-VERDICT-NOT-FOUND` (fixture 34) | `L7-GATE-UNCONFIRMED` (fixture 44) |
| **present, non-matching** | `L7-VERDICT-NOT-FOUND` (fixture 36) | `L7-GATE-UNCONFIRMED` (fixture 38) |

Fixture 44 is the row this table forced into existence; it is VUL-56's fixture 33 and §9 had
dropped it. An attested `Lane-7-Gate: PASS` on a commit with **zero** check runs is not an
unreachable API — it is a required workflow that never ran, which is the more likely real defect
of the two and the one §6.5 class 8 names first.

### 6.1.1 `expected-checks/<sha>` — the inventory class 8 compares against

**Revision 4's seam could not express class 8's *missing* limb at all, and that is why this file
exists.** `check-runs/<sha>` was described as *"one line per required check"*, which conflates the
**observed** set with the **expected** one: a check run the commit does not have has no line to be
written on, so the one case §6.5 class 8 calls the *missing* limb — a declared applicable job with no
check run of that name on `Lane-7-Head` — was unrepresentable in a fixture and unasserted by any row.
A limb of a finding class that no fixture can construct is a limb no implementation has to get right.

**In production the inventory is read from the tree, not from the fixture dir and not from branch
protection.** ADR-0005 §6.5 class 8 decides this and this document only transcribes it: the expected
set is the jobs declared by the workflow files in the tree at `Lane-7-Head` **whose triggers apply to
that commit** — `on:` events with any `branches`, `paths` or `paths-ignore` filters evaluated against
the event that produced the commit. `required_status_checks.contexts` is rejected there as
present-tense and unretained, and this document adds no route to it.

**Under `LANE7_FIXTURE_DIR` the fixture supplies that set directly**, because the fixture repository
has no workflow files and resolving `on:` filters is not what these 48 rows are testing:

| State of `expected-checks/<sha>` | Means |
|---|---|
| **present, non-empty** | each line is one expected job name |
| **present, empty** | the tree declares no applicable job |
| **absent**, `tree-has-gate-workflow: no` | the same — no applicable job |
| **absent**, `tree-has-gate-workflow: yes` | the four job names of `.github/workflows/lane-gate.yml`: `lane-partition`, `test-erosion`, `inline-test-modules`, `gate-self-test` |

**Absence is a default rather than a lookup failure, and that is a deliberate exception to §6.1's
rule.** The other four files answer a question put to a remote service, so "no file" means "no
answer" and the detector must report `UNCHECKED`. The inventory is computed from the commit's own
tree, which the detector always has: there is no state in which it cannot be derived, so it gets no
`UNCHECKED` and no code. The default is keyed on `tree-has-gate-workflow` and not on nothing, because
that field already means *the tree declares `lane-gate.yml`* and the applicable jobs of that file are
those four — so **every one of rows 1–47 keeps its setup and its expected set verbatim**, which is
the test of whether this file was added correctly.

**Class 8's comparison is then two limbs over two sets, and only the first consults the inventory.**
When `Lane-7-Gate: PASS` is attested:

- **missing** — a name in `expected-checks/<sha>` with no line in `check-runs/<sha>`;
- **failing or still-running** — **any** line in `check-runs/<sha>` whose conclusion is not
  `success`, whether or not that name is expected.

Both are `L7-GATE-UNCONFIRMED`. The second limb is deliberately *not* scoped to the expected set: an
attested `PASS` over a check run that actually went red is false whichever inventory the job appears
in, and scoping it would let an unexpected red check excuse the claim. The inventory exists to make
**absence** detectable, which is the one thing the observed set cannot express — not to license
anything the observed set already contradicts. A check run that is `success` and unexpected is
reported by nothing: it is neither a weakening nor a claim the block makes.

This is why fixtures 38 and 44 keep their one code each and need no new field: **38** has a red
observed run and fires the second limb with an empty inventory, **44** has an empty observed set
against the four-name default and fires the first. They were already the two limbs; revision 4 just
had no vocabulary that distinguished them, which is also why §9.5's declining to split the code
survives revision 5 unchanged.

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

**"Required to be non-empty" is an obligation on the caller, not a precondition the detector
enforces — revision 3, and fixture 24 is authoritative.** Forge read the sentence above as the
grammar of a precondition, noticed that a precondition is **exit 2** by §5, and noticed that
exit 2 would make fixture 24 — which asserts a *finding* and **exit 1** — unsatisfiable. The
reading is a real ambiguity in the previous phrasing and the resolution is the one Forge was
going to implement:

- The rule binds **the fixture and the workflow**: §9.0's baseline sets both credentials, and
  `attest-history` supplies both (§10.1). A row that unsets one is varying its single variable.
- The **detector never treats an absent or empty credential as a usage error.** It degrades the
  classes that credential gates, reports the corresponding `*-UNCHECKED` code **per commit**, and
  exits `1`. Exit `2` stays reserved for *the detector did not complete* and must not be reachable
  by a missing token — a run that could not check and said so is a complete ledger, and §5's whole
  point is that the two outcomes are not interchangeable.
- `LANE7_GITHUB_TOKEN` absent is the same shape, and until revision 3 no row asserted it. It is
  now **fixture 45** and **fixture 46**.

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

### 6.4 Forge's `VF_ATTEST_VERDICT_CMD` — considered, and superseded by §6.1

Forge proposed a single command indirection: `VF_ATTEST_VERDICT_CMD`, invoked with a `VUL-<n>`,
exit 0 meaning the lookup ran and stdout carrying the record set. It is the same mechanism (3) and
it was proposed in a ratifiable form, so it gets an answer rather than silence. **`LANE7_FIXTURE_DIR`
is adopted instead, for two reasons that are about coverage rather than taste.**

- **It is one seam for three endpoints.** Scribe's N6 counted the real surface: `commits/{sha}/pulls`,
  `commits/{sha}/check-runs` and the Paperclip issue lookup, each needing both a content mode and a
  failure mode, across more than twenty rows. A verdict-only indirection covers one of the three, and
  the other two would need their own variables — three seams instead of one, which §6.3's third
  mitigation is specifically about not having.
- **`exit 0` plus empty stdout is one state, and the seam needs two.** Forge's table collapses *the
  issue does not exist* and *the issue exists and records nothing* into the same observation, and §4.1
  makes those two different findings — `L7-VERDICT-ISSUE-MISSING` and `L7-VERDICT-NOT-FOUND` — with
  two different owners and two different repairs. The absent-file / empty-file distinction in §6.1
  expresses both; a command's exit status and stdout cannot without adding a second channel.

**What is adopted from Forge's proposal, because it is the better half of it:** the requirement that
the run state which mode produced it. That is §3's `MODE live` / `MODE fixture <dir>` line, made
unconditional and made the *first* stdout line, so a fixture-mode ledger can never be offered as
evidence for R5b without saying so. Forge asked for it and it is in the contract.

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
as item 7. **A new file, `.github/workflows/lane7-attest.yml` in `platform`**, NEUTRAL by §4.2
("other workflows").

**That is not a separation claim, and revision 4 stops it reading as one.** Scribe raised it and
Assay filed it as advisory A3; both are right and the previous sentence here was wrong. §4.2
permits GATE and NEUTRAL in one diff, so `lane-partition` refuses **neither** arrangement — a job
in `lane-gate.yml` would make the runner move inside a GATE diff, and a NEUTRAL file of its own
lets the runner ride with anything at all, **including PROD**. Neither is a partition and the new
file is the weaker of the two on that axis. The decision stands on two other reasons, which do
not claim to be gates:

- **Coupling, honestly named.** `lane-gate.yml` is GATE. A job added to it puts the attestation
  runner's credential wiring inside the file that defines the gate, and puts every later edit to
  the runner inside a GATE-class diff. That is real coupling even though no check refuses it.
- **Blast radius.** `lane-gate.yml` carries all four required contexts. A malformed job added to
  that file can break the required checks for every pull request in `platform`; a job in a
  separate NEUTRAL workflow cannot. This is the reason that would survive even if §4.2 did
  partition them.

The separation both reviewers actually want is the `ci/X.sh` / `ci/X-test.sh` same-diff rule
shape, which is **VUL-68** (§8.1, §11) — not something this file's name can supply.

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
48 fixtures being asserted twice ever — once by Crucible confirming red, once by Forge reaching
green — and being asserted on every pull request forever, which was Assay's actual point.

---

## 9. The fixture table — 48 rows

**Rows 1–28 keep the numbers they had in VUL-56 as filed**, so Scribe's
one-`check`-per-row-in-table-order commitment survives. Rows 29–40 were added by this document's
first revision; **rows 41–44 are added by its second**, and three of the four are VUL-56 rows this
document had silently dropped — see the correction note after the table, which is a defect in
this spec rather than in either implementer's reading of it. **Rows 45 and 46 are added by its
third**, and they close the one credential the table did not assert; §9.3 says why they are two
rows and not one. **Row 47 is added by its fourth** — Assay's blocking finding B1, the
`Lane-7-Ledger` value check that did not exist (§4.4, §9.4). **Row 48 is added by its fifth** — the
lane-5.5 finding that class 8's *missing* limb had no inventory to compare against and so no fixture
(§6.1.1, §9.6).

**Rows 1–46 are unchanged by revision 4.** No expected set, no code and no exit code moves. The
only addition is 47; everything else revision 4 does is transcription (§4.3's rule B converse)
or prose repair (§8.2).

**Rows 1–47 are unchanged by revision 5, in setup as well as in expectation.** The only addition is
**48**, and §6.1.1's absent-file default is built the way it is specifically so that no earlier row
acquires a field — see §9.6. Everything else revision 5 does is in §4.3 and §6.1 and moves no row.

### 9.0 The field-default rule — answers Scribe's item 9 (N1, N2)

Scribe is right that the expected code *set* is undetermined for 30 of the rows without this, and
right to refuse to guess it. **One rule, and it is the paragraph below rather than a per-field
table, because the fields are not independent — `shape` decides which PR facts exist at all.**

**Unless a row says otherwise, every fixture is the baseline with exactly one variable changed:**

- `shape: merge` — two parents, `Lane-7-Head` equal to the second parent;
- a well-formed six-key block: `Lane-7-Gate: PASS`, `Lane-7-Ledger: PASS`, `Lane-7-Verdict-1`
  Assay and `Lane-7-Verdict-2` Warren, both `APPROVE`, both citing a `VUL-<n>`, both `covers`
  equal to `Lane-7-Head`, `Lane-7-Merged-By: Crucible`;
- the block is the end of the message — **no trailers** (§7);
- `tree-has-gate-workflow: no`; `touches-crates: no`;
- `LANE7_FIXTURE_DIR` set; **both** §6.2 credentials set to a dummy non-empty value;
- and **every lookup the detector actually performs succeeds, keyed on whatever value the block
  names** — so the `pulls/`, `pull-head/`, `check-runs/` and `verdicts/` files the row does not
  itself vary are present, and their contents agree with the block; **the verdict lookup's key is
  `Lane-7-Head`** (§4.3), so `verdicts/VUL-<n>` reads `<reviewer> APPROVE <Lane-7-Head>` even on the
  rows that carry a wrong or malformed `covers` — 22, 26 and 37;
- `expected-checks/<sha>` is **not written**, so §6.1.1's default applies and follows
  `tree-has-gate-workflow`;

That last clause is the one doing the work, and it is phrased as *the lookups the detector
performs* rather than *the lookups on the correct sha* on purpose. It is what makes rows 6, 7, 22
and 26 — which carry a deliberately **wrong** `Lane-7-Head` — single-variable rather than
double: the harness writes `check-runs/<the-wrong-head>` with a passing conclusion, so class 8
runs, succeeds, and reports nothing, and the row asserts class 2's finding alone (§4.3 rule C).
Scribe's N3 is answered there and not here.

`subject-suffix` has **no baseline value and is never load-bearing**, because §6.3 condition 4's
route to a pull request is the API and only the API. Rows 41 and 42 exist to assert that, and they
are the only two rows on which the suffix is mentioned at all.

The expectation column is the **complete** set of `L7-*` lines for that commit — a row expecting
`OK` expects no `L7-*` line at all — **together with the exit code.**

**The exit code is asserted on every row, not on five.** Scribe offered this and it is accepted:
it is derivable from §5 with no further ruling, since the advisory set is exactly
`{L7-LEDGER-VACUOUS}`. So every row expecting `OK` or expecting only `L7-LEDGER-VACUOUS` asserts
**exit 0**, every row expecting any other finding asserts **exit 1**, and no row in this table
asserts exit 2. The table still spells the code out on the rows where it is the surprising half
of the assertion; where it is silent, that rule supplies it rather than leaving it unasserted.

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
| 41 | Squash, `pulls/<sha>` = `30`, `pull-head/30` = the head, **subject carries no `(#n)` suffix** | `OK` |
| 42 | Squash, **subject reads `… (#99)`**, `pulls/<sha>` = `none` | `L7-NOT-PR` |
| 43 | One parent, `pulls/<sha>` = `none`, **and no `Lane-7-*` line at all** | **both `L7-MISSING` and `L7-NOT-PR`** |
| 44 | `Lane-7-Gate: PASS`, `check-runs/<head>` **present and empty**, `tree-has-gate-workflow: yes` | `L7-GATE-UNCONFIRMED` |
| 45 | Baseline block and **merge** shape, `LANE7_GITHUB_TOKEN` **unset**; `pulls/`, `pull-head/` and `check-runs/` all present and complete | **both** `L7-PR-UNCHECKED` and `L7-GATE-UNCHECKED`, **and no `L7-NOT-PR`, `L7-HEAD`, `L7-HEAD-UNRESOLVABLE` or `L7-VERDICT-*`**, exit 1 |
| 46 | **Squash**, `LANE7_GITHUB_TOKEN` **unset**; `pulls/`, `pull-head/` and `check-runs/` all present and complete | **exactly one** `L7-PR-UNCHECKED` and one `L7-GATE-UNCHECKED`, **and no `L7-HEAD-UNRESOLVABLE`, `L7-HEAD` or `L7-NOT-PR`**, exit 1 |
| 47 | `Lane-7-Ledger: FAIL` | `L7-LEDGER-VALUE`, **exit 1** |
| 48 | `Lane-7-Gate: PASS`, `tree-has-gate-workflow: yes`, `expected-checks/<head>` lists all four `lane-gate.yml` jobs, `check-runs/<head>` records **three** of them `success` and **omits `gate-self-test`** | `L7-GATE-UNCONFIRMED`, **exit 1** |

### 9.1 Correction — rows 41–44, and a defect in this document's first revision

**This document's 40-row table was not a superset of VUL-56's 34, and it was presented as one.**
*"Rows 1–28 are unchanged … rows 29–40 are added"* is true of the numbering and false of the
content: VUL-56's rows 29–34 were not carried forward, and four of their six cases survived only
by coincidence of having a near-equivalent among the new rows. Scribe read the amended issue
table and I read my own additions, and neither of us diffed the two. Found while answering
Scribe's N1 and N3; the finding is mine, against me.

What was lost, and where it is restored:

| VUL-56 row | Fate in the 40 | Restored as |
|---|---|---|
| 29 — squash, PR resolved by API, **suffix absent** → `OK` | **dropped** | **41** |
| 30 — **suffix lies**, `pr-number: none` → `L7-NOT-PR` | **dropped** | **42** |
| 31 — `pr-number: unreachable` → `L7-PR-UNCHECKED`, not `L7-NOT-PR` | survived as row **40** | — |
| 32 — `check-runs: fail` → class 8 | survived as row **38** | — |
| 33 — `check-runs: none`, tree has the workflow → class 8 | **dropped** | **44** |
| 34 — `check-runs: unreachable` → `L7-GATE-UNCHECKED` | survived as row **39** | — |

And one more, which is not in that table because it is a *meaning* change rather than a deletion:
**VUL-56's fixture 9 carried no attestation block**, and its stated expectation was *both*
`L7-MISSING` *and* `L7-NOT-PR` — the row ADR-0005 §6.5 added specifically to assert that class 3
survives class 1's subsumption. Row 9 of the 40 inherits the baseline, which includes a
well-formed block, so class 1 never fires and the row asserts `L7-NOT-PR` alone. **Both rows are
worth having** — row 9 is now class 3 in isolation, and **row 43 is VUL-56's fixture 9 as
written**, the subsumption-survival assertion. §6.5's rule is unassertable without it.

**Why the two suffix rows matter more than their count suggests.** 41 and 42 are the only rows in
the table that can fail an implementation which resolved the pull request by parsing `(#n)` out of
the subject — 41 goes red with a false `L7-NOT-PR` on a legitimate merge, 42 goes green on a
commit that reached `main` without one. Every other row in the table is indifferent to how the
number was obtained. Dropping them removed the only assertion of §6.3 condition 4's
API-is-the-only-route rule, which §7 item 10's typed merge message is the reason for.

**One code was also renamed and the rename was not called out.** VUL-56's rows 32–34 expect
`L7-GATE-UNTRUE`; §4's register calls it **`L7-GATE-UNCONFIRMED`**, and splits the unreachable
case out as `L7-GATE-UNCHECKED`. The register's names are the ones to implement: *untrue* asserts
the `PASS` is false, and a check run that is **still running** or **absent** does not establish
that — it establishes that nothing confirmed it, which is what the detector can honestly say.
`UNCHECKED` for the lookup failing then matches `L7-PR-UNCHECKED` and `L7-VERDICT-UNCHECKED`, so
one suffix means one thing across all three. **`L7-GATE-UNTRUE` is not a code**; it appears in no
row of §9 and the harness must not assert on it.

**Why each of rows 29–40 exists.**

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

### 9.2 One case left uncovered, on the record

Fixtures 2 and 11 both carry a repository name inside `n/a (no workflow on <repo>)`, and
`L7-GATE-VACUOUS` keys on the commit's tree rather than on whether that name matches the
repository being scanned. **No code is enumerated for a reason naming the wrong repository, and
the harness asserts nothing about it.** Scribe raised this and declined to invent a code, which
was correct. It stays uncovered: the reason string is prose for a human, and a detector that
parsed it would be asserting on prose, which §3 rules out for the harness and should rule out
for the detector too.

### 9.3 Rows 45 and 46 — the credential the table did not assert, and why it costs two rows

Forge found the gap and offered one row, declining to add it. **Accepted, and it takes two**, for
a reason the one-row version cannot reach.

The gap is real and it is exactly fixture 24's shape pointed at the other credential.
`LANE7_GITHUB_TOKEN` gates three lookups and **no row unset it**, so nothing in the table
distinguished *the credential was absent* from *the fact was absent* on classes 2, 3 and 8 —
which is §6.2 corollary 4's shape, and the shape §6.4 actually had. Rows 39 and 40 assert
`*-UNCHECKED` from an absent **file**; they say nothing about an absent **credential**.

**Why 45 alone is not enough.** Forge's row is merge-shaped, because §9.0's baseline is. On a
merge the head is the second parent — **a git fact needing no credential** — so `L7-HEAD` is
silent because the head is correct, and `L7-HEAD-UNRESOLVABLE` is unreachable on a merge at all.
Both negative clauses in row 45 are therefore **vacuously** satisfied, and an implementation that
got rule A wrong on `pull-head` would still pass it. The clause reads like an assertion and
constrains nothing. Leaving that on the record is worse than not having the row.

**46 is the non-vacuous half.** On a squash, class 2 resolves the head through `pulls` and then
`pull-head`, so an absent `LANE7_GITHUB_TOKEN` means the PR number never exists and rule A
suppresses `L7-HEAD-UNRESOLVABLE` — a suppression that *is* otherwise reachable there (fixture
8). 46 also asserts the **de-duplication** §3 states on the emitting side: classes 2 and 3 both
reach `L7-PR-UNCHECKED` on this commit and the ledger carries **one** line, which no other row in
the table forces.

Row 46 varies **two** fields against the baseline — shape and the credential — and says so, as
rows 2, 7, 8, 41 and 42 already do. 45 is kept as offered rather than folded into 46 because the
merge path and the squash path degrade differently and a row that passed for either reason would
not say which.

### 9.4 Row 47, and why it is 47 and not the 45 the review asked for

**Row 47 is Assay's B1 repair** (§4.4): the baseline with `Lane-7-Ledger: FAIL`, expecting
`L7-LEDGER-VALUE` and exit `1`. It is fixture 12 with one key changed, which is the point — 12
and 47 are the same experiment run against the two keys §6.5 says follow the same rule, and an
implementation that has class 4 right and class 5 wrong fails exactly one of them.

**The renumber is called out because this document has twice shipped a numbering defect and
neither was caught by the author.** Assay's review proposed the ledger-value row as **45** and
the `LANE7_GITHUB_TOKEN` row as **46**. It was written against head `213a749` — revision 2 — and
revision 3 landed first, taking **45 and 46** for the credential rows (§9.3). Assay's B2 is
therefore **already closed by revision 3**, not by this one: §9.3's row 45 is Forge's row field
for field, which is Assay's B2 repair word for word, and row 46 is the squash variant that makes
45's negative clauses non-vacuous. **B1's row is 47.** Rows 1–46 do not move, and a harness
written against revision 3 appends one `check` call.

### 9.5 Declined on the record — rows 38 and 44 keep one code

**Assay's advisory A2 is declined, and the reasoning is recorded rather than the conclusion
alone.** The finding: fixtures 38 (a required check reporting `failure`) and 44 (the commit has
no check runs at all) are two different events with two different owners — a lane-3/lane-4
defect, and a required workflow that never ran — and they share `L7-GATE-UNCONFIRMED`. §3 makes
the prose after ` -- ` explicitly unassertable, so the published ledger, which is R5b's whole
value, cannot distinguish them by contract. Assay is right that §4.1's own argument — *four codes
because four different pieces of work* — cuts toward splitting.

It is declined for two reasons, neither of which is that the point is weak:

- **ADR-0005 §6.5 class 8 sanctions the single code**, and §11's last bullet is that this
  document implements §6.5 and does not revisit it. Splitting class 8 is an ADR-0005 amendment
  authored against evidence, not a spec edit made before the detector has run once.
- **The behaviour is asserted either way.** 38 and 44 are two rows with two different fixture
  states, so a detector that conflates the *inputs* still fails one of them. What is lost is a
  machine-readable distinction in the ledger — and today nothing routes a repair off the code
  alone; a human reads the prose, which carries it.

**The revisit trigger is specific, and it is the evidence that would overturn this:** a published
`attest-history` ledger in which an `L7-GATE-UNCONFIRMED` line has to be opened by hand to decide
who owns the repair. At that point §6.5 class 8 is amended to two codes and §4 and §9 gain one
each in the same change. Recorded in §12.

**Revision 5 strengthens the case for declining rather than weakening it.** §6.1.1 now states the two
limbs as two different comparisons over two different sets, so an implementation cannot satisfy 38
and 44 by accident — but the two limbs still share `L7-GATE-UNCONFIRMED`, and naming them in prose
while they share a code is exactly the arrangement this subsection defends. Row 48 is a third fixture
on the missing limb and does not change that: three rows, two limbs, one code.

### 9.6 Row 48 — the limb the seam could not express

**Row 48 exists because revision 4's fixture table asserted class 8's *failing* limb three times and
its *missing* limb never.** Fixtures 38, 39 and 44 are a red observed run, an unreachable lookup and
an empty observed set; none of them is the case §6.5 class 8 states first — *a declared applicable
job with no check run of that name on `Lane-7-Head`* — because `check-runs/<sha>` carried only what
was observed and nothing carried what was expected. The limb was not merely unasserted; it was
**unimplementable from this document**, which is a lane-1 defect and not a lane-2 or lane-3 one.

**The row is the one shape that separates a set comparison from a conclusion scan**: four expected
jobs, three of them present and `success`, the fourth simply absent. There is no red conclusion
anywhere in the fixture, so an implementation that reads `check-runs/<sha>` and looks for a
non-`success` line passes 38, 39 and 44 and fails only this row. Fixture 44 does not catch it —
with *zero* observed runs, "scan what is there" and "compare against what is expected" happen to
agree.

**It is 48 and nothing renumbers.** Revisions 2 and 4 both shipped numbering defects in this
document, so: rows 1–47 keep their numbers and their setups, §6.1.1's default is keyed on
`tree-has-gate-workflow` precisely so that 44 does not have to gain a field, and a harness written
against revision 4 appends one `check` call.

---

## 10. Acceptance

**`ci/lane7-attest-test.sh` asserts one verdict per fixture for all 47 fixtures in §9 — each
verdict comparing the complete set of `L7-*` lines **and** the exit code, per §9.0 — with the
finding codes as written, and prints an `N passed, M failed` line in `ci/lane-gate-test.sh`'s
format.** That is the whole of VUL-56, and the closing line reads `47 passed, 0 failed`.

**`ci/lane7-attest.sh`, with its manifest at `ci/lane7-attest-floors.txt`, its workflow at
`.github/workflows/lane7-attest.yml`, and both §6.2 credentials supplied, passes all 47 fixtures
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
- **Whether `attest-self-test` becomes a required status check**, and the branch-protection
  change that would wire it. ADR-0005 **R8**, entangled with VUL-68, and filed as **VUL-75**
  (ADR-0005 amendment 5), owner Atlas. §8.2 states the consequence of it not being one. Forge is
  right that wiring a context is org configuration rather than a repository change and that it
  must not be done unilaterally; it is authorised on VUL-75, not here and not in a pull request.
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
- **An `L7-GATE-UNCONFIRMED` line in a published ledger has to be opened by hand to decide who
  owns the repair.** Fixtures 38 and 44 are two events — a required check reporting `failure`,
  and a commit with no check runs at all — sharing one code, and §3 makes the prose that
  distinguishes them unassertable. §9.5 declines the split today because ADR-0005 §6.5 class 8
  sanctions the single code and both inputs are still asserted. The first ledger line that costs
  a human that lookup is the evidence; §6.5 class 8 is amended to two codes, and §4 and §9 gain
  one each in that change.
- **A §6.3 defect is found inside a range a published run reported clean.** That is a second R5b
  event, and it means a finding class or a fixture is missing — re-open §4 and §9 together.
- **`LANE7_FIXTURE_DIR` appears in a production invocation.** The seam has leaked; §6.3's three
  mitigations have failed and the separation is re-designed rather than patched.
- **The attestation block format changes.** §4's register is keyed to §6.3 condition 4's six
  keys; a seventh key or a renamed one invalidates rows 3–5 and 30–32 and the harness is rewritten
  from the amended table by its original author, on a test-only pull request.
- **A second repository gains `.github/workflows/lane-gate.yml` without the four required
  contexts.** Forge measured this one and it is recorded as a revisit trigger rather than a
  fixture because **it is unreachable today**: at the time of writing only `platform` has the
  workflow, and `platform` has all four contexts required, while `docs`, `infra` and `vf-api` have
  branch protection with no `required_status_checks` block at all and `docs` has no
  `.github/workflows` directory. The gap is the mirror of fixture 11 — a tree that *has* the
  workflow while nothing requires its checks makes `Lane-7-Gate: PASS` true and worthless, and
  class 8 would confirm it. **This document deliberately enumerates no code for it**, because a
  code with no reachable fixture is a code the harness asserts nothing about. The moment the
  workflow is copied to a second repository, §4 gains a code and §9 gains a row in the same
  change.
