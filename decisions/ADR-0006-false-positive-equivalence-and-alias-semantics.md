# ADR-0006 — False-positive equivalence fields, alias semantics, revocation, and scanner-version invalidation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-10-01 |
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
  already `CHECK (action IN ('revoked'))`. **This record specifies the contents of those columns and
  requires no schema change.**

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
| 1 | What is the equivalence key? | A **ten-component versioned tuple** (§3), compared by byte equality after a single canonicalization pass. `match_version = 1`. |
| 2 | Which fields, per scanner? | Named concretely in §4, read from the v5.9.0 parser source (`subfinder`, `nuclei`) and from the upstream tools' own output structs (`dnsx`, `httpx`, for which secureCodeBox v5.9.0 ships **no** scanner — §4.1). |
| 3 | What is "absent"? | A typed third state. Components are `Present(v)` / `NotApplicable` / `Unknown`. **Never SQL `NULL`, never `""`.** An `Unknown` in any component makes the key **non-storable and non-matchable** (§3.4). |
| 4 | Alias semantics | Aliases map `check_id` **only**; **directed, non-transitive, equal-or-narrower, evidence-bearing, versioned, cross-scanner forbidden**, evaluated at match time and never written back into a stored decision (§5). |
| 5 | Revocation | Append-only; affects **future** matching only; never rewrites a past applied record or a saved report; never reactivates (§6). |
| 6 | Scanner version bump | A bump alone invalidates nothing. A **declared field-semantics change** does: per-scanner `field_semantics_version`, bumped in the scanner-pin PR, **stales** every decision stored under the prior value — retained and visible, suppressing nothing (§7). |
| 7 | Evidence | Ten worked negative examples in §8, at least two per scanner, each with the loose key that collapses the pair and the consequence of the collapse. The nuclei case ships as an **upstream fixture file** Ledger can use verbatim. |

### 2.1 Acceptance statement

> A stored false-positive decision suppresses a later observation **if and only if** every component
> of the §3 match key is byte-equal after §3.3 canonicalization, the stored and current
> `field_semantics_version` for the producing scanner agree, the decision is not revoked, and — where
> the two `check_id`s differ — exactly one directed alias edge admitted by §5 connects the stored
> `check_id` to the observation's. Every other later observation is `new`.

---

## 3. The match key

### 3.1 Shape

`match_scope` (on the decision) and `false_positive_match` (on the observation) both carry this
object. `match_version` on the decision carries the integer in the first row.

| Component | Type | Meaning |
|---|---|---|
| `match_version` | integer | **1.** The version of this specification. Bumped only by an ADR or an amendment to this one. A decision only ever matches an observation keyed at the same `match_version`. |
| `tenant_id` | uuid | Partition, stored for audit — see §3.2. |
| `scanner_id` | enum | `subfinder` \| `dnsx` \| `httpx` \| `nuclei`. Closed set; a fifth value requires an amendment adding its §4 row. |
| `field_semantics_version` | integer | The producing scanner's field-semantics generation (§7). **1** for all four at this record's refs. |
| `check_id` | string | The check class. Per-scanner source field in §4. Verbatim bytes, **no case folding**. |
| `scope_root` | string | The canonical form of the **authorized target row** (`findings.target_id` → `targets`) the observation is attributed to — *not* any scanner-supplied echo of our own input. See §3.5. |
| `canonical_host` | component | Canonical hostname per §3.3, or `NotApplicable` when the finding's subject is an IP literal. |
| `canonical_addr` | component | Normalized IP literal, `Present` **only** when `canonical_host` is `NotApplicable`. The two are mutually exclusive and at least one must be `Present`. |
| `port` | component | u16, resolved per §3.3. `NotApplicable` for checks with no port dimension. |
| `protocol` | component | Lowercase scheme or transport, per §4. `NotApplicable` where the check has no protocol dimension. |
| `location` | component | Normalized path-and-query per §3.3. `NotApplicable` where the check has no sub-host location. |
| `discriminator` | component | The per-scanner remainder named in §4 — the part that carries "which of the several findings this check can produce on one location is this one". `NotApplicable` where §4 says so. |

### 3.2 `tenant_id` is a partition *and* a stored component

