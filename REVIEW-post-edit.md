# Post-Edit Full Review — Consolidated Findings

Four independent reviewers (framework self-consistency, DAI self-consistency,
cross-document consistency, fresh-reader quality) over both specs at commit
d39f3b4. `(Nx)` = how many reviewers independently found it. "VERIFIED" =
quote checked against source in the synthesis pass.

Overall: the v1-readiness edits landed mostly clean — all cross-doc §-title
references resolve, registry columns match, the tenant/caching/skew/bounds
stories are consistent, and both TXT examples parse. The defects below are
almost all **seams of the surgical pass**: stale pre-edit sentences, renumbered
targets, and rules now stated in two places with different strength.

FW = draft-mcguinness-oauth-id-assertion-framework.md
DAI = draft-mcguinness-oauth-domain-authorized-issuer.md

---

## BLOCKERS

### P1. FW:1715 still says "this specification defines no criticality mechanism" (3x, VERIFIED)
§Shared Infrastructure contradicts §Critical Members (FW:1214) and
§Signed Policy Metadata (FW:1194: "lists `signed_policy` in the document's
`crit` member"), and DAI:274 relies on the same crit path. Pre-crit leftover.
The paragraph's conclusion survives on different grounds: `crit` lives in the
unsigned outer document, so an edge attacker strips `crit` together with
`signed_policy` — publisher-side criticality cannot defend against stripping;
consumer-side required-signing is still needed.
**Fix:** reword the clause to the crit-stripping rationale; keep the
consumer-MUST conclusion.

### P2. DAI:888 cites "steps 3 through 6" of a 4-step procedure (3x, VERIFIED)
The HTTPS-only variant's step 4 references the pre-edit numbering of
{{dii-verification}}, which now has steps 1-4 (old 4-6 collapsed into the
(a)-(d) predicate). Normative procedure unimplementable as written.
**Fix:** "using steps 3 and 4 of {{dii-verification}}".

### P3. FW:768-772 Trust Method Object Structure keeps filter-and-continue (1x, VERIFIED)
"If no recognized, well-formed Trust Method object remains, the policy
identifies no usable issuer trust method" — residual-set semantics, i.e. drop
unrecognized objects and proceed. RASP step 5a (FW:1320) now MUST-rejects on
any unrecognized/malformed object precisely because that filtering is a
fail-open category downgrade. Two MUSTs, opposite behavior; §trust-method-
structure was not revisited in the edit pass (its anchor is referenced
nowhere).
**Fix:** scope the residual-set semantics to clients doing capability
discovery; RAS evaluation defers to step 5a. Delete or rescope the last
sentence.

### P4. DAI admits `signed_policy` on the IAP but specifies no key-resolution mechanism (1x)
FW:1170-1173 (new): "A profile that admits `signed_policy` on the Issuer
Authorization Policy MUST specify at least one of the following key-resolution
mechanisms and state its trust assumptions: (a) DNSSEC-published key; (b)
federation/trust-anchor relationship; (c) out-of-band configuration." DAI is
exactly such a profile (member defined DAI:266, registered DAI:1326, in the
validation table) and specifies none. The only existing profile violates the
framework's new MUST; signed IAP is uninteroperable as specified.
**Fix:** add a short normative paragraph to DAI's `signed_policy` member (or
§dii-security) selecting (a) and/or (c) with trust assumptions.

---

## MAJOR — cross-corroborated

