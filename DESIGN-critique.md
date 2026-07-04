# Design Critique: Authority Delegation Model + DAI

An assessment of the *solution* (not the prose): what the design gets right,
where it is genuinely weak, what the alternatives are, and how they compare.

## The problem, precisely

A Resource Authorization Server receives an identity assertion. Issuer
authentication (federation membership, key possession) proves who signed it —
not that the signer is entitled to speak for `alice@acme.example`'s namespace.
Nothing in OAuth lets `acme.example` say, in a machine-checkable way, "these
issuers may assert identities in my namespace." The design must supply a
verifiable binding: **namespace owner → authorized issuer**, consumable by a
verifier that may have no prior relationship with either party.

Any solution must answer four questions:
1. **Anchor** — what roots the namespace owner's authority? (DNS control, a
   federation operator's say-so, a government registry, a key ceremony?)
2. **Artifact** — how is the authorization expressed? (record, credential,
   registry entry, live proof?)
3. **Delivery** — how does the verifier obtain it? (fetch at verification,
   presented with the assertion, pre-provisioned?)
4. **Lifecycle** — how is it revoked, rotated, monitored?

The chosen design answers: DNS control; a declarative record (TXT or JSON
policy); verifier-side fetch at verification time; revocation by
record change bounded by cache TTLs.

---

## Critique of the chosen solution

### What it gets right

**C1. Authority stays with the authority holder.** The namespace owner
self-publishes; no intermediary can grant or withhold its ability to
authorize issuers. Every alternative that routes through a third party
(federation trust marks, trusted lists) creates an entity that can authorize
issuers for namespaces it doesn't own. This is the design's strongest
property and the right default for the open web.

**C2. Deployment burden matches the deployer.** The common case is one TXT
record — a form field in any DNS console, the same motion as SPF/DKIM/DMARC
that every IT admin already performs. No signing infrastructure, no
enrollment ceremony, no accounts with third parties. Adoption economics
dominate security-mechanism outcomes, and this is the cheapest credible
mechanism.

**C3. Revocation latency is bounded and operator-controlled.** Remove the
entry; staleness is capped by TTL + the 24h ceiling + the 1h outage window.
Credential-based alternatives push revocation into status protocols or short
lifetimes, both of which historically under-deliver (CRL/OCSP).

**C4. The two-category AND is the correct minimal algebra.** Separating "is
this issuer authentic?" from "is it authorized for this namespace?" — and
refusing to let one satisfy the other — is the actual insight of the
framework. The applicability-bypass and unverified-claim analyses defend it
well.

**C5. Fail-closed three-state lookup is unusually disciplined** for this
genre. SPF/DMARC tolerate ambiguity because spam filtering is probabilistic;
this design correctly recognizes sign-in is binary and specifies
Indeterminate ≠ Negative with no cross-channel fallback on Indeterminate.

### Where it is genuinely weak

**W1. Binary, high-stakes decisions on spam-grade infrastructure.** The
default integrity anchor is unauthenticated DNS resolution. SPF/DKIM survive
this because their consumers make probabilistic scoring decisions with a
human-recoverable failure mode (a spam folder). DAI makes a binary
authorization decision in a sign-in path. DNSSEC is SHOULD; the honest
statement (now in the spec) is that inline-form integrity equals the
resolver path. The design imports an email-era trust anchor into an
authentication decision without requiring the one mechanism (DNSSEC or
authenticated resolver) that would harden it. Mitigated but not solved by
the security-considerations text.

**W2. No monitoring / report-only deployment mode.** DMARC's adoption
succeeded substantially because of `p=none` + aggregate reports: publishers
could deploy, observe who would have failed, then enforce. DAI is
enforce-only. A Subject Authority publishing its first record has no way to
learn what would break (an unlisted departing IdP, a forgotten regional
tenant) except by breaking sign-in. This is the largest *missing feature*
relative to its own precedent family, and it directly hurts the adoption
story.

**W3. The token path acquires a DNS+HTTPS dependency.** Per-verification
lookup (cache-amortized) puts third-party infrastructure availability in the
sign-in critical path, and fail-closed converts a customer's DNS outage into
an authentication outage for that customer's users across every RAS. The
caching bounds trade revocation latency against availability, but the
structural coupling is inherent to verifier-side fetch.

**W4. PSL/eTLD+1 fuzziness in a security decision.** The registrable-domain
boundary is a mutable, non-IETF artifact (DBOUND died trying to fix this).
The spec now says so honestly and pins ICANN+PRIVATE, but two verifiers with
different snapshots can still compute different Subject Authorities. Every
DNS-boundary alternative shares this; credential and registry alternatives
do not.

