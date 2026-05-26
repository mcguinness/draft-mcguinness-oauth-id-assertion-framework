---
title: "OAuth Trust Policy Discovery"
abbrev: "OAuth Trust Policy Discovery"
docname: draft-mcguinness-oauth-trust-policy-discovery-latest
category: std
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - trust policy
  - issuer discovery
  - DNS authority
  - open-world OAuth

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
  RFC6749:
  RFC8126:
  RFC8552:
  RFC8553:
  RFC8615:
  RFC9728:
  TRUST-POLICY:
    title: "OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-issuer-trust-policy/
    date: false
  AUTHORITY-DELEGATION:
    title: "OAuth Authority Delegation Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-authority-delegation-framework/
    date: false
  DAI:
    title: "OAuth Domain-Authorized Issuer Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer-discovery/
    date: false

informative:
  RFC8414:
  RFC8693:
  TX-DISCO:
    title: "OAuth 2.0 Token Exchange Target Service Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-token-xchg-target-svc-disco/
    date: false
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/

---

--- abstract

A Resource Authorization Server's Trust Policy ({{TRUST-POLICY}})
declares the trust criteria the server enforces against incoming
identity assertions. The policy is published at a well-known URL
on the server's own host; a consumer must already know the
server's identifier to locate it.

This document defines Trust Policy Discovery (TPD): a DNS-based
publication mechanism by which the owner of an authority (a
domain hosting protected resources or a Resource Authorization
Server) makes its Trust Policy discoverable through DNS. TPD is
the dual of Domain-Authorized Issuer Discovery ({{DAI}}): DAI
publishes which Assertion Issuers a namespace owner authorizes;
TPD publishes which trust criteria a resource authority enforces.
Together they close the bilateral first-contact trust gap in
open-world OAuth deployments where neither side has
pre-configured knowledge of the other.

--- middle

# Introduction

OAuth deployments increasingly involve first-contact interactions
where a client or Assertion Issuer encounters a resource without
prior configuration. The Identity Assertion Issuer Trust Policy
{{TRUST-POLICY}} lets a Resource Authorization Server (RAS)
declare its trust criteria in a JSON document served at a
well-known URL. To consume this policy, a client must already
know the RAS's identifier.

The discovery sequence for an agent encountering an unknown
resource is currently multi-hop:

1. The agent fetches the resource URL and expects an HTTP 401
   response.
2. The agent parses Protected Resource Metadata
   ({{RFC9728}}) from the `WWW-Authenticate` header or the
   resource's well-known URL.
3. The agent fetches the Authorization Server Metadata
   ({{RFC8414}}) at the AS identifier given by PRM.
4. The agent locates `identity_assertion_trust_policy_uri` in the
   AS metadata.
5. The agent fetches the Trust Policy at that URL.
6. The agent evaluates compatibility between the policy's
   required Trust Methods and its own capabilities.

Steps 1-5 each require an HTTP request; in adversarial network
conditions the discovery surface is substantial.

Equally important is the symmetric problem: an Assertion Issuer
wanting to know "which Resource Authorization Servers will accept
my assertions" has no wire-format mechanism. The issuer's
compatibility is established only by attempting issuance against
a known RAS and observing acceptance or rejection. There is no
ambient discovery.

Trust Policy Discovery (TPD) addresses both gaps. An
authority that hosts a Resource Authorization Server publishes,
in DNS at `_oauth-trust-policy.{authority}`, a discoverable
pointer to its Trust Policy. Agents, clients, and Assertion Issuers
perform a single DNS lookup to determine which Trust Policy applies;
the DNS record optionally includes a non-authoritative summary of
accepted Trust Methods so pre-flight compatibility decisions can be
made before fetching the full policy.

