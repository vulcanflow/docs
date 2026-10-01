# ADR-0007 — The agent process bootstrap is checked into `vulcanflow/platform`

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Decides** | How the Superpowers skill library and the lane rules reach an agent session, given that two independent delivery mechanisms exist and one of them is blocked |
| **Closes** | No TDD §27 item. This is process, not product |
| **§25 identifiers** | **None.** This record creates no production behaviour, so no §25 identifier maps to it. Its acceptance is observational and is stated in §10 |
| **Depends on** | [ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md) — the lanes this bootstrap carries, and the gate that classifies its paths |
| **Bears on** | ADR-0004 (PR docs#27) — what `vulcanflow/platform` is allowed to contain |
| **Implementation** | `vulcanflow/platform` — `CLAUDE.md`, `.claude/skills/`, `.claude/superpowers/INSTALL.md`, `.claude/superpowers/LICENSE`, `.gitignore` (PR platform#5) |
| **Issue** | VUL-24 |

---

## 1. The question

The Superpowers library is how work is supposed to be done here: `brainstorming` before
design, `systematic-debugging` before a bug, `verification-before-completion` before anyone
says *passing*. VUL-13 installed it and reported it installed "in the vulcanFlow workspace".

It is not. It landed in the **Onboarding** project workspace, which only CEO sessions are
rooted at. The VulcanFlow project's workspace is a different directory — a git checkout of
`vulcanflow/platform` — and until this record it had neither `CLAUDE.md` nor
`.claude/skills/`. Every engineering agent has been working without any of it.

Two mechanisms can fix that, and they are not the same mechanism:

- **The control plane.** Paperclip's skill catalog holds the 15 skills plus the vulcanFlow
  adapter; attaching them per agent materialises them into that agent's runtime. VUL-22.
- **The repository.** A checked-in `CLAUDE.md` plus `.claude/skills/` reaches every session
  rooted at the checkout.

So the question is not "how do we get the skills to the agents". It is **which mechanism is
the design of record, what does the other one become, and what does `platform` have to carry
to make it work** — because vendoring 62 third-party Markdown files into a repository that
ADR-0004 defines as *the Cargo workspace* is a real cost and needs a reason, not a shrug.

---

## 2. What is true today, with the evidence

Checked during VUL-24, not recalled.

**The grant has not landed.** `GET /api/agents/me/skills` as Atlas returns
`403 Missing permission: agents:suggest-changes` with `reason: deny_missing_grant`. The 16
catalog entries report `attachedAgentCount: 1` — CEO, and nobody else. The run that wrote
this record had exactly one skill materialised: `paperclip`. VUL-22 is correct that the
content is ready and only the permission is missing; it is also still missing.

**The upstream pin is real.** At `github.com/obra/superpowers`,
`git rev-parse v6.4.2^{commit}` and `git rev-parse HEAD` both return
`8ca22dba9a94f28898bbce59f2537ff4d87c747d`, committed 2026-09-25, and `package.json` at
that commit reports `"version": "6.4.2"`. 15 skills, 74 files, 608 KB. Nine of those files
carry the executable bit.

**The catalog bodies are upstream plus an appendix, nothing more.** Diffed file by file: the
9 company-adapted skills are upstream verbatim with a trailing `## Paperclip note — helper
scripts not shipped` and/or `## vulcanFlow lane note` section appended, and the 12 executable
helper files removed. The 6 skills.sh-sourced skills are byte-identical to upstream at the
pin, verified against both a fresh clone and Paperclip's runtime cache.

**The VUL-13 install shipped the executables.** The Onboarding workspace tree is upstream
verbatim including all nine `+x` files. That contradicts the rule the catalog follows and is
one more reason this record does not vendor by copying that tree.

**`.claude/**` and `CLAUDE.md` cross no lane.** `ci/lane-gate.sh` classifies TEST from
`tests/*`, `crates/*/tests/*`, `fuzz/*`, `conformance/*`, `*/testdata/*`, `*/golden/*`,
`*.golden`, `*.snap`. None of those match `.claude/skills/test-driven-development/…` — the
name contains "test" but the path shape does not match. Run on the real branch,
`ci/lane-gate.sh all` reports 67 NEUTRAL paths, 0 PROD, 0 TEST, and passes all three checks.
`erosion` counts `#[test]`-family markers in `*.rs` only, and this change adds no `.rs` file.

---

## 3. The options

| | Option | Repo cost | Reach | Depends on |
|---|---|---|---|---|
| **1** | Vendor all 15 skills under `.claude/skills/` plus a `CLAUDE.md` bootstrap | 62 files, ~544 KB in `platform` | Every session rooted at the checkout: agent, subagent, human | Nothing |
| **2** | Commit only `CLAUDE.md` — the bootstrap rule and the precedence order — and let VUL-22 supply the skill bodies | One file | Only agents the board has granted attachment for | The grant |
| **3** | Vendor into a separate `vulcanflow/agent-process` repository and reference it | Zero in `platform` | Only sessions that clone the second repo | A second checkout per workspace |

---

## 4. Decision

**Option 1. Vendor the full tree plus `CLAUDE.md` into `vulcanflow/platform`.**

And — this is the part the options list does not cover — **the two mechanisms both stand
permanently. Neither supersedes the other.** The repository bootstrap is not a stopgap that
gets torn out when the grant lands.

Three reasons, in order of weight.

**4.1 — They do not have the same reach, in either direction.** The control plane reaches a
*given agent* wherever it works, including in `docs`, where Atlas does most of its work and
where no `platform` checkout exists. The repository reaches *every session rooted at the
checkout* regardless of identity — including a subagent, a human clone, and a fresh
workspace after re-provisioning. Each mechanism covers a case the other cannot. Treating
either as redundant would leave a real gap, and the gap in the repository's favour is the
one that bites hardest: a subagent dispatched by a coding agent is precisely the session that
most needs to be told it inherits its parent's lane.

**4.2 — Option 2 is conditional on a decision that is not ours.** It is the right shape *if*
the grant is certain, and the grant is a board action with no date. A process rule that
silently fails to load is worse than no rule, because nobody can tell from the inside that it
is missing — which is exactly how VUL-13 came to be reported as done. Option 1 holds whether
or not the board ever acts. CEO's reading on VUL-24 was option 2 if the grant lands and
option 1 if it does not; §2 settles which branch we are on, and §4.1 is why the answer does
not flip back later.

**4.3 — Versioned, reviewed, diffable.** The *design of record* lens says a decision that
lives only in a comment thread gets re-litigated. A process rule that lives only in a
control-plane blob is the same failure in a different store: it can be edited by anybody with
the grant, with no diff, no reviewer and no history. In the repository it goes through lane 6
like any other change, and `git log` answers when a rule changed and who approved it.

The cost is paid honestly: `platform` is a Cargo workspace repository and now also carries
~544 KB of vendored Markdown. §9 is where that cost is argued against the alternatives, and
R1 is the trigger that reopens it.

---

## 5. The vendored tree is byte-identical to what the catalog materialises

This is a decision, not an accident, and it is what makes two mechanisms tolerable instead of
two slowly diverging sources of truth.

- The 9 annotated skills are copied **from the catalog**, not from upstream — so they carry
  the same `## Paperclip note` and `## vulcanFlow lane note` appendices.
- The 6 skills.sh-sourced skills are copied from upstream at the pin, which is where the
  catalog gets them and what it was verified against.
- `vulcanflow-superpowers` is copied from the catalog verbatim.

Verified from the committed tree, after `.gitattributes` normalisation: `diff -r` is clean
for all 16 directories against their respective sources, and the MIT `LICENSE` matches
upstream.

The property this buys: **drift is a `diff`, not an argument.** A reviewer can check in one
command whether the repository and the control plane still agree, and the update procedure in
`INSTALL.md` makes moving both together the documented path. Had the repository carried a
third, locally-edited variant of the skill text, no such check would exist and the question
"which copy is current" would have no mechanical answer.

This is also why the 6 unannotated skills stay unannotated here, even though four of them
(`using-superpowers`, `dispatching-parallel-agents`, `finishing-a-development-branch`,
`requesting-code-review`) have genuine lane collisions and the repository — unlike the
catalog, which cannot modify a skills.sh-sourced body — *could* annotate them in place. That
was considered and rejected: annotating four files would create exactly the third variant
this section exists to prevent, for reconciliation text that already exists, in full, in
`vulcanflow-superpowers` §4. Point-of-use annotation is worth less than a mechanical
drift check.

---

## 6. The consequence that constrains real work: vendoring cannot be scoped per agent

A control-plane attachment is per agent. **A vendored directory is per repository.** Every
session rooted at `platform` sees all 16 skills, and there is no mechanism to show one agent
a subset.

That matters concretely, because the catalog deliberately withholds some:

| Skill | Catalog gives it to | Vendored tree gives it to |
|---|---|---|
| `test-driven-development` | Atlas, Scribe, Ledger, Assay | everyone |
| `dispatching-parallel-agents`, `subagent-driven-development`, `writing-skills`, `diagnosing-superpowers` | CEO, Atlas | everyone |
| `finishing-a-development-branch` | Crucible, Atlas | everyone |

The withholding is not arbitrary. `test-driven-development`'s first instruction is to write a
failing test, which is the one act Forge, Anvil and Kiln may never perform.
`subagent-driven-development` reads, to a coding agent, like a lawful way to get a test
written by somebody else. `finishing-a-development-branch` evaluates integration options,
which only Crucible does.

**Omitting those skills from the vendored tree is not the fix.** A directory is all-or-nothing
per repository, so omitting `test-driven-development` would deny it to Scribe, Ledger, Assay
and Atlas *in the exact workspace where they do the work it governs* — trading a soft
exposure for a hard denial.

So the compensating control is a rule, not an absence:

1. `CLAUDE.md` §6 states plainly that **availability is not permission**, and names
   `test-driven-development` as the worked example.
2. `CLAUDE.md` §4 states the **subagent-inheritance rule** — a subagent you dispatch is you,
   and it inherits your lane and every prohibition — and `CLAUDE.md` §7 imports
   `vulcanflow-superpowers`, whose §3 and §4 give the per-lane reading of every collision.
3. `ci/lane-gate.sh partition` refuses a pull request carrying both production source and
   tests regardless of which skill suggested it, and `erosion` refuses a drop in test count.
   The mechanism that actually says no is unchanged by what is on disk (ADR-0005 §4).

Item 3 is the load-bearing one. Items 1 and 2 reduce how often an agent heads down the wrong
path; only the gate refuses to let the result merge.

---

## 7. One source of truth per half of the tree

| Text | Source of truth | Changed by |
|---|---|---|
| The 15 upstream skills | `obra/superpowers` at the pinned commit | An update PR on `platform` that moves the pin, plus a catalog re-sync to the same commit |
| The `## Paperclip note` appendices | The catalog's rule: no externally sourced executables | Paperclip's policy, mirrored into `platform` |
| `vulcanflow-superpowers` and `CLAUDE.md` | **`vulcanflow/platform`** | Atlas, on a `platform` PR through lane 6; the catalog entry is then re-synced from the merged copy |

The third row changes where `vulcanflow-superpowers` is amended, and it is deliberate. That
text is vulcanFlow's own process writing: it states the subagent-inheritance rule and the
per-lane reading of every Superpowers collision. By §4.3 it belongs where it can be reviewed
and diffed. The catalog entry becomes a mirror.

This has a cost worth naming: a catalog edit made outside the repository is now
*unauthorised but not impossible*, and nothing mechanically detects it. R4 is the trigger.
Separately, this changes a mechanism VUL-22 owns, so it is flagged to CEO on VUL-24 rather
than assumed.

---

## 8. No executables, and no hook

**The 12 upstream executable helpers are not vendored.** Paperclip does not install
externally sourced executables into an agent runtime, which is why the catalog omits them and
why five skills carry a note saying which files are missing. `INSTALL.md` lists all 12 by
skill. The invariant is checkable: `find .claude -type f -perm -u+x` must return nothing, and
does. `brainstorming/scripts/frame-template.html` and `writing-skills/graphviz-conventions.dot`
are kept — inert data, not executables, and the catalog ships them too.

**No `SessionStart` hook.** Upstream ships one and the VUL-13 install kept a copy. A hook is
an executable, a project hook needs a trust prompt on first interactive use, and the
`CLAUDE.md` import achieves the same bootstrap without either. Keeping the whole of
`.claude/` to Markdown and inert data is the point — it means the entire 544 KB can be
reviewed by reading it, and nothing in it can run.

**`.claude/settings.local.json` is `.gitignore`d.** Tracking `.claude/` creates a hazard that
did not exist before: the harness writes that file per workspace, so a `git add -A` would
commit one agent's local state over the next agent's. The ignore closes it. It is the only
entry; the Rust ignores belong with the Cargo workspace when it lands.

---

## 9. Alternatives considered, and why they lost

**Option 2 — `CLAUDE.md` only.** Lost on §4.2: it is conditional on a board grant with no
date, and its failure mode is silent. It also does not reach a subagent or a human clone,
since those get the skill bodies from the workspace or not at all. Worth revisiting only
under R2, and even then §4.1 says it becomes a *reduction* of the vendored tree, not a
replacement for it.

**Option 3 — a separate `vulcanflow/agent-process` repository.** Clean separation, and it
keeps `platform` to the Cargo workspace that ADR-0004 describes. It loses on the mechanism:
process files reach a session because they are *in the checkout the session is rooted at*. A
second repository is not in that checkout, so something has to clone it into place — a
submodule, a bootstrap script, or a workspace-provisioning step. Each of those is a moving
part that can fail quietly, which is the failure mode §4.2 rejects. ADR-0004's objection is
real but it is about repository *purpose*, and `CLAUDE.md` at the root of the repository an
agent works in is not a foreign concern; it is how that repository tells the agent how to
work in it.

**Vendor only the subset each lane may use.** Rejected in §6: a directory cannot be scoped per
agent, so the subset that protects the coding agents is the subset that disarms the test
authors.

**Re-annotate the 6 unannotated skills in the repository.** Rejected in §5: it would create a
third variant of the skill text and destroy the one-command drift check, to duplicate
reconciliation that `vulcanflow-superpowers` §4 already states in full.

**Copy the VUL-13 tree from the Onboarding workspace.** Rejected: it ships all nine
executables (§2), and it lacks the `## Paperclip note` and `## vulcanFlow lane note`
appendices, so it is neither policy-compliant nor catalog-identical.

**A `SessionStart` hook instead of the `CLAUDE.md` import.** Rejected in §8.

---

## 10. Consequences

1. **`platform` carries ~544 KB of third-party Markdown**, 62 files, in a repository ADR-0004
   defines as the Cargo workspace. Every `platform` diff from here on has a `.claude/` subtree
   above it in the tree listing. This is the price and it is paid once.
2. **A Superpowers update is now a reviewed change.** Moving the pin is a `platform` pull
   request through lane 6, and it must re-sync the catalog in the same breath or break the
   byte-identity in §5. `INSTALL.md` carries the procedure; the discipline is Atlas's.
3. **Every agent sees skills its lane does not entitle it to use** (§6). The gate is
   unchanged and still refuses the result; what changed is that an agent can now *read* a
   skill that tells it to do something it may not do. `CLAUDE.md` §4 and §6 exist for exactly
   that session.
4. **The supply-chain surface grows by 62 Markdown files, and by nothing executable.** No
   file under `.claude/` carries the executable bit, no file is compiled, and nothing is
   fetched at runtime. Reviewing it is reading it.
5. **`vulcanflow-superpowers` now has two homes** with the repository authoritative (§7). A
   catalog-only edit is unauthorised and mechanically undetectable. R4.
6. **This record's acceptance is observational, not a §25 identifier.** It is satisfied when a
   fresh agent session rooted at the `platform` workspace reports the 16 skills in its
   available-skills list, invocable by bare name, with `using-superpowers` injected at session
   start. **A session cannot verify this for itself** — its skill list is fixed before the
   commit lands — so the confirming observation comes from the *next* session, and that is the
   closing step of VUL-24.
7. **The two stale lines VUL-22 fixes in the nine `AGENTS.md` files are already correct here.**
   `CLAUDE.md` §5 names the CI path gates rather than `CODEOWNERS`, and states lane 6 as two
   live reviewers. Agents will read one true version and one stale version until VUL-22 lands;
   precedence (`CLAUDE.md` §1) does not resolve that, because `AGENTS.md` outranks this file.
   Named here so it is not discovered as a surprise. It is VUL-22's to fix, not this record's.

---

## 11. Revisit triggers

- **R1 — the vendored tree stops being cheap.** If `platform`'s `.claude/` subtree exceeds
  roughly 2 MB, or an upstream release adds compiled or fetched-at-runtime content, or the
  Markdown starts colliding with repository tooling, reopen §4 with option 3 as the leading
  alternative and measurements rather than impressions.
- **R2 — the VUL-22 grant lands and is verified.** When all nine agents report the expected
  keys from `GET /api/agents/{id}/skills`, re-examine whether the vendored tree can shrink to
  `CLAUDE.md` plus `vulcanflow-superpowers` and let attachment supply the 15 bodies. The
  answer is *not automatically yes* — §4.1 still applies to subagents and to human clones —
  so the test is specifically: does a dispatched subagent in a `platform` session inherit the
  attached skills? If yes, the reduction is available. If no, the tree stays.
- **R3 — upstream changes its install story.** A non-interactive, project-scoped plugin install
  would remove the reason for vendoring in §1 of `INSTALL.md`. Re-read upstream's `README.md`
  on every pin move.
- **R4 — the repository and the catalog are observed to have drifted.** Any divergence in the
  16 skill bodies, or an amendment to `vulcanflow-superpowers` found in the catalog but not in
  `platform`'s history, is a §7 violation. Reconcile to the repository copy, then decide
  whether §7's convention needs a mechanical check instead of a rule.
- **R5 — the lane gate's path classification changes.** §2's finding that `.claude/**` is
  NEUTRAL is a property of the current `classify_path`. If a future pattern makes any
  `.claude/` path TEST or PROD, this subtree would start crossing lanes on every edit.
  Re-run `ci/lane-gate.sh all` on a `.claude/`-only diff whenever the gate changes.
- **R6 — a coding agent cites a vendored skill to justify a lane crossing.** §6's compensating
  control failed at the advisory layer. Reproduce the reasoning in `vulcanflow-superpowers`
  as an explicit refusal, and check whether the gate caught the attempt; if it did not, that
  is an ADR-0005 R5 event as well.

---

## 12. Relationship to the other records

- **ADR-0005** supplies the lanes `CLAUDE.md` §5 restates and the gate §2 was checked
  against. Nothing here changes a lane, an owner, a prohibition, or the escape hatch.
  `CLAUDE.md` is a *delivery vehicle* for ADR-0005, not an amendment to it; where the two
  could be read as differing, ADR-0005 wins, and `CLAUDE.md` §1 says so.
- **ADR-0004** defines `vulcanflow/platform` as the Cargo workspace, and this record adds a
  subtree that is not part of it. That tension is the whole of §4 and §9, and R1 is its
  revisit trigger. Note that ADR-0004 is **not on `main`** — it is PR docs#27, in lane 6 under
  VUL-23 — so the constraint this record weighs against is not yet itself a merged record.
- **Does not touch** ADR-0001, ADR-0002, ADR-0003 or ADR-0006. No language, crate, pin,
  integration-boundary or false-positive-semantics question is in scope. The
  `8ca22dba9a94f28898bbce59f2537ff4d87c747d` pin is a *documentation* pin, not a dependency
  pin: Superpowers is not in `Cargo.toml`, is not built, and ADR-0002's no-pin-bump-without-an-ADR
  rule does not reach it. Moving it is governed by §7 and R3 instead.
- **VUL-22** remains the primary per-agent mechanism and is not superseded (§4.1). Two things
  there are affected and are CEO's to take or reject: §7 moves where `vulcanflow-superpowers`
  is amended, and §10 item 7 names the stale-`AGENTS.md` window this record cannot close.

---

## 13. Amendment history

| | Date | Change |
|---|---|---|
| — | 2026-10-01 | Accepted as recorded. No amendments. |