**W5. Granularity ceiling.** Registrable-domain only (no subdomain
delegation — future work); depth-1 only (no chains — deliberate); no
audience scoping (a Subject Authority cannot constrain *which RASes* may
consume assertions about its users — a compromised-but-listed issuer can
assert its users to every RAS in existence). The blast-radius asymmetry is
disclosed but structural: the artifact authorizes issuer→namespace, and
nothing lets the namespace owner shape issuer→namespace→audience.

**W6. Framework weight vs mechanism size.** The deliverable mechanism is
~1 page (a TXT record, a JSON document, a comparison rule). The framework
wraps it in an abstract vocabulary (Authority Holder / Delegate / Delegation
Artifact / Validator / Authority Source), three registries, a policy
document, grant bindings, and a criticality mechanism. Each piece is
individually justified, but the ratio invites the WG critique that the
architecture is speculative generality for a single concrete profile. The
counter-argument (EVP, url_host, DIDs will reuse the machinery) is plausible
but unproven.

**W7. The Trust Policy document is the least-motivated component.** Its
operational content reduces to roughly three members. AS metadata
(`RFC 8414`) could carry them directly (the way
`dpop_signing_alg_values_supported` and similar landed) without a second
well-known document, a members registry, a signing mechanism, and a crit
mechanism. The separate document earns its keep only if policies grow rich
(trust marks, per-scope requirements) or need independent signing — both
currently hypothetical. This is the piece most likely to be merged away
under WG pressure.

---

## Alternatives

### A1. Status quo: bilateral configuration (the null alternative)
Admin configures per-customer issuer allowlists at the RAS; customer proves
domain control to the RAS out of band (email loop, DNS challenge at
onboarding).