TPD is the dual of Domain-Authorized Issuer Discovery
({{DAI}}): DAI lets a namespace owner publish authorized issuers
(answering "is this issuer authorized to assert about this
namespace?"); TPD lets a resource authority publish accepted trust
criteria (answering "what would my assertion need to satisfy to
be accepted here?"). Together they make first-contact trust
discovery symmetric.

## Relationship to the OAuth Profile Family

This document is the bilateral counterpart of {{DAI}} in the OAuth
profile family of {{AUTHORITY-DELEGATION}}:

- **{{AUTHORITY-DELEGATION}}**: the abstract Authority Delegation
  Pattern.
- **{{TRUST-POLICY}}**: the OAuth-side framework defining the
  Trust Policy document and Trust Method machinery.
- **{{DAI}}**: namespace-owner-side publication of authorized
  Assertion Issuers via DNS+HTTPS.
- **This document**: resource-owner-side publication of accepted
  trust criteria via DNS+HTTPS.

This document depends normatively on {{TRUST-POLICY}} (Trust
Policy document format) and reuses the DNS+HTTPS
authority-publication pattern of {{DAI}} at a different owner
name with policy-pointing semantics.

## Relationship to Token Exchange Discovery

OAuth Token Exchange Target Service Discovery ({{TX-DISCO}})
defines a runtime endpoint at the Authorization Server by which a
client presenting a subject token discovers acceptable
exchange targets (audiences, resources, scopes, token types). The
two specifications are complementary, operating at different
phases of an open-world flow:

- **TPD** operates at discovery time, before any token exists.
  Any party with knowledge of an authority's domain name can
  query DNS to discover the Trust Policy.
- **{{TX-DISCO}}** operates at runtime, after the client has
  obtained a subject token. The client queries the AS endpoint to
  discover what the token can be exchanged for.

A complete first-contact flow:

1. Agent encounters resource URL `https://api.example.com/users/123`.
2. **TPD** lookup at `_oauth-trust-policy.api.example.com`
   reveals the Trust Policy and the RAS identifier.
3. Agent fetches the Trust Policy and determines what assertion
   it needs.
4. Agent's IdP mints the assertion.
5. Agent presents the assertion to the RAS.
6. **{{TX-DISCO}}** call to the AS refines the exchange target
   list for the presented token.
7. Agent performs token exchange with discovered audience and
   scope.

TPD and {{TX-DISCO}} can be implemented independently. TPD does
not require Token Exchange; {{TX-DISCO}} does not require TPD.
Deployments using both gain the most complete discovery
coverage.

## Relationship to Protected Resource Metadata

Protected Resource Metadata ({{RFC9728}}) publishes per-resource
metadata at a well-known URL on the resource host. PRM is
resource-scoped: each protected resource publishes its own
metadata. TPD is authority-scoped: the entire DNS-named authority
publishes one Trust Policy applicable across its resources.

When both TPD and PRM are deployed, PRM is authoritative for
per-resource refinements; TPD provides the authority-level
discovery shortcut. {{prm-composition}} describes the composition.

## Scope and Non-Goals

This document defines:

- A DNS TXT record format at `_oauth-trust-policy.{A}` carrying a
  Trust Policy pointer and optional compact summary hints.
- An HTTPS document URL fallback at
  `https://{A}/.well-known/oauth-trust-policy-discovery`.
- A lookup procedure producing an Affirmative / Negative /
  Indeterminate outcome per
  {{AUTHORITY-DELEGATION}} §Exception-Handling and Fallback Model.
- Authority Delegation Profile registration.
- Composition guidance with {{TRUST-POLICY}}, {{RFC9728}}, and
  {{TX-DISCO}}.

This document does not:

- Replace {{TRUST-POLICY}}. TPD is a discovery layer. The
  authoritative Trust Policy document remains at the URL TPD
  points at; trust evaluation is performed against the
  authoritative policy, not the TPD summary.
- Define new trust-evaluation logic. RAS Processing follows
  {{TRUST-POLICY}} unchanged.
- Address per-resource trust-policy variation. {{RFC9728}} owns
  per-resource refinements.
- Address per-token exchange-target discovery. {{TX-DISCO}} owns
  that runtime question.
- Define a wire-format mechanism for an Assertion Issuer to
  publish "which authorities accept me." That direction (the
  issuer-perspective dual) is future work.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

This document uses the terminology of {{TRUST-POLICY}} (Resource
Authorization Server, Assertion Issuer, Trust Policy) and
{{AUTHORITY-DELEGATION}} (Authority Holder, Delegate, Delegation
Artifact, Validator). In addition:

Resource Authority:
: The party controlling the DNS-named authority `{A}` under which
  one or more protected resources and the associated Resource
  Authorization Server are hosted. The Resource Authority publishes
  the TPD record. This term is distinct from the OAuth "resource owner"
  role in {{RFC6749}}.

Trust Policy Pointer:
: A reference (URL) carried in the TPD record that locates the
  authoritative Trust Policy document published by the Resource
  Owner.

Trust Policy Summary:
: An optional compact representation of selected Trust Policy
  parameters (accepted Trust Methods, grant profiles, RAS
  identifier) carried inline in the TPD record. The summary is a
  hint for pre-flight compatibility decisions; the authoritative
  Trust Policy is consulted before any assertion is presented or
  verified.

# DNS Record Format {#dns-record}

The Resource Authority publishes one or more DNS TXT records at
`_oauth-trust-policy.{A}`, where `{A}` is the Resource Authority's
DNS-named authority. Internationalized authority names are
converted to A-labels per {{RFC5891}} before the prefix is
prepended.

The string segments of a TXT record ({{RFC1035}}) are concatenated
without separator and interpreted as UTF-8 text.

A record is *recognized* if it begins with the version directive
`v=oauth-trust-policy1` (case-sensitive), optionally followed by
trailing whitespace and a `;` separator. Records that do not
begin with a supported version directive MUST be ignored.

The remainder of a recognized record is a sequence of `name=value`
directives separated by `;`. Parsing rules:

- Whitespace (space and horizontal tab) immediately before and
  after each directive is ignored. Whitespace inside a value is
  not.
- Directive names are case-insensitive ASCII. Values are
  case-sensitive and preserved as written.
- A directive splits into name and value at the first `=`.
  Subsequent `=` characters are part of the value.
- The value runs to the next `;` or end of record. The character
  `;` MUST NOT appear in a value; URLs containing `;` MUST be
  percent-encoded.
- CR, LF, and NUL MUST NOT appear in a value.
- Unrecognized directives MUST be ignored unless listed in a
  `crit=` directive.

The parsing rules above are summarized in the following ABNF
{{RFC5234}} for implementer convenience; the prose above is
normative if any disagreement exists.

~~~ abnf
record       = version *( ";" directive ) [ ";" ]
version      = "v=oauth-trust-policy1"
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
: REQUIRED. The Resource Authority's authority identifier this record
  binds. Consumers MUST normalize this value to A-label form per
  {{RFC5891}}, then verify it matches the authority computed from
  the query name (which is already in A-label form) using
  case-insensitive ASCII comparison. Records whose `authority=`
  value does not match MUST be ignored.

`policy_uri=URL`
: REQUIRED. An HTTPS URL identifying the authoritative Trust
  Policy document defined in {{TRUST-POLICY}}. The URL MAY be on
  a different host than `{A}`. MUST use the `https://` scheme
  and MUST NOT contain a fragment component.

`ras=URL`
: OPTIONAL. The Authorization Server identifier the Resource
  Owner has selected as the Resource Authorization Server for
  resources under `{A}`. This value is a discovery hint until the
  authoritative Trust Policy is fetched and its
  `resource_authorization_server` member is verified. MUST be an
  absolute HTTPS URL and MUST NOT contain a fragment component.

`trust_methods=METHODS`
: OPTIONAL. Comma-separated, case-sensitive list of Trust Method
  identifiers the RAS requires. A compact summary of the
  authoritative policy's `issuer_trust_methods` member, intended
  for pre-flight compatibility decisions without fetching the
  full policy. Consumers MUST treat this as a hint; the
  authoritative policy at `policy_uri` is the source of truth.

`grant_profiles=PROFILES`
: OPTIONAL. Comma-separated list of grant profile identifiers the
  RAS accepts. Summary of the authoritative policy's
  `authorization_grant_profiles_supported`. Same hint semantics
  as `trust_methods`.

`crit=REQUIRED_EXTENSIONS`
: OPTIONAL. Comma-separated, case-insensitive list of directive
  names, extension identifiers, or feature identifiers declared
  by the publisher as critical for correct interpretation.
  Directive names defined by this document (`authority`,
  `policy_uri`, `ras`, `trust_methods`, `grant_profiles`, and
  `crit`) are always recognized. If a consumer does not recognize
  any value named in `crit=`, the consumer MUST treat the record
  as malformed per {{failures}}. If a value names a directive,
  that directive MUST be present in the same record; otherwise
  the consumer MUST treat the record as malformed. Future
  specifications SHOULD use stable extension identifiers in
  `crit=` when correct processing depends on more than
  recognizing a single DNS directive. Critical extension
  identifiers use the syntax and recognition rules in
  {{TRUST-POLICY}} §Critical Extension Identifiers.

A recognized record MUST contain exactly one `policy_uri` directive. A
record with no `policy_uri` directive, an empty `policy_uri` directive,
or more than one `policy_uri` directive MUST be treated as malformed per
{{failures}}. The `trust_methods` and `grant_profiles` directives are
only hints; they do not make a record usable without `policy_uri`.

A Resource Authority MUST publish records for at most one version of
this mechanism at a time. This document defines only
`oauth-trust-policy1`. Future versions that are not understood by
a consumer are ignored by the recognition rule above.

## Example Records

Pointer form (delegates to authoritative Trust Policy):

~~~
_oauth-trust-policy.api.example.com.  IN  TXT
  "v=oauth-trust-policy1;
   authority=api.example.com;
   policy_uri=https://api.example.com/.well-known/identity-assertion-trust-policy;
   ras=https://auth.example.com"
~~~

Combined pointer and summary:

~~~
_oauth-trust-policy.api.example.com.  IN  TXT
  "v=oauth-trust-policy1;
   authority=api.example.com;
   policy_uri=https://api.example.com/.well-known/identity-assertion-trust-policy;
   ras=https://auth.example.com;
   trust_methods=openid_federation,domain_authorized_issuer"
~~~

# HTTPS Document URL {#https-url}

The default Trust Policy Discovery URL is:

~~~
https://{A}/.well-known/oauth-trust-policy-discovery
~~~

where `{A}` is the Resource Authority's authority value rendered as a
host (A-label form for internationalized names). Consumers fetch
this URL using HTTPS with TLS server authentication and interpret
the response body as a JSON object with the following members:

`authority`
: REQUIRED. String. The Resource Authority's authority identifier.
MUST match the authority used in the lookup.

`policy_uri`
: REQUIRED. String. The HTTPS URL of the authoritative Trust
Policy document. MUST use the `https://` scheme and MUST NOT contain a
fragment component.

`ras`
: OPTIONAL. String. The Resource Authorization Server identifier. When
present, MUST be an absolute HTTPS URL and MUST NOT contain a fragment
component.

`trust_methods`
: OPTIONAL. Array of strings. Trust Method identifier summary.

`grant_profiles`
: OPTIONAL. Array of strings. Grant profile identifier summary.

`crit`
: OPTIONAL. Array of strings. Critical extension identifiers per
{{TRUST-POLICY}}.

The JSON document MUST contain `policy_uri`. The `trust_methods` and
`grant_profiles` members are non-authoritative summary hints and do not
make the document usable without `policy_uri`.

Unrecognized members MUST be ignored unless listed in `crit`. If a
consumer does not recognize any value listed in `crit`, the document is
malformed. Member names defined by this document (`authority`,
`policy_uri`, `ras`, `trust_methods`, `grant_profiles`, and `crit`) are
always recognized.

Example:

~~~ json
{
  "authority": "api.example.com",
  "policy_uri": "https://api.example.com/.well-known/identity-assertion-trust-policy",
  "ras": "https://auth.example.com",
  "trust_methods": ["openid_federation", "domain_authorized_issuer"],
  "grant_profiles": ["urn:ietf:params:oauth:grant-profile:id-jag"]
}
~~~

# Lookup Procedure {#lookup}

To retrieve trust policy discovery information for an authority
`{A}`, a consumer performs the following steps. The procedure is
shared by Assertion Issuers determining pre-flight compatibility
and by clients/agents performing first-contact discovery.

DNS at `_oauth-trust-policy.{A}` and HTTPS at the default
well-known URL are two channels of the same Resource Authority.
Consulting HTTPS after DNS returns `negative-authoritative` is
not an Authority Source fallback in the sense forbidden by
{{AUTHORITY-DELEGATION}} §Exception-Handling and Fallback Model;
the abstract Negative state is reached only when both channels
return absence ({{failures}}). The abstract no-fallback rule
continues to forbid falling through to a DIFFERENT Resource
Owner's policy.

1. Query the DNS TXT resource record set at
   `_oauth-trust-policy.{A}`. Classify the response as:

   `negative-authoritative`
   : NXDOMAIN, or NOERROR with an empty answer section or with no
   recognized records after parsing.

   `indeterminate`
   : SERVFAIL, REFUSED, timeout, truncation with no successful
   retry, or any other failure that prevents a definitive
   negative result.

   `affirmative`
   : NOERROR with at least one recognized record after parsing.

2. If the DNS response is `affirmative`:

   a. Discard recognized records whose `authority=` directive
      does not match `{A}` under the comparison rules in
      {{dns-record}}. If any recognized record is missing an
      `authority=` directive, treat the response as malformed.
      If all recognized records are discarded because of
      `authority=` mismatch, treat the response as malformed.

      Validate the remaining recognized records against the
      directive rules in {{dns-record}}. Any malformed
      `policy_uri`, `ras`, `trust_methods`, `grant_profiles`, or
      `crit` value, or any record without exactly one `policy_uri`,
      is a malformed outcome.

   b. If multiple distinct `policy_uri` values are present across
      the remaining records, treat as malformed.

   c. The consumer constructs a Trust Policy Discovery result
      from the union of the recognized records' directives. The
      result has the single `policy_uri` value and MAY include the
      union of `trust_methods` and `grant_profiles` summary hints.

3. If the DNS response is `negative-authoritative`, fetch the
   discovery document from the default HTTPS well-known URL per
   {{https-url}}. Process the document as the discovery result.

4. If the DNS response is `indeterminate`, discovery has failed.
   Consumers MUST NOT fall back to HTTPS, since an attacker who
   suppresses DNS responses could otherwise force the consumer
   onto a path the attacker has compromised separately.

5. The consumer uses the discovered policy reference as input to
   subsequent processing:

   - The consumer fetches the authoritative Trust Policy from
     `policy_uri` per {{TRUST-POLICY}} before evaluating or producing
     any assertion.
   - If summary hints are present, the consumer MAY use them for
     pre-flight compatibility estimates before fetching the policy, but
     any discrepancy is resolved in favor of the authoritative Trust
     Policy.

A consumer MAY skip TPD lookup as a matter of local policy when
the authoritative Trust Policy URL is already known. TPD is a
discovery shortcut; failure of TPD is not a security failure and
does not preclude traditional discovery (resource fetch -> PRM
-> AS Metadata -> Trust Policy).

## Failure Handling {#failures}

TPD lookup outcomes map onto the Affirmative / Negative /
Indeterminate states of the Authority Delegation Pattern's
Exception-Handling and Fallback Model
({{AUTHORITY-DELEGATION}} §Exception-Handling and Fallback Model):

| State | TPD outcome |
|-|-|
| Affirmative | A recognized TPD record set or HTTPS discovery document was retrieved, `authority` matches `{A}`, exactly one `policy_uri` is present, and structural validation succeeds. |
| Negative | DNS `negative-authoritative` followed by HTTPS 404 or 410 at the well-known URL. The Resource Authority authoritatively publishes no TPD record. |
| Indeterminate | Any other outcome, including DNS `indeterminate`, HTTPS 5xx, HTTPS 4xx other than 404 or 410, malformed content at either layer, or a redirect loop. Fail-closed default applies. |

Consumers treat a TPD Negative outcome as "no TPD record
available" and fall back to traditional discovery
(resource-fetch -> PRM -> AS Metadata). A TPD Negative is not an
assertion-rejection event; TPD is a discovery aid, not a trust
gate.

Consumers MUST treat TPD Indeterminate as transient. The
consumer SHOULD retry TPD after a profile-appropriate delay (the
recommended cumulative-indeterminate cap is 1 hour per {{DAI}}).
If TPD remains Indeterminate beyond the cap, the consumer MUST
fall back to traditional discovery; failing the resource access
because TPD is unavailable would be an availability-driven
denial of service.

# Integration with Protected Resource Metadata {#prm-composition}

Protected Resource Metadata ({{RFC9728}}) publishes per-resource
metadata at a well-known URL on the resource host. PRM is
resource-scoped; TPD is authority-scoped.

A Resource Authority SHOULD publish both:

- A TPD record at `_oauth-trust-policy.{authority}` for ambient
  discovery.
- PRM at `https://{resource-host}/.well-known/oauth-protected-resource`
  with `identity_assertion_trust_policy_uri` per {{TRUST-POLICY}}.

When both are available, consumers MAY use either path:

- TPD-first: query DNS, then fetch the Trust Policy directly.
  Fastest for ambient agent first-contact.
- PRM-first: fetch the resource, parse PRM from the 401 response
  or the well-known URL, follow to the Trust Policy. Familiar to
  existing OAuth implementations.

When both paths yield divergent Trust Policy URLs, the value
referenced from PRM is authoritative for the specific resource
being accessed. TPD is broader-grained; PRM is the per-resource
refinement.

# Composition with Token Exchange Discovery {#tx-disco-composition}

TPD and Token Exchange Target Service Discovery ({{TX-DISCO}})
together cover the open-world discovery lifecycle:

| Phase | Mechanism | What the consumer asks |
|-|-|-|
| 1. Encounter | TPD DNS lookup | What Trust Policy applies to this authority? |
| 2. Pre-flight | TPD summary | Are my issuer capabilities compatible? |
| 3. Policy fetch | HTTPS Trust Policy | What are the full trust criteria? |
| 4. Assertion | Issuer mints assertion | (Out of scope) |
| 5. Token exchange | {{TX-DISCO}} endpoint | Given my token, what targets are available? |
| 6. Token use | Resource access | (Out of scope) |

TPD covers phases 1-3. {{TX-DISCO}} covers phase 5. Phase 4 is
the existing assertion-issuance flow defined by the grant profile
(e.g., {{ID-JAG}}).

Deployments using both gain the most complete discovery coverage.
A deployment using only TPD supports first-contact discovery
without runtime exchange-target negotiation. A deployment using
only {{TX-DISCO}} supports runtime exchange but assumes the
client already knew which AS to contact.

# Authority Binding

The authority binding for TPD mirrors {{DAI}}: control of the DNS
namespace `{A}` IS the authority for purposes of TPD. A Resource
Authority publishing a TPD record at `_oauth-trust-policy.{A}`
attests, by virtue of DNS control, that the named Trust Policy
applies to `{A}`'s resources.

The Trust Policy referenced by `policy_uri` MAY be hosted on a
different origin than `{A}`. In that case, consumers fetching
the policy MUST verify the `signed_policy` member per
{{TRUST-POLICY}} §Signed Policy Metadata before relying on the
document's contents. This is the same third-party-hosting
guarantee provided by {{DAI}}'s signed-policy publication
profile.

Consumers fetching a `policy_uri` whose host IS under `{A}`'s
DNS-namespace control MAY rely on TLS server authentication
alone, without requiring `signed_policy`.

# Caching {#caching}

DNS TPD records are cached per DNS TTL, capped at the consumer's
local maximum cache lifetime (recommended ceiling: 24 hours per
{{DAI}}). The HTTPS discovery document and the referenced Trust
Policy are cached per HTTP `Cache-Control` headers, capped by the
same local maximum.

Negative TPD lookups (NXDOMAIN at the well-known DNS name) MAY
be cached briefly. The recommended Negative cache lifetime is 5
minutes; longer caching of TPD Negative results risks masking a
later publication for the cache's lifetime.

Cumulative Indeterminate retrievals MUST NOT extend the effective
cache lifetime beyond the profile-specified maximum (recommended
1 hour cumulative cap per {{DAI}}).

A Resource Authority changing its Trust Policy (whether updating
`policy_uri`, switching RAS, or adding required Trust Methods)
SHOULD reduce TPD DNS TTLs in advance of the change to bound the
discovery-window for stale results.

# Operational Considerations

## Lifecycle and Change Management

A Resource Authority's Trust Policy is a security-critical document.
Changes to TPD records (especially `policy_uri` redirection or
`ras` changes) propagate at DNS-TTL plus consumer-cache speeds.
Resource Authorities SHOULD:

- Reduce TPD DNS TTL to operational minimums (minutes) before
  planned policy changes.
- Maintain backward-compatible Trust Policies during transitions
  (per {{TRUST-POLICY}} §Operational Considerations).
- Restore steady-state DNS TTL only after a full cache lifetime
  has elapsed since the change.
- Monitor `last_updated` on the referenced Trust Policy.

## Co-publication with PRM and DAI

A Resource Authority deploying TPD should consider co-publication of:

- {{RFC9728}} Protected Resource Metadata for per-resource
  refinements.
- {{DAI}} records under the customer namespaces the deployment
  serves (if the Resource Authority also operates as a Subject
  Authority).
- {{TRUST-POLICY}} signed-policy publication for third-party
  hosting.

The four mechanisms are complementary; deploying all of them
maximizes discovery interoperability.

## Monitoring

Resource Authorities SHOULD monitor:

- DNS record presence and content at `_oauth-trust-policy.{A}`.
- HTTPS discovery document availability at the well-known URL.
- The referenced Trust Policy's `last_updated` and HTTP
  Cache-Control.
- Cross-region and cross-resolver consistency of TPD lookups.

Resource Authorization Servers SHOULD log:

- Whether a consuming request followed TPD discovery (inferable
  from request timing, User-Agent, or explicit signal).
- The Trust Policy version applied at evaluation time.

# Security Considerations

## DNS Integrity

DNS-published TPD records are only as trustworthy as the
resolution path that delivers them. An attacker who can
substitute the DNS response can redirect consumers to a
substituted Trust Policy or RAS.

Resource Authorities SHOULD:

- Deploy DNSSEC on the zone containing `_oauth-trust-policy.{A}`.
- Apply registrar-lock to prevent registrar-account-compromise
  redirection.
- Monitor for unexpected changes to the TPD record set.

Consumers SHOULD:

- Use DNSSEC-validating resolvers where available.
- Use privacy-preserving resolution paths (DNS-over-HTTPS,
  DNS-over-TLS) when the queried authority is sensitive.

DNS hijack mitigations are addressed in {{AUTHORITY-DELEGATION}}
§Authority Source Compromise; TPD inherits this analysis. TPD's
unique exposure is that a successful DNS hijack redirects
discovery, not authority: the consumer is misled to the wrong
Trust Policy, but the eventual trust evaluation against the
substituted policy still fails closed if the actual RAS endpoint
is not under the attacker's control.

## Trust Policy Summary Hint Misuse

The inline `trust_methods` and `grant_profiles` directives are
hints for pre-flight compatibility. An attacker who can spoof DNS
but not the HTTPS Trust Policy may publish misleading summary
hints.

Consumers MUST NOT make trust decisions based on TPD summary
hints alone. The authoritative Trust Policy MUST be fetched
before issuing or verifying assertions. Consumers SHOULD log
discrepancies between the TPD summary and the authoritative
policy as potential tampering indicators.

## Third-Party Trust Policy Hosting

The `policy_uri` MAY point at a Trust Policy hosted on a
different origin than `{A}`. In that case:

- The Trust Policy MUST be a signed policy per
  {{TRUST-POLICY}} §Signed Policy Metadata.
- Consumers MUST verify the signature with a key controlled by
  the Resource Authorization Server.
- A compromise of the third-party host that serves a tampered
  document is detected by signature validation failure.

This is the same third-party-hosting safety property provided by
{{DAI}}'s signed-policy publication profile.

## Resource Authorization Server Substitution

A TPD record's `ras` value names the Authorization Server the
Resource Authority has selected. An attacker who substitutes the
`ras` value redirects consumers to an attacker-controlled AS;
tokens minted there would be accepted by the consumer (since the
attacker's AS would issue valid-looking tokens).

This attack is mitigated by:

- DNSSEC on `_oauth-trust-policy.{A}`.
- Cross-checking `ras` against the authoritative Trust Policy's
  `resource_authorization_server` member.
- Refusing TPD `ras` values that do not match the policy's RAS.

Consumers MUST verify that the Trust Policy fetched from
`policy_uri` declares the same RAS identifier as the TPD `ras`
directive. Discrepancy MUST be treated as a malformed TPD outcome.

## Open-World Adversary Considerations

TPD makes Trust Policy URLs discoverable from a domain name. This
expands the attack surface compared to bilateral configuration:
an attacker can enumerate TPD records across many domains to map
trust deployments at scale.

This is a reconnaissance enabler, not a vulnerability per se.
Operators concerned about deployment-relationship disclosure
SHOULD weigh the trade-off between discoverability (benefit) and
fingerprinting (cost) when deciding whether to publish TPD.
Deployments with confidentiality requirements MAY omit TPD and
require consumers to use traditional resource-then-PRM-then-AS
discovery.

## Privacy of Discovery

Every TPD DNS query discloses the queried authority to DNS
resolvers and to the authoritative nameserver. Privacy
considerations are addressed in {{privacy}}.

# Privacy Considerations {#privacy}

TPD DNS lookups disclose:

- The queried authority `{A}` to the recursive DNS resolver, the
  authoritative DNS server, and any on-path observers without
  encrypted DNS.
- The fact that the consumer is performing trust-policy
  discovery for `{A}`.

This is comparable to {{DAI}}'s privacy properties. Mitigations:

- Use a privacy-preserving DNS resolution path (DNS-over-HTTPS,
  Oblivious DNS-over-HTTPS, or a vetted enterprise resolver) to
  reduce resolver-path exposure.
- Cache TPD results aggressively (within the recommended cache
  lifetime bounds) to reduce the rate of fresh DNS queries.
- Defer TPD lookup until a concrete trust-policy decision is
  required; do not pre-resolve speculatively.

The HTTPS discovery document at the well-known URL discloses the
same information to the policy host (and to network observers
if TLS is broken). Privacy properties are the same as fetching
any well-known URL.

# IANA Considerations

## Well-Known URI

IANA is requested to register the following well-known URI in the
"Well-Known URIs" registry {{RFC8615}}:

URI Suffix:
: `oauth-trust-policy-discovery`

Change Controller:
: IETF

Specification Document:
: This document, {{https-url}}

Status:
: permanent

Related Information:
: None

## Underscored DNS Node Name

IANA is requested to register the following entry in the
"Underscored and Globally Scoped DNS Node Names" registry per
{{RFC8552}} and {{RFC8553}}:

RR Type:
: TXT

Node Name:
: `_oauth-trust-policy`

Specification Document:
: This document, {{dns-record}}

## Authority Delegation Profile Registration

This document registers an entry in the Authority Delegation
Profile registry of {{AUTHORITY-DELEGATION}} (§Authority
Delegation Profile Registry):

Profile Identifier:
: `urn:ietf:params:oauth:authority-delegation-profile:trust-policy-discovery`

Authority Holder Form:
: DNS-named resource-owner domain.

Delegate Form:
: Trust Policy (the document published by the Resource Authority per
{{TRUST-POLICY}}). Note: this profile delegates POLICY
publication, not assertion authority. The Delegation Artifact
is the Trust Policy document; this draft defines its
discovery mechanism.

Publication Channel:
: DNS TXT at `_oauth-trust-policy.{authority}` with optional
HTTPS pointer; HTTPS fallback at
`/.well-known/oauth-trust-policy-discovery`; signed-policy
publication of the referenced Trust Policy at arbitrary HTTPS
origins via {{TRUST-POLICY}}'s `signed_policy` member.

Subdelegation:
: Not permitted. The TPD record names exactly one authoritative
Trust Policy.

Reference:
: This document.

Profile Mapping Matrix:

`source_selection`
: The Resource Authority is extracted from the queried
domain name (the DNS authority of the resource URL or
namespace identifier). Selection uses one property (the queried
name); the function is invariant under other request properties.

`revocation_lifecycle`
: Cache-bounded with no push mechanism. Positive cache lifetime
is bounded by DNS TTL and HTTP `Cache-Control`, capped at the
profile-specified maximum (recommended ≤24h ceiling).
Negative results capped at 5 minutes recommended. Cumulative
Indeterminate cap of 1 hour mirrors {{DAI}}.

`transitivity_limit`
: Depth-1. The TPD record names exactly one Trust Policy URL;
no further indirection is permitted. The referenced Trust
Policy's own delegation semantics (e.g., to a signed policy
hosted at a third-party origin) are governed by
{{TRUST-POLICY}}.

# Worked Example (Non-Normative)

This appendix is non-normative.

## Cast

- **Resource Authority**, `api.example.com`. Hosts a multi-tenant
  SaaS API. Publishes TPD and Trust Policy.
- **Resource Authorization Server**, `https://auth.example.com`.
  Issues access tokens for API resources.
- **Agent Provider**, `https://idp.agentprovider.example`. An
  Assertion Issuer wanting to mint assertions for users
  accessing the SaaS.
- **End user**, Alice (`alice@acme.example`).
- **Agent platform client**, the agent runtime running on
  behalf of Alice.

## TPD Publication

The Resource Authority publishes:

~~~
_oauth-trust-policy.api.example.com.  IN  TXT
  "v=oauth-trust-policy1;
   authority=api.example.com;
   policy_uri=https://api.example.com/.well-known/identity-assertion-trust-policy;
   ras=https://auth.example.com;
   trust_methods=openid_federation,domain_authorized_issuer"
~~~

The HTTPS Trust Policy at the referenced URL:

~~~ json
{
  "resource_authorization_server": "https://auth.example.com",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer"
    }
  ]
}
~~~

