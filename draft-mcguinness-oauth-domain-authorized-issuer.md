---
title: "OAuth Domain-Authorized Issuer Trust Method"
abbrev: "Domain-Authorized Issuer Trust Method"
docname: draft-mcguinness-oauth-domain-authorized-issuer-latest
date: 2026-07-04
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
  RFC7519:
  RFC8414:
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
  RFC7033:
  RFC7489:
  RFC7523:
  RFC8126:
  RFC8461:
  RFC8659:
  RFC9728:
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  I-D.hardt-email-verification:
    title: "Email Verification Protocol"
    target: https://datatracker.ietf.org/doc/draft-hardt-email-verification/
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
verification, where no entry can match ({{dii-verification}}),
subject to `mode`. Consumers MUST NOT fall through to any other
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
  no tenant constraint applies. Grant profiles that do not carry a
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
  this Subject Authority.

  `valid_from`
  : OPTIONAL. {{RFC3339}} date-time. The delegation MUST NOT be treated
  as valid before this time. A consumer MAY apply a small clock-skew
  tolerance (≤5 minutes), consistent with JWT `nbf` conventions
  ({{RFC7519}} Section 4.1.5). This tolerance applies only to
  `valid_from`; it MUST NOT be applied to `valid_until` to extend a
  delegation past its stated end.

  `valid_until`
  : OPTIONAL. {{RFC3339}} date-time. The delegation MUST NOT be treated
  as valid at or after this time. No clock-skew tolerance is permitted
  on this bound.

`mode`
: OPTIONAL. String, either `enforce` or `monitor`; default `enforce`
when absent. In monitor mode the policy is provisional: consumers
evaluate it and log the outcome but do not reject assertions on the
basis of a mismatch ({{monitor-mode}}). A policy whose `mode` value
is any other string is malformed; an unrecognized future mode MUST
NOT be silently treated as either defined value.

`last_updated`
: OPTIONAL. {{RFC3339}} date-time at which the policy was last published.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, using the
  representation defined in {{TRUST-FRAMEWORK}} §Signed Policy Metadata. This member
  follows the signed metadata pattern used by {{RFC8414}} and
  {{RFC9728}}.
  Its presence does not by itself require all consumers to support
  signed policy processing; consumers apply the requirements in
  {{TRUST-FRAMEWORK}} §Signed Policy Metadata and any local policy
  that requires object-level integrity. A Subject Authority that needs
  to force signature processing lists `signed_policy` in `crit`.
  Per the key-resolution requirement of {{TRUST-FRAMEWORK}} §Signed
  Policy Metadata, the verification key for this profile MUST be
  resolved through one of: a key published under DNSSEC-signed
  records for the Subject Authority, or a key configured out of
  band at the consumer. Both channels are independent of the
  DNS/HTTPS path that carries the policy document; the trust
  assumption is, respectively, DNSSEC validation to the Subject
  Authority's zone, or the consumer's own key-provisioning process.
  This document does not define a DNS record format for key
  publication; deployments using the DNSSEC-published-key option do
  so via a mechanism agreed with their consumers until one is
  standardized.

`crit`
: OPTIONAL. Array of member names a consumer MUST understand to process
  the policy safely, as defined in {{TRUST-FRAMEWORK}} §Critical
  Members. A consumer that fails to recognize, or does not implement
  processing for, one or more listed members MUST treat the policy as
  malformed.

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
| `mode` | string, optional | exactly `enforce` or `monitor`; any other value is malformed |
| `signed_policy` | string, optional | signed JWT as defined in {{TRUST-FRAMEWORK}} §Signed Policy Metadata |
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

# Publication

A Subject Authority publishes the Issuer Authorization Policy
through one of the publication channels defined in
{{publication-profiles}}. Throughout this document, `{A}` denotes
the Subject Authority rendered as a DNS name (A-label form,
{{dii-dns-record}}). DNS publication uses the TXT record at
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
as the sole publication (Channel 3) or alongside a DNS record. A
Subject Authority with no DAI DNS record relies on the natural
`negative-authoritative` DNS response to bring consumers to the
well-known URL ({{dii-lookup}}).

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
  `=` characters are part of the value.
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

