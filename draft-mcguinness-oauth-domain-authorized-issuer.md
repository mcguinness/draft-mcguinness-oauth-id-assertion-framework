---
title: "OAuth Domain-Authorized Issuer Trust Method"
abbrev: "Domain-Authorized Issuer Trust Method"
docname: draft-mcguinness-oauth-domain-authorized-issuer-latest
date: 2026-10-09
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
  RFC3339:
  RFC5234:
  RFC5891:
  RFC7405:
  RFC7515:
  RFC7519:
  RFC7638:
  RFC8725:
  RFC8414:
  RFC8126:
  RFC8552:
  RFC8553:
  RFC8615:
  RFC9493:
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  TRUST-FRAMEWORK:
    title: "OAuth Identity Assertion Trust Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework/
    date: false

informative:
  OIDF-WKB:
    title: "OpenID Federation Well-Known Binding 1.0"
    target: https://dickhardt.github.io/well-known-binding/main.html
    date: 2026-09-29
    author:
      - name: Dick Hardt
        ins: D. Hardt
  RFC7033:
  RFC9989:
  RFC9990:
  RFC7523:
  RFC8461:
  RFC8659:
  RFC9728:
  RFC2308:
  RFC8785:
  RFC9111:
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  I-D.hardt-email-verification:
    title: "Email Verification Protocol"
    target: https://datatracker.ietf.org/doc/draft-hardt-email-verification/
    date: false
  I-D.sanz-openid-dns-discovery:
    title: "OpenID Connect DNS-based Discovery"
    target: https://datatracker.ietf.org/doc/draft-sanz-openid-dns-discovery/
    date: 2018-04-16
    author:
      - name: Vittorio Bertola
        ins: V. Bertola
      - name: Marcos Sanz
        ins: M. Sanz
  SHIBMD:
    title: "ShibMetaExt V1.0"
    target: https://shibboleth.atlassian.net/wiki/spaces/SC/pages/1843887946/ShibMetaExt+V1.0
    author:
      - org: Shibboleth Consortium
    date: false
  FASTFED:
    title: "FastFed Core 1.0"
    target: https://openid.net/specs/fastfed-core-1_0.html
    author:
      - org: OpenID Foundation
    date: false
  EDUGAIN:
    title: "What is eduGAIN"
    target: https://edugain.org/about-edugain/what-is-edugain/
    author:
      - org: GEANT
    date: false

---

--- abstract

This document defines the Domain-Authorized Issuer (DAI) Trust
Method: a `subject_namespace_authorization` Trust Method for the
OAuth Identity Assertion Trust Framework in which the owner of a
subject namespace (typically a DNS domain) publishes a policy
listing the OAuth authorization servers it authorizes to assert
identities in that namespace. The mechanism uses the DNS-based
authority-publication pattern operators already deploy for CAA,
MTA-STS, SPF, and DKIM. A Resource Authorization Server uses the
published policy to verify that an identity assertion's issuer is
authorized for the asserted subject namespace.

The lookup defined by this document is verifier-side: given an
identity assertion in hand, the Resource Authorization Server
locates the Subject Authority's issuer authorization policy.
Client-side discovery of which Assertion Issuer to use before an
assertion exists is a separate use case and is deferred to future
work.

This document also defines the Issuer Authorization Policy wire
format that the Trust Method consumes. The parent trust framework
specification owns the generic Trust Policy document, Trust Method
category structure, cross-category combination rule, and Subject
Authority Determination concept.

--- middle

# Introduction

OAuth deployments using identity-assertion grants (e.g., ID-JAG, or
generic JWT-bearer assertions carrying an identity claim; see
{{TRUST-FRAMEWORK}}) need to answer "is this Assertion Issuer
authorized to assert about subjects in this namespace?". An issuer authenticated by federation
membership is not, by that membership alone, entitled to assert
about subjects in any particular namespace; the namespace owner is
the only authoritative source of that delegation.

The Domain-Authorized Issuer (DAI) Trust Method lets the owner of a
DNS-namable subject namespace publish, in DNS, the set of Assertion
Issuers it authorizes for its namespace. The DNS record can carry
the authorized issuers inline (for simple deployments) or point at
an HTTPS-hosted JSON document containing richer policy (validity
windows, format restrictions, tenant binding, multiple issuers).
A Resource Authorization Server that does not take policy content
from DNS can instead fetch the policy from a dedicated HTTPS host,
`oauth-issuer-policy.{domain}`. In both cases the DNS record is the
Subject Authority's explicit opt-in: a namespace that publishes no
record is not covered.
Authority is established by control of the publication channel:
whoever can publish at `_oauth-issuer-policy.{domain}` is itself
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

DAI extends {{TRUST-FRAMEWORK}}, which owns the Trust Policy
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

This document uses terminology from {{TRUST-FRAMEWORK}}: Resource
Authorization Server, Assertion Issuer, Subject Authority, Trust
Policy, Issuer Authorization Policy, Authority Holder, Delegate,
Delegation Artifact, and Validator. Subject Identifier formats
follow {{RFC9493}}.

One term is specific to this document:

Domain-Authorized Issuer (DAI):
: The Trust Method defined by this document, in which a Subject
Authority publishes, over DNS and HTTPS, the set of Assertion
Issuers it authorizes for its namespace.

# Issuer Authorization Policy Document {#dii-document}

The Issuer Authorization Policy is a Delegation Artifact: a Subject
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

The Issuer Authorization Policy is a JSON object served over HTTPS.
Publishers MUST serve it with media type `application/json`;
consumers additionally accept any media type using the structured
`+json` suffix ({{dii-failures}}). It has the following members:

`subject_authority`
: REQUIRED. String. The Subject Authority identifier this policy
  applies to. For DNS-domain Subject Authorities the publisher MUST
  write this value in A-label (ASCII) form per {{RFC5891}}, mirroring
  the `authority=` directive ({{dii-dns-record}}). Consumers MUST
  verify this value matches the Subject Authority computed in
  {{TRUST-FRAMEWORK}} §Subject Authority Determination after applying
  the comparison rules for that Subject Authority form.

`authorized_issuers`
: REQUIRED. JSON array of authorized issuer objects, which MAY be
empty. An empty array is an explicit denial: the Subject Authority
authoritatively authorizes no issuer. The retrieval of such a policy
is Affirmative ({{dii-failures}}); the denial takes effect at
verification, where no entry can match ({{dii-verification}}).
Consumers MUST NOT fall through to any other
channel on encountering a denial. Each object has:

  `issuer`
  : REQUIRED. String. The Assertion Issuer identifier. For OAuth
  authorization server issuer identifiers, this value MUST be an
  absolute HTTPS URL with no fragment component. The URL MAY contain a
  path component. No normalization is applied before comparison: the
  value and the JWT `iss` claim are compared by exact case-sensitive
  string equality (octet-for-octet), consistent with {{RFC8414}}
  issuer-identifier comparison. Consequently `https://a` and
  `https://a/` are distinct issuers, and no trailing-slash or
  percent-encoding normalization is performed. Query components are
  NOT RECOMMENDED because many issuer metadata profiles do not use
  them. Issuer identifiers whose string form contains a `;` cannot be
  expressed in the inline DNS form ({{dii-dns-record}}) and MUST be
  published via the HTTPS document instead.

  `tenant`
  : OPTIONAL. Non-empty string. The issuer-side tenant identifier
  authorized for this Subject Authority, corresponding to a top-level
  `tenant` claim carried by the assertion (in ID-JAG, {{ID-JAG}} §6.1).
  When present, the assertion's top-level `tenant` claim MUST be
  present and MUST exactly match this value using case-sensitive
  string comparison; an assertion that lacks the `tenant` claim or
  carries a different value does not match this entry. When absent,
  the entry matches only assertions that carry no top-level `tenant`
  claim, so under a grant profile that carries `tenant`, listing a
  shared issuer without `tenant` authorizes none of its tenants
  ({{dii-multi-tenant}} covers grant profiles that do not). Grant
  profiles that do not carry a
  `tenant` claim (e.g., the generic JWT-bearer grant of {{RFC7523}})
  match only entries that omit `tenant`. Tenant values are
  issuer-specific and MUST NOT be compared across issuers. To
  authorize multiple tenants of the same shared issuer, publish one
  entry per (issuer, tenant) pair. Guidance for shared-issuer
  multi-tenant Identity Providers is in {{dii-multi-tenant}}.

  `subject_identifier_formats`
  : OPTIONAL. JSON array of Subject Identifier format names
  ({{RFC9493}}) this Assertion Issuer is authorized for. If omitted,
  the Assertion Issuer is authorized for any format that resolves to
  this Subject Authority, including formats whose extraction
  procedures are registered after the policy is published; a Subject
  Authority that wants to exclude future formats lists the formats it
  authorizes.

  `valid_from`
  : OPTIONAL. {{RFC3339}} date-time. The delegation MUST NOT be treated
  as valid before this time. A consumer MAY apply a small clock-skew
  tolerance (at most 5 minutes), consistent with JWT `nbf` conventions
  ({{RFC7519}} Section 4.1.5). This tolerance applies only to
  `valid_from`; it MUST NOT be applied to `valid_until` to extend a
  delegation past its stated end.

  `valid_until`
  : OPTIONAL. {{RFC3339}} date-time. The delegation MUST NOT be treated
  as valid at or after this time. No clock-skew tolerance is permitted
  on this bound.

`last_updated`
: OPTIONAL. {{RFC3339}} date-time at which the policy was last published.
  It is not decision-affecting ({{TRUST-FRAMEWORK}} §Terminology).

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims
  ({{signed-policy}}). This member follows the signed metadata pattern
  used by {{RFC8414}} and {{RFC9728}}. Its presence does not by itself
  require all consumers to support signed policy processing;
  consumers apply {{signed-policy}} and any local policy that requires
  object-level integrity. A Subject Authority that needs to force
  signature processing lists `signed_policy` in `crit`.

`crit`
: OPTIONAL. Array of member names a consumer MUST understand to process
  the policy safely, as defined in {{TRUST-FRAMEWORK}} §Critical
  Members. A consumer that fails to recognize, or does not implement
  processing for, one or more listed members MUST treat the policy as
  malformed. Publishers MUST place `crit` in the outer (unsigned)
  document: a `crit` present only in a `signed_policy` JWT payload is
  invisible to consumers that do not process signatures and therefore
  has no effect on them. It MAY additionally be duplicated as a claim
  in the signed JWT so that its value is integrity-protected. The DNS
  record form does not carry `crit` ({{crit-dns-form}}).

Consumers MUST ignore unrecognized members, except those named in
`crit`. A document containing duplicate member names (at the top level
or within an `authorized_issuers` entry object) is malformed and MUST
be rejected. The following validation rules apply; a policy failing
any of them is malformed:

