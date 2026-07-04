# V1-Readiness Review — Expert Panel Synthesis

Four independent expert reviews (normative precision/interop, security/threat
model, wire-format correctness, IETF/WG process) of both drafts. Findings are
deduplicated and ranked; the parenthetical `(Nx)` is how many of the four
reviewers independently raised it (higher = higher confidence). "VERIFIED"
means checked against the source in this pass.

FW = draft-mcguinness-oauth-id-assertion-framework.md
DAI = draft-mcguinness-oauth-domain-authorized-issuer.md

---

## BLOCKERS — fix before submission

### B1. Framework still declares DAI a NORMATIVE reference (3x, VERIFIED)
`FW:52-55` — DAI is under `normative:`. Commit 5097362 scrubbed the *prose*
dependencies but not the reference classification. Effects: (a) framework, a
candidate WG doc, normatively depends on an individual draft; (b) mutual
normative dependency with DAI (DAI→FW is correct and stays); (c) contradicts
the doc's own layering claim (`FW:245-248`). idnits/shepherd flags immediately.
**Fix:** move DAI to `informative:`. All `{{DAI}}` prose uses are example/
positioning and survive as informative.

### B2. Category-applicability self-contradiction → federation-only bypass (2x, VERIFIED)
`FW:470-481` (normative): applicability "is a property of the Validator's
published Trust Policy, not of the incoming Assertion … Profiles MUST NOT
define applicability conditions that depend on properties of the Assertion."
`FW:1151-1153` (RASP step 5b): a `subject_namespace_authorization` method
"applies when the assertion carries a Subject Identifier" — an assertion
property. Step 5c only closes the "SI present but authority undeterminable"
case. **Hole:** an authentic federation member submits an assertion with *no*
email claim but another identity claim the RAS consumes for account-linking
(e.g. opaque `sub`, or generic-JWT-bearer `sub`); namespace category is deemed
inapplicable; only `issuer_authentication` runs; accepted with zero namespace
check — the exact attack the family exists to prevent (`FW:1327-1352`).
**Fix:** per grant binding, define when an assertion MUST carry a Subject
Identifier; require rejection when the policy lists a namespace method and the
RAS will consume any identity claim but no Subject Authority is determinable.

### B3. Unrecognized Trust Method objects silently dropped → fail-open downgrade (1x)
`FW:723-728, 1125-1127, 1144-1147`. Category membership is a registry attribute;
a verifier that doesn't recognize a listed method drops it *before* category
partitioning and cannot know a category was required. Policy `openid_federation`
+ `some_future_namespace_method` is evaluated by a non-implementing verifier as
issuer-auth-only — operator's namespace requirement silently waived. Two
conforming impls reach opposite verdicts. **Fix:** RAS MUST reject (treat policy
as unusable) when `issuer_trust_methods` contains any unrecognized/malformed
method object, or any object that cannot be assigned a known category.

### B4. DAI `authorized_issuers` first-match-then-verify divergence (1x, VERIFIED)
`DAI:648-667`. Step 4 defines "match" on issuer+tenant only, "evaluated in
array order until one matches"; steps 5-6 then check format/validity of "the
matched entry". Two entries, same issuer — one expired, one current (the natural
rotation pattern): a literal impl matches the expired one first and fails; a
reasonable impl searches for any satisfying entry and accepts. **Fix:** fold
steps 5-6 into the match predicate — "an entry matches when issuer, tenant,
format, and validity all hold; the method is satisfied iff ≥1 entry matches."
Drop array-order semantics.

### B5. `signed_policy` on the Issuer Authorization Policy has no key anchor (3x)
`FW:1028-1035`, `DAI:253-261`. Trust Policy key resolution is defined (AS
`jwks_uri`/federation/local). The IAP requires `iss == subject_authority` (a
bare domain like `acme.example`) but defines **no way to obtain that domain's
verification key**. If the key is fetched over the same DNS/HTTPS channel the
signature protects, a channel attacker publishes both forged policy and forged
key → signature verifies against attacker key. So `signed_policy` on the IAP is
redundant (channel intact) or worthless (channel compromised) — in exactly the
shared-CDN threat it's sold for (`FW:1474-1479`). **Fix:** define an
independent key channel (DNSSEC-published key, key pinned in the TXT record,
federation-resolved, or explicit out-of-band), or scope IAP signing to
locally-configured keys and say so plainly.