`mode=MODE`
: OPTIONAL. Either `enforce` or `monitor`; default `enforce` when
absent ({{monitor-mode}}). At most one `mode=` directive per record;
a second is malformed. If any remaining record after `authority=`
filtering carries `mode=`, all remaining records MUST carry `mode=`
with the same value; a mix of differing or partially present `mode=`
directives across records is malformed. A `mode=` value other than
the two defined values is malformed.

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
form for internationalized names). The URL uses the `https` scheme,
the host `{A}`, the default HTTPS port (443), and the well-known path
{{RFC8615}}; it has no userinfo, port, query, or fragment component. A
Subject Authority whose form has no host representation (none is
defined in this document beyond the `email`-derived DNS domain) cannot
use the default well-known channel. Consumers fetch this URL with an
HTTP GET over HTTPS with TLS server authentication and interpret the
response body as the JSON document defined in {{dii-document}}.

A URL obtained from a DNS `uri=` directive is fetched the same way;
the host serving the URL is responsible for TLS server authentication
of itself, not of `A`.

HTTP redirect handling is fixed so that two consumers reach the same
outcome and so that a redirect cannot silently move the authority
anchor to a host the Subject Authority does not control:

- Consumers MUST follow HTTPS redirects up to a limit of 5 hops; a
  request that would exceed 5 hops, or that loops, is classified as
  Indeterminate ({{dii-failures}}).
- Every redirect target MUST use the `https` scheme and MUST NOT
  contain a fragment component.
- Every redirect target MUST remain within the same registrable
  domain as the initial fetch host (the Subject Authority `{A}` for
  the default well-known channel; the `uri=` pointer's host for the
  pointer channel), where the registrable domain is computed with
  the Public Suffix List algorithm of {{TRUST-FRAMEWORK}} §Subject
  Authority Determination (ICANN and PRIVATE divisions) applied to
  the A-label host. A redirect to a different registrable domain
  MUST NOT be followed and is classified as Indeterminate;
  cross-registrable-domain policy hosting is expressed only through
  the `uri=` pointer, where the delegation is published in DNS and
  visible to the Subject Authority.
- The final response MUST have HTTP status 200 and a media type of
  `application/json` or any media type using the structured `+json`
  suffix (media type parameters ignored); any other final status is
  classified per {{dii-failures}}.

## HTTPS Policy Document Contract {#https-policy-document-contract}

The document retrieved from either the default well-known URL or a
DNS `uri=` pointer is the Issuer Authorization Policy defined in
{{dii-document}}; consumers MUST NOT interpret any other JSON shape
as an Issuer Authorization Policy. Consumers MUST validate the
complete policy as a unit; a malformed entry makes the whole policy
malformed, not just the offending entry.

Bounds (a response exceeding any bound is classified per
{{dii-failures}}): consumers MUST accept a policy document of at least
64 KiB and MAY reject one larger; publishers MUST keep the document
within 64 KiB. Consumers MUST accept at least 100 `authorized_issuers`
entries and MAY reject more. Consumers SHOULD apply an overall fetch
timeout and SHOULD limit JSON nesting depth (the defined document has a
fixed shallow structure). Consumers SHOULD send a conditional request
(for example, `If-None-Match`) when they hold a cached policy, treating
a 304 response per {{dii-failures}}.