Matching runs inside the tenant transaction context, so RLS (§3.5) already makes a cross-tenant
match impossible by construction. `tenant_id` is nevertheless written into `match_scope` and compared,
because a tenant-crossing suppression is the single worst outcome this feature can produce and one
mechanism is not enough for it. If the two ever disagree, that is a hard error and an isolation
incident, not a cache miss. Belt and braces, deliberately, under **blast radius**.

### 3.3 Canonicalization — exactly these rules, and no others

Every rule below is a chance for two distinct things to become one key, so the list is deliberately
short. Where a rule is omitted, the omission is stated and its consequence is named.

**`canonical_host`.** NFKC, then IDNA 2008 ToASCII under UTS-46 with `transitional_processing =
false`, then ASCII-lowercase; strip **exactly one** trailing `.`; reject empty labels, a label over
63 octets, or a name over 253 octets.

> **This must be the same canonicalizer as §5.3 scope matching, in `vf-core`.** Not a second one.
> A second host canonicalizer is a design failure: the two would drift, and the drift would be a
> suppression that applies to a host the scope matcher considers different. §25's
> `authz/configured-scope` and `authz/psl-exact-root` own that canonicalizer's tests; this record
> **reuses** it and adds none. (**Crate purity** — `vf-core` is pure, so this is shared library code,
> not a service call.)

**`canonical_addr`.** IPv4 in dotted-quad; IPv6 per RFC 5952 (lowercase hex, maximal `::`
compression, no leading zeros). Set only when the finding's subject is an IP literal rather than a
name.

**`port`.** Taken from the URL when present; otherwise defaulted **from the scheme** — `http`→80,
`https`→443. If the check has a port dimension and neither a port nor a known scheme is available,
the component is `Unknown` (§3.4). Default-port folding is required, not optional: without it
`https://example.com` and `https://example.com:443` are two keys for one service.

**`location`.** Path **and query**, fragment stripped (a fragment is never sent on the wire).
Empty path becomes `/`. Otherwise **verbatim**: no percent-decoding, no case folding, no collapsing
of repeated separators, no trailing-slash normalization.

> *Why verbatim.* Percent-decoding merges `%2F` with `/`, which are different paths to a server.
> Case folding merges `/Admin` with `/admin`, which are different resources. The query must be in
> the key because §10.2 names *parameter* as a location discriminator, and for an injection template
> `?id=1` and `?id=2` are two findings. **Omitted on purpose:** query-parameter *ordering* is not
> normalized, so `?a=1&b=2` and `?b=2&a=1` are two keys. That can only under-suppress, and the
> scanners emit parameters deterministically, so the extra rule is not worth its collision risk.

**Set-valued components** (`discriminator` inputs that are arrays — DNS record values, nuclei
`extracted_results`). Digest = SHA-256 over the concatenation of each element, NFC-normalized,
sorted by its normalized bytes, and **length-prefixed** with a `u32` big-endian octet count.

> Sorting because no scanner guarantees array order. Length-prefixing because plain concatenation is
> ambiguous: `["ab","c"]` and `["a","bc"]` both yield `abc`, and an attacker who controls one
> extracted value controls which other finding it collides with.

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

### 3.6 Matching must be an index lookup

§10.5 requires the findings path to stay interactive, and §10.4 applies matching to **every**
observation of **every** scan. The canonical serialization of §3.1 is therefore required to be a
deterministic byte string, and matching is equality on it (or on its digest) — never a `jsonb`
containment scan over the tenant's decisions. Whether that lands as a stored generated column or an
expression index is Forge's call; that it is not a per-row scan is not.

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
| `discriminator` | **Set digest (§3.3) of the record values for that type**, each lowercased for name-valued types (`cname`, `mx`, `ns`, `ptr`, `srv`, `soa.ns`) and with a single trailing `.` stripped; verbatim for `txt` and `caa`; normalized per `canonical_addr` rules for `a`/`aaaa` | Never `NotApplicable`. If the values could not be read, `Unknown` — and §3.4 applies |

