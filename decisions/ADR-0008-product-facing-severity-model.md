# ADR-0008 — The product-facing severity model

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-02 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Decided by** | The board, on the VUL-139 decision card, 2026-10-02T09:40:24Z. Option 1 — derive at ingest. The non-CVE fallback was answered *"CEO's call in the ADR"*; CEO exercised that delegation and fixed the fallback and the parameters in the `severity-model-decision` document on the same card |
| **Decides** | Where the `severity` on a VulcanFlow finding comes from, which values it may take, what every finding must carry so the value can be recomputed, and what a report may claim about it |
| **Closes** | **No TDD §27 item.** The severity model was open by omission — §27 has no entry for it (§7.2). This record discharges the *"Needs an owner"* row in `decisions/README.md` "Still open" and the deferral in ADR-0006's **Does not close** |
| **§25 identifiers** | **None covers the ladder.** §25 has no identifier for ingest-time severity derivation; §7.4 names the five existing identifiers this record bears on, and names the gap rather than minting an identifier here |
| **Depends on** | [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) **A1 §7.2** — the artifact cannot carry `CRITICAL`, and §3.5's remedy does not recover it (PR docs#26). [ADR-0006](./ADR-0006-false-positive-equivalence-and-alias-semantics.md) **§8.4.5** — severity is excluded from the equivalence key, which is the only interaction between the two records (PR docs#26) |
| **Bears on** | TDD §6.3 (`findings`), §10.3 (enrichment), §10.5 (the severity index), §16.2–§16.3 (reports), §24.2 (Phase 1 scope). Corrections recorded in [`README.md`](./README.md) |
| **Model version** | `severity-model/1.0` — §6 |
| **Issues** | VUL-139 (the board decision), VUL-165 (this record) |

---

## 1. The question, the options, and the answer

### 1.1 The question

**Where does the `severity` on a VulcanFlow finding come from, and what does a report assert when
it prints it?**

Two sub-questions travel with it and are answered here because leaving either open would leave the
first one undecided in practice: **which enum the product uses**, and **what severity a finding gets
when there is no CVE to enrich** — which is most of what Phase 1 emits.

### 1.2 What made it a question

Three facts, each read from upstream at the pinned release rather than recalled. The release is
secureCodeBox **v5.9.0**, the pin in ADR-0002 §6.

1. **The ingest envelope has four severity values and `CRITICAL` is not one of them.**
   `parser-sdk/nodejs/findings-schema.json` at tag `v5.9.0` constrains `severity` to
   `enum: ["INFORMATIONAL", "LOW", "MEDIUM", "HIGH"]`, and lists `severity` in the object's
   `required` array. **No schema-conformant secureCodeBox parser can emit `CRITICAL`**, for any
   scanner, ever, at this release.
2. **The nuclei parser destroys the value before the artifact is written.**
   `scanners/nuclei/parser/parser.js` at the same tag calls `getAdjustedSeverity` on every finding,
   which maps `CRITICAL → HIGH`, `INFO → INFORMATIONAL`, `UNKNOWN → LOW`. The original value is not
   preserved in `attributes` or anywhere else. Recorded as ADR-0003 A1 §7.2 and ADR-0006 §8.4.5.
3. **We ask for the templates whose severity we could then never report.** TDD §7.1's default
   pipeline configures nuclei with `"severity": ["medium","high","critical"]`. So the product
   selects critical templates, and the parser collapses every critical result to `HIGH` on the way
   out.

Fact 3 is the one that states the problem better than any argument: under a pass-through model
VulcanFlow would deliberately scan for critical vulnerabilities and be structurally incapable of
reporting one.

A fourth fact decided which option was actually cheap. TDD §6.3's `findings` table already carries
`cve_ids`, `cwe_ids`, `cvss_vector`, `cvss_score`, `epss_score`, `in_kev boolean NOT NULL DEFAULT
false` and **`enrichment_snapshot jsonb NOT NULL`**. That last column is non-nullable: **there is no
legal finding row without an enrichment snapshot.** The data model already assumed ingest-time
enrichment. What was missing was a sentence saying the severity comes out of it.

### 1.3 The options, with their costs