| Member | Required type / structure | Additional rule |
|-|-|-|
| `subject_authority` | string, required | matches the computed Subject Authority |
| `authorized_issuers` | array of objects, required (MAY be empty for explicit denial) | each element has `issuer` |
| `issuer` | string, required | syntactically valid issuer identifier for the applicable grant profile |
| `tenant` | string, optional | non-empty (see member definition) |
| `subject_identifier_formats` | array of strings, optional | |
| `valid_from`, `valid_until`, `last_updated` | {{RFC3339}} date-time, optional | |
| `signed_policy` | string, optional | signed JWT as defined in {{signed-policy}} |
| `crit` | array of strings, optional | non-empty; every listed member recognized and implemented, else malformed ({{TRUST-FRAMEWORK}} §Critical Members) |

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
      "issuer": "https://accounts.shared.example",
      "tenant": "example.com",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-01T00:00:00Z"
}
~~~

## Signed Policy {#signed-policy}

The `signed_policy` member provides cryptographic integrity for the
policy members it carries as claims. When the signed JWT contains
every decision-affecting member ({{TRUST-FRAMEWORK}} §Terminology), it
provides object-level integrity for the whole document. It follows
the signed metadata pattern defined for authorization server metadata
in {{RFC8414}} and protected resource metadata in {{RFC9728}}.

A consumer processes `signed_policy` only when it has an acceptable
verification key for it: the key a `key=` directive pins, or a key
configured out of band for the Subject Authority. A consumer without
one treats the policy as malformed if `crit` lists `signed_policy` or
its local policy requires object-level integrity for the Subject
Authority, and otherwise ignores `signed_policy` and evaluates the
unsigned members. Such a consumer has no object-level integrity to
lose, and a publisher never publishes outer and signed values that
conflict, so on a well-formed policy it reaches the same result as a
consumer that processes the signature. The requirements on consumers
in the rest of this section apply to a consumer that processes
`signed_policy`.

The `signed_policy` value is a JWT {{RFC7519}} in JWS Compact
Serialization {{RFC7515}} containing policy members as claims. The
JWT MUST be digitally signed using an asymmetric algorithm, MUST
contain an `iss` claim identifying the party attesting to the signed
policy claims, and MUST contain `iat`. It MUST contain `exp`, so that
a superseded signed policy cannot be replayed indefinitely (relevant
when the policy is hosted on shared infrastructure,
{{third-party-policy-hosts}}); consumers MUST reject an expired
`signed_policy`. Publishers SHOULD keep `exp` close to `iat` (for
example, a few days), because a policy host can replay a superseded
signed policy until its `exp` to a consumer that holds no cached
copy. A consumer that holds a cached signed policy for the
same Subject Authority MUST reject a `signed_policy` whose `iat` is
earlier than the cached one's, so that an older signed policy cannot
be replayed within its validity period. The JOSE header SHOULD
contain a `kid` identifying the signing key. The JWT payload SHOULD
NOT contain a `signed_policy` claim.

Algorithms: {{RFC8725}} (JWT Best Current Practices) applies
unchanged. In addition, the JWT MUST NOT use a MAC algorithm
(HS256/384/512); verification keys are widely distributed and a MAC
scheme would require sharing the signing key with every Validator,
defeating the authority binding. Implementations MUST support ES256
and SHOULD support EdDSA and ES384; RS256 with >=2048-bit keys MAY be
supported for compatibility.

Per {{RFC8725}} §3.11 (cross-context confusion), the JWT `typ`
header MUST be `issuer-authorization-policy+jwt`. This is the
registered media subtype with the `application/` prefix omitted, per
the {{RFC8725}} §3.11 convention; the corresponding media type is
registered in {{iana-dii-media-type}}.

The JWT payload MUST contain `subject_authority`. When a `key=`
directive pins the verification key, a signature that verifies with
that key binds the signer to the Subject Authority, and the JWT `iss`
claim is not further constrained. Otherwise the JWT `iss` claim MUST
either equal the `subject_authority` value (for example,
`acme.example`, not `https://acme.example`) or identify a signing
authority that local policy or an applicable Trust Method
establishes as controlled by the Subject Authority. Consumers MUST
NOT treat a signature by an Assertion Issuer the policy authorizes as
proof of Subject Authority authorization unless such a relationship
is explicitly established.

The verification key MUST be resolved through a channel independent
of the one that carried the policy document, and MUST be one of:

- a key whose thumbprint the Subject Authority publishes in a
  `key=` directive of its DNS pointer record ({{dii-dns-record}}). A
  `key=` thumbprint binds the document to what the consumer already
  trusts DNS for: it defeats substitution by the policy host or a
  shared edge in front of it, not compromise of DNS, which already
  selects the policy host; DNSSEC raises that bound.
- a key configured out of band at the consumer, which rests on the
  consumer's own key-provisioning process.

Under the HTTPS-only lookup mode, a `key=` directive in the opt-in
record applies as it does to any pointer record
({{trust-method-https-authorized-issuer}}). An attacker who controls the
publication channel can substitute both the policy and, if the key
is fetched over that same channel, the key. Absent an independent
channel, the signature provides integrity no stronger than channel
control (which already establishes authority), and consumers MUST NOT
rely on it to defend against compromise of that channel.

If both unsigned policy members and `signed_policy` are present, the
signed policy claims MUST be used as the policy values for all claims
present in the JWT. Unsigned members that are not represented as
claims in the JWT MAY be used subject to the normal processing rules
for unrecognized members. A conflict exists when a member name
appears in both the unsigned outer document and the signed JWT
payload AND the two values are not equal when compared as parsed
JSON values (member order and insignificant whitespace ignored;
equivalently, their JCS {{RFC8785}} serializations differ). Consumers
MUST reject a policy that contains any such conflict; an attacker who
can modify the outer document but not the signed JWT otherwise has a
lever to inject visible-but-ignored members that may mislead
operators or downstream tooling.

A Subject Authority that needs to require `signed_policy` processing
by all conforming consumers lists `signed_policy` in `crit`; a
consumer that does not implement `signed_policy` processing then
rejects the document rather than silently ignoring the signature.
Absent a `crit` entry, the presence of `signed_policy` provides
integrity only for consumers that support and verify it, for
deployments where local policy requires signed policy processing, or
where a `key=` directive requires it ({{dii-dns-record}}).

If a consumer's local policy requires object-level integrity through
`signed_policy`, the consumer MUST verify the signed JWT before
acting on the policy, and the JWT payload MUST contain every
recognized decision-affecting member used by that consumer. The
consumer MUST NOT use unsigned recognized decision-affecting members
that are absent from the JWT payload, except `crit`, which a consumer
always honors from the outer document because it can only cause
rejection. If signature verification
fails, if the verification key is unacceptable, if the JWT is
malformed, if the required issuer binding above is not satisfied, or
if the JWT omits a recognized decision-affecting member required for
evaluation, the consumer MUST reject the policy as malformed.

# Publication

A Subject Authority publishes the Issuer Authorization Policy
through one of the publication channels defined in
{{publication-profiles}}. Throughout this document, `{A}` denotes
the Subject Authority rendered as a DNS name (A-label form,
{{dii-dns-record}}). DNS publication uses the TXT record at
`_oauth-issuer-policy.{A}`, following the pattern of CAA, MTA-STS,
SPF, and DKIM ({{dns-authority-patterns}}). For Resource
Authorization Servers that take no issuers or policy location from
DNS, the policy is also served at the default URL on a dedicated
policy host, `oauth-issuer-policy.{A}` ({{dii-https-url}}); a TXT
record whose `uri=` names that URL is the opt-in.

## Publication Channels {#publication-profiles}

This document defines three publication channels. A Subject
Authority chooses based on operational constraints; all carry the
same document model ({{dii-document}}), although the inline form
expresses only part of it ({{mechanism-limits}}). The canonical
lookup procedure ({{dii-lookup}}) consults only DNS and the `uri=`
target a DNS record names; the dedicated policy host is consulted
only by the HTTPS-only lookup mode
({{trust-method-https-authorized-issuer}}).

| Channel | DNS form | Document | Authority binding | When to use |
|-|-|-|-|-|
| 1: DNS-Inline | TXT with `issuer=` ({{dii-dns-record}}) | None (carried in TXT) | DNS control of `{A}` | Common case: authorize an issuer for a namespace with no rich policy |
| 2: Authority-Hosted HTTPS | TXT with `uri=`, consumed by canonical lookup | HTTPS-hosted JSON under Subject Authority's operational control | DNS control of `{A}` AND TLS on the authority-operated host | Rich policy (validity windows, format restrictions, tenant binding) at a controlled origin |
| 3: Dedicated Policy Host | TXT with `uri=` naming the default URL, consumed by the HTTPS-only lookup mode as the opt-in | HTTPS-hosted JSON at `https://oauth-issuer-policy.{A}/.well-known/oauth-issuer-policy` | DNS control of `{A}` (opt-in record) AND of `oauth-issuer-policy.{A}`, AND TLS on that host | Serving Resource Authorization Servers that use the HTTPS-only lookup mode |

## Dedicated Policy Host {#dedicated-policy-host}

A Subject Authority that serves Resource Authorization Servers using
the HTTPS-only lookup mode publishes the policy at the default URL
on the dedicated policy host `oauth-issuer-policy.{A}`
({{dii-https-url}}). The host name alone is not an opt-in: some
domains let untrusted users claim subdomains or serve content on
them, and an attacker could claim `oauth-issuer-policy` ({{RFC8461}}
Section 10.3). Nor is every TXT record at `_oauth-issuer-policy.{A}`:
a Subject Authority that publishes only an inline record, or a pointer
to another host, has not set up the dedicated host. The opt-in to the
HTTPS-only lookup mode is a recognized record at
`_oauth-issuer-policy.{A}` ({{dii-dns-record}}) whose `uri=` value is
exactly the default URL. That mode requires such a record before it
fetches from the dedicated host
({{trust-method-https-authorized-issuer}}), and the same record
directs canonical lookups to the same document. Subject Authorities
whose domains let others claim subdomains SHOULD also reserve the
`oauth-issuer-policy` label. Unlike the apex, the dedicated host can
be delegated to a hosting provider (for example, by a CNAME record)
without giving that provider control of the Subject Authority's web
origin; a `key=` directive in the opt-in record keeps that provider
from altering the policy ({{third-party-policy-hosts}}).

## DNS Record {#dii-dns-record}

The Subject Authority publishes one or more DNS TXT records at
`_oauth-issuer-policy.{A}`, where `{A}` is the Subject
Authority rendered as a DNS name. Internationalized names are converted
to A-labels by the procedure of {{TRUST-FRAMEWORK}} §Subject Authority
Determination before the prefix is prepended. DNS lookup
names are formed without a trailing dot for comparison purposes; the
wire-format root label is not part of the Subject Authority value.

The string segments of a TXT record ({{RFC1035}}) are concatenated
without separator and interpreted as ASCII text (a record containing
a non-ASCII octet is malformed, since all defined directive values
are ASCII by construction).