**Deliberately excluded.** `ttl`, `timestamp`, `query-time` — time, not identity. `resolver` — which
resolver answered is evidence about the measurement, not about the record. `cdn`, `cdn-name`,
`cdn-type`, `asn.*` — enrichment derived from the record value, so including them would double-count
the value and make the key move when a CDN re-labels an IP range. `status_code` / `status_code_raw` —
a `NOERROR` and a later `SERVFAIL` for the same name are not the same observation, but a failed
lookup produces no record observation at all, so this never reaches the key.

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
| `discriminator` | — | `NotApplicable` at `match_version = 1` |

**Deliberately excluded.** `final_url` — attacker-influenceable and a mass-collapse primitive
(§8.3.1). `host_ip`, `a`, `aaaa`, `cname`, `asn.*`, `cdn*` — address-level, and §6.2 is explicit that
*"IP/port alone is not a universal asset identity"* (§8.3.2). `status_code`, `title`, `webserver`,
`tech`, `cpe`, `content_length`, `words`, `lines`, `favicon*`, `hash`, `jarm_hash`, `body_preview`,
`time` — response content and fingerprints, which change on any deploy; a key that moves when the
page changes would silently drop every suppression on the next release. `input` — our own echo, see
§3.5. `vhost` — a boolean about the *host*, already carried by `canonical_host`.

### 4.5 `nuclei` — template findings

Check class: `nuclei/<template-id>`, taken from the envelope `category`, which the v5.9.0 parser sets
to `finding["template-id"]` verbatim.

| Key component | Source field | When absent |
|---|---|---|
| `check_id` | `nuclei/` + `attributes.template_id` (= envelope `category`) | Never absent; required by the envelope schema |
| `canonical_host` | `attributes.hostname` — the parser's `parseHostname(finding.host)`, which strips the scheme when `finding.host` is URL-shaped | `NotApplicable` if it is an IP literal, which then goes to `canonical_addr` |
| `canonical_addr` | `attributes.hostname` when it is an IP literal. **Never `attributes.ip_addresses`** | See §8.4.4 |
| `port` | Parsed from `attributes.matched_at`, else defaulted from `attributes.type`/scheme per §3.3 | `Unknown` if the protocol has a port dimension and neither is available |
| `protocol` | `attributes.type` (`http`, `dns`, `ssl`, `tcp`, `javascript`, …), lowercased | `Unknown` if absent. See §8.4.2 |
| `location` | Path-and-query of `attributes.matched_at`, per §3.3 | `NotApplicable` when `attributes.type` has no sub-host location (`dns`, `ssl`) |
| `discriminator` | **`attributes.matcher_name` ⧺ the set digest (§3.3) of `attributes.extracted_results`.** Both are part of the finding's identity | `matcher_name` is `null` for single-matcher templates — that is a genuine `NotApplicable`, not `Unknown`. An empty `extracted_results` is likewise `NotApplicable` |

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
evidence. Envelope `location` (= `finding.host` verbatim) — §8.4.2. Envelope `severity` — §8.4.5.

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
  have reviewed it.
- **Reason recorded.** `false_positive_decisions.reason` exists for the decision; the revocation
  event carries its own actor, and a reason field for it is a UI requirement, not a schema change.

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

This hangs off the existing bump checklist in ADR-0002 §6.3 and the approved check/template catalog
in §21.3, and it is the behaviour §25's `supply-chain/check-catalog` has to cover for templates.

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
for `findings/fp-only-persistence` (§9).

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

**8.4.4 — `ip_addresses`.** The fixture's findings all carry `ip: 34.159.58.69`, which the parser
maps to `attributes.ip_addresses`. Keying on it reproduces §8.3.2 exactly: two hostnames behind one
address become one key. Excluded, per §4.5.

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
> because it needs an owner, not because it is unimportant.

**8.4.6 — the broadened matcher.** See §7.4. A template that keeps its id and its matcher names while
broadening what an existing matcher detects produces an unchanged key, and an existing decision keeps
applying to a check that now detects more. The key cannot see it; §7.2's bump gate is the only
control, which is why that gate is in this record.

---

## 9. What has to be built and tested, and by whom

**§25 identifier.** `findings/fp-only-persistence` (§10.2) is the one §25 identifier this record
unblocks. It stays **one test function**, table-driven over the §8 corpus — the §25 mapping is
injective in both directions and this record does not change that. The table asserts, per case, both
directions: the pairs in §8 must **not** match, and §8.1.3 and the stated accepted-noise cases
**must** match.