### B6. Entire DoS / amplification / SSRF threat class absent (1x)
Neither doc has a DoS section. The `domain_authorized_issuer` lookup runs on an
attacker's own validly-signed assertion (signature passes; it's the attacker's
key). **(a) Reflection/amplification + cache exhaustion:** flood of assertions
with `email:x@<random-N>.example` → RAS does a live DNS+HTTPS lookup per distinct
authority, becoming a reflector and exhausting its own cache. **(b) SSRF:** the
`uri=` pointer (`DAI:401-405`) and cross-host redirects (`DAI:624-628`) let an
attacker controlling the queried domain's DNS point the RAS HTTPS client at
`https://10.0.0.5/` or internal hosts; the fetch happens regardless of body
validation → blind internal probing via timing/error differentials. **Fix:** add
Security Considerations covering per-domain/global rate limits, bounded
concurrency, negative caching, a max distinct-authority budget, redirect/size
caps, and an SSRF address denylist (no private/loopback/link-local/ULA targets).

---

## MAJOR — reviewers will object; fix before or during WG process

### M1. The generic JWT-bearer binding (added commit 5097362) is under-specified (3x)
This binding, added two commits ago, is the most-flagged *new* problem.
- **No grant-profile identifier** → RASP step 3 unsatisfiable. `FW:1140-1141`
  requires the profile be listed in the REQUIRED `authorization_grant_profiles_supported`;
  `FW:1236-1239` says generic JWT-bearer "MAY omit an entry" (no URN exists).
  A jwt-bearer-only deployment can't populate the REQUIRED member; step 3 can't
  run. **Fix:** mint an identifier (reuse the grant-type URN) and require it, or
  make step 3 conditional and state empty-array legality.
- **`sub`-in-email-form bypasses the `email_verified` gate.** `FW:1242-1243`
  allows deriving Subject Authority from `sub` "when its value is in email form"
  with no `email_verified` requirement; the registered `email` extraction
  (`FW:936-944`) requires `email_verified=true`. `sub` is an opaque issuer-scoped
  id that may merely look like an email → silent downgrade of the verification
  precondition.
- **Multi-source "or" reintroduces the selection lever** the framework's own
  deterministic-source rule forbids (`FW:1308-1325`). "taken from `sub` … or from
  `email`" reads as a per-assertion fallback; attacker sets `sub:alice@victim`
  (email-form, unverified) and `email:attacker@attacker`.
- **"email form" is undefined** — no grammar/reference; untestable.
- **Shared multi-tenant issuers can't be tenant-scoped** under this binding
  (`DAI:222-224`): no `tenant` claim → matches only unconstrained entries →
  authorizes *every* tenant. Cross-tenant forgery unmitigated for this grant.
**Fix:** pick one fixed per-Trust-Policy source (not per-assertion fallback),
require `email_verified=true` (or equivalent) whenever `sub` is used as email,
define "email form" by reference, extend §Subject Authority Determination to
cover the `sub` source, and warn against generic JWT-bearer for shared
multi-tenant issuers.

### M2. PSL dependency breaks the determinism property and is normatively load-bearing but informative (4x — all reviewers)
`FW:950-960` (MUST-level extraction "using a current Public Suffix List"),
PSL declared informative `FW:62-64`, hazard admitted `FW:972-980`. Two verifiers
with different PSL snapshots compute different Subject Authorities for the same
email → query different names → DAI's "two verifiers MUST reach the same
conclusion" (`DAI:963-969`) is false across the snapshot boundary. This is the
DBOUND problem; DNS-directorate reviewers will raise that history. Also
unspecified: ICANN vs PRIVATE section (changes `team.github.io` vs `github.io`),
wildcard/exception rules, A-label-before-or-after-PSL order.
**Fix:** move PSL to normative refs; cite the publicsuffix.org algorithm; specify
ICANN±PRIVATE and A-label-then-match; acknowledge DBOUND and state the snapshot-
divergence consequence as accepted local-policy input (or pin snapshots among
participants). Do not leave it an unremarked MUST.