A record is *recognized* if the version token `v=oauth-issuer-policy1`
(case-sensitive) appears at the start, terminated by end of record, by
optional whitespace, or by a `;` separator; leading whitespace before
the token is permitted. The version token matches only that exact
string: a different token (for example, a future
`v=oauth-issuer-policy2`) is not recognized, so the record is treated
as unrecognized (not malformed) and is ignored by the recognition
rule. Records that do not begin with a supported version token MUST be
ignored. A `v=` directive appearing anywhere other than the start of
a recognized record makes the record malformed ({{dii-failures}}).

The remainder of a recognized record is a sequence of `name=value`
directives separated by `;`. Parsing rules:

- Whitespace (space and horizontal tab) immediately before and after
  each directive is ignored. A value MUST NOT contain whitespace or
  non-ASCII characters; the directive values defined here (domain
  names in A-label form and absolute HTTPS URLs) are ASCII by
  construction.
- Directive names are case-insensitive ASCII. Values are
  case-sensitive and preserved as written.
- A directive splits into name and value at the first `=`. Subsequent
  `=` characters are part of the value. A directive with no `=` makes
  the record malformed ({{dii-failures}}).
- The value runs to the next `;` or end of record. The character `;`
  MUST NOT appear in a value (see the issuer-identifier constraint in
  {{dii-document}}).
- CR, LF, and NUL MUST NOT appear in a value.
- An empty directive (two consecutive `;` separators with no
  intervening `name=value`) makes the record malformed
  ({{dii-failures}}).
- Unrecognized directives MUST be ignored.

The parsing rules above are summarized in the following ABNF
({{RFC5234}} with the case-sensitive string extension of {{RFC7405}})
for implementer convenience; the prose above is normative if any
disagreement exists.

~~~ abnf
record        = OWS version OWS *( ";" directive ) [ ";" OWS ]
version       = %s"v=oauth-issuer-policy1"
directive     = OWS name "=" value OWS
name          = 1*( ALPHA / DIGIT / "_" / "-" )
value         = *vchar-no-semi
vchar-no-semi = %x21-3A / %x3C-7E   ; VCHAR (%x21-7E) minus ";"
OWS           = *( SP / HTAB )
~~~

The following directives are defined:

`authority=A`
: REQUIRED. The Subject Authority identifier this record claims to
  bind. The publisher MUST write this value in A-label (ASCII) form
  per {{RFC5891}}; consumers compare it against the Subject Authority
  computed from the query name (also in A-label form) using the
  case-insensitive ASCII comparison rules in {{TRUST-FRAMEWORK}} §Subject Authority Determination.
  Records whose `authority=` value does not match are discarded (see
  the lookup procedure in {{dii-lookup}}). A recognized record
  containing more than one `authority=` directive is malformed
  ({{dii-failures}}).

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

`key=THUMBPRINT`
: OPTIONAL. The JWK SHA-256 thumbprint {{RFC7638}}, base64url-encoded
without padding, of the key that signs the `signed_policy` of the
document named by `uri=`. Valid only in a record that carries
`uri=`; in any other record it is malformed. At most two distinct
`key=` values may appear across the remaining records, so that a
Subject Authority can publish an old and a new key during a
rollover; more than two is malformed. When
present, the fetched document MUST carry a `signed_policy` whose JWS
header carries the signing key in its `jwk` parameter ({{RFC7515}}
Section 4.1.3), the thumbprint of that key MUST equal one of these
values, and
the signature MUST verify with it ({{signed-policy}}); a document
that fails any of these checks is malformed. A `key=` directive also
makes the consumer process the document as one whose object-level
integrity its local policy requires ({{signed-policy}}): the signed
JWT MUST
contain every decision-affecting member the consumer uses, and the
consumer MUST NOT use unsigned decision-affecting members that are
absent from it, other than `crit` ({{signed-policy}}).

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
https://oauth-issuer-policy.{A}/.well-known/oauth-issuer-policy
~~~

where `{A}` is the Subject Authority rendered as a DNS name (A-label
form). The URL uses the `https` scheme, the host
`oauth-issuer-policy.{A}`, the default HTTPS port (443), and the
well-known path {{RFC8615}}; it has no userinfo, port, query, or
fragment component. A Subject Authority whose form has no DNS-name
representation (none is defined in this document beyond the
`email`-derived DNS domain) cannot use the dedicated host. Consumers
fetch this URL with an HTTP GET over HTTPS with TLS server
authentication and interpret the response body as the JSON document
defined in {{dii-document}}.

A URL obtained from a DNS `uri=` directive is fetched the same way;
the host serving the URL is responsible for TLS server authentication
of itself, not of `A`.

Every policy fetch defined here (the default URL and a `uri=`
target) is an outbound fetch subject to {{TRUST-FRAMEWORK}}
§Outbound Fetches; see also {{dos-ssrf}}.

Consumers MUST NOT follow HTTP redirects when fetching a policy, as
in MTA-STS ({{RFC8461}} Section 3.3): a 3xx response other than a 304
validating a held cached policy is Indeterminate ({{dii-failures}}).
Policy hosting on another host is expressed only through the `uri=`
pointer, where the delegation is published in DNS and visible to the
Subject Authority. The response MUST have HTTP status 200 and a media
type of `application/json` or any media type using the structured
`+json` suffix (media type parameters ignored); any other status is
classified per {{dii-failures}}.

## HTTPS Policy Document Contract {#https-policy-document-contract}

The document retrieved from either the default URL or a DNS `uri=`
pointer is the Issuer Authorization Policy defined in
{{dii-document}}; consumers MUST NOT interpret any other JSON shape
as an Issuer Authorization Policy. Consumers MUST validate the
complete policy as a unit; a malformed entry makes the whole policy
malformed, not just the offending entry.

Bounds (a response exceeding any bound is classified per
{{dii-failures}}): consumers MUST accept a policy document of at least
64 KiB and MAY reject one larger; publishers MUST keep the document
within 64 KiB. Consumers MUST accept at least 100 `authorized_issuers`
entries and MAY reject more; publishers MUST NOT publish more than
100. Consumers SHOULD limit JSON nesting
depth (the defined document has a fixed shallow structure); fetch
timeouts follow {{TRUST-FRAMEWORK}} §Outbound Fetches. Consumers
SHOULD send a conditional request
(for example, `If-None-Match`) when they hold a cached policy, treating
a 304 response per {{dii-failures}}. A consumer MUST NOT treat a 304
as renewing a held policy unless the current record's `uri=` and
`key=` values equal those under which the held policy was retrieved
and verified, and any `signed_policy` in it has not expired; it
stores those values with the cached policy, and otherwise fetches
without a conditional request.

The document's shape and members are defined, with an example, in
{{dii-document}}.

# Lookup Procedure {#dii-lookup}

To retrieve the Issuer Authorization Policy for a Subject Authority `A`,
a Resource Authorization Server performs the following steps. The
procedure consults DNS only, plus the `uri=` target a DNS record
names. The TXT record is the Subject Authority's opt-in: its absence
is Negative, and no other channel is consulted.

This is the canonical procedure used by the
`domain_authorized_issuer` Trust Method. The HTTPS-only lookup mode
defined in {{trust-method-https-authorized-issuer}} uses the DNS
record only as an opt-in and fetches the policy only from the
dedicated policy host.

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
      mismatch, treat the response as `malformed`. (A `malformed`
      outcome is classified as Indeterminate, {{dii-failures}}.)
      Records discarded for `authority=` mismatch are not otherwise
      validated.

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
        records are ignored: their values do not contribute to the
        policy, but the directive-validity rules of
        {{dii-dns-record}} were already applied in step 2a, so a
        record set malformed under those rules never reaches this
        step. If a `key=` directive is present, the document is
        verified as that directive requires.

   c. Otherwise (no `uri=` present), construct a virtual Issuer
      Authorization Policy with `subject_authority` set to `A` and
      one entry in `authorized_issuers` for each distinct `issuer=`
      value across the remaining records. Entry order carries no
      semantics ({{dii-verification}}); the deduplicated values form
      a set. Entries have no `tenant`,
      `subject_identifier_formats`, `valid_from`, or `valid_until`.
      The virtual policy is processed identically to one fetched over
      HTTPS, except that its cache lifetime is derived from DNS TTLs
      as described in {{dii-caching}}.

3. If the DNS response is `negative-authoritative`, the lookup is
   Negative ({{dii-failures}}): the Subject Authority has not
   published a policy for this mechanism. Consumers MUST NOT fetch
   the policy from any HTTPS location in this case.

4. If the DNS response is `indeterminate`, lookup has failed
   (Indeterminate). Consumers MUST NOT consult another channel,
   since an attacker who suppresses DNS responses could otherwise
   force the consumer onto a path the attacker has compromised
   separately.

Under the HTTPS-only lookup mode, a Subject Authority is found only
if it publishes both a record whose `uri=` is the default URL and a
policy on the dedicated host ({{combining-dai-methods}}). A Resource
Authorization Server that cannot resolve DNS (resolver failure,
untrusted resolution path) classifies the lookup as Indeterminate
({{dii-failures}}).

## Failure Handling {#dii-failures}