- **For:** no new protocol; no runtime dependency; audience scoping implicit
  (each RAS's list is its own).
- **Against:** O(customers × RASes) manual configuration; no wire-format
  check that the configured issuer is the one the customer authorizes;
  drift and offboarding failures are invisible; exactly the pain the drafts
  document.
- **Verdict:** the baseline being replaced. DAI is strictly better wherever
  DNS is manageable; the null alternative survives only inside closed
  enterprise agreements.

### A2. Federation-attested namespace authority (trust marks)
The federation operator verifies domain control at issuer onboarding and
issues a trust mark ("authorized for `acme.example`") carried in the entity
configuration. Verifier checks the mark during chain validation — no DNS at
verification time.

- **For:** single verification infrastructure (the chain walk you already
  do); no second lookup; no PSL in the verifier; revocation via federation
  mechanisms; works where DNS control is awkward.
- **Against:** *centralizes namespace attestation*. The trust anchor becomes
  able to authorize any issuer for any namespace — a scope-escalation single
  point of compromise and a governance chokepoint. The namespace owner must
  enroll with (and stay current at) every relevant federation instead of
  self-publishing once. Cross-federation namespaces need marks in each.
- **Contrast with chosen design:** this is the fundamental fork —
  *self-published authority* vs *attested authority*. DAI composes with
  federation rather than competing (AND across categories); the honest
  framing is that trust marks are the right answer inside closed,
  well-governed ecosystems (healthcare, government) and DAI is the right
  answer on the open web. The framework's category model deliberately lets a
  deployment use either or both — arguably the design's best structural
  decision.

### A3. Presented delegation credential (the strongest rival)
Invert delivery: the namespace owner signs a delegation credential
("`https://idp.example.net` may assert for `acme.example` until T"), gives
it to the issuer, and the issuer presents it with the assertion (header
chain, à la `x5c`, or a separate JWT). Verifier validates the delegation
signature instead of fetching a policy. This is the OpenID
Federation-subordinate-statement / openid-authority / VC family shape.

- **For:** verification is offline once the namespace owner's key is known —
  no per-verification lookup, no DNS/HTTPS in the token path (fixes W3);
  the credential can carry audience restrictions, scopes, constraints (fixes
  W5's audience gap); granularity is arbitrary (subdomains, chains) without
  new lookup semantics; evidence travels with the assertion, so debugging
  and audit are local.
- **Against:** (1) *Key bootstrapping is the same problem recursed* — the
  verifier must discover and trust `acme.example`'s signing key, and the
  natural channels are DNS or HTTPS well-known: DAI's channel, one level
  down. The win is that keys rotate rarely and cache well vs policies, but
  the anchor is unchanged. (2) *Revocation inverts*: a signed delegation is
  valid until expiry; you either add status checking (reintroducing the
  lookup you removed) or use short-lived delegations (requiring the
  namespace owner to run automated re-signing infrastructure — a much
  heavier ask than editing a TXT record). (3) *Deployment burden lands on
  the least-equipped party*: every customer needs key management and signing
  tooling; the TXT-record motion becomes a PKI ceremony. History (DKIM vs
  S/MIME) says self-service key ceremonies kill adoption.
- **Contrast:** this is *artifact + delivery* flipped (credential/presented
  vs record/fetched) on the *same anchor* (domain control). It trades
  runtime availability coupling and audience-scoping limits for key
  management burden and revocation machinery. For high-assurance,
  low-churn, sophisticated namespace owners it is the better design; for
  the mass market it is not. A future DAI extension could adopt it as a
  second Trust Method — the framework accommodates that.

### A4. WebFinger (RFC 7033) repurposed for authorization
Query `https://acme.example/.well-known/webfinger?resource=acct:alice@acme.example`
for a link asserting the authorized issuer.

- **For:** existing RFC with deployment (Mastodon, OIDC issuer discovery);
  per-*user* granularity for free; no new DNS record type; pure HTTPS.
- **Against:** per-user query leaks the **full identifier (local-part
  included) to the Subject Authority and the network** on every
  verification — strictly worse privacy than DAI's domain-only query;
  per-user responses defeat namespace-level caching (every user is a cache
  miss); requires a running web server answering per-user queries at every
  namespace (small domains fail); anchor is TLS-only — equivalent to DAI's
  HTTPS-only mode with worse caching; no authorization semantics defined
  (validity windows, tenant binding would all need inventing on top).
- **Verdict:** dominated by DAI on privacy, caching, and deployment floor.
  Right tool for *discovery* (its actual job), wrong artifact for
  *authorization*. The drafts' positioning is correct.

### A5. DANE-style key publication (bind keys, not identifiers)
Publish the authorized issuer's key material (JWKS thumbprints) at a
DNS name (cf. TLSA/SMIMEA); the verifier validates assertion signatures
directly against DNS-published keys.

- **For:** removes issuer-identifier indirection — authorization and key
  binding become one artifact; no HTTPS fetch of issuer JWKS needed in
  principle.
- **Against:** conflates authorization with key distribution: every issuer
  key rotation becomes a customer DNS change (operationally untenable —
  issuers rotate keys unilaterally and often); DANE is meaningless without
  DNSSEC (the record IS the security artifact), so the SHOULD-DNSSEC
  weakness becomes a MUST-DNSSEC adoption cliff — and DANE's own history
  (near-zero deployment outside SMTP) is the cautionary tale; loses tenant
  binding and validity windows unless reinvented.
- **Verdict:** strictly worse lifecycle properties. DAI's identifier
  indirection (compare `iss`, let keys flow through the issuer's own
  `jwks_uri`) is the right factoring.

### A6. Central trusted list / registry (eIDAS-style)
An authority (regulator, consortium, or IANA-like registry) maintains the
(namespace → authorized issuers) list; verifiers consume the signed list.

- **For:** highest assurance per entry (identity-proofed registration);
  single artifact to cache; no per-domain deployment; auditability.
- **Against:** governance centralization (who runs it? who pays? whose law?);
  registration friction kills the long tail; the registry can censor or be
  compelled; single point of technical and political failure; global scale
  (every email domain) is implausible.
- **Verdict:** viable inside regulated verticals (the eIDAS trusted-list
  precedent is real); not viable as *the* web-scale mechanism. Could appear
  later as another Trust Method for those verticals — again, the category
  model accommodates it.

### A7. Proof-of-control at registration (ACME-style challenge)
The RAS challenges at onboarding: "have `acme.example` publish this token"
(DNS-01-shaped). One successful challenge creates a standing issuer
registration at that RAS; the record can then be removed.

- **For:** no verification-time lookup; the DNS record is transient
  (smaller standing attack surface).
- **Against:** proof is per-RAS (O(customers × RASes) ceremonies — the
  bilateral problem returns with better tooling); freshness decays from the
  moment of proof — detecting a revoked delegation requires re-challenging,
  and continuous re-challenge converges on... a standing record checked
  periodically, i.e., DAI with extra steps.
- **Verdict:** notable mainly because it shows DAI's primitive (a DNS record
  at an underscore name proving domain-authorized state) is the same one
  ACME already standardized; DAI amortizes it across all RASes and keeps it
  fresh. The steady-state record is the feature, not the bug.

### A8. Issuance-time verification (EVP-style)
Verify subject control (or namespace authorization) at *issuance*: the
issuer proves, per user or per domain, that it may assert, and embeds
evidence; or the RAS runs a live verification flow on first contact.

- **For:** evidence created when it's cheapest (at issuance, amortized over
  a session or account lifetime); aligns with draft-hardt-email-verification.
