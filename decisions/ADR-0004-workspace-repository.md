# ADR-0004 — The Cargo workspace lives in `vulcanflow/platform`

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | the repository question left open by [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) appendix row #14 — *"repository list follows the workspace topology"* |
| **Does not close** | any TDD §27 item. This is a repository-topology decision, not an open engineering item |
| **Decides nothing about** | the implementation language ([ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md)), any dependency pin ([ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md)), any integration boundary ([ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md)), or the delivery pipeline ([ADR-0005](./ADR-0005-delivery-pipeline-and-lane-enforcement.md)) |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §2.5.2, §2.5.3, Appendix C glossary |
| **Issues** | [VUL-12](/VUL/issues/VUL-12) (this record). Consumed by [VUL-4](/VUL/issues/VUL-4) (the program plan). Executed by [VUL-6](/VUL/issues/VUL-6) (the fold) |

## Why this record is being filed a second time

This decision was first written on PR [`docs#23`](https://github.com/vulcanflow/docs/pull/23)
(branch `atlas/adr-0004-workspace-repo-and-adr-0002-a2`, commit `cee1edc`). That PR was
**closed unmerged at 2026-10-01T18:50:19Z with zero reviews and zero comments**, and tagged
`archive/pr-23-adr-0004`. It was not rejected on its merits; it was closed along with
`docs#20` and `docs#22` when the board that raised it was retired, so every `VUL-*` identifier
it cited (`VUL-6`, `VUL-39`) now points at nothing.

Three things therefore had to change, and the re-file is not a copy:

1. **The board identifiers are resolved against the current board.** The stale ones are gone.
2. **`docs#23` bundled two decisions in one commit** — ADR-0004 and ADR-0002 **amendment A2**
   (§3.7, the pin corrections). This record carries ADR-0004 **only**. A2 is a pin change and
   is explicitly out of scope here; see [Loose end](#loose-end-adr-0002-amendment-a2) below.
3. **The repository state was re-audited rather than restated**, and the audit changed three
   factual claims the draft made — one of which this record's own first commit repeated. See
   [Provenance](#provenance).

The substance of the decision is unchanged, because the reasoning was never the problem. Under
the design-of-record lens an unmerged PR decides nothing, and [ADR-0005 §11](./ADR-0005-delivery-pipeline-and-lane-enforcement.md)
plus the Atlas role briefing both already cite ADR-0004 as settled. That gap is what this
closes.

---

## 1. Context

### 1.1 What the design of record says

TDD §2.5.2 and the Appendix C glossary define the Cargo workspace as:

> The single Rust build unit containing all VulcanFlow crates, with one pinned toolchain and
> lockfile.

ADR-0001 closed the six per-repository Go bootstraps in favour of that one workspace, and
recorded the consequence in its appendix — *"repository list follows the workspace topology"* —
**without naming a repository**. The organisation holds **fifteen** repositories (§2.1); one
build unit cannot span them. So: which repository holds it?

### 1.2 What actually exists

Two incompatible topologies are live in the `vulcanflow` organisation at once.

**The workspace.** `vulcanflow/platform` PR
[#1](https://github.com/vulcanflow/platform/pull/1) declared **fourteen** member crates in one
`Cargo.toml`, against one `rust-toolchain.toml` (channel `1.98.1`, target
`aarch64-unknown-linux-gnu`) and one committed `Cargo.lock`:

| Kind | Crates |
|---|---|
| Library (6) | `vf-core`, `vf-graph`, `vf-translator`, `vf-authz`, `vf-db`, `vf-store` |
| Binary (8) | `vf-api`, `vf-operator`, `vf-admission`, `vf-ingest`, `vf-report`, `vf-meter`, `vf-abuse`, `vf-hook-notify` |

**Thirteen of those fourteen have a record; one does not.** TDD §2.5.2 `[PROPOSED]` names the
workspace layout as library crates `vf-core`, `vf-graph`, `vf-translator`, `vf-authz`, `vf-db`
and binary crates `vf-api`, `vf-operator`, `vf-admission`, `vf-ingest`, `vf-report`, `vf-meter`,
`vf-abuse`, `vf-hook-notify` — **thirteen**. `vf-store` is the fourteenth and **has no
authority on `docs@main` at all**: `grep vf-store decisions/ADR-0002-*.md` returns nothing, and
ADR-0002 §3 is the **third-party dependency pin set** (§3.1–§3.6), not the workspace member
list. An earlier draft of this record cited "ADR-0002 §3" for the member set; that citation was
wrong and is withdrawn here.

This record does not fix that gap, because it decides repository layout and the crate set is not
a layout question. It **names** it: the workspace member set needs a record of its own — an
ADR-0002 amendment filed alongside the A2 re-file (§4.4) — and until then `vf-store` is a crate
that exists in an archived manifest and in no decision. Tracked in `decisions/README.md` under
**Still open**.

**That PR is closed, and it does not weaken this record.** `platform#1` was closed unmerged at
2026-10-01T19:29:06Z by board decision on [VUL-1](/VUL/issues/VUL-1), which chose *rebuild
from scratch against the TDD* over landing it or re-raising it from its tag. The reason was
provenance, not topology: the workspace was authored before ADR-0005 existed, so landing it
would have made the largest commit in the program the one change that never passed the lane
gate. Its head survives as tag `archive/pr-1-cargo-workspace`
(`66f4a91cb5a916fde597d5e4e4e698d108e4f5ae`, 48 files) and at `refs/pull/1/head`; the head
branch is deleted. `platform@main` therefore holds **no crates yet** — only `.gitattributes`
and the ADR-0005 lane gate merged as `platform#2`.

The fourteen-crate set is cited here as **what `platform#1`'s manifest declared** — evidence of
the shape of the workspace, not an approved member list (see the paragraph above). The rebuild
re-authors those crates under the pipeline; it does not re-decide which repository they live in.
**A repository-layout decision does not depend on the state of any one pull request**, which is
precisely why this record is worth having independently of the rebuild.

**The polyrepo.** Separately, twelve per-component repositories were created on
2026-09-09. **Eight** of them are named after crates in that same manifest — `vf-api`,
`vf-authz`, `vf-translator`, `vf-operator`, `vf-ingest`, `vf-meter`, `vf-report`, `vf-abuse`.
A ninth, `vf-remediation`, is named after a TDD §2.3 *component* that has **no crate at all**,
in the archived manifest or anywhere else; it is folded for the same reason as the eight but
on a different ground, and §2.1 states that ground rather than letting the name imply a crate.
The organisation holds **fifteen** repositories in total.

### 1.3 What breaks if both persist

This is the part that is not a tidiness argument.

- **`vf-core` version skew returns, which is the exact failure ADR-0001 closed.** A repository
  named `vf-authz` is an invitation to put a lockfile in it. Two lockfiles means two resolved
  versions of `vf-core`, across precisely the crates that hold scope matching, allowance
  accounting and finding state. §2.5.2 puts safety-relevant logic in pure library crates so it
  can be property-tested in isolation; two copies of those types silently defeats that.
- **Crate purity stops being checkable.** ADR-0002's boundaries (no I/O in `vf-core`,
  `vf-authz`, `vf-graph`, `vf-translator`) are enforced by a `ci/crate-boundaries.sh` +
  `ci/pure-crate-dependencies.txt` pair that runs against the whole workspace. A crate in its
  own repository is outside that gate's reach, and a boundary that is only enforced in one of
  two places is not enforced.
- **The single-TLS-provider rule becomes unverifiable.** ADR-0002 §3.6 names duplicate `rustls`
  / `hmac` / `sha2` / `digest` and any `ring` edge as an *observed* failure mode, and a
  `ci/tls-provider-assertions.sh` gate asserts against it. That assertion is a property of
  **one** dependency graph. Split the graph and the gate's answer stops meaning anything.

  Both gates were written on `platform#1` and now exist only at
  `archive/pr-1-cargo-workspace`; re-establishing them is part of the rebuild. The argument
  here is about what a *split graph* would do to them, so it does not depend on their being
  live today — a gate that cannot yet run is a scheduling fact, whereas a gate that cannot be
  written is a design failure, and the polyrepo is the second one.
- **The lane gate has nothing to stand on.** ADR-0005 §4.2 classifies paths like
  `crates/*/src/**` and `crates/*/tests/**`; its own §7 item 5 records that the gate is live on
  `platform` only. Nine unfolded repositories are nine places the pipeline is not enforced.
- **A reader is actively misdirected.** The next agent looking for `vf-authz` finds an
  organisation that offers two plausible answers and no record saying which is wrong. That is
  the concrete thing this record prevents.

**Nothing has to be migrated, because there is nothing in those repositories.** Every one of
the twelve per-component repositories has a content-free default branch: eleven have no commits
at all (zero refs), and `vf-api`'s `main` is one commit, `45d7ded9` *"chore: initialize
repository"*, with an empty tree. The only code any of them ever held is `vf-api`'s closed
PR #1 Go scaffold, which survives as `refs/pull/1/head` (`773357eb`) and which ADR-0001 already
retired. **The fold is an archive-and-redirect operation, not a code migration.** That is why it
is cheap now and expensive later.

---

## 2. Decision

**The Cargo workspace is `vulcanflow/platform`, and it is the only repository that holds
VulcanFlow Rust crates.** Forge created that repository mid-task on the predecessor board and
opened `platform#1` against it; this record ratifies the name rather than renaming after the
fact.

**The rule, stated once so the sixteenth repository does not need a decision of its own.** It is
stated over **crates**, not over repositories, and that distinction is load-bearing:

> **Every VulcanFlow-owned Cargo crate lives in `vulcanflow/platform`. No other repository
> contains a `Cargo.toml`.** Everything that is not a Cargo crate stays where it is.

The earlier phrasing — *"a repository joins the workspace if and only if it builds with
`cargo`"* — gets the common cases right and the interesting one wrong. TDD §21.2 runs per-tenant
schema migrations from **a Rust binary** invoked as an Argo CD PreSync hook. Read over
repositories, the old rule says `infra` acquires a `cargo` build and therefore joins the
workspace, which is absurd: `infra` is the GitOps repository. Read over crates, the answer is
immediate and correct — **the migration binary is a crate in `platform`; `infra` holds the
manifest that runs its published image.** A repository is not pulled in by needing a Rust
artifact; it consumes one.

This is the general form, and it is why §2.2's last bullet holds: **source lives in exactly one
repository, build artifacts cross freely.** `vf-graph` compiled to WebAssembly and consumed by
`vf-web` is the same pattern as the migration binary's image consumed by `infra`.

The workspace root is the **repository root** — `Cargo.toml`, `Cargo.lock`,
`rust-toolchain.toml`, `deny.toml` and `crates/` at the top level, not under a `rust/`
subdirectory. One build unit per repository means the repository *is* the build unit. A
subdirectory implies a sibling that is not going to exist, and it costs a `working-directory:`
on every CI step and a prefix on every ADR-0005 §4.2 path class. `platform#1` already did this;
the rebuild inherits the layout as a **requirement of this record**, not as a convention copied
from a closed branch.

### 2.1 Disposition of all fifteen repositories

Every repository in the organisation is named here, as **active** or **to be archived**, with
the reason. There is no residual category.

**Active — 6.**

| Repository | Role | Why it is not folded in |
|---|---|---|
| `platform` | The Cargo workspace: every VulcanFlow Rust crate, one toolchain, one lockfile, the CI gates. Today it holds `.gitattributes` and the ADR-0005 lane gate; the crates arrive with the rebuild. **How many and which is the open member-set question of §1.2, not a layout question, and nothing in this record authorises a count** | It *is* the workspace |
| `docs` | The design of record — the TDD, these ADRs, `process/`, and `plans/` once `docs#24` lands (not on `main` today) | Markdown. Holds no `Cargo.toml` |
| `infra` | Kubernetes manifests and GitOps configuration (Argo CD/Kargo, CNPG, ClickHouse, Valkey, Keycloak, secureCodeBox, Harbor) | Declarative YAML/Kustomize/Helm, not a `cargo` target (TDD §2.5.3, *"SQL migrations, Kubernetes manifests"*). **It holds nothing today:** `infra#1` was closed unmerged at 2026-10-01T19:29:11Z — five seconds after `platform#1`, in the same purge — its `factory/phase0-foundation` branch is deleted, its head survives as tag `archive/pr-1-phase0-gitops` (`ed1b42cf`), and `infra@main` is the single commit `304b300e` *"Initialize main"* over the **empty tree** `4b825dc6`. The Phase 0 GitOps work is to be re-authored through the pipeline, and is separately gated on the CEO cluster decision. None of that bears on the disposition: `infra` stays active because it holds no `Cargo.toml` and never will (§2) |
| `vf-web` | The browser SPA | TypeScript/React. TDD §2.5.3 states it is explicitly **not** rewritten |
| `scanners` | Custom arm64 secureCodeBox scanner images, parsers, conformance | Container builds wrapping upstream Go/C tools. TDD §2.5.3: not rewritten |
| `vf-aigw` | Self-hosted OpenAI-compatible AI gateway | LiteLLM, operated as an approved third-party Python service. TDD §2.5.3: not rewritten |

**To be archived — 9.** Each would otherwise be a second plausible home for code that belongs
in `platform`. The ground differs between them, and the difference matters: **eight** are
named after crates the archived manifest declared and the rebuild will re-author under
`platform/crates/`; the ninth, `vf-remediation`, is named after a component that has no crate
anywhere. Note the tense — `platform@main` holds **no `crates/` directory today** (§1.2), which
is exactly why §4.2 orders the archiving after it does.

| Repository | Where the code belongs instead | Content to migrate |
|---|---|---|
| `vf-api` | `crates/vf-api` (binary) | None. `main` is an empty-tree initial commit; the Go scaffold on `refs/pull/1/head` was retired by ADR-0001 |
| `vf-authz` | `crates/vf-authz` (**library**, not a service — TDD §2.3, §2.5.2) | None. No commits |
| `vf-translator` | `crates/vf-translator` (library); graph *validation* is the separate `crates/vf-graph` per TDD §7.3 | None. No commits |
| `vf-operator` | `crates/vf-operator` (binary); the admission surface is the separate `crates/vf-admission` | None. No commits |
| `vf-ingest` | `crates/vf-ingest` (binary) | None. No commits |
| `vf-meter` | `crates/vf-meter` (binary) | None. No commits |
| `vf-report` | `crates/vf-report` (binary) | None. No commits |
| `vf-abuse` | `crates/vf-abuse` (binary) | None. No commits |
| `vf-remediation` | **No crate of its own.** The remediation surface is types and queries in `crates/vf-core` and `crates/vf-db`; verification-rescan orchestration is in `crates/vf-api` | None. No commits |

`vf-remediation` is the one row that is a judgment and not bookkeeping, so it is stated
explicitly: **`vf-remediation` does not become a fifteenth crate without a later record that
says so.** TDD §2.5.2's thirteen-crate layout names no remediation crate, and TDD §2.3 lists
`vf-remediation` as a Phase 3 *component* whose logic the table above places in existing crates.
A reader should not infer a crate from the archived repository name — and because the member set
itself has no record yet (§1.2), the one to amend when that changes is the crate-set record, not
this one.

### 2.2 What this gives the program plan

[VUL-4](/VUL/issues/VUL-4) needs a per-phase in-scope / out-of-scope repository column.
This record supplies the **domain** of that column and leaves the phase assignment to the plan,
which is where phasing belongs (TDD §24):

- The repository set is **closed at fifteen**, each one active or archived with a reason (§2.1).
- Every VulcanFlow Rust crate maps to **exactly one** repository, `platform`. So "which
  repositories does phase *N* touch?" reduces to "which crates, plus which of the five
  non-`cargo` active repositories" — a lookup, not a judgment.
- The nine archived repositories are **out of scope in every phase, by construction**. The plan
  states that once by citing this record, not once per phase.
- Cross-repository artifacts are **build artifacts, not source**. `vf-graph` compiled to
  WebAssembly is consumed by `vf-web` (TDD §7.3, §14.1, §14.3); the WASM module crosses the
  repository boundary, the source does not. `vf-graph` is not moved into `vf-web`, and a future
  shared type does not justify a shared repository.

---

## 3. Alternatives considered, and why each lost

**A. One repository for the workspace — `vulcanflow/platform`. → Chosen.**
One build unit, one lockfile, one toolchain, one CI configuration, one place the ADR-0002
boundary and TLS-provider gates can assert a whole-graph property. `platform` is the name
because the thing is not a service: it is the platform the services are built from, and it is
the name that still reads correctly whatever the approved member set turns out to be, and
when the one after that lands.

**B. Promote one existing repository — put every crate under `vf-api`, or create
`vf-core` as a repository. → Lost.**
Every candidate name is a lie about the contents. A reader who finds `vf-meter`'s allowance
accounting inside a repository called `vf-api` has been actively misled, and every
per-repository issue and PR reference becomes wrong at the same moment. The option buys nothing
over A: it is A with a worse name and a confusing history.

**C. Keep the polyrepo, one workspace per crate. → Lost.**
This is the option ADR-0001 already closed; it is listed so the closure is visible from here
too. It produces nine lockfiles and nine toolchains and reinstates `vf-core` version skew across
the scope, allowance and finding crates (§1.3). It also makes the ADR-0002 purity boundaries and
the single-TLS-provider assertion unenforceable, because both are properties of one dependency
graph.

**D. Git submodules, or a `[patch]` arrangement across the existing repositories. → Lost.**
C with extra steps: the same skew, plus a second failure mode. A submodule pins a commit that
can be force-pushed away — which is exactly the objection `deny.toml` raises against git
dependencies, and it is not theoretical here, because ADR-0005 §7 item 8 records that force-push
refusal does not yet exist on any VulcanFlow repository. `[patch]` moves the version conflict
from "two lockfiles" to "one lockfile that lies about its sources".

**E. Fold the non-`cargo` repositories in too — a monorepo. → Lost, and not seriously
considered.**
It would put the design of record, the GitOps manifests that configure the cluster, and the
SPA in the same change surface as the crates. ADR-0005's lane gate partitions *paths*; widening
the repository to hold unrelated trees widens every classification decision with it. §2's
crate-level rule is narrower and needs no exceptions.

---

## 4. Consequences

### 4.1 Immediate, on this record landing

1. **The repository set is decided and closed.** Fifteen repositories, each dispositioned.
   A sixteenth is answered by the §2 rule without a new record.
2. `decisions/README.md` loses its *"ADR-0004 is not on `main` yet"* note, and the root
   `README.md`'s "Repository architecture (polyrepo)" section is marked superseded. That table
   still names Go, chi, Huma and Connect-RPC; **rewriting it is not this record's job** — the
   banner exists so the stale table cannot be read as a live decision in the meantime. It was
   `docs#20`'s job, and `docs#20` **was closed unmerged at 2026-10-01T18:50:51Z** in the same
   purge as `docs#23` (tag `archive/pr-20-tdd-v2.3-rename`), so the rewrite currently has no
   owner. Naming a closed PR as the owner of outstanding work is the defect this record was
   corrected for twice (§6 items 3 and 6); it is not repeated here. The rewrite needs a work
   item, and the banner holds until it gets one.
3. ADR-0005 §11's *"does not touch ADR-0001 through ADR-0004"* stops being a reference to an
   absent document.
4. **A counting collision becomes visible, and is resolved here rather than left to a reader.**
   This record calls **six** repositories active (§2.1); ADR-0005 §3.2 and §7, and
   `decisions/README.md`, say **four**. Both are right about different sets, and the
   distinguishing property is not content — ADR-0005 §7 item 5 says in terms that `infra` and
   `vf-api` are *empty* and get the gate *"the moment it receives source"*. ADR-0005's four
   (`docs`, `platform`, `infra`, `vf-api`) are the repositories with an **initialised default
   branch**: the ones a branch-protection or force-push rule could be applied to at all, which
   is what §3.2 and §7 item 8 are counting. The other eleven have **zero refs**, so there is no
   `main` to protect. §2.1's six are the repositories that **remain active after the fold**
   (`platform`, `docs`, `infra`, `vf-web`, `scanners`, `vf-aigw`). Note that `vf-api` is in
   ADR-0005's four and is one of the nine to be archived here — so the two sets are not nested,
   and "active" must be read with its qualifier. The fold reduces ADR-0005's four to three.

### 4.2 On the workspace reaching `platform@main` — the fold, owned by [VUL-6](/VUL/issues/VUL-6)

**The trigger is a state, not a pull request:** the fold runs once `platform@main` carries a
`Cargo.toml` whose `[workspace].members` is the workspace member set **then of record**.
There is no such record today — that is §1.2's gap, and closing it is a precondition of the
rebuild, not of this record. Whichever PR delivers that state satisfies the condition, and the
fold does not need to know which crates the record names, only that a record names them.

The original draft tied this to `platform#1` merging, and `platform#1` is now closed — stating the trigger as a state is what keeps this record true
across the rebuild, and is the correction worth generalising: *an ordering constraint should
name the condition it needs, never the change that happened to be carrying it.*

The archive step is **ordered after** that state, not before. Until the workspace is on
`platform@main`, archiving the nine would leave the organisation with no home for those crate
names at all. Since the crates are being re-authored rather than copied, the gap between this
record landing and the fold is now longer than the draft assumed; §4.3 item 9 is why that gap
costs little.

For each of the nine: **archive, do not delete.** Archiving makes the repository read-only and
preserves its issues, pull requests and refs — including `vf-api`'s `refs/pull/1/head`, which
ADR-0001 keeps for provenance and which deletion would destroy. Specifically, each archived
repository:

- becomes read-only: no pushes, no new issues or PRs, no branch creation;
- keeps every existing ref, issue and PR readable at its current URL;
- stays resolvable, so existing links in ADR-0001's appendix and in the `docs` issue history do
  not rot;
- is **reversible** — unarchiving is one action, which is what makes this the low-risk ordering.

Renaming or deleting instead would redirect or break those links, and GitHub's rename redirects
do not survive a later repository of the same name. Archiving is the only disposition that keeps
the history and removes the ambiguity.

### 4.3 Standing consequences

4. **No per-crate repository is created again.** A new crate is a directory under
   `platform/crates/` and a `members` entry. Creating a repository for it is a design failure,
   not a convenience.
5. **The CI gates are whole-graph assertions by construction.** `crate-boundaries`,
   `tls-provider-assertions`, `forbid-unsafe`, `reproducible-build` and `sbom` each assert a
   property of one dependency graph, and in one repository they see every crate. This is the
   property §1.3 says a split destroys, and it is the main thing the decision buys. The scripts themselves
   are not on `platform@main` today — they exist at `archive/pr-1-cargo-workspace` and the
   rebuild must re-establish them; this record fixes the topology that lets them be
   whole-graph, and does not claim they are currently running.
6. **Every crate in the workspace shares one lockfile, so one pin applies to all of them.**
   A dependency bump is a workspace-wide decision and needs an ADR-0002 amendment (ADR-0002 §1). There is no
   per-crate pin and no way to introduce one.
7. **Compile time is shared.** A touch to `vf-core` can rebuild its dependents. Accepted: the
   answer at this scale is `sccache` and per-crate CI scoping, not a repository split (§5).
8. **ADR-0005's lane gate needs to exist in exactly one Rust repository.** Its §7 item 5 records the
   gate is live on `platform` only, which under this record is complete rather than partial
   coverage. The five other active repositories hold no `Cargo.toml`, so they
   have no Rust lane to cross. Under §2's crate-level rule they never will: a Rust artifact they
   need is built in `platform` and consumed as an image or module. They would need their own
   lane classes only for their *own* languages — TypeScript in `vf-web`, manifests in `infra`.
9. **Archiving is the only enforcement available today.** ADR-0005 §3.2 and §7 item 8 record that
   this organisation is on the GitHub free plan with private repositories, so branch protection
   and rulesets are unavailable and `main` is directly writable and force-pushable everywhere.
   A read-only archived repository cannot be pushed to **at all**, which makes §4.2 the
   strongest enforcement of this decision that the current plan permits — and one more reason
   not to defer it. The plan upgrade stays escalated under ADR-0005 §8; this record does not
   depend on it.

### <a id="loose-end-adr-0002-amendment-a2"></a>4.4 Loose end: ADR-0002 amendment A2

Recorded here because it is a direct consequence of `docs#23` closing, and a reader comparing
this record with the archive will otherwise conclude it was dropped silently.

`docs#23` carried **ADR-0002 amendment A2 (§3.7)** in the same commit as ADR-0004. ADR-0002 on
`main` carries **A1 only**. A2 is a set of dependency-pin corrections and is therefore out of
scope for this record, which decides repository layout and pins nothing.

The consequence is live, not hypothetical, and `platform#1`'s closure made it **more** live
rather than less. That PR's workspace `Cargo.toml` stated its versions came from *"ADR-0002 §3
and amendments A1 (§3.6) and A2 (§3.7)"*, with **§3.7 the authority where they disagree** — and
§3.7 records that A2 corrects six feature names that did not exist or meant the opposite of what
§3 said, and withdraws two crates. §3.7 is not on `docs@main`.

So the rebuild is now scheduled to re-author the workspace against a pin set whose only
correct form lives in an archived tag. Were it merely a matter of merging a finished branch, the
gap would be caught at review; instead the rebuild will reach for ADR-0002 §3, find the six
wrong feature names, and either rediscover the corrections or ship them wrong.

This is Atlas's to re-file, as a separate record on a separate PR. It blocks the workspace
rebuild (the fold's precondition in §4.2) rather than this record, which pins nothing. The draft
text is recoverable from `archive/pr-23-adr-0004`, and every claim in it must be re-derived
against crates.io before it is re-filed, not copied.

**There are therefore two missing records between here and the rebuild, not one, and they are
different in kind.** A2 is missing *pin corrections* (above). §1.2's gap is a missing *member
list* — nothing on `docs@main` says how many crates the workspace has or what they are called.
The rebuild needs both, and **neither of them is this record's to supply**; both are tracked
under **Still open** in `decisions/README.md`.

Stated as an instruction, because this is the sentence a rebuild agent will act on:
**no count in this document is an authorisation.** The fourteen in §1.2 and in the §6
provenance table is a reading of an archived manifest, cited as evidence of the workspace's
*shape*, and it is one more than TDD §2.5.2's thirteen. The rebuild authors the member set
the crate-set record approves; if that record does not exist yet, the rebuild is blocked on
it rather than on a count recovered from a closed pull request.

---

## 5. What would make me revisit this

- **A crate that genuinely cannot share the lockfile** — a dependency incompatible at the
  workspace level that cannot be reconciled. That is an argument for a **second workspace**, and
  only then possibly a second repository; it is not an argument for one repository per crate.
  None exists today: ADR-0002 §3 resolves as a single graph, and §3.3 records the cross-check.
- **Build times at a scale this workspace is nowhere near.** A workspace of this order — a
  dozen-odd crates, whatever the approved member set turns out to be — is not that scale.
  The ordered answers are `sccache`, then per-crate CI scoping, then a split — in that order.
  **The number, so this trigger is checkable like the other four:** a cold workspace
  `cargo build --workspace --locked` over 30 minutes, or a warm incremental rebuild after a
  one-crate edit over 5 minutes, measured on the CI runner and sustained across a week. Below
  that, a split is an impression.
- **A crate that ships to a third party** on different licence or release terms. Everything here
  was `publish = false` and `LicenseRef-Proprietary` in `platform#1`'s manifest — an archived
  source, so treat it as the intended baseline rather than a current fact until the rebuild
  re-establishes it. A crate that stops being either reopens this.
- **An owner reversal of TDD §2.5.2's "single Rust build unit … one pinned toolchain and
  lockfile."** This record implements that sentence. If the sentence changes, this changes with
  it.
- **A §2.5.3 exception becoming VulcanFlow-owned Rust.** If `scanners`' parsers become Rust
  crates under §27 item 20, §2's crate-level rule folds **those crates** in automatically —
  that is the rule working, and it needs no new record. Note what it does *not* do: the
  `scanners` repository keeps its container builds and stays active, because the rule moves
  crates, not repositories. What *would* need a new record is a §2.5.3 component staying
  non-Rust while nonetheless needing to be inside the workspace.

**Not a reason to revisit:** that nine repository names now resolve to read-only archives
rather than to code, and someone expected to find code there. That is the decision working as intended, and §2.1 is the place it is
written down.

---

## 6. <a id="provenance"></a>Provenance

Everything asserted above was read on **2026-10-01**, in this order, and not recalled. This
record pins no dependency versions; the one version it cites is the toolchain channel, read from
the file that is authoritative for it.

| Claim | Read from |
|---|---|
| Fifteen repositories; names, default branches, archived flag, creation dates | GitHub REST `GET /orgs/vulcanflow/repos` |
| Eleven per-component repositories have **zero** refs | `GET /repos/vulcanflow/{name}/branches` → `[]`, and `GET …/git/trees/main` → `409 Git Repository is empty` |
| `vf-api` `main` is one empty-tree commit `45d7ded9` *"chore: initialize repository"*; the Go scaffold survives only as `refs/pull/1/head` `773357eb` | `GET /repos/vulcanflow/vf-api/git/refs`, `…/commits`, `…/contents` → `[]` |
| The fourteen workspace members, split 6 library / 8 binary | `Cargo.toml` `[workspace].members` at tag `archive/pr-1-cargo-workspace` (`66f4a91c`) in `platform` — `platform#1`'s preserved head |
| Toolchain channel **`1.98.1`**, components `rustfmt`/`clippy`, target `aarch64-unknown-linux-gnu`, profile `minimal` | `rust-toolchain.toml` at that same tag — the file ADR-0002 §2 makes authoritative. Read at a tagged, immutable ref, not recalled |
| `platform#1` named ADR-0002 §3.7 as the pin authority, and what A2 corrects (§4.4) | header comment of that same `Cargo.toml`; `platform#1` review thread, comment of 2026-10-01T12:21:26Z |
| `platform#1` closed unmerged `2026-10-01T19:29:06Z`, head branch deleted, head preserved at tag `archive/pr-1-cargo-workspace` / `refs/pull/1/head`; board chose rebuild-from-scratch on [VUL-1](/VUL/issues/VUL-1) at 19:07Z | `GET /repos/vulcanflow/platform/pulls/1`; its closing comment; `GET …/branches`; `GET …/git/refs/tags/archive/pr-1-cargo-workspace` |
| ADR-0002's gates are **seven `ci/` files — five scripts** (`crate-boundaries.sh`, `tls-provider-assertions.sh`, `forbid-unsafe.sh`, `reproducible-build.sh`, `sbom.sh`) **and two data files** (`pure-crate-dependencies.txt`, `tool-versions.env`) — and all seven exist only at the archive tag, not on `platform@main` | `GET /repos/vulcanflow/platform/git/trees/{main,archive/pr-1-cargo-workspace}?recursive=1`, compared |
| `infra#1` closed unmerged `2026-10-01T19:29:11Z`, head `ed1b42cf` on the now-deleted `factory/phase0-foundation`, preserved as tag `archive/pr-1-phase0-gitops`; `infra@main` is commit `304b300e` *"Initialize main"* over the empty tree `4b825dc6` and `contents` returns `[]` | `GET /repos/vulcanflow/infra/pulls/1`; `GET …/branches` → `["main"]`; `GET …/tags`; `GET …/commits`; `GET …/contents` |
| Eleven repositories have **zero branches**, so ADR-0005's "four" are exactly the four with an initialised default branch (§4.1 item 4) | `GET /repos/vulcanflow/{name}/branches` → `[]` for each |
| `docs#20` closed unmerged `2026-10-01T18:50:51Z`, archived as tag `archive/pr-20-tdd-v2.3-rename`; `docs#24` and `docs#26` are still open | `GET /repos/vulcanflow/docs/pulls/{20,24,26}`; `GET /repos/vulcanflow/docs/tags` |
| Eight of the twelve 2026-09-09 per-component repositories share a name with a crate in the archived manifest; `vf-remediation` does not, and neither do `scanners`, `vf-web`, `vf-aigw` | the org repository list above, intersected with the `[workspace].members` list in the row above |
| `platform@main` holds `.gitattributes` plus the ADR-0005 lane gate (`ci/lane-gate.sh`, `ci/lane-gate-test.sh`, `.github/workflows/lane-gate.yml`), merged as `platform#2` | `GET /repos/vulcanflow/platform/git/trees/main?recursive=1` |
| `docs#23` closed unmerged `2026-10-01T18:50:19Z`, no reviews, no comments; archived as tag `archive/pr-23-adr-0004`, commit `cee1edc` | `GET /repos/vulcanflow/docs/pulls/23`; `git tag -l` in `docs` |
| TDD §2.5.2's workspace-layout sentence: **thirteen** crates, 5 library / 8 binary, `[PROPOSED]`; the glossary wording; the §2.5.3 exception table; §21.2's Rust per-tenant migration binary; §2.3's `vf-remediation` Phase 3 row | `VulcanFlow_Technical_Design_Document_v2.2.md` on `docs@main` |
| ADR-0001 appendix row #14, and that `main` was empty in all 14 repositories | `decisions/ADR-0001-retire-go-scaffold-vf-api.md` on `docs@main` |
| ADR-0002 carries A1 only; A2 is not on `main` | `decisions/ADR-0002-rust-crate-set-and-phase0-pins.md` §10 and `decisions/README.md` on `docs@main` |
| **`vf-store` appears nowhere in ADR-0002**, and ADR-0002 §3 is the third-party pin set (§3.1 runtime/HTTP, §3.2 database/Kubernetes/object store, §3.4 test-side, §3.5 excluded, §3.6 crypto) — not a workspace member list | `grep -c vf-store` → **0**, and the §3 subsection headings, in `decisions/ADR-0002-rust-crate-set-and-phase0-pins.md` on `docs@main` |
| ADR-0005 §7 is a numbered list of eight consequences, so "§7.5"/"§7.8" are not literal sections; cited as "§7 item 5" / "§7 item 8" | `decisions/ADR-0005-delivery-pipeline-and-lane-enforcement.md` §7 on `docs@main` |
| All fifteen repositories return `archived: false` — none of the nine is archived yet | `GET /orgs/vulcanflow/repos`, `isArchived` field |
| ADR-0005 §3.2 / §4.2 / §7 items 5 and 8 / §8 / §11 (§7 is a numbered list, not subsections) | `decisions/ADR-0005-delivery-pipeline-and-lane-enforcement.md` on `docs@main` |

**Ten claims were corrected rather than carried forward — three by this audit (1–3), three by
the first hand review ([VUL-21](/VUL/issues/VUL-21), Assay, on commit `489dc7f`; items 4–6),
and four by the re-review of the fixes ([VUL-23](/VUL/issues/VUL-23), Assay, on commit
`98ac439`; items 7–10). Items 7–10 are all residue of the 3 and 4 fixes, which is the pattern
worth noticing: a withdrawn claim leaves downstream sentences that still assume it, and
withdrawing it in one place is not the same as retracting it.**

1. The draft said `platform`'s `main` *"carries a single `.gitattributes` commit so the PR had a
   base."* It no longer does — `platform#2` merged the ADR-0005 lane gate to `main` after the
   draft was written.
2. The draft described the nine as repositories that *"fold into `platform`"*, which reads as a
   migration. There is nothing to migrate: all twelve non-`cargo`/per-crate default branches are
   content-free (§1.3). The fold is archive-and-redirect only, and saying so is what makes the
   §4.2 ordering obviously cheap.
3. **Both the draft and this record's own first commit (`489dc7f`, 19:25:29Z) described
   `platform#1` as open.** It was closed four minutes later, at 19:29:06Z. Every claim that
   depended on that state has been restated: §1.2 records the closure and the rebuild decision,
   §4.2 makes the fold trigger a *state* of `platform@main` rather than a PR merging, §4.3
   item 5 no longer implies the ADR-0002 gate scripts are live, and §4.4's loose end is
   re-aimed at the rebuild. The decision itself did not move, which is the test a
   layout record should pass.
4. **The draft, and this record's first two commits, cited "ADR-0002 §3" as authority for the
   fourteen-crate member set. It is not.** §3 is the third-party pin set and contains no
   `vf-store`. Found by grep in hand review, not by this audit — the audit had re-read §3 for
   the *pin* claims and carried the member-set citation forward unchecked, which is exactly the
   failure a provenance table is supposed to prevent and did not. §1.2 now withdraws the
   citation and names the gap; `decisions/README.md` tracks it under **Still open**. This is the
   one substantive defect the review found, as opposed to a claim the world invalidated.
5. **The draft's rule was stated over repositories** — *"a repository joins the workspace iff it
   builds with `cargo`"* — which answers TDD §21.2's Rust migration binary in `infra` wrongly,
   by pulling the GitOps repository into the workspace. §2 now states it over **crates**. Also
   found in hand review.
6. **The root `README.md` banner announced the nine as *"archived"*.** All fifteen repositories
   return `archived: false`; the nine are writable today, and §4.2 defers the step on purpose.
   Announcing it as done would have removed the pressure to do it while §4.3 item 9 was
   simultaneously calling archiving the only enforcement the free plan permits. Found in hand
   review; the banner now says *"to be archived, and are not archived yet"*.
7. **§2.1 and §4.4 still instructed the rebuild to author *fourteen* crates** after item 4
   withdrew that number's authority — §4.4 said *"scheduled to re-author fourteen crates"* and
   §2.1's `platform` row said *"the fourteen crates arrive with the rebuild"*, while §4.2 said
   thirteen-plus-later-records in the same document. A rebuild agent reading §4.4 authors
   `vf-store`, which is the exact harm item 4 names. Both now defer to the member-set record,
   and §4.4 ends with the instruction in terms: **no count in this document is an
   authorisation.**
8. **§2.1's `infra` row asserted a branch that no longer exists** — *"its
   `factory/phase0-foundation` branch stays unmerged on purpose"*. `infra#1` closed five
   seconds after `platform#1` in the same purge; the branch is deleted and `infra@main` is an
   empty tree. This is item 3 again, one row below, and the §6 table carried no provenance row
   for `infra`, so the *"read and not recalled"* claim over-reached. Both fixed.
9. **"Nine named after crates" was eight**, and §2.1 said they *"now live in
   `platform/crates/`"* when `platform@main` holds four files and no `crates/`.
   `vf-remediation` is the ninth and its own row says it has no crate — so the preamble made
   the inference the section exists to block. §1.2 and §2.1 now split the eight from the ninth
   and state the tense.
10. **The rule item 5 withdrew was still operative in two places** — §3 alternative E and §5
    both still read *"the if-and-only-if-`cargo` rule"*, which over repositories would pull
    `scanners`' container builds into the workspace. Both now cite §2's crate-level rule, and
    §5 says explicitly that the `scanners` repository stays active.

**Not verified, and deliberately so:** nothing here was compiled. The crate set is read as a
manifest, not as a successful build, and that manifest is now historical. §25
`build/rust-supply-chain` is the identifier that closes that gap; it is Crucible's to run
against the rebuilt workspace, not against the archived tag. This record does not depend on the
outcome, because the repository layout is the same whatever the build says.

---

## 7. A convention this decision is worth remembering for

Recovered from the `docs#23` draft because it is durable, and restated without the predecessor
board's issue numbers.

The `platform` repository was created mid-task rather than by blocking on a decision, with more
than twenty issues waiting behind it, and the agent said so in its handoff. That was the right
call: a repository rename is one action with automatic redirects, so a reversible action
taken-and-reported beat a week of waiting. This record ratifies the name rather than renaming
after the fact, which is the point.

Recorded in the same spirit, because it went the other way: while establishing whether creating
a repository was permitted at all, that agent created and deleted an empty probe repository
(`vulcanflow/vf-workspace-probe-delete-me`, no content, deleted about a minute later) and
disclosed it unprompted. The disclosure is the behaviour we want. The action is not — the
permission question was answerable by reading the instructions, and the organisation is not a
test fixture.

**The convention, for everyone: do not probe a permission against live infrastructure.** If
reasoning does not settle whether an action is permitted, ask. If the action is reversible and
the cost of waiting is real, take it deliberately and report it — which is what happened with
`platform` itself.

---

## 8. Amendment history

None. Amendments to this record are added here with a date and a section reference, never by
silently editing the text above (`decisions/README.md`, "Amendments").