DAI lookup outcomes map onto the Affirmative / Negative /
Indeterminate lookup states of the Authority Delegation Model
({{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed). The normative
requirements (fail closed on Negative and Indeterminate; no
fallback to a different Authority Source; bounded cache reuse on
transient Indeterminate) apply unchanged; this section maps the
concrete DAI outcomes onto those states.

| State | DAI outcomes |
|-|-|
| Affirmative | A well-formed Issuer Authorization Policy was retrieved (inline DNS, DNS pointer plus HTTPS fetch, or the dedicated host under the HTTPS-only lookup mode), its `subject_authority` matches `A`, and its structural validation succeeds; this includes a policy whose `authorized_issuers` array is empty (explicit denial, evaluated in {{dii-verification}}). HTTPS responses, when applicable, are 200 OK with a media type of `application/json` or a `+json`-suffixed type. A 304 (Not Modified) response to a conditional request validating a held cached policy within the absolute ceiling of {{dii-caching}}, under the conditions of {{https-policy-document-contract}}, renews its freshness and is classified as the held policy's state; it does not reset the absolute cache-entry age. |
| Negative | Under canonical lookup, DNS `negative-authoritative`. Under the HTTPS-only lookup mode ({{trust-method-https-authorized-issuer}}), DNS `negative-authoritative` for the opt-in record, or no remaining record whose `uri=` is the default URL. No policy is published at the location the lookup consults. |
| Indeterminate | Any other outcome, fail-closed by default. See enumeration below. |

The Indeterminate state covers:

- **DNS-side**: SERVFAIL, REFUSED, timeout, truncation with no
  successful retry.
- **HTTPS transport**: TLS error, connection failure, or a policy
  host that cannot be resolved, including a dedicated host name that
  does not exist.
- **HTTPS response**: 5xx; any 4xx, including 404 and 410, since
  the Subject Authority published a record naming the location and a
  missing document there is not an authoritative absence; 2xx other
  than 200; any 3xx other than a
  304 validating a held cached policy (redirects are not followed,
  {{dii-https-url}}); unsupported media type; a body larger than the
  size limit, or containing more `authorized_issuers` entries than
  the consumer's entry limit ({{https-policy-document-contract}}).
- **DNS record validation**: `authority=` missing from any recognized
  record; all recognized records discarded for `authority=` mismatch;
  more than one `authority=` in a record; a recognized record with
  neither `uri=` nor `issuer=`; multiple distinct `uri=` values; a
  `key=` directive in a record without `uri=`, or more than two
  distinct `key=` values; an empty or otherwise malformed directive.
- **HTTPS document validation**: body that is not a syntactically
  valid Issuer Authorization Policy; `subject_authority` that does
  not match `A`; a document that fails the verification a `key=`
  directive requires.

Any outcome not explicitly mapped to Affirmative or Negative above
MUST be treated as Indeterminate.

The following deterministic conflict rules apply:

- Multiple recognized TXT records containing no `uri=` directive
  are merged into a single virtual policy. `issuer=` values across
  records are deduplicated into a set; order carries no semantics
  ({{dii-lookup}}, step 2c).

- If any recognized record contains a `uri=` directive after
  `authority=` filtering, the HTTPS document at the single `uri=`
  target is authoritative and all `issuer=` directives across all
  records are ignored ({{dii-lookup}}, step 2b). More than one distinct
  `uri=` value is `malformed`.

- Canonical lookup takes the policy from the DNS record and the
  `uri=` target it names; the HTTPS-only lookup mode requires a
  record whose `uri=` is the default URL and takes the policy only
  from that URL. Because the opt-in record directs canonical lookup
  to the same URL, both modes retrieve the same document
  ({{dedicated-policy-host}}).

- Multiple `authorized_issuers` entries for the same `issuer` value but
  different `tenant` values are independent authorizations. Entries with
  `tenant` do not shadow entries without `tenant` for the same `issuer`.

- Only the Subject Authority computed by the extraction procedure for
  the assertion's Subject Identifier applies. Another Subject Authority's
  policy cannot grant authority over that subject.

- If the assertion's claims conflict with the matched policy entry, the
  assertion fails the Trust Method.

Consumers MUST NOT treat a Negative or Indeterminate outcome as
satisfying the Trust Method, except that a cached Affirmative policy
MAY be used during an Indeterminate live retrieval within the
stale-if-error bound of {{dii-caching}}. Whether the assertion is
then rejected follows {{TRUST-FRAMEWORK}} §Multiple Authority Sources
Within a Category:
an Indeterminate outcome, like a policy that does not authorize the
issuer, leaves the `subject_namespace_authorization` category
unsatisfied whatever other methods yield, while a Negative outcome
lets another method in that category supply evidence.

# Verification {#dii-verification}

When the `domain_authorized_issuer` Trust Method is evaluated, the
Resource Authorization Server MUST:

1. Determine the Subject Authority from the assertion's Subject
   Identifier per {{TRUST-FRAMEWORK}} §Subject Authority Determination. If the format is not registered
   in {{TRUST-FRAMEWORK}} §Subject Authority Extraction Procedures Registry, reject the assertion.

2. Retrieve the Issuer Authorization Policy by applying the
   procedure in {{dii-lookup}}. Negative and Indeterminate
   outcomes ({{dii-failures}}) MUST NOT satisfy the Trust Method,
   except as {{dii-caching}} permits for a cached Affirmative policy
   during an Indeterminate live retrieval.

3. Verify the policy's `subject_authority` matches the computed
   Subject Authority. (Virtual policies satisfy this by
   construction.)

4. Determine whether any entry in `authorized_issuers` matches. An
   entry matches when ALL of the following hold:

   a. `issuer` equals the JWT `iss` claim under case-sensitive URL
      string comparison (the comparison rule fixed in
      {{dii-document}}).

   b. If the entry contains a `tenant` member, its value exactly
      matches the assertion's top-level `tenant` claim (in ID-JAG,
      {{ID-JAG}} §6.1) under case-sensitive string comparison. An
      entry with `tenant` does not match an assertion that lacks the
      `tenant` claim; an entry without `tenant` does not match an
      assertion that carries a `tenant` claim. Under a grant profile that
      carries no `tenant` claim, any `tenant` claim physically present
      in the assertion MUST be ignored for entry matching.

   c. If the entry contains `subject_identifier_formats`, the Subject
      Identifier format of the assertion's subject is listed.

   d. If `valid_from` or `valid_until` are present, the current time
      is within the validity window (with the skew rules of
      {{dii-document}}).

   The Trust Method is satisfied when step 3 succeeds and at least
   one entry matches under (a)-(d). Entry order in
   `authorized_issuers` carries no semantics: the
   outcome is the boolean "does any entry match," so two consumers
   evaluating the same policy against the same assertion reach the
   same result regardless of array order or of which matching entry
   they examine first.

If step 1 fails, the assertion is rejected ({{TRUST-FRAMEWORK}}
§Resource Authorization Server Processing, step 5c). If step 2 does
not yield a policy, the Trust Method is not satisfied, with the
category-level effect stated in {{dii-failures}}. When the Trust
Method is not
satisfied and, as a result, the cross-category combination rule
({{TRUST-FRAMEWORK}} §Cross-Category Combination Rule) is not met,
the Resource Authorization Server MUST reject the assertion with an
OAuth `invalid_grant` error.

## Observing Before Enforcing {#observe-before-enforce}

This document defines no mode in which a published policy is
evaluated without being enforced. Under fail-closed evaluation, a
namespace with no policy gets no authorization from this Trust
Method ({{dii-failures}}), so a Subject Authority's first publication
withdraws nothing that this method granted. A mode that accepted assertions despite
a mismatch would instead admit more issuers than publishing nothing.

A Resource Authorization Server introducing `domain_authorized_issuer`
can observe before it enforces: it evaluates the Trust Method for
incoming assertions and logs the outcome (Subject Authority,
Assertion Issuer, matched or not) while its Trust Policy does not yet
list the method. Because only listed methods are applicable
({{TRUST-FRAMEWORK}} §Category Applicability), observation does not
affect acceptance. The Resource Authorization Server can share what
it observes with affected Subject Authorities out of band; a
reporting mechanism is sketched in {{future-extensions}}.

# Caching {#dii-caching}

Freshness and cache limits for the Issuer Authorization Policy:

- **Freshness lifetime.** The inline form's virtual policy is fresh
  for the minimum TTL of the records that construct it. A policy
  fetched through a `uri=` pointer is fresh for the lesser of the
  pointer record's TTL and the document's HTTP freshness lifetime
  ({{RFC9111}}). A policy fetched from the dedicated host under the
  HTTPS-only lookup mode is fresh for the lesser of the opt-in
  record's TTL and its HTTP freshness lifetime. A
  document without explicit freshness information (no `max-age` and
  no `Expires`) is fresh for a local default that MUST NOT exceed 1
  hour. Consumers MUST NOT treat a cached policy as fresh beyond
  these lifetimes, except that they MAY apply a minimum freshness
  lifetime of up to 5 minutes even when a TTL or HTTP lifetime is
  shorter, to coarsen the timing signal discussed in {{privacy}}.
- **Unvalidated DNS.** Consumers MUST NOT treat a DNS result that was
  not DNSSEC-validated as fresh for more than 1 hour, whatever its
  TTL: a spoofed answer chooses its own TTL
  ({{dns-integrity-and-compromise}}).
- **Steady-state lifetimes.** Subject Authorities SHOULD publish
  records and HTTPS policies with a freshness lifetime of at most 1
  hour, reducing it further during an active revocation.
- **Absolute ceiling.** Consumers MUST enforce an absolute local
  ceiling of at most 24 hours on the age of any cached policy entry,
  regardless of TTL or `Cache-Control`. A cached entry older
  than the ceiling MUST NOT be used and MUST be re-fetched with an
  unconditional request.
- **Stale-if-error.** When a live retrieval is Indeterminate, a
  consumer MAY continue to use a cached Affirmative policy for at
  most 1 hour after its freshness lifetime ended, and never past the
  absolute ceiling. Repeated Indeterminate retrievals MUST NOT extend
  this window; a successful live retrieval (Affirmative or Negative)
  replaces the cached entry. The bound keeps an attacker who can
  sustain denial of service against the publication channel from
  extending revocation latency toward the absolute ceiling.
- **Negative results** SHOULD be cached, to bound lookup work under
  load ({{dos-ssrf}}), for no longer than the lesser of the DNS
  negative TTL ({{RFC2308}}) and 1 hour
  (recommended: 5 minutes). The short cap makes a Subject Authority's
  first publication, and its recovery from a brief publication-channel
  takeover, visible promptly. The same cap SHOULD apply to a cached
  explicit-denial policy ({{dii-document}}).
- Indeterminate outcomes MAY be cached for a short period
  (recommended: no more than 5 minutes) to absorb retry storms;
  an Indeterminate cache entry MUST NOT be treated as a policy and
  never satisfies the Trust Method.

# Trust Methods {#trust-methods}

This document defines `domain_authorized_issuer` as a
`subject_namespace_authorization` Trust Method of {{TRUST-FRAMEWORK}}.
DNS at `_oauth-issuer-policy.{authority}` is the publication channel
for canonical lookup; under the HTTPS-only lookup mode the record
is an opt-in and a dedicated HTTPS host serves the policy. The Trust
Method
registers in the Identity Assertion Issuer Trust Methods registry
({{TRUST-FRAMEWORK}} §Identity Assertion Issuer Trust Methods Registry).

## domain_authorized_issuer {#trust-method-domain-authorized-issuer}

The `domain_authorized_issuer` method indicates that the Assertion
Issuer is acceptable if the Subject Authority identified by the
assertion's Subject Identifier authorizes the Assertion Issuer to
assert identities in that namespace, using the publication channels
({{publication-profiles}}), lookup procedure ({{dii-lookup}}), and
verification rules ({{dii-verification}}) of this document.

This Trust Method natively expresses authorization for either form of
multi-tenant Assertion Issuer:

- **Per-tenant issuer identifiers** (for example,
  `https://login.microsoftonline.com/{tenant-id}/v2.0`): the
  `authorized_issuers[].issuer` field accepts any absolute HTTPS URL
  issuer identifier, and case-sensitive comparison against the JWT
  `iss` claim distinguishes tenants under the same host. Each
  authorized tenant is one `authorized_issuers` entry. If such an
  issuer also sends a `tenant` claim, the entry carries the same
  `tenant` value ({{dii-verification}}, step 4b), which only the
  pointer form can express.

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
`_oauth-issuer-policy.{A}`, with no HTTPS fallback. The optional `lookup`
member is used only for the HTTPS-only deployment variant in
{{trust-method-https-authorized-issuer}}.

## HTTPS-Only Deployment Variant {#trust-method-https-authorized-issuer}

Some deployments will not take policy content from DNS. Those
deployments can use the same Issuer Authorization Policy document
format, retrieved over HTTPS from the Subject Authority's dedicated
policy host ({{dedicated-policy-host}}), with the DNS record serving
as the Subject Authority's opt-in. This
is a deployment variant of the DAI
mechanism, not a second Trust Method registered by this document.

Under this variant, the Assertion Issuer is acceptable if the Subject
Authority identified by the assertion's Subject Identifier publishes
the opt-in record and an Issuer Authorization Policy on its
dedicated policy host authorizing the Assertion Issuer.

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
({{dii-verification}}), and caching rules ({{dii-caching}}), and
the DNS query of {{dii-lookup}} for the opt-in, but it takes no
issuers or policy location from DNS.

When evaluated, the Resource Authorization Server MUST:

1. Determine the Subject Authority `A` from the assertion's Subject
   Identifier per {{TRUST-FRAMEWORK}} §Subject Authority Determination. If the format is not registered in
   {{TRUST-FRAMEWORK}} §Subject Authority Extraction Procedures Registry, reject the assertion.

2. Query `_oauth-issuer-policy.{A}` and classify the response as in
   steps 1 and 2a of {{dii-lookup}}; more than one distinct `uri=`
   value, or more than two distinct `key=` values, across the
   remaining records is `malformed`. A `negative-authoritative` response is
   Negative. The Subject Authority has opted in to this mode only if
   a remaining record carries a `uri=` whose value is exactly the
   default URL ({{dedicated-policy-host}}); otherwise the outcome is
   Negative. An `indeterminate` or `malformed` response is
   Indeterminate. The records' `issuer=` values are not used.

3. Fetch the Issuer Authorization Policy from the default URL
   `https://oauth-issuer-policy.{A}/.well-known/oauth-issuer-policy`
   per {{dii-https-url}}. The Resource Authorization Server MUST NOT
   take issuers or a policy location from the DNS records: it checks
   that the opt-in record's `uri=` is the default URL rather than
   following it. If the opt-in record carries a `key=` directive, the
   document is verified as that directive requires
   ({{dii-dns-record}}). Because the Subject Authority has opted in, a
   dedicated host name that does not exist (NXDOMAIN, or no address
   records) is Indeterminate ({{dii-failures}}).

4. Classify HTTPS retrieval and document validation outcomes per
   {{dii-failures}}. Negative and Indeterminate states MUST NOT
   satisfy the Trust Method, except as {{dii-caching}} permits for a
   cached Affirmative policy during an Indeterminate live retrieval.

5. Verify the fetched policy and match the Assertion Issuer against
   `authorized_issuers` using steps 3 and 4 of {{dii-verification}}.

A Resource Authorization Server uses this variant when it requires
the policy itself to come over HTTPS and does not accept policy
content published in DNS (inline or DNS pointer). Compared to
canonical DNS-first lookup, this mode:

- Removes the TXT record's issuers and policy location from the
  trust path; the record remains the opt-in and can pin the policy's
  signing key with `key=`. The dedicated host is
  still located through DNS, and issuance of its TLS certificate
  typically relies on DNS-based validation, so DNS integrity still
  matters; what changes is that an attacker must also obtain a
  certificate for the host, not only answer a TXT query.

- Rejects the inline DNS form (where the policy is carried in the
  TXT record itself) and the DNS pointer form (where DNS points
  at a different HTTPS host). Both require trusting DNS for the
  authoritative selection of either issuers or policy host.

- Requires the Subject Authority to provision the dedicated host,
  which it can delegate to a hosting provider without giving up
  control of its apex web origin.

## Choosing the Lookup Mode {#combining-dai-methods}

Within `domain_authorized_issuer`, a Resource Authorization Server
selects one lookup mode:

- **Canonical DNS-first lookup** is the default. The Resource
  Authorization Server queries `_oauth-issuer-policy.{A}` and
  consults nothing else except the `uri=` target that record names
  ({{dii-lookup}}).

- **HTTPS-only lookup** is an explicit deployment variant. Use it
  when local policy distrusts policy content published in DNS and
  requires the policy itself to come over TLS-authenticated HTTPS
  from the dedicated host. The DNS record is still required, as the
  opt-in, and its `uri=` names the default URL; its inline issuers
  are not used.

A Trust Policy MUST NOT list more than one `domain_authorized_issuer`
object; a Resource Authorization Server treats a policy that does as
listing a malformed Trust Method object ({{TRUST-FRAMEWORK}}
§Resource Authorization Server Processing, step 5a) and rejects.
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
and, under the HTTPS-only lookup mode, of the dedicated policy host
name under it. This follows the
bootstrap model of CAA {{RFC8659}} and MTA-STS {{RFC8461}}: an
explicit DNS opt-in, a dedicated policy host, and no redirects.
Pre-existing DNS
realities (typosquatting, expired-domain takeover, registrar
account compromise) apply unchanged and are not introduced by
this document. The registrable-domain default contains
subdomain-takeover impact ({{TRUST-FRAMEWORK}} §Subject Authority
Determination); identity binding beyond DNS control (legal-entity
verification) requires out-of-band mechanisms.

DNS control is also, in practice, control of the domain's mail:
whoever controls the zone can change its MX records and receive
password-reset and account-recovery messages for its addresses.
Where email-based recovery alone is enough to take over an account
at a Resource Authorization Server, an attacker who controls the
domain's DNS gains little from this Trust Method that it did not
already have; {{I-D.hardt-email-verification}} makes the same
observation about its own DNS delegation. Where accounts are
protected by stronger recovery (for example, phishing-resistant
authenticators or administrator-approved recovery), DNS control is
not equivalent to account takeover, and this Trust Method makes DNS
control a direct route to asserting identities in the namespace.

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
differ from the canonical DNS-first lookup of
`domain_authorized_issuer`:

- DNS-record attacks ({{dns-integrity-and-compromise}}) can create
  or remove the opt-in, or publish a `key=` that makes retrieval fail
  (Indeterminate), but cannot choose the policy or its host: the
  opt-in record's `uri=` is the default URL, and its `issuer=` values
  are not used.
- A domain that lets untrusted users claim subdomains could lose
  the dedicated host name to an attacker ({{RFC8461}} Section 10.3).
  Only a record whose `uri=` is the default URL opts in, so a domain
  that publishes only an inline record, or a pointer to another
  host, is not exposed; a domain that opts in has set up the
  dedicated host itself ({{dedicated-policy-host}}).
- Resolution of the dedicated host still uses DNS. A DNS redirect
  of that host combined with TLS misissuance substitutes the
  policy; CAA records and Certificate Transparency monitoring are
  the primary defenses ({{transport-integrity}}).
- The policy is served from the Subject Authority's dedicated
  host, so the risk is that of whoever operates that host
  ({{third-party-policy-hosts}}).
- Because the dedicated host is a separate name, a Subject
  Authority whose apex is hosted on a marketing site or CDN it does
  not control can still participate, by provisioning or delegating
  that name.

The trade-off between the two lookup modes is discussed in
{{rationale-https-only}}.

## DNS Integrity and Compromise {#dns-integrity-and-compromise}

An adversary who can substitute a forged DNS response (off-path
resolver spoofing, authoritative nameserver hijack, registrar
account compromise, BGP hijack, recursive cache poisoning) can
substitute the Subject Authority's policy: add an attacker-
controlled Assertion Issuer to the inline form, redirect a `uri=`
pointer, or force `indeterminate` outcomes to benefit a cached
attacker-friendly policy. The pointer form additionally depends on
TLS authentication of the pointed-at host: TLS does not mitigate
the DNS compromise that selected that host.

Absent DNSSEC or an authenticated resolver path, the inline DNS form's
integrity is no stronger than the recursive resolver path between
consumer and authoritative server. Deployments needing a stronger
guarantee SHOULD sign the zone with DNSSEC or use the HTTPS document
form with the controls of {{TRUST-FRAMEWORK}} §Shared Infrastructure
and Hosted Well-Known Paths; the inline form's "common case"
simplicity ({{publication-profiles}}) is an operability tradeoff, not
an integrity guarantee.

Required framework defenses:

- Consumers MUST discard records whose `authority=` does not match
  the queried Subject Authority. This is the wildcard mitigation:
  a permissive parent-zone wildcard cannot authorize subdomains
  because consumers reject records lacking the matching
  `authority=` directive.
- Consumers MUST verify TLS for any host named by `uri=`.
- Consumers MUST bound cache lifetimes ({{dii-caching}}). A spoofed
  answer that was not DNSSEC-validated stays fresh for at most 1
  hour, and can be served for at most 1 more hour while the live
  channel is Indeterminate.
- Consumers performing DNSSEC validation MUST treat validation
  failure as `indeterminate`, not `negative-authoritative`, so that a
  stripped or broken signature produces a retryable failure rather
  than a cacheable Negative.
- Subject Authorities SHOULD sign `_oauth-issuer-policy.{A}` with
  DNSSEC, and consumers that do not validate DNSSEC SHOULD use a
  trustworthy resolver path (DoH/DoT to a vetted resolver).

Forged negative answers: a `negative-authoritative` DNS result is
Negative ({{dii-lookup}}). When `domain_authorized_issuer` is the only
`subject_namespace_authorization` method the Trust Policy lists, an
attacker who spoofs one can deny service for the namespace but cannot
substitute a policy, and because Negative caching is capped
({{dii-caching}}) the denial ends soon after the spoofing does. When
the Trust Policy also lists another namespace method, a Negative lets
that method supply evidence ({{TRUST-FRAMEWORK}} §Multiple Authority
Sources Within a Category). A spoofed negative answer can then
suppress a published policy that does not authorize the issuer and
let the other method authorize it. In that configuration a consumer
protects the Subject Authority's denial only if it authenticates the
absence, for example by validating DNSSEC. Signing the zone does not
help a consumer that accepts unvalidated negative answers, and the
cache cap limits reuse of one forged answer, not an attacker who
keeps forging fresh ones.

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
§Shared Infrastructure and Hosted Well-Known Paths. DAI-specific
points:

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
- A Subject Authority whose policy host is shared infrastructure, or
  is operated by a provider, can publish a `key=` thumbprint
  ({{dii-dns-record}}) so that the host cannot alter the policy. The
  host can still withhold the policy, and can replay an older signed
  policy whose `exp` has not passed to a consumer that holds no
  cached copy, since the `iat` check of {{signed-policy}} needs one.
  A short `exp` bounds that replay; removing a hostile host takes a
  `key=` rotation or a new `uri=`.

A Subject Authority that hosts its Issuer Authorization Policy on
shared infrastructure it does not control end to end SHOULD publish
`signed_policy` with a signing key held outside that infrastructure
and resolvable through a channel independent of it, such as a `key=`
thumbprint ({{signed-policy}}).

Because the `crit` member is itself carried in the unsigned document,
an attacker who can strip `signed_policy` can strip `crit` with it;
publisher-side criticality therefore does not defend against
stripping by an on-path or edge attacker. A signature is effective
against such an attacker only if consumers require it, through local
configuration or a `key=` directive. A consumer configured to require
`signed_policy` for a Subject Authority MUST verify it before acting
on that Subject Authority's policy, MUST reject a policy whose
signature is missing or invalid, and MUST NOT treat a valid TLS
connection to a shared edge as sufficient by itself.

## Policy Conflicts and Determinism {#policy-conflicts}

The lookup procedure ({{dii-lookup}}) is deterministic across the
common conflict scenarios that arise when multiple records or
sources coexist. Determinism is a security property: two verifiers
receiving the same DNS and HTTPS responses, using the same Public
Suffix List snapshot ({{TRUST-FRAMEWORK}} §Public Suffix List
Versioning), and within the limits of
{{https-policy-document-contract}}, MUST reach the same conclusion
about what (if any) policy applies. An attacker with
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
  `valid_until`, or explicit denial (an empty `authorized_issuers`
  array, {{dii-document}}) MUST use the HTTPS or DNS pointer form;
  a recognized inline record with no `issuer=` is malformed, so the
  inline form cannot publish an empty delegation set. An inline
  record that names a shared multi-tenant issuer authorizes none of
  its tenants under a grant profile that carries `tenant`
  ({{dii-multi-tenant}}).

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
in the Shared Issuer Variant of the End-to-End Example. The
following security points apply:

- **Shared issuers are authorized per tenant.** Under a grant
  profile that carries `tenant`, an entry without `tenant` matches
  only assertions that carry no `tenant` claim ({{dii-verification}}),
  so listing a shared issuer without `tenant`, including in the inline
  DNS form, authorizes none of its tenants; a Subject Authority
  authorizes a tenant by listing the
  (issuer, tenant) pair. This relies on the Identity Provider
  sending the `tenant` claim whenever it is multi-tenant, which
  {{TRUST-FRAMEWORK}} §ID-JAG requires of every shared issuer. A shared
  issuer that omits the claim anyway would again match an entry
  without `tenant`, so Subject Authorities SHOULD list a shared
  issuer only with `tenant`.

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

- **Grant profiles without a `tenant` claim.** Under a grant
  profile that carries no `tenant` claim (e.g., the generic
  JWT-bearer grant, {{TRUST-FRAMEWORK}} §Generic JWT-Bearer
  Assertion Grant), only entries that omit `tenant` can match, so
  tenant-scoped authorization is not expressible. Any `tenant` claim
  in such an assertion is ignored for matching (it is not attested
  by the grant profile), so an entry without `tenant` for a shared
  issuer matches assertions from every one of its tenants; the
  default above does not help here. A Subject
  Authority relying on a shared multi-tenant Assertion Issuer
  SHOULD NOT authorize that issuer for such a grant unless
  per-tenant issuer identifiers are used.

## Denial of Service, Amplification, and SSRF {#dos-ssrf}

The `domain_authorized_issuer` lookup can be triggered before the
Assertion Issuer is known to be trustworthy: the assertion signature
validates against the issuer's own key, which an attacker operating
their own authorization server controls. An attacker can therefore
drive lookups at will, creating three risks:

- **Reflection/amplification and cache exhaustion.** A flood of
  assertions carrying distinct Subject Authorities
  (`email: x@{random}.example`) makes the Resource Authorization
  Server issue a live DNS query per distinct authority (and an HTTPS
  fetch when the record names a `uri=` target), turning it into a
  request amplifier
  aimed at third-party DNS/HTTPS infrastructure and filling its own
  cache with distinct-authority entries. Consumers MUST bound lookup
  work: enforce per-Subject-Authority and global rate limits and
  bound concurrent outstanding lookups. Consumers SHOULD cache
  Negative and Indeterminate outcomes per {{dii-caching}}, and MAY
  impose a maximum number of distinct-authority lookups per unit
  time, shedding load by treating excess as Indeterminate
  (fail-closed).
- **Server-Side Request Forgery.** Every HTTPS policy fetch resolves
  a host the attacker may control: the `uri=` target named by a
  Subject Authority under the attacker's control, or the dedicated
  host under the HTTPS-only mode (the attacker presents an assertion
  whose namespace is a domain it owns, then points the host's A/AAAA
  records at an internal address). The fetch occurs regardless of
  whether the body validates, enabling blind internal probing
  through timing and error differentials. Every such fetch follows
  {{TRUST-FRAMEWORK}} §Outbound Fetches, which forbids connecting to
  addresses that are not globally routable and requires timeouts and
  size bounds. Because redirects are not followed
  ({{dii-https-url}}), the target is always the host the lookup
  named.
- **Cost asymmetry.** A single small assertion can cause a full
  DNS+HTTPS round trip. Consumers SHOULD prefer cached results and
  SHOULD NOT perform a live lookup until the client is authenticated
  and the assertion has passed the cheaper grant-profile validation
  checks (signature, audience, expiry; {{TRUST-FRAMEWORK}}
  §Resource Authorization Server Processing).

# Privacy Considerations {#privacy}

The lookup is verifier-side and per-verification, so it leaks metadata
in two directions, and the policy itself is public:

- **To the resolver path.** A DNS query for
  `_oauth-issuer-policy.{A}` exposes the Subject Authority `{A}` being
  evaluated (and its timing) to every resolver and on-path observer.
  It does not carry the full subject identifier for formats such as
  `email`, but the authority plus timing can reveal organizational
  relationships and login activity. Resource Authorization Servers
  SHOULD use a privacy-preserving resolver path (DoH/DoT to a vetted
  resolver) and SHOULD NOT perform the lookup until it is needed for a
  concrete verification decision.
- **To the Subject Authority.** For the dedicated-host and `uri=`
  channels, the Subject Authority's own server (or its chosen policy
  host) observes the Resource Authorization Server's egress IP address
  and the timing of each fetch, learning which Resource Authorization
  Servers its namespace's users are authenticating to; the
  authoritative DNS operator for `{A}` observes per-lookup timing over
  DNS. An employer acting as its own Subject Authority can thereby
  obtain a near-real-time signal of where employees sign in. Caching
  ({{dii-caching}}) is the primary mitigation: it coarsens timing and
  collapses repeated lookups, so consumers SHOULD cache to the bounds
  permitted rather than re-fetching per verification. The minimum
  freshness lifetime of {{dii-caching}} keeps a Subject Authority
  from observing every sign-in by publishing a very short TTL.
