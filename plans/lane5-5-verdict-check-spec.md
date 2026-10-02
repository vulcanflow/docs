# Lane-1 spec — the `pre-pr-review-verdict` required check

| | |
|---|---|
| **Owner of this spec** | Atlas (lane 1) — VUL-28 |
| **Authority** | ADR-0005 §6.6, which is the only statement of lane 5.5's mechanics. Where this spec and §6.6 disagree, §6.6 wins and this file is corrected. |
| **Lane 2 — fixture harness** | Scribe · `ci/pre-pr-review-verdict-test.sh` |
| **Lane 3 — implementation** | Forge · `ci/pre-pr-review-verdict.sh` and `.github/workflows/pre-pr-review-verdict.yml` — §4.4's first row, pure computation over its inputs |
| **Lane 4 — the run** | Crucible |
| **§25 identifiers** | **n/a (no §25 identifier in scope).** §25 is a traceability matrix over TDD §2–§23 product requirements and the delivery pipeline's own tooling has no row there. Inventing one would break the identifier→requirement mapping. ADR-0005 §4.4. |
| **Repositories** | `vulcanflow/platform` and `vulcanflow/docs`. `docs` has no workflow today, so this is its first — see §7. |

## 0. The acceptance statement

> Given a pull-request body and that pull request's head sha, `ci/pre-pr-review-verdict.sh` exits
> non-zero and emits at least one `L55-` code for every fixture in §5 the table marks **FAIL**,
> exits zero and emits no `L55-` code for every fixture marked **PASS**, and reads the body only
> from a file whose contents are never interpolated into a shell command.

One sentence, judgable by a stranger holding §5's table and the delivered harness. **It is the
whole of the acceptance** — there is no §25 identifier to go green, so there is nothing else.

## 1. What this check is for, and what it is not

**It fires at the moment of omission.** CEO's audit of 2026-10-02 measured nine pull requests
opened after the board rule took effect and found **two carrying the verdict block and seven
not** (ADR-0005 §6.6). Seven of nine carried *no block*, not a false one. A check over the
block's shape catches every one of those on the event that creates them.

**It cannot verify that the CLI ran, and nothing in this spec pretends otherwise.** The CLI keeps
per-review state under `$HOME/.coderabbit/reviews` on the agent runner; a GitHub Actions runner
has a different `$HOME` and no route to that one. So the check reads a **claim typed by an
author** and tests its shape against a sha. An author who types a well-formed block naming the
right sha and zero findings passes without having run anything. Lane 5.5 is author-attested;
ADR-0005 §6.6 says so in the record and R13's second limb is what fires if that ever changes.

**It is not a merge condition.** ADR-0005 §6.3 is. This check is a required status check on the
pull request and contributes nothing to §6.3's five conditions.

## 2. Invocation

| Invocation | Does |
|---|---|
| `ci/pre-pr-review-verdict.sh <body-file> <head-sha>` | Classifies one pull-request body. `<body-file>` is a path to a file holding the body verbatim. `<head-sha>` is 40 lowercase hex. Reads nothing else — no network, no git, no environment beyond `TMPDIR`. |

**Exit status** — three values, and the third is not a pass:

| Exit | Meaning |
|---|---|
| `0` | No finding. The check is green. |
| `1` | At least one finding. The check is red. |
| `2` | Usage error — wrong argument count, unreadable `<body-file>`, `<head-sha>` not 40 lowercase hex. **Red**, and distinct from `1` so a broken workflow is not reported as a non-compliant author. |

**No argument is ever a shell word.** Both arguments are read as data. The body is **never**
passed on a command line, never `eval`'d, never used as a `printf` format string, and never
interpolated into a `${{ }}` expression — §4 states how the workflow delivers it and why.

## 3. Output

One finding per line on stdout, in the order the codes appear in §4's register, then a verdict
line. Deterministic for a given input; no timestamps, no paths.