### M3. `crit` deferral is unsound — it can't be retrofitted (2x)
`FW:1767-1784` defers a criticality mechanism; v1 consumers are told to ignore
unrecognized members (`DAI:264,371`). Bootstrap flaw: a future spec defining
`crit` can't make already-deployed v1 consumers reject documents they don't
understand, so fail-closed semantics never take effect against the deployed base.
The DNS record has the `v=oauth-issuer-policy1` version escape hatch, but the two
JSON documents (Trust Policy, IAP) have neither a version member nor `crit`.
**Fix:** define `crit` (or a version member) in the base JSON documents now, even
with no initial critical members — a paragraph plus a registry entry.

### M4. `signed_policy` is trivially downgradeable + relies on undefined canonicalization (2x)
`FW:1037-1053`. No mechanism forces signature processing, so an edge/CDN attacker
(the shared-infra threat) strips `signed_policy` and serves unsigned members;
any consumer not locally pre-configured to require signatures accepts the forgery.
The SHOULD at `FW:1481-1484` needs to be MUST-require-and-verify for shared-infra.
Separately, conflict detection requires values "byte-identical after JSON
canonicalization" (`FW:1042`) but names no scheme (RFC 8785/JCS) → unimplementable
and a smuggling vector. **Fix:** MUST-require signatures where integrity matters;
cite RFC 8785 or define comparison over parsed JSON values.

### M5. DNS security posture: default is unauthenticated, "cannot be tricked" overstated (2x)
`FW:1276-1279` claims a RAS "cannot be tricked into accepting an assertion from
an AS the Subject Authority has not listed" — false for the documented default.
Affirmative needs only NOERROR + a recognized record (`DAI:499-501`); DNSSEC is
only SHOULD (`DAI:929-931`); inline "rests on DNS resolution alone" (`DAI:872`)
and is steered as the "Common case" (`DAI:321`). Spoof/poison a TXT →
`issuer=https://attacker.example` accepted. Related: forged NXDOMAIN forces the
DNS→HTTPS-well-known fallback (`DAI:536-537`), a harmful-direction downgrade to a
channel the attacker may control (dangling origin / shared-CDN tenant / TLS
misissuance). **Fix:** qualify the "cannot be tricked" claim with the channel-
integrity assumption; raise DNSSEC (or authenticated-resolver path) to MUST for
the inline form, or mark inline+no-DNSSEC as integrity-no-stronger-than-the-
resolver-path; treat fallback-triggering Negatives with Indeterminate-level
suspicion for non-DNSSEC consumers.

### M6. Cross-host HTTPS redirects break the TLS authority anchor (3x)
`DAI:624-628` permits redirects with only "https scheme, no fragment, local
limit" — no same-origin constraint. A 30x from `https://{A}/.well-known/…` to
`https://attacker.example/policy.json` is followed; TLS then authenticates the
attacker host and the fetched `subject_authority` is attacker-set. Turns a common
apex open-redirect into a policy-substitution primitive. A non-following consumer
gets an unmapped 3xx → Indeterminate → reject, while a following one accepts →
divergence. **Fix:** for the default well-known channel, redirects MUST stay
within the Subject Authority's origin/registrable domain (cross-host is the
`uri=` pointer's job, where it's disclosed); make follow-or-not deterministic
(forbid, or REQUIRE up to a fixed limit like RFC 8461).