- **Policy contents.** A published policy is public. It reveals the
  Subject Authority's Identity Providers, tenant identifiers, and,
  through `valid_until`, when its contracts end. Subject Authorities
  that consider these sensitive publish only what verification needs.

# Operational Considerations {#operational}

Publishing a DAI record makes DNS/HTTPS control a real-time input to
sign-in authorization: a lapse in the record or its host can stop
legitimate assertions from being accepted, and an unauthorized change
can authorize an attacker. Subject Authorities SHOULD operate the
record and any policy host with the same rigor as other sign-in-path
infrastructure. Specific guidance:

- **Rollout.** A first publication withdraws no acceptance that this
  Trust Method granted, since a namespace without a policy is
  Negative ({{dii-failures}}). At a Resource Authorization Server
  that also lists another namespace method, however, the new policy
  becomes final for the namespace ({{TRUST-FRAMEWORK}} §Multiple
  Authority Sources Within a Category), so it needs to list every
  issuer that method accepted. Check the issuer list for
  completeness before publishing; forgotten
  regional tenants and departing Identity Providers are the usual
  omissions, and they surface as rejections at such servers. Subject
  Authorities SHOULD sign the zone with DNSSEC or publish via the
  HTTPS document form, since every published decision carries the
  full weight of the publication channel's integrity
  ({{dns-integrity-and-compromise}}).