| Shape | Emitted | Meaning |
|---|---|---|
| `MODE body <head-sha>` | always, as the first stdout line | the sha the run compared against |
| `L55-<CODE>[ -- <free text>]` | once per finding | one line per finding |
| `VERDICT PASS` / `VERDICT FAIL` | always, as the last stdout line | matches the exit status; `FAIL` on exit 1 and on exit 2 |

Free text after `--` is for a human and is **never** parsed by the harness. The harness asserts
on codes and on exit status only, so a reworded message is not a test failure.

## 4. The finding-code register

The block is located by its heading, exactly `### Pre-PR CodeRabbit CLI review (lane 5.5)`, at
the start of a line. Everything below is relative to the first such heading.

| Code | Fires when |
|---|---|
| `L55-MISSING` | No such heading anywhere in the body. **Suppresses every other code** — there is no block to say anything further about. |
| `L55-DUPLICATE` | Two or more such headings. Classification proceeds on the first; this code is additive, because two blocks mean two claims and the author has not said which is current. |
| `L55-ROW-MISSING` | One of the five rows — `CLI version`, `Command`, `Tree reviewed`, `Findings`, `Run at` — is absent from the block. One code per absent row, each naming the row in free text. |
| `L55-PLACEHOLDER` | A row's value still holds an HTML comment, or is empty. This is the unedited-template case, and it is called out separately because the template ships `Findings` pre-filled as `0`: an author who edits nothing otherwise would satisfy `L55-FINDINGS` while having claimed nothing. One code per such row. |
| `L55-SHA-SHAPE` | `Tree reviewed` is not 40 lowercase hex, optionally wrapped in backticks. **Full 40 required**, not an abbreviation: a 7-hex prefix makes the comparison in the next row ambiguous and the author has the full sha in front of them. |
| `L55-SHA-MISMATCH` | `Tree reviewed` is well-shaped and is not `<head-sha>`. **This is the code that makes lane 5.5 per-push** (ADR-0005 §6.6). Suppressed by `L55-SHA-SHAPE` — there is nothing to compare. |
| `L55-FINDINGS` | `Findings` is not exactly `0`. Any other value fails, including a number, a word, and a range. Under the board's "all issues fixed" the only passing value is zero; if VUL-1 is answered "severity floor", this code and the next are respecified then and not before. |
| `L55-DECLINED` | The line `Declined findings:` is absent, or its value is not exactly `none.` — fail-closed, for the same reason. |
| `L55-CLI-VERSION` | `CLI version` is not `<major>.<minor>.<patch>`, each a decimal integer. Shape only: this spec pins **no** version, because a version floor is a supply-chain decision and belongs in an ADR, not in a check (ADR-0002's rule, applied to a tool rather than a crate). |
| `L55-TIMESTAMP` | `Run at` is not an ISO-8601 UTC instant — `YYYY-MM-DDTHH:MM[:SS]Z`. A local time or a bare date fails. Shape only: the check does **not** compare it to anything, since it has no trustworthy clock relationship to the push and a staleness window would be a guess. |
| `L55-COMMAND` | `Command` is absent, or names a flag ADR-0005 §6.6 forbids — `--deep`, `--remote`, `--api-key` — or **contains an absolute path under `/paperclip/`**. The last limb is not style: all four repositories are public (ADR-0005 §8), the CLI is referenced as `$CODERABBIT_BIN`, and a pull-request body is the easiest place for the instance path to leak into the public record. |

**Every code is blocking. There is no advisory code in this register**, which is a deliberate
difference from the lane-7 detector: that one classifies history it did not gate and needs a
severity axis, and this one gates a body its author can fix in thirty seconds.

## 5. The fixture set — each with the outcome it must produce

**This is the control.** ADR-0005 §4.4: a reviewer checking a delivered harness against a list of
*names* can detect deletion but not relaxation, so each fixture is named **with its required
outcome**, and a delivered harness that preserves a name while hollowing its assertion is a
blocking lane-6 finding. Each fixture is a pull-request body in a file plus a head sha.

Let `H` = `0123456789abcdef0123456789abcdef01234567` and `H2` = `89abcdef0123456789abcdef0123456789abcdef`.

| # | Fixture | Required outcome |
|---|---|---|
| 1 | The committed `.github/pull_request_template.md` with every row filled, `Tree reviewed` = `H`, head `H` | exit `0`, `VERDICT PASS`, no `L55-` line |
| 2 | As 1, with the block's heading line deleted | exit `1`, `L55-MISSING` **and no other code** |
| 3 | An empty body | exit `1`, `L55-MISSING` only |
| 4 | A body with prose but no block | exit `1`, `L55-MISSING` only |
| 5 | As 1, with the whole block duplicated verbatim | exit `1`, `L55-DUPLICATE` |
| 6 | As 1, with a second block whose `Findings` is `3` | exit `1`, `L55-DUPLICATE`, and **no** `L55-FINDINGS` — the first block governs |
| 7 | As 1, with the `Run at` row deleted | exit `1`, `L55-ROW-MISSING` naming `Run at` |
| 8 | As 1, with all five rows deleted | exit `1`, five `L55-ROW-MISSING` codes |
| 9 | The template **exactly as committed**, unedited | exit `1`, `L55-PLACEHOLDER` for `CLI version`, `Tree reviewed` and `Run at`, and **no** `L55-FINDINGS` — this is the fixture that proves the pre-filled `0` is not a free pass |
| 10 | As 1, with `Tree reviewed` = `0123456` (7 hex), head `H` | exit `1`, `L55-SHA-SHAPE`, and **no** `L55-SHA-MISMATCH` |
| 11 | As 1, with `Tree reviewed` = `0123456789ABCDEF0123456789ABCDEF01234567`, head `H` | exit `1`, `L55-SHA-SHAPE` — uppercase is not accepted, so the comparison has one normal form |
| 12 | As 1, `Tree reviewed` = `H`, head `H2` | exit `1`, `L55-SHA-MISMATCH` |
| 13 | As 1, `Tree reviewed` = `` `H` `` in backticks, head `H` | exit `0`, `VERDICT PASS` — backticks are the template's own typography |
| 14 | As 1, `Findings` = `1` | exit `1`, `L55-FINDINGS` |
| 15 | As 1, `Findings` = `none` | exit `1`, `L55-FINDINGS` |
| 16 | As 1, `Findings` = `0 (2 declined)` | exit `1`, `L55-FINDINGS` — the declined path is not built |
| 17 | As 1, `Declined findings:` line deleted | exit `1`, `L55-DECLINED` |
| 18 | As 1, `Declined findings: one, see below` | exit `1`, `L55-DECLINED` |
| 19 | As 1, `CLI version` = `0.8` | exit `1`, `L55-CLI-VERSION` |
| 20 | As 1, `CLI version` = `v0.8.2` | exit `1`, `L55-CLI-VERSION` |
| 21 | As 1, `Run at` = `2026-10-02 09:40` | exit `1`, `L55-TIMESTAMP` |
| 22 | As 1, `Run at` = `2026-10-02T09:40Z` | exit `0`, `VERDICT PASS` |
| 23 | As 1, `Run at` = `2026-10-02T09:40:11Z` | exit `0`, `VERDICT PASS` |
| 24 | As 1, `Command` = `` `$CODERABBIT_BIN review --agent --base main --deep` `` | exit `1`, `L55-COMMAND` |
| 25 | As 1, `Command` = `` `$CODERABBIT_BIN review --agent --remote` `` | exit `1`, `L55-COMMAND` |
| 26 | As 1, `Command` naming an absolute `/paperclip/instances/.../tools/bin/coderabbit` | exit `1`, `L55-COMMAND` |
| 27 | As 1, with `$(id)` and `` `id` `` in the "What this changes" prose | exit `0`, `VERDICT PASS`, **and no evidence of either having been evaluated** — the fixture asserts on the absence of command execution, not only on the verdict |
| 28 | As 1, with a body line reading `exit 1; rm -rf /` inside a fenced block | exit `0`, `VERDICT PASS` |
| 29 | As 1, body file containing CRLF line endings throughout | exit `0`, `VERDICT PASS` — GitHub returns bodies with CRLF |
| 30 | Wrong argument count (one argument) | exit `2`, `VERDICT FAIL` |
| 31 | `<body-file>` does not exist | exit `2`, `VERDICT FAIL` |
| 32 | `<head-sha>` = `main` | exit `2`, `VERDICT FAIL` |

**Fixtures 27 and 28 are the security fixtures and they are not optional.** A pull-request body is
attacker-controlled on a public repository. They assert the property §2 states — that the body is
read as data — and they are the two a relaxation would most plausibly drop, because they pass
today and look redundant.

## 6. What is deferred, named so it is not read as covered

**§4.5's three sub-checks do not reach this pair.** ADR-0005 §4.5 specifies monotonicity, an
assertion floor and a sentinel *as steps inside the existing `gate-self-test` job*, written
against `classify_path()`, `ci/lane-gate.sh` and `ci/lane-gate-test.sh` by name. There is no
generic detector/harness mechanism for a second pair to inherit, and `pre-pr-review-verdict`
shipped on the belief that §4.5 already covers it would have **none** of the three. §4.4 has
already had to withdraw exactly this overclaim once; it is not made here.

**The interim control is this table and lane 6.** The fixture count is **32** and every row above
names its outcome, so a reviewer has something to check a delivered harness against. Extending
§4.5's three sub-checks to cover this pair is lane-1 work that does not exist yet and is named as
R8's tail rather than assumed.

## 7. Dependencies, as blocker edges and not as prose ordering

| This | Blocks on | Why |
|---|---|---|
| lane-2 fixtures (Scribe) | **`docs#39` / `platform#9`** — the PR template | Fixture 1 and fixture 9 are *the committed template*. Writing them against a draft would make the harness assert on a file that does not exist. |
| lane-2 fixtures (Scribe) | **ADR-0005 amendment 8** | §6.6 is this spec's authority. |
| lane-3 implementation (Forge) | the lane-2 fixtures | Tests precede code. |
| **merging** the lane-3 implementation | the GATE reclassification — (i) §4.2/§4.5 amendment (Atlas), (ii) `ci/lane-gate-test.sh` fixtures over the new paths (Scribe), (iii) `classify_path()` extended (Forge) | The day this check becomes required is the day ADR-0005 §4.2 is wrong about its paths. §6.6 states why the row is not simply edited: `classify_path()` enumerates three paths literally and §4.5's arguments count that set. |
| **requiring** the check | `PUT /repos/vulcanflow/{docs,platform}/branches/main/protection` | R2's live remainder: whoever adds a repository's first workflow owns making its checks required. On `platform` it joins the four lane-gate contexts; on `docs` it is the first. A check that runs and is not required is a red mark nobody has to clear. |

## 8. The workflow, and the one way it must not be written

`.github/workflows/pre-pr-review-verdict.yml`:

- **Trigger `pull_request`, types `[opened, edited, synchronize, reopened]`.** `edited` is
  required — the body can change without a push, and the body is what this check reads.
  `synchronize` is required — it is what makes the head-sha clause bite (ADR-0005 §6.6).
- **Never `pull_request_target`.** That trigger runs with the base repository's token against the
  head's content, which is the exact combination ADR-0005 R11 was opened over. This check needs
  no write token and no secret.
- **`permissions: contents: read`.** Nothing else.
- **The body goes through the environment, never through `${{ }}` in a `run:` block.** Set
  `env: PR_BODY: ${{ github.event.pull_request.body }}`, then write it to a file with
  `printf '%s' "$PR_BODY" > "$body"`. A `${{ }}` expression inside `run:` is textually
  substituted into the shell script **before** the shell parses it, so a body containing
  `"; id; "` executes. Fixtures 27 and 28 assert the script's half of this; the workflow's half
  is not fixture-testable and is therefore a **named lane-6 review obligation** on the
  implementation pull request.
- **`CODERABBIT_BIN` is not referenced.** The check does not run the CLI; it reads a claim about
  a run. Nothing in CI needs the binary.