### P5. Empty `authorized_issuers`: prose says Negative, state table says Affirmative (3x, VERIFIED)
DAI:203-205 "the HTTPS-document way to publish the abstract Negative state";
DAI:613 table maps any well-formed retrieved policy to Affirmative (and an
empty array is well-formed per DAI:291). States carry different cache
semantics (stale-serving vs 5-minute negative cap), so conforming consumers
diverge. FW:567's Negative example "an explicit denial entry" also doesn't
match (it's an empty array, not an entry).
**Fix:** classify it once. Cleanest: retrieval is Affirmative; the denial is
expressed at Trust Method evaluation (no entry can match); drop the "Negative
state" sentence and fix FW:567 to "an explicit denial published through the
channel (for example, a policy authorizing no issuers)". Or map it to
Negative in the table with the 5-minute cap. Pick one.

### P6. Redirect rules cited at {{dii-lookup}} but live in {{dii-failures}} (2x)
DAI:625, DAI:1158, DAI:1160 all point readers at {{dii-lookup}} for the 5-hop /
same-registrable-domain / final-200 rules, which are actually the closing
paragraph of §Failure Handling (DAI:675-691).
**Fix:** move the redirect paragraph into {{dii-https-url}} (it is a retrieval
rule, not a failure classification) and repoint all three references.

### P7. Stale "ordered issuers" text vs order-carries-no-semantics (2x)
DAI:110 ("multiple ordered issuers"), DAI:1616 ("multiple issuers with
ordering"), DAI:1482 (client-discovery sketch: "resolve the first authorized
issuer"), and the step-2c/conflict-rule "in the order first seen" (DAI:578,
644) — all contradict or attach meaning to an order the verification predicate
now declares meaningless (and RRset order isn't stable anyway).
**Fix:** delete "ordered"/"with ordering"; drop the first-seen order language
from 2c and the conflict rule (entries are a set); note order-unspecified in
the discovery sketch.

### P8. JWT-bearer designation: "a Trust Policy MUST designate" vs "not carried in the Trust Policy" (2x)
FW:1429 vs FW:1454. The binding means "per-deployment configuration attached
to the policy," but says "Trust Policy MUST designate," then denies the policy
carries it.
**Fix:** "a deployment that admits this grant MUST designate exactly one
top-level claim ... as Resource Authorization Server configuration" and keep
the second paragraph as the explanation.

### P9. Observability logs `last_updated`, which the Trust Policy doesn't have (2x, VERIFIED)
FW:1752. No such Trust Policy member exists (only DAI's IAP has it).
**Fix:** log "the policy document's retrieval time or validator (ETag)"
instead, or the IAP's `last_updated` where DAI is in use — say which.

### P10. Orphaned scope paragraph at the end of §Critical Members (2x)
FW:1242-1251 ("The trust policy governs whether ... necessary but not
sufficient") has nothing to do with criticality and now appears in three
places (FW:466, FW:1248, FW:1629).
**Fix:** delete from §Critical Members; keep one statement (end of
§Trust Policy Processing intro is the natural home).

### P11. `crit` rejection wording ambiguous in DAI (2x)
DAI:280 "A consumer that does not recognize any listed member MUST treat the
policy as malformed" parses as "recognizes none of them." Framework rule is
reject if even ONE is unrecognized.
**Fix:** "that fails to recognize, or does not implement processing for, one
or more listed members." Also add a `crit` row to the validation table
(DAI:288-296) and align the two registries' `crit` descriptions.

### P12. `subject_authority` JSON member form unpinned (2x)
DNS `authority=` mandates A-label publication (DAI:424); the JSON member
(DAI:193) never says which form goes on the wire. U-label publishers fail
ASCII A-label comparison unpredictably.
**Fix:** "The publisher MUST write this value in A-label form per RFC 5891,"
mirroring the directive.

---

## MAJOR — single-reviewer, verified or logically sound

### P13. Negative caching: MUST in §DoS (DAI:1146) vs MAY in §Caching (DAI:778) (VERIFIED)
Also: §Caching has no rule at all for caching Indeterminate outcomes, which
the DoS section's "(subject to {{dii-caching}})" implies exists.
**Fix:** align keyword (SHOULD in DoS, or make §Caching normative MUST) and
add an Indeterminate-caching bound or drop "and Indeterminate."

### P14. 304 handling only covers a *fresh* cached policy
DAI:492 advises conditional requests whenever a cached policy is held, but the
Affirmative row (DAI:613) and Indeterminate enumeration only bless a 304 that
refreshes a *fresh* policy — revalidating a TTL-expired (within-ceiling) copy,
the main use of If-None-Match, lands in Indeterminate. Whether 304 resets the
24-hour ceiling is unstated.
**Fix:** a 304 validating any held policy within the absolute ceiling renews
freshness and is Affirmative; state it does NOT reset the 24-hour entry age.

### P15. ABNF version literal is case-insensitive under RFC 5234
DAI:413 `version = "v=oauth-issuer-policy1"` — RFC 5234 quoted strings match
case-insensitively; prose says case-sensitive. Grammar also lacks the OWS the
prose allows after the version token (DAI:380 vs 412).
**Fix:** `version = %s"v=oauth-issuer-policy1"` (RFC 7405, add reference);
`record = OWS version OWS *( ";" directive ) [ ";" OWS ]`.

### P16. `sub`-designated source: `email_verified` attests the wrong claim
FW:1440-1448: when `sub` is designated, the assertion must carry
`email_verified=true` but the RAS must IGNORE the `email` claim —
`email_verified` is defined (OIDC semantics, FW:1021) as attesting the `email`
claim's value. With `sub: alice@a.example`, `email: alice@b.example`,
`email_verified: true`, the required signal attests the ignored claim.
Related: FW:1000-1003 (§Subject Authority Determination) still says "for the
bindings in this document, a top-level `email` claim ... is the `email`
format" — stale vs the `sub` option — and no registered extraction reads
`sub`, so a `sub` designation has no registry-backed path through step 5c.
**Fix (one move):** require that when `sub` is designated, the `email` claim,
if present, MUST equal `sub` (else reject); state that the `email` extraction
procedure is applied to the designated claim's value; update the SAD sentence
to defer to {{jwt-bearer-profile}} for the generic binding. (Or drop the
`sub` option entirely — simpler and loses little.)

### P17. Fetch contract references an undefined standalone "JWT form"
FW:657-661 permits serving "the JWT form of {{signed-policy-metadata}}", but
that section only defines `signed_policy` as a member INSIDE the JSON
document; no standalone JWT representation (processing, precedence, required
members) exists. The media-type registration text hints at one.
**Fix:** delete the parenthetical, or define the standalone form properly.

### P18. Shared-infra key-resolution MUST vs Trust Policy `jwks_uri` MAY
FW:1709-1713 requires the signing key be resolvable independent of the shared
edge for BOTH document types, citing a requirement (FW:1162) that is scoped to
IAP signers only; and FW:1148 lets Trust Policy consumers resolve the key from
the RAS's `jwks_uri` — same origin, same shared edge.
**Fix:** extend the independent-channel requirement to Trust Policy signers
when {{shared-infrastructure}} applies; qualify the `jwks_uri` MAY.

### P19. `openid_federation` fails the document's own new checklist
§Requirements on Trust Method Specifications (FW:803) demands an A/N/I
lookup-state mapping and cache bounds from every Trust Method spec "whether in
this document or a companion" — the in-document `openid_federation` method
provides neither (no state mapping for federation fetch failures; cache is
"follows OIDF-FEDERATION").
**Fix:** add a short state-mapping + cache-bound paragraph to
{{trust-method-openid-federation}}.

### P20. IAP normative machinery rests on the informative DAI reference
FW normatively constrains IAP members it doesn't define (`subject_authority`,
`authorized_issuers` in the signer rules FW:1153-1160; `crit` on the IAP;
`issuer-authorization-policy+jwt` typ + media-type registration) while
declaring the wire format out of scope and DAI informative.
**Fix:** phrase these as hooks ("a profile that defines a JSON Issuer
Authorization Policy form and adopts `signed_policy` MUST bind the signer to
its authority identifier member...") with concrete member names moved to DAI;
or accept the looser layering at -00.

### P21. Framework's bare "NXDOMAIN = Negative" example vs DAI's composite channel
FW:562-565 lists DNS NXDOMAIN as a Negative example; in DAI, DNS-negative
triggers HTTPS fallback and abstract Negative needs both channels absent. A
literal reader concludes DAI violates fail-closed.
**Fix:** qualify the example ("where DNS is the profile's sole or final
publication channel").

### P22. Version token overstated as the DNS form's crit analog
FW:1238/2113 call the version token a "fail-closed path": actually an
unrecognized future version ⇒ record ignored ⇒ negative-authoritative ⇒ HTTPS
fallback (possibly an older Affirmative policy) — not rejection.
**Fix:** soften to "prevents misinterpretation of future syntax (records are
ignored, driving lookup to the HTTPS channel or Negative)".

### P23. Registrable-domain for redirect targets: PSL rules unstated
DAI:682-687 requires same-registrable-domain redirects, but the ICANN+PRIVATE
PSL algorithm is specified only inside the `email` extraction; for arbitrary
`uri=` hosts, consumers can disagree.
**Fix:** one sentence citing the framework's PSL computation for arbitrary
A-label hosts.

---

## MINOR (selected; full lists in reviewer outputs)

Readability/structure (fresh-reader):
- FW combination rule stated in 4 places with divergent anchors; make
  {{combination-rule}} the single normative home (fresh-reader #10).
- Applicability rule stated normatively in 3 places (FW:488/789/1336);
  5c is the operational home (fresh-reader #11).
- FW Intro para 1 vs para 6 repeat; "this policy" with no antecedent at
  FW:127; Open-World stub duplicated 3x (#12, #13).
- Federation-only MUST-reject buried as unnumbered hanging paragraph inside
  {{rasp}} — promote to step 5e (#14). Also "MUST X (or Y)" phrasing is
  untestable (framework m1): restate as "MUST reject unless local policy
  independently establishes authority."
- Density spikes to split into lists: FW:196 motivating-use-cases sentence,
  FW:1162 key-resolution para, FW:656 fetch-contract sentence, DAI:223 tenant
  definition, DAI:675 redirect para.
- DAI `malformed` label introduced outside step-1's three classes (#21);
  add "malformed ⇒ Indeterminate" pointer.
- DAI validation table: `tenant` "non-empty" not in prose; `crit` row missing
  (#22).
- DNS-first rule restated 3x in DAI (#23); §domain_authorized_issuer restates
  the intro (#24); "MAY skip the DNS query" paragraph conflicts with the
  https_only variant selection story (#25, DAI #15).
- "the two methods" at DAI:981 — should be "the two lookup modes" (#19).
- Terminology: "Trust Method" and "consumer" never defined in Terminology;
  "Relying Party" used undefined; capitalization drift (Delegation Artifact
  lowercase at DAI:177; `indeterminate` code-case for abstract state at
  DAI:769); state-vocabulary case convention worth an explicit sentence.
- Caps-for-emphasis in new text ("IS", "EACH", "NOT", "BEFORE") reads as
  pseudo-2119 (multiple reviewers; idnits will flag).
- Privacy Considerations placement differs (FW top-level vs DAI nested in
  Security) — align.
- FW §Signed Policy Metadata (IAP signer rules) lives under "# Trust Policy"
  heading — wrong home for DAI implementers (#36).
- Duplicate IAP JSON example in DAI (#37); DoS cites {{dii-verification}} for
  checks that live in grant-profile validation (#38); `{A}` used before
  defined (#39); ID-JAG binding step 2 wording (#40).
- Profile disambiguation: how does an RAS decide ID-JAG vs generic binding for
  one incoming jwt-bearer request (both bindings share the grant type)? One
  sentence on discrimination (assertion `typ`) needed (framework m4,
  fresh-reader #28).
- `crit` in the signed JWT payload only is ineffective for non-signed_policy
  consumers — require outer-document placement (framework m2).
- {{iana-trust-policy-members-registry}} never cited from the body (m6).
- Media-type "Encoding considerations binary" wrong for JWS Compact — use
  8bit per RFC 7519 §10.3 precedent (m7).
- FW walkthrough uses JSON-member vocabulary against a DNS-form example
  (m8); untestable client MUST in {{downgrade}} (m3).
- DAI Negative row unreachable in HTTPS-only mode — add well-known 404/410
  under that variant (DAI #8, cross-doc F11).
- Inline DNS form cannot express explicit denial; state it in Mechanism
  Limits (DAI #9).
- Entry-count bound missing from the Indeterminate enumeration (DAI #13).
- `v` registered as a directive though the grammar treats it as not-a-
  directive; state mid-record `v=` handling (DAI #12).
- Publisher-vs-consumer media-type split unstated (DAI #11).
- UTF-8 sentence now vestigial after the ASCII-only value rule (DAI #17).
- Wildcard failure-variant example implies fallback where the rule gives
  Indeterminate (DAI #16, fresh-reader corroborated).
- Step 2c absent-member list omits `tenant` (DAI #19).
- Registry `crit`/`signed_policy` rows in DAI cite "This document" though
  semantics live in TRUST-FRAMEWORK (cross-doc F6).

---

## Verified-consistent (no action)
All cross-doc §-title citations resolve (18 distinct titles checked verbatim).
Registry template columns match DAI's registration exactly. Tenant matching
semantics consistent across 5 locations. Skew valid_from-only consistent.
64 KiB/100-entry bounds consistent. Duplicate-member, duplicate-authority=,
single-uri=, discard-and-continue rules each consistent. Both TXT examples
parse under the final grammar. DAI delivers every operational defense the
framework asks profiles to recommend. Privacy/DoS split complementary, not
duplicated. framework FAQ/walkthrough DAI mechanics all accurate.

## Suggested fix order
1. P1-P4 (blockers: two stale-text, one contradiction, one missing normative
   plug) — small, surgical.
2. P5, P13, P14 (state/caching coherence) — decide classifications once.
3. P6, P7, P9, P10, P11, P12 (stranded refs and wording) — mechanical.
4. P8, P16 (JWT-bearer coherence) — decide: keep `sub` option (and repair
   email_verified semantics) or drop it.
5. P15, P17-P23 as time permits before submission.
6. Minors: batch the readability items (density splits, dedup of the
   combination/applicability rules, terminology entries) as a single
   editorial pass.