The document's shape and members are defined, with an example, in
{{dii-document}}.

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
      mismatch, treat the response as `malformed`. (A `malformed`
      outcome is classified as Indeterminate, {{dii-failures}}.)

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
        Authorization Policy, including its `mode`. All `issuer=`
        and `mode=` directives across all records are ignored:
        their values do not contribute to the policy, but the
        directive-validity rules of {{dii-dns-record}} (including
        `mode=` consistency) were already applied in step 2a, so a
        record set malformed under those rules never reaches this
        step.

   c. Otherwise (no `uri=` present), construct a virtual Issuer
      Authorization Policy with `subject_authority` set to `A` and
      one entry in `authorized_issuers` for each distinct `issuer=`
      value across the remaining records. Entry order carries no
      semantics ({{dii-verification}}); the deduplicated values form
      a set. The virtual policy's `mode` is the common `mode=` value
      of the remaining records, or `enforce` when none carries
      `mode=` ({{dii-dns-record}}). Entries have no `tenant`,
      `subject_identifier_formats`, `valid_from`, or `valid_until`.
      The virtual policy is processed identically to one fetched over
      HTTPS, except that its cache lifetime is derived from DNS TTLs
      as described in {{dii-caching}}.

3. If the DNS response is `negative-authoritative`, fetch the policy
   from the default HTTPS well-known URL per {{dii-https-url}}.

4. If the DNS response is `indeterminate`, lookup has failed.
   Consumers MUST NOT fall back to HTTPS, since an attacker who
   suppresses DNS responses could otherwise force the consumer onto a
   path the attacker has compromised separately.

Two distinct situations remove DNS from the picture, with different
outcomes. A deployment whose Trust Policy selects the HTTPS-only
lookup mode ({{trust-method-https-authorized-issuer}}) never consults
DNS by design; note that a Subject Authority publishing only inline
DNS records (a common pattern) is unfindable under that mode. By
contrast, under the canonical DNS-first mode specified here, a
Resource Authorization Server that cannot resolve DNS (resolver
failure, untrusted resolution path) treats the lookup as
`indeterminate` and rejects the assertion, fail-closed; it MUST NOT
improvise a fallback to the HTTPS channel.

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
| Affirmative | A well-formed Issuer Authorization Policy was retrieved (inline DNS, DNS pointer + HTTPS fetch, or HTTPS well-known URL), its `subject_authority` matches `A`, and its structural validation succeeds; this includes a policy whose `authorized_issuers` array is empty (explicit denial, evaluated in {{dii-verification}}). HTTPS responses, when applicable, are 200 OK with a media type of `application/json` or a `+json`-suffixed type. A 304 (Not Modified) response to a conditional request validating a held cached policy within the absolute ceiling of {{dii-caching}} renews its freshness and is classified as the held policy's state; it does not reset the absolute cache-entry age. |
| Negative | DNS `negative-authoritative` followed by HTTPS 404 or 410 at the well-known URL; HTTPS 404 or 410 reached from a DNS `uri=` pointer; or, under the HTTPS-only lookup mode ({{trust-method-https-authorized-issuer}}), HTTPS 404 or 410 at the well-known URL. No policy is published at the location(s) the lookup consults. |
| Indeterminate | Any other outcome, fail-closed by default. See enumeration below. |

The Indeterminate state covers:

- **DNS-side**: SERVFAIL, REFUSED, timeout, truncation with no
  successful retry.
- **HTTPS transport**: TLS error, connection failure, redirect loop,
  non-HTTPS redirect target.
- **HTTPS response**: 5xx; 4xx other than 404 and 410 (for example
  401, 403, 405, 429, 451); 2xx other than 200; a 3xx that is not a
  followed redirect ({{dii-https-url}}) and not a 304 validating a
  held cached policy; unsupported media type; a body larger than the
  size limit, or containing more `authorized_issuers` entries than
  the consumer's entry limit ({{https-policy-document-contract}}).
- **DNS record validation**: `authority=` missing from any recognized
  record; all recognized records discarded for `authority=` mismatch;
  more than one `authority=` or `mode=` in a record; differing or
  partially present `mode=` values across remaining records; a
  `mode=` value other than `enforce` or `monitor`; a recognized
  record with neither `uri=` nor `issuer=`; multiple distinct `uri=`
  values; an empty or otherwise malformed directive.
