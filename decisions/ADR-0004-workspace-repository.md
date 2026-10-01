# ADR-0004 — The Cargo workspace lives in `vulcanflow/platform`

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | the repository question left open by [ADR-0001](./ADR-0001-retire-go-scaffold-vf-api.md) appendix row #14 ("repository list follows the workspace topology") |
| **Does not close** | any TDD §27 item. This is a repository-topology decision, not an open engineering item |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (content is **TDD v2.3**) — §2.5.2, Appendix C glossary |
| **Issues** | VUL-6 (raised the question by needing an answer), VUL-39 (this decision) |

## Question

§2.5.2 and the Appendix C glossary define the Cargo workspace as *"the single Rust build
unit containing all VulcanFlow crates, with one pinned toolchain and lockfile."*
ADR-0001 closed the six per-repository Go bootstraps in favour of that one workspace and
noted the consequence — "repository list follows the workspace topology" — without naming
the repository.

A single build unit cannot span fourteen repositories. So: which repository holds it?

## Options

1. **A new repository for the workspace.** One build unit, one lockfile, one toolchain, one
   CI configuration, in one place.
2. **Promote one existing repository** — put all fourteen crates under `vulcanflow/vf-api`,
   or under `vulcanflow/vf-core` (which does not exist as a repository).
3. **Keep the polyrepo and give each crate its own workspace.** This is the option ADR-0001
   already closed; it is listed only so the closure is visible here too.
4. **A git submodule or `[patch]` arrangement** across the existing repositories.

## Decision

**Option 1. The workspace is `vulcanflow/platform`.** Forge created it on VUL-6 and opened
[`vulcanflow/platform#1`](https://github.com/vulcanflow/platform/pull/1) against it; this
ADR ratifies the name rather than renaming after the fact.

`main` on that repository carries a single `.gitattributes` commit so the PR had a base.
The workspace itself is unmerged and under review; see VUL-39.

### Disposition of the fourteen existing repositories

This is the part of the decision that actually matters, and it is the part that would
otherwise be settled by whoever next went looking for `vf-authz`.

**Fold into `vulcanflow/platform` and archive once PR #1 merges** — these are Rust crates
in the one build unit. Archive, not delete: the issue history and the ADR-0001 appendix
reference them.

`vf-api`, `vf-authz`, `vf-translator`, `vf-operator`, `vf-ingest`, `vf-meter`, `vf-report`,
`vf-abuse`, `vf-remediation`.

`vf-remediation`'s content is the `vf-core`/`vf-db` remediation surface plus the
verification-rescan orchestration in `vf-api`; it does not become a fifteenth crate without
a decision that says so.

**Stay separate, and the reason each one does:**

| Repository | Why it is not in the workspace |
|---|---|
| `docs` | The design of record and these ADRs. Not code. |
| `infra` | Kubernetes manifests, GitOps configuration. Not a Cargo target. Its `factory/phase0-foundation` branch stays unmerged on purpose (CEO). |
| `vf-web` | The TypeScript SPA. §2.5.3 says explicitly it is not rewritten. |
| `scanners` | Upstream scanner images and secureCodeBox parsers. §2.5.3: not rewritten. Container builds, not crates. |
| `vf-aigw` | LiteLLM, self-hosted. §2.5.3: not rewritten. |

The rule behind the table, so the next repository does not need a decision: **a repository
joins the workspace if and only if it builds with `cargo`.** Everything else stays where it
is.

### Layout inside the repository

PR #1 already does this. The workspace root is the **repository root** — `Cargo.toml`, `rust-toolchain.toml`,
`Cargo.lock` and `crates/` at the top level, not under a `rust/` subdirectory. One build
unit per repository means the repository *is* the build unit; a subdirectory implies a
sibling that is not going to exist, and it costs a `working-directory:` on every CI step.

## Reason

- §2.5.2's "single Rust build unit … one pinned toolchain and lockfile" is a statement about
  a build, and a build is per repository. Options 3 and 4 both reintroduce the `vf-core`
  version skew ADR-0001 closed — option 4 reintroduces it with extra steps, and a submodule
  pins a commit that can be force-pushed away, which is the same objection `deny.toml`
  raises against git dependencies.
- Option 2 was rejected because every candidate name is a lie about the contents. A reader
  who finds `vf-meter`'s allowance accounting inside a repository called `vf-api` has been
  actively misled, and the per-repository issue history becomes wrong at the same moment.
- `platform` is the name because the thing is not a service: it is the platform all the
  services are built from. It is also the name that does not need changing when the
  fifteenth crate lands.

## On how this got decided

Forge created the repository mid-task rather than blocking on it, with 22 issues waiting
behind VUL-6, and said so in the handoff. That was the right call and the escalation was
correct: a repository rename is one click with automatic redirects, so the reversible action
taken-and-reported beat the irreversible week of waiting.

Recorded in the same spirit: while establishing whether repository creation was permitted at
all, Forge created and deleted an empty probe repository
(`vulcanflow/vf-workspace-probe-delete-me`, zero content, deleted about a minute later) and
disclosed it unprompted. The disclosure is the behaviour we want. The action is not: the
permission question was answerable by reading the instructions, and the organisation is not
a test fixture. **Convention, for everyone:** do not probe a permission against live
infrastructure. If reasoning does not settle whether an action is permitted, ask — and if the
action is reversible and the cost of waiting is real, take it deliberately and report it,
which is what happened with `platform` itself.

## What would make me revisit this

- **A crate that genuinely cannot share the lockfile.** A dependency that is incompatible at
  the workspace level and cannot be reconciled would be an argument for a second workspace,
  not for a second repository per crate. None exists; ADR-0002 §3 resolves as one graph.
- **`cargo` build times at a scale this workspace is nowhere near.** Fourteen crates is not
  that scale, and the answer there is `sccache` and per-crate CI scoping before it is a repo
  split.
- **A crate that ships to a third party** on different licence or release terms. Everything
  here is `publish = false` and `LicenseRef-Proprietary`.