- **Change management and TTLs.** Reduce DNS TTLs in advance of any
  planned change to the record or `uri=` pointer ({{third-party-policy-hosts}}),
  and choose steady-state TTLs balancing propagation speed against
  resolver load and the consumer cache bounds of {{dii-caching}}.
- **Rotation.** To rotate an authorized issuer, publish the new
  `authorized_issuers` entry alongside the old one and remove the old
  entry only after the old issuer is decommissioned and caches have
  expired; overlapping validity windows (`valid_from`/`valid_until`)
  make the transition observable and bounded.
- **Signing-key rollover.** To roll over a `key=`-pinned signing
  key, publish the new thumbprint beside the old one, re-sign the
  policy with the new key, and remove the old thumbprint once caches
  have expired.
- **Withdrawal.** To withdraw every authorization, remove the TXT
  record (Negative) or publish an empty `authorized_issuers` array
  (explicit denial). Deleting the policy document instead makes
  lookups Indeterminate, so consumers can keep using a cached policy
  within the stale-if-error bound of {{dii-caching}}.
- **Monitoring.** Monitor the record set and any HTTPS policy document
  for unexpected changes, and perform cross-region/cross-resolver
  checks to detect localized substitution ({{dns-integrity-and-compromise}}).
- **Availability.** Because Negative and Indeterminate outcomes fail
  closed ({{dii-failures}}), loss of the record or policy host blocks
  sign-in for the namespace; provision the authoritative DNS and any
  policy host for availability accordingly.
- **Zone hygiene.** Avoid wildcard TXT records in zones participating
  in this mechanism ({{dns-integrity-and-compromise}}); a wildcard
  with a non-matching `authority=` causes recognized-but-discarded
  records and can push a lookup to Indeterminate.
- **Media type.** Serve the policy document with
  `Content-Type: application/json`. Static hosts often label a file
  without an extension `application/octet-stream`, which consumers
  treat as Indeterminate ({{dii-failures}}).
- **Stable issuer identifiers.** Issuer comparison is octet-for-octet
  ({{dii-document}}), so an Identity Provider operator that changes an
  issuer identifier breaks every Subject Authority that lists it
  until each republishes. Identity Provider operators keep issuer
  identifiers stable and announce changes ahead of time.
- **Many domains.** Each registrable domain is its own Subject
  Authority with its own policy (`subject_authority` names exactly
  one). An organization with many domains publishes a record, and
  for the pointer form a document, for each.
- **Users from other namespaces.** An assertion about a user whose
  email is in another organization's namespace (for example, a guest
  at `partner.example` signing in through the host organization's
  Identity Provider) is accepted only if `partner.example` authorizes
  that Identity Provider. This follows from namespace authorization
  and is not a defect.
- **Consumer mail domains.** This Trust Method is designed for
  organizational namespaces. A consumer mail provider is unlikely to
  authorize the Identity Providers its users sign in with elsewhere,
  so under a Trust Policy whose only namespace method is
  `domain_authorized_issuer`, assertions about those users are
  rejected.

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

The suffix names the document rather than the consuming framework;
see {{rationale-generic-name}}.

## Underscored DNS Node Name