- **HTTPS document validation**: body that is not a syntactically
  valid Issuer Authorization Policy; a `mode` value other than
  `enforce` or `monitor`; `subject_authority` that does
  not match `A`.

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

4. Determine whether any entry in `authorized_issuers` matches. An
   entry matches when ALL of the following hold:

   a. `issuer` equals the JWT `iss` claim under case-sensitive URL
      string comparison (the comparison rule fixed in
      {{dii-document}}).

   b. If the entry contains a `tenant` member, its value exactly
      matches the assertion's top-level `tenant` claim (in ID-JAG,
      {{ID-JAG}} §6.1) under case-sensitive string comparison. An
      entry with `tenant` does not match an assertion that lacks the
      `tenant` claim; an entry without `tenant` matches regardless of
      any `tenant` claim in the assertion. Under a grant profile that
      carries no `tenant` claim, any `tenant` claim physically present
      in the assertion MUST be ignored for entry matching.

   c. If the entry contains `subject_identifier_formats`, the Subject
      Identifier format of the assertion's subject is listed.

   d. If `valid_from` or `valid_until` are present, the current time
      is within the validity window (with the skew rules of
      {{dii-document}}).

   Under a policy whose `mode` is `enforce` (the default), the Trust
   Method is satisfied when step 3 succeeds and at least one entry
   matches under (a)-(d); under `monitor`, see {{monitor-mode}}.
   Entry order in `authorized_issuers` carries no semantics: the
   outcome is the boolean "does any entry match," so two consumers
   evaluating the same policy against the same assertion reach the
   same result regardless of array order or of which matching entry
   they examine first.

Steps 1 and 2 are prerequisites; their failure causes assertion
rejection per {{dii-failures}} unconditionally and is not classified
as a Trust Method satisfaction outcome. When the Trust Method is not
satisfied and, as a result, the cross-category combination rule
({{TRUST-FRAMEWORK}} §Cross-Category Combination Rule) is not met,
the Resource Authorization Server MUST reject the assertion with an
OAuth `invalid_grant` error.

## Monitor Mode {#monitor-mode}

A Subject Authority deploying its first policy cannot easily know
whether its issuer list is complete; an omission breaks sign-in for
its users. Monitor mode, following the deployment pattern of the
DMARC {{RFC7489}} `p=none` policy, lets the Subject Authority
publish, observe, and then enforce.

When the retrieved policy's `mode` is `monitor` ({{dii-document}}):

- The consumer MUST evaluate steps 3 and 4 normally, and SHOULD log
  every evaluation under the monitored policy (Subject Authority,
  assertion issuer, matched or mismatched, timestamp) so the Subject
  Authority can be informed out of band.
- If no entry matches (including the empty-array explicit-denial
  case), the Trust Method is nevertheless satisfied: the consumer
  MUST NOT reject the assertion on the basis of the mismatch.
- If an entry matches, the outcome is identical to enforce mode.

Monitor mode affects only the evaluation of a successfully retrieved
policy. It does not alter lookup-state classification: Indeterminate
outcomes still fail closed (the consumer cannot know the mode of a
policy it could not retrieve), and Negative outcomes are unchanged.

Monitor mode provides no protection: while it is in effect, any
authenticated issuer is accepted for the namespace exactly as if no
policy were enforced, with logging as the only difference. It is a
transitional state; Subject Authorities SHOULD move to enforce mode
promptly once the observed mismatches are resolved (see
{{operational}} for the rollout sequence and {{monitor-security}}
for the downgrade risk).

