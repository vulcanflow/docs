# ADR-0006 — False-positive equivalence fields, alias semantics, revocation, and scanner-version invalidation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01; revised 2026-10-02 in lane 6 review — see §15 |
| **Owner** | Atlas (Staff Architect / Tech Lead) |
| **Closes** | TDD **§27 item 6** |
| **Depends on** | [ADR-0002](./ADR-0002-rust-crate-set-and-phase0-pins.md) §6.1 — the secureCodeBox **v5.9.0** pin, which is the ref every field name below was read at; [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) §3 — parsers stay on the upstream JavaScript SDK, so the per-scanner `attributes` shapes are upstream's for `subfinder`/`nuclei` and ours for `dnsx`/`httpx` |
| **Does not close** | §27 item 13 (DNS risk labels / dangling-resource evidence); the scanner **image and template-pack digest pins** (§21.3, Phase 2); the product-facing severity model that falls out of §8.4.5 |
| **Corrects** | [ADR-0003](./ADR-0003-rpc-scb-parser-and-billing-clients.md) §3.2 and §3.5 — see ADR-0003 **amendment A1**, filed with this record |
| **Design of record** | `VulcanFlow_Technical_Design_Document_v2.2.md` (filename says v2.2; the content is **TDD v2.3**) — §6.2, §6.3, §10.1–10.5, §13.2, §21.3, §23.1, §23.5, §24.1, §25, §26 ("False positives"), §27 item 6 |
| **Issue** | VUL-10, routed from VUL-8 as register entry D16 |
| **Implementation** | `vulcanflow/platform` — key builder and matcher in `vf-core` (pure, no I/O); application at ingest in `vf-ingest`; the alias registry as a committed, CI-validated file. Owners in §9 |

---

## 1. The question, and what is already fixed

> **§27 item 6** (Engineering, blocks false-positive persistence): *Concrete scanner-specific
> equivalence fields, alias semantics, and examples proving an old decision cannot suppress
> unrelated findings?*

Three things constrain the answer before it is written, and this record holds all three rather
than reopening them.

- **§26 fixes the shape.** The "False positives" row reads: *narrow versioned target/check/location
  matcher with explicit decision revocation.* Narrow, versioned, revocable. That is the brief.
- **§10.2 fixes the floor and names the failure mode.** *"It is not sufficient to match a CVE or IP
  alone. Preserve hostname context on shared infrastructure. The stored matching input and mapping
  version explain why a decision applied."* And: *"A renamed check must not silently broaden an old
  false-positive decision."*