**Adjunct test ids**, outside §25, for the mechanism §25 does not name — the same pattern ADR-0003 §3.4
used for `scb/hook-invocation-contract`:

| Test id | Asserts | Author |
|---|---|---|
| `findings/fp-match-key-canonicalization` | §3.3 exactly: IDNA/case/trailing-dot, default-port folding, fragment stripping, no percent-decoding, length-prefixed set digests (including the `["ab","c"]` vs `["a","bc"]` case), and `Unknown` → 422 on create and no match at read | Scribe (unit + proptest) |
| `findings/fp-alias-non-transitive` | §5.2 and §5.7: one hop only; `A→B, B→C` does not match `A` against `C`; reverse direction does not match; cycles and duplicate `from_check_id` rejected at registry load; cross-scanner edge rejected | Scribe |
| `findings/fp-revocation-not-retroactive` | §6: revoked decision matches nothing afterwards; already-applied observations keep `applied_fp_decision_id` and their state history; no reactivation path | Ledger (integration) |
| `findings/fp-scanner-semantics-stale` | §7.2: a decision stored at `field_semantics_version = 1` suppresses nothing once the scanner is at 2, is retained and reported stale, and is not revoked | Ledger |

**Lane ownership** (ADR-0005). Spec: this record. Key builder and matcher: **Forge**, in `vf-core` —
pure, bytes in and a verdict out, no I/O, so it is property-testable and fuzzable without standing
anything up (**crate purity**). Application at ingest and the 422 path: **Anvil**, in `vf-ingest`.
Unit and property tests: **Scribe**. Integration, conformance and the §8 corpus: **Ledger**.
Execution and the PASS/FAIL/MISSING ledger: **Crucible**. Review: **Assay** and **Warren**.

**Fixtures.** The nuclei corpus is upstream's own file at a pinned ref
(`secureCodeBox/secureCodeBox` `v5.9.0`, `scanners/nuclei/parser/__testFiles__/secureCodeBox-test.jsonl`).
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
- **The product-facing severity model** that §8.4.5 shows is needed. Named, not settled; it needs an
  owner.
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
- **Any confirmed case of a decision applying to a finding a user says they never reviewed.** That is
  a P1 and reopens §3–§4 immediately, whatever else is in flight. It is the failure this record
  exists to prevent, so a single instance is evidence that the specification, not just the code, was
  wrong.
- **Sustained complaints about re-surfacing** traced to `extracted_results` being in the nuclei key
  (§4.5). The fix would be a per-template `extraction_is_evidence` allowlist behind a
  `match_version` bump — not an ad-hoc exclusion, and not a silent one.
- **`findings-schema.json` changing** at a secureCodeBox bump: the `severity` enum, the `location`
  description, or any constraint on `attributes`. Part of ADR-0002 §6.3's checklist.
- **Upstream secureCodeBox adding `dnsx` or `httpx` scanners.** That would replace our parsers and
  their field mapping, and §4.3/§4.4 would be rewritten against upstream's `attributes` shape instead
  of ours.
- **A need for cross-scanner aliases** (§5.5) or for aliases that move a location component (§5.1).
  Both are amendments with evidence, and both are the kind of request that should be refused by
  default.

---

## 13. Provenance

Every field name, enum value, default, and quoted line above was read on **2026-10-01** from
`raw.githubusercontent.com` and the GitHub contents API at the exact refs named:

| Ref | Files read |
|---|---|
| `secureCodeBox/secureCodeBox` @ **`v5.9.0`** | `parser-sdk/nodejs/findings-schema.json`; `scanners/` directory listing (which is how §4.1's missing-scanner fact was established); `scanners/subfinder/parser/parser.js`; `scanners/nuclei/parser/parser.js`; `scanners/nuclei/parser/__testFiles__/` listing; `scanners/nuclei/parser/__testFiles__/secureCodeBox-test.jsonl`; `scanners/subfinder/values.yaml` |
| `projectdiscovery/httpx` @ **`v1.12.0`** | `runner/types.go` (the `Result` struct and its JSON tags); `runner/runner.go` (`Host: parsed.Hostname()`, `HostIP: ip`) |
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