### M7. ABNF contradicts normative prose; DNS examples don't parse (2x)
`DAI:378-386`: `vchar-no-semi = %x21-3A / %x3C-7E` excludes SP and all non-ASCII,
but prose says values are UTF-8 (`DAI:351`), interior whitespace is preserved
(`DAI:363`), and `authority=` MAY be a U-label (non-ASCII, `DAI:398`). Grammar-
derived parsers reject records the prose calls valid → for DAI a rejected record
is Indeterminate/hard-fail. Separately, every TXT example (`DAI:1254-1259,
1351-1356`; `FW:1895-1899, 2019-2024`) is a single quoted string spanning lines
with embedded LF+indent — malformed per the doc's own "no LF in a value" rule and
invalid zone syntax. **Fix:** correct the ABNF (allow %x20 + UTF-8, or restrict
`authority=` to A-label and forbid interior whitespace); render examples as
one-line or parenthesized multi-string records.

### M8. `authority=` mismatch has three inconsistent dispositions (2x)
`DAI:397` ("MUST be ignored") vs step 2a `DAI:504-511` (discard-and-continue;
all-discarded → malformed) vs failure list `DAI:579` ("missing or mismatched
`authority=`" → Indeterminate wholesale). Impls diverge on accept/reject for the
same RRset. The wildcard failure example (`DAI:1427-1429`) also says "discarded"
where the real outcome for a sole record is malformed→Indeterminate→reject with
no fallback. **Fix:** one rule (discard-and-continue; all-discarded → malformed);
fix the 579 bullet and the example.

### M9. Cache rules self-contradictory; 304 breaks caching (2x)
`DAI:689-697` presents a 24-hour absolute ceiling and a 1-hour indeterminate-
streak-to-reject bound as if the latter implements the former (different numbers);
"cumulative … streaks" is itself contradictory (cumulative never resets; streak
resets on success). And `DAI:576-587`: Affirmative requires exactly 200, so a
conditional-GET 304 is Indeterminate → an origin correctly serving 304s forces
rejection within an hour, while `DAI:680` says respect `Cache-Control`. **Fix:**
separate the two bounds explicitly (absolute entry-age ceiling; max continuous
indeterminate duration during which fresh cache may be served, reset on any
live Affirmative/Negative); make 304 refresh the cached Affirmative.

### M10. Federation-only-policy protection is only SHOULD (1x)
`FW:1175-1179`: when a policy lists only `issuer_authentication` and the assertion
carries a namespace-bound subject, the RAS "SHOULD reject." That's the family's
core failure mode (federation membership ≠ namespace authority) left advisory,
and it contradicts the absolute "cannot be tricked" at `FW:1276-1279`. **Fix:**
MUST-reject (or MUST-apply a namespace method) in that configuration.

### M11. IANA: no Designated Expert guidance; missing registries; dead extension claims (2x)
`FW:1600-1697` — three registries, all Specification Required, zero DE review
criteria (RFC 8126 §4.5). Trust Methods template lacks a Change Controller field
(inconsistent with the Trust Policy Members template). `FW:1615-1619` promises new
*categories* "via Specification Required" but categories are closed prose
(`FW:730-753`) with no registry. DAI's own extension surfaces have no registries:
TXT directive names (`DAI:371`) and IAP JSON members (`DAI:264`), both with
ignore-unknown semantics. **Fix:** add DE instructions per registry; add Change
Controller to the Trust Methods template; create a Categories registry or delete
the claim; add DAI directive + IAP-member registries.

### M12. Positioning gaps that draw adoption fire (2x)
- **WebFinger (RFC 7033) unmentioned in both docs.** It's the canonical IETF
  "given alice@acme.example, find the issuer," and it's the discovery mechanism
  in OIDC Discovery §2 (which FW references normatively). A WG reviewer asks "why
  not WebFinger?" week one. Good answers exist (per-namespace vs per-user,
  verifier-side authorization vs discovery, DNS vs HTTPS-only) — put them in.
- **draft-hardt-email-verification overlap** is addressed only in a deferred-
  bridge appendix (`DAI:1184-1204`). From altitude it's the same `_x.{domain}`
  TXT "issuer X may assert for this domain" mechanism; Hardt is active in the WG.
  Promote the differentiation into the DAI intro and state whether you seek
  convergence.