Registers the following entry in the IANA "Underscored and Globally
Scoped DNS Node Names" registry per {{RFC8552}} and {{RFC8553}}, for
use by this document:

RR Type:
: TXT

_NODE NAME:
: `_oauth-issuer-policy`

Reference:
: This document, {{dii-dns-record}}


## Trust Method Registrations

This document registers the following entry in the Identity
Assertion Issuer Trust Methods registry
({{TRUST-FRAMEWORK}} §Identity Assertion Issuer Trust Methods Registry):

Identifier:
: `domain_authorized_issuer`

Categories:
: `subject_namespace_authorization`

Parameters:
: `lookup` (string, OPTIONAL; the value `https_only` selects the
  HTTPS-only lookup mode, {{trust-method-https-authorized-issuer}})

Change Controller:
: IETF

Reference:
: This document

## Issuer Authorization Policy Directives Registry {#iana-dii-directives}

IANA is requested to establish a new registry titled "OAuth Issuer
Authorization Policy DNS Directives" under the "OAuth Parameters"
registry group, for the `name=value` directives carried in the DNS
TXT record ({{dii-dns-record}}).

Registration policy: Specification Required {{RFC8126}}.

Each entry contains a Directive Name (character set `[a-z0-9_-]`), a
Description, a Change Controller, and a Reference. Designated Expert
instructions: the expert verifies the directive name is unique, its
value syntax is specified within the ABNF value production of
{{dii-dns-record}} (ASCII, no `;`, no whitespace), and its
multiplicity and duplicate-handling rules are stated. A directive
that narrows what a record authorizes, so that a consumer ignoring
it would accept more than the Subject Authority intended, MUST NOT
be registered unless its specification also defines a new version
token ({{dii-dns-record}}), so that consumers that do not implement
it ignore the record rather than the directive.

Initial entries:

| Directive Name | Description | Change Controller | Reference |
|-|-|-|-|
| `v` | Version token; MUST appear first | IETF | This document |
| `authority` | Subject Authority this record binds (A-label) | IETF | This document |
| `uri` | HTTPS URL of an Issuer Authorization Policy document | IETF | This document |
| `key` | Thumbprint of the `signed_policy` signing key (pointer records only) | IETF | This document |
| `issuer` | An authorized Assertion Issuer identifier | IETF | This document |

## Issuer Authorization Policy Members Registry {#iana-dii-members}

IANA is requested to establish a new registry titled "OAuth Issuer
Authorization Policy Members" under the "OAuth Parameters" registry
group, for the JSON members of the Issuer Authorization Policy
document ({{dii-document}}).

Registration policy: Specification Required {{RFC8126}}.

Each entry contains a Member Name, a Description, whether the member
is decision-affecting, a Change Controller, and a Reference. Designated Expert instructions: the expert verifies
the member name does not collide with an existing member, its JSON
type and semantics are specified, the registration states whether
the member is decision-affecting ({{TRUST-FRAMEWORK}}
§Terminology), and any decision-affecting member
states how a consumer that does not recognize it behaves (the default
is to ignore unrecognized members; a member requiring fail-closed
handling uses the `crit` mechanism of {{TRUST-FRAMEWORK}} §Critical
Members). A member that narrows what a policy authorizes, so that a
consumer ignoring it would accept more than the Subject Authority
intended, MUST NOT be registered unless publishers can list it, or a
top-level member its specification defines alongside it, in `crit`;
`crit` does not reach members of `authorized_issuers` entries.

Initial entries; all are decision-affecting except `last_updated` and
`signed_policy`:

| Member Name | Description | Change Controller | Reference |
|-|-|-|-|
| `subject_authority` | Subject Authority this policy applies to | IETF | This document |
| `authorized_issuers` | Array of authorized issuer objects | IETF | This document |
| `issuer` | Authorized Assertion Issuer identifier (within an entry) | IETF | This document |
| `tenant` | Issuer-side tenant identifier (within an entry) | IETF | This document |
| `subject_identifier_formats` | Permitted Subject Identifier formats (within an entry) | IETF | This document |
| `valid_from` | Delegation start time (within an entry) | IETF | This document |
| `valid_until` | Delegation end time (within an entry) | IETF | This document |
| `last_updated` | Policy publication time | IETF | This document |
| `signed_policy` | Signed JWT of the policy members | IETF | This document |
| `crit` | Names decision-affecting members a consumer MUST understand or reject the document | IETF | This document; {{TRUST-FRAMEWORK}} §Critical Members |

## Media Type Registration {#iana-dii-media-type}

IANA is requested to register the following media type in the "Media
Types" registry for the signed Issuer Authorization Policy
({{signed-policy}}). Following {{RFC8725}} §3.11, the JWT `typ`
header value is the media subtype with the `application/` prefix
omitted (`issuer-authorization-policy+jwt`), as required in
{{signed-policy}}.

Type name:
: `application`

Subtype name:
: `issuer-authorization-policy+jwt`

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: 8bit; the value is a JWT in JWS Compact Serialization, a sequence
  of base64url-encoded values separated by periods ({{RFC7519}}
  Section 10.3.1).

Security considerations:
: See {{signed-policy}} and {{dii-security}} of this document.

Interoperability considerations:
: N/A

Published specification:
: This document ({{signed-policy}})

Applications that use this media type:
: Subject Authorities that sign Issuer Authorization Policies, and
  Resource Authorization Servers that verify them

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A;
  File extension(s): N/A; Macintosh file type code(s): N/A

Person and email address to contact for further information:
: Karl McGuinness, public@karlmcguinness.com

Intended usage:
: COMMON

Restrictions on usage:
: none

Author:
: Karl McGuinness

Change controller:
: IETF

--- back

# Design Rationale

This appendix is non-normative.

## Following Existing DNS Authority Patterns {#dns-authority-patterns}

The Domain-Authorized Issuer Trust Method applies the same
authority-publication pattern that domain owners already use for
CAA {{RFC8659}}, MTA-STS {{RFC8461}}, SPF, DKIM, DMARC, and the Email
Verification Protocol {{I-D.hardt-email-verification}}. {{TRUST-FRAMEWORK}}
§Authority Delegation Model covers the abstract pattern; this
document chooses DNS at `_oauth-issuer-policy.{domain}` as the
authoritative publication channel. The `name=value` record syntax is
closest to DMARC's, and DMARC's operational experience with the
organizational-domain boundary informs the Subject Authority
Determination approach in {{TRUST-FRAMEWORK}} §Subject Authority
Determination. DMARC itself has since replaced its use of the Public
Suffix List with a DNS tree walk ({{RFC9989}}); see the Public Suffix
List discussion there.

Unlike CAA (deployment-time), SPF/DKIM (spam-score signal), and
MTA-STS (inbound mail), the `_oauth-issuer-policy` record is
consumed during user sign-in; the operational consequences of that
are covered normatively in {{operational}}.

Relationship to issuer discovery. WebFinger {{RFC7033}} and OpenID
Connect Discovery answer a different question: given a user
identifier, where does a client go to authenticate the user? This
Trust Method is verifier-side and authorization-oriented: given an
assertion already in hand, is its issuer authorized for the subject's
namespace? DAI is published per namespace (not per user), over a DNS
channel whose control establishes the authority binding, and it
carries authorization semantics (validity windows, tenant binding,
format restrictions) that a discovery record does not. An earlier
proposal, {{I-D.sanz-openid-dns-discovery}}, published a domain's
OpenID issuer in a DNS TXT record of similar shape, also for
discovery. A deployment could layer client-side discovery on top
(see {{assertion-issuer-discovery-client-side}}), but that is out of
scope here.

Relationship to federation-scoped and bilateral mechanisms. SAML
federations such as InCommon constrain the namespaces an Identity
Provider may assert with the scope metadata extension {{SHIBMD}}, and
interfederation services such as eduGAIN {{EDUGAIN}} carry that
metadata between federations. There, scope is attested by the
federation operator and distributed in trusted federation metadata;
DAI's authorization is published by the namespace owner, where any
verifier can retrieve it. Software-as-a-service providers commonly
have a customer prove control of its domain with a one-time DNS
challenge and then configure the customer's Identity Provider.
FastFed {{FASTFED}} automates establishing and maintaining such a
federation relationship between an Identity Provider and an
application provider, with administrator approval on both sides.
Both are bilateral configuration, held by the parties to one
relationship, not an authorization that any verifier can retrieve.

Relationship to the Email Verification Protocol. EVP
{{I-D.hardt-email-verification}} also publishes, in DNS, the issuer
for an email domain, and its relying party also checks that record at
verification time: it resolves `_email-verification.{domain}` itself
and rejects a token whose `iss` does not match. The two records
differ in role and in scope. In role, EVP names the issuer that
verifies *control of an email address*, while DAI names the issuers
*authorized to assert identities* in a namespace; a domain that uses
one provider for mail and another for single sign-on can rightly
give different answers to the two questions. In scope, EVP allows
exactly one issuer, identified by an HTTPS origin with no path, for
the raw email domain and for EVP tokens only. DAI allows several
issuers with full issuer identifiers (including path components),
binds them to tenants, carries authorization constraints, and
normalizes to the registrable domain. Convergence with EVP on a
shared record or node name is possible future work;
{{email-verification-protocol-bridge}} sketches a bridge.

## Why a Generic Record Name {#rationale-generic-name}

The DNS node name, well-known URI suffix, and version token name the
document they locate (the Issuer Authorization Policy), not the
identity framework consuming it. The wire format is
subject-class-agnostic: no member or directive is specific to email
or identity assertions; the subject class is expressed through
`subject_identifier_formats` ({{RFC9493}}), and the identity-specific
machinery lives in {{TRUST-FRAMEWORK}}, whose own well-known URI is
identity-scoped for that reason. A future Subject Identifier format
with a DNS-publishable Subject Authority reuses
`domain_authorized_issuer` and this same record
({{TRUST-FRAMEWORK}} §Future Extensions); an identity-scoped name
would force a second record at a second DNS name. One per-domain
policy surface, extended through registered members rather than new
names, follows the `oauth-authorization-server` metadata precedent.

## Why First-Class Tenant Binding

Shared-issuer multi-tenant Identity Providers (Google Workspace,
Auth0 in some configurations, Microsoft Entra B2B in some flows)
serve many customer tenants under a single issuer URL. The
deployment reality is that these Identity Providers are common;
authorizing them without tenant binding effectively authorizes
every tenant of the Identity Provider for the namespace, which is
almost never the intent. An entry without `tenant` therefore
matches only assertions that carry no `tenant` claim.

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
HTTPS-only lookup is resilient to substitution of the TXT record's
issuers or policy location but depends on
resolution of the dedicated host plus the public CA trust system; a
DNS redirect combined with TLS misissuance defeats it.
`domain_authorized_issuer` in the inline form is resilient to TLS
misissuance because the authority artifact is the TXT record itself;
an attacker needs DNS-write or DNSSEC-bypass capability. The DNS
pointer form inherits TLS-misissuance risk on the pointed-at host
while also depending on DNS for selection.