An aggregate reporting mechanism by which consumers deliver
monitored-mismatch reports to the Subject Authority (analogous to
DMARC's `rua`) is deferred; see {{future-extensions}}.

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
- Consumers MUST enforce an absolute local cache ceiling on the age
  of any cached policy entry (recommended: 24 hours) regardless of TTL
  or `Cache-Control`. A cached entry older than the ceiling MUST NOT be
  used and MUST be re-fetched.
- Serving stale cache during outages is separately bounded. Consumers
  MAY serve a fresh cached Affirmative policy (one within its normal
  TTL/`Cache-Control` lifetime) when a live retrieval is
  Indeterminate. Independently of the absolute ceiling above,
  consumers MUST NOT serve a cached policy across a continuous run of
  Indeterminate live results lasting longer than 1 hour: once the
  most recent successful (Affirmative or Negative) live retrieval is
  more than 1 hour old, the consumer MUST stop serving the cached
  policy and treat the lookup as Indeterminate (reject). Any successful
  live retrieval resets this 1-hour outage window. This bound prevents
  an attacker who can sustain denial of service against the policy
  endpoint from extending revocation latency up to the 24-hour ceiling.
- Negative results SHOULD be cached, subject to the same ceiling,
  to bound lookup work under load ({{dos-ssrf}}). Consumers whose
  threat model includes brief publication-channel takeover SHOULD
  cap negative-cache lifetime at a shorter value (recommended: 5
  minutes) so that a Negative cached during a takeover does not
  hide the legitimate Authority Holder's later publication; the
  same shorter cap SHOULD apply to a cached explicit-denial policy
  ({{dii-document}}).
- Indeterminate outcomes MAY be cached for a short period
  (recommended: no more than 5 minutes) to absorb retry storms;
  an Indeterminate cache entry MUST NOT be treated as a policy and
  never satisfies the Trust Method.

# Trust Methods {#trust-methods}

This document defines `domain_authorized_issuer` as a
`subject_namespace_authorization` Trust Method of {{TRUST-FRAMEWORK}}.
DNS at `_oauth-issuer-policy.{authority}` is the primary publication
channel, with the HTTPS well-known URL as fallback. The Trust Method
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
   `authorized_issuers` using steps 3 and 4 of {{dii-verification}},
   including {{monitor-mode}} when the policy's `mode` is `monitor`.

A Resource Authorization Server uses this variant when it requires
authority publication via HTTPS only and explicitly does not accept
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
selects one lookup mode:

- **Canonical DNS-first lookup** is the default. The Resource
  Authorization Server queries `_oauth-issuer-policy.{A}` first and
  falls back to the HTTPS well-known URL on `negative-authoritative`
  DNS responses ({{dii-lookup}}). The "no DNS record, only HTTPS
  document" Subject Authority case is therefore already covered
  without the HTTPS-only variant.

- **HTTPS-only lookup** is an explicit deployment variant. Use it
  when local policy distrusts DNS-published authority entirely and
  requires authority retrieval over TLS-authenticated HTTPS only.
  Inline DNS records and DNS pointer forms are rejected.

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
differ from the canonical DNS-first lookup of
`domain_authorized_issuer`:

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
- Consumers MUST bound cache lifetimes ({{dii-caching}};
  recommended absolute ceiling 24 hours).
- Consumers performing DNSSEC validation MUST treat validation
  failure as `indeterminate`, not `negative-authoritative`, to
  prevent signature-strip downgrade to HTTPS fallback.
- Subject Authorities SHOULD sign `_oauth-issuer-policy.{A}` with
  DNSSEC, and consumers that do not validate DNSSEC SHOULD use a
  trustworthy resolver path (DoH/DoT to a vetted resolver).

Forged-Negative downgrade: a `negative-authoritative` DNS result
(NXDOMAIN/NODATA) triggers the HTTPS well-known fallback
({{dii-lookup}}). A consumer that does not validate DNSSEC cannot
distinguish a forged negative answer from a genuine one, so an
attacker able to both spoof a negative DNS response and serve (or
redirect to) the victim's apex HTTPS origin can suppress a
legitimately published inline record and substitute an
attacker-controlled HTTPS policy. Consumers that do not validate
DNSSEC and expect a Subject Authority to publish inline records
SHOULD NOT honor a Negative-to-HTTPS transition for that authority
without an authenticated resolver path; where the Subject Authority's
publication channel is configured or discoverable, treat an
unexpected Negative with the same suspicion as Indeterminate.

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

## Monitor-Mode Downgrade {#monitor-security}

An attacker who can modify the published policy has a stealthier
option than adding their own issuer: flipping `mode` from `enforce`
to `monitor`. The policy remains present and superficially intact,
but enforcement is silently off ({{monitor-mode}}). A variant
targets signed policies: because unsigned members absent from the
signed JWT are not conflict-checked ({{TRUST-FRAMEWORK}} §Signed
Policy Metadata), an attacker who can edit the outer document but
not the JWT can inject an unsigned `mode: monitor` beside an intact
signature. Subject Authorities publishing `signed_policy` SHOULD
therefore include `mode` among the signed claims, so that an
injected outer value is rejected as a conflict. Consumers SHOULD
surface mode transitions for a given Subject Authority in their
logs, and Subject Authorities SHOULD monitor their published records
for unexpected `mode` changes with the same rigor as for issuer-list
changes ({{operational}}). Because monitor mode accepts any
authenticated issuer for the namespace, a policy observed in monitor
mode for an extended period SHOULD be treated by operators on both
sides as a misconfiguration signal.

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
  `valid_until`, or explicit denial (an empty `authorized_issuers`
  array, {{dii-document}}) MUST use the HTTPS or DNS pointer form;
  a recognized inline record with no `issuer=` is malformed, so the
  inline form cannot publish an empty delegation set.

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
  a shared issuer with no `tenant` value authorizes every tenant
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

- **Grant profiles without a `tenant` claim.** Under a grant
  profile that carries no `tenant` claim (e.g., the generic
  JWT-bearer grant, {{TRUST-FRAMEWORK}} §Generic JWT-Bearer
  Assertion Grant), only entries that omit `tenant` can match, so
  tenant-scoped authorization is not expressible. A Subject
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
  Server issue a live DNS query, and on a Negative result an HTTPS
  fetch, per distinct authority, turning it into a request amplifier
  aimed at third-party DNS/HTTPS infrastructure and filling its own
  cache with distinct-authority entries. Consumers MUST bound lookup
  work: enforce per-Subject-Authority and global rate limits and
  bound concurrent outstanding lookups. Consumers SHOULD cache
  Negative and Indeterminate outcomes per {{dii-caching}}, and MAY
  impose a maximum number of distinct-authority lookups per unit
  time, shedding load by treating excess as Indeterminate
  (fail-closed).
- **Server-Side Request Forgery via `uri=` and redirects.** An
  attacker who controls DNS for a queried authority can point `uri=`
  (or a redirect) at an arbitrary HTTPS target, including internal
  addresses; the fetch occurs regardless of whether the body
  validates, enabling blind internal probing through timing and error
  differentials. Consumers MUST NOT fetch a `uri=` target or follow a
  redirect whose resolved host is a private-use, loopback, link-local,
  unique-local, or otherwise non-globally-routable IPv4 or IPv6
  address, MUST cap redirects and response size ({{dii-https-url}},
  {{https-policy-document-contract}}), and SHOULD apply a fetch
  timeout. The same-registrable-domain constraint on redirects
  ({{dii-https-url}}) further limits redirect-based SSRF.
- **Cost asymmetry.** A single small assertion can cause a full
  DNS+HTTPS round trip. Consumers SHOULD prefer cached results and
  SHOULD NOT perform a live lookup until the assertion has passed the
  cheaper grant-profile validation checks (signature, audience,
  expiry; {{TRUST-FRAMEWORK}} §Resource Authorization Server
  Processing step 2).

# Privacy Considerations {#privacy}

The lookup is verifier-side and per-verification, so it leaks metadata
in two directions:

- **To the resolver path.** A DNS query for
  `_oauth-issuer-policy.{A}` exposes the Subject Authority `{A}` being
  evaluated (and its timing) to every resolver and on-path observer.
  It does not carry the full subject identifier for formats such as
  `email`, but the authority plus timing can reveal organizational
  relationships and login activity. Resource Authorization Servers
  SHOULD use a privacy-preserving resolver path (DoH/DoT to a vetted
  resolver) and SHOULD NOT perform the lookup until it is needed for a
  concrete verification decision.
- **To the Subject Authority.** For the HTTPS well-known and `uri=`
  channels, the Subject Authority's own server (or its chosen policy
  host) observes the Resource Authorization Server's egress IP address
  and the timing of each fetch, learning which Resource Authorization
  Servers its namespace's users are authenticating to; the
  authoritative DNS operator for `{A}` observes per-lookup timing over
  DNS. An employer acting as its own Subject Authority can thereby
  obtain a near-real-time signal of where employees sign in. Caching
  ({{dii-caching}}) is the primary mitigation: it coarsens timing and
  collapses repeated lookups, so consumers SHOULD cache to the bounds
  permitted rather than re-fetching per verification.

