---
title: "OAuth Domain-Authorized Issuer Discovery"
abbrev: "Domain-Authorized Issuer Discovery"
docname: draft-mcguinness-oauth-domain-authorized-issuer-discovery-latest
date: 2026-05-28
category: std
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - JWT authorization grants
  - identity assertion
  - issuer discovery
  - DNS authority

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC1035:
  RFC5234:
  RFC5891:
  RFC7519:
  RFC8126:
  RFC8414:
  RFC8552:
  RFC8553:
  RFC8615:
  RFC9493:
  RFC9728:
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  TRUST-FRAMEWORK:
    title: "OAuth Identity Assertion Trust Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-trust-framework/
    date: false

informative:
  RFC8461:
  RFC8659:
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  OIDF-FEDERATION:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
  WICG-EMAIL-VERIF:
    title: "Email Verification Protocol"
    target: https://wicg.github.io/email-verification-protocol/
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/

---

--- abstract

This document defines Domain-Authorized Issuer Discovery (DAI), a
mechanism by which the owner of a subject namespace (typically a
DNS domain) publishes a policy listing the OAuth authorization
servers it authorizes to assert identities in that namespace. The
mechanism uses the DNS-based authority-publication pattern operators
already deploy for CAA, MTA-STS, SPF, and DKIM. A Resource
Authorization Server uses the published policy to verify that an
identity assertion's issuer is authorized for the asserted subject
namespace.

The "discovery" defined by this document is verifier-side discovery
of the Subject Authority's issuer authorization policy. Client-side
discovery of which Assertion Issuer to use before an assertion exists
is a separate use case and is deferred to future work.

DAI is a profile of the OAuth Identity Assertion Issuer Trust Policy
({{TRUST-FRAMEWORK}}): it defines the Issuer Authorization Policy
wire format and the canonical `domain_authorized_issuer`
`subject_namespace_authorization` Trust Method that consumes it.
The parent trust policy specification owns the generic Trust Policy
document, Trust Method category structure, cross-category
combination rule, and Subject Authority Determination concept.

--- middle

# Introduction

OAuth deployments using identity-assertion grants ({{TRUST-FRAMEWORK}})
need to answer "is this Assertion Issuer authorized to assert about
subjects in this namespace?". An issuer authenticated by federation
membership is not, by that membership alone, entitled to assert
about subjects in any particular namespace; the namespace owner is
the only authoritative source of that delegation.

Domain-Authorized Issuer Discovery (DAI) lets the owner of a
DNS-namable subject namespace publish, in DNS, the set of Assertion
Issuers it authorizes for its namespace. The DNS record can carry
the authorized issuers inline (for simple deployments) or point at
an HTTPS-hosted JSON document containing richer policy (validity
windows, format restrictions, tenant binding, multiple ordered
issuers). Authority is established by control of the publication
channel: whoever can publish at `_oauth-issuer-policy.{domain}` IS
the Subject Authority for `{domain}`. The pattern follows CAA
{{RFC8659}}, MTA-STS {{RFC8461}}, SPF, and DKIM
({{dns-authority-patterns}}).

DAI is consumed by Resource Authorization Servers at assertion
verification time: given an identity assertion, the RAS uses the
Subject Authority's published policy to decide whether the
asserting issuer is authorized for the subject's namespace.

DAI is non-transitive by design. The Issuer Authorization Policy is
a flat list of Assertion Issuer identifiers; a Subject Authority
that wants to authorize an Assertion Issuer publishes its identifier
directly. The depth-1 bound and its security implications are
covered in {{TRUST-FRAMEWORK}} §Open-World Delegation and Bounded
Transitivity and §Transitive Authorization is Bounded.

DAI is also independent of OpenID Federation: a Subject Authority
can publish DAI records and a Resource Authorization Server can
evaluate them without any federation infrastructure.

## Relationships {#relationships}