A Subject Authority with strong DNSSEC and weak TLS-issuance
controls favors the inline DNS form. A Subject Authority with
strong CAA/CT monitoring that prefers to keep its issuer list out of
DNS favors the dedicated host and HTTPS-only lookup.

## Why a DNS Opt-In and a Dedicated Policy Host {#rationale-opt-in}

Fetching a well-known URL on the Subject Authority's apex whenever
DNS reports no record would make every domain DAI-covered unless it
opted out. An attacker who could serve content on a domain's apex (a
dangling address record, a site builder that lets customers place
files at arbitrary paths, or a takeover of a host the apex redirects
to) could publish a policy for a domain that never adopted DAI, with
no DNS forgery at all.

This document follows MTA-STS {{RFC8461}} instead. The TXT record is
the opt-in in both lookup modes, and for the HTTPS-only lookup mode
its `uri=` names the dedicated host's default URL; a policy for that
mode lives on the dedicated host; and redirects are not followed. A
host name alone would not be an opt-in, since domains that let users
claim subdomains could lose it ({{RFC8461}} Section 10.3). The cost
is that a Subject Authority with no control of its DNS cannot
participate. The full JSON policy remains available to every Subject
Authority through the `uri=` pointer.


# Future Extensions {#future-extensions}

This appendix is non-normative. It sketches features intentionally
deferred from this document; future specifications may register them.

## Federation-Bound Issuer Authorization Policy

For deployments where a Subject Authority is a federation Entity,
a future extension could authenticate the Issuer Authorization Policy
on the dedicated policy host through a digest in the Entity
Configuration of the Entity that serves it, using a mechanism such
as the proposed Well-Known Binding specification {{OIDF-WKB}}. The
Subject Authority would still author and publish the policy under
`{A}`; federation would add integrity protection, not transfer
namespace authority to a trust anchor. The added protection depends
on federation enrollment and key control being independent of the
domain publication channel.

Such an extension would need to define all of the following:

- Coverage of the HTTPS-only lookup mode. Inline DNS TXT policies
  would not be covered; a `uri=` pointer would be covered only if it
  names the default URL.
- An exact mapping from Subject Authority `{A}` to Entity Identifier
  `https://oauth-issuer-policy.{A}`, the dedicated host, without a
  path or explicit port. A binding keyed to an Entity's own
  well-known URIs ({{OIDF-WKB}}) covers documents only on the Entity
  Identifier's host, and the fixed label keeps any other subdomain
  Entity from speaking for the registrable domain. {{OIDF-WKB}}
  requires such a mapping from any protocol whose peer identifier is
  not an `https` URL.
- A separate trust-anchor parameter on `domain_authorized_issuer`,
  independent of any `openid_federation.trust_anchors` configuration
  for the issuer-authentication category.
- Indeterminate outcomes for a missing binding, digest mismatch, or
  trust-chain failure, with no fallback to an unbound policy.
- A cache lifetime bounded by the minimum of this document's caching
  limits, the Entity Configuration's `exp`, and the trust chain's
  expiry.
- Revocation handling: overlapping old and new digests lets an origin
  attacker replay a policy that still authorizes a revoked issuer.
  Emergency revocations should omit that overlap, as {{OIDF-WKB}}
  specifies for any replacement that is a revocation, while
  accounting for residual exposure from cached Entity Configurations.

This document does not define that extension or change DAI's lookup,
authority binding, or integrity mechanisms to depend on federation.

## Critical Directives for the DNS Record Form {#crit-dns-form}

The JSON document carries a `crit` member ({{dii-document}}), so an
extension that adds a decision-affecting top-level member to the
Issuer Authorization Policy can mark it critical and have
already-deployed consumers honor it. `crit` does not reach members
of `authorized_issuers` entries, so an extension that narrows
entries (for example, a `permitted_audiences` member) also defines a
top-level member for publishers to list in `crit`
({{iana-dii-members}}). The DNS record form has no analogous
per-directive criticality mechanism today; its version token
({{dii-dns-record}}) prevents misinterpretation of incompatible future
syntax by making unrecognized versions ignored, so a Subject Authority
that publishes only an unrecognized version is Negative to an older
consumer. A future extension that needs true per-directive
fail-closed semantics in the DNS form would define a `crit=` directive
and its recognition rules at that time.

## Evaluation Reports

A Subject Authority learns which assertions Resource Authorization
Servers reject for its namespace only out of band
({{observe-before-enforce}}). A future extension can define an
aggregate reporting mechanism, analogous to DMARC aggregate reports
{{RFC9990}}: a policy member naming a reporting endpoint, a report
format (observed issuers, match and mismatch counts, time window),
and delivery requirements. It is deferred because report formats and transport
carry privacy and abuse considerations (a reporting endpoint learns
which Resource Authorization Servers a namespace's users sign in to,
concentrating the metadata discussed in {{privacy}}) that deserve
their own document.

## Audience-Scoped Delegations

An `authorized_issuers` entry authorizes an issuer for a namespace
without constraining which Resource Authorization Servers may accept
the resulting assertions; a compromised-but-listed issuer can assert
the namespace's users to any consumer ({{TRUST-FRAMEWORK}} §Scope of
Namespace Authorization). A future `permitted_audiences` member on
entries would let a Subject Authority bound that blast radius by
enumerating or pattern-matching acceptable audiences. It is deferred
because audience identifiers are grant-profile-specific and an
enumerable audience set does not exist for the open-world deployments
this mechanism targets; a workable design likely needs audience
patterns and an interaction rule with the assertion's `aud` claim.

## Email Verification Protocol Bridge {#email-verification-protocol-bridge}

The Email Verification Protocol {{I-D.hardt-email-verification}}
defines a DNS TXT record at `_email-verification.{domain}` whose
`iss=` value names an authorized issuer for the namespace, using
a bare hostname rather than a full HTTPS issuer identifier. A
future Trust Method (provisionally `email_verification_dns`)
could let a Resource Authorization Server honor those records
without requiring the Subject Authority to also publish an
`_oauth-issuer-policy` record.

The bridge is deferred because it forces the reader to learn a
second record format, a different issuer-identifier shape
(bare-origin only, no path component), and a different
email-domain semantics (the Email Verification Protocol parses
the email's raw domain part, whereas this document normalizes to
the registrable domain via the Public Suffix List). It is also
deferred because {{I-D.hardt-email-verification}} is progressing
on its own timeline independent of this document. Deployments wanting the bridge can either publish
both records (an `_oauth-issuer-policy` record satisfying
`domain_authorized_issuer` plus their existing
`_email-verification` record for other consumers) or wait for the
future Trust Method specification.

## Assertion Issuer Discovery (Client-Side) {#assertion-issuer-discovery-client-side}

The same DAI records that let a Resource Authorization Server
verify an assertion can also let a client discover which Assertion
Issuer is authoritative for a subject identifier's namespace
before any assertion exists: given `alice@acme.example`, a client
can query `_oauth-issuer-policy.acme.example`, retrieve the Issuer
Authorization Policy, and resolve an authorized issuer's
authorization server metadata to find its token endpoint (entry
order carries no semantics, so issuer selection would need to be
specified by the profiling document).

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
_oauth-issuer-policy.acme.example. IN TXT ( "v=oauth-issuer-policy1;"
    "authority=acme.example;"
    "issuer=https://idp.example.net" )
~~~

The quoted segments are concatenated without a separator, yielding
`v=oauth-issuer-policy1;authority=acme.example;issuer=https://idp.example.net`.
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
   &client_assertion_type=
   urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=eyJhbGciOiJFUzI1NiIs...
   ~~~

   Line breaks in the request body are for display only.

## Verification (Resource Authorization Server Side)

3. The Resource Authorization Server validates `private_key_jwt`
   client authentication, then the ID-JAG: signature (via
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
      the single entry's `issuer` value. Verification succeeds.

5. The Resource Authorization Server issues an access token in the
   response body.

## Migration Variant: Pointer Form

If `acme.example` later wants to express validity windows, format
restrictions, or tenant binding, it can switch to the pointer form
without changing any consumer behavior:

~~~
_oauth-issuer-policy.acme.example. IN TXT ( "v=oauth-issuer-policy1;"
    "authority=acme.example;"
    "uri=https://oauth-issuer-policy.acme.example"
    "/.well-known/oauth-issuer-policy" )
~~~

and publish the richer JSON document at that URL, the default URL on
its dedicated policy host, which also serves Resource Authorization
Servers that use the HTTPS-only lookup mode:

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
  Authorization Server consults no HTTPS location; it returns
  `invalid_grant`.

- If `acme.example` published no TXT record, the lookup would be
  Negative and the assertion rejected, even if a document existed at
  the default URL on `oauth-issuer-policy.acme.example`: the TXT
  record is the opt-in.

- A wildcard record at `*.example` covering `acme.example` would
  have to carry `authority=acme.example` to be accepted. A wildcard
  with a different `authority=` is discarded; if no other recognized
  record remains, the response is `malformed` (Indeterminate) and
  the assertion is rejected.

- If `acme.example` rotates its authorized Assertion Issuer and the
  Resource Authorization Server has a cached virtual policy, the
  Resource Authorization Server may continue accepting assertions
  from the old issuer until its cache expires. Subject Authorities
  are encouraged to use short DNS TTLs during rotation; consumers
  enforce a local cache ceiling per {{dii-caching}}.

# Document History

This appendix is non-normative and will be removed before publication.

-01

  * Make the DNS TXT record an explicit opt-in: a namespace with no
    record is Negative, with no HTTPS fallback, in both lookup modes.
    The HTTPS-only lookup mode fetches from a dedicated host,
    `oauth-issuer-policy.{A}`, instead of the apex, and uses the TXT
    record only as the opt-in. Policy fetches no longer follow
    redirects.
  * Remove the `mode` member and `mode=` directive (monitor mode),
    which admitted more issuers than publishing nothing.
  * An entry without `tenant` no longer matches assertions that
    carry a `tenant` claim.
  * Add the `key=` directive, which pins the `signed_policy` signing
    key for a DNS pointer record.
  * Rewrite caching: an explicit stale-if-error bound, a one-hour cap
    on DNS results that are not DNSSEC-validated, and a one-hour cap
    on Negative caching.
  * Sketch a federation-bound Issuer Authorization Policy as a
    non-normative future extension.
  * Define signed-policy processing and register the
    `issuer-authorization-policy+jwt` media type in this document
    (moved from the framework); describe how a spoofed negative
    answer can suppress a published denial when another namespace
    method is configured.
  * Correct the relationship to the Email Verification Protocol,
    whose relying party also checks its record at verification time;
    add SAML scope metadata, bilateral domain verification, FastFed,
    and DNS-based OpenID discovery to the related mechanisms; bound
    the comparison between DNS control and control of email
    recovery; state that the Trust Method targets organizational
    namespaces.

-00

  * initial draft