# Operational Considerations {#operational}

Publishing a DAI record makes DNS/HTTPS control a real-time input to
sign-in authorization: a lapse in the record or its host can stop
legitimate assertions from being accepted, and an unauthorized change
can authorize an attacker. Subject Authorities SHOULD operate the
record and any policy host with the same rigor as other sign-in-path
infrastructure. Specific guidance:

- **Rollout.** Deploy in three phases: publish with `mode=monitor`
  ({{monitor-mode}}); observe logged mismatches until the issuer
  list is known complete (forgotten regional tenants and departing
  Identity Providers surface here rather than as sign-in outages);
  then switch to `enforce`. Before enforcing, Subject Authorities
  SHOULD sign the zone with DNSSEC or publish via the HTTPS document
  form, since enforce-mode decisions carry the full weight of the
  publication channel's integrity ({{dns-integrity-and-compromise}}).
- **Change management and TTLs.** Reduce DNS TTLs in advance of any
  planned change to the record or `uri=` pointer ({{third-party-policy-hosts}}),
  and choose steady-state TTLs balancing propagation speed against
  resolver load and the consumer cache bounds of {{dii-caching}}.
- **Rotation.** To rotate an authorized issuer, publish the new
  `authorized_issuers` entry alongside the old one and remove the old
  entry only after the old issuer is decommissioned and caches have
  expired; overlapping validity windows (`valid_from`/`valid_until`)
  make the transition observable and bounded.
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