## Agent Provider Pre-Flight

Before issuing an assertion, the agent provider determines
compatibility:

1. The agent provider learns the target resource URL
   `https://api.example.com/v1/users/alice`.
2. It queries DNS at `_oauth-trust-policy.api.example.com`. The
   inline summary reveals
   `trust_methods=openid_federation,domain_authorized_issuer`.
3. The agent provider knows it IS a member of
   `https://federation.example.org` (satisfying
   `openid_federation`) and IS listed in `acme.example`'s DAI
   policy (satisfying `domain_authorized_issuer` for Alice's
   email namespace). Pre-flight check passes.
4. The agent provider fetches the authoritative Trust Policy at
   `policy_uri` and confirms no additional requirements.

## Agent First-Contact

A separate agent encountering the same resource URL for the
first time, with no prior configuration:

1. Agent extracts authority `api.example.com` from the resource
   URL.
2. Agent queries DNS at `_oauth-trust-policy.api.example.com`.
   Affirmative. Receives `policy_uri` and `ras`.
3. Agent fetches the Trust Policy at `policy_uri`. Learns
   required Trust Methods.
4. Agent determines its IdP supports the required methods.
5. Agent's IdP mints an ID-JAG.
6. Agent presents the ID-JAG to
   `https://auth.example.com/token` (the `ras` value).