| | Option | What a report asserts | Cost |
|---|---|---|---|
| **1** | **Derive at ingest** from §10.3's enrichment; retain the scanner's value as evidence | VulcanFlow's own assertion, recomputable from recorded evidence | The §10.3 daily feed mirror and the ingest-time computation must exist **before the first customer-visible report** (§7.3) |
| **2** | **Pass through** the scanner's severity, four values, and present it as ours | Whatever the parser said, relabelled | Cheap to build; `critical` unreachable (§1.2 facts 1–3); a scanner swap silently re-scores the estate; leaves `enrichment_snapshot NOT NULL` with nothing to put in it |
| **3** | **Present both** — scanner severity and a derived score — side by side, neither authoritative | Nothing; the reader arbitrates | No decision is made, two numbers to explain in every dispute, and the remediation queue has no sort key |

Option 2 was never the cheap one. It is cheap to *implement* and expensive to *own*: the column that
makes it cheap (`severity` straight from the artifact) sits next to a `NOT NULL` column it never
fills, and the first customer who asks "why is this 9.8 only `HIGH`" gets no answer we authored.

### 1.4 The answer

**The board took option 1 on 2026-10-02T09:40:24Z**, and answered the second question — whether the
proposed non-CVE fallback stood — with *"CEO's call in the ADR."* CEO exercised that delegation and
fixed the fallback (§3, rules 4–7) and the three parameters (§4).

**A finding's `severity` is VulcanFlow's assertion**, computed at ingest from the enrichment evidence
in TDD §10.3. **The scanner's severity is evidence**: retained in its own column, shown as
provenance, never the number the report asserts.

### 1.5 Where the decision text lives, and what wins

The board card and CEO's `severity-model-decision` document on VUL-139 are the **drafts this record
was written from**. This record is the design of record. Where they differ, **this record wins** —
which is the whole reason a decision does not stay in an issue thread.

Four things in the draft are corrected here **on their facts, not re-decided**. Three are
cross-references: the `findings` table is TDD **§6.3**, not §8; the nuclei template list is **§7.1**;
and §25 has no identifier to cite (§7.4). The fourth is a measured quantity in P1's reasoning, and it
is in **§4.1** — the threshold itself is unchanged.

---

## 2. Two enums, side by side

The most likely way this record gets misread is a reader meeting the four-value enum and assuming it
is ours. So both are stated here, adjacent, once.

| | **The product enum — ours** | **The ingest envelope enum — not ours** |
|---|---|---|
| Values | `informational` `low` `medium` `high` `critical` | `INFORMATIONAL` `LOW` `MEDIUM` `HIGH` |
| Count | five | four — **no `CRITICAL`** |
| Case | **lowercase** | uppercase |
| Defined by | this record | `parser-sdk/nodejs/findings-schema.json` at secureCodeBox `v5.9.0`, where `severity` is also `required` |
| Where it appears | REST, SSE, report JSON, `findings.severity`, the graph DSL's node config | the secureCodeBox findings artifact, and nothing downstream of ingest |
| Who may widen it | an amendment to this record (§6.1 MAJOR) | upstream, at a release boundary (§9 R5) |

**Lowercase is not a new convention; it is the existing one.** TDD §9.2's `finding.new` event already
carries `"severity":"high"`, and §7.1's graph config already carries
`"severity": ["medium","high","critical"]`. Uppercase appears on a VulcanFlow surface only when we
quote a secureCodeBox artifact verbatim as evidence — which is exactly what `scanner_severity`
(§5.1) is for.

### 2.1 `findings.severity` gets a `CHECK`

TDD §6.3 declares `severity text NOT NULL` with **no `CHECK`**. That is how a sixth spelling arrives:
one parser mapping, one migration, one hand-written insert, and `MED` or `Critical` is in the table
with nothing refusing it. The five values are load-bearing on §10.5's `(tenant_id, severity,
observed_at)` index and on §16.2's "findings by severity" buckets, so a stray value is not cosmetic —
it is a bucket nobody renders and a row nobody sorts.

**Required:** a `CHECK` over the five product values on `findings.severity`, written in the style
§6.3 already uses for `finding_states.state`:

```sql
severity text NOT NULL CHECK (severity IN
  ('informational','low','medium','high','critical')),