_NODE NAME:
: `_oauth-issuer-policy`

Reference:
: This document, {{dii-dns-record}}


## Trust Method Registrations

This document registers the following entry in the Identity
Assertion Issuer Trust Methods registry
({{TRUST-FRAMEWORK}} §Identity Assertion Issuer Trust Methods Registry):

| Identifier | Categories | Parameters | Change Controller | Reference |
|-|-|-|-|-|
| `domain_authorized_issuer` | `subject_namespace_authorization` | `lookup` (string, OPTIONAL; value `https_only` selects the HTTPS-only lookup variant) | IETF | This document |

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
multiplicity and duplicate-handling rules are stated.

Initial entries:

| Directive Name | Description | Change Controller | Reference |
|-|-|-|-|
| `v` | Version token; MUST appear first | IETF | This document |
| `authority` | Subject Authority this record binds (A-label) | IETF | This document |
| `uri` | HTTPS URL of an Issuer Authorization Policy document | IETF | This document |
| `issuer` | An authorized Assertion Issuer identifier | IETF | This document |
| `mode` | Enforcement mode: `enforce` (default) or `monitor` | IETF | This document |

## Issuer Authorization Policy Members Registry {#iana-dii-members}

IANA is requested to establish a new registry titled "OAuth Issuer
Authorization Policy Members" under the "OAuth Parameters" registry
group, for the JSON members of the Issuer Authorization Policy
document ({{dii-document}}).

Registration policy: Specification Required {{RFC8126}}.