- **DMARC/BIMI/DBOUND** absent from the prior-art list (`DAI:1092-1097`); the
  record syntax is closest to DMARC's and DMARC's PSL/PSD history is on point.

### M13. Framework altitude / status tension (1x)
~200 lines of abstract "Authority Delegation Model" with `Profiles MUST…`
requirements binding future docs (`FW:383-599`) reads as an architecture/BCP doc
grafted onto a protocol doc; the generic vocabulary exceeds OAuth WG charter on
its face. Predictable WG responses: "make it Informational" (wrong — it creates
registries, metadata, a well-known URI, wire format) or "move the abstract model
to an appendix" (likely outcome). **Recommendation:** keep `std`; compress §3 to
the minimum needed for the two categories, combination rule, and three lookup
states; move the generalized pattern (RATS parallel, attribute-authority scoping)
to an appendix. Defuses both arguments.

---

## GAPS — machinery a v1 needs that is entirely absent

- **G1. Subject Identifier format-determination algorithm.** `subject_identifier_formats_supported`
  (`FW:689-696`), RASP step 4, and DAI step 5 all require knowing "the format the
  assertion uses," but nothing maps assertion claims → an RFC 9493 format name.
  Format checks are unimplementable without it.
- **G2. Media-type / `typ` registrations.** `FW:1014-1017` mandates
  `trust-policy+jwt` and `issuer-authorization-policy+jwt` with no
  `application/…+jwt` registrations; no media types for the two JSON documents.
- **G3. Well-known URI derivation rule.** `/.well-known/identity-assertion-trust-policy`
  (`FW:623-627,1580-1598`) — no origin/port rules and no RFC 8414 §3 path-insertion
  transform for issuers with path components → breaks for any path-bearing issuer.
- **G4. Duplicate-JSON-member rule.** None in either doc; combined with the
  signed/unsigned conflict rule this is a smuggling vector. One MUST-reject
  sentence needed.
- **G5. Explicit-denial publication form in DAI.** FW's Negative state
  contemplates "an explicit denial entry" (`FW:543-550`) but DAI defines none —
  a Subject Authority can't publish "no issuers, don't consult my HTTPS origin"
  (empty inline = malformed; absence = HTTPS fallback).
- **G6. Size bounds.** "over-size body" is a normative Indeterminate trigger
  (`DAI:577`) with no threshold; no caps on document size, `authorized_issuers`
  length, TXT RRset size, redirect count → non-deterministic fail-closed.
- **G7. Framework's own Trust Policy fetch contract.** DAI fully specifies its
  fetch (status/redirect/media-type/error taxonomy); the framework specifies none
  for `identity_assertion_trust_policy_uri`.
- **G8. Privacy Considerations in DAI.** The DNS-lookup doc has none (privacy
  lives in FW, which does no lookups). Per-login queries leak (customer domain,
  RAS identity, timing) to the resolver path; the HTTPS/`uri=` fetch shows the
  Subject Authority the RAS's egress IP + timing → employer-as-SA gets a
  near-real-time feed of where employees sign in. Add a section: resolver choice
  (DoH/DoT), caching as mitigation, the RAS-identity leak.
- **G9. Operational Considerations in DAI.** MTA-STS (RFC 8461 §8) set the
  expectation. DAI's operational content is scattered; consolidate TTL strategy,
  rotation runbooks, monitoring, wildcard-zone hazards, "what breaks sign-in."
- **G10. Internationalization Considerations.** Neither doc has one. Email syntax
  is simplified to "exactly one @" (`FW:948`) — breaks on quoted local-parts;
  cite RFC 5321/5322. No EAI/SMTPUTF8 (RFC 6530-6533); no NFC-before-A-label
  statement; IDN homograph residual risk unstated.
- **G11. Consolidated profile-deferral checklist.** Requirements on Trust Method
  specs are scattered (state mapping, deterministic binding function, cache
  bounds); the registry template has no fields for them. A "a Trust Method spec
  MUST define: …" section + template fields make the deferral contract testable.