- **§6.3 fixes the storage.** `false_positive_decisions` already has `match_version integer` and
  `match_scope jsonb`; `findings` already has `false_positive_match jsonb` ("canonical matching
  input, not scan fingerprint") and `applied_fp_decision_id`; `false_positive_events.action` is
  already `CHECK (action IN ('revoked'))`. **This record specifies the contents of those columns. No
  new table and no new column is required for the match key itself** — but three migrations do fall
  out of it, and they are named rather than left for Phase 1 planning to discover:
  **(1)** `false_positive_events.reason text` for §6's recorded revocation reason, which §6.3 has
  nowhere to put; **(2)** the stored generated column or expression index §3.6 requires, either of
  which is DDL, on `false_positive_decisions` because that is the table matching probes **into**;
  **(3)** the same serialization as a generated column on the observation side,
  `findings.fp_match_key`, plus a covering index `findings (tenant_id, applied_fp_decision_id,
  fp_match_key)`. (3) exists because §6 makes the suppression-reach count **mandatory before a user
  confirms a decision** and §12 evaluates it per scan per decision, and because — per §6 — the unit
  of that count is the **distinct match key**, not the `findings` row. TDD §6.3 declares
  `applied_fp_decision_id` with no index at all, so without (3) a requirement this record calls
  mandatory is a sequential scan plus a distinct-sort over the tenant's findings on an interactive
  path. By §3.6's own standard — matching must not be a per-row scan — that is the same defect in the
  other direction. The original claim — *"requires no schema change"* — was broader than the
  evidence, was first narrowed to two migrations, and is now three; and the count of them is
  enumerated here rather than asserted at a distance, which is the lesson of §9.1.

What was missing is the only part §10.2 deferred: *which fields*, per scanner, for the
`subfinder → dnsx → httpx → nuclei` pipeline of §24.1. Without it `findings/fp-only-persistence` is
unwritable, because the test's job is to prove a bound and the bound did not exist.

**Why this is specified at ADR level and not left to the implementation.** A false-positive
suppression is the one place in VulcanFlow where the product tells a customer *"we looked at this and
it is not a problem."* If a decision travels to a finding nobody looked at, the product has lied on
the customer's behalf about their security posture. That is a defect in the same consequence class as
a tenant-isolation failure, and it gets the same scrutiny (**blast radius**). The direction of error
is therefore not symmetric and every judgment call below resolves the same way: **a matcher that is
too narrow produces noise; a matcher that is too wide produces silence. Prefer noise.**

---

## 2. The decision in summary

| # | Question | Decision |
|---|---|---|
| 1 | What is the equivalence key? | A **twelve-component versioned tuple** (§3.1), compared component-wise after a single canonicalization pass. Eleven components admit **byte equality only**; `check_id` admits byte equality **or** exactly one directed §5 alias edge, and is the only component that does (§3.1.1). `match_version = 1`. |
| 2 | Which fields, per scanner? | Named concretely in §4, read from the v5.9.0 parser source (`subfinder`, `nuclei`) and from the upstream tools' own output structs (`dnsx`, `httpx`, for which secureCodeBox v5.9.0 ships **no** scanner — §4.1). |
| 3 | What is "absent"? | A typed third state. Components are `Present(v)` / `NotApplicable` / `Unknown`. **Never SQL `NULL`, never `""`.** An `Unknown` in any component makes the key **non-storable and non-matchable** (§3.4). |
| 4 | Alias semantics | Aliases map `check_id` **only**; **directed, non-transitive, equal-or-narrower, evidence-bearing, versioned, cross-scanner forbidden**, evaluated at match time and never written back into a stored decision (§5). |
| 5 | Revocation | Append-only; affects **future** matching only; never rewrites a past applied record or a saved report; never reactivates (§6). |
| 6 | Scanner version bump | A bump alone invalidates nothing. A **declared field-semantics change** does: per-scanner `field_semantics_version`, bumped in the scanner-pin PR, **stales** every decision stored under the prior value — retained and visible, suppressing nothing (§7). |
| 7 | Evidence | Worked negative examples in §8 — **at least two per scanner**, each with the loose key that collapses the pair and the consequence of the collapse. The corpus is **enumerated by identifier in §9.1** rather than counted, because three successive drafts stated a count and all three were wrong (§8's preamble). Two cases (§8.4.1, §8.4.7) ship as **upstream fixture files** Ledger can use verbatim. |
| 8 | Schema | No new table or column for the match key itself; **three** migrations named in §1 — `false_positive_events.reason text` (§6), §3.6's generated column or expression index on `false_positive_decisions`, and `findings.fp_match_key` with a covering index `findings (tenant_id, applied_fp_decision_id, fp_match_key)` for §6's mandatory suppression-reach count, whose unit is the **distinct match key** and not the observation row. |

### 2.1 Acceptance statement

> A stored false-positive decision suppresses a later observation **if and only if** all four of the
> following hold: every component of the §3 match key **other than `check_id`** is byte-equal after
> §3.3 canonicalization; the two `check_id`s are byte-equal **or** exactly one directed alias edge
> admitted by §5 runs from the stored `check_id` to the observation's; the stored and current
> `field_semantics_version` for the producing scanner agree; and the decision is not revoked. Every
> other later observation is `new`.

---

## 3. The match key

### 3.1 Shape

`match_scope` (on the decision) and `false_positive_match` (on the observation) both carry this
object. `match_version` on the decision carries the integer in the first row.

> **`false_positive_match` is written on every observation whose key is storable, whether or not it
> matched anything.** TDD §6.3 calls the column *"canonical matching input, not scan fingerprint"* —
> it records the key we probed with, not the outcome of the probe, and its sibling
> `applied_fp_decision_id` is the column that records the outcome. So: key computed and free of
> `Unknown` ⇒ `false_positive_match` is written and `applied_fp_decision_id` is `NULL` on a miss;
> key containing `Unknown` (§3.4) ⇒ **both** are `NULL` and the observation is `new`. Writing it only
> on a hit would make §12's second revisit trigger — the rate of `new` observations differing from a
> suppressed one in exactly one sub-value — uncomputable, because the `new` observations are
> precisely the ones that matched nothing. A trigger whose input is not recorded is the same defect
> R12 was filed for, so the storage rule belongs here, next to the key, rather than being inferred
> from the trigger that needs it.

| Component | Type | Meaning |
|---|---|---|
| `match_version` | integer | **1.** The version of this specification. Bumped only by an ADR or an amendment to this one. A decision only ever matches an observation keyed at the same `match_version`. |
| `tenant_id` | uuid | Partition, stored for audit — see §3.2. |
| `scanner_id` | enum | `subfinder` \| `dnsx` \| `httpx` \| `nuclei`. Closed set; a fifth value requires an amendment adding its §4 row. |
| `field_semantics_version` | integer | The producing scanner's field-semantics generation (§7). **1** for all four at this record's refs. |
| `check_id` | string | The check class. Per-scanner source field in §4. Verbatim bytes, **no case folding**. The **only** component that may differ between a stored decision and an observation it suppresses, and only through one §5 alias edge — see §3.1.1. |
| `scope_root` | string | The canonical form of the **authorized target row** (`findings.target_id` → `targets`) the observation is attributed to — *not* any scanner-supplied echo of our own input. See §3.5. |
| `canonical_host` | component | Canonical hostname per §3.3, or `NotApplicable` when the finding's subject is an IP literal. |
| `canonical_addr` | component | Normalized IP literal, `Present` **only** when `canonical_host` is `NotApplicable`. The two are mutually exclusive and at least one must be `Present`. |
| `port` | component | u16, resolved per §3.3. `NotApplicable` for checks with no port dimension. |
| `protocol` | component | Lowercase scheme or transport, per §4. `NotApplicable` where the check has no protocol dimension. |
| `location` | component | Normalized path-and-query per §3.3. `NotApplicable` where the check has no sub-host location. |
| `discriminator` | component | The per-scanner remainder named in §4 — the part that carries "which of the several findings this check can produce on one location is this one". An **ordered sequence** of the sub-values §4 names for that scanner, serialized by the tag-and-length rule in §3.3; one component, one slot, one encoding. `NotApplicable` only where §4 says the scanner has no discriminator dimension at all. |

### 3.1.1 The comparison rule — eleven components equal, one aliasable

The §3.1 table has **twelve** rows. (Earlier drafts of this record and the issue comment that
announced it said *"ten-component"*; that was a miscount of the same table, corrected here before
acceptance. Nothing downstream depended on the number — §9's tests enumerate the components by name.)

The key is compared **component-wise**, not as one opaque blob, because exactly one component has a
weaker rule than the rest:

1. **`match_version`, `tenant_id`, `scanner_id`, `field_semantics_version`, `scope_root`,
   `canonical_host`, `canonical_addr`, `port`, `protocol`, `location`, `discriminator` — byte
   equality, no exceptions.** No alias, no widening, no normalization beyond §3.3. `NotApplicable`
   equals `NotApplicable`; `Unknown` equals nothing, including itself (§3.4).
2. **`check_id` — byte equality, or exactly one directed §5 alias edge from the stored value to the
   observed one.** §5.7 already guarantees *at most* one admissible edge between a given stored
   `check_id` and the observed one, so "exactly one" only excludes the zero case: no edge means no
   match, and the observation is `new`. Chaining two edges to reach the observed id is never a match
   (§5.2). Where several *different* stored `check_id`s each hold one valid edge into the observed
   one, §3.6 states which decision applies.

Stated the other way: an alias is the **only** mechanism in this record by which a stored decision
reaches an observation it is not byte-identical to, and it reaches exactly one component. That is why
§5 constrains aliases harder than anything else here, and why §5.1 forbids an alias on any of the
other components — an alias on `canonical_host` or `location` would be a widening of *where*, which
§10.2 forbids outright.

> **A matcher that compares all twelve components by byte equality and then also honours aliases is not
> implementable** — the two clauses contradict each other whenever the `check_id`s differ. The rule
> above is the normative one; §2.1 is its acceptance form. If a test asserts the stricter reading,
> the test is asserting behaviour this record does not promise.

### 3.2 `tenant_id` is a partition *and* a stored component

Matching runs inside the tenant transaction context, so RLS (**TDD §3.5** — qualified because this
record has a §3.5 of its own, `scope_root`, and an unqualified "§3.5" here read as a self-reference
says something false) already makes a cross-tenant match impossible by construction. `tenant_id` is nevertheless written into `match_scope` and compared,
because a tenant-crossing suppression is the single worst outcome this feature can produce and one
mechanism is not enough for it. If the two ever disagree, that is a hard error and an isolation
incident, not a cache miss. Belt and braces, deliberately, under **blast radius**.

### 3.3 Canonicalization — exactly these rules, and no others

Every rule below is a chance for two distinct things to become one key, so the list is deliberately
short. Where a rule is omitted, the omission is stated and its consequence is named.

**`canonical_host`.** **UTS-46 Processing applied to the original input**, with
`UseSTD3ASCIIRules = true`, `Transitional_Processing = false`, `CheckHyphens = true`,
`CheckBidi = true` and `CheckJoiners = true`, followed by ToASCII (Punycode) on each label; then strip
**exactly one** trailing `.`; reject empty labels, a label over 63 octets, or a name over 253 octets.
Any error UTS-46 records makes the component `Unknown`.

> **There is no NFKC pre-pass, and an earlier version of this rule had one — which was a
> key-collapse primitive, not a harmless redundancy.** The rule read *"NFKC, then IDNA 2008 ToASCII
> under UTS-46 …, then ASCII-lowercase"*. Three things are wrong with it, and the third is a defect
> in this record's own subject matter.
>
> First, **UTS-46 normalizes to NFC, not NFKC.** Its Processing is Map → *Normalize the domain_name
> string to Unicode Normalization Form C* → Break (at U+002E **only**) → Convert/Validate
> (UTS #46 revision 31, §4 Processing, read 2026-10-02). Nothing in the algorithm uses NFKC.
> Second, UTS-46's own Map step already lowercases and already maps the dot-like characters that
> *should* become separators, so the two bookend steps were redundant: `IdnaMappingTable.txt` at
> Unicode 15.1.0 has `3002 ; mapped ; 002E`, `FF0E ; mapped ; 002E` and `FF61 ; mapped ; 002E`.
>
> Third, and the reason this is a correction rather than a tidy-up: **NFKC maps characters to `.`
> that UTS-46 deliberately rejects, so the pre-pass manufactured label separators UTS-46 would never
> have produced.** The same Unicode 15.1.0 table has `2024..2026 ; disallowed` (ONE DOT LEADER,
> TWO DOT LEADER, HORIZONTAL ELLIPSIS) and `FE52 ; disallowed` (SMALL FULL STOP) — and
> `NFKC(U+2024) = "."`, `NFKC(U+2025) = ".."`, `NFKC(U+FE52) = "."` (checked against the Unicode
> 15.1.0 normalization data, 2026-10-02). So `exam␣ple.com` written with U+2024 canonicalized, under
> the pre-pass, to **`exam.ple.com`** — a *different, valid* host — instead of being rejected. Two
> distinct inputs collapsing to one key is the direction §1 forbids, and here the collapsing input is
> chosen by whoever supplies the name. Without the pre-pass UTS-46 records an error, the component is
> `Unknown`, and §3.4 makes the key non-storable: noise, not silence.
>
> `UseSTD3ASCIIRules = true` is part of the fix and not decoration — several dot-like characters are
> `disallowed_STD3_mapped`, which means *mapped to `.`* when that flag is false and *rejected* when it
> is true. Under §1 we take the rejection.
>
> **This is the shared `vf-core` canonicalizer, so the correction propagates.** §25's
> `authz/psl-exact-root` and `authz/configured-scope` own this canonicalizer's tests (see the note
> below); they must assert the corrected rule, including at least one `disallowed` dot-like code point
> rejected rather than canonicalized. That is a change to what those two identifiers assert, not a new
> identifier, and it is the one place this record reaches back into theirs — stated here rather than
> left for whoever writes them to discover.

> **"Reject" means the component is `Unknown`, not that the key is silently short one field.** A
> value that does not canonicalize to a hostname has not been understood, and §3.4 then makes the
> whole key non-storable and non-matchable. Stating this is not pedantry: §4.5's `canonical_host`
> source can carry a `host:port` string (see §4.5's row), and the difference between "rejected" and
> "`Unknown`" is the difference between an implementer inventing a fallback and one returning 422.

> *Which normalization applies where, and why NFKC appears nowhere.* **Both** paths end at NFC.
> Hostnames are **identifiers**, so they get the whole of UTS-46 — its mapping step (case folding,
> the `mapped` dot-like characters, the `disallowed` rejections) and then NFC, which is what a
> resolver will do. Set elements are **opaque payloads** — an extracted string, a TXT record — so
> they get NFC and nothing else: no mapping, no case folding. NFKC is used on neither, and the
> reason is the same in both places. On a payload it folds distinct bytes together (`ﬁ` → `fi`) and
> merges two findings into one key. On a hostname it manufactures label separators from code points
> UTS-46 rejects (see above). Compatibility folding is a *merging* operation, and §1 forbids merging
> in every component. (Advisory A3, as corrected by the second automated reviewer.)

> **This must be the same canonicalizer as §5.3 scope matching, in `vf-core`.** Not a second one.
> A second host canonicalizer is a design failure: the two would drift, and the drift would be a
> suppression that applies to a host the scope matcher considers different. §25's
> `authz/configured-scope` and `authz/psl-exact-root` own that canonicalizer's tests; this record
> **reuses** it and adds no identifier of its own for it. It does, however, **correct its rule** —
> the NFKC pre-pass above — so those two identifiers must assert the corrected behaviour. Reuse
> without a second implementation; not reuse without consequence. (**Crate purity** — `vf-core` is
> pure, so this is shared library code, not a service call.)

**`canonical_addr`.** IPv4 in dotted-quad; IPv6 per RFC 5952 (lowercase hex, maximal `::`
compression, no leading zeros). Set only when the finding's subject is an IP literal rather than a
name.

**`port`.** Taken from the URL when present; otherwise defaulted **from the scheme**, using exactly
this table and no other: `http`→80, `https`→443. **No other scheme defaults** — not `ftp`, not
`ssh`, and in particular not the nuclei `type` values `dns`, `ssl`, `tcp`, `javascript`, which are
not schemes at all. If the check **has** a port dimension and neither a port nor a defaulting scheme
is available, the component is `Unknown` (§3.4); if the check has **no** port dimension it is
`NotApplicable`, and which checks those are is named per scanner in §4, not inferred here.
Default-port folding is required, not optional: without it `https://example.com` and
`https://example.com:443` are two keys for one service.

**`location`.** Path **and query**, fragment stripped (a fragment is never sent on the wire).
Empty path becomes `/`. Otherwise **verbatim**: no percent-decoding, no case folding, no collapsing
of repeated separators, no trailing-slash normalization.

> *Why verbatim.* Percent-decoding merges `%2F` with `/`, which are different paths to a server.
> Case folding merges `/Admin` with `/admin`, which are different resources. The query must be in
> the key because §10.2 names *parameter* as a location discriminator, and for an injection template
> `?id=1` and `?id=2` are two findings. **Omitted on purpose:** query-parameter *ordering* is not
> normalized, so `?a=1&b=2` and `?b=2&a=1` are two keys. **The omission is justified by its
> direction alone:** order-insensitivity could only ever *merge* two keys into one, which is the
> direction §1 forbids, so the missing rule can only cost noise. It is **not** justified by scanner
> determinism, which does not hold across all four: it holds vacuously for `subfinder` and `dnsx`
> (`location` is `NotApplicable`), and for `httpx` (`location` derives from `url`, which echoes the
> request we built from our own `targets` row) — but **not for `nuclei`**, whose `location` derives
> from `attributes.matched_at`, a value the scanned host can influence (§4.5a) and which carries
> generated parameters for `fuzzing:`/`payloads:` templates. Stating this matters because the record
> is precedent: "the scanners emit it deterministically" must not be reusable as a reason to drop the
> next rule.

**Set-valued components** (`discriminator` inputs that are arrays — DNS record values, nuclei
`extracted_results`). The digest is:

> SHA-256 over the concatenation of, **for each element in ascending order of its NFC-normalized
> bytes**, a `u32` big-endian octet count **of that element** followed by that element's
> NFC-normalized bytes.

The length prefix is **per element**, not one prefix over the whole concatenation. That distinction
is the entire point of the rule, so it is spelled out rather than implied: a single prefix over the
concatenation gives `["ab","c"]` and `["a","bc"]` the same total length `3` and therefore the
identical digest, which is the collision the rule exists to stop. Sorting is required because no
scanner guarantees array order; per-element length-prefixing is required because plain concatenation
is ambiguous, and an attacker who controls one extracted value would otherwise control which other
finding it collides with.

**Duplicates are kept, not deduplicated.** `["a","a"]` and `["a"]` digest differently. Two records
carrying the same value is an observable difference in the answer a resolver gave or in what a
template extracted, and collapsing it could only ever *merge* two keys — the direction §1 forbids.
The cost is accepted and named: a set whose multiplicity changes between scans re-surfaces as `new`.

**`discriminator` is a sequence, and its serialization is specified here, not left to Forge.** §4
defines, per scanner, an **ordered list of sub-values** for this component — one for `dnsx`, two for
`nuclei`, none for `subfinder` and `httpx`. Because the key is compared by byte equality, a
multi-sub-value component with an unstated delimiter is not a specification. The canonical form is:

> For each sub-value, in the order §4 lists it: one tag octet — `0x00` for `NotApplicable`, `0x01`
> for `Present` — followed, **only when `Present`**, by a `u32` big-endian octet count and that
> sub-value's bytes. A sub-value that is itself a set is first reduced to its 32-octet set digest
> above and carried as `Present` with the digest as its bytes. The component's `Present(String)` value
> (§3.4) is the lowercase hex of this byte sequence.

Three consequences, all deliberate:

- The component is `NotApplicable` **only** where §4 says the scanner has no discriminator at all. A
  sequence in which *every* sub-value happens to be `NotApplicable` is `Present` — the two tag octets
  `0x00 0x00` — and does **not** equal `NotApplicable`. "This template has no matcher name and
  extracted nothing" and "this scanner has no discriminator dimension" are different statements and
  must not share a key.
- Any sub-value that is `Unknown` makes the whole component `Unknown`, and §3.4 then applies.
- The tag octets mean the four `(matcher_name, extracted_results)` combinations in §4.5 are four
  distinct byte strings, with no delimiter to collide with a sub-value's own bytes.

### 3.4 Absence is a typed third state, and `Unknown` is fatal

Each component is `MatchComponent = Present(String) | NotApplicable | Unknown`, serialized as
`{"s":"p","v":"…"}`, `{"s":"na"}`, `{"s":"u"}`.

- **Never SQL `NULL`.** `NULL = NULL` is `UNKNOWN` in SQL, so a key with a `NULL` component would
  never equal itself. A false-positive decision that silently never applies looks to a user exactly
  like the feature being broken, and it would be found by a customer, not by us.
- **Never `""`.** An empty string makes "this check has no port" indistinguishable from "we failed
  to determine the port", and those must not share a key.
- **`Unknown` is fatal in both directions.** `POST /v1/findings/{id}/state` with a false-positive
  action on an observation whose key contains `Unknown` returns **422** and creates no decision. At
  match time, an observation whose key contains `Unknown` matches **nothing** and is `new`. A finding
  we could not fully locate is a finding we must not suppress and must not suppress *with*.

### 3.5 `scope_root` comes from our row, not from the scanner

`scope_root` is the canonical form of the authorized `targets` row the observation resolved to at
ingest (§10.1), reached through `findings.target_id`.

It would be easier to read the scanner's echo of our own input — subfinder's `attributes.domain`,
httpx's `input`. That is wrong for two reasons. It makes the key depend on the formatting of a
string a scanner chose to echo back, so an upstream change to that echo silently changes every key.
And for subfinder it is an outright trap: `attributes.domain` is the *root queried*, not the
finding's host (§8.1.2). Using our own authorized target row removes both problems and is the
tenant-scoped, trustworthy source by construction.

**The cost, named rather than discovered later.** `scope_root` can over-narrow. If one host resolves
to a *different* `targets` row between scans — a target reorganised, or deleted and re-added at a
different scope — `scope_root` changes and every stored decision on that host stops matching. The
user re-marks. That is the direction §1 permits and the component stays, but it is a real behaviour a
support engineer will see and should not have to rediscover: **"I deleted and re-added the target and
all my false-positive marks came back"** is expected, not a bug. (Advisory A2.)

### 3.6 Matching must be an index lookup

§10.5 requires the findings path to stay interactive, and §10.4 applies matching to **every**
observation of **every** scan. The canonical serialization of §3.1 is therefore required to be a
deterministic byte string, and matching is equality on it (or on its digest) — never a `jsonb`
containment scan over the tenant's decisions. Whether that lands as a stored generated column or an
expression index is Forge's call; that it is not a per-row scan is not. **Either choice is DDL**, and
it is one of the three migrations enumerated in §1 and §2's "no schema change" row — see §6 for the
other two.

**The §3.1.1 alias clause does not weaken this, because it is resolved before the lookup, not during
it.** Aliasing substitutes one component of the probe, so it turns one equality lookup into a small
fixed set of them:

1. Serialize the observation's own key and look it up. A hit is a match with no alias edge recorded.
2. On a miss, consult the §5 registry for edges **into** the observation's `check_id` that are valid
   at this `(match_version, field_semantics_version)`. For each such edge `A → observed`, serialize
   the same key with `check_id` replaced by `A` and look that up. A hit records the edge and
   `registry_version` in `false_positive_match` (§5.4).

The probe count is `1 + indegree(check_id)` equality lookups, bounded by the committed registry file
rather than by tenant data, and the registry is small and CI-validated (§5.7). §5.7's rule that a
`from_check_id` appears on at most one valid edge bounds the *out*-degree, which is what makes step 2
deterministic. An in-degree above one is admissible — several retired checks may alias into one
survivor — so more than one probe can hit, and the tie needs a stated rule rather than row order:

> **Precedence.** Step 1's exact hit always wins over any step-2 hit. If two or more step-2 probes
> hit, the applied decision is the one with the lowest `(created_at, id)`, and the edge recorded is
> that decision's. Suppression itself is not in question in that case — every matching decision is a
> false-positive mark on an equal-or-narrower check (§5.3) — so the rule exists to make
> `applied_fp_decision_id` and the recorded edge **deterministic and auditable**, not to decide
> whether to suppress. Revoking the applied decision (§6) re-runs this resolution on the next
> observation, which may then apply a different one; that is correct, and it is why §6 is future-only.

---

## 4. The equivalence fields, per scanner

### 4.1 First, a fact about the pipeline that changes two records

**secureCodeBox v5.9.0 ships no `dnsx` scanner and no `httpx` scanner.** The `scanners/` tree at tag
`v5.9.0` contains exactly: `ffuf`, `git-repo-scanner`, `gitleaks`, `kube-hunter`, `ncrack`, `nikto`,
`nmap`, `nuclei`, `screenshooter`, `semgrep`, `ssh-audit`, `sslyze`, `subfinder`, `test-scan`,
`trivy`, `trivy-sbom`, `whatweb`, `wpscan`, `zap-automation-framework`.

The TDD already anticipated the *images*: §21.3 says *"Build and validate the existing custom arm64
images for dnsx, httpx, tlsx, masscan, and optional Amass."* What it did not say, and what
ADR-0003 §3.2 asserted incorrectly, is that the **parsers** come for free. They do not: a custom
image needs a custom `ScanType` **and** a custom `ParseDefinition` with a parser behind it. So
**ADR-0003 §3.2's claim that "upstream ships parsers for all four" and that "Phase 1 requires zero
custom parsers" is false.** Phase 1 requires two custom parsers. That is recorded as ADR-0003
**amendment A1**, filed with this record; it does not reverse ADR-0003's decision — staying on the
upstream JavaScript SDK is *more* clearly right once we are definitely writing parsers, because the
alternative is reimplementing the whole `parser-wrapper.ts` contract in Rust for two scanners.

For this record the consequence is a split in provenance, and it is stated per row:

| Scanner | Where the `attributes` shape comes from | Read at |
|---|---|---|
| `subfinder` | **Upstream**, fixed | `secureCodeBox/secureCodeBox` `v5.9.0`, `scanners/subfinder/parser/parser.js` |
| `nuclei` | **Upstream**, fixed | `secureCodeBox/secureCodeBox` `v5.9.0`, `scanners/nuclei/parser/parser.js` |
| `dnsx` | **VulcanFlow's**, specified here, mapped from the tool's own output struct | `projectdiscovery/dnsx` `v1.3.1` → `projectdiscovery/retryabledns` `v1.0.116` `DNSData` |
| `httpx` | **VulcanFlow's**, specified here, mapped from the tool's own output struct | `projectdiscovery/httpx` `v1.12.0`, `runner/types.go` + `runner/runner.go` |

All four are additionally constrained by the SCB envelope: `findings-schema.json` at `v5.9.0` requires
`id`, `parsed_at`, `severity`, `category`, `name`, `scan`, allows `attributes` as a free-form object
(*"Attributes are not standardized. They differ from Scanner to Scanner"*), and restricts `severity`
to `INFORMATIONAL | LOW | MEDIUM | HIGH`.

### 4.2 `subfinder` — subdomain existence

Check class: `subfinder/subdomain`. One class; subfinder has no notion of a check.

| Key component | Source field | When absent |
|---|---|---|
| `check_id` | Constant `subfinder/subdomain`. (The envelope `category` is the literal `"Subdomain"` for every subfinder finding, including the synthetic apex one — it carries no discriminating information.) | n/a |
| `canonical_host` | `attributes.hostname` (parser: `item.host`; for the synthetic apex finding, the extracted domain) | Never absent; a subfinder finding with no hostname is malformed and rejected at ingest |
| `canonical_addr` | — | Always `NotApplicable`; subfinder findings are about names |
| `port` / `protocol` / `location` | — | Always `NotApplicable` |
| `discriminator` | — | Always `NotApplicable` |

**Deliberately excluded, and why.** `attributes.domain` — the *root queried* (`item.input`), not this
finding's host; §8.1.2. `attributes.source` — the data source that reported the subdomain
(`crtsh`, `virustotal`, …, or the literal `"parser"` for the synthetic apex finding under
`INCLUDE_TARGET_DOMAIN=true`); the same hostname seen through two sources is one subdomain, and
keying on source would under-suppress on every re-scan. `attributes.ip_address` /
`attributes.ip_addresses` — volatile and shared. `location` — equals the hostname here, so it is
redundant, and relying on it would be relying on a field whose format is per-scanner (§8.4.2).

### 4.3 `dnsx` — DNS record observations

Check class: `dnsx/dns-record:<RRTYPE>` where `<RRTYPE>` ∈ `A, AAAA, CNAME, MX, NS, TXT, SRV, CAA,
SOA, PTR`. One observation per (name, record type) actually returned.

| Key component | Source field (`DNSData` JSON key) | When absent |
|---|---|---|
| `check_id` | Derived from which record array the observation came from — `a`, `aaaa`, `cname`, `mx`, `ns`, `txt`, `srv`, `caa`, `soa`, `ptr` | The record type is what produced the observation; it cannot be absent |
| `canonical_host` | `host` | Never absent |
| `canonical_addr` | — | `NotApplicable` unless the queried name *is* an IP literal (PTR lookups), in which case `canonical_host` is `NotApplicable` and this carries the normalized address |
| `port` | — | `NotApplicable`, except `SRV`, whose port is part of the record value and therefore of the `discriminator`, not of this component |
| `protocol` | — | Always `NotApplicable`. The DNS transport (UDP/TCP/DoH) is not part of the finding's identity |
| `location` | — | Always `NotApplicable` |
| `discriminator` | **One sub-value** (§3.3): the set digest of the record values for that type, each lowercased for name-valued types (`cname`, `mx`, `ns`, `ptr`, `srv`) and with a single trailing `.` stripped; **verbatim for `txt` and `caa`** — meaning no case folding and no content rewriting, but still NFC-normalized by §3.3's set-element rule, which every element goes through and which is the only normalization applied to a payload; normalized per `canonical_addr` rules for `a`/`aaaa`; for `soa`, the per-element string defined in §4.3.1 | Never `NotApplicable`. If the values could not be read, `Unknown` — and §3.4 applies |

### 4.3.1 `soa` is a struct, not a string — and only three of its eight fields are identity

`retryabledns` `v1.0.116` declares `SOA` as a struct, not a string:
`name`, `ns`, `mailbox`, `serial`, `refresh`, `retry`, `expire`, `minttl` (`client.go:760–769`, read
at that tag). `DNSData.soa` is therefore `[]SOA`, the only record array that is not `[]string`. A rule
that names only `soa.ns` leaves seven fields undefined, and one of them — `serial` — increments on
**every zone edit**, so "include everything" and "include `ns` only" differ by a feature that churns
on every DNS change versus a record type keyed far more narrowly than every other.

**The per-element string for an `soa` observation is `name ‖ 0x1F ‖ ns ‖ 0x1F ‖ mailbox`**, each of
the three lowercased with a single trailing `.` stripped, before the §3.3 set digest is taken.
`serial`, `refresh`, `retry`, `expire` and `minttl` are **excluded**: they are timers and counters —
"time, not identity", the same reason §4.3 already gives for `ttl`. `mailbox` is kept because it is
identity-bearing: a zone's responsible mailbox changing is a change in who owns the zone, which is
exactly the kind of thing a suppression should not outlive. `0x1F` (unit separator) cannot occur in a
DNS name, so no escaping rule is needed.

**This is the one widening in §4 that is argued for rather than avoided, so it is argued.** Two
`soa` observations differing *only* in `serial` or in a timer share a key, so a decision made before
a zone edit still applies after it. The second automated reviewer flagged exactly this and proposed
serializing all eight fields; it is rejected on the ground §4.3 already uses for `ttl` — a counter is
**not identity**. A zone whose serial incremented is the same zone, with the same nameserver and the
same responsible mailbox, observed again; nothing the user reviewed has changed. Including `serial`
would re-surface every SOA finding on every zone edit, which is churn rather than caution, and §7.1
records what churn at that cadence does to a user's willingness to mark anything at all. The three
fields kept are the three that move when **who controls the zone** moves.

**Deliberately excluded.** `ttl`, `timestamp`, `query-time` — time, not identity. `resolver` — which
resolver answered is evidence about the measurement, not about the record. `cdn`, `cdn-name`,
`cdn-type`, `asn.*` — enrichment derived from the record value, so including them would double-count
the value and make the key move when a CDN re-labels an IP range. `status_code` / `status_code_raw` —
a `NOERROR` and a later `SERVFAIL` for the same name are not the same observation, but a failed
lookup produces no record observation at all, so this never reaches the key.
**`all`** (`DNSData.AllRecords`, `json:"all"`) — the flattened list of every record across **all**
types. This one is named explicitly because it is the single most inviting wrong answer in the
struct: an implementer reaching for it as the discriminator source merges A and CNAME values into one
digest, which is precisely the collapse §8.2.2 exists to prevent, reached by *reading* §4.3 rather
than by ignoring it. Also excluded: `raw`, `raw_resp`, `trace`, `axfr` — wire-level evidence;
`internal_ips`, `has_internal_ips`, `hosts_file` — derived labels about the measurement environment.

> **And the list does not have to be exhaustive to be safe:** *any `DNSData` field not named in the
> §4.3 table above is excluded from the key by construction.* The table is the allowlist; this
> paragraph only explains the ones a reasonable implementer would reach for anyway. The distinction
> matters more here than for `subfinder`/`nuclei`, because §4.1/A1 §7.1 established that **we** write
> this parser — §4.3 is the only specification its author has.

> *Note the mixed JSON conventions in dnsx's output, so a parser author does not guess:* `DNSData`
> uses snake_case (`status_code`, `has_internal_ips`) while the `ResponseData` wrapper added by dnsx
> itself uses hyphens (`cdn-name`, `cdn-type`, `query-time`, `as-number`). Both appear in one object.

### 4.4 `httpx` — HTTP service and endpoint observations

Check class: `httpx/http-service`. One class at `match_version = 1`.

| Key component | Source field (`runner.Result` JSON key) | When absent |
|---|---|---|
| `check_id` | Constant `httpx/http-service` | n/a |
| `canonical_host` | **`host`** — which at `v1.12.0` is `parsed.Hostname()`. Read §7.3 before touching this field | If `host` is an IP literal, `NotApplicable` and `canonical_addr` carries it |
| `canonical_addr` | `host` when it is an IP literal. **Never `host_ip`** | See §8.3.2 |
| `port` | `port`, else defaulted from `scheme` per §3.3 | `Unknown` if neither — §3.4 applies |
| `protocol` | `scheme`, lowercased | `Unknown` if absent |
| `location` | `path` (plus query) from **`url`** — **never `final_url`** | Empty → `/`. See §8.3.1 |
| `discriminator` | — | Always `NotApplicable` at `match_version = 1`; httpx has no discriminator dimension |

**`failed` and `error` are preconditions, not key components** — the §4.5 `matcher_status` rule, for
the scanner that needs it just as much. `runner.Result` carries `failed bool` (`json:"failed"`) and
`error string` (`json:"error,omitempty"`), and `port`, `scheme` and `url` are all `omitempty`. A
`Result` with `failed: true` or a non-empty `error` is a **measurement failure, not a service
observation**: it must not become an observation, and so it never reaches the matcher. Without this
rule a failed probe becomes an observation whose `port` and `protocol` are `Unknown`, which under
§3.4 returns 422 when a user tries to mark it — a finding that cannot be marked a false positive,
with nothing in the product able to explain why.

**Deliberately excluded.** `final_url` — attacker-influenceable and a mass-collapse primitive
(§8.3.1). **`location`** — and this one needs saying out loud, because it name-collides with the key
component called `location` and is the more direct form of the same defect: `Result.Location` is the
**raw `Location` response header**, assigned `resp.GetHeaderPart("Location", ";")` at
`runner/runner.go:2685`, `v1.12.0`. §8.3.1's whole argument is that `final_url` must stay out because
it is *derived from* that header; the header's own field is §8.3.1 with the indirection removed, and
an implementer reading the `location` row of the table above and reaching for `Result.location` would
put an attacker-set value straight into the key. **`sni`** — a plausible and wrong `canonical_host`
source: it is the name we sent, not the name the finding is about. **Excluding it is only safe
because of §4.4a, which is therefore mandatory rather than a note.**
`host_ip`, `a`, `aaaa`, `cname`, `asn.*`, `cdn*` — address-level, and §6.2 is explicit that
*"IP/port alone is not a universal asset identity"* (§8.3.2). `status_code`, `title`, `webserver`,
`tech`, `cpe`, `content_length`, `words`, `lines`, `favicon*`, `hash`, `jarm_hash`, `body_preview`,
`time` — response content and fingerprints, which change on any deploy; a key that moves when the
page changes would silently drop every suppression on the next release. `input` — our own echo, see
§3.5. `vhost` — a boolean about the *host*, already carried by `canonical_host`. Also excluded, one
reason for the group: `method`, `content_type`, `csp`, `tls`, `chain`, `chain_status_codes`,
`extracts`, `extract_regex`, `header`, `raw_header`, `request`, `body`, `headless_body`,
`body_fqdn`, `body_domains`, `knowledgebase`, `trace`, `resolvers`, `timestamp`, `websocket`,
`http2`, `pipeline`, `wordpress`, `screenshot_*`, `stored_response_path`, `favicon_md5`,
`favicon_path`, `favicon_url` — response content, probe configuration or measurement evidence, none
of them part of *which service at which location* this observation is about.

> **Same closer as §4.3, and for the same reason:** *any `runner.Result` field not named in the §4.4
> table above is excluded from the key by construction.* We write this parser (§4.1), so §4.4 is the
> only specification its author has, and an allowlist that fails closed is the only safe shape for
> it.

### 4.4a The httpx `ScanType` must not set a custom TLS SNI

The second automated reviewer raised this and it is right: at `v1.12.0` httpx takes
`--sni-name` / `-sni` (`runner/options.go:535`), and `runner/runner.go:1118–1119` sets
`resp.SNI = r.options.SniName` **only when that flag is non-empty**. A custom SNI is chosen
independently of the request URL and selects which TLS virtual host answers. So two probes of the
same `url` with different SNI values can reach **different services** and — with `sni` excluded from
the key — produce the **same** key. A decision about one virtual host would then suppress a finding
about another: the §8.3.2 collapse, one layer down.

**Therefore: the VulcanFlow httpx `ScanType` does not pass `--sni-name`.** This is a property of the
approved catalog (§21.3), the same shape of control as §4.5a's redirect rule, and
`supply-chain/check-catalog` asserts it. With the flag unset, `sni` is always empty, so adding it to
the key would contribute a constant and the exclusion is exact rather than merely convenient.

**If that ever changes, `sni` becomes a key component and the change is a `match_version` bump** —
not an edit to §4.4's table, because keys stored without it cannot be compared with keys stored with
it. A revisit trigger for it is in §12. Adding the component *now*, as the reviewer suggested, was
rejected for the same reason §4.4 excludes `vhost`: a component that is a constant under our own
configuration buys nothing and hides the fact that the real control is the configuration.

### 4.5 `nuclei` — template findings

Check class: `nuclei/<template-id>`, taken from the envelope `category`, which the v5.9.0 parser sets
to `finding["template-id"]` verbatim.

| Key component | Source field | When absent |
|---|---|---|
| `check_id` | `nuclei/` + `attributes.template_id` (= envelope `category`) | Never absent; required by the envelope schema |
| `canonical_host` | `attributes.hostname`, **after the §4.5b shape rules** — the parser's `parseHostname(finding.host)` returns the host verbatim when it carries no `scheme://`, and `null` on falsy or unparseable input, so it is not always a hostname | `NotApplicable` if it is an IP literal, which then goes to `canonical_addr`. `Unknown` when §4.5b says so |
| `canonical_addr` | `attributes.hostname` when it is an IP literal. **Never `attributes.ip_addresses`** | See §8.4.4 |
| `port` | **Only** per the §4.5c port-dimension table — from the `:port` in `attributes.hostname` (§4.5b), else the port in `attributes.matched_at` **when §4.5a admits it**, else the §3.3 scheme default | `NotApplicable` for `type: dns`; `Unknown` where §4.5a/§4.5c say so |
| `protocol` | `attributes.type` (`http`, `dns`, `ssl`, `tcp`, `javascript`, …), lowercased | `Unknown` if absent. See §8.4.2 |
| `location` | Path-and-query of `attributes.matched_at`, per §3.3, **when §4.5a admits it** | `NotApplicable` per the §4.5c table; `Unknown` where §4.5a says so |
| `discriminator` | A **two-sub-value sequence** (§3.3), in this order: **(1)** `attributes.matcher_name`, **(2)** the set digest of `attributes.extracted_results`. Both are part of the finding's identity | Each sub-value is independently `NotApplicable` — `matcher_name` is `null` for single-matcher templates, and `extracted_results` is often empty. Per §3.3 the component itself is still `Present`: `0x00 0x00` for the both-absent case, which is **not** the same as `NotApplicable` |

### 4.5a `matched_at` may name a different host than `attributes.hostname` — and then the key is not storable

This is the one place where §8.3.1's standard was applied to `httpx` and not to `nuclei`, and
upstream's own fixture at the cited ref contains the case.
`scanners/nuclei/parser/__testFiles__/example-com-test.jsonl` at secureCodeBox `v5.9.0`, last line:

```
template-id: azure-domain-tenant   type: http
host:        https://example.com
matched-at:  https://login.microsoftonline.com:443/example.com/v2.0/.well-known/openid-configuration
ip:          40.126.32.140
```

Under §4.5's table as originally written that observation keys as `canonical_host = example.com`
(from `attributes.hostname`) with `port = 443` and
`location = /example.com/v2.0/.well-known/openid-configuration` — **a port and a path on a host where
neither exists**. The record gave no rule for the divergence, so two implementers would do two
different things and the text would not call either wrong.

**The rule.** Take `matched_at`'s host, run it through the **same §4.5b branch table** — which is what
makes `[2001:db8::1]` and `2001:db8::1` compare as the one address rather than as two strings — and
compare the result to the observation's `canonical_host` when that is `Present`, or to its
`canonical_addr` when `canonical_host` is `NotApplicable`. The comparison is between like components:
a host and an address never compare equal, so a `matched_at` naming an address on a finding keyed by
name is a divergence. If they differ, `port` and `location` are **`Unknown`**, and §3.4 then makes
the whole key non-storable and non-matchable: `POST …/state` returns 422 and the observation matches
nothing. A finding whose location belongs to a host we did not key is a finding we cannot fully
locate, and §3.4 already says such a finding must not be suppressed and must not be suppressed *with*.

**Precedence between this rule and §4.5c, stated rather than left to reading order.** §4.5c decides
whether a `type` has a port dimension and a location dimension at all; this rule decides whether a
`type` that *has* those dimensions may take their values from `matched_at`. **§4.5c is evaluated
first and this rule second**, and the two compose by one rule only: `NotApplicable` from §4.5c is
**never** promoted to `Unknown` by a divergence here. So `type: dns` keeps `port` and `location`
`NotApplicable` even when `matched_at` names a different host, because there is no port and no
location to be wrong about — which is why §8.4.2's `cname-fingerprint` case stays storable. For
`type: http`, `ssl`, `tcp` and `network` the divergence applies and both components go `Unknown`. The
unspecified-`type` row of §4.5c is already `Unknown`, so the order cannot change it. Without this
sentence an implementer could read a divergence as overriding `NotApplicable` and make every DNS
nuclei finding unsuppressible — the outcome §4.5c was written to prevent.

The alternative — key `port`/`location` only when the hosts agree and `NotApplicable` otherwise — is
**rejected**: it collapses every divergent-host observation of **one template** on one host into a
single key (`check_id` keeps *different* templates apart, so the collapse is within a check class, not
across them), which is the §8.4.1 collapse with a different cause. See §10 for the full reasoning,
including the overbroad version of this sentence that review caught.

**And the same field is attacker-influenceable in the §8.3.1 sense, so there is a second control.**
`matched_at` is the URL where the match occurred, which under redirect following is the
**post-redirect** URL — a value the scanned host chooses. §8.3.1 excludes httpx's `final_url` for
exactly this property and concludes that *"a key component an outside party can set is a suppression
primitive."* Therefore: **the VulcanFlow nuclei `ScanType` must disable redirect following**, as a
property of the approved check/template catalog (§21.3), and `supply-chain/check-catalog` is the
identifier that has to assert it. With redirects off, `matched_at`'s host is stable for `http`
templates and the §4.5a rule above is the backstop rather than the primary control. The worked
negative example is §8.4.7.

> Precision about evidence: the **host divergence is confirmed** from the fixture above, read at
> `v5.9.0` on 2026-10-02. The **redirect behaviour is an inference** from nuclei's documented
> `-follow-redirects` option, not something executed here — this record runs no suites (ADR-0005
> lane 4). The control is specified as a `ScanType` property precisely because it is cheap to assert
> statically and does not depend on that inference being right.

### 4.5b The shapes `attributes.hostname` actually takes, and the one rule that reads all of them

`parseHostname` (`scanners/nuclei/parser/parser.js:96–118`, `v5.9.0`) returns **`null`** on falsy
input; returns the input **verbatim** unless it matches `/^[a-zA-Z][a-zA-Z0-9+.-]*:\/\//`; and for a
scheme-bearing string returns `new URL(host).hostname`, falling back to `null` when `new URL()`
throws. So `attributes.hostname` is a hostname, a `host:port` pair, an IP literal bracketed or bare,
or `null` — and for IPv6 the brackets are **retained**, because WHATWG `URL.hostname` keeps them:
`https://[2001:db8::1]:8443/x` → `[2001:db8::1]` (verified at the ref, 2026-10-02).

> **A correction to this section as first written, and the one place in this record where the failure
> direction was the unsafe one.** The rule here was *"split a single trailing `:<port>` off before
> §3.3 canonicalization"*, applied unconditionally. It breaks IPv6 twice over. `[2001:db8::1]` has no
> trailing `:<port>`, so it reaches §3.3 where `[` and `]` are not valid label octets ⇒ `Unknown` ⇒
> 422, and **no IPv6 nuclei finding could be marked a false positive at all** — a silent feature
> outage, but at least a safe one. The bare form `2001:db8::1` is worse: it *does* match "a single
> trailing `:<port>`", splitting to host `2001:db8:` and port `1`. That is a **parsed-but-wrong key**,
> not a safe `Unknown`, and §1 forbids exactly that direction. Both are fixed by reading the value's
> shape before splitting anything off it.

**The rule.** Applied to `attributes.hostname` **before** §3.3 canonicalization. Branches are tried in
order and the first match wins; the ordering is the substance of the rule, not presentation.

| # | Shape | `canonical_host` / `canonical_addr` | `port` |
|---|---|---|---|
| 1 | `null`, empty, or any value that reaches no branch below | both `Unknown` | `Unknown` |
| 2 | Begins with `[` — must match `^\[([^\[\]]+)\](?::([0-9]{1,5}))?$` **and** the bracketed text must parse as an IPv6 literal | `canonical_host = NotApplicable`; `canonical_addr` = that literal per §3.3 (RFC 5952). If either condition fails, **both** are `Unknown` | the captured group when present and in `1..=65535`, else `Unknown`; absent ⇒ per the §4.5c table |
| 3 | Contains **two or more** `:` | parsed whole as an IPv6 literal: `canonical_host = NotApplicable`, `canonical_addr` per §3.3. Not a valid literal ⇒ both `Unknown` | **no port is present in this shape** ⇒ per the §4.5c table |
| 4 | Contains **exactly one** `:` | the left side, by branch 5 or 6 applied to it alone; if the right side is rejected, the host component is `Unknown` too | the right side when it is 1–5 decimal digits in `1..=65535`, else `Unknown` |
| 5 | No `:`, and parses as an IPv4 dotted-quad | `NotApplicable` / that literal per §3.3 | per the §4.5c table |
| 6 | No `:`, anything else | `canonical_host` per §3.3 — which may itself yield `Unknown` / `NotApplicable` | per the §4.5c table |

**Why branch 3 must precede branch 4.** A bare IPv6 literal always contains at least two `:`; an
unbracketed `host:port` pair always contains exactly one. The two shapes are therefore disjoint on
colon count, and testing for IPv6 first is what makes it impossible for an address group to be read
as a port. Branch 2 exists because the scheme-bearing path hands us brackets and `[` never
canonicalizes under §3.3. Branch 4 exists for the nuclei `network`/`tcp` convention
(`example.com:8443`): without it §3.3 rejects the `:` ⇒ `Unknown` ⇒ 422, and no nuclei `tcp`/`network`
finding could ever be marked a false positive — for protocols §4.5's `protocol` row and §4.5c's table
both list. **Every failure in the table resolves to `Unknown`, never to a parsed-but-wrong key**,
which is §1's direction and is the property `findings/fp-nuclei-field-shapes` has to assert.

Two shapes worth naming because upstream or convention supplies them:

- **No scheme, bare hostname.** Upstream ships this as its own fixture — `hostname-without-port.jsonl`
  at `v5.9.0` is a `caa-fingerprint` finding with `host: example.com`, which `parseHostname` returns
  unchanged. Branch 6, and harmless: it is already a bare hostname and §3.3 accepts it.
- **`null`.** Branch 1, mapping to `Unknown` explicitly. §3.4 applies: non-storable, non-matchable, 422.

### 4.5c Which nuclei `type` values have a port dimension and a location dimension

§4.3 gets this right for `dnsx` by saying `NotApplicable` outright; §4.5 left `port` to be inferred
and the fixture shows the cost. `cname-fingerprint` in `secureCodeBox-test.jsonl` is `type: dns` with
`matched-at: https://www.securecodebox.io`, so the original rule — *"parsed from `matched_at`, else
defaulted from the scheme"* — gives a DNS observation **port 443**, off an `https` scheme on a finding
that never touched TCP 443. The other available reading yields `Unknown` → 422, making DNS nuclei
findings unsuppressible. One reading produces a nonsense key; the other disables the feature. Neither
is acceptable, and the fix is to stop inferring:

| `attributes.type` | `port` | `location` |
|---|---|---|
| `http` | Present (from `matched_at` per §4.5a, else §3.3's `http`→80 / `https`→443) | Present (path-and-query, §4.5a) |
| `ssl` | Present (from `matched_at`/`hostname`, else `https`→443) | `NotApplicable` |
| `tcp`, `network` | Present (from the `:port` in `attributes.hostname`, §4.5b; `Unknown` if absent — a TCP check with no port is not located) | `NotApplicable` |
| `dns` | **`NotApplicable`** | `NotApplicable` |
| any other `type` | `Unknown` | `Unknown` |

The last row is deliberate and is the §3.4 posture applied to our own ignorance: a `type` this record
has not specified (`javascript`, `websocket`, `whois`, anything upstream adds) is a check class we
cannot locate, so it gets no suppression until an amendment adds its row. Noise, not silence.

**`matcher_status` is a precondition, not a key component.** The parser passes
`attributes.matcher_status` straight through, and nuclei only emits `false` under `-matcher-status`.
A finding with `matcher_status == false` is "this check ran and did not match". It must not become an
observation at all, and so it never reaches the matcher.

**On including `extracted_results` — a real trade, decided one way on purpose.** For nuclei the
extraction frequently *is* the finding: `email-extractor` tells you *which* address is exposed,
`ssl-dns-names` *which* names are on the certificate, `cname-fingerprint` *which* third party the
name points at. Excluding it would let a decision made about one extracted value suppress a finding
carrying a different one. Including it costs re-surfacing whenever a volatile extraction changes — a
version string that moves on every patch will come back as `new`. Under §1's asymmetry that is the
right way round: re-surfacing is noise a user dismisses, and the alternative hides a finding nobody
read. The cost is accepted and its revisit trigger is in §12.

**Deliberately excluded.** `attributes.ip_addresses` (§8.4.4). `attributes.tags`,
`attributes.author`, `attributes.metadata`, `attributes.reference` — template *metadata*, which moves
with every template-pack release and would stale every key monthly for no identity gain.
`attributes.template`, `attributes.template_url` — the path and URL the template was loaded from;
`template_id` is the identity, the path is where it happened to live. `attributes.timestamp`,
`attributes.request`, `attributes.response`, `attributes.curl_command`, `attributes.matched_line` —
evidence. `attributes.path` — the parser emits `finding.path || null` and it is `null` in all five
v5.9.0 fixtures; the located path comes from `matched_at` under §4.5a, and this is the one
`attributes.*` key the tables above would otherwise neither use nor exclude. Envelope `location`
(= `finding.host` verbatim) — §8.4.2. Envelope `severity` — §8.4.5.

---

## 5. Alias semantics

An alias is a claim that a suppression should **travel**. That makes it the most dangerous object in
this design, so it is the most constrained.

An alias edge lives in a committed registry file in `vulcanflow/platform`, validated in CI, and is:

```
{ from_check_id, to_check_id, scanner_id,
  valid_for_match_version, valid_for_field_semantics_version,
  evidence,          # the upstream ref where the equivalence was observed
  actor, created_at, registry_version }
```

### 5.1 Aliases map `check_id`, and nothing else

No alias may map a host, an address, a port, a protocol, a location, or a discriminator. Those are
*where* and *which one*, not *what check*. An alias that moves any of them is not an alias; it is a
broadening, and §10.2 forbids exactly that.

### 5.2 Directed and non-transitive

`A → B` does not imply `B → A`. `A → B` plus `B → C` does **not** imply `A → C`; matching follows at
most **one** edge. Each hop would otherwise add reach without adding evidence, and a four-hop chain
of individually-reasonable renames becomes an equivalence class nobody approved. If `A → C` is
genuinely true, it is filed as its own edge with its own evidence.

### 5.3 Equal-or-narrower only

An edge `A → B` is admissible only if B's detection is a subset of A's. A rename where the new check
also gained a matcher, a protocol, or a path is **two checks**, not an alias — the decision was made
against the old detection and cannot speak for the added part.

### 5.4 Evaluated at match time; stored decisions are never rewritten

A decision stores the `check_id` it was made on, verbatim and forever. Aliasing is applied when a new
observation is matched, against the registry as of that moment, and the edge used is recorded in the
observation's `false_positive_match` alongside `registry_version`. Rewriting the stored decision would
make an alias irrevocable: withdrawing the edge would no longer stop the suppressions it caused.

### 5.5 Cross-scanner aliases are forbidden at `match_version = 1`

A nuclei finding and an httpx finding are never equivalent, even about the same host, port and CVE.
Different detection methods produce different evidence and different false-positive reasons, and
§10.2 already rules that matching a CVE alone is insufficient. Lifting this needs an amendment, not a
registry entry.

### 5.6 No alias is ever created automatically

An upstream check id that disappears and a new one that appears is exactly as consistent with *"the
check changed"* as with *"the check was renamed"*. Guessing wrong hides findings. The default on any
rename is therefore **no carry-over**: the user sees the finding again and re-marks it. Noise, not
silence.

### 5.7 Registry hygiene

Cycles are rejected at load. A `from_check_id` appearing on more than one edge valid at the same
`(match_version, field_semantics_version)` is rejected at load — ambiguity here would make matching
depend on row order. An edge whose `scanner_id` does not match both check ids' prefixes is rejected.
Withdrawing an edge is append-only, like a revocation, and takes effect on future matching only.

### 5.8 Phase 1 has no use for this machinery, and that is the right outcome

Worth stating so the next reader does not reach §8.4.1a and conclude the registry is broken. Across
the four Phase-1 scanners, `check_id` is a **constant** for `subfinder` and `httpx`, an **RRTYPE** for
`dnsx` (never renamed), and `nuclei/<template-id>` — and the one upstream rename this record
documents from refs, `permission-policy` → `permissions-policy` (§8.4.1a), is a **matcher-name**
rename under an unchanged `template-id`. `matcher_name` lives in the `discriminator`, and §5.1 forbids
aliasing a discriminator, so the registry cannot express it.

That is not a gap to close. **Matcher-name renames are out of scope for aliasing**, deliberately: a
matcher is *which of several findings this check produced*, and carrying a decision across it would be
a broadening of exactly the kind §10.2 forbids. The correct handling is §5.6's default — no carry-over,
the user re-marks, which is noise — plus §7.2's staling when the template pack's
`field_semantics_version` moves. So §5 is expected to sit unused in Phase 1. The machinery exists
because the first alias anyone proposes will be proposed under pressure, and the constraints are
cheaper to write now than to argue about then.

---

## 6. Revocation

`DELETE /v1/false-positive-decisions/{id}` (§13.2) revokes; it does not delete. The mechanism is the
existing `false_positive_events` row with `action = 'revoked'`.

- **Append-only.** The decision row stays. §23.5 requires that revocation *"affects future matching
  without rewriting saved reports"*, and §10.2 requires preserving historical decision applications.
- **Future-only.** A revoked decision matches nothing from the moment of revocation. Observations
  that already recorded it keep their `applied_fp_decision_id` and their `false_positive_match`, and
  their `finding_states` history is untouched — §10.4 is explicit that states *"apply to that
  observation only"*. Revocation is not retroactive rewriting, and a report already assembled from
  an immutable input snapshot (§16.4) does not change.
- **Never reactivates.** `false_positive_events.action` already permits only `'revoked'`, and that is
  correct: re-suppressing is a **new decision** with a new id, a new actor, a new timestamp and a
  fresh key computed under the current `field_semantics_version`. A reactivation path would let a
  decision made under one set of field meanings silently resume under another.
- **The blast radius is shown before, not after.** Revocation is the one operation whose effect is
  countable in advance, and so is suppression: the UI states how many currently-suppressed
  observations a decision covers. A user who cannot see what a suppression reaches cannot be said to
  have reviewed it. **The unit of that count is the distinct match key, not the `findings` row**, and
  the difference is not a refinement — it is the difference between a number that means something and
  one that does not. `findings` holds one row per observation per scan (§10.4, and §6.3's
  `UNIQUE (scan_id, source_finding_id)`), so a row count rises on every scan for a decision whose
  reach never changed: a decision covering one finding reads "30" after thirty scans. A user shown
  that number cannot distinguish a suppression that travelled from a scanner that ran often, which is
  the one judgment the number exists to support. So:

  > **Reach** = `COUNT(DISTINCT fp_match_key) FROM findings WHERE tenant_id = $1 AND
  > applied_fp_decision_id = $2`, over the generated column of migration (3).

  It runs on an interactive path (§10.5) before the user confirms, and §12 runs it again per scan per
  decision as a revisit-trigger instrument. Stating both the unit and the query here is what makes
  the covering index in §1 a consequence of the decision rather than a Phase 1 surprise, and it is
  why (3) carries `fp_match_key` instead of being an index on `applied_fp_decision_id` alone.
- **Reason recorded — and this one needs a column.** `false_positive_decisions.reason` exists for the
  decision. The revocation event carries its own actor and timestamp, but TDD §6.3's
  `false_positive_events` is `(id, tenant_id, decision_id, action, actor_id, created_at)`: there is no
  column a revocation reason can be written to. So either the reason is not persisted — in which case
  this bullet is unsatisfied and §10.2's audit trail is incomplete — or `false_positive_events` gains
  **`reason text`**. This record takes the second: the reason is part of the audit trail, and the
  migration is named in §1 rather than described as a UI concern. An earlier draft of this bullet
  called it *"a UI requirement, not a schema change"*; that was wrong and is corrected here.

---

## 7. Scanner version bumps

### 7.1 A bump alone invalidates nothing

The obvious rule — invalidate every decision on any scanner bump — was rejected. `nuclei-templates`
ships roughly every three weeks: `v10.4.2` 2026-04-15, `v10.4.3` 2026-05-05, `v10.4.4` 2026-05-28,
`v10.4.5` 2026-06-23, `v10.4.6` 2026-07-16, `v10.4.7` 2026-08-03, `v10.4.8` 2026-08-24, `v10.4.9`
2026-09-16. Wiping every decision at that cadence makes the feature useless, and a user whose
false-positive marks evaporate every three weeks stops marking false positives and starts disabling
checks instead. That is strictly worse: a disabled check produces no evidence at all.

### 7.2 What invalidates is a declared field-semantics change

Each scanner carries a `field_semantics_version`, stored on every decision. **The scanner-pin PR that
changes a pinned image or template pack must either leave it alone or increment it**, and incrementing
it is required when the bump changes the meaning, presence, or population of **any** field named in
§4 for that scanner.

Decisions stored under a superseded value become **stale**: retained, shown to the user as stale with
the version that produced them, and **suppressing nothing**. Stale is not revoked — the record of the
user's judgment survives — and re-affirming is a new decision under §6.

**Where the *current* value lives.** The stored value is in `match_scope`; staleness is a comparison
against a current value, and that value is **committed configuration in `vulcanflow/platform`, in the
same CI-validated file set as the §5 alias registry** — not a database row and not an environment
variable. That is what keeps §7.2 free of the migrations §1 now enumerates, it is what makes the
increment reviewable in the scanner-pin PR that causes it, and it gives
`supply-chain/check-catalog` something concrete to assert against: the file exists, every
`scanner_id` in §3.1's enum has a row, and the pin PR that changes an image digest either leaves the
row alone or increments it. (Advisory A6.)

This hangs off the existing bump checklist in ADR-0002 §6.3 and the approved check/template catalog
in §21.3, and it is the behaviour §25's `supply-chain/check-catalog` has to cover for templates. The
nuclei `ScanType`'s redirect-following setting (§4.5a) is a property of that same catalog.

### 7.3 Why this gate is mandatory and not advisory — the httpx `host` case

The field `host` in httpx's `runner.Result` is the same name across releases and has changed meaning:

| httpx ref | `runner.go` assignment |
|---|---|
| `v1.3.5` | `Host: ip` |
| `v1.6.0` | `Host: ip` |
| `v1.12.0` | `Host: parsed.Hostname()`, with `HostIP: ip` carrying the address |

So a parser written against `v1.6.0` puts an **IP address** into the component §4.4 designates as
`canonical_host`, and the same parser against `v1.12.0` puts a **hostname** there. Identical field
name, inverted meaning, no compile error, no schema violation — the SCB envelope permits any
`attributes` shape. Decisions stored under the old reading would be keyed on an address, which is
precisely the collapse §8.3.2 exists to prevent. This is not a hypothetical: it is a verified change
in a tool already named in §21.3 and §24.1.

### 7.4 The nuclei template case, which the key cannot see on its own

`template-id` is the check class, and template content moves independently of the nuclei binary.
Three cases, two of which the key handles and one of which it cannot:

| Upstream change | What the key does | Verdict |
|---|---|---|
| Template renamed | New `check_id`; nothing matches | Correct. No carry-over without an alias (§5.6) |
| Matcher added under the same id | New `matcher_name`; new key; nothing matches | Correct. Noise, not silence |
| Matcher **renamed** under the same id | New `matcher_name`; nothing matches | Correct. Noise |
| An existing named matcher **broadened** under the same id | Key is unchanged; the old decision still matches a check that now detects more | **The key cannot see this.** §7.2's gate is the only thing that catches it |

The first three rows are not hypotheses: §8.4.1a records a rename, seven removals and three
additions in a single template id across two refs this record read.

The bump checklist therefore requires, for every `template-id` with at least one stored decision in
any tenant: diff the template's matcher-name set and matcher bodies between the outgoing and incoming
template pack, and increment `field_semantics_version` for `nuclei` if any matcher body changed. This
is the single place where a false-positive suppression can widen without any field changing, and it
is why §7 is part of this record rather than a note in an operations runbook.

---

## 8. Negative examples — pairs that look equivalent and must not be

Each case states the pair, the loose key that collapses it, and the consequence. These are the corpus
for `findings/fp-only-persistence`, and **the corpus is the enumeration in §9.1, not a count stated
here.** Three successive revisions of this record stated a count — "ten", then "eleven" — and all
three were wrong against this section's own contents, each time because a case was added or
reclassified without the number being recomputed. A count is a claim that has to be re-derived on
every edit; a list is a claim that edits itself. §9.1 lists the cases by identifier, says which are
negative and which are positive controls, and is the only place Ledger should read the corpus from.
What this section guarantees is the property, which does not change when a case is added: **at least
two negative cases per scanner, each with its loose key and the consequence of the collapse.**

### 8.1 subfinder

**8.1.1 — PSL broadening.** A user marks `old-staging.example.com` a false positive: a decommissioned
host whose stale record sits in a third-party zone.

- Loose key `(check_id, registrable_domain = example.com)` also matches `admin.example.com`.
- Correct key includes `canonical_host`, and the two differ.
- **Consequence:** one click suppresses every subdomain the discovery stage will ever find for that
  tenant — the entire output of the first pipeline step. This is the same failure class as §5.3's
  scope broadening, which §25 covers as `authz/psl-exact-root`; the shared `vf-core` canonicalizer
  (§3.3) is what keeps the two from diverging.

**8.1.2 — the `attributes.domain` trap.** `attributes.domain` is `item.input` in
`scanners/subfinder/parser/parser.js` — the **root that was queried**, not this finding's host. Under
`INCLUDE_TARGET_DOMAIN=true` the parser emits a synthetic finding where `getTargetDomainFinding` sets
`domain === hostname === ` the apex, so a parser author who reads `attributes.domain` as "the
finding's host" is correct for that one synthetic finding and wrong for every real one.

- Loose key on `attributes.domain`: a false positive on the apex finding matches **every** discovered
  subdomain, because they all carry the same `attributes.domain`.
- Correct key uses `attributes.hostname`, and `scope_root` comes from our own `targets` row (§3.5),
  not from this field at all.
- **Consequence:** total suppression of the discovery stage from a single click on the apex — and the
  bug is invisible in any fixture that contains only the synthetic finding.

**8.1.3 — the positive control (the matcher must not be too narrow either).** `api.example.com`
reported with `attributes.source: "crtsh"` and `attributes.ip_address: "203.0.113.9"`, and the same
host on the next scan with `attributes.source: "virustotal"` and `attributes.ip_address: null`.
These are one subdomain. The key **must** match, because `source` and the address fields are excluded
by §4.2. A matcher that keys on them re-surfaces every suppressed subdomain on every scan, which is
how a user learns to distrust the feature.

### 8.2 dnsx

**8.2.1 — the record value must be in the key.** A user marks `legacy.example.com A 203.0.113.10` a
false positive, reasoning that the host is decommissioned and the record is stale. A later scan
returns `legacy.example.com A 198.51.100.77`.

- Loose key `(check_id = dnsx/dns-record:A, canonical_host)` suppresses it.
- Correct key includes the record-value digest, and the two differ.
- **Consequence:** the name now resolves somewhere else — the exact signal that a dangling record has
  been claimed by someone — and it is suppressed by a decision whose stated premise was that the
  record was stale. The decision's own reasoning is what the new value refutes.

**8.2.2 — the record type must be in the key.** Same false positive on the `A` observation for
`legacy.example.com`.

- Loose key `(canonical_host)` alone also suppresses the `CNAME` observation for that name.
- Correct key includes `check_id`, which encodes the RRTYPE.
- **Consequence:** the CNAME is the dangling-resource evidence §27 item 13 is about. Suppressing the
  `A` record and losing the `CNAME` with it destroys the one observation that distinguishes "stale
  record" from "takeover".

**8.2.3 — wildcard DNS makes value-only matching catastrophic.** With `*.example.com` in place,
`nonexistent-1234.example.com` and `nonexistent-5678.example.com` return the **same** `A` value.

- Loose key `(check_id, record-value digest)` without the host makes them one key, along with every
  other name that resolves through the wildcard.
- Correct key includes `canonical_host`.
- **Consequence:** one false positive on a wildcard artefact suppresses every wildcard-resolving name
  — including a real host whose `A` record happens to equal the wildcard target.

### 8.3 httpx

**8.3.1 — the `final_url` trap, and an attacker-settable key component.** Forty services in one
tenant each return `302` to `https://sso.example.com/login`.

- Loose key using `final_url` for `canonical_host`/`location` makes all forty one key.
- Correct key derives host, port, scheme and path from `url` — never from `final_url` (§4.4).
- **Consequence:** a false positive on one suppresses thirty-nine services nobody reviewed. Worse:
  `final_url` is derived from a `Location` header returned by **the scanned host**. A key component an
  outside party can set is a suppression primitive — anyone who can set a redirect on one in-scope
  host can aim an existing decision at a different one.

**8.3.2 — shared infrastructure.** `blog.example.com` and `payments.example.com` both resolve to
`host_ip: 203.0.113.5`, both `port: 443`, `scheme: https`, `path: /`.

- Loose key using `host_ip` makes them byte-identical.
- Correct key uses `host` (the hostname) and excludes `host_ip` entirely (§4.4).
- **Consequence:** a false positive on the marketing blog suppresses the payment service. §10.2 says
  *"Preserve hostname context on shared infrastructure"* and §6.2 says *"IP/port alone is not a
  universal asset identity"*; this is the case both sentences were written for.

**8.3.3 — port.** `https://example.com/` and `https://example.com:8443/`, the second being an exposed
admin interface.

- Loose key without `port` collapses them.
- Correct key includes `port`, defaulted from the scheme on the first (§3.3) so that
  `https://example.com` and `https://example.com:443` do **not** become two keys.

**8.3.4 — the version trap.** See §7.3. A decision stored by a parser built against httpx `v1.6.0`
carries an **IP** in the slot §4.4 reserves for `canonical_host`, which turns 8.3.2 from a design
error into a silent regression introduced by an image bump. This is why §7.2's gate is mandatory.

### 8.4 nuclei

**8.4.1 — one template id, sixteen findings. (The centrepiece, and it ships as an upstream fixture.)**
`scanners/nuclei/parser/__testFiles__/secureCodeBox-test.jsonl` at secureCodeBox `v5.9.0` contains
**sixteen** findings with `template-id: http-missing-security-headers`, all with
`host: https://www.securecodebox.io`, all with the **identical** `matched-at`, all with
`type: http`, differing **only** in `matcher-name`: `referrer-policy`,
`access-control-allow-origin`, `access-control-allow-credentials`, `access-control-expose-headers`,
`access-control-allow-methods`, `x-content-type-options`, `x-permitted-cross-domain-policies`,
`access-control-max-age`, `content-security-policy`, `cross-origin-opener-policy`, `x-frame-options`,
`cross-origin-resource-policy`, `cross-origin-embedder-policy`, `access-control-allow-headers`,
`permission-policy`, `clear-site-data`.

- Loose key `(canonical_host, check_id)` makes all sixteen one key.
- Correct key includes `matcher_name` in the `discriminator` (§4.5).
- **Consequence:** a user who decides that one missing header is acceptable on that host suppresses
  **fifteen other findings they never saw**, on the same host, for an unbounded future. Ledger can use
  this file verbatim: it is upstream's own fixture, it needs no fabrication, and it is the single
  clearest demonstration that `(host, check)` is not an equivalence class.

**8.4.1a — and the same pair of refs demonstrates §7.4 without anyone constructing it.** The
template at `nuclei-templates` `v10.4.9`
(`http/misconfiguration/http-missing-security-headers.yaml`) declares **twelve** named matchers
under `matchers-condition: or`: `content-security-policy`, `content-type-charset-specification`,
`cross-origin-embedder-policy`, `cross-origin-opener-policy`, `cross-origin-resource-policy`,
`missing-content-type`, `permissions-policy`, `referrer-policy`, `strict-transport-security`,
`x-content-type-options`, `x-frame-options`, `x-permitted-cross-domain-policies`.

The fixture above was captured at an **earlier** template pack, and the two sets do not agree:

| Change between the fixture's pack and `v10.4.9` | Matcher names |
|---|---|
| **Renamed** | `permission-policy` → `permissions-policy` |
| **Gone** | the six `access-control-*` matchers, `clear-site-data` |
| **New** | `strict-transport-security`, `missing-content-type`, `content-type-charset-specification` |

So all three of §7.4's visible cases are already on the record for one template id, read from two
refs rather than imagined: a rename, removals, and additions, with the `template-id` unchanged
throughout. Under §4.5 each of them yields a different `discriminator` and therefore no carry-over —
which is the correct outcome and is noise, not silence. The one case these refs do *not* show is a
matcher body broadened under an unchanged name, which is exactly the case §7.2's gate exists for.

**8.4.1b — and a third pack is in the same directory, which is why §7.2's gate is about *declared*
semantics and not about release numbers.** `example-com-test.jsonl` at the **same** `v5.9.0` ref
carries **seventeen** `http-missing-security-headers` findings for one host: §8.4.1's sixteen matcher
names **plus** `strict-transport-security`. So one upstream tree contains two fixtures captured at two
different template packs (16 and 17 matchers) and the pinned pack `v10.4.9` is a third (12). "New
relative to the fixture's pack" is the only claim §8.4.1a makes, and this is why it is worded that
way: "new since v5.9.0" would be false, because v5.9.0 itself contains two different packs' output.
A record that keyed anything to a *release number* rather than to a declared
`field_semantics_version` (§7.2) would already be wrong inside one upstream tag.

**8.4.2 — `location` is URL-shaped even when the finding is not HTTP, so `protocol` must be in the
key.** In the same fixture, `cname-fingerprint` has `type: dns`, `tls-version` and `ssl-dns-names`
have `type: ssl`, and the rest have `type: http` — and **all** of them carry
`host: https://www.securecodebox.io`. The v5.9.0 parser sets the envelope `location` to
`finding.host` verbatim, so a DNS finding gets an `https://` location.

- Loose key using the envelope `location` as the host, with no `protocol`, collapses findings across
  the DNS, TLS and HTTP layers of one name.
- Correct key uses `attributes.hostname` (the parser's `parseHostname`, which strips the scheme) plus
  `attributes.type`.
- **Consequence:** besides the collapse, this is the general argument against the "generic
  target + check + location" matcher the issue warned off. The SCB schema documents `location` as
  *"Full URL with protocol, port, and path if existing"*, and subfinder puts a bare hostname there
  while nuclei puts a URL-shaped string on a DNS finding. `location` has no cross-scanner meaning, so
  a cross-scanner key over it is a key over an undefined field.

**8.4.3 — `extracted_results`.** In the same fixture `email-extractor` on
`https://www.securecodebox.io` extracted `securecodebox@iteratec.com`. A user marks it a false
positive: that address is published on purpose.

- Loose key excluding `extracted_results` suppresses a later `email-extractor` finding on the same
  host that extracted an internal address leaked into the page.
- Correct key includes the set digest of `extracted_results` (§3.3, §4.5).
- **Consequence:** the finding the user actually cared about is the one that gets hidden, by a
  decision about a different value. Accepted cost of the other direction: `ssl-dns-names`
  (which extracted `docs.securecodebox.io` and `www.securecodebox.io` here) will re-surface when a
  certificate is reissued with a different SAN set. That is noise.

**8.4.4 — `ip_addresses`.** In `secureCodeBox-test.jsonl` at `v5.9.0`, **21 of the 22** findings carry
`ip: 34.159.58.69` — every finding but the `dns` one, `cname-fingerprint`, which has no `ip` key at
all and for which the parser emits `ip_addresses: []`. (An earlier draft said *"all"*; counted at the
tag on 2026-10-02, it is 21 of 22. The argument is unaffected and the exception is itself useful: the
one finding with no address is the one whose observation class has no address dimension.) Keying on it
reproduces §8.3.2 exactly: two hostnames behind one address become one key. Excluded, per §4.5.

**8.4.5 — severity is not an identity, and cannot be made into one.** `findings-schema.json` at
`v5.9.0` restricts `severity` to `INFORMATIONAL | LOW | MEDIUM | HIGH`, and the v5.9.0 nuclei parser's
`getAdjustedSeverity` maps `CRITICAL → HIGH`, `INFO → INFORMATIONAL`, `UNKNOWN → LOW`. So severity is
lossy, scanner-mapped, and version-dependent; a key component must be none of those, and it is
excluded.

> **This also corrects ADR-0003 §3.5.** That section correctly warns that `Scan.status.findings` has
> only four severity buckets and that VulcanFlow's severity model must not be derived from it — but
> its remedy, *"read severities from the findings artifact, which carries the parser's actual severity
> values"*, does not hold. The artifact cannot carry `CRITICAL`: the envelope schema's enum has no
> such value, so **no schema-conformant secureCodeBox parser can ever emit it**, and the nuclei parser
> collapses it before the artifact is written. The original value is not preserved anywhere in
> `attributes`. VulcanFlow's severity must therefore be derived at ingest from enrichment — which is
> what §10.3 already prescribes (`KEV → EPSS → CVSS`) — not read from the artifact. Recorded as
> ADR-0003 amendment **A1 §2**; the product-facing severity model is named in "Does not close" above
> because the board settled its direction on 2026-10-02 and the record for it is a separate ADR, not
> because it is unimportant.

**8.4.6 — the broadened matcher.** See §7.4. A template that keeps its id and its matcher names while
broadening what an existing matcher detects produces an unchanged key, and an existing decision keeps
applying to a check that now detects more. The key cannot see it; §7.2's bump gate is the only
control, which is why that gate is in this record.

**8.4.7 — `matched_at` on a host that is not the finding's host. (Upstream fixture, added in review,
and it is the one §4.5a exists for.)** `scanners/nuclei/parser/__testFiles__/example-com-test.jsonl`
at secureCodeBox `v5.9.0` — same directory as §8.4.1's fixture — ends with the `azure-domain-tenant`
finding quoted in §4.5a: `host: https://example.com`, but
`matched-at: https://login.microsoftonline.com:443/example.com/v2.0/.well-known/openid-configuration`
and `ip: 40.126.32.140`. The finding is *about* `example.com`; the match happened on Microsoft's
login endpoint.

- **Loose key** sources `port` and `location` from `matched_at` without checking its host, and keys
  the observation as `example.com:443/example.com/v2.0/.well-known/openid-configuration`. Two
  different templates that each probe a different third-party endpoint on behalf of the same tenant
  domain can then share `canonical_host`, `check_id`-adjacent structure and a path neither host
  serves; worse, the stored key asserts a location that does not exist on the host it names, so
  nobody auditing the decision can tell what was actually suppressed.
- **Correct key** applies §4.5a: `matched_at`'s canonicalized host (`login.microsoftonline.com`) is
  not the observation's `canonical_host` (`example.com`), so `port` and `location` are `Unknown`, and
  §3.4 makes the observation non-storable and non-matchable — 422 on an attempt to mark it, and no
  match ever.
- **Consequence of the collapse:** the second, sharper form is the redirect case in §4.5a. `T` fires
  at `https://app.example.com/legacy/debug` and is marked a false positive; later `T` fires on a newly
  exposed `/v2/debug` on the same host which 302s to `/legacy/debug`; with redirect following on,
  `matched_at` becomes the suppressed path and the key is byte-identical. **The host operator chooses
  which of our findings disappear.** That is §8.3.1's "suppression primitive", not merely a collapse —
  which is why §4.5a both disables redirect following in the `ScanType` and makes the divergent-host
  key unstorable. One control for the mistake, one for the adversary.

---

## 9. What has to be built and tested, and by whom

**§25 identifier.** `findings/fp-only-persistence` (§10.2) is the one §25 identifier this record
unblocks. It stays **one test function**, table-driven over the corpus enumerated in §9.1 — the §25
mapping is injective in both directions and this record does not change that. The table asserts both
directions: the negative rows must **not** match, and the positive-control rows **must**.

### 9.1 The corpus, enumerated

This list, not a count anywhere in this record, is what Ledger builds the table from. Every §8 item
appears exactly once, with where it belongs. A case added to §8 by a future amendment is added here in
the same edit; that is the obligation the enumeration carries instead of a number.

**Negative rows — the pair must not match** (13):

| Case | Scanner | The loose key it rules out |
|---|---|---|
| 8.1.1 | subfinder | registrable domain instead of the full canonical host |
| 8.1.2 | subfinder | `attributes.domain` (the root queried) instead of the finding's host |
| 8.2.1 | dnsx | record type and name without the record **value** |
| 8.2.2 | dnsx | name and value without the record **type** |
| 8.2.3 | dnsx | value-only matching under a wildcard zone |
| 8.3.1 | httpx | `final_url` — attacker-settable, and it collapses an SSO redirect fan-in |
| 8.3.2 | httpx | the resolved address instead of the canonical host |
| 8.3.3 | httpx | host without `port` |
| 8.4.1 | nuclei | `(host, template-id)` without the `matcher_name` sub-value — **upstream fixture** |
| 8.4.2 | nuclei | location without `protocol`, on a URL-shaped location for a DNS finding |
| 8.4.3 | nuclei | template and location without the `extracted_results` sub-value |
| 8.4.4 | nuclei | `attributes.ip_addresses` — §8.3.2's collapse reached by a different field |
| 8.4.7 | nuclei | `port`/`location` from a divergent-host `matched_at` — **upstream fixture** |

That satisfies §8's stated property at two subfinder, three dnsx, three httpx and five nuclei cases.

**Positive-control rows — the pair must match** (4). A matcher that is too narrow re-surfaces every
suppressed finding on every scan, and a corpus of only negative rows cannot catch it:

| Case | Asserts |
|---|---|
| 8.1.3 | subfinder `source` and address fields are excluded, so one subdomain across two scans is one key |
| 8.4.5 | envelope `severity` is excluded, so two observations differing only in severity are one key — the field is lossy, scanner-mapped and version-dependent, which is why it cannot be identity |
| §3.3 accepted noise | `?a=1&b=2` vs `?b=2&a=1` are **two** keys (accepted noise, asserted as such, not as a match) |
| §3.3 duplicates | `["a","a"]` and `["a"]` digest differently (likewise asserted as two keys) |

**Not rows of this identifier**, and named so nobody adds them to the wrong table:

| Case | Where it belongs |
|---|---|
| 8.3.4 | `findings/fp-scanner-semantics-stale` — it is §7.3's httpx `host` inversion, a staleness case, not a key-shape case |
| 8.4.6 | `findings/fp-scanner-semantics-stale` — §7.4's broadened matcher produces an **unchanged** key by design; the key cannot detect it and the §7.2 gate is the only control |
| 8.4.1a, 8.4.1b | Upstream evidence for §7.2/§7.4 — three template packs disagreeing inside one secureCodeBox tag. They support the gate's design; they are not pairs to assert |

**Adjunct test ids**, outside §25, for the mechanism §25 does not name — the same pattern ADR-0003 §3.4
used for `scb/hook-invocation-contract`:

| Test id | Asserts | Author |
|---|---|---|
| `findings/fp-match-key-canonicalization` | §3.3 exactly: UTS-46 Processing applied to the original input with **no NFKC pre-pass** — specifically that a `disallowed` dot-like code point (`U+2024`, `U+2025`, `U+FE52`) yields `Unknown` rather than canonicalizing to a different valid host, while a `mapped` one (`U+FF0E`, `U+3002`, `U+FF61`) becomes a label separator; case folding and trailing-dot; a rejected host yielding `Unknown`, default-port folding **and that no scheme outside `http`/`https` defaults**, fragment stripping, no percent-decoding, **per-element** length-prefixed set digests (the `["ab","c"]` vs `["a","bc"]` case **and** `["a","a"]` vs `["a"]` digesting differently), the §3.3 `discriminator` tag-and-length serialization including all four nuclei sub-value combinations and `0x00 0x00` ≠ `NotApplicable`, and `Unknown` → 422 on create and no match at read | Scribe (unit + proptest) |
| `findings/fp-nuclei-field-shapes` | §4.5a/§4.5b/§4.5c, the rules review added: `matched_at` host ≠ `canonical_host` ⇒ `port`/`location` `Unknown` ⇒ 422 and no match (the §8.4.7 fixture row); **§4.5c is evaluated before §4.5a and `NotApplicable` is never promoted to `Unknown`** — `type: dns` with a divergent `matched_at` stays storable; all six branches of §4.5b's table, each in both outcomes, and specifically that `[2001:db8::1]` and `[2001:db8::1]:8443` key to `canonical_addr` rather than `Unknown`, that bare `2001:db8::1` keys to `canonical_addr` with **no** port rather than to host `2001:db8:` port `1`, that `example.com:8443` splits, that `example.com:99999` and `example.com:0` are `Unknown` in **both** components, and that `null` ⇒ `Unknown`; `type: dns` ⇒ `port` **and** `location` `NotApplicable`; an unspecified `type` ⇒ `Unknown`. The invariant to assert over the whole branch table is the §1 direction: **every rejected input yields `Unknown`, never a parsed key** | Scribe (unit, table-driven over the two upstream fixtures plus constructed IPv6 rows, which no fixture supplies) |
| `findings/fp-alias-non-transitive` | §3.1.1, §5.2 and §5.7: `check_id` is the only aliasable component — an edge-shaped difference in any of the other eleven matches nothing; one hop only, so `A→B, B→C` does not match `A` against `C`; reverse direction does not match; cycles and duplicate `from_check_id` rejected at registry load; cross-scanner edge rejected; §3.6's precedence rule — exact hit beats alias hit, and lowest `(created_at, id)` among several alias hits | Scribe |
| `findings/fp-revocation-not-retroactive` | §6: revoked decision matches nothing afterwards; already-applied observations keep `applied_fp_decision_id` and their state history; no reactivation path | Ledger (integration) |
| `findings/fp-scanner-semantics-stale` | §7.2: a decision stored at `field_semantics_version = 1` suppresses nothing once the scanner is at 2, is retained and reported stale, and is not revoked | Ledger |

**Lane ownership** (ADR-0005). Spec: this record. Key builder and matcher: **Forge**, in `vf-core` —
pure, bytes in and a verdict out, no I/O, so it is property-testable and fuzzable without standing
anything up (**crate purity**). Application at ingest and the 422 path: **Anvil**, in `vf-ingest`.
Unit and property tests: **Scribe**. Integration, conformance and the §8 corpus: **Ledger**.
Execution and the PASS/FAIL/MISSING ledger: **Crucible**. Review: **Assay** and **Warren**.

**Fixtures.** The nuclei corpus is upstream's own files at a pinned ref
(`secureCodeBox/secureCodeBox` `v5.9.0`,
`scanners/nuclei/parser/__testFiles__/secureCodeBox-test.jsonl` for §8.4.1–§8.4.5, and
`…/example-com-test.jsonl` for §8.4.7 and `…/hostname-without-port.jsonl` for §4.5b).
The subfinder, dnsx and httpx cases are constructed from the field tables in §4 and need no cluster
and no network. `findings/fp-only-persistence` must not require a running scanner.

**Blocking dependencies.** `findings/fp-only-persistence` was blocked on this record and is now
unblocked. Nothing else was blocked on it. This record is blocked by nothing: §4's `dnsx` and `httpx`
rows specify the mapping our own parsers must produce and do not wait on the scanner-image pin, which
is Phase 2 (§24.2) and is listed in "Does not close".

---

## 10. Alternatives considered, and why each lost

| Alternative | Why it lost |
|---|---|
| **Content hash of the whole finding** | Changes on every scan — `parsed_at`, `timestamp`, `curl_command`, response bodies — so it suppresses almost nothing and the feature appears broken. §10.2 and §21.3 also forbid deriving identity from finding content. |
| **CVE / CWE as the equivalence class** | §10.2 rules it out in terms: *"It is not sufficient to match a CVE or IP alone."* One CVE across 200 hosts means one click suppresses 200. And three of the four Phase-1 scanners emit no CVE at all, so it would not even cover the pipeline. |
| **A generic scanner-agnostic `target + check + location` tuple** | `location` has no cross-scanner meaning: the SCB schema calls it a full URL, subfinder puts a bare hostname in it, and nuclei puts a URL-shaped string on a DNS finding (§8.4.2). A generic key over it is a key over an undefined field, and §8.3.1/§8.4.2 are the collapses that follow. |
| **Fuzzy or learned similarity matching** | Non-deterministic, not table-testable, and unexplainable. A suppression you cannot justify to a customer line by line has no place in a security product, and §18.6 already rules AI out of risk scoring — the argument is stronger for suppression than for ranking. |
| **Transitive alias closure** | Each hop adds reach without adding evidence (§5.2). |
| **Automatic aliasing on detected upstream renames** | A disappeared id is exactly as consistent with a changed check as with a renamed one (§5.6). Guessing wrong hides findings. |
| **Invalidate all decisions on any scanner bump** | Template packs move monthly (§7.1); users would stop marking false positives and start disabling checks, which produces less evidence, not more. |
| **Excluding `extracted_results` from the nuclei key** | Suppresses findings carrying a different extracted value under the same template (§8.4.3). Rejected under §1's asymmetry. |
| **`NULL` or `""` for absent components** | `NULL = NULL` is `UNKNOWN`, so a key would not equal itself; `""` merges "not applicable" with "could not determine" (§3.4). |
| **Splitting `discriminator` into two named components** (so nuclei's `matcher_name` and `extracted_results` each get a §3.1 row) | The reviewer's preferred remedy for the ambiguity §3.3 now fixes, and a close call. It lost on generality: it writes one scanner's shape into the generic key, and leaves `subfinder` and `httpx` carrying two permanently-`NotApplicable` rows. A single component with a specified tag-and-length serialization keeps §3.1 scanner-agnostic and puts the shape where the shapes live, in §4. §3.4 applies unchanged either way, because the serialized sequence is one `Present(String)`. **A third leg of this argument is withdrawn, and the cost the row did not weigh is now in it.** The withdrawn leg read *"and still would not escape per-scanner serialization — `dnsx`'s `SRV` discriminator is itself multi-valued (§4.3.1)"*. That is **false**: per §4.3 the dnsx discriminator is **one** sub-value for every record type, `SRV` included — an `SRV` record's port lives inside each record-value string, so it needs the same single set digest as `A` or `TXT` — and §4.3.1 is about `soa`, not `srv`. The unweighed cost is one R12 created after this row was written: §12's second trigger compares two keys for equality in `matcher_name` and difference in `extracted_results`, which two components would give directly and one component gets only through §12's `vf-core` decoder. The decision stands on the two legs that hold, with the cost stated. |
| **Keying nuclei `port`/`location` from `matched_at` whenever the hosts disagree** | Asserts a port and a path on a host where neither exists, and makes the key settable by whoever controls the redirect (§4.5a, §8.4.7). |
| **Dropping `port`/`location` to `NotApplicable` when `matched_at`'s host diverges** | The other way to handle §4.5a, and rejected — but the reason first given here was overbroad and is corrected. It said the option *"collapses every third-party-probe template on one host into one key"*, which cannot happen: `check_id` is a key component (§3.1), so two different templates never share a key however `port` and `location` are set. What it actually collapses is **every divergent-host observation of the *same* template on one host** — one `azure-domain-tenant` match at `login.microsoftonline.com` and another at a different third party both key to `NotApplicable`/`NotApplicable`, so one click suppresses both, and a user who reviewed one has silently cleared the other. It also writes "this check has no location dimension" into a key for a check whose §4.5c row says it does, which is the `""`-versus-`NULL` conflation of §3.4 in a different costume. The conclusion is unchanged: `Unknown` → 422 is the only answer consistent with §1. |

---

## 11. What this record does not decide

- **§27 item 13 — DNS risk labels.** Still open, still Engineering's. §4.3 covers the DNS *record*
  observation class only. A dangling-resource check class is a **new `check_id`**, and under §4 it
  cannot create or receive a suppression until an amendment adds its row and its negative example.
  Stated as a blocker edge, not as prose: item 13 blocks any `dnsx/dangling-*` check class.
- **The scanner image and template-pack digest pins.** §21.3's job and Phase 2's (§24.2). This record
  pins the **refs the field names were read at**, which is what the §9 tests need, and sets
  `field_semantics_version = 1` against those refs. The pin ADR must re-verify §4 against whatever it
  pins and declare the version it corresponds to (§7.2).
- **The product-facing severity model** that §8.4.5 shows is needed. Named here, settled elsewhere:
  the board decided on 2026-10-02 that customer-visible severity is **derived at ingest** from TDD
  §10.3's `KEV → EPSS → CVSS`, with the scanner's `severity` retained as evidence only, and that the
  record must also fix the fallback for findings carrying no CVE — which is most of Phase 1's output,
  since `subfinder` and `dnsx` emit none and most nuclei misconfiguration and exposure templates emit
  none either. That record is a separate ADR and is **not** this one; until it is accepted the item
  stays in `decisions/README.md` "Still open", because a decision recorded only in an issue thread is
  not the design of record. Nothing in §3–§4 depends on the outcome: envelope `severity` is excluded
  from the key either way (§8.4.5), and that exclusion is the only interaction between the two
  records.
- **Revocation UI.** §6 fixes the semantics and the requirement that reach be shown before
  confirmation, not the surface.
- **Dedup of repeated artifact delivery.** A separate concern per §10.2 and §25's
  `findings/replayed-artifact`; the `UNIQUE (scan_id, source_finding_id)` constraint in §6.3 handles
  it and nothing here changes that.

---

## 12. Revisit triggers

- **A fifth scanner in the pipeline** — `nmap` or the `masscan` pool (§24.2 Phase 2), `tlsx`, or
  Amass (§27 item 16). Each needs its own §4 row, its own `field_semantics_version`, and at least one
  negative example **before** it may create decisions. Required amendment, not an implementation
  detail.
- **A decision whose reach grows — measured, not waited for.** The original wording of this trigger
  was *"any confirmed case of a decision applying to a finding a user says they never reviewed"*,
  which is the exact failure §1 says this record exists to prevent and had **no observable signal**:
  it fires only when a customer notices and reports it, which is the discovery path §3.4 rejects in
  terms (*"it would be found by a customer, not by us"*). A revisit trigger we cannot observe is a
  hope, not a trigger. The instrument already exists — §6's **reach**, the number of distinct match
  keys a decision currently suppresses — so the trigger is that number:
  **alert when a decision's reach exceeds a configured N, or when a match key appears under that
  decision that was not under it in the scan the decision was created in.** A decision whose reach
  keeps growing is the definition of a suppression that travelled. A confirmed user report remains a
  P1 that reopens §3–§4 immediately; it is now the backstop rather than the detector.

  > **The unit is the distinct key, and this trigger is the reason it has to be.** Stated over
  > `findings` **rows** — "applied-observation count exceeds N, or grows in any scan after the
  > creating scan" — the growth limb fires for **every** decision on **every** subsequent scan,
  > because each scan writes a fresh observation row for the same finding (§10.4). A trigger that
  > fires on everything identifies nothing, and it would have been worse than the unobservable one it
  > replaced: that one was silent, this one would have been noise with an alert attached. Over
  > distinct `fp_match_key` values both limbs mean what they say — N is a reach, and growth is a
  > genuinely new identity entering the decision's shadow. Found by the second automated reviewer;
  > the same counting-unit class as the §2.1 contradiction the first pass found, and the third defect
  > in this record traceable to a number stated where a set was meant.
- **Re-surfacing, made countable.** The original *"sustained complaints about re-surfacing"* had no
  threshold and no measure, which makes it a preference rather than a decision input. The measurable
  form: **the rate of `new` observations whose key differs from a suppressed one in the
  `extracted_results` sub-value of the `discriminator` and in nothing else** (§4.5). That is the exact
  signal, not a proxy for it, and it is computable from the keys we already store — **but only because
  §3.1 requires `false_positive_match` to be written on observations that matched nothing**, which is
  exactly the population this trigger measures. Two mechanics it needs, named so the trigger is
  implementable rather than aspirational: the comparison is over the eleven byte-equality components
  with `discriminator` decoded, so `vf-core` exposes the inverse of §3.3's tag-and-length
  serialization — `decode_discriminator(&str) -> Result<Vec<MatchComponent>>`, pure, and the
  round-trip is a property `findings/fp-match-key-canonicalization` already owns (§9). And the
  comparison is only meaningful within one `(match_version, field_semantics_version, scanner_id)`,
  because across a `field_semantics_version` boundary §7.2 has already staled the decision and the
  difference is not re-surfacing. If the rate is high, the fix is a per-template
  `extraction_is_evidence` allowlist behind a `match_version` bump — not an ad-hoc exclusion, and not
  a silent one.
- **`findings-schema.json` changing** at a secureCodeBox bump: the `severity` enum, the `location`
  description, or any constraint on `attributes`. Part of ADR-0002 §6.3's checklist.
- **Upstream secureCodeBox adding `dnsx` or `httpx` scanners.** That would replace our parsers and
  their field mapping, and §4.3/§4.4 would be rewritten against upstream's `attributes` shape instead
  of ours.
- **Any scanner invocation acquiring a TLS-SNI or redirect-following setting that §4.4a and §4.5a
  forbid.** Both exclusions are exact only while the catalog holds those settings off, so a change to
  either is a `match_version` bump and an amendment here, not a catalog edit — `sni` would become a
  key component and nuclei's `matched_at` would stop being host-stable. This trigger is observable in
  the one place that matters: the diff of the catalog file, which `supply-chain/check-catalog`
  already asserts against.
- **A need for cross-scanner aliases** (§5.5) or for aliases that move a location component (§5.1).
  Both are amendments with evidence, and both are the kind of request that should be refused by
  default.

---

## 13. Provenance

Every field name, enum value, default, and quoted line above was read from
`raw.githubusercontent.com` and the GitHub contents API at the exact refs named — the original set on
**2026-10-01**, and the material added in review on **2026-10-02**: the `example-com-test.jsonl` and
`hostname-without-port.jsonl` fixtures and the `parseHostname` body (§4.5a, §4.5b, §8.4.7), the `SOA`
struct's eight fields and `DNSData.AllRecords` (§4.3.1, §4.3), httpx's `Location`, `SNI`, `Failed` and
`Error` tags with `Location`'s assignment at `runner/runner.go:2685` (§4.4), and the §8.4.4 recount
(21 of 22, not 22 of 22).

The re-review round added one upstream read and one standards fact, both for §4.5b:
`parseHostname`'s third branch returns **`new URL(host).hostname`** — re-read at
`scanners/nuclei/parser/parser.js` lines 96–118, `secureCodeBox/secureCodeBox` `v5.9.0`, on
**2026-10-02** — and WHATWG URL defines `hostname` to serialize an IPv6 address **with its brackets**
(URL Standard, host serializer), which a check against that serializer confirms:
`new URL("https://[2001:db8::1]:8443/x").hostname === "[2001:db8::1]"`. That is the fact B5 turns on,
so it is cited rather than asserted.

The third automated pass added three more, all read on **2026-10-02** and all from primary sources
rather than from recollection of what IDNA does:

- **UTS #46 revision 31, §4 Processing** (`unicode.org/reports/tr46/`) — the step order is Map →
  *"Normalize the domain_name string to Unicode Normalization Form C"* → Break *"into labels at
  U+002E ( . ) FULL STOP"* → Convert/Validate. NFC, and no NFKC anywhere in the algorithm.
- **`IdnaMappingTable.txt` at Unicode 15.1.0** (`unicode.org/Public/idna/15.1.0/`, dated 2023-08-10)
  — `2024..2026 ; disallowed`, `FE52 ; disallowed`, `3002 ; mapped ; 002E`, `FF0E ; mapped ; 002E`,
  `FF61 ; mapped ; 002E`. The `disallowed`/`mapped` split is the whole of Bot-3.1's argument.
- **The Unicode 15.1.0 normalization data** — `NFKC(U+2024) = "."`, `NFKC(U+2025) = ".."`,
  `NFKC(U+FE52) = "."`, against `NFC` leaving all three unchanged. Checked against the normalization
  tables at that version.

No suite was run for any of this, and none of it touches VulcanFlow code: these are reads of
upstream specifications and data files, the same class of probe as §4's field names. Lane 4 is
Crucible's (ADR-0005).

Nothing in this record is recalled:

| Ref | Files read |
|---|---|
| `secureCodeBox/secureCodeBox` @ **`v5.9.0`** | `parser-sdk/nodejs/findings-schema.json`; `scanners/` directory listing (which is how §4.1's missing-scanner fact was established); `scanners/subfinder/parser/parser.js`; `scanners/nuclei/parser/parser.js` (including `parseHostname`, lines 96–118, for §4.5b); `scanners/nuclei/parser/__testFiles__/` listing; `scanners/nuclei/parser/__testFiles__/secureCodeBox-test.jsonl`; `scanners/nuclei/parser/__testFiles__/example-com-test.jsonl` (§4.5a, §8.4.7); `scanners/nuclei/parser/__testFiles__/hostname-without-port.jsonl` (§4.5b); `scanners/subfinder/values.yaml` |
| `projectdiscovery/httpx` @ **`v1.12.0`** | `runner/types.go` (the `Result` struct and its JSON tags, including `Location`, `SNI`, `Failed`, `Error`); `runner/runner.go` (`Host: parsed.Hostname()`, `HostIP: ip`; `Location: resp.GetHeaderPart("Location", ";")` at `:2685`; `resp.SNI = r.options.SniName` at `:1118–1119`); `runner/options.go` (`--sni-name` / `-sni` at `:535`) |
| `projectdiscovery/httpx` @ **`v1.3.5`**, **`v1.6.0`** | `runner/runner.go` (`Host: ip`) — the §7.3 meaning change |
| `projectdiscovery/dnsx` @ **`v1.3.1`** | `libs/dnsx/dnsx.go` (`ResponseData`, `AsnResponse`); `go.mod` (`retryabledns v1.0.116`) |
| `projectdiscovery/retryabledns` @ **`v1.0.116`** | `client.go` (the `DNSData` and `SOA` structs and their JSON tags) |
| `projectdiscovery/nuclei-templates` @ **`v10.4.9`** | `http/misconfiguration/http-missing-security-headers.yaml` (`matchers-condition: or` and the named matchers) |

Release tags and dates were resolved from the GitHub releases API on the same date: `dnsx` v1.3.1
(2026-08-31), `httpx` v1.12.0 (2026-09-08), `subfinder` v2.16.0 (2026-08-22), `nuclei` v3.11.1
(2026-08-08), `nuclei-templates` v10.4.9 (2026-09-16), plus the eight-release `nuclei-templates`
cadence quoted in §7.1. These are the refs the field names were read at; they are **not** a
deployment pin, which §11 of this record leaves to Phase 2 and §21.3.

These are source-level probes of upstream output contracts, not executions, which is the right form of
evidence for a record that asks *what fields exist and what they mean*. Every claim whose truth
depends on VulcanFlow's own code running instead carries a test id and an owner in §9.

---

## 14. Amendment history

None. The amendment filed alongside this record is **ADR-0003 A1**, which corrects ADR-0003 §3.2 and
§3.5 as described in §4.1 and §8.4.5.

Revisions made **before acceptance**, during lane 6 review, are §15 — not amendments. An amendment
records a change to an accepted record; these are the review's effect on the record being accepted,
and the distinction matters because §5.4 and §6 both turn on what "stored" means.

---

## 15. Pre-acceptance revisions (lane 6, 2026-10-02)

Five review rounds, all routed to the author: the automated reviewer's first pass on `4ec50a2` (one
finding); the hand review of `4ec50a2` (twelve blocking, six advisory — **all six advisories taken**);
the automated reviewer's second pass on `fe570de` (five findings, four of which the R1–R12 work had
already closed and one new, §4.4a); the hand **re-review** of `41264fd` (six blocking, five advisory —
all eleven taken); and the automated reviewer's **third** pass on `41264fd`, which landed while the
re-review fixes were being written (three findings — two substantive, one advisory — all taken, and
neither substantive one a re-raise of the two declined suggestions). No decision in §2 is reversed by
any of them. What changed:

| # | Section(s) | Change |
|---|---|---|
| Bot-1 | §2.1, §3.1, **§3.1.1** (new), §3.6 | The acceptance statement required every component byte-equal *and* admitted an aliased `check_id`. Unsatisfiable. The comparison rule is now stated once and normatively: eleven components byte-equal, `check_id` aliasable, and the alias resolved as a bounded set of equality probes with a stated precedence rule. |
| R1 | **§4.5a** (new), §4.5 table, §10, **§8.4.7** (new) | `matched_at` can name a different host than `attributes.hostname` — upstream's own fixture contains the case. Divergence now makes `port`/`location` `Unknown` (⇒ 422, no match), and the nuclei `ScanType` must disable redirect following. §8.3.1's standard now applies to both scanners instead of one. |
| R2 | **§4.5b** (new), §3.3 | `attributes.hostname` is not always a hostname: a trailing `:port` is split off before canonicalization, `null` ⇒ `Unknown`, and §3.3 now says a rejected host yields `Unknown`. Without this, no nuclei `tcp`/`network` finding could ever be marked a false positive. **The split rule is superseded by B5** — it was wrong for IPv6 — but the finding and its conclusion stand. |
| R3 | **§4.5c** (new), §3.3 | The per-`type` port and location dimensions are named instead of inferred: `dns` ⇒ `NotApplicable` for both; the scheme-default table is closed to `http`/`https`; an unspecified `type` ⇒ `Unknown`. |
| R4 | §3.3 | The set digest's length prefix is **per element**, written out; duplicates are kept, with the reason. |
| R5 | §3.3, §3.1, §10 | `discriminator` is an ordered sequence with a specified tag-and-length serialization, so its four nuclei sub-value combinations are four distinct byte strings. The reviewer's preferred split-into-two-components remedy is in §10 with why it lost. |
| R6 | **§4.3.1** (new) | `soa` is a struct of eight fields, not a string. The digest input is `(name, ns, mailbox)`; the five timers and counters are excluded, `serial` because it moves on every zone edit. |
| R7 | §1, §6, §3.6, §2 row 8 | "Requires no schema change" was broader than the evidence. Two migrations were named here: `false_positive_events.reason text`, and §3.6's generated column or expression index. **Superseded by B3**, which found a third. |
| R8 | §4.3, §4.4 | httpx `location` (the raw `Location` header) and `sni`, and dnsx `all`, are named exclusions with their reasons; both lists now close with *any field not named above is excluded by construction*. We write those two parsers, so those tables are the only spec their authors have. |
| R9 | §4.4 | `failed`/`error` are preconditions, not key components — the `matcher_status` rule, for httpx. |
| R10 | §8.4.4, `README.md` | Two factual fixes: 21 of 22 fixture findings carry `ip`, not all 22; and the severity correction is pointed at ADR-0003 §3.5 and the absence of any TDD rule, not at §10.1/§16.3, which do not make the claim. |
| R11 | §3.3 | The query-order omission keeps its conclusion and loses its false premise: it is justified by direction, not by scanner determinism, which does not hold for nuclei. |
| R12 | §12 | The two unobservable revisit triggers are now measurable: a decision's applied-observation count exceeding N or growing after its creating scan, and the rate of `new` observations differing from a suppressed one only in the `extracted_results` sub-value. **Both limbs are since corrected** — the unit by Bot-3.2 (distinct keys, not observation rows) and the second limb's uncomputability by B4. The finding stands; its first expression did not. |
| A1–A4, A6 | §5.8 (new), §3.5, §3.3, §4.5, §7.2 | Aliasing has no Phase 1 use case and says so; `scope_root`'s over-narrowing cost is named; NFKC-vs-NFC is explained; `attributes.path` is excluded explicitly; the current `field_semantics_version` is committed config beside the alias registry. |
| A5 | **§8.4.1b** (new) | A third template pack is in the same upstream tree (17 matchers), which is why §7.2's gate is about declared semantics and not release numbers. |
| Bot-2.1 | **§4.4a** (new), §4.4, §12 | New in the second automated pass, and correct: excluding `sni` is only safe while nothing passes `--sni-name`, since a custom SNI selects the TLS virtual host independently of the URL. The httpx `ScanType` is now forbidden from setting it, as a catalog property `supply-chain/check-catalog` asserts; `sni` entering the key later is a `match_version` bump, and §12 gains the trigger. Adding the component now was rejected: under our own configuration it is a constant, and it would hide the fact that the control is the configuration. |
| Bot-2.2–2.5 | — | Already closed by the R1–R12 work before the pass ran: the schema claim (R7), fail-closed handling of divergent nuclei URLs (R1/§4.5a), the `soa` serialization (R6/§4.3.1 — with the eight-field variant rejected in §4.3.1, argued rather than assumed), and the README severity attribution (R10). |
| B1 | `README.md` | ADR-0007's index row was **deleted** rather than joined by this branch's merge of `main`, and the prose below it still cited ADR-0007. Merging would have regressed `main`. Row restored, ADR-0006 added alongside it. Caused by the merge resolution in `f3dd027`, not by this record's content — which is why a doc-only diff still needs the merge-base read that found it. |
| B2 | `README.md` | The retained *"ADR-0004 and ADR-0006 are not on `main` yet"* paragraph contradicted the table above it and was false the moment this merges. Narrowed to ADR-0004, with one sentence saying why ADR-0006 is in the table. |
| B3 | §1, §2 row 8, §3.6, §6 | A **third** migration was hiding behind R7's narrowing. §6 makes the suppression-reach count mandatory **before** a user confirms and §12 re-evaluates it per scan, both over `findings.applied_fp_decision_id`, which TDD §6.3 declares with no index — a mandatory interactive sequential scan, which is §3.6's own prohibition in the other direction. The index is now named, and §6 states the query so the DDL reads as a consequence of the decision rather than as Phase 1 trivia. |
| B4 | §3.1 | R12's second trigger was **not computable** as specified: it measures `new` observations, which are exactly the ones that matched nothing, and nothing said `false_positive_match` is written on those. §3.1 now states the storage rule normatively — written on every storable key regardless of outcome, `applied_fp_decision_id` carrying the outcome — and §12 cites it. Same defect class R12 was filed for, which is the point: a trigger whose input is unrecorded is not a trigger. |
| B5 | **§4.5b** (rewritten), §4.5a, §9 | R2's `host:port` split rule was **wrong for IPv6**, and in the unsafe direction. `[2001:db8::1]` (what `URL.hostname` actually returns — verified) has no trailing `:<port>` and dies in §3.3 ⇒ no IPv6 nuclei finding could be marked a false positive; bare `2001:db8::1` *matched* the split rule and produced host `2001:db8:` port `1` — a **parsed-but-wrong key**, the only place in the reviewed text where the failure direction was not §1's. Replaced by a six-branch shape table tried in order, IPv6 tested before `host:port` on colon count so the shapes are disjoint, every failure resolving to `Unknown`. |
| B6 | §8 preamble, §2 row 7, **§9.1** (new), §8.4.7 | The corpus count was wrong for the **third** time ("ten", then "eleven", against twelve-or-thirteen depending on the criterion). The fix is not a fourth count: §9.1 now **enumerates** the corpus by case identifier — 13 negative rows, 4 positive controls, and 4 §8 items explicitly assigned to `findings/fp-scanner-semantics-stale` or to §7's evidence instead. §8 keeps the property (at least two negatives per scanner), which an edit cannot falsify, and drops the number, which every edit could. |
| Bot-3.1 | §3.3 `canonical_host` (rewritten), the normalization note, §9 | **The NFKC pre-pass was a key-collapse primitive, and this is the record's own subject matter.** The rule read *"NFKC, then IDNA 2008 ToASCII under UTS-46"*. UTS-46 normalizes to **NFC**, not NFKC (revision 31 §4 Processing: Map → Normalize to NFC → Break at U+002E → Convert/Validate), and its Map step already lowercases and already maps the dot-like characters that should become separators. The pre-pass was not merely redundant: `IdnaMappingTable.txt` at Unicode 15.1.0 has `2024..2026 ; disallowed` and `FE52 ; disallowed`, while `NFKC(U+2024) = "."` and `NFKC(U+2025) = ".."` — so a host written with U+2024 canonicalized to a **different valid host** instead of being rejected, which is two inputs collapsing to one key, chosen by whoever supplies the name. Replaced by UTS-46 Processing on the original input with `UseSTD3ASCIIRules = true`. Propagates to `authz/psl-exact-root` and `authz/configured-scope`, which own this shared canonicalizer's tests. |
| Bot-3.2 | §6, §12, §1, §2 row 8 | **The reach count had the wrong unit, and it made §12's growth limb fire on everything.** `findings` holds one row per observation per scan, so a row count rises on every scan for a decision whose reach never changed: "grows in any scan after the creating scan" would have been true of every decision, always. The unit is now the **distinct match key** — reach is `COUNT(DISTINCT fp_match_key)` — which turns both limbs into statements about reach rather than about scan frequency, and turns B3's index into a covering one. Same counting-unit class as the §2.1 contradiction found in pass 1. |
| Bot-3.3 | §4.3 `discriminator` row | Advisory, taken: `txt` and `caa` values are "verbatim" in the sense of no case folding and no rewriting, but they are still NFC-normalized by §3.3's set-element rule like every other element. Said explicitly, because "verbatim" next to a normalization rule invites the reading that one bypasses the other. |
| A7–A11 | §4.5a, §10, §12, §3.2, §15 | The five advisories, all taken: §4.5c is evaluated **before** §4.5a and `NotApplicable` is never promoted to `Unknown` (so `type: dns` stays storable under a divergent `matched_at`); §10's rejection reason for the `NotApplicable` alternative was overbroad — `check_id` already separates templates, so the collapse is *within* a check class, and the corrected reason is stated with the conclusion unchanged; §12's trigger names the pure `vf-core` decoder it needs and the `(match_version, field_semantics_version, scanner_id)` scope it is only meaningful in; TDD §3.5 is qualified where it collided with this record's §3.5; and this ledger's own round-2 accounting is corrected below. |

**This ledger's own accounting, corrected (A11).** Round 2's header previously read *"five of the six
taken"* and the closing note named **R5** as the untaken advisory. Both were wrong in the same way.
R5 was a **blocking** finding, not an advisory, and it **is** taken — the ambiguity it identified is
fixed in §3.3's tag-and-length serialization. What was declined is the reviewer's *preferred remedy*
for it (splitting `discriminator` into two §3.1 components), which the reviewer then accepted as
explicitly non-blocking. All six of round 2's advisories A1–A6 were taken, which is what the rows
above show. Two reviewer-suggested remedies are declined on the record rather than in a comment, so a
re-raise argues with the record: the R5 split (§10) and serializing all eight `soa` fields (§4.3.1).
The reviewer's reading of `hostname-without-port.jsonl` is also narrowed in §4.5b — that fixture is a
**no-scheme** case (`host: example.com`), not a port-in-host case — and the reviewer confirmed the
narrowing on re-review.

**And a declined remedy still has to rest on true reasons.** The re-review's recorded non-blocking
disagreement carried two notes on §10's R5 row that are not findings against the decision but are
corrections to its *stated reasoning*, and the first round of re-review fixes missed both. They are
applied now: §10's third leg — *"`dnsx`'s `SRV` discriminator is itself multi-valued (§4.3.1)"* — is
**withdrawn as false** (per §4.3 the dnsx discriminator is one sub-value for every record type, `SRV`
included; §4.3.1 is about `soa`), and the one advantage the split had and the row never weighed — that
§12's trigger 2 would not need the §12 decoder under two components — is now stated as a cost of the
decision taken. The decision is unchanged. The reason it is justified by is. §10's rows get cited as
precedent, which is exactly why a row may not keep a leg that does not hold: **design of record**
means the written reason is the reason, so a false one is a defect even when the conclusion survives
it.

§25 effect of all of the above: `findings/fp-only-persistence` stays **one** test function over the
corpus **enumerated in §9.1**; `supply-chain/check-catalog` gains the §4.5a redirect setting, the
§4.4a SNI setting and the §7.2 config file as things it must assert; one adjunct id is added
(`findings/fp-nuclei-field-shapes`), colliding with no §25 name. The §25 mapping remains injective in
both directions. Three migrations now fall out of the record (§1), which is a Phase 1 scope fact for
Anvil and Forge rather than a change to any §25 identifier.

**One correction to a claim this record made about §25 in earlier rounds.** Up to and including the
re-review I said `authz/psl-exact-root` and `authz/configured-scope` were *"unchanged on purpose"*,
because §3.3 reuses the `vf-core` canonicalizer those identifiers own rather than adding a second
one. The reuse is still the right design and still the reason there is no third identifier — but
**Bot-3.1 corrects the canonicalizer's own rule**, and §3.3 is where that rule is written down. So the
two authz identifiers must now assert the corrected UTS-46 behaviour, including at least one
`disallowed` dot-like code point rejected rather than canonicalized. They are changed, not unchanged,
and saying so is the honest version: a record that reuses a component and then corrects it does not
get to claim it left the component alone.