Each entry contains a Member Name, a Description, a Change Controller,
and a Reference. Designated Expert instructions: the expert verifies
the member name does not collide with an existing member, its JSON
type and semantics are specified, and any decision-affecting member
states how a consumer that does not recognize it behaves (the default
is to ignore unrecognized members; a member requiring fail-closed
handling uses the `crit` mechanism of {{TRUST-FRAMEWORK}} §Critical
Members).

Initial entries:

| Member Name | Description | Change Controller | Reference |
|-|-|-|-|
| `subject_authority` | Subject Authority this policy applies to | IETF | This document |
| `authorized_issuers` | Array of authorized issuer objects | IETF | This document |
| `issuer` | Authorized Assertion Issuer identifier (within an entry) | IETF | This document |
| `tenant` | Issuer-side tenant identifier (within an entry) | IETF | This document |
| `subject_identifier_formats` | Permitted Subject Identifier formats (within an entry) | IETF | This document |
| `valid_from` | Delegation start time (within an entry) | IETF | This document |
| `valid_until` | Delegation end time (within an entry) | IETF | This document |
| `mode` | Enforcement mode: `enforce` (default) or `monitor` | IETF | This document |
| `last_updated` | Policy publication time | IETF | This document |
| `signed_policy` | Signed JWT of the policy members | IETF | This document; {{TRUST-FRAMEWORK}} §Signed Policy Metadata |
| `crit` | Names decision-affecting members a consumer MUST understand or reject the document | IETF | This document; {{TRUST-FRAMEWORK}} §Critical Members |

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
closest to DMARC's, and DMARC's operational experience with the Public
Suffix List and the organizational-domain boundary informs the Subject
Authority Determination approach in {{TRUST-FRAMEWORK}} §Subject
Authority Determination (see also the DBOUND discussion there).

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
format restrictions) that a discovery record does not. A deployment
could layer client-side discovery on top (see
{{assertion-issuer-discovery-client-side}}), but that is out of scope
here.

Relationship to the Email Verification Protocol. EVP
{{I-D.hardt-email-verification}} also publishes, in DNS, issuers
associated with an email domain, but for a different purpose: it names
issuers that can *verify control* of an email address (an issuance-time
question), whereas DAI names issuers *authorized to assert* identities
in a namespace (a verification-time authorization question). The
records differ accordingly: DAI uses full issuer identifiers
(including path components) and PSL-normalized Subject Authorities, and
carries authorization constraints. Convergence with EVP on a shared
record or node name is possible future work; {{email-verification-protocol-bridge}}
sketches a bridge.

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

## Monitoring Reports

Monitor mode ({{monitor-mode}}) relies on consumer-side logging with
out-of-band delivery to the Subject Authority. A future extension can
define an aggregate reporting mechanism, analogous to DMARC's `rua`:
a policy member naming a reporting endpoint, a report format
(observed issuers, match/mismatch counts, time window), and delivery
requirements. It is deferred because report formats and transport
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
      the single entry's `issuer` value. Verification succeeds.

5. The Resource Authorization Server validates `private_key_jwt`,
   then issues an access token in the response body.

## Migration Variant: Pointer Form

If `acme.example` later wants to express validity windows, format
restrictions, or tenant binding, it can switch to the pointer form
without changing any consumer behavior:

~~~
_oauth-issuer-policy.acme.example. IN TXT ( "v=oauth-issuer-policy1;"
    "authority=acme.example;"
    "uri=https://acme.example/.well-known/oauth-issuer-policy" )
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
  with a different `authority=` is discarded; if no other recognized
  record remains, the response is `malformed` (Indeterminate) and
  the assertion is rejected with no fallback to the well-known URL.

- If `acme.example` rotates its authorized Assertion Issuer and the
  Resource Authorization Server has a cached virtual policy, the
  Resource Authorization Server may continue accepting assertions
  from the old issuer until its cache expires. Subject Authorities
  are encouraged to use short DNS TTLs during rotation; consumers
  enforce a local cache ceiling per {{dii-caching}}.

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