```

The `[PROPOSED — v2.3]` rule in §6.3 — text-typed state columns map to `vf-core` Rust enums with
tested conversions, and an unknown value read from the database is a hard error, not a silent
default — applies to `severity` as it does to `state`. The `CHECK` and that rule are two halves of
one invariant: the database refuses the write, and the reader refuses the read.

---

## 3. The ladder

**First matching rule wins, evaluated 1 → 7.** This is the authoritative statement of the rules;
everywhere else in this record points at it.

| # | Condition | Product severity | `severity_source` |
|---|---|---|---|
| 1 | A CVE on the finding is in CISA KEV | `critical` | `kev` |
| 2 | A CVE on the finding has EPSS ≥ `epss_critical_threshold` | `critical` | `epss` |
| 3 | A CVE on the finding has a CVSS base score | `high` (≥ 7.0) / `medium` (≥ 4.0) / `low` (> 0) — **never `critical`** | `cvss` |
| 4 | 1–3 unresolved, and the row is **asset/discovery output** (`subfinder`, `dnsx`, `httpx`: a resolved name, a live record, a detected service) | **not a finding** — inventory per §10.1. If a tool emits one into the findings stream: `informational` | `inventory` |
| 5 | 1–3 unresolved, and the finding class is **asserted by VulcanFlow itself**, not by a scanner | the value its lane-1 class spec declares, **capped at `high`** | `class_spec` |
| 6 | 1–3 unresolved, and the finding is **scanner-asserted** (misconfiguration, exposure — most nuclei templates) | the scanner's declared severity, **capped at `high`**, flagged `unenriched` | `scanner_unenriched` |
| 7 | Nothing above matches | `informational` | `default` |

### 3.1 The load-bearing rule: `critical` means exploitation evidence

**Rules 1 and 2 are the only routes to `critical`. Nothing else reaches it.** Not a CVSS score, not
a scanner's opinion, not a first-party class spec (§3.7), not a human override (§4.2).

That is a position defensible in a sales conversation and in a customer dispute, and it is precisely
what a scanner pass-through can never produce — see §1.2 fact 3. Every cap and exclusion elsewhere in
this record exists to hold this one sentence.

### 3.2 Rule 3 diverges from the CVSS qualitative scale, deliberately

CVSS v3.1's own qualitative rating scale — Table 14 of the v3.1 specification document, read at
`first.org/cvss/v3.1/specification-document` on 2026-10-02 — is `None 0.0`, `Low 0.1–3.9`,
`Medium 4.0–6.9`, `High 7.0–8.9`, `Critical 9.0–10.0`.

Rule 3 matches it at every boundary except the top one. **9.0–10.0 reads `high` for us, not
`critical`**, and that single divergence is the whole of the difference.

It is stated in the open because a customer who knows CVSS will ask. The answer is that the two
scales assert different things: **our `critical` asserts exploitation; CVSS's asserts consequence.**
A 9.8 that nobody is exploiting is not what belongs at the top of a remediation queue. So we print
the CVSS score, the vector and **CVSS's own band verbatim** next to ours (§5.1), and never present
one as the other.

Two readings follow from the rule as written, both intended:

- A base score of **0.0** does not resolve rule 3 — `> 0` is false and no band applies — so
  evaluation continues at rule 4. This is well defined only because of §3.6's wording.
- Rule 3's condition is *has a CVSS base score*. A CVE with a vector but no base score does not
  resolve it either, and also continues at rule 4.

### 3.3 More than one CVE on a finding

`findings.cve_ids` is `text[]`. **Evaluate rules 1–3 per CVE and take the strongest outcome**, and
record in `enrichment_snapshot` which CVE drove it (§5.1). A finding is as exploitable as its most
exploitable component.

### 3.4 Rule 6 is a cap, not a pass-through

The scanner's number is used, labelled as the scanner's, flagged `unenriched`, and **cannot exceed
`high`** whatever the scanner says.

Today that cap is almost inert: v5.9.0 cannot emit `CRITICAL` at all (§1.2 fact 1). It is written
for the scanner swap that ADR-0006 §7's version-gate machinery exists to survive. **The first tool
that does emit a fifth value must not be able to mint a `critical` without exploitation evidence.**
The cap is the enforcement point for §3.1, not a hedge — which is why R5 (§9) says to re-read the
cap rather than the enum when such a scanner is adopted.

### 3.5 Rules 4–7 key on "rules 1–3 did not resolve", never on "no CVE"

This is the wording, and it is load-bearing. **Rules 4–7 key on *"rules 1–3 did not resolve"*.**

Conditioning them on the finding having *no CVE* leaves a hole. A finding **with** a CVE that is not
in KEV, has no EPSS score and has no CVSS base score — a reserved or not-yet-scored CVE, which is
routine for recent ones — matches none of rules 1–3, and under the *"no CVE"* wording also fails the
test on rules 5 and 6. It falls through to rule 7 and is reported `informational`.

That is the silent degradation this fallback exists to prevent, reintroduced by the wording of the
fix. Under the correct wording an unscored CVE on a scanner-asserted finding lands on **rule 6** —
the scanner's number, capped, flagged `unenriched` — which is the right answer and what the ladder
was meant to say. §3.2's two readings depend on this wording too.

### 3.6 Rule 5 — findings VulcanFlow asserts itself

There are three origins for a finding, not two. A scanner said it; an asset was discovered; or
**VulcanFlow derived it from its own data**. TDD §27 item 13's **dangling-resource / takeover class**
is already sitting in the third: VulcanFlow derives it from its own discovery data, so it has no CVE
and no scanner severity.

Without rule 5, a confirmed subdomain takeover reaches rule 7 and is reported `informational`. That
is the single worst output this ladder could produce.

So a first-party finding class gets **the severity its lane-1 class spec declares, capped at
`high`**, with `severity_source = class_spec` and the class-spec version in provenance (§5.1).

**The route to `critical` for such a class is a named amendment to this record** (§9 R6), and that
amendment must cite the demonstrated-exploitation evidence the class's spec requires before it may
assert it. §3.1 holds either way: a class spec cannot reach `critical` by declaring it.

**§27 item 13 is named here as the first candidate, and is not resolved here.** Its owner is
Engineering and its open question is exactly *"what is the stronger evidence required for a confirmed
dangling-resource finding"*. That question has to be answered before the class can earn `critical`,
and answering it is not this record's to do (§8.4).

---

## 4. The parameters

Three, fixed. P1 is **versioned** — it is part of the severity model version (§6.1), so changing it
is a MAJOR bump and is visible in every report that follows.

### 4.1 P1 — `epss_critical_threshold` = **0.10 probability**

**Probability, not percentile.** A named, versioned parameter, **reviewed quarterly**, never changed
silently. The value in force at ingest is recorded in each finding's `enrichment_snapshot` (§5.1), so
a finding scored under an earlier threshold still explains itself.

Why 0.10: EPSS is a 30-day exploitation probability, and the bar is set to keep `critical` a list a
customer can work through in a week. That is the only thing that makes the label mean anything
operationally, and it is an empirical claim rather than an axiom — **R1 (§9) is the measurement that
tests it**, in both directions.

**One figure in that reasoning is corrected here on its facts.** The decision document characterises
0.10 as *"roughly the top couple of per cent of scored CVEs"*. Measured against the published score
set — `epss_scores-current.csv`, model `v2026.06.15`, score date `2026-10-01T12:00:22Z`, read
2026-10-02 — **17,275 of 381,682 scored CVEs score ≥ 0.10, which is 4.53%**, about twice the stated
share. **The parameter is unchanged at 0.10**; what changes is the strength of the volume argument
behind it, and this is the honest version of it. Two things keep that from being a problem: the bar
applies only to CVEs *on our own findings*, not to the whole corpus, and **R1 is the control** —
observed `critical` volume per tenant is the measurement that decides whether 0.10 holds, and it now
has a reason to be taken seriously at the first quarterly review rather than treated as a formality.

**A percentile bar was considered and rejected** (§10.3). Note that the correction above is a
statement about the corpus, which is exactly the quantity §10.3 says not to set a threshold on; it is
used here to size the parameter's effect, not to define it.

### 4.2 P2 — human override: yes, and the derived value is retained

A named user may override a finding's severity. Three conditions, all required.

1. **`findings.severity` keeps the derived value.** The override is a separate record keyed by
   finding, carrying `actor_id`, a timestamp and a **required** reason. Presentation resolves the two
   and the report shows **both**, with the override labelled as a human judgement. Overwriting the
   derived value in place would destroy the only audit trail the model has, and would break §5.2:
   the number would no longer be recomputable from the evidence.
2. **The reason is required, not optional.** An unexplained override is indistinguishable from a bug
   in the ladder — and R2 (§9) counts overrides as evidence about the ladder, which only works if
   each one says why.
3. **An override does not survive into a later observation.** TDD §10.4 already says a new scan
   produces a fresh `new` observation inheriting no state; severity follows that rule. An override is
   re-applied deliberately or not at all. **A severity override that silently persisted would be a
   suppression rule by another name**, and §10.2 is explicit that only an explicit false-positive
   decision creates one. §8.3 is where that boundary is left alone.

An override is a presentation-layer judgement, not a route to `critical` in the model: §3.1 is about
what *VulcanFlow asserts*, and the derived value remains what VulcanFlow asserts.

### 4.3 P3 — CVSS version precedence: **v3.1 → v4.0 → v2.0**, with the version recorded

Rule 3's band is read from **CVSS v3.1** where present. **v4.0, where present, is recorded as
evidence and does not set the band.** With no v3.1 score, use v4.0. With neither, and only v2.0,
use v2.0 and flag `cvss_v2_only`.

The reason is ranking, not quality. v4.0 is the better instrument, but its coverage is sparse, and
**banding part of a report on one scale and part on another produces a queue order nobody can explain
to the person working through it.** v3.1 is the near-universal denominator, so it is the denominator.

**Precedence flips to v4.0-first under a stated trigger — R3 (§9) — not under anyone's discretion.**

**Schema consequence.** TDD §6.3's `findings` has `cvss_vector` and `cvss_score` and **no version
column**, so a stored `7.5` does not say which scale produced it. `cvss_version` is **required**
(§7.1).

---

## 5. Provenance

### 5.1 What every finding carries

| Field | Content |
|---|---|
| `severity` | the asserted product value, one of the five (§2) |
| `severity_source` | `kev` / `epss` / `cvss` / `inventory` / `class_spec` / `scanner_unenriched` / `default` — **which rule fired** (§3) |
| `severity_model_version` | §6 |
| `scanner_severity` | the scanner's declared value, **verbatim and uppercase** |
| `cvss_version` | which scale set the band (§4.3) |
| `unenriched` | boolean; true for rule 6 |
| `scanner`, scanner version, template id, template version | partly present already: §6.3 has `scanner text NOT NULL` |
| `enrichment_snapshot` | already `jsonb NOT NULL`: the KEV catalogue as-of date, the EPSS model date and score, `cvss_version` / vector / score, **which CVE drove the outcome** (§3.3), the class-spec version for rule 5, and **the threshold value in force at ingest** (§4.1) |

**`scanner_severity` gets its own column.** §6.3 has `raw jsonb`, and *"dig it out of `raw`"* is not
a provenance claim a report can be built on: `raw` is per-scanner, unversioned and unindexed, and a
report that has to parse it in order to print a provenance line will eventually print the wrong one.

### 5.2 The rule that makes the columns worth it: a customer can reconstruct the number

**Given a finding's provenance, the asserted severity is recomputable.** The rule that fired, the
evidence it fired on, the snapshot dates, the threshold in force, the CVSS version, and the model
version — all recorded on the row, none of it requiring a live feed lookup.

That is what turns *"our severity"* from an opinion into something defensible line by line in a
dispute, and it is the entire case for option 1 over option 3 (§1.3). It is also the constraint that
decides §4.2 condition 1 and §6.2: anything that overwrites or re-scores in place destroys it.

### 5.3 Enrichment staleness is a stated condition, not an assumption

TDD §10.3 mirrors the feeds into Postgres on a daily job precisely so that enrichment never blocks on
a third-party endpoint. What it does not say is what the severity means when that mirror is behind,
and the honest answer is not *"nothing changes"*: **a KEV lookup against a stale mirror returns
`false` and is indistinguishable from a real negative.** Rules 1 and 2 degrade silently, and a
silently-degraded `critical` is the one failure this model cannot tolerate (§3.1).

- **Ingest does not block.** A feed outage never stalls a customer's scan.
- Findings computed while the mirror's as-of date is older than **48 hours** are flagged
  **`enrichment_stale`** with that as-of date.
- **Any report containing them says so** (§7.2).

Combined with §6 this is coherent: we do not silently pretend KEV said no, and we do not hold a scan
hostage to someone else's uptime. R4 (§9) is the same discipline applied to a feed that changes shape
or stops.

---

## 6. `severity_model_version`

### 6.1 What it binds

Format **`severity-model/MAJOR.MINOR`**, starting at **`severity-model/1.0`**. §6.3 is its changelog.

- **MAJOR** when the same evidence could now produce a different severity — a rule, a band boundary,
  the threshold (§4.1), the enum (§2), or the CVSS precedence (§4.3).
- **MINOR** when the change cannot alter any existing value.

### 6.2 No report is ever silently re-scored

Severity is **computed once at ingest and pinned with its snapshot**. EPSS moves daily, and a
severity that drifts under a customer with no new scan is not something anyone can act on — and it
would break §5.2, because the number on screen would no longer match the evidence recorded beside it.

Re-enrichment is an **explicit event** that produces a **new** report with a new model version. The
old report keeps its numbers and its version. This is the same shape as ADR-0006's staleness
handling, and the similarity is deliberate.

**The version lives on the finding; the report carries the set.** `severity_model_version` is a
column on `findings`, and **a report lists the distinct versions present** — and says so plainly when
there is more than one. A single version stamped on the report would be a claim about findings it
does not hold: TDD §16.4 resolves a report to concrete observations at a cutoff, and observations
ingested weeks apart can legitimately carry different model versions. The case where a report spans a
model change is exactly the case versioning exists for, so it is the case the field must get right.

### 6.3 Changelog

| Version | Date | Change |
|---|---|---|
| `severity-model/1.0` | 2026-10-02 | Initial model: the five-value enum (§2), the seven-rule ladder (§3), `epss_critical_threshold = 0.10`, override retention, CVSS precedence `v3.1 → v4.0 → v2.0` (§4). |

---

## 7. Consequences

### 7.1 The schema changes, in one place

Against TDD §6.3's `findings`, this record requires five new columns, one constraint, and a named
payload. They are listed here as the consequence; the correction entry that carries them into the
next TDD revision is in [`README.md`](./README.md).

| Change | Why |
|---|---|
| `scanner_severity` | §5.1 — provenance cannot live in `raw` |
| `severity_source` | §3 — which rule fired |
| `severity_model_version` | §6 — on the finding, not the report |
| `cvss_version` | §4.3 — a stored score that does not name its scale is ambiguous |
| `unenriched` | §3 rule 6 — the flag a report must surface |
| `CHECK` on `severity` | §2.1 — the five values are load-bearing |
| named contents of `enrichment_snapshot` | §5.1, §5.2 — recomputability needs the dates and the threshold, not just the scores |

`enrichment_stale` (§5.3) is derived from the snapshot's as-of dates against the 48-hour bound; it is
a report-surfaced condition, not a sixth column.

### 7.2 What the reports must now state

TDD §16.2's *"findings by severity"* is **five buckets**, not four. The §16.3 technical report lists
the `severity_model_version`(s) present (§6.2) and flags `enrichment_stale` (§5.3) and `unenriched`
(§3, rule 6) findings. §16.4's immutable `input-snapshot.json` already stores "enrichment and
guidance versions"; the severity model version belongs in that set.

### 7.3 Phase 1 — the cost the board accepted

Option 1 requires the enrichment path **before the first customer-visible report**: §10.3's daily
mirror of KEV, EPSS and CVE/CVSS into Postgres, plus the ingest-time computation in §3.

**§10.3's mirror is `[PROPOSED]` in the TDD. This record promotes it to required for Phase 1.** That
is the cost the board took when it took option 1, and it is recorded here as a consequence rather
than argued.

What keeps it shippable: **rules 4–7 need no feed.** A finding with no resolvable CVE never waits on
an enrichment lookup, and that is most of what Phase 1 emits — `subfinder` and `dnsx` produce no
CVEs, and most nuclei misconfiguration and exposure templates produce none either (ADR-0006 §4).
**The feeds gate `critical`, not ingest** — which is the same property §5.3 relies on.

The **scoping and sequencing** of that mirror against the rest of Phase 1 is a budget matter, not an
architecture one, and goes on its own Phase 1 issue. This record fixes what the mirror must support;
it does not schedule it.

### 7.4 §25 traceability — five identifiers bear on this, and none covers it

**No §25 identifier covers ingest-time severity derivation.** The matrix was written before this
question was asked and has no row for it. Stated rather than papered over, and **no identifier is
minted here**: a new §25 identifier creates a new required test, and the Phase 1 lane-1 spec for the
enrichment path (§7.3) is where it belongs, under the one-identifier-one-test-function rule.

The five existing identifiers this record bears on:

| §25 identifier | How this record touches it |
|---|---|
| `findings/new-per-scan` (§6.3, §10.4) | §4.2 condition 3 — a severity override does not survive into a later observation |
| `findings/replayed-artifact` (§8.4) | §6.2 — severity is computed once at ingest and pinned. Re-ingesting the same artifact must not produce a different severity |
| `findings/fp-only-persistence` (§10.2) | §4.2 condition 3 again, from the other side — an override must not become a suppression rule |
| `report/cross-consistency` (§16.4, §16.8) | §6.2, §7.2 — the version set, `enrichment_stale` and `unenriched` are part of one snapshot's self-description |
| `report/matrix` (§16.1–§16.4) | §7.2 — five buckets, in both audiences and both groupings |

### 7.5 One `critical` definition, enforced in three places

§3.1 is a single sentence, and three separate mechanisms hold it: rule 3 refuses the top CVSS band
(§3.2), rule 6 caps the scanner (§3.4), rule 5 caps a first-party class spec (§3.6). Each is a
different route that would otherwise reach `critical` without exploitation evidence. A change to any
one of them without the other two is a weakening of the definition, not a local tweak.

---

## 8. What this record does not claim

### 8.1 It does not close a §27 item, because there is none

TDD §27 has **no item covering the severity model**. It was open by omission — which is a worse way
for a question to be open than being listed, because nothing ever came up for review. Recorded here
the same way ADR-0005 and ADR-0007 record having no §27 item, and carried into `README.md` as a §27
correction so the next TDD revision closes the gap in the register rather than in this file.

### 8.2 It does not touch §10.3's configurable sort

§10.3's `KEV → EPSS → CVSS` ordering is *"exposed as a configurable sort"*. That orders a list. This
record fixes what `severity` **asserts**. Both stand, and they are different things: a user may sort
by whatever they like, and the number being sorted is still ours.

This is said explicitly because §10.3's one sentence about ordering is the nearest thing in the TDD
to a severity decision, and it is **not one** — which is how the question came to be open by omission
in the first place.

### 8.3 It is not a risk score, and it does not reopen §10.2 or ADR-0006

**Severity is one field.** A composite risk score — severity combined with asset criticality,
exposure, business context — is not decided here, and the ladder must not later be read as one.

**§10.2 and ADR-0006 are untouched.** False-positive equivalence is a different question about the
same rows. §4.2 condition 3 touches the boundary and **defers to §10.2** rather than moving it: only
an explicit false-positive action creates a reusable decision. ADR-0006 §8.4.5 excludes envelope
severity from the equivalence key either way, and that exclusion is the only interaction between the
two records.

### 8.4 It does not resolve §27 item 13

§3.6 names item 13's dangling-resource class as the **first candidate** for a `critical`-capable
first-party class. It does not pre-decide it. Item 13's owner is Engineering, its question is the
stronger evidence a confirmed dangling-resource finding requires, and under R6 (§9) that question
must be answered in a named amendment here before the class may assert `critical`. ADR-0006 §11
separately blocks any `dnsx/dangling-*` check class until item 13's equivalence row exists; nothing
in this record relaxes that.

---

## 9. Revisit triggers

| | Observable | What it means |
|---|---|---|
| **R1** | `critical` volume per tenant is more than a week's work, or is empty across a quarter | the threshold is wrong. Re-set P1 (§4.1), bump MAJOR |
| **R2** | human overrides exceed a quarter of `critical` findings in a quarter | **the ladder is wrong, not the users.** Read the required reasons (§4.2 condition 2) before changing anything |
| **R3** | ≥ 80% of CVEs on our own findings in a trailing quarter carry a v4.0 base score | flip P3's precedence to v4.0-first (§4.3), bump MAJOR |
| **R4** | KEV or EPSS changes publication shape, or ceases | rules 1–2 degrade to rule 3. **The failure mode is doing it silently**: the degradation bumps the model version and is stated in the report (§5.3, §6.2) |
| **R5** | a scanner that emits a fifth severity value is adopted | **re-read rule 6's cap, not the enum.** The cap is what holds §3.1 (§3.4) |
| **R6** | a first-party class wants `critical` | an amendment to this record naming that class's demonstrated-exploitation evidence (§3.6). **Never a code change alone** |

R1 and R2 are quarterly reviews with owners, not alarms: P1's quarterly review (§4.1) is where R1 is
measured, and R2 is measured in the same pass.

---

## 10. Alternatives considered, and why they lost

**Option 2 — pass the scanner's severity through.** Lost on §1.2: at the pinned release it cannot
produce `critical` for any scanner, and the default pipeline asks nuclei for critical templates. It
also fails §5.2 — there is no evidence trail, so no severity is defensible in a dispute — and it
leaves `enrichment_snapshot NOT NULL` (§1.2) with nothing to fill it. Its worst property is not the
missing value but the invisible coupling: a scanner upgrade re-scores a customer's estate with no
decision recorded anywhere.

**Option 3 — present both numbers, neither authoritative.** Lost because it is not a decision. Two
numbers must be arbitrated by the reader on every finding, the remediation queue has no sort key, and
in a dispute we have twice as much to explain and nothing to stand behind. §3.2's "print CVSS
verbatim next to ours" keeps everything option 3 was reaching for, with one number authoritative.

**An EPSS percentile bar instead of a probability.** Rejected. **A percentile moves when the corpus
moves**, so the same finding changes severity with no change in any evidence about it — which breaks
§5.2 and §6.2 together — and we would have no honest answer when a customer asks why. A probability
is a statement about the world; a percentile is a statement about the other rows.

**Overwriting the derived severity on a human override.** Rejected in §4.2 condition 1: it destroys
the audit trail and the recomputability rule in one write, for the sole benefit of one fewer join.

**A silently-optional override reason.** Rejected in §4.2 condition 2: it makes R2 unreadable.

**v4.0-first CVSS precedence now.** Rejected in §4.3 on coverage, not on quality — v4.0 is the better
instrument. Mixing scales within one report is worse than using the weaker scale consistently,
because it makes the queue order unexplainable. R3 is the trigger that flips it on evidence rather
than preference.

**Conditioning rules 4–7 on "no CVE".** Rejected in §3.5. It reads naturally and it sends
reserved-or-unscored-CVE findings — routine for recent CVEs — to `informational`.

**Letting a first-party class spec declare `critical` directly.** Rejected in §3.6. It would make
§3.1 a convention rather than a rule, and the first class to use it would be the one with the least
evidence behind it.

**Blocking ingest on a stale enrichment mirror.** Rejected in §5.3. It converts a third-party feed
outage into a customer-visible scan outage, which is exactly what §10.3's mirror exists to prevent.
Flagging is strictly better: the scan completes and the report tells the truth about what it knew.

**Stamping one `severity_model_version` on the report.** Rejected in §6.2. It is false whenever a
report spans a model change, which is the case the field was written for.

---

## 11. Relationship to the other records

- **ADR-0003 A1 §7.2** established the two facts this record is built on and explicitly left the
  product-facing model to a separate ADR. This is that ADR. Nothing in ADR-0003's decisions about
  the parser SDK, the hook language or the billing clients is touched.
- **ADR-0006 §8.4.5** excludes envelope `severity` from the false-positive equivalence key and names
  this record in its **Does not close** row. That exclusion holds under any outcome here, and it
  remains the only interaction between the two records (§8.3). ADR-0006 §7's scanner-version gate is
  what makes §3.4's cap worth writing.
- **ADR-0002 §6** is where secureCodeBox `v5.9.0` is pinned, which is the release every upstream fact
  in §1.2 was read at. **This record changes no pin.** If that pin moves, §1.2 facts 1–3 are
  re-verified against the new tag and R5 is consulted.
- **ADR-0005** governs how this record was produced: lane 1 is the spec, lane 6 is Assay and Warren,
  and nothing merges without both. This record is a lane-1 artifact and claims no lane-gate effect —
  `docs` is NEUTRAL class.
- **Both ADR-0003 A1 and ADR-0006 are on PR docs#26 and not yet on `main`.** The two records this one
  depends on are therefore themselves unmerged, which is stated here so a reader on `main` is not
  surprised by the dangling references. The dependency is on their *facts*, which were re-verified
  first-hand at `v5.9.0` for this record (§1.2) and do not wait on that merge.
- **Does not touch** ADR-0001, ADR-0004 or ADR-0007. No language, workspace-layout or process
  question is in scope.

---

## 12. Amendment history

| | Date | Change |
|---|---|---|
| — | 2026-10-02 | Accepted as recorded. No amendments. |

Amendments land here with an entry, never as a silent edit. Under §6.1 an amendment that could change
an existing severity is a **MAJOR** model-version bump and gets a row in §6.3 as well; R6 (§9) names
the one amendment this record already anticipates.