DAI is a concrete profile of {{TRUST-FRAMEWORK}} for OAuth
identity assertions: the Subject Authority is the Authority
Holder; the Issuer Authorization Policy is the Delegation
Artifact; the DNS record (or DNS pointer plus HTTPS document) is
one publication channel; the Assertion Issuer is the Delegate.
Lookup state classification and fail-closed requirements follow
{{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed.

DAI is consumed by {{TRUST-FRAMEWORK}}, which owns the Trust Policy
document format, Trust Method machinery, Subject Authority
Determination, and OAuth grant-profile bindings. This document
defines the Issuer Authorization Policy document
({{dii-document}}), the DNS and HTTPS publication channels
({{publication-profiles}}), the lookup procedure ({{dii-lookup}}),
the verification rules ({{dii-verification}}), the
`domain_authorized_issuer` Trust Method, and an HTTPS-only
deployment variant ({{trust-method-https-authorized-issuer}}).
A bridge to the Email Verification Protocol's DNS records and a
client-side discovery use case are sketched as future extensions
in {{future-extensions}}.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

This document uses terminology from {{TRUST-FRAMEWORK}} (Resource
Authorization Server, Assertion Issuer, Subject Authority, Trust
Policy, Issuer Authorization Policy) and from {{TRUST-FRAMEWORK}}
(Authority Holder, Delegate, Delegation Artifact, Validator).
Subject Identifier formats follow {{RFC9493}}.

One term is specific to this document:

Domain-Authorized Issuer Discovery (DAI):
: The mechanism defined by this document for a Subject Authority
to publish, via DNS, the set of Assertion Issuers it authorizes
for its namespace.

# Issuer Authorization Policy Document {#dii-document}

The Issuer Authorization Policy is a delegation artifact: a Subject
Authority (Authority Holder for a subject namespace) uses it to
delegate the right to assert identities in its namespace to one or
more named Assertion Issuers. Each `authorized_issuers` entry is a
delegation; `valid_from` and `valid_until` bound the delegation in
time; `tenant` constrains the delegation to a specific tenant of a
shared issuer; `subject_identifier_formats` constrains the
delegation to specific Subject Identifier formats. The verification
procedure ({{dii-verification}}) validates whether a given identity
assertion is covered by a current delegation; absent a matching
delegation, the Assertion Issuer is not authorized for the asserted
namespace regardless of any other property of the assertion.

The Issuer Authorization Policy is a JSON object served over HTTPS
with media type `application/json`. It has the following members:

`subject_authority`
: REQUIRED. String. The Subject Authority identifier this policy
  applies to. Consumers MUST verify this value matches the Subject
  Authority computed in {{TRUST-FRAMEWORK}} §Subject Authority Determination after applying the
  comparison rules for that Subject Authority form.

`authorized_issuers`
: REQUIRED. Non-empty JSON array of authorized issuer objects. Each
object has:

  `issuer`
  : REQUIRED. String. The Assertion Issuer identifier. For OAuth
  authorization server issuer identifiers, this value MUST be an
  absolute HTTPS URL with no fragment component. The URL MAY contain a
  path component; if the path is empty it is equivalent only to an
  issuer identifier that uses the same string form according to the
  applicable profile's issuer comparison rules. Query components are
  NOT RECOMMENDED because many issuer metadata profiles do not use
  them. The value is compared to the JWT `iss` claim using the issuer
  identifier comparison rules of the applicable assertion grant
  profile. For JWT authorization grants that use OAuth authorization
  server issuer identifiers, this is case-sensitive string comparison,
  consistent with {{RFC8414}}.

  `tenant`
  : OPTIONAL. String. The issuer-side tenant identifier authorized for
  this Subject Authority, corresponding to the top-level `tenant`
  claim defined in {{ID-JAG}} §6.1. When present, the assertion's
  top-level `tenant` claim MUST be present and MUST exactly match this
  value using case-sensitive string comparison; an assertion that lacks
  the `tenant` claim or carries a different value does NOT match this
  entry. When absent, no tenant constraint applies and the Assertion
  Issuer is authorized for this Subject Authority regardless of its
  `tenant` claim. Tenant values are issuer-specific: the string has
  meaning only for the Assertion Issuer named by the same entry and
  MUST NOT be compared across issuers. Subject Authorities authorizing
  a shared-issuer multi-tenant Identity Provider (an Assertion Issuer
  whose `iss` is shared across many tenants) SHOULD set `tenant` to the
  specific tenant they intend to authorize; see {{dii-multi-tenant}}
  for the rationale and trust assumptions. To authorize multiple
  tenants of the same shared issuer, publish one `authorized_issuers`
  entry per (issuer, tenant) pair.

  `subject_identifier_formats`
  : OPTIONAL. JSON array of Subject Identifier format names
  ({{RFC9493}}) this Assertion Issuer is authorized for. If omitted,
  the Assertion Issuer is authorized for any format that resolves to
  this Subject Authority.

  `valid_from`
  : OPTIONAL. RFC 3339 date-time. The delegation MUST NOT be treated
  as valid before this time. A small clock-skew tolerance (≤5
  minutes) is consistent with JWT `nbf` conventions ({{RFC7519}}
  Section 4.1.5).

  `valid_until`
  : OPTIONAL. RFC 3339 date-time. The delegation MUST NOT be treated
  as valid at or after this time.

`last_updated`
: OPTIONAL. RFC 3339 date-time at which the policy was last published.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, using the
  representation defined in {{TRUST-FRAMEWORK}} §Signed Policy Metadata. This member
  follows the signed metadata pattern used by {{RFC8414}} and
  {{RFC9728}}.
  Its presence does not by itself require all consumers to support
  signed policy processing; consumers apply the requirements in
  {{TRUST-FRAMEWORK}} §Signed Policy Metadata and any local policy
  that requires object-level integrity.

Consumers MUST ignore unrecognized members. Validation rules; a
policy failing any is malformed:

| Member | Required type / structure | Additional rule |
|-|-|-|
| `subject_authority` | string, required | matches the computed Subject Authority |
| `authorized_issuers` | non-empty array of objects, required | each element has `issuer` |
| `issuer` | string, required | syntactically valid issuer identifier for the applicable grant profile |
| `tenant` | string, optional | non-empty |
| `subject_identifier_formats` | array of strings, optional | |
| `valid_from`, `valid_until`, `last_updated` | RFC 3339 date-time, optional | |
| `signed_policy` | string, optional | signed JWT as defined in {{TRUST-FRAMEWORK}} §Signed Policy Metadata |

For OAuth authorization server issuer identifiers, a non-HTTPS URL,
a relative URL, or a URL with a fragment component is a malformed
`issuer` value.

Example:

~~~ json
{
  "subject_authority": "example.com",
  "authorized_issuers": [
    {
      "issuer": "https://idp.example.com",
      "subject_identifier_formats": ["email"],
      "valid_until": "2027-01-01T00:00:00Z"
    },
    {
      "issuer": "https://accounts.google.com",
      "tenant": "example.com",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-01T00:00:00Z"
}
~~~

# Publication

A Subject Authority publishes the Issuer Authorization Policy
through one of the publication channels defined in
{{publication-profiles}}. DNS publication uses the TXT record at
`_oauth-issuer-policy.{A}`, following the pattern of CAA, MTA-STS,
SPF, and DKIM ({{dns-authority-patterns}}). HTTPS publication uses
the default well-known URL on the Subject Authority's own host.

## Publication Channels {#publication-profiles}

This document defines three publication channels. A Subject
Authority chooses based on operational constraints; all produce
semantically equivalent policy validated by the same lookup procedure
({{dii-lookup}}). The canonical lookup procedure consults DNS first
and uses the HTTPS well-known channel only when DNS authoritatively
indicates absence.

| Channel | DNS form | Document | Authority binding | When to use |
|-|-|-|-|-|
| 1: DNS-Inline | TXT with `issuer=` ({{dii-dns-record}}) | None (carried in TXT) | DNS control of `{A}` | Common case: authorize an issuer for a namespace with no rich policy |
| 2: Authority-Hosted HTTPS | TXT with `uri=` | HTTPS-hosted JSON under Subject Authority's operational control | DNS control of `{A}` AND TLS on the authority-operated host | Rich policy (validity windows, format restrictions, tenant binding) at a controlled origin |
| 3: Default HTTPS Well-Known | No DNS record | HTTPS-hosted JSON at `https://{A}/.well-known/oauth-issuer-policy` | TLS on `{A}` | Subject Authority has well-known HTTPS infrastructure but no DAI DNS record |

## Default HTTPS Well-Known URL

A Subject Authority MAY publish a JSON document at the default
HTTPS well-known URL on its own host ({{dii-https-url}}), either
as the sole publication (Channel 3) or alongside a DNS record.
The lookup procedure ({{dii-lookup}}) consults DNS first and uses
the HTTPS well-known URL only when the DNS response is
`negative-authoritative`. A Subject Authority with no DAI DNS
record relies on the natural `negative-authoritative` DNS response
to bring consumers to the well-known URL.

Operators are encouraged to publish the DNS record where
practical because it matches the operational model of the
prior-art mechanisms in {{dns-authority-patterns}} and because
consumer lookup behavior is DNS-first.

## DNS Record {#dii-dns-record}

The Subject Authority publishes one or more DNS TXT records at
`_oauth-issuer-policy.{A}`, where `{A}` is the Subject
Authority rendered as a DNS name. Internationalized names are converted
to A-labels per {{RFC5891}} before the prefix is prepended. DNS lookup
names are formed without a trailing dot for comparison purposes; the
wire-format root label is not part of the Subject Authority value.

The string segments of a TXT record ({{RFC1035}}) are concatenated
without separator and interpreted as UTF-8 text.

A record is *recognized* if it begins with the version directive
`v=oauth-issuer-policy1` (case-sensitive), optionally
followed by trailing whitespace and a `;` separator. Records that do
not begin with a supported version directive MUST be ignored.

The remainder of a recognized record is a sequence of `name=value`
directives separated by `;`. Parsing rules:

- Whitespace (space and horizontal tab) immediately before and after
  each directive is ignored. Whitespace inside a value is not.
- Directive names are case-insensitive ASCII. Values are
  case-sensitive and preserved as written.
- A directive splits into name and value at the first `=`. Subsequent
  `=` characters are part of the value.
- The value runs to the next `;` or end of record. The character `;`
  MUST NOT appear in a value; URLs containing `;` MUST be
  percent-encoded.
- CR, LF, and NUL MUST NOT appear in a value.
- Unrecognized directives MUST be ignored.

The parsing rules above are summarized in the following ABNF
{{RFC5234}} for implementer convenience; the prose above is
normative if any disagreement exists.

~~~ abnf
record       = version *( ";" directive ) [ ";" ]
version      = "v=oauth-issuer-policy1"
directive    = OWS name "=" value OWS
name         = 1*( ALPHA / DIGIT / "_" / "-" )
value        = *vchar-no-semi
vchar-no-semi = %x21-3A / %x3C-7E
                  ; printable ASCII minus ";" and excluding
                  ; CR, LF, NUL
OWS          = *( SP / HTAB )
~~~

The following directives are defined:

`authority=A`
: REQUIRED. The Subject Authority identifier this record claims to
  bind. Consumers MUST normalize this value to A-label form per
  {{RFC5891}}, then verify it matches the Subject Authority computed
  from the query name (which is already in A-label form) using the
  case-insensitive ASCII comparison rules in {{TRUST-FRAMEWORK}} §Subject Authority Determination.
  Records whose `authority=` value does not match MUST be ignored.
  The publisher's `authority=` value MAY be written in U-label or
  A-label form; consumers perform the conversion.

`uri=URL`
: OPTIONAL. An HTTPS URL identifying an Issuer Authorization Policy
  document. The URL MAY be on a different host than `A`. MUST use the
  `https://` scheme and MUST NOT contain a fragment component.

`issuer=ISSUER_URL`
: OPTIONAL. An Assertion Issuer identifier authorized by the Subject
Authority for `A`. For OAuth authorization server issuer identifiers,
the value MUST be an absolute HTTPS URL and MUST NOT contain a
fragment component. MAY appear multiple times within a record and
across records.

A recognized record MUST contain at least one `uri=` directive or at
least one `issuer=` directive. Recognized records containing neither
MUST be treated as malformed (see {{dii-failures}}).

A Subject Authority MUST publish records for at most one version of
this mechanism at a time. This document defines only
`oauth-issuer-policy1`. Future versions that are not
understood by a consumer are ignored by the recognition rule above.

## HTTPS Document URL {#dii-https-url}

The default Issuer Authorization Policy URL is:

~~~
https://{A}/.well-known/oauth-issuer-policy
~~~

where `{A}` is the Subject Authority value rendered as a host (A-label
form for internationalized names). Consumers fetch this URL using
HTTPS with TLS server authentication and interpret the response body
as the JSON document defined in {{dii-document}}.

A URL obtained from a DNS `uri=` directive is fetched the same way;
the host serving the URL is responsible for TLS server authentication
of itself, not of `A`.

## HTTPS Policy Document Contract {#https-policy-document-contract}

The document retrieved from either the default well-known URL or a
DNS `uri=` pointer is the Issuer Authorization Policy defined in
{{dii-document}}; consumers MUST NOT interpret any other JSON shape
as an Issuer Authorization Policy. Consumers MUST validate the
complete policy as a unit; a malformed entry makes the whole policy
malformed, not just the offending entry.

The following JSON shape illustrates the policy contract; the prose
in {{dii-document}} is normative.

~~~ json
{
  "subject_authority": "example.com",
  "authorized_issuers": [
    {
      "issuer": "https://idp.example.com",
      "tenant": "example-tenant",
      "subject_identifier_formats": ["email"],
      "valid_from": "2026-01-01T00:00:00Z",
      "valid_until": "2027-01-01T00:00:00Z"
    }
  ],
  "last_updated": "2026-05-01T00:00:00Z"
}
~~~

# Lookup Procedure {#dii-lookup}

To retrieve the Issuer Authorization Policy for a Subject Authority `A`,
a Resource Authorization Server performs the following steps. The
procedure consults DNS first and falls back to the HTTPS well-known
URL only on `negative-authoritative` outcomes.

DNS at `_oauth-issuer-policy.{A}` and HTTPS at the default well-known
URL are two publication channels of the same Subject Authority.
Consulting HTTPS after DNS returns `negative-authoritative` is NOT an
Authority Source fallback in the sense forbidden by
{{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed; the
abstract Negative state is reached only when both channels return
absence ({{dii-failures}}). The abstract no-fallback rule continues
to forbid falling through to a DIFFERENT Subject Authority's policy.

This is the canonical procedure used by the
`domain_authorized_issuer` Trust Method. The HTTPS-only lookup mode
defined in {{trust-method-https-authorized-issuer}} skips DNS
entirely and retrieves only from the HTTPS well-known URL.

1. Query the DNS TXT resource record set at
   `_oauth-issuer-policy.{A}`. Classify the response as:

   `negative-authoritative`
   : NXDOMAIN, or NOERROR with an empty answer section or with no
   recognized records after parsing.

   `indeterminate`
   : SERVFAIL, REFUSED, timeout, truncation with no successful retry,
   or any other failure that prevents a definitive negative result.

   `affirmative`
   : NOERROR with at least one recognized record after parsing.

2. If the DNS response is `affirmative`:

   a. Discard recognized records whose `authority=` directive does
      not match `A` under the Subject Authority comparison rules in
      {{TRUST-FRAMEWORK}} §Subject Authority Determination. If any recognized record is missing an
      `authority=` directive, treat the response as `malformed`. If
      all recognized records are discarded because of `authority=`
      mismatch, treat the response as `malformed`.

      Before continuing, validate the remaining recognized records
      against the directive rules in {{dii-dns-record}}. This includes
      rejecting malformed `uri=` or `issuer=` values, and records with
      neither `uri=` nor `issuer=`. Any such condition is a `malformed`
      outcome.

   b. If any remaining record contains a `uri=` directive:

      - If more than one distinct `uri=` value is present across the
        remaining records, treat as `malformed`.

      - Otherwise fetch the JSON policy from that URL per
        {{dii-https-url}}. The fetched document is the Issuer
        Authorization Policy. All `issuer=` directives across all
        records are ignored.

   c. Otherwise (no `uri=` present), construct a virtual Issuer
      Authorization Policy with `subject_authority` set to `A` and
      one entry in `authorized_issuers` for each distinct `issuer=`
      value across the remaining records, in the order first seen.
      Entries have no `subject_identifier_formats`, `valid_from`, or
      `valid_until`. The virtual policy is processed identically to
      one fetched over HTTPS, except that its cache lifetime is
      derived from DNS TTLs as described in {{dii-caching}}.

3. If the DNS response is `negative-authoritative`, fetch the policy
   from the default HTTPS well-known URL per {{dii-https-url}}.

4. If the DNS response is `indeterminate`, lookup has failed.
   Consumers MUST NOT fall back to HTTPS, since an attacker who
   suppresses DNS responses could otherwise force the consumer onto a
   path the attacker has compromised separately.

A Resource Authorization Server MAY skip the DNS query as a matter of
local policy (for example, when deployed in an environment with an
untrusted DNS resolution path), but SHOULD NOT do so in production
deployments: a Subject Authority that publishes only inline DNS records
(a common pattern given the operational simplicity of the inline form)
will be unfindable by such a Resource Authorization Server. A Resource
Authorization Server that cannot resolve DNS treats the lookup as
`indeterminate` and rejects the assertion, fail-closed.

## Failure Handling {#dii-failures}

DAI lookup outcomes map onto the Affirmative / Negative /
Indeterminate states of the Authority Delegation Pattern's
Exception-Handling and Fallback Model
({{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed). The normative
requirements (fail closed on Negative and Indeterminate; no
fallback to a different Authority Source; bounded cache reuse on
transient Indeterminate) apply unchanged; this section maps the
concrete DAI outcomes onto those states.

| State | DAI outcomes |
|-|-|
| Affirmative | A well-formed Issuer Authorization Policy was retrieved (inline DNS, DNS pointer + HTTPS fetch, or HTTPS well-known URL), its `subject_authority` matches `A`, and its structural validation succeeds. HTTPS responses, when applicable, are 200 OK with a media type compatible with `application/json`. |
| Negative | DNS `negative-authoritative` followed by HTTPS 404 or 410 at the well-known URL, or HTTPS 404 or 410 reached from a DNS `uri=` pointer. The Authority Holder authoritatively publishes no delegation. |
| Indeterminate | Any other outcome, fail-closed by default. See enumeration below. |

The Indeterminate state covers:

- **DNS-side**: SERVFAIL, REFUSED, timeout, truncation with no
  successful retry.
- **HTTPS transport**: TLS error, connection failure, redirect loop,
  non-HTTPS redirect target.
- **HTTPS response**: 5xx; 4xx other than 404 and 410 (for example
  401, 403, 405, 429, 451); 2xx other than 200; unsupported media
  type; over-size body.
- **DNS record validation**: missing or mismatched `authority=`;
  neither `uri=` nor `issuer=`; multiple distinct `uri=` values;
  malformed directive.
- **HTTPS document validation**: body that is not a syntactically
  valid Issuer Authorization Policy; `subject_authority` that does
  not match `A`.

Any outcome not explicitly mapped to Affirmative or Negative above
MUST be treated as Indeterminate.

The following deterministic conflict rules apply:

- Multiple recognized TXT records containing only `issuer=` directives
  are merged into a single virtual policy. `issuer=` values across
  records are deduplicated, with order preserved by first-seen
  ({{dii-lookup}}, step 2c).

- If any recognized record contains a `uri=` directive after
  `authority=` filtering, the HTTPS document at the single `uri=`
  target is authoritative and all `issuer=` directives across all
  records are ignored ({{dii-lookup}}, step 2b). More than one distinct
  `uri=` value is `malformed`.

- If both DNS and the HTTPS well-known URL are populated with differing
  content, DNS is authoritative when it returns an Affirmative response.
  Consumers do not consult or reconcile the HTTPS well-known URL in
  that case.

- Multiple `authorized_issuers` entries for the same `issuer` value but
  different `tenant` values are independent authorizations. Entries with
  `tenant` do not shadow entries without `tenant` for the same `issuer`.

- Only the Subject Authority computed by the extraction procedure for
  the assertion's Subject Identifier applies. Another Subject Authority's
  policy cannot grant authority over that subject.

- If the assertion's claims conflict with the matched policy entry, the
  assertion fails the Trust Method.

Consumers MUST treat both Negative and Indeterminate as
assertion rejection. A fresh cached Affirmative policy MAY be
used during transient Indeterminate on the live channel, subject
to {{dii-caching}}; the cache lifetime MUST NOT be extended by
repeated Indeterminate retrievals.

Consumers MAY follow HTTPS redirects subject to a local redirect
limit. Every redirect target MUST use the `https` scheme and MUST
NOT contain a fragment component. The final response MUST have a
successful HTTP status and a media type compatible with
`application/json`.

# Verification {#dii-verification}

When the `domain_authorized_issuer` Trust Method is evaluated, the
Resource Authorization Server MUST:

1. Determine the Subject Authority from the assertion's Subject
   Identifier per {{TRUST-FRAMEWORK}} §Subject Authority Determination. If the format is not registered
   in {{TRUST-FRAMEWORK}} §Subject Authority Extraction Procedures Registry, reject the assertion.

2. Retrieve the Issuer Authorization Policy by applying the
   procedure in {{dii-lookup}}. Negative and Indeterminate
   outcomes ({{dii-failures}}) MUST result in rejection, except
   that a fresh cached policy MAY be used when the live retrieval
   is Indeterminate.

3. Verify the policy's `subject_authority` matches the computed
   Subject Authority. (Virtual policies satisfy this by
   construction.)

4. Find an entry in `authorized_issuers` whose `issuer` matches the
   JWT `iss` claim using case-sensitive URL string equality AND, if
   the entry contains a `tenant` member, whose `tenant` value
   exactly matches the assertion's top-level `tenant` claim
   ({{ID-JAG}} §6.1) using case-sensitive string comparison. An
   entry with `tenant` does NOT match an assertion that lacks the
   `tenant` claim; an entry without `tenant` matches regardless of
   the assertion's `tenant` claim. When multiple entries share the
   same `issuer` value, candidate entries are evaluated in array
   order until one matches; entries with `tenant` are independent
   matches per (issuer, tenant) pair.

5. If the matched entry contains `subject_identifier_formats`,
   verify the Subject Identifier format of the assertion's subject
   is listed.

6. If `valid_from` or `valid_until` are present, verify the current
   time is within the validity window.

Steps 1 and 2 are prerequisites; their failure causes assertion
rejection per {{dii-failures}} and is not classified as a Trust
Method satisfaction outcome. The Trust Method is satisfied when
steps 3 through 6 all succeed. On failure, the Resource
Authorization Server MUST reject the assertion with an OAuth
`invalid_grant` error when the cross-category combination rule of
{{TRUST-FRAMEWORK}} §Resource Authorization Server Processing is not met.

# Caching {#dii-caching}

Cache lifetimes for the Issuer Authorization Policy:

- Consumers SHOULD respect HTTP `Cache-Control` on HTTPS documents
  and DNS TTL on records. The DNS inline form's virtual-policy
  lifetime is the minimum TTL of the constructing records. For
  the DNS pointer form, the effective lifetime is the lesser of
  the pointer's DNS TTL and the fetched document's HTTP cache
  lifetime.
- Subject Authorities SHOULD publish records and HTTPS policies
  with a steady-state TTL of at most 1 hour, reducing further
  during an active revocation.
- Consumers MUST enforce an absolute local cache ceiling
  (recommended: 24 hours) regardless of TTL or `Cache-Control`.
- Consumers MAY serve a fresh cached Affirmative policy when the
  live retrieval is `indeterminate`, but MUST NOT extend the
  effective cache lifetime past the ceiling: cumulative
  `indeterminate` retrieval streaks exceeding 1 hour MUST
  transition to reject. This bound prevents an attacker who can
  sustain denial of service against the policy endpoint from
  extending revocation latency indefinitely.
- Negative results MAY be cached subject to the same ceiling.
  Consumers whose threat model includes brief publication-channel
  takeover SHOULD cap negative-cache lifetime at a shorter value
  (recommended: 5 minutes) so that a Negative cached during a
  takeover does not hide the legitimate Authority Holder's later
  publication.

# Trust Methods {#trust-methods}

This document defines `domain_authorized_issuer` as the canonical
`subject_namespace_authorization` Trust Method of {{TRUST-FRAMEWORK}}.
DNS at `_oauth-issuer-policy.{authority}` is the primary publication
channel, with the HTTPS well-known URL as fallback. The Trust Method
registers in the Identity Assertion Issuer Trust Methods registry
({{TRUST-FRAMEWORK}} §Identity Assertion Issuer Trust Methods Registry).

## domain_authorized_issuer {#trust-method-domain-authorized-issuer}

The `domain_authorized_issuer` method indicates that the Assertion
Issuer is acceptable if the Subject Authority identified by the
assertion's Subject Identifier authorizes the Assertion Issuer to
assert identities in that namespace, as defined in this document.

Domain-Authorized Issuer Discovery, defined in this document, is the
mechanism by which a Subject Authority publishes its authorized
Assertion Issuers. A Subject Authority publishes a DNS TXT record at
`_oauth-issuer-policy.{authority}` (carrying authorized issuers
inline, or pointing to a richer HTTPS-hosted document) or an HTTPS
JSON document at `.well-known/oauth-issuer-policy` on the Subject
Authority's host. The full mechanism (record syntax, lookup
procedure, failure handling, and security considerations) is
specified in this document.

This Trust Method natively expresses authorization for either form of
multi-tenant Assertion Issuer:

- **Per-tenant issuer identifiers** (for example,
  `https://login.microsoftonline.com/{tenant-id}/v2.0`): the
  `authorized_issuers[].issuer` field accepts any absolute HTTPS URL
  issuer identifier, and case-sensitive comparison against the JWT
  `iss` claim distinguishes tenants under the same host. Each
  authorized tenant is one `authorized_issuers` entry.

- **Shared issuer with a tenant claim** (for example,
  `https://accounts.google.com` serving every Google Workspace tenant
  via the top-level `tenant` claim of {{ID-JAG}} §6.1): the
  `authorized_issuers[].tenant` field binds the authorization to a
  specific tenant of the shared issuer, with the security properties
  described in {{dii-multi-tenant}}.

~~~ json
{
  "method": "domain_authorized_issuer"
}
~~~

With no additional members, the Resource Authorization Server
retrieves the Issuer Authorization Policy by applying the canonical
lookup procedure in {{dii-lookup}}: a DNS query at
`_oauth-issuer-policy.{A}` with HTTPS well-known URL fallback when the
DNS response is `negative-authoritative`. The optional `lookup`
member is used only for the HTTPS-only deployment variant in
{{trust-method-https-authorized-issuer}}.

## HTTPS-Only Deployment Variant {#trust-method-https-authorized-issuer}

Some deployments cannot or will not depend on DNS-published
authority. Those deployments can use the same Issuer Authorization
Policy document format with direct HTTPS retrieval from the Subject
Authority's well-known URL. This is a deployment variant of the DAI
mechanism, not a second Trust Method registered by this document.

Under this variant, the Assertion Issuer is acceptable if the Subject
Authority identified by the assertion's Subject Identifier publishes
an HTTPS well-known Issuer Authorization Policy authorizing the
Assertion Issuer.

~~~ json
{
  "method": "domain_authorized_issuer",
  "lookup": "https_only"
}
~~~

The `lookup` member is OPTIONAL. If present, its value MUST be
`https_only`. This variant reuses the Issuer Authorization Policy
document format ({{dii-document}}), Subject Authority determination
rules ({{TRUST-FRAMEWORK}} §Subject Authority Determination), HTTPS
document URL ({{dii-https-url}}), verification rules
({{dii-verification}}), and caching rules ({{dii-caching}}), but it
does not perform the DNS lookup defined in {{dii-lookup}}.

When evaluated, the Resource Authorization Server MUST:

1. Determine the Subject Authority `A` from the assertion's Subject
   Identifier per {{TRUST-FRAMEWORK}} §Subject Authority Determination. If the format is not registered in
   {{TRUST-FRAMEWORK}} §Subject Authority Extraction Procedures Registry, reject the assertion.

2. Fetch the Issuer Authorization Policy directly from the default
   HTTPS well-known URL `https://{A}/.well-known/oauth-issuer-policy`
   per {{dii-https-url}}. The Resource Authorization Server MUST NOT
   query `_oauth-issuer-policy.{A}` as part of this Trust Method and
   MUST NOT use a DNS `uri=` pointer.

3. Classify HTTPS retrieval and document validation outcomes per
   {{dii-failures}}. Negative and Indeterminate states MUST result
   in rejection, except that a fresh cached policy MAY be used
   when the live retrieval is Indeterminate.

4. Verify the fetched policy and match the Assertion Issuer against
   `authorized_issuers` using steps 3 through 6 of
   {{dii-verification}}.

A Resource Authorization Server uses this variant when it requires
authority publication via HTTPS only and explicitly does NOT accept
DNS-published authority (inline or DNS pointer). Compared to
canonical DNS-first lookup, this mode:

- Eliminates dependence on DNS resolution integrity for the
  authority lookup. The Subject Authority is determined per
  {{TRUST-FRAMEWORK}} §Subject Authority Determination, but the authorization itself is retrieved
  only over TLS-authenticated HTTPS.

- Rejects the inline DNS form (where the policy is carried in the
  TXT record itself) and the DNS pointer form (where DNS points
  at a different HTTPS host). Both require trusting DNS for the
  authoritative selection of either issuers or policy host.

- Forces the Subject Authority to control the apex web origin
  identified by the well-known URL.

## Choosing the Lookup Mode {#combining-dai-methods}

Within `domain_authorized_issuer`, a Resource Authorization Server
uses one lookup mode:

- **Canonical DNS-first lookup** is the default. It accepts
  the DNS-first canonical lookup, which falls back to the HTTPS
  well-known URL on `negative-authoritative` DNS responses; the "no
  DNS record, only HTTPS document" Subject Authority case is
  therefore already covered without the HTTPS-only variant.

- **HTTPS-only lookup** is an explicit deployment variant. Use it
  when local policy distrusts
  DNS-published authority entirely and requires authority retrieval
  over TLS-authenticated HTTPS only. Inline DNS records and DNS
  pointer forms are rejected.

Resource Authorization Servers MUST NOT evaluate both lookup modes
as alternatives for the same assertion. Doing so creates an
availability-driven fallback: an attacker who can drive the DNS
lookup to `indeterminate` (resolver denial-of-service, BGP
disruption) could force fallthrough to HTTPS-only evaluation,
defeating the fail-closed property of the DNS-first lookup.

# Security Considerations {#dii-security}

The Issuer Authorization Policy is a security-critical document. Its
integrity determines which Assertion Issuers can assert identities in
the Subject Authority's namespace.

## Namespace Authority Bootstrap

DAI's authority binding is DNS control of the registrable domain
(or corresponding HTTPS well-known URL), the bootstrap model used
by CAA {{RFC8659}} and MTA-STS {{RFC8461}}. Pre-existing DNS
realities (typosquatting, expired-domain takeover, registrar
account compromise) apply unchanged and are not introduced by
this document. The registrable-domain default contains
subdomain-takeover impact ({{TRUST-FRAMEWORK}} §Subject Authority
Determination); identity binding beyond DNS control (legal-entity
verification) requires out-of-band mechanisms.

## Transport Integrity {#transport-integrity}

HTTPS retrieval integrity rests on TLS server authentication of
the policy host; the inline DNS form rests on DNS resolution
alone; the pointer form rests on both. Subject Authorities
concerned about TLS misissuance are encouraged to publish CAA
records {{RFC8659}} for the policy-hosting domain and to monitor
Certificate Transparency logs.

## HTTPS-Only Authority Trust Model {#https-only-trust-model}

The HTTPS-only lookup mode
({{trust-method-https-authorized-issuer}}) substitutes TLS-server
authentication for DNS-published authority. The threat surfaces
differ from `domain_authorized_issuer`:

- DNS-record attacks ({{dns-integrity-and-compromise}}) do NOT
  affect authority retrieval; the TXT record is not fetched.
- Apex-hostname resolution still uses DNS. A DNS-redirect of the
  apex combined with TLS misissuance substitutes the policy;
  CAA records and Certificate Transparency monitoring are the
  primary defenses ({{transport-integrity}}).
- The DNS pointer form's policy-host risk
  ({{third-party-policy-hosts}}) does not apply; the policy is
  served from the Subject Authority's own apex origin.
- Subject Authorities that cannot control the apex web origin
  (apex hosted on a marketing CDN with no path control) cannot
  participate via this mode.

The trade-off between the two methods is discussed in
{{rationale-https-only}}.

## DNS Integrity and Compromise {#dns-integrity-and-compromise}

An adversary who can substitute a forged DNS response (off-path
resolver spoofing, authoritative nameserver hijack, registrar
account compromise, BGP hijack, recursive cache poisoning) can
substitute the Subject Authority's policy: add an attacker-
controlled Assertion Issuer to the inline form, redirect a `uri=`
pointer, or force `indeterminate` outcomes to benefit a cached
attacker-friendly policy. The pointer form additionally depends on
TLS authentication of the pointed-at host: TLS does NOT mitigate
the DNS compromise that selected that host.

Required framework defenses:

- Consumers MUST discard records whose `authority=` does not match
  the queried Subject Authority. This is the wildcard mitigation:
  a permissive parent-zone wildcard cannot authorize subdomains
  because consumers reject records lacking the matching
  `authority=` directive.
- Consumers MUST verify TLS for any host named by `uri=`.
- Consumers MUST bound cache lifetimes ({{dii-caching}};
  recommended absolute ceiling 24 hours).
- Consumers performing DNSSEC validation SHOULD treat validation
  failure as `indeterminate`, not `negative-authoritative`, to
  prevent signature-strip downgrade to HTTPS fallback.
- Subject Authorities SHOULD sign `_oauth-issuer-policy.{A}` with
  DNSSEC, and consumers without DNSSEC SHOULD use a trustworthy
  resolver path (DoH/DoT to a vetted resolver).

Operational defenses Subject Authorities are encouraged to apply:
registrar account lock; monitoring of the record set and policy
document for unexpected changes; cross-region/cross-resolver
checks to detect localized substitution; avoiding wildcard TXT
records in zones participating in this mechanism.

## Policy Hosts {#third-party-policy-hosts}

The DNS pointer form lets the Subject Authority move rich policy
from DNS into an HTTPS document. In this version, the pointed-to
host is expected to be under the Subject Authority's operational
control. General shared-infrastructure risks (multi-tenant CDNs,
cache rules, dangling origins) are covered in {{TRUST-FRAMEWORK}}
§Shared Infrastructure and Hosted Well-Known Paths. Three
DAI-specific points:

- The pointer target is trusted fully for the policy contents and
  is appropriate only when the host is operated for, or otherwise
  controlled by, the Subject Authority.
- Both the DNS-side `authority=` directive and the JSON-side
  `subject_authority` member MUST match the queried Subject
  Authority. Each binding is published through a different control
  path; one missing check would let a compromise of either party
  claim arbitrary namespaces.
- The `uri=` pointer is a long-lived trust delegation. Subject
  Authorities SHOULD reduce DNS TTLs in advance of any planned
  change of policy host.

## Policy Conflicts and Determinism {#policy-conflicts}

The lookup procedure ({{dii-lookup}}) is deterministic across the
common conflict scenarios that arise when multiple records or
sources coexist. Determinism is a security property: two verifiers
receiving the same DNS and HTTPS responses MUST reach the same
conclusion about what (if any) policy applies. An attacker with
partial control of one publication channel cannot exploit
interpretive ambiguity at the consumer.

Common conflict scenarios and their deterministic dispositions are
specified in {{dii-failures}}.

## Mechanism Limits {#mechanism-limits}

- **Authentication.** The inline DNS form has no signing mechanism;
  its authority binding is DNS control. The HTTPS document form
  relies on DNS selection plus TLS to the selected policy host.
- **Scope.** A Resource Authorization Server MUST NOT use a
  matched Issuer Authorization Policy to establish trust for
  subjects outside that Subject Authority's namespace.
- **Inline-form features.** The inline DNS form expresses only
  "issuer X is authorized for Subject Authority A." Deployments
  needing `tenant`, `subject_identifier_formats`, `valid_from`,
  or `valid_until` MUST use the HTTPS or DNS pointer form.

## Email Local-Part Is Not Authenticated {#email-local-part}

`domain_authorized_issuer` evaluates only the email's registrable
domain. The local-part is unauthenticated by this Trust Method: it
attests that the Assertion Issuer may assert about emails in the
namespace, not that any specific local-part is correct. Resource
Authorization Servers that normalize local-parts (case folding,
plus-address stripping, alias collapsing) inherit those assumptions
from the Assertion Issuer; an attacker controlling an
issuer-accepted alias can land on a different normalized user.
Either disable local-part normalization or verify the local-part
through a mechanism outside this document. See also
{{TRUST-FRAMEWORK}} §Scope of Namespace Authorization.

## Single-Issuer Multi-Tenant Identity Providers {#dii-multi-tenant}

The shared-issuer case and its `tenant` binding are demonstrated
in the Shared Issuer Variant of the End-to-End Example. Two
security points apply:

- **Unconstrained-listing risk.** A Subject Authority that lists
  a shared issuer with no `tenant` value authorizes EVERY tenant
  of that Identity Provider, almost never the intent. Subject
  Authorities listing a shared issuer SHOULD include `tenant`.
  Resource Authorization Servers SHOULD log a warning when
  accepting under an unconstrained entry and SHOULD consider
  rejecting as a matter of local policy.

- **Tenant-isolation dependency.** The `tenant` binding is a
  wire-format expression of trust, not a cryptographic guarantee.
  The Resource Authorization Server verifies match against the
  Subject Authority's chosen tenant value, but cannot verify the
  Identity Provider's tenant-isolation implementation. A
  tenant-isolation defect (cross-tenant minting bug,
  misconfigured admin operation) defeats the wire-format check.
  Subject Authorities SHOULD prefer per-tenant issuer identifiers
  where offered and treat shared-issuer listings as long-lived
  delegations requiring due diligence on the Identity Provider's
  tenant-isolation guarantees.

## Enumeration and Query Privacy

DNS-based lookup exposes the Subject Authority being queried to the
resolver path. It does not expose the full subject identifier for
formats such as `email`, but it can reveal organizational
relationships and timing. Resource Authorization Servers SHOULD use
privacy-preserving resolver configurations where available and
SHOULD avoid performing lookup until it is needed for a concrete
verification decision.


# IANA Considerations

## Well-Known URI for Issuer Authorization Policy

Registers the following well-known URI in the IANA "Well-Known URIs"
registry {{RFC8615}}, for use by this document:

URI Suffix:
: `oauth-issuer-policy`

Change Controller:
: IETF

Specification Document:
: This document

Status:
: permanent

Related Information:
: None

## Underscored DNS Node Name

Registers the following entry in the IANA "Underscored and Globally
Scoped DNS Node Names" registry per {{RFC8552}} and {{RFC8553}}, for
use by this document:

RR Type:
: TXT

Node Name:
: `_oauth-issuer-policy`

Specification Document:
: This document, {{dii-dns-record}}


## Trust Method Registrations

This document registers the following entries in the Identity
Assertion Issuer Trust Methods registry
({{TRUST-FRAMEWORK}} §Identity Assertion Issuer Trust Methods Registry):

| Identifier | Categories | Parameters | Reference |
|-|-|-|-|
| `domain_authorized_issuer` | `subject_namespace_authorization` | `lookup` (string, OPTIONAL; value `https_only` selects the HTTPS-only lookup variant) | This document |

--- back

# Design Rationale

This appendix is non-normative.

## Following Existing DNS Authority Patterns {#dns-authority-patterns}

Domain-Authorized Issuer Discovery applies the same
authority-publication pattern that domain owners already use for
CAA {{RFC8659}}, MTA-STS {{RFC8461}}, SPF, DKIM, and the Email
Verification Protocol {{WICG-EMAIL-VERIF}}. {{TRUST-FRAMEWORK}}
§Authority Delegation Model covers the abstract pattern; this
document chooses DNS at `_oauth-issuer-policy.{domain}` as the
authoritative publication channel.

Unlike CAA (deployment-time), SPF/DKIM (spam-score signal), and
MTA-STS (inbound mail), the `_oauth-issuer-policy` record is
consumed during user sign-in. Misconfiguration or staleness
directly blocks user sign-in to dependent applications, so
Subject Authorities deploying this framework SHOULD treat records
with sign-in-dependency operational rigor (change-management
review, monitoring for unexpected changes, rapid rollback).

## Why First-Class Tenant Binding

Shared-issuer multi-tenant Identity Providers (Google Workspace,
Auth0 in some configurations, Microsoft Entra B2B in some flows)
serve many customer tenants under a single issuer URL. The
deployment reality is that these Identity Providers are common;
authorizing them without tenant binding effectively authorizes
every tenant of the Identity Provider for the namespace, which is
almost never the intent.

The `tenant` member on `authorized_issuers[]` entries binds
authorization to the specific tenant identifier the Identity
Provider populates in the top-level `tenant` claim defined in
{{ID-JAG}} §6.1. The binding makes the Subject Authority's choice
of authorized tenant observable on the wire and verifiable per
assertion. It does not eliminate the trust assumption on the
Identity Provider's tenant-isolation enforcement; it makes the
assumption explicit and auditable. See {{dii-multi-tenant}}.

A generic claim-matching mechanism (matching arbitrary JWT claims
against publisher-specified values) was considered as an
alternative. First-class `tenant` was preferred because the claim
name is standardized in {{ID-JAG}}, the deployment intent is
unambiguous, and the wire-format expression is simpler than a
generic claim-matching object.

## Choosing Between DNS-Published and HTTPS-Only Authority {#rationale-https-only}

Canonical DNS-first lookup and HTTPS-only lookup are not strictly
ordered by security strength; they trade different risks.
HTTPS-only lookup is resilient to TXT-record attacks but depends on
apex-hostname resolution plus the public CA trust system; a DNS
redirect combined with TLS misissuance defeats it.
`domain_authorized_issuer` in the inline form is resilient to TLS
misissuance against the apex because the authority artifact is the
TXT record itself; an attacker needs DNS-write or DNSSEC-bypass
capability. The DNS pointer form and HTTPS fallback inherit
TLS-misissuance risk on the pointed-at host while also depending
on DNS for selection.

A Subject Authority with strong DNSSEC and weak TLS-issuance
controls favors the inline DNS form. A Subject Authority with
strong CAA/CT monitoring and limited DNS control favors HTTPS-only.

## Why HTTPS Fallback

The framework supports an HTTPS-hosted Issuer Authorization Policy
at a well-known URL on the Subject Authority's host. This serves
two purposes:

- **Backward-compatibility runway.** Subject Authorities that
  cannot or do not yet operate the DNS record can participate via
  HTTPS publication only. The lookup procedure consults the HTTPS
  well-known URL when the DNS query returns
  `negative-authoritative` ({{dii-lookup}}).

- **Richer policy expression.** The HTTPS form supports the full
  Issuer Authorization Policy JSON schema (validity windows, tenant
  binding, format restrictions) that the inline DNS form cannot
  express.

DNS remains the entry point; HTTPS is consulted only as fallback
when the DNS query returns `negative-authoritative`. This preserves
the DNS-authority pattern as the primary channel while
accommodating deployment variations.


# Future Extensions {#future-extensions}

This appendix is non-normative. It sketches features intentionally
deferred from this document; future specifications may register them.

## Email Verification Protocol Bridge

The WICG Email Verification Protocol {{WICG-EMAIL-VERIF}} defines a
DNS TXT record at `_email-verification.{domain}` whose `iss=` values
name authorized issuers for the namespace, using bare hostnames
rather than full HTTPS issuer identifiers. A future Trust Method
(provisionally `email_verification_dns`) could let a Resource
Authorization Server honor those records without requiring the
Subject Authority to also publish an `_oauth-issuer-policy` record.

The bridge is deferred because it forces the reader to learn a
second record format and a different issuer-identifier shape
(bare-origin only, no path component), and because
{{WICG-EMAIL-VERIF}} is an external draft whose status is independent
of this document. Deployments wanting the bridge can either publish
both records (an `_oauth-issuer-policy` record satisfying
`domain_authorized_issuer` plus their existing
`_email-verification` record for other consumers) or wait for the
future Trust Method specification.

## Assertion Issuer Discovery (Client-Side)

The same DAI records that let a Resource Authorization Server
verify an assertion can also let a client discover which Assertion
Issuer is authoritative for a subject identifier's namespace
before any assertion exists: given `alice@acme.example`, a client
can query `_oauth-issuer-policy.acme.example`, retrieve the Issuer
Authorization Policy, and resolve the first authorized issuer's
authorization server metadata to find its token endpoint.

This client-side use case is deferred from the first version of
DAI because it adds a second mental model (back-channel discovery
vs. verification at token exchange), introduces privacy concerns
distinct from verification (the discovery query reveals the
queried Subject Authority to DNS resolvers and to the policy
host before any user interaction), and is not on the Resource
Authorization Server implementer's critical path. A future
specification can profile the discovery flow with appropriate
privacy guidance and integration with OAuth Authorization Server
Metadata {{RFC8414}} and OpenID Connect Discovery
{{OIDC-DISCOVERY}}.


# DNS-Based Domain-Authorized Issuer End-to-End Example

This appendix is non-normative.

This example walks through an end-to-end verification flow: the
Subject Authority publishes a DAI record, an Assertion Issuer issues
an identity assertion, and the Resource Authorization Server uses
the record to verify that the issuer is authorized for the asserted
namespace.

## Cast

- **Subject Authority**, `acme.example`. A small organization that owns
  its DNS but does not operate an authorization server capable of
  issuing identity assertions.
- **Assertion Issuer**, `https://idp.example.net`. A managed
  authorization server service `acme.example` has contracted with.
- **Resource Authorization Server**, `https://api.resource.example`.
- **Client**, a backend SaaS integration.
- **End user**, Alice (`alice@acme.example`).

## Publication

The Subject Authority publishes a single DNS TXT record:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://idp.example.net"
~~~

No HTTPS endpoint is operated on `acme.example`.

The Resource Authorization Server publishes a trust policy that accepts
domain-authorized issuer delegations with DNS-based discovery:

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods": [
    {
      "method": "domain_authorized_issuer"
    }
  ]
}
~~~

## Issuance and Token Request

1. The Client is configured, by a deployment-specific mechanism
   outside DAI, to use `https://idp.example.net` for Acme users. It
   authenticates Alice at that issuer
   and requests an ID-JAG with audience
   `https://api.resource.example` carrying Alice's email:

   ~~~ json
   {
     "iss": "https://idp.example.net",
     "aud": "https://api.resource.example",
     "exp": 1780166400,
     "iat": 1780166100,
     "jti": "5a17...",
     "sub": "user-9241ab",
     "email": "alice@acme.example",
     "email_verified": true
   }
   ~~~

2. The Client posts to the Resource Authorization Server's token
   endpoint with `private_key_jwt` client authentication:

   ~~~ http
   POST /token HTTP/1.1
   Host: api.resource.example
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion=eyJhbGciOiJSUzI1NiIs...
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=eyJhbGciOiJFUzI1NiIs...
   ~~~

## Verification (Resource Authorization Server Side)

3. The Resource Authorization Server validates the ID-JAG:
   signature (via
   `https://idp.example.net/.well-known/openid-configuration`
   JWKS), `aud`, `exp`, `iat`, replay protection.

4. The Resource Authorization Server evaluates the
   `domain_authorized_issuer` Trust Method.

   a. It extracts the Subject Authority from the top-level `email`
      claim (with `email_verified=true`): `acme.example`.

   b. Applying the canonical lookup procedure ({{dii-lookup}}), the
      Resource Authorization Server queries DNS TXT at
      `_oauth-issuer-policy.acme.example` and parses the record
      published earlier in this example.

   c. The `authority=acme.example` directive matches. No `uri=` is
      present. The Resource Authorization Server constructs a
      virtual policy for `acme.example` with one
      `authorized_issuers` entry for `https://idp.example.net`.

   d. The ID-JAG `iss` value `https://idp.example.net` matches
      the single `authorized_issuers[0].issuer`. Verification
      succeeds.

5. The Resource Authorization Server validates `private_key_jwt`,
   then issues an access token in the response body.

## Migration Variant: Pointer Form

If `acme.example` later wants to express validity windows, format
restrictions, or multiple issuers with ordering, it can switch to
the pointer form without changing any consumer behavior:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   uri=https://acme.example/.well-known/oauth-issuer-policy"
~~~

and publish the richer JSON document at the pointed-at URL:

~~~ json
{
  "subject_authority": "acme.example",
  "authorized_issuers": [
    {
      "issuer": "https://idp.example.net",
      "subject_identifier_formats": ["email"],
      "valid_until": "2027-05-30T00:00:00Z"
    },
    {
      "issuer": "https://idp-backup.example.net",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-29T00:00:00Z"
}
~~~

Resource Authorization Servers transparently follow the `uri=`
directive and consume the JSON document. No verifier software
changes are required.

## Shared Issuer Variant: Multi-Tenant Identity Provider

The simple example above uses a per-tenant issuer identifier
(`https://idp.example.net` is dedicated to Acme). Some Identity
Providers serve every tenant under a single shared issuer (for
example, `https://accounts.google.com`) and distinguish tenants via
the ID-JAG top-level `tenant` claim ({{ID-JAG}} §6.1). For these,
the `authorized_issuers[].tenant` member binds the authorization to
a specific tenant of the shared issuer.

Suppose Acme uses a multi-tenant Identity Provider with shared
issuer `https://accounts.shared.example` and Acme's tenant
identifier in that Identity Provider is `acme-corp`. Acme publishes
the pointer form pointing at the richer JSON:

~~~ json
{
  "subject_authority": "acme.example",
  "authorized_issuers": [
    {
      "issuer": "https://accounts.shared.example",
      "tenant": "acme-corp",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-29T00:00:00Z"
}
~~~

Verification adds one check to the simple flow: in addition to
matching `iss`, the Resource Authorization Server requires the
ID-JAG's top-level `tenant` claim to equal `"acme-corp"`. An
assertion from `https://accounts.shared.example` with a different
`tenant` value (or no `tenant`) does not match this entry. The
security properties and operational guidance for this case are in
{{dii-multi-tenant}}.

## Failure Variants

- A DNS SERVFAIL at `_oauth-issuer-policy.acme.example` is
  classified as `indeterminate` ({{dii-failures}}). The Resource
  Authorization Server does not fall back to
  `https://acme.example/.well-known/oauth-issuer-policy`; it
  returns `invalid_grant`.

- A wildcard record at `*.example` covering `acme.example` would
  have to carry `authority=acme.example` to be accepted. A wildcard
  with a different `authority=` is discarded.

- If `acme.example` rotates its authorized Assertion Issuer and the
  Resource Authorization Server has a cached virtual policy, the
  Resource Authorization Server may continue accepting assertions
  from the old issuer until its cache expires. Subject Authorities
  are encouraged to use short DNS TTLs during rotation; consumers
  enforce a local cache ceiling per {{dii-caching}}.

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft, split from draft-mcguinness-oauth-identity-assertion-issuer-trust-policy