- **G12. Reference/process hygiene.** ID-JAG hand-rolled instead of
  `I-D.ietf-oauth-identity-assertion-authz-grant`; hard `§6.1/§7.2` section pins
  into a moving WG draft (cite by claim name); OIDC-DISCOVERY and RFC 9728 are
  normative but used only in "follows the pattern" asides (demote); RFC 3339 used
  as a wire type but unreferenced; no `venue:` discussion note in either doc.

---

## MINOR / editorial (verified where noted)

- **Worked example** `FW:887,901` (VERIFIED): JWT carries `sub: alice@acme.example`
  but step 3 extracts "from the `email` claim" (no email claim shown). Move the
  address to an `email` claim to match the ID-JAG binding.
- **Real domain in wire example** `DAI:292` (VERIFIED): `https://accounts.google.com`
  inside a JSON example — use a `.example` name. (Prose uses at `DAI:742,1387` are
  arguably fine as real-world illustration in rationale.)
- **Stale cross-ref** `DAI:556-558`: "Authority Delegation Pattern's Exception-
  Handling and Fallback Model" — section is now "Lookup States and Fail-Closed."
- **Double citation** `DAI:158-161`: "from {{TRUST-FRAMEWORK}} (…) and from
  {{TRUST-FRAMEWORK}} (…)".
- **DAI intro says "via DNS"** (`DAI:166-169`) contradicting the HTTPS-only variant
  the same doc defines.
- **Pseudo-2119 caps** ("IS", "AND/OR", "NOT", "BOTH") scattered — idnits/Gen-ART
  flag each; lowercase + italics.
- **RFC 2119 keyword in non-normative appendix** `DAI:1106-1108`.
- **FAQ + appendix weight** (`FW:1701-2059`): FAQ in an RFC draws "fold or delete";
  two full walkthroughs + an in-body worked example is one too many. The first two
  FAQ Q/As are the strongest "why not X" text in either doc — promote to the intro.
- **Static `date: 2026-06-23`** in both — let the toolchain date the submission.
- **Untestable MUSTs** `FW:1108,1246-1247,1489-1491` — downgrade to prose/SHOULD or
  make observable.

---

## What's genuinely strong (preserve through revision)

- The authenticity-vs-delegation two-category framing and cross-category
  combination rule.
- The three-state (Affirmative/Negative/Indeterminate) lookup taxonomy with
  fail-closed rules.
- The deterministic-source-selection security analysis (`FW:1282-1325`).
- DAI's conflict-determinism table and failure enumeration (`DAI:553-628`).
- The end-to-end DAI example (`DAI:1229-1436`).
These are shepherd-quality and earn technical respect even from reviewers hostile
to the framework's altitude.

---

## Suggested sequencing

1. **Trivial + high-value now:** B1 (DAI→informative), the verified editorial
   bugs (worked example, google.com, stale xref, double citation).
2. **Correctness blockers before submission:** B2, B3, B4 (interop divergences a
   shepherd/implementer will hit immediately).
3. **Rework the JWT-bearer binding (M1)** — it's a recent addition and the
   most-flagged new problem; narrow or fully specify it.
4. **Security hardening:** B5, B6, M4, M5, M6 (+ add DoS/Privacy/Operational
   sections). These get expensive after implementations exist.
5. **Structural/positioning for adoption:** M2 (PSL/DBOUND), M3 (crit), M11
   (IANA), M12 (WebFinger/EVP), M13 (altitude) — the WG-adoption survival set.
6. **Gaps + reference hygiene:** the G-series, as the drafts mature.

**Biggest soft factor (not a doc bug):** single author, "Independent," no
Acknowledgments/Contributors, no named implementations. Recruiting 1-2 co-authors
from the ID-JAG/identity-chaining orbit (and an OpenID Federation implementer to
bless `openid_federation`) is the highest-leverage pre-adoption move.