- **Against:** answers a different question — "did this user control this
  mailbox at time T?" is subject-control, not namespace authorization; the
  verifier still must trust the *issuer's* claim of having verified, which
  is precisely the trust being established; evidence staleness (delegation
  revoked after issuance) is invisible without... a verification-time check.
- **Verdict:** complementary, not competitive — a future
  `subject_control_verification` category, exactly as positioned in the
  drafts' EVP bridge. Correctly kept out of scope.

---

## Comparison

| | Anchor | Artifact / delivery | Verifier runtime dep | Revocation latency | Deployment burden (namespace owner) | Audience scoping | Governance | Precedent |
|---|---|---|---|---|---|---|---|---|
| **DAI (chosen)** | DNS control | record / fetched | DNS+HTTPS in token path (cached) | TTL-bounded (≤24h, rec. ≤1h) | one TXT record | none | self-sovereign | SPF/DKIM/DMARC/CAA |
| A1 bilateral | out-of-band proof | config / pre-provisioned | none | manual | per-RAS onboarding | implicit | bilateral | universal |
| A2 trust marks | federation operator | mark / in chain | federation fetch (already paid) | federation-speed | enroll per federation | possible in mark | centralized | OIDF, eduGAIN |
| A3 delegation credential | domain-anchored key | credential / presented | none (after key cached) | exp-bounded or status check | key mgmt + signing automation | natural | self-sovereign | x5c, OIDF subordinate stmts |
| A4 WebFinger | TLS to domain | per-user doc / fetched | HTTPS per user (poor caching) | immediate | per-user web endpoint | none | self-sovereign | RFC 7033, OIDC discovery |
| A5 DANE-style | DNSSEC (mandatory) | keys in DNS / fetched | DNS(SEC) in token path | TTL-bounded | DNS change per issuer key rotation | none | self-sovereign | DANE (cautionary) |
| A6 trusted list | registry operator | list entry / pre-fetched | list refresh | list-publication speed | registration process | possible | centralized | eIDAS/ETSI |
| A7 ACME-style | DNS control (transient) | challenge / at onboarding | none | decays; re-challenge needed | one record per RAS onboarding | implicit | bilateral | ACME DNS-01 |

The design space has two load-bearing axes:

- **Self-published vs attested** (who can speak for the namespace). DAI, A3,
  A4, A5 are self-published; A2, A6 are attested. The drafts choose
  self-published as the base and let attestation compose via the
  `issuer_authentication` category — the right layering for the open web.
- **Fetched vs presented** (where the artifact meets the verifier). DAI
  fetches; A3 presents. Fetching buys cheap publication and fast revocation
  at the cost of runtime coupling and no audience scoping; presenting buys
  offline verification and rich constraints at the cost of key ceremonies
  and revocation machinery. DAI sits on the adoption-optimal side; A3 is
  the natural "assurance tier" extension.

## Verdict and recommendations

The chosen design is the right *default*: it is the only alternative that a
mid-size company's IT admin can deploy in five minutes with tools they
already use, and the category model means the alternatives that beat it in
specific dimensions (A2 in closed ecosystems, A3 for high assurance, A6 in
regulated verticals) can arrive later as additional Trust Methods without
re-architecture. That extensibility is the framework's real payoff and the
best answer to "why the abstraction weight."

Actionable gaps the critique surfaces, in priority order:

1. **Add a monitoring mode (the DMARC `p=none` lesson).** A policy member or
   directive (e.g., `enforce=false` / `mode=monitor`) plus RAS logging
   guidance would let Subject Authorities deploy → observe → enforce. This
   is cheap, precedented, and materially improves the adoption story. (New
   feature — consider before WG adoption, while the wire format is fluid.)
2. **Decide the DNSSEC posture consciously.** Either accept resolver-grade
   integrity for the inline form as an explicit tier (current, now honestly
   documented) or require an authenticated path for enforce-mode
   deployments. Splitting assurance by mode (monitor = any DNS; enforce =
   DNSSEC-or-HTTPS) would be a defensible middle.
3. **Name the audience-scoping gap as future work** (a `permitted_audiences`
   member on `authorized_issuers[]` entries is a natural extension) so the
   blast-radius critique has a documented answer.
4. **Hold the line on Trust Policy minimalism.** If WG feedback pushes to
   fold the three operational members into AS metadata and drop the
   separate document, little of value is lost; the framework's substance is
   the category algebra and the extraction/lookup machinery, not the policy
   document format.
5. **Expect and pre-position the A3 extension.** A "signed delegation"
   Trust Method (namespace-owner key published via DNSSEC/well-known,
   delegation JWTs presented in-band) is the most likely serious follow-on;
   nothing in the current design blocks it, which is worth saying somewhere
   visible.