7. (Optional) Agent invokes {{TX-DISCO}} at the AS for runtime
   exchange-target refinement.
8. Agent calls `https://api.example.com/v1/users/alice` with
   the issued token.

The agent has completed first-contact discovery in a single DNS
roundtrip plus one HTTPS fetch, rather than the four-roundtrip
discovery sequence required without TPD.

## Migration Variant: PRM Co-Publication

The Resource Authority also publishes PRM at
`https://api.example.com/.well-known/oauth-protected-resource`:

~~~ json
{
  "resource": "https://api.example.com",
  "authorization_servers": ["https://auth.example.com"],
  "identity_assertion_trust_policy_uri":
    "https://api.example.com/.well-known/identity-assertion-trust-policy"
}
~~~

A consumer following the traditional PRM-first path finds the
same Trust Policy URL. The two discovery paths converge on the
same authoritative policy.

## Failure Variants

- **DNS Indeterminate** (resolver timeout): the agent retries
  briefly, then falls back to traditional discovery (resource
  fetch -> PRM). TPD is a shortcut; its failure does not block
  the agent's progress.

- **DNS hijack publishes a substituted `ras` value pointing at
  an attacker-controlled AS**: the agent fetches `policy_uri`
  (which the attacker may or may not have substituted). The
  consumer cross-checks the policy's `resource_authorization_server`
  against the TPD `ras` value. If they match (attacker
  substituted both), the consumer relies on TLS plus the
  attacker's inability to obtain a valid TLS certificate for
  the attacker's AS without ALSO compromising the authority's
  DNS for the well-known URL. Layered defenses (DNSSEC,
  Certificate Transparency monitoring, CAA) raise the bar.

- **TPD published but Trust Policy fetch fails (HTTPS 5xx)**:
  the agent treats this as Indeterminate per the Trust Policy
  fetch's own failure handling. TPD itself succeeded; the
  authoritative policy is unavailable. Agent retries or falls
  back to traditional discovery.

# Acknowledgments

The author thanks the OAuth WG and the broader community for
discussions of open-world OAuth discovery problems.

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
