---
title: "OAuth Identity Assertion Issuer Trust Framework"
abbrev: "Identity Assertion Trust Framework"
docname: draft-mcguinness-oauth-identity-assertion-trust-framework-latest
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
  - trust policy

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC7515:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8615:
  RFC9493:
  RFC9728:
  RFC1035:
  RFC5891:
  RFC8126:
  RFC8552:
  RFC8553:
  OIDF-FEDERATION:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  ACTOR-PROFILE:
    title: "OAuth Actor Profile for Delegation"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-profile/
    date: false

informative:
  RFC5234:
  RFC7009:
  RFC7662:
  RFC8461:
  RFC8555:
  RFC8659:
  WICG-EMAIL-VERIF:
    title: "Email Verification Protocol"
    target: https://wicg.github.io/email-verification-protocol/
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/
  AUTHORITY-DELEGATION:
    title: "OAuth Authority Delegation Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-authority-delegation/
    date: false

---

--- abstract

Issuer authentication alone does not prove an OAuth authorization
server's authority over the subject namespace its identity assertions
claim. A federated authorization server can mint an identity assertion
naming any email domain; federation membership establishes that the
server is a recognized member of an ecosystem, not that the server is
entitled to assert about subjects in any particular namespace. Nothing
in OAuth today lets a namespace owner declare which authorization
servers are authorized to assert identities in its namespace.

This document defines a trust framework that keeps issuer
authentication and subject namespace authorization as distinct,
independent questions, and lets a Resource Authorization Server
require evidence for both. The framework defines:

- An Identity Assertion Issuer Trust Policy that an OAuth
  authorization server publishes to declare its trust criteria,
  including trust methods, trust anchors, subject identifier
  formats, and grant profiles.

- Domain-Authorized Issuer Discovery, which applies the established
  DNS authority-declaration pattern used by CAA, MTA-STS, SPF, and
  DKIM: the owner of a subject namespace (for example, an email
  domain) publishes a DNS record that declares which authorization
  servers may assert identities about subjects in the namespace.
  Richer policy may be expressed in an HTTPS-hosted document
  referenced from the DNS record. The same records enable clients
  to discover the Assertion Issuer for an identifier in that
  namespace.

--- middle

# Introduction

OAuth assertion-based authorization grants {{RFC7521}} {{RFC7523}} allow
identity-bearing assertions issued by one authorization server to be
presented to another. The Identity Assertion JWT Authorization Grant
(ID-JAG) {{ID-JAG}} is one such profile. Identity chaining and related
deployments share a discovery problem: the Resource Authorization Server
needs to decide whether the issuer of an identity assertion is
acceptable for subject resolution, account linking, or delegated access.

In many deployments the set of acceptable Assertion Issuers cannot be
enumerated in advance. It may be large, dynamic, or governed by external
trust frameworks. A static issuer allowlist also cannot express the
conditions under which an Assertion Issuer is acceptable: for example,
whether the Assertion Issuer is a member of a recognized federation, or
whether the Assertion Issuer has authority for the subject namespace
being asserted.

This is an open-world issuer-trust problem: the Resource Authorization
Server may receive identity assertions from issuers that were not
individually configured in advance, but whose acceptability can be
evaluated from published evidence at request time. The purpose of this
framework is not to remove policy from the Resource Authorization
Server, but to give it a standard way to express which evidence it
requires and how that evidence is evaluated.

This document defines an Identity Assertion Issuer Trust Policy
that a Resource Authorization Server publishes to describe its trust
criteria. The Resource Authorization Server does not say "these are
all the Assertion Issuers I trust"; it says "these are the conditions
an Assertion Issuer must satisfy." Conditions are evaluated by
validating concrete evidence (a federation trust chain or a
domain-authorized issuer record) when an assertion is presented.

This document also defines Domain-Authorized Issuer
Discovery, a self-contained mechanism for one of those conditions, by
which a subject namespace owner authorizes specific Assertion Issuers
via DNS or HTTPS. Because both verifiers and clients consume the same
records, the mechanism doubles as an Assertion Issuer discovery
facility for the namespace.

## Two Trust Questions {#two-trust-questions}

An identity assertion is the result of an authority delegation: some
party with authority over a subject namespace has delegated, explicitly
or implicitly, the right to assert about subjects in that namespace to
the Assertion Issuer. A Resource Authorization Server resolves whether
that delegation exists and applies by asking two sub-questions:

1. *Who signed it?* Is the entity identified by the JWT `iss` claim
   authentic, that is, a recognized authorization server in a trusted
   ecosystem?

2. *Who authorized it?* Does the Authority Holder for the subject
   namespace (the namespace owner) delegate to this Assertion Issuer
   for assertions about subjects in that namespace (for example,
   emails in `example.com`)?

A trust framework that conflates these two questions cannot prevent a
federated authorization server from impersonating users in a namespace
it has no authority over. This document keeps them separate. Each Trust
Method belongs to exactly one of two categories (`issuer_authentication`
for question 1, `subject_namespace_authorization` for question 2), and a
Resource Authorization Server's trust policy requires evidence from BOTH
categories when both apply to the assertion. Trust Method categories are
defined in {{trust-method-categories}}; the combination rule is in
{{rasp}}.

A consequence is that transitive *authentication* (federation chains
that prove an issuer is a member of a recognized ecosystem) is in scope
and supported via the `issuer_authentication` category. Transitive
*namespace authorization* (chains in which a namespace owner delegates
to one party which delegates further) is intentionally not supported:
each Subject Authority lists its authorized Assertion Issuers directly,
and the namespace-authorization trust graph is bounded to a depth of
one. See {{transitive-authz-bounded}}.

## Motivating Use Cases {#motivation}

Deployment patterns where the gap between issuer authentication
and namespace authorization has practical consequences:

- **Workforce SSO into multi-vendor SaaS.** Each vendor configures
  the accepted Assertion Issuer bilaterally per customer; no vendor
  can independently verify that the configured Identity Provider
  is the one the customer's domain authorizes for its email
  namespace.

- **AI agent platforms acting across tool boundaries.** A tool's
  Resource Authorization Server needs to know whether the agent
  platform is entitled to assert about users in the customer's
  namespace, not merely that the platform is signing with a known
  key.

- **B2B integrations carrying end-user identity.** Without
  wire-format namespace authorization, the called organization
  either accepts any authenticated Identity Provider (no namespace
  control) or maintains manual allowlists (fragile at scale).

- **Shared-issuer multi-tenant Identity Providers.** A customer
  authorizing such an Identity Provider today must trust its
  tenant-isolation enforcement implicitly. This framework makes
  the customer's choice of authorized tenant observable on the wire
  ({{dii-multi-tenant}}).

Today's alternatives are bilateral OAuth configuration, federation
membership treated incorrectly as a proxy for namespace authority,
or implicit trust in tenant-domain bindings. This document provides
a wire-format alternative for each.

## Open-World Trust Evaluation

This document targets open-world deployments: acceptance is based on
verifiable evidence (a federation trust chain, a DNS or HTTPS
authority policy, or other Trust-Method-registered evidence) rather
than prior bilateral enumeration. Open-world evaluation does not
mean open acceptance: the Resource Authorization Server publishes
the Trust Methods it accepts, applies local authorization policy,
and fails closed when required evidence is missing, malformed, or
indeterminate.

## Relationship to Existing Mechanisms

This framework is complementary to OpenID Federation: OpenID Federation
can authenticate that an issuer belongs to a trusted ecosystem, while
Domain-Authorized Issuer Discovery lets the namespace owner say which
issuers may assert about subjects in that namespace. The framework also
follows existing DNS authority-publication patterns such as CAA,
MTA-STS, SPF, DKIM, and the Email Verification Protocol. Background and
positioning details are in {{relationship-to-oidf}} and
{{dns-authority-patterns}}.

## Two Documents Defined Here

This document defines two JSON documents that should not be confused:

- The Identity Assertion Issuer Trust Policy
  ({{trust-policy-document}}) is published by a Resource Authorization
  Server and describes the conditions under which it accepts identity
  assertions.

- The Issuer Authorization Policy ({{dii-document}}) is
  published by a Subject Authority and lists Assertion Issuers
  authorized to assert about subjects in its namespace.

The first governs what the Resource Authorization Server accepts; the
second supplies one kind of evidence that an Assertion Issuer can
offer to satisfy the first.

## Scope and the Broader Authority-Discovery Pattern {#scope}

This document defines a single mechanism: publication by a Subject
Authority of which Assertion Issuers may issue identity assertions
about subjects in its namespace. It is intentionally narrow.

The broader pattern (an authority publishes which issuers may
assert a class of claims about subjects in its scope) applies to
attribute types whose authority is not a namespace owner:
government-issued credentials, professional certifications,
employment claims, financial attributes, and decentralized
credentials. The Trust Method category structure
({{trust-method-categories}}) and the Issuer Authorization Policy
document format ({{dii-document}}) are designed to allow other
specifications to define publication and lookup mechanisms suited
to those authorities; this document does not attempt to do so.

Implementers needing attribute attestation for non-namespace-bound
attributes should look to W3C Verifiable Credentials, OpenID for
Verifiable Credentials, and domain-specific frameworks; this
document does not replace them. Where their attestations are
conveyed as OAuth identity assertions whose subject is
namespace-bound, the trust framework defined here can govern the
issuer-trust layer regardless of the attribute-attestation layer
beneath it.

Within the scope of namespace-bound subject identity, future
extensions to additional Subject Identifier formats are
straightforward and require no changes to the trust evaluation
model. See {{extending}}.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

This document uses Authority Delegation Pattern terminology
(Authority Holder, Delegate, Delegation Artifact, Validator) as
defined in {{AUTHORITY-DELEGATION}}. Subject Identifier formats
follow {{RFC9493}}; Trust Method identifiers are registered in
{{iana-trust-methods-registry}}.

Five terms are specific to this document:

Resource Authorization Server (RAS):
: The OAuth authorization server that receives an identity
assertion as a grant, evaluates the trust policy, and issues
access tokens for a protected resource. Not a new OAuth protocol
role; OAuth readers may read it as "the authorization server
receiving the assertion grant."

Assertion Issuer:
: The service issuing an identity-bearing assertion, identified by
the JWT `iss` claim. The same string is the `issuer` value in
OAuth Authorization Server Metadata {{RFC8414}}, OpenID Connect
Discovery {{OIDC-DISCOVERY}}, and the federation entity identifier
in {{OIDF-FEDERATION}}.

Subject Authority:
: The Authority Holder for a Subject Identifier namespace. For
`email`, the registrable domain.

Trust Policy:
: The JSON document defined in {{trust-policy-document}}, published
by a Resource Authorization Server to declare its trust criteria.

Issuer Authorization Policy:
: The JSON document defined in {{dii-document}}, published by a
Subject Authority to authorize Assertion Issuers in its namespace.

# Trust Policy

## Metadata Publication

A Resource Authorization Server publishes the location of its Identity
Assertion Issuer Trust Policy in its authorization server metadata
{{RFC8414}} using the following member:

`identity_assertion_trust_policy_uri`
: OPTIONAL. HTTPS URI identifying the Identity Assertion Issuer Trust
  Policy document.

A Protected Resource that publishes OAuth 2.0 Protected Resource
Metadata {{RFC9728}} MAY include the same member to advertise the
policy applied by its Resource Authorization Server. Clients that
obtain the URI from Protected Resource Metadata MUST verify that the
policy document's `resource_authorization_server` value identifies an
authorization server listed in the Protected Resource's
`authorization_servers` array. If the same member appears in both
metadata documents, the authorization server metadata value is
authoritative.

A Resource Authorization Server MAY additionally publish the trust
policy document at the well-known URI registered in
{{iana-trust-policy-well-known}}. If both the metadata member and the
well-known URI are available and identify different documents, the
metadata member is authoritative.

Example authorization server metadata:

~~~ json
{
  "issuer": "https://api.resource.example",
  "token_endpoint": "https://api.resource.example/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "identity_assertion_trust_policy_uri":
    "https://api.resource.example/.well-known/identity-assertion-trust-policy"
}
~~~

## Trust Policy Document {#trust-policy-document}

The Identity Assertion Issuer Trust Policy is a JSON document served
over HTTPS with media type `application/json`. The document MUST be a
JSON object. Consumers MUST ignore unrecognized members unless
otherwise specified. Consumers MUST reject a policy document that is
not syntactically valid JSON, whose top-level value is not an object,
or whose required members are absent or have the wrong JSON type.

Example:

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
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

### Members

`resource_authorization_server`
: REQUIRED. The issuer identifier of the Resource Authorization Server.
MUST exactly match the authorization server metadata `issuer` value
{{RFC8414}}.

`authorization_grant_profiles_supported`
: REQUIRED. JSON array of identity assertion grant profile identifiers
supported by this policy. This member uses the same name and identifier
space as the OAuth Authorization Server Metadata parameter defined in
{{ID-JAG}} §7.2; each value MUST match a profile identifier listed in
the authorization server metadata's
`authorization_grant_profiles_supported` parameter, where present.

`subject_identifier_formats_supported`
: OPTIONAL. JSON array of Subject Identifier format names that the
Resource Authorization Server accepts. Values are defined by
{{RFC9493}}, by grant profile specifications, or by other
specifications. This member applies to all Trust Methods; an
individual Trust Method MAY constrain the formats it evaluates (for
example, `domain_authorized_issuer` evaluates only formats registered
in {{iana-authority-registry}}).

`issuer_trust_methods`
: REQUIRED. Non-empty JSON array of Trust Method objects (see
{{trust-methods}}). This member states the trust REQUIREMENTS the
Resource Authorization Server enforces against incoming identity
assertions, not a list of capabilities. An assertion is rejected
unless an Assertion Issuer satisfies the Trust Method combination
rule in {{rasp}} against the methods listed here; an issuer that
happens to satisfy a method NOT listed here is not acceptable. The
member is deliberately named without the `_supported` suffix used
by capability-style OAuth metadata parameters
({{RFC8414}}) to signal that distinction. Local policy MAY impose
additional requirements for specific clients, subjects, or scopes;
see {{downgrade}}.

`crit`
: OPTIONAL. JSON array of strings. Each string names an extension
identifier, feature identifier, or policy member whose recognition the
publisher considers critical for correct interpretation. If a consumer
does not recognize any string listed in `crit`, the consumer MUST treat
the policy as malformed. Member names defined in this document
(`resource_authorization_server`,
`authorization_grant_profiles_supported`,
`subject_identifier_formats_supported`, `issuer_trust_methods`,
`signed_policy`, and `crit`) are always recognized. Critical
extension identifiers use the syntax and recognition rules in
{{critical-extension-identifiers}}.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, using the
  representation defined in {{signed-policy-metadata}}. This member
  follows the signed metadata pattern used by {{RFC8414}} and
  {{RFC9728}}. Consumers that do not support signed policy metadata MAY
  ignore this member unless it is listed in `crit`.

Unrecognized Trust Policy members MUST be ignored, except when the
member name or an extension identifier governing the member is listed
in `crit`.

## Trust Methods {#trust-methods}

### Trust Method Object Structure {#trust-method-structure}

Each Trust Method object has a `method` member containing a Trust Method
identifier registered in {{iana-trust-methods-registry}}, together with
any additional members required by that identifier. Each Trust Method
object MUST be a JSON object and MUST contain exactly one `method`
member whose value is a string. Unrecognized Trust Method identifiers
and unrecognized members within a recognized object MUST be ignored,
except when required by a future extension's criticality mechanism.
For a recognized Trust Method identifier, consumers MUST reject the
Trust Method object as malformed if any member required by that
identifier is absent, has the wrong JSON type, or has a value outside
the constraints defined for that identifier. Malformed Trust Method
objects are not usable. If no recognized and well-formed Trust Method
object remains after this processing, the policy does not identify any
usable issuer trust method.

### Trust Method Categories {#trust-method-categories}

Each Trust Method belongs to exactly one of the two categories
below and is registered in {{iana-trust-methods-registry}}. The
cross-category combination rule ({{rasp}}) requires evidence from
EACH category present in the policy and applicable to the
assertion: AND across categories, OR within.

`issuer_authentication`
: Applies unconditionally. Is the entity identified by the JWT
`iss` claim authentically a member of a recognized ecosystem?

`subject_namespace_authorization`
: Applies when the assertion carries a Subject Identifier. Is
this Assertion Issuer entitled to assert about subjects in the
named namespace? If the Subject Authority cannot be determined,
the assertion fails closed.

Deployments accepting assertions about namespace-bound subjects
SHOULD list at least one `subject_namespace_authorization`
method; federation membership alone does not establish namespace
authority. Future specifications MAY register additional
categories.

### Issuer Authentication Methods {#issuer-authentication-methods}

`issuer_authentication`-category methods are satisfied by evidence
that the Assertion Issuer belongs to a recognized ecosystem.
Membership alone does NOT establish authority over any particular
subject namespace.

#### openid_federation {#trust-method-openid-federation}

The `openid_federation` method indicates that the Assertion Issuer is
acceptable if its OpenID Federation {{OIDF-FEDERATION}} trust chain
validates to a listed trust anchor and (when required by the policy)
the leaf holds the required Trust Marks.

~~~ json
{
  "method": "openid_federation",
  "trust_anchors": ["https://federation.example.org"],
  "trust_marks": [
    {
      "id": "https://federation.example.org/marks/loa3",
      "issuer": "https://federation.example.org"
    }
  ]
}
~~~

`trust_anchors`
: REQUIRED. Non-empty JSON array of OpenID Federation trust anchor
entity identifiers.

`trust_marks`
: OPTIONAL. JSON array of Trust Mark requirement objects. When
present, the leaf's Entity Configuration MUST include at least one
Trust Mark satisfying every requirement object listed. Each
requirement object has:

  `id`
  : REQUIRED. String. The Trust Mark identifier.

  `issuer`
  : REQUIRED. String. The Entity Identifier of the Trust Mark
  Issuer expected to have signed the Trust Mark.

Evaluation of this Trust Method delegates to {{OIDF-FEDERATION}} for
the procedural mechanics: trust chain validation
({{OIDF-FEDERATION}} §10), metadata policy application
({{OIDF-FEDERATION}} §6), Trust Mark validation
({{OIDF-FEDERATION}} §7), and entity identifier comparison. The
Resource Authorization Server MUST perform those procedures per
{{OIDF-FEDERATION}}; failure of any of them is failure of this
Trust Method. Cache freshness and revocation follow the rules of
{{OIDF-FEDERATION}}.

In addition to the procedures in {{OIDF-FEDERATION}}, the Resource
Authorization Server MUST apply the following framework-specific
requirements:

1. **Trust anchor match.** The terminal trust anchor of the
   validated chain MUST exactly match one of the listed
   `trust_anchors` values.

2. **Entity type constraint.** The leaf's policy-applied federation
   metadata MUST declare one of the entity types `openid_provider`
   or `oauth_authorization_server` ({{OIDF-FEDERATION}} §5). The
   presence of other entity types alone does not satisfy this
   requirement.

3. **Federation-bound JWKS resolution.** The signing key for the
   identity assertion JWT MUST be resolved from the leaf's
   policy-applied federation metadata (the `jwks` or `jwks_uri`
   value in `metadata.openid_provider` or
   `metadata.oauth_authorization_server`). The Resource
   Authorization Server MUST NOT use a JWKS retrieved outside the
   federation (for example, fetched from the assertion `iss` URL
   via {{RFC8414}} Authorization Server Metadata) unless that
   JWKS exactly matches the federation-resolved JWKS. This
   prevents a downgrade attack in which an attacker compromises
   the AS metadata endpoint without compromising the federation
   infrastructure.

4. **Trust Mark satisfaction.** If the Trust Method object
   contains `trust_marks`, the leaf's Entity Configuration MUST
   include Trust Marks satisfying every requirement object: each
   requirement is satisfied when at least one Trust Mark in the
   leaf's `trust_marks` array matches both `id` and `issuer` and
   validates per {{OIDF-FEDERATION}} §7.

For OpenID Federation deployments, this Trust Method is the primary
integration point between the federation and this framework; see
{{relationship-to-oidf}} for positioning.

### Subject Namespace Authorization Methods {#subject-namespace-authorization-methods}

`subject_namespace_authorization`-category methods are satisfied
by evidence published by the Subject Authority itself (a DNS or
HTTPS record under the Subject Authority's domain). The
namespace-authorization trust graph is bounded to depth one
({{transitive-authz-bounded}}).

#### domain_authorized_issuer {#trust-method-domain-authorized-issuer}

The `domain_authorized_issuer` method indicates that the Assertion
Issuer is acceptable if the Subject Authority identified by the
assertion's Subject Identifier authorizes the Assertion Issuer to
assert identities in that namespace, as defined in {{dii}}.

Domain-Authorized Issuer Discovery, defined in {{dii}}, is the
mechanism by which a Subject Authority publishes its authorized
Assertion Issuers. A Subject Authority publishes a DNS TXT record at
`_oauth-issuer-policy.{authority}` (carrying authorized issuers
inline, or pointing to a richer HTTPS-hosted document) or an HTTPS
JSON document at `.well-known/oauth-issuer-policy` on the Subject
Authority's host. The full mechanism (record syntax, lookup
procedure, failure handling, and security considerations) is
specified in {{dii}}.

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

This Trust Method has no parameters. The Resource Authorization Server
retrieves the Issuer Authorization Policy by applying the canonical
lookup procedure in {{dii-lookup}}: a DNS query at
`_oauth-issuer-policy.{A}` with HTTPS well-known URL fallback when the
DNS response is `negative-authoritative`.

#### https_authorized_issuer {#trust-method-https-authorized-issuer}

The `https_authorized_issuer` method indicates that the Assertion
Issuer is acceptable if the Subject Authority identified by the
assertion's Subject Identifier publishes an HTTPS well-known Issuer
Authorization Policy authorizing the Assertion Issuer.

~~~ json
{
  "method": "https_authorized_issuer"
}
~~~

This Trust Method has no parameters. It reuses the Issuer
Authorization Policy document format ({{dii-document}}), Subject
Authority determination rules ({{dii-authority}}), HTTPS document URL
({{dii-https-url}}), verification rules ({{dii-verification}}), and
caching rules ({{dii-caching}}), but it does not perform the DNS lookup
defined in {{dii-lookup}}.

When evaluated, the Resource Authorization Server MUST:

1. Determine the Subject Authority `A` from the assertion's Subject
   Identifier per {{dii-authority}}. If the format is not registered in
   {{iana-authority-registry}}, reject the assertion.

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

A Resource Authorization Server lists this method when it requires
authority publication via HTTPS only and explicitly does NOT accept
DNS-published authority (inline or DNS pointer). Compared to
`domain_authorized_issuer`, this method:

- Eliminates dependence on DNS resolution integrity for the
  authority lookup. The Subject Authority is determined per
  {{dii-authority}}, but the authorization itself is retrieved
  only over TLS-authenticated HTTPS.

- Rejects the inline DNS form (where the policy is carried in the
  TXT record itself) and the DNS pointer form (where DNS points
  at a third-party HTTPS host). Both require trusting DNS for the
  authoritative selection of either issuers or policy host.

- Forces the Subject Authority to control the apex web origin
  identified by the well-known URL.

A Resource Authorization Server lists `domain_authorized_issuer`
instead when it accepts the DNS forms or the canonical DNS-first
lookup. A Resource Authorization Server MAY list both
`domain_authorized_issuer` and `https_authorized_issuer`; see
{{combining-dai-methods}}.

#### Combining domain_authorized_issuer and https_authorized_issuer {#combining-dai-methods}

`domain_authorized_issuer` and `https_authorized_issuer` are peer
Trust Methods in the `subject_namespace_authorization` category;
within a category, OR-semantics apply ({{trust-method-categories}}).
A Resource Authorization Server that lists both methods accepts an
assertion satisfying EITHER.

Listing both methods produces a weaker combined trust posture than
listing either alone. `domain_authorized_issuer` fails closed under
`indeterminate` DNS outcomes (see {{dii-failures}}); adding
`https_authorized_issuer` to the same Trust Policy creates a path to
acceptance via the direct HTTPS retrieval, defeating that
fail-closed property. An attacker who can drive the DNS lookup to
`indeterminate` (resolver denial-of-service, BGP disruption of the
authoritative nameserver path, or similar) can force the verifier
to fall through to `https_authorized_issuer` evaluation; if the
attacker can additionally substitute the apex HTTPS response
(through DNS-redirect plus TLS misissuance, for example), the
attacker reaches assertion acceptance that the fail-closed
`domain_authorized_issuer` evaluation would have prevented.

Resource Authorization Servers SHOULD therefore list only one of
the two methods. Listing both is appropriate only when the
deployment has explicitly accepted the weaker combined trust model
in exchange for higher availability: a Subject Authority is
reachable when EITHER DNS or apex HTTPS is available, not only
when both are. Deployments that combine both methods SHOULD
mitigate by tightening cache freshness and applying enhanced
local-policy scrutiny.

A Resource Authorization Server that distrusts DNS-published
authority and wants STRICT HTTPS-only authority resolution lists
ONLY `https_authorized_issuer` and omits `domain_authorized_issuer`.

A Resource Authorization Server that accepts DNS-published
authority lists ONLY `domain_authorized_issuer`; the canonical
lookup procedure already falls back to the HTTPS well-known URL on
`negative-authoritative` DNS responses, so the "no DNS record,
only HTTPS document" Subject Authority case is already covered.

#### email_verification_dns {#trust-method-email-verification-dns}

The `email_verification_dns` method indicates that the Assertion
Issuer is acceptable if the Subject Authority of the assertion's
`email` Subject Identifier publishes a DNS TXT record per the Email
Verification Protocol {{WICG-EMAIL-VERIF}} naming the Assertion
Issuer. This Trust Method exists to interoperate with deployments
that already publish a `_email-verification.{domain}` record; it
lets a Resource Authorization Server honor those records without
requiring the Subject Authority to also publish an
`_oauth-issuer-policy` record.

~~~ json
{
  "method": "email_verification_dns"
}
~~~

This Trust Method has no parameters.

When evaluated, the Resource Authorization Server MUST:

1. Verify that the assertion's Subject Identifier format is `email`.
   If not, the Trust Method is not satisfied by this assertion.

2. Determine the Subject Authority `A` from the `email` Subject
   Identifier per the `email` extraction procedure in
   {{dii-authority}} (registrable domain after `@`).

3. Query the DNS TXT resource record set at `_email-verification.{A}`.
   Classify the DNS response as follows:

   `negative-authoritative`
   : NXDOMAIN, or NOERROR with an empty answer section or with no
   records parseable as Email Verification Protocol records after
   parsing per {{WICG-EMAIL-VERIF}}.

   `indeterminate`
   : SERVFAIL, REFUSED, timeout, truncation with no successful retry,
   or any other failure that prevents a definitive negative result.

   `affirmative`
   : NOERROR with at least one record parseable as an Email
   Verification Protocol record after parsing per
   {{WICG-EMAIL-VERIF}}.

4. If `negative-authoritative` or `indeterminate`, the Trust Method
   is not satisfied; the `indeterminate` outcome MUST be treated per
   the failure-handling rules of {{dii-failures}} (a verifier MUST
   reject the assertion unless a fresh cached result is available).

5. If `affirmative`, parse each record per {{WICG-EMAIL-VERIF}} and
   collect the `iss=` values. The set of authorized issuer
   identifiers is the union of those values.

6. Compare the JWT `iss` claim to the parsed `iss=` values.
   {{WICG-EMAIL-VERIF}} uses bare hostnames (for example,
   `issuer.example`) while OAuth issuer identifiers are HTTPS URLs
   that MAY carry a path. The Resource Authorization Server MUST
   parse the JWT `iss` as an absolute HTTPS URL and treat the
   record's bare hostname `H` as matching only when the JWT `iss`
   has scheme `https`, host equal to `H` (case-insensitive ASCII
   after IDNA conversion per {{RFC5891}}), default port (or no port
   component), and an empty path (or the path `/`). Any other shape
   (non-default port, non-empty path component, query string, or
   fragment) does not match. Consequently, this Trust Method is
   interoperable only with Assertion Issuers whose identifier is
   the bare-origin form `https://{H}` (or `https://{H}/`).
   Deployments whose Assertion Issuer identifiers carry paths (for
   example, `https://login.example/{tenant}`) cannot be authorized
   by this Trust Method; they SHOULD use `domain_authorized_issuer`
   ({{trust-method-domain-authorized-issuer}}), which expresses
   full-URL issuer identifiers in the `authorized_issuers[].issuer`
   field.

7. Apply the cross-category combination rule of {{rasp}}.

A Subject Authority MAY publish both an `_email-verification` record
and an `_oauth-issuer-policy` record. When a Resource Authorization
Server's trust policy lists both `domain_authorized_issuer` and
`email_verification_dns` as `subject_namespace_authorization` Trust
Methods, the cross-category combination rule of {{rasp}} treats them
as alternatives within the same category (OR-semantics): satisfying
either is sufficient.

## Critical Extension Identifiers {#critical-extension-identifiers}

A critical extension identifier names a feature whose correct
processing is required to safely interpret a policy document or
DNS record. Used by the `crit` JSON member and the `crit=` DNS
directive. A critical extension identifier is recognized when it
is either a member or directive name defined by this document, or
an absolute URI whose defining specification establishes it as a
critical extension identifier and defines the required processing.
Future specifications SHOULD use absolute URI identifiers when
correct processing depends on more than recognizing a single
member or directive.

Example: a policy that depends on a future audience-constraints
extension lists the identifier in `crit` and adds the
extension-defined members. Consumers that do not recognize the URI
reject the policy rather than ignoring `audiences` and
over-accepting:

~~~ json
{
  "subject_authority": "example.com",
  "crit": [
    "https://example.net/oauth-issuer-policy/audience-constraints"
  ],
  "authorized_issuers": [
    {
      "issuer": "https://idp.example.com",
      "audiences": ["https://api.resource.example"]
    }
  ]
}
~~~

## Signed Policy Metadata {#signed-policy-metadata}

The Trust Policy and Issuer Authorization Policy documents MAY include
a `signed_policy` member that provides object-level cryptographic
integrity for the policy document. This member follows the signed
metadata pattern defined for authorization server metadata in
{{RFC8414}} and protected resource metadata in {{RFC9728}}.

The `signed_policy` value is a JWT {{RFC7519}} containing policy
members as claims. The JWT MUST be digitally signed or MACed using JWS
{{RFC7515}}, MUST contain an `iss` claim identifying the party
attesting to the signed policy claims, and MUST NOT use `alg` value
`none`. The JWT payload SHOULD NOT contain a `signed_policy` claim.

The acceptable signer depends on which policy document is signed:

- For the Trust Policy document, the JWT `iss` claim MUST equal the
  `resource_authorization_server` claim. The verification key MUST be
  controlled by that Resource Authorization Server. Consumers MAY
  resolve the key from the Resource Authorization Server's
  authorization server metadata `jwks_uri`, federation entity
  configuration, or local configuration.

- For the Issuer Authorization Policy document, the JWT payload MUST
  contain the `subject_authority` claim. The JWT `iss` claim MUST
  either equal `subject_authority` or identify a signing authority that
  local policy or an applicable Trust Method establishes as controlled
  by the Subject Authority. Consumers MUST NOT treat a signature by the
  Assertion Issuer named in an `authorized_issuers` entry as proof of
  Subject Authority authorization unless such a relationship is
  explicitly established.

If both unsigned policy members and `signed_policy` are present, the
signed policy claims MUST be used as the policy values for all claims
present in the JWT. Unsigned members that are not represented as claims
in the JWT MAY be used subject to the normal processing rules for
unrecognized members and `crit`. Consumers SHOULD reject a policy when
an unsigned member conflicts with the corresponding signed claim.

If a consumer requires object-level integrity by local policy, or if
`signed_policy` is listed in `crit`, the consumer MUST verify the
signed JWT before acting on the policy. If signature verification
fails, if the verification key is unacceptable, if the JWT is
malformed, or if the required issuer binding above is not satisfied,
the consumer MUST reject the policy as malformed.

# Trust Policy Processing

The trust policy governs whether an Assertion Issuer's identity
assertion is acceptable to the Resource Authorization Server. It does
not, by itself, authorize any particular access, scope, role, or
attribute. The Resource Authorization Server's local policy continues
to determine, for an accepted assertion, which scopes are granted,
which subject claims are honored for account linking, and which local
authorization decisions follow. The trust policy is necessary but not
sufficient: passing trust policy evaluation only means the Resource
Authorization Server is willing to consider the assertion as input to
its access-control logic.

## Client Processing

A client that wants to obtain an identity assertion JWT for a Resource
Authorization Server:

1. Discovers the Resource Authorization Server metadata {{RFC8414}}.

2. Confirms that the desired grant profile is supported by checking
   the `authorization_grant_profiles_supported` parameter
   ({{ID-JAG}} §7.2) in the authorization server's metadata.

3. Fetches the `identity_assertion_trust_policy_uri` document.

4. Determines whether an available Assertion Issuer can satisfy the
   Trust Method combination rule in {{rasp}}. For namespace-bound
   Subject Identifiers, this usually means satisfying both an
   `issuer_authentication` method (such as federation membership) and
   a `subject_namespace_authorization` method (such as authorization
   by the Subject Authority).

5. Determines whether the Assertion Issuer can produce a Subject
   Identifier in a format listed in
   `subject_identifier_formats_supported`, if that member is present,
   and required by the applicable grant profile.

6. Requests an identity assertion JWT from the selected Assertion
   Issuer.

The client MUST NOT treat the trust policy as a guarantee that a
particular assertion will be accepted. The Resource Authorization
Server always applies local policy at token request time.

## Resource Authorization Server Trust Evaluation Model {#ras-evaluation-model}

A Resource Authorization Server processing an identity assertion
answers six questions. Each MUST be answered affirmatively; failure
of any one rejects the assertion. {{rasp}} specifies the normative
procedure; this section names the questions so implementations
remain compatible at the conceptual level regardless of internal
ordering.

1. **Is the assertion well-formed and fresh?** Grant-profile
   validation: signature, `aud`, `exp`, `iat`, replay protection.
   This framework does not replace these checks.

2. **Is the Assertion Issuer authentic?** Answered by an
   `issuer_authentication` Trust Method
   ({{issuer-authentication-methods}}).

3. **Is the assertion's Subject Identifier format accepted?** The
   format MUST appear in `subject_identifier_formats_supported`
   when that member is present.

4. **Is the Assertion Issuer authoritative for this subject's
   namespace?** Answered by a `subject_namespace_authorization`
   Trust Method ({{subject-namespace-authorization-methods}}). The
   authorization unit is the (issuer, tenant) pair when the matched
   `authorized_issuers` entry carries a `tenant` value
   ({{dii-multi-tenant}}); otherwise it is the issuer alone.

5. **Is the authorization still valid?** `valid_from`/`valid_until`
   on the matched entry and cache freshness ({{dii-caching}},
   {{caching}}) together bound the temporal scope.

6. **Does local policy permit this use?** Account-linking, consent,
   scope, risk, and other access-control logic remain
   local-policy decisions.

Questions 2 and 4 are independent: federation membership does NOT
establish namespace authority, and namespace authorization does NOT
establish issuer authenticity. The cross-category combination rule
in {{rasp}} enforces this independence. See
{{transitive-authz-bounded}}.

Implementations MAY use a different internal order from {{rasp}} as
long as the answer to every applicable question is determined and
rejection occurs on the first negative answer.

## Resource Authorization Server Processing {#rasp}

This subsection specifies the normative processing order for the
trust evaluation described conceptually in {{ras-evaluation-model}}.
When evaluating an identity assertion JWT presented in a token
request, the Resource Authorization Server MUST:

1. Select and parse the trust policy that applies to the token request.
   If the policy document is malformed or no recognized Trust Method is
   usable, reject the assertion.

2. Validate the assertion according to the applicable grant profile,
   including `iss`, signature, `aud`, `exp`, `iat`, replay protection,
   and any client binding requirements defined by the profile.

3. Verify that the applicable grant profile is listed in
   `authorization_grant_profiles_supported`.

4. If `subject_identifier_formats_supported` is present, verify that
   the Subject Identifier format used by the assertion is listed,
   unless the applicable grant profile defines a different format
   selection rule.

5. Verify that the Assertion Issuer identified by `iss` satisfies the
   Trust Method combination rule:

   a. Partition the recognized Trust Method objects in
      `issuer_trust_methods` by category (see
      {{trust-method-categories}}). A Trust Method registered with
      multiple categories belongs to each.

   b. For each category that is BOTH present in the partitioned policy
      AND applicable to the assertion, the Assertion Issuer MUST
      satisfy at least one Trust Method from that category. An
      `issuer_authentication` method applies unconditionally. A
      `subject_namespace_authorization` method applies when the
      assertion carries a Subject Identifier.

   c. If the assertion carries a Subject Identifier and the policy
      contains a recognized `subject_namespace_authorization` Trust
      Method, but no Subject Authority can be determined for the
      assertion per {{dii-authority}} or an equivalent registry, the
      Resource Authorization Server MUST reject the assertion. The
      Resource Authorization Server MUST NOT treat the
      `subject_namespace_authorization` category as inapplicable in
      this case.

   d. If no Trust Method object in the policy is both recognized,
      present in a category, and applicable to the assertion, the
      policy provides no trust evaluation basis for this assertion;
      the Resource Authorization Server MUST reject the assertion.
      This case can arise when, for example, the policy lists only
      `subject_namespace_authorization` methods and the assertion
      does not carry a Subject Identifier.

   e. Local policy MAY require satisfaction of additional Trust
      Methods for specific clients, subjects, or scopes.

   When the policy lists only `issuer_authentication` methods and the
   assertion carries a namespace-bound Subject Identifier, the
   Resource Authorization Server SHOULD reject the assertion or apply
   a stricter local policy, since federation membership alone does not
   establish authority over a particular subject namespace.

6. Apply account-linking, consent, authorization, risk, and local
   business policy. Client authentication, sender-constraining, and
   client identifier resolution are governed by the applicable
   grant profile and OAuth client-authentication mechanisms; they
   are not specified by this trust framework.

Failure to satisfy issuer trust, subject identifier, or assertion
claim requirements in the trust policy MUST result in an OAuth
`invalid_grant` error unless another error is defined by the
applicable grant profile. For administrative diagnostics, the
Resource Authorization Server SHOULD record the trust-evaluation
failure reason ({{diagnostic-reasons}}) in its audit log; that
reason MUST NOT be returned to public clients in the OAuth error
response.

## Trust Evaluation Diagnostic Reasons {#diagnostic-reasons}

Resource Authorization Servers SHOULD use the following structured
reason codes for trust-evaluation failures in logs, monitoring,
audit, and signed introspection responses to authenticated callers.
The codes MUST NOT be returned to public clients in the OAuth
error response and SHOULD NOT appear in `error_description` for
unauthenticated callers: detailed trust-evaluation state is a
recon target for adversaries probing for misconfiguration.

| Reason Code | Meaning |
|-|-|
| `issuer_not_authenticated` | The Assertion Issuer did not satisfy any `issuer_authentication`-category Trust Method (no federation chain validated, no recognized signer). |
| `issuer_not_authorized_for_subject_authority` | The Assertion Issuer satisfied authentication but is not listed by the Subject Authority's Delegation Artifact for the asserted Subject Identifier's namespace. |
| `subject_authority_policy_not_found` | The Subject Authority Extraction succeeded but no Delegation Artifact (DAI policy) was located through any configured publication channel; the lookup state was Negative. |
| `subject_authority_policy_indeterminate` | The Delegation Artifact lookup returned Indeterminate (DNS SERVFAIL, HTTPS 5xx, malformed response, signature failure). Distinct from `_policy_not_found`. |
| `subject_authority_policy_malformed` | A Delegation Artifact was retrieved but failed structural validation (parsing, schema, signature binding). |
| `tenant_binding_required` | The Delegation Artifact's authorized entry requires a tenant claim that the assertion does not carry. |
| `tenant_binding_mismatch` | The assertion carries a tenant claim that does not match the value required by the matched authorized entry. |
| `subject_identifier_format_not_authorized` | The matched authorized entry does not include the Subject Identifier format of the asserted subject in its `subject_identifier_formats`. |
| `policy_expired` | The Delegation Artifact's `valid_until` (or equivalent) is in the past at evaluation time. |
| `policy_not_yet_valid` | The Delegation Artifact's `valid_from` (or equivalent) is in the future at evaluation time. |
| `trust_method_not_satisfied` | A specific `issuer_trust_methods` entry was applicable but its Trust-Method-specific verification procedure failed for reasons not captured by a more specific code above. |
| `applicability_bypass_attempted` | The assertion was constructed to drive the Validator toward waiving an applicable category (see Authority Delegation Framework §Applicability Bypass); the Validator rejected. |

Additional reason codes for future Trust Methods are registered
in {{iana-diagnostic-reasons-registry}}. Resource Authorization
Servers SHOULD log one reason code per failed evaluation,
selecting the code for the EARLIEST checkpoint that failed.

# Grant Profile and Token Bindings {#bindings}

This section defines bindings to ID-JAG ({{id-jag-profile}}) and
the OAuth Actor Profile ({{actor-profile-binding}}).

## ID-JAG {#id-jag-profile}

This section provides the binding for ID-JAG {{ID-JAG}}; other grant
profiles would supply analogous bindings, naming their grant profile
identifier, their Subject Identifier-bearing claim, and any
profile-specific JWT claims that participate in Trust Method
evaluation.

For ID-JAG {{ID-JAG}}, `authorization_grant_profiles_supported` contains the value
`urn:ietf:params:oauth:grant-profile:id-jag`. When the trust policy
requires subject identifier evaluation, the Resource Authorization
Server applies the policy to the email attribution carried by the
ID-JAG. The email is taken from the top-level `email` claim,
accompanied by `email_verified=true`, per {{dii-authority}}.

In addition to the processing in {{rasp}}, the Resource Authorization
Server MUST:

1. Validate the ID-JAG according to {{ID-JAG}}, including `iss`,
   signature, `aud`, `exp`, `iat`, `jti`, and `sub`.

2. Verify that the ID-JAG `aud` identifies the Resource Authorization
   Server.

3. Verify that the email attribution carried by the ID-JAG, when
   present or required, uses a Subject Identifier format registered
   in `subject_identifier_formats_supported`, if that trust policy
   member is present.

4. Use the email attribution only for subject resolution and Subject
   Authority evaluation. The Resource Authorization Server MUST NOT
   treat the mere presence or value of the email claim as proof
   that the ID-JAG issuer is authoritative for the subject; issuer
   acceptability is established only by evaluating the Trust Methods.

If the `domain_authorized_issuer` Trust Method is used with ID-JAG,
the Subject Authority is determined from the top-level
`email`/`email_verified` claims per {{dii-authority}} and the
Resource Authorization Server validates the delegation using the
procedure in {{dii}}.

## OAuth Actor Profile {#actor-profile-binding}

The OAuth Actor Profile {{ACTOR-PROFILE}} defines a standardized
representation of delegated actors carried in identity assertions, JWT
access tokens, and Transaction Tokens through an `act` object with
members including `act.iss`, `act.sub`, and (optionally)
`act.sub_profile`. The actor profile establishes that the token
issuer's authority to assert a specific `(act.iss, act.sub)` actor
identifier pair is not automatic and must be evaluated against a
trust framework, but defers the framework itself to local policy or
bilateral agreement.

This section binds the Trust Method machinery of this document
to that framework. The binding applies when a presented assertion or
token carries an `act` object and the Resource Authorization Server's
trust policy is in scope for the assertion's grant profile.

When evaluating such an assertion, in addition to the processing in
{{rasp}}, the Resource Authorization Server MUST:

1. Treat the actor as a namespace-bound subject when `act.sub` carries
   a Subject Identifier format with a registered Subject Authority
   extraction procedure ({{iana-authority-registry}}) whose registry
   entry applies to actor carriage. The format MAY be carried in
   `act.sub_profile` or be evident from the structure of `act.sub`;
   in either case the determination uses the registered extraction
   procedure. See {{actor-coverage}} for the scope and limitations
   of this document's registered extractions when applied to actor
   identifiers.

2. If the trust policy lists one or more
   `subject_namespace_authorization` Trust Methods, evaluate at least
   one such method with `act.sub` substituted for the Subject
   Identifier and the assertion's top-level `iss` (the token issuer)
   substituted for the Assertion Issuer being authorized. This
   answers the question "is this token issuer entitled to assert this
   actor identifier?"

3. Apply the cross-category combination rule of {{rasp}} unchanged.
   `issuer_authentication` methods evaluate the token's `iss` and are
   not affected by the presence of an actor object.

When `act.iss` and `iss` are the same string, the actor profile
treats the token as not expressing delegation. The
`subject_namespace_authorization` evaluation in step 2 still applies
to `act.sub`: an Assertion Issuer asserting an actor in its own
claimed namespace MUST still appear in the Issuer Authorization
Policy for that namespace if a policy exists, otherwise self-issued
actor assertions cannot be evaluated and MUST be rejected unless
local policy explicitly accepts them, consistent with {{ACTOR-PROFILE}}.

When `act.iss` and `iss` differ, the assertion expresses
cross-namespace delegation. The Resource Authorization Server MUST
NOT substitute `act.iss` for `iss` in any Trust Method evaluation;
trust in an Assertion Issuer is always trust in the entity that
signed the token. `act.iss` is interpreted only as a namespace-context
identifier for the actor, consistent with {{ACTOR-PROFILE}}.

This binding establishes only issuer trust for the actor identity.
Whether the actor identified by `(act.iss, act.sub)` is permitted to
act on behalf of the token's subject for a specific scope or
resource remains a local authorization decision, consistent with the
scope rules of {{ACTOR-PROFILE}}.

### Coverage of Actor Identities {#actor-coverage}

Trust evaluation of an actor's identity by this framework's
`subject_namespace_authorization` category is supported only when
`act.sub` uses a Subject Identifier format with a registered Subject
Authority Extraction Procedure ({{iana-authority-registry}}) AND
that procedure's registry entry applies to actor carriage (the
"Applies To" column in the registry).

This document's only registered extraction (`email`,
{{dii-authority}}) applies to direct Subject Identifiers only; it
does NOT apply to `act.sub`. As a consequence, this framework
currently provides NO `subject_namespace_authorization`-category
trust evaluation for actor identities. Actor identity trust is
governed entirely by:

- Local policy at the Resource Authorization Server, AND
- Any trust framework established between the Resource Authorization
  Server and the actor's claimed namespace outside the scope of this
  document.

This is a documented gap. A future specification MAY register an
actor-applicable extraction procedure (for example, an actor-email
extraction that defines how actor-email verification is carried in
the assertion's claim shape) to close it. Deployments that need
wire-format-enforced trust evaluation for actor identities require
that future work; this document does not provide such a mechanism.

# Domain-Authorized Issuer Discovery {#dii}

Domain-Authorized Issuer Discovery is one mechanism in the
`subject_namespace_authorization` Trust Method category. It applies
when the Subject Authority can be expressed as a domain that controls
DNS, and reuses the DNS-based authority pattern operators already use
for CAA, MTA-STS, SPF, and DKIM ({{dns-authority-patterns}}). The
Subject Identifier formats currently registered for this mechanism
resolve to domain-form Subject Authorities. The initial registered
format is `email`, which extracts to the registrable domain
({{dii-authority}}). Future Trust Methods in the
`subject_namespace_authorization` category MAY define alternative
publication and lookup mechanisms for namespaces whose authority is
not a domain; see {{extending}}.

A Subject Authority publishes an Issuer Authorization Policy
that authorizes one or more Assertion Issuers to assert identities in
the Subject Authority's namespace. The same policy answers two related
questions about a subject namespace such as an email domain:

1. **Verification.** Given an identity assertion, is this issuer
   authorized to assert identities for this subject?

2. **Assertion Issuer discovery.** Given a subject identifier,
   which authorization server should I direct the user (or my own
   request) to in order to obtain an assertion?

Verification is defined in {{dii-verification}}. Assertion Issuer
discovery is defined in {{dii-hrd}}. Both consumers use the same records
and the same document; the difference is the direction of use.

Domain-Authorized Issuer Discovery is non-transitive by design. The
Issuer Authorization Policy is a flat list of Assertion Issuer
identifiers. It cannot express "delegate further to whoever Issuer X
federates" or other chained delegations of namespace authority; a
Subject Authority that wants to authorize a new Assertion Issuer
publishes that Assertion Issuer's identifier directly. The reasoning,
threat model, and consequences of this design choice are documented
in {{transitive-authz-bounded}}.

Domain-Authorized Issuer Discovery is also independent of OpenID
Federation: a Subject Authority can publish DAI records and a
Resource Authorization Server can evaluate them without any
federation infrastructure in either party. See
{{relationship-to-oidf}} for how the two mechanisms compose when
both are present.

This section is structured as a self-contained module so that it can
be extracted into a separate specification in the future without
breaking changes to existing implementations. Section anchors use the
`dii-` prefix to make extraction mechanical.

## Relationship to the Email Verification Protocol {#dii-prior-art}

This subsection is non-normative.

Domain-Authorized Issuer Discovery follows the DNS authority-publication
pattern summarized in {{dns-authority-patterns}}: the holder of a DNS
namespace publishes security policy for that namespace at a predictable
owner name, optionally pointing to richer HTTPS policy content.

The `email_verification_dns` Trust Method
({{trust-method-email-verification-dns}}) is included for compatibility
with the Email Verification Protocol {{WICG-EMAIL-VERIF}}, which uses
the same DNS-authority pattern at a different owner name
(`_email-verification.{domain}`) and with a different record format.
Because that protocol is a non-IETF specification still under active
development, interoperability for `email_verification_dns` depends on
the stability of {{WICG-EMAIL-VERIF}}; implementations should track
that specification for breaking changes.

## Subject Authority Determination {#dii-authority}

The Subject Authority associated with a Subject Identifier is
determined as registered in {{iana-authority-registry}}. This
document registers the `email` extraction as the initial entry. The
extraction-procedure pattern (Subject Identifier format to Subject
Authority) is open: future specifications register additional
formats for service identities, decentralized identifiers,
federated handles, and other namespaces by adding entries to the
same registry without changes to the rest of the framework. See
{{extending}}.

Initial extractions:

`email`
: The Subject Authority is derived from the assertion's top-level
  `email` claim, which MUST be accompanied by a top-level
  `email_verified` claim with the boolean value `true`. If the
  `email_verified` claim is absent or has any value other than
  `true`, consumers MUST treat the `email` Subject Identifier as
  invalid for purposes of this Trust Method and MUST reject the
  assertion when `email` is the Subject Identifier being evaluated.
  Consumers MUST NOT treat an unverified `email` claim as though the
  assertion carried no Subject Identifier.

  Consumers MUST reject an `email` claim value that does not contain
  exactly one `@` character or whose domain part is empty. The
  local-part is not used. The domain is then normalized to the
  registrable domain ("eTLD+1") using a current Public Suffix List
  {{PSL}}: the Subject Authority is the shortest suffix of the
  email's domain that is not itself a public suffix. The result is
  converted to A-label form per {{RFC5891}} and compared using
  case-insensitive ASCII comparison. Normalization to the
  registrable domain is consistent with the email-domain handling
  in the Email Verification Protocol {{WICG-EMAIL-VERIF}} and
  prevents an attacker who controls a subdomain (for example, via
  subdomain takeover) from publishing a Subject Authority record
  that would override the legitimate record at the registrable
  domain. Consumers MUST reject an email whose domain is itself a
  public suffix (no registrable domain exists).

Subject Identifier formats not registered for this purpose MUST NOT
be evaluated under this Trust Method; the Resource Authorization
Server MUST reject the assertion with an OAuth `invalid_grant`
error.

### Subdomain Authority and the Registrable-Domain Default {#subdomain-authority}

The registrable-domain default for `email` is intentionally
conservative: it prevents a party that controls only a subdomain
(through legitimate operational delegation or via subdomain
takeover) from publishing Subject Authority records that override
the organizational policy at the registrable domain. This is the
right default for the open-world case where a Resource
Authorization Server has no a priori knowledge of which
subdomains are administered separately within a namespace.

Enterprise deployments often operationally delegate identity
administration at subdomain boundaries, including patterns such
as:

- `students.university.edu` administered separately from the
  university's faculty/staff identity stack;
- `contractors.company.com` administered by a vendor management
  team distinct from the parent's IdP team;
- `partners.example.com` issued through a partner-federation
  service;
- `agency.department.gov` administered by an agency subordinate to
  a departmental parent;
- `division.example.co.uk` administered by a business unit with its
  own identity infrastructure.

These deployments cannot be modeled by the registrable-domain
extraction alone. The architectural mechanism for supporting them
is the Subject Authority Extraction Procedures Registry
({{iana-authority-registry}}): a future specification registers an
extraction procedure that derives the Subject Authority from the
exact email domain (for example, an `email_domain_exact`
extraction), and Resource Authorization Servers that accept that
extraction can recognize subdomain-level authority.

A subdomain-exact extraction MUST require explicit parent
authorization to avoid the subdomain-takeover risk the
registrable-domain default protects against. Concretely, a
specification registering such an extraction MUST specify how the
parent registrable-domain authority delegates to a subdomain
authority, for example by publishing a delegation record at the
parent's Subject Authority publication channel that names the
authorized subdomain authorities. Without parent delegation, a
subdomain-takeover attacker would otherwise be able to publish a
competing policy at the subdomain.

This document does not register a subdomain-exact extraction. The
registrable-domain default is the only `email` extraction defined
here; deployments needing subdomain-level authority SHOULD use the
registrable-domain default with the parent as Subject Authority
until a subdomain-exact extraction is registered or use
out-of-band arrangements with the parent's identity team. A
non-normative sketch of how `email_domain_exact` could be defined
is provided in {{extending}}.

## Issuer Authorization Policy Document {#dii-document}

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

The Issuer Authorization Policy is a JSON document served over HTTPS
with media type `application/json`. The document MUST be a JSON
object. Consumers MUST reject a policy document that is not
syntactically valid JSON, whose top-level value is not an object, or
whose required members are absent or have the wrong JSON type. It has
the following members:

`subject_authority`
: REQUIRED. String. The Subject Authority identifier this policy
  applies to. Consumers MUST verify this value matches the Subject
  Authority computed in {{dii-authority}} after applying the
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
  as valid before this time. Consumers MAY apply a small clock skew
  tolerance (no more than 5 minutes) when comparing against the
  current time, consistent with conventions for JWT `nbf`
  ({{RFC7519}} Section 4.1.5).

  `valid_until`
  : OPTIONAL. RFC 3339 date-time. The delegation MUST NOT be treated
  as valid at or after this time. Consumers MAY apply a small clock
  skew tolerance (no more than 5 minutes) when comparing against the
  current time.

`last_updated`
: OPTIONAL. RFC 3339 date-time at which the policy was last published.

`crit`
: OPTIONAL. JSON array of strings. Each string names an extension
identifier, feature identifier, or policy member whose recognition the
publisher considers critical for correct interpretation. If a consumer
does not recognize any string listed in `crit`, the consumer MUST treat
the policy as `malformed` ({{dii-failures}}). Member names defined in
this document (`subject_authority`, `authorized_issuers`, `issuer`,
`tenant`, `subject_identifier_formats`, `valid_from`, `valid_until`,
`last_updated`, `signed_policy`, and `crit`) are always recognized. Future
specifications SHOULD use stable extension identifiers in `crit` when
correct processing depends on more than recognizing a single member,
for example when an extension defines several members or changes the
authorization decision procedure. Critical extension identifiers use
the syntax and recognition rules in {{critical-extension-identifiers}}.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, using the
  representation defined in {{signed-policy-metadata}}. This member
  follows the signed metadata pattern used by {{RFC8414}} and
  {{RFC9728}}. Consumers that do not support signed policy metadata MAY
  ignore this member unless it is listed in `crit`.

Unrecognized members MUST be ignored, except when the member name or
an extension identifier governing the member is listed in `crit`.
Unrecognized members that are not listed in `crit` MUST NOT affect the
authorization decision. A consumer MUST NOT assign vendor-specific
semantics to an unrecognized member unless those semantics are defined
by an extension identifier that the consumer recognizes.
Consumers MUST reject the policy when any of the following validation
rules fails:

| Member | Required type / structure | Additional rule |
|-|-|-|
| `subject_authority` | string, required | matches the computed Subject Authority |
| `authorized_issuers` | non-empty array of objects, required | each element has `issuer` |
| `issuer` | string, required | syntactically valid issuer identifier for the applicable grant profile |
| `tenant` | string, optional | non-empty |
| `subject_identifier_formats` | array of strings, optional | |
| `valid_from`, `valid_until`, `last_updated` | RFC 3339 date-time, optional | |
| `signed_policy` | string, optional | signed JWT as defined in {{signed-policy-metadata}} |
| `crit` | array of strings, optional | every value recognized |

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

## Publication

A Subject Authority publishes the Issuer Authorization Policy
through one of three publication profiles defined in
{{publication-profiles}}. The DNS record at
`_oauth-issuer-policy.{A}` is the entry point in all three
profiles, following the pattern of CAA, MTA-STS, SPF, and DKIM
({{dns-authority-patterns}}); the DNS record's mode determines
which profile applies.

### Publication Profiles {#publication-profiles}

This document defines three publication profiles. A Subject
Authority chooses one based on operational constraints; all three
produce semantically equivalent policy validated by the same
lookup procedure ({{dii-lookup}}).

| Profile | DNS form | Document | Authority binding | When to use |
|-|-|-|-|-|
| 1: DNS-Inline | TXT with `issuer=` ({{dii-dns-record}}) | None (carried in TXT) | DNS control of `{A}` | Common case: authorize an issuer for a namespace with no rich policy |
| 2: Authority-Hosted HTTPS | TXT with `uri=` | HTTPS-hosted JSON under Subject Authority's operational control | DNS control of `{A}` AND TLS on the authority-operated host | Rich policy (validity windows, format restrictions, tenant binding) at a controlled origin |
| 3: Signed Policy at Arbitrary Origin {#publication-profile-signed} | TXT with `uri=` | HTTPS-hosted JSON with `signed_policy` ({{signed-policy-metadata}}); MAY be at any HTTPS origin | DNS control of `{A}` AND JWS signature binding the signer to the Subject Authority | Third-party hosting, CDN mirroring, disaster recovery, transparency-log integration |

A Subject Authority using Profile 3 MUST publish `signed_policy`
in the served document satisfying the binding rules of
{{signed-policy-metadata}}. Consumers retrieving a document from
a host NOT under the Subject Authority's DNS-namespace control
MUST verify `signed_policy` before relying on the document's
contents. Profiles 1 and 2 do not require `signed_policy` because
the authority binding is established by channel control (DNS
plus, for Profile 2, TLS on the authority-operated host).

### Default HTTPS Well-Known URL

A Subject Authority MAY additionally publish a JSON document at
the default HTTPS well-known URL on its own host
({{dii-https-url}}). The lookup procedure ({{dii-lookup}})
consults DNS first and uses the HTTPS well-known URL only as a
fallback when the DNS response is `negative-authoritative`. This
fallback is independent of the three publication profiles above;
it covers the case where a Subject Authority has no DNS record
but does have well-known HTTPS infrastructure.

Operators are encouraged to publish the DNS record in all
deployments because it matches the operational model of the
prior-art mechanisms in {{dns-authority-patterns}} and because
consumer lookup behavior is DNS-first.

### DNS Record {#dii-dns-record}

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
- Unrecognized directives MUST be ignored unless listed in a `crit=`
  directive.

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
  case-insensitive ASCII comparison rules in {{dii-authority}}.
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

`crit=REQUIRED_EXTENSIONS`
: OPTIONAL. Comma-separated, case-insensitive list of directive names,
extension identifiers, or feature identifiers declared by the publisher
as critical for correct interpretation of this record. Directive names
defined by this document (`authority`, `uri`, `issuer`, and `crit`) are
always recognized. If a consumer does not recognize any value named in
`crit=`, the consumer MUST treat the record as `malformed` (see
{{dii-failures}}). If a value names a directive, that directive MUST be
present in the same record; otherwise the consumer MUST treat the
record as `malformed`. Future specifications SHOULD use stable
extension identifiers in `crit=` when correct processing depends on
more than recognizing a single DNS directive. Critical extension
identifiers use the syntax and recognition rules in
{{critical-extension-identifiers}}.

A recognized record MUST contain at least one `uri=` directive or at
least one `issuer=` directive. Recognized records containing neither
MUST be treated as malformed (see {{dii-failures}}).

A Subject Authority MUST publish records for at most one version of
this mechanism at a time. This document defines only
`oauth-issuer-policy1`. Future versions that are not
understood by a consumer are ignored by the recognition rule above.

### HTTPS Document URL {#dii-https-url}

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

### HTTPS Policy Document Contract {#https-policy-document-contract}

The document retrieved from either the default well-known URL or a
DNS `uri=` pointer is the Issuer Authorization Policy defined in
{{dii-document}}; consumers MUST NOT interpret any other JSON shape
as an Issuer Authorization Policy. The same member rules apply
whether members appear unsigned or inside `signed_policy`. Consumers
MUST validate the complete policy as a unit; a malformed entry or
failed `crit` requirement makes the whole policy malformed, not
just the offending entry.

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
  "last_updated": "2026-05-01T00:00:00Z",
  "crit": [
    "https://example.net/oauth-issuer-policy/audience-constraints"
  ],
  "signed_policy": "eyJ..."
}
~~~

## Lookup Procedure {#dii-lookup}

To retrieve the Issuer Authorization Policy for a Subject Authority `A`,
a consumer performs the following steps. The procedure is shared by
Resource Authorization Servers performing verification
({{dii-verification}}) and clients performing Assertion Issuer
discovery ({{dii-hrd}}); both consult DNS first and fall back to the
HTTPS well-known URL only on `negative-authoritative` outcomes.

This is the canonical procedure used by the
`domain_authorized_issuer` Trust Method. Other Trust Methods in the
`subject_namespace_authorization` category MAY define alternative
retrieval procedures; in particular, `https_authorized_issuer`
({{trust-method-https-authorized-issuer}}) skips DNS entirely and
retrieves only from the HTTPS well-known URL.

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
      {{dii-authority}}. If any recognized record is missing an
      `authority=` directive, treat the response as `malformed`. If
      all recognized records are discarded because of `authority=`
      mismatch, treat the response as `malformed`.

      Before continuing, validate the remaining recognized records
      against the directive rules in {{dii-dns-record}}. This includes
      rejecting malformed `uri=`, `issuer=`, or `crit=` values,
      records whose `crit=` lists an unknown or absent directive, and
      records with neither `uri=` nor `issuer=`. Any such condition is
      a `malformed` outcome.

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

4. If the DNS response is `indeterminate`, discovery has failed.
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

### Failure Handling {#dii-failures}

DAI lookup outcomes map onto the Affirmative / Negative /
Indeterminate states of the Authority Delegation Pattern's
Exception-Handling and Fallback Model
({{AUTHORITY-DELEGATION}} §exception-handling). The normative
requirements (fail closed on Negative and Indeterminate; no
fallback to a different Authority Source; bounded cache reuse on
transient Indeterminate) apply unchanged; this section maps the
concrete DAI outcomes onto those states.

| State | DAI outcomes |
|-|-|
| Affirmative | A well-formed Issuer Authorization Policy retrieved (inline DNS, DNS pointer + HTTPS fetch, or HTTPS well-known URL) whose `subject_authority` matches `A` and whose structural validation succeeds. |
| Negative | DNS `negative-authoritative` followed by HTTPS 404/410 at the well-known URL; or HTTPS 404/410 reached from a DNS `uri=` pointer. The Authority Holder authoritatively publishes no delegation. |
| Indeterminate | DNS `indeterminate` (SERVFAIL, REFUSED, timeout, truncation); HTTPS transport failure (TLS error, connection failure); HTTPS 5xx; or `malformed` content at any layer (DNS record missing/mismatched `authority=`, neither `uri=` nor `issuer=`, multiple distinct `uri=` values, malformed directive, unrecognized `crit=` value; HTTPS body that is not a syntactically valid Issuer Authorization Policy; `subject_authority` that does not match `A`; unsupported media type; over-size body; non-HTTPS redirect; redirect loop). |

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

### Examples

Inline form (no HTTPS endpoint required on `acme.example`):

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://idp.example.net;
   issuer=https://idp-backup.example.net"
~~~

Pointer form (policy document hosted on a third party):

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   uri=https://delegation.example.org/acme.example.json"
~~~

## Verification {#dii-verification}

When the `domain_authorized_issuer` Trust Method is evaluated, the
Resource Authorization Server MUST:

1. Determine the Subject Authority from the assertion's Subject
   Identifier per {{dii-authority}}. If the format is not registered
   in {{iana-authority-registry}}, reject the assertion.

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
{{rasp}} is not met.

## Assertion Issuer Discovery {#dii-hrd}

Assertion Issuer Discovery is an optional client-side convenience. A
client that already knows which Assertion Issuer to use does not need
this procedure. A client that only holds a subject identifier and wants
to obtain an identity assertion can use the same records to find
candidate authorization server(s) for the namespace:

1. Determine the Subject Authority `A` from the input subject
   identifier per {{dii-authority}}.

2. Retrieve the Issuer Authorization Policy by applying the canonical
   procedure in {{dii-lookup}}.

3. Treat each `authorized_issuers` entry whose validity window includes
   the current time, whose `subject_identifier_formats` (if present)
   includes the input subject's format, and whose tenant context (if
   present) the client can request from the issuer, as a candidate.

4. For each candidate, resolve the authorization server's endpoints
   from the issuer identifier. The client SHOULD attempt OAuth
   Authorization Server Metadata {{RFC8414}}
   (`/.well-known/oauth-authorization-server`) first; on `404 Not
   Found` or equivalent the client MAY fall back to OpenID Connect
   Discovery {{OIDC-DISCOVERY}}
   (`/.well-known/openid-configuration`). For path-bearing issuer
   identifiers (for example,
   `https://login.example.com/{tenant}/v2.0`), the two standards
   construct the metadata URL differently: {{RFC8414}} inserts
   `/.well-known/oauth-authorization-server` between the host and
   the path, yielding
   `https://login.example.com/.well-known/oauth-authorization-server/{tenant}/v2.0`,
   while OpenID Connect Discovery appends
   `/.well-known/openid-configuration` after the path, yielding
   `https://login.example.com/{tenant}/v2.0/.well-known/openid-configuration`.
   For such issuers the client SHOULD attempt both constructions in
   order before declaring a candidate unresolvable. The client MUST
   verify that the discovered document's `issuer` value exactly
   matches the issuer identifier from the Issuer Authorization
   Policy using the comparison rules of {{RFC8414}}. A discovery
   transport failure or HTTP server error (5xx) SHOULD be treated
   as an `indeterminate` outcome for that candidate; the client MAY
   try other candidates.

When the policy lists more than one candidate, selection is
application-defined. The HTTPS and DNS pointer forms preserve
`authorized_issuers` array order as a Subject Authority preference
hint; the inline DNS form preserves the first-seen order of `issuer=`
directives the same way. Interactive clients MAY present candidates to
the end user, but SHOULD avoid exposing low-level issuer URLs when a
more usable display name is available from issuer metadata.

When a candidate entry includes a `tenant` value, a discovery client
SHOULD use that value as the tenant context when requesting an
assertion from the resolved Assertion Issuer: the resulting ID-JAG
MUST carry a top-level `tenant` claim ({{ID-JAG}} §6.1) equal to the
entry's `tenant` value or the assertion will not match this entry
during verification. The mechanism by which the client conveys
tenant context to the Assertion Issuer is Identity Provider-specific
(for example, a `login_hint`, a tenant-scoped authorization
endpoint, or a per-tenant API key) and is outside the scope of this
document; a client that cannot direct the Assertion Issuer to the
required tenant MUST treat the entry as unresolvable and try other
candidates.

A client performing Assertion Issuer discovery requires no
relationship with a Resource Authorization Server; the records
stand on their own.

The discovery flow is back-channel (server-to-server) and not
privacy-preserving: each retrieval exposes the queried Subject
Authority to DNS resolvers, the authoritative nameserver, the
policy host (under DNS pointer form), and the discovered
Assertion Issuer. For enterprise deployments these leaks can
convey business-sensitive information (Identity Providers under
evaluation, partner relationships, M&A signals). Clients SHOULD
use a privacy-preserving DNS resolution path, defer discovery
until a concrete issuance decision is required, cache results
within {{dii-caching}} bounds, and randomize timing where
applicable. For browser-mediated verification flows, the Email
Verification Protocol {{WICG-EMAIL-VERIF}} provides materially
stronger privacy properties and SHOULD be used instead.

## Caching {#dii-caching}

Resource Authorization Servers and clients SHOULD respect HTTP caching
headers on HTTPS policy documents and DNS TTLs on DNS records used by
this mechanism. For the DNS inline form, the cache lifetime of the
virtual policy is the minimum TTL of the recognized records used to
construct it. For the DNS pointer form, consumers SHOULD cache the DNS
pointer according to DNS TTLs and the fetched HTTPS policy according to
HTTP caching headers; the effective lifetime of a combined result
cannot exceed either lifetime.

A Subject Authority that revokes an authorization SHOULD publish
updated policy with short cache lifetimes during the revocation
window. Because revocation latency is bounded by the cache lifetime of
the most stale layer, Subject Authorities SHOULD publish DNS records
and HTTPS policies with a steady-state TTL of at most 1 hour and
SHOULD reduce the TTL further during an active revocation. Consumers
SHOULD enforce a local maximum cache lifetime (recommended: 24 hours,
absolute) regardless of HTTP caching headers or DNS TTLs, to bound
exposure to a stale delegation. Consumers MAY use a fresh cached
policy when live retrieval fails with an `indeterminate` outcome,
but MUST NOT use a cached policy after its effective lifetime has
expired. Consumers MUST NOT continue serving from cache for more
than 1 hour of cumulative `indeterminate` retrieval failures since
the most recent successful retrieval; beyond this window, the
consumer MUST treat the policy as `indeterminate` and reject
assertions that depend on it. This bound prevents an attacker who
can sustain DNS or HTTPS denial of service against the policy
endpoint from extending the effective cache lifetime indefinitely
and delaying revocation.

## Operational Considerations {#dii-operations}

DAI publication is a security-critical control plane. The cache
lifetime the Subject Authority chooses in steady state sets the
floor for revocation latency; high-value namespaces SHOULD
operate at short steady-state TTLs (minutes, not hours), well
below the 24-hour cache ceiling ({{dii-caching}}).

**Planned transitions** (vendor migration, IdP tenant migration,
issuer URL change, restructuring): reduce TTLs in advance of
cutover; publish overlapping `authorized_issuers` entries during
the migration window; use `valid_from`/`valid_until` to scope
coexistence deterministically; remove the retired entry only
after the full coexistence window has elapsed.

**Emergency deauthorization** (key compromise, contract
termination, incident response): publish the policy with the
affected issuer removed; update `last_updated`; notify dependent
Resource Authorization Servers out of band. This framework
provides no push invalidation mechanism, so the maximum exposure
window is `max-cache-lifetime + cumulative-indeterminate-cap`
(~25 hours with recommended defaults).

**Staging, rollout, rollback.** Treat policy publication as
production configuration management: publish to a staging Subject
Authority and validate consumer behavior before production; keep
the previous well-known-good policy text in change-control
records for rollback; restrict publication permissions and use
change-control workflows to mitigate delegated-admin mistakes.

**Third-party hosting.** Subject Authorities that do not host
their own HTTPS SHOULD use the signed-policy publication profile
({{publication-profiles}}) so integrity does not depend on host
behavior. Monitor the served policy against intended content
out-of-band.

**Monitoring.** Log retrieval outcomes (lookup state, cache hit
vs. live, cache age), the served `last_updated`, the matched
`authorized_issuers` entry on acceptance, and cumulative
Indeterminate retrieval streaks.

## Security Considerations {#dii-security}

The Issuer Authorization Policy is a security-critical document. Its
integrity determines which Assertion Issuers can assert identities in
the Subject Authority's namespace.

### Namespace Authority Bootstrap

The framework establishes authority over a namespace via the Subject
Authority Extraction Procedure ({{iana-authority-registry}}) plus
control of the publication channel. For the initial registered
extraction (`email` to registrable domain via the Public Suffix
List {{PSL}}), authority is established by DNS control: whoever can
publish at `_oauth-issuer-policy.{registrable-domain}` (or the
corresponding HTTPS well-known URL) is treated as the Subject
Authority for that namespace. This mirrors the bootstrap model of
CAA {{RFC8659}}, MTA-STS {{RFC8461}}, and the Email Verification
Protocol {{WICG-EMAIL-VERIF}}: DNS control IS namespace control.

The framework does not provide an independent bootstrap mechanism;
it inherits the DNS bootstrap end-to-end. Threats that follow from
this inheritance:

- **Namespace squatting via domain registration.** An attacker who
  registers a typosquat domain (`exampIe.com` versus `example.com`)
  becomes the Subject Authority for that typosquat and can publish
  any DAI policy for it. Resource Authorization Servers cannot
  distinguish a typosquat from the intended domain by examining the
  policy alone. Preventing typosquat consumption is the
  responsibility of layers that render Subject Identifiers to humans
  (display rendering, IDN homograph checks per {{RFC5891}}); this is
  a standard DNS reality and is not framework-specific.

- **Expired-domain takeover.** If a domain expires and is
  re-registered, the new registrant becomes the Subject Authority
  and can publish a new DAI policy. Subject Authorities of
  high-value namespaces SHOULD apply registry-lock and
  renewal-automation practices to prevent inadvertent loss of the
  domain.

- **Subdomain takeover.** Authority for `email` Subject Identifiers
  is the REGISTRABLE domain (eTLD+1 via {{PSL}}), not the queried
  subdomain. An attacker who controls only a subdomain (for
  example, via dangling DNS record takeover of `app.example.com`)
  cannot become the Subject Authority for `example.com`. See
  {{dii-authority}}.

- **Public-suffix masquerading.** A Subject Identifier whose
  registrable domain is itself a public suffix has no valid
  Subject Authority and is rejected ({{dii-authority}}). This
  prevents an attacker who controls a top-level or shared suffix
  from claiming authority over emails under it.

The framework does NOT establish prior identity of the Subject
Authority beyond DNS control: there is no notion of "verified
domain owner" or "legal-entity binding" within DAI. Deployments
requiring stronger identity binding (for example, real-world
legal-entity verification) MUST use out-of-band mechanisms; this
framework does not substitute for them.

### Transport Integrity {#transport-integrity}

For HTTPS retrieval, the integrity of the policy depends on TLS
server authentication of the host serving the policy. For the inline
DNS form, it depends entirely on DNS resolution. For the DNS pointer
form, it depends on DNS resolution plus TLS authentication of the
host named by the `uri=` directive.

Subject Authorities concerned about TLS misissuance SHOULD publish
CAA records {{RFC8659}} constraining the certificate authorities
permitted to issue for the policy-hosting domain.

### HTTPS-Only Authority Trust Model {#https-only-trust-model}

The `https_authorized_issuer` Trust Method
({{trust-method-https-authorized-issuer}}) bypasses DNS for
authority retrieval entirely. The policy is fetched directly from
the Subject Authority's apex well-known URL over TLS, and DNS is
not consulted for the authority lookup at all.

The trust model differs from `domain_authorized_issuer`:

- The DNS-side attacks described in
  {{dns-integrity-and-compromise}} (cache poisoning, BGP hijack of
  the resolution path, registrar account compromise, DNSSEC
  stripping) do NOT directly affect this method's authority
  retrieval. The TXT record at `_oauth-issuer-policy.{A}` is not
  fetched; substituting or suppressing it cannot redirect the
  consumer.

- DNS is still required to resolve the Subject Authority's
  hostname to an IP address for the HTTPS connection. A DNS
  compromise that redirects the apex hostname to an
  attacker-controlled IP can still substitute the policy IF the
  attacker also obtains a valid TLS certificate for the apex. The
  CAA-based defense in {{transport-integrity}} and Certificate
  Transparency monitoring become the primary defenses.

- The DNS pointer form's third-party policy host risk
  ({{third-party-policy-hosts}}) does not apply. The policy is
  hosted on the Subject Authority's own apex origin; there is no
  delegation to a separate host.

- Subject Authorities that operate their own DNS but cannot
  control the apex web origin (for example, the apex is hosted on
  a marketing CDN with no path control) cannot serve as
  `https_authorized_issuer` Subject Authorities. For such
  deployments, `domain_authorized_issuer` with the DNS pointer
  form is the appropriate mechanism.

Resource Authorization Servers selecting `https_authorized_issuer`
exclusively (omitting `domain_authorized_issuer` from the trust
policy) are explicitly distrusting DNS-published authority
artifacts. The cost is narrower Subject Authority coverage (only
those operating their own apex web origin can participate).

The two methods are NOT strictly ordered by security strength.
They trade different risks:

- `https_authorized_issuer` is resilient to attacks on the DNS
  TXT record itself but depends on hostname-to-IP DNS resolution
  plus the public CA trust system. A DNS-redirect of the apex
  hostname combined with a misissued TLS certificate succeeds
  against this method.

- `domain_authorized_issuer` (inline DNS form) is resilient to
  TLS-misissuance attacks on the apex hostname because the
  authority artifact is the TXT record itself; an attacker
  needs DNS write or DNSSEC-bypass capability rather than a
  forged TLS certificate.

- `domain_authorized_issuer` (DNS pointer form and HTTPS
  fallback) inherits the TLS-misissuance risk of
  `https_authorized_issuer` while ALSO depending on DNS for
  the pointer or for negative-authoritative routing.

The right choice depends on which threats the deployment can
defend against. A Subject Authority with strong DNSSEC and weak
TLS issuance controls favors the inline DNS form; a Subject
Authority with strong CAA/CT monitoring and limited DNS control
favors HTTPS-only.

### DNS Integrity and Compromise {#dns-integrity-and-compromise}

DNS-published authorization is only as trustworthy as the resolution
path that delivers it. An adversary who can substitute a forged DNS
response can substitute the Subject Authority's policy: by spoofing
an off-path resolver, hijacking the authoritative nameserver,
compromising the registrar account, executing a BGP hijack on the
resolution path, or corrupting a recursive resolver's cache.

The consequence of a successful substitution is loss of namespace
authority: the attacker can add an attacker-controlled Assertion
Issuer to the inline form and have it accepted as authoritative,
replace a legitimate `uri=` pointer with one pointing at an
attacker-controlled JSON document, or suppress the legitimate
response to force `indeterminate` outcomes that benefit a previously
cached attacker-friendly policy.

The framework's defenses are:

- **DNSSEC.** Subject Authorities SHOULD sign the zone containing
  `_oauth-issuer-policy.{A}` with DNSSEC. Consumers that perform
  DNSSEC validation SHOULD treat validation failure as
  `indeterminate`, not as `negative-authoritative`; otherwise an
  attacker who can strip DNSSEC signatures could downgrade the
  outcome to a negative result and force HTTPS fallback onto a
  separately-compromised path.

- **Resolver hygiene.** Consumers of DNS-published delegations
  without DNSSEC SHOULD use a trustworthy resolver path
  (DNS-over-HTTPS or DNS-over-TLS to a vetted resolver) and apply
  a conservative cache maximum ({{dii-caching}}).

- **HTTPS for richer policy.** The DNS pointer form (`uri=`)
  delegates policy CONTENTS to TLS-authenticated HTTPS retrieval.
  Because the DNS `uri=` target is authoritative, a DNS compromise can
  redirect consumers to an attacker-controlled HTTPS host and thereby
  rewrite the policy contents, provided that host presents a valid TLS
  certificate for itself. Consumers MUST verify TLS to the host named
  by `uri=`, but TLS does not mitigate compromise of the DNS record
  that selects the host. Subject Authorities relying on pointer form
  therefore depend on DNS integrity for target selection and HTTPS
  integrity for the selected policy contents.

- **Mandatory `authority=` directive.** A wildcard or substituted
  record MUST still carry `authority=A` matching the queried
  Subject Authority; see {{wildcard-dns-records}}.

- **Bounded cache lifetimes.** Cache lifetimes ceiling the
  exposure window from a single successful substitution
  ({{dii-caching}}, recommended absolute ceiling 24 hours).

Operational defenses Subject Authorities SHOULD apply in addition:

- Registrar account lock (registry-lock) preventing unauthorized
  domain transfers and DNS configuration changes through the
  registrar's control plane.

- Monitoring for unexpected changes to the
  `_oauth-issuer-policy.{A}` record set and to the policy document
  body, treated as a high-severity security event.

- Cross-region and cross-resolver checks (querying the record from
  multiple vantage points) to detect localized substitution.

### Wildcard DNS Records {#wildcard-dns-records}

A permissive DNS wildcard at a parent zone could answer queries for
every subdomain. Without an integrity check, every subdomain would
inherit the wildcard's authorization. The mandatory `authority=A`
directive defined in {{dii-dns-record}} mitigates this: consumers
MUST discard records whose `authority=` value does not match the
queried Subject Authority, so a single wildcard record cannot
authorize issuers for arbitrary subdomains.

Subject Authorities deploying DNS records SHOULD verify that no
broader wildcard in the parent zone could shadow their explicit
record, and SHOULD avoid publishing wildcard TXT records at zones
participating in this mechanism.

### Third-Party Policy Hosts {#third-party-policy-hosts}

The DNS pointer form lets the Subject Authority delegate policy
hosting to a different domain. This is intentional and enables
managed delegation services and third-party hosting profiles
(see {{publication-profiles}}). Resource Authorization Servers
MUST NOT impose a same-origin restriction on the `uri=` target
beyond requiring `https://`.

Two distinct authority models apply depending on which
publication profile the Subject Authority uses:

- **Profile 2 (authority-hosted HTTPS)**: the named host is fully
  trusted to publish authorizations on the Subject Authority's
  behalf. A host compromise substitutes the Subject Authority's
  voice with the attacker's. This model is appropriate only when
  the policy host is under the Subject Authority's operational
  control.

- **Profile 3 (signed policy at arbitrary origin)**: the host's
  identity does not establish authority. A `signed_policy` member
  in the document carries a JWS signature bound to the Subject
  Authority's key per {{signed-policy-metadata}}; a host
  compromise that produces a tampered document is detected at
  signature validation. This model is the safe choice for
  third-party hosting, CDN-fronted delivery, and partner-channel
  distribution.

Subject Authorities using third-party hosting SHOULD use Profile
3 unless they have specific reason to extend full trust to the
host. Resource Authorization Servers that fetch a document whose
HTTPS origin is NOT under the Subject Authority's DNS-namespace
control MUST verify `signed_policy` before relying on the
document; if `signed_policy` is absent or fails verification,
the policy MUST be treated as malformed.

The `authority=` directive in the DNS record and the
`subject_authority` member of the JSON document at the pointed-at
URL both bind the policy to a specific namespace. Both bindings are
required even when one would seem to suffice: the DNS-side
`authority=` is published by the namespace owner and attests that
the policy at the `uri=` target is intended for namespace `A`; the
JSON-side `subject_authority` is published by the policy host and
declares which namespace the policy describes. A consumer requires
both to match because a compromise of either party should not allow
an arbitrary namespace claim. If only the JSON were checked, a
compromised policy host could rewrite `subject_authority` to claim
any namespace. If only the DNS were checked, a compromised DNS
publisher could point at any unrelated JSON document and have it
treated as authoritative.

The `uri=` directive is a long-lived trust delegation. Once
consumers begin caching a virtual policy that resolved through it,
revocation of the delegation is bounded by the DNS TTL of the
pointer record. Subject Authorities SHOULD treat the selection of a
`uri=` target with the operational care applied to long-lived key
delegations, and SHOULD reduce DNS TTLs in advance of any planned
change of policy host.

### Delegation Lifecycle: Transfer and Revocation {#delegation-lifecycle}

A Subject Authority's authorized issuers change over time, both
through planned transfers (provider migrations, acquisitions,
re-architectures) and through unplanned revocations (security
incidents). The framework provides no remote cache-invalidation
mechanism; revocation latency is bounded by DNS TTLs, HTTP cache
lifetimes, and the consumer cache ceilings in {{dii-caching}}.
Subject Authorities that need fast revocation MUST operate with
short steady-state cache lifetimes. Operational guidance is in
{{dii-operations}}.

### Policy Conflicts and Determinism {#policy-conflicts}

The lookup procedure ({{dii-lookup}}) is deterministic across the
common conflict scenarios that arise when multiple records or
sources coexist. Determinism is a security property: a verifier and
a discovery client receiving the same DNS and HTTPS responses MUST
reach the same conclusion about what (if any) policy applies. An
attacker with partial control of one publication channel cannot
exploit interpretive ambiguity at the consumer.

Common conflict scenarios and their deterministic dispositions are
summarized in {{dii-operational-conflicts}}.

### Document Authentication Limits

The inline DNS form has no signing mechanism; its authority binding
is DNS control. Deployments requiring cryptographic proof of intent
or third-party hosting use Publication Profile 3
({{publication-profile-signed}}) with `signed_policy`
({{signed-policy-metadata}}).

### Scope of Authorization

A Resource Authorization Server MUST NOT use the Issuer Authorization
Policy to establish trust in an Assertion Issuer for subjects outside
the matched Subject Authority. The policy authorizes an Assertion
Issuer only within the namespace the Subject Authority controls.

### Email Local-Part Is Not Authenticated by This Trust Method {#email-local-part}

The Subject Authority Extraction for `email` uses only the domain
part (registrable domain per {{dii-authority}}). The local-part of
the email address is unauthenticated by the
`domain_authorized_issuer` Trust Method: the Trust Method
establishes only that the Assertion Issuer has the Subject
Authority's permission to assert about emails in the namespace, not
that any particular local-part value is correct.

Resource Authorization Servers that link accounts using the
local-part (for example, treating `alice@example.com` and
`Alice@example.com` as the same user, or normalizing plus-addresses
like `alice+test@example.com` to `alice@example.com`) inherit the
trust assumptions of their normalization rules from the Assertion
Issuer, not from this Trust Method. An attacker who controls an
alias accepted by the Assertion Issuer (for example, a
plus-addressed alias the issuer's registration process accepted
without verification) can obtain an assertion that the Resource
Authorization Server's normalization rules then map to a different
user. Resource Authorization Servers SHOULD either disable
local-part normalization or independently verify the local-part
through a mechanism outside the scope of this document.

### Inline-Form Feature Limits {#inline-form-feature-limits}

The inline DNS form expresses only "issuer X is authorized for
Subject Authority A." It cannot express `tenant`,
`subject_identifier_formats`, `valid_from`, or `valid_until`. Subject
Authorities needing any of those, including any binding to a
specific tenant of a shared-issuer multi-tenant Identity Provider
({{dii-multi-tenant}}), MUST use the HTTPS or DNS pointer form.

### Single-Issuer Multi-Tenant Identity Providers {#dii-multi-tenant}

Some Identity Providers serve many tenants under a single issuer
identifier (for example, `https://accounts.google.com` for every
Google Workspace tenant). The tenant is distinguished by the
top-level `tenant` claim ({{ID-JAG}} §6.1) rather than by `iss`.
Identity Providers that publish per-tenant issuer identifiers
(for example, `https://login.microsoftonline.com/{tenant-id}/v2.0`)
avoid this case entirely; each tenant has a distinct `iss`.

For shared-issuer Identity Providers, Subject Authorities SHOULD
set the `tenant` member on the `authorized_issuers` entry to the
authorized tenant. Verification ({{dii-verification}} step 4)
requires exact match against the assertion's `tenant` claim. A
Subject Authority listing a shared issuer with NO `tenant`
constraint authorizes EVERY tenant of that Identity Provider for
its namespace; this is almost never the intent and SHOULD be
avoided. Resource Authorization Servers SHOULD log a warning when
accepting under an unconstrained entry and SHOULD consider
rejecting as a matter of local policy.

The `tenant` binding is a wire-format expression of trust, not a
cryptographic guarantee. The framework verifies match against the
Subject Authority's chosen tenant value, but cannot verify the
Identity Provider's tenant-isolation implementation. A
tenant-isolation defect (coding bug, misconfigured admin
operation, exploitable feature) defeats the wire-format check;
this framework offers no defense beyond the Identity Provider's
engineering quality. The Identity Provider MUST bind users to
their tenant correctly (no cross-tenant assertion minting),
verify tenant claim over the namespace before provisioning, and
sustain these properties under operational change.

Subject Authorities SHOULD prefer per-tenant issuer identifiers
where offered, treat shared-issuer listings as long-lived
delegations requiring due diligence on the Identity Provider's
tenant-isolation guarantees, and reduce DAI cache lifetimes
during known instability to bound exposure.

### Assertion Issuer Discovery Threats

A client performing Assertion Issuer discovery is directed by
the discovered policy to an authorization server it may not have
previously interacted with. A successful spoof of the Subject
Authority's DNS (in the absence of DNSSEC) can redirect a user to an
attacker-controlled authorization server, with the consequences of
credential phishing. Interactive clients SHOULD treat
newly-discovered issuers with the scrutiny applied to any unfamiliar
authorization server, prefer DNSSEC-validated resolution paths, and
avoid silently following discovery results across domain boundaries
the user did not expect.

### Enumeration and Query Privacy

DNS-based lookup exposes the Subject Authority being queried to the
resolver path. It does not expose the full subject identifier for
formats such as `email`, but it can reveal organizational
relationships and timing. Clients and Resource Authorization Servers
SHOULD use privacy-preserving resolver configurations where available
and SHOULD avoid performing discovery until it is needed for a concrete
issuance or verification decision.

# Security Considerations

## Transitive Authorization is Bounded {#transitive-authz-bounded}

This framework deliberately distinguishes transitive *authentication*
(in scope) from transitive *namespace authorization* (out of scope).

Transitive authentication is supported via OpenID Federation. The
`openid_federation` Trust Method
({{trust-method-openid-federation}}) accepts a chain from a leaf entity
to a trust anchor, where the trust anchor's signature on intermediate
Subordinate Statements transitively authenticates the leaf. This
answers question 1 from {{two-trust-questions}} ("who signed it?")
through a chain.

Transitive namespace authorization is NOT supported. A Subject
Authority's Issuer Authorization Policy ({{dii-document}}) lists
specific Assertion Issuers by identifier; it cannot express
"delegate further to whoever Issuer X federates" or "honor any issuer
named by Authority Y." If `example.com` wants `https://workos.example`
to assert about its users, `example.com` MUST list
`https://workos.example` directly in `authorized_issuers`. The
namespace-authorization trust graph is bounded to a depth of one.

This bounding is a load-bearing security property, not a limitation:

- A Resource Authorization Server is never required to compute a
  multi-hop authorization chain to evaluate a namespace claim. There
  is always a single, customer-published policy that names the
  Assertion Issuer.

- Revocation latency is bounded by the Subject Authority's own
  cache lifetime; it does not compound across delegations.

- A compromise at any intermediate party (federation operator,
  delegated issuer) cannot expand the namespace authorization of an
  Assertion Issuer that the Subject Authority did not list directly.

Federation membership and namespace authorization are independent
axes. An Assertion Issuer authenticated by federation membership
gains no namespace authorization from that membership; a Subject
Authority that lists an Assertion Issuer in DAI grants no federation
membership through that listing. The cross-category combination rule
in {{rasp}} enforces this independence at evaluation time.

OpenID Federation handles transitive authentication; this framework
does not duplicate that mechanism. What this framework adds,
non-transitive namespace authorization, has no analog in OpenID
Federation. See {{relationship-to-oidf}} for the broader positioning.

## Scope of Namespace Authorization {#scope-of-namespace-authorization}

A trust policy describes issuer acceptability. It does not
validate an assertion's signature, audience, expiration, replay
protection, subject, or client binding; Resource Authorization
Servers MUST validate the assertion according to the applicable
grant profile before issuing an access token.

A Subject Authority's authorization of an Assertion Issuer is a
narrow statement. It says exactly:

> This Assertion Issuer is authorized by the namespace authority to
> assert this class of subject identifiers under this namespace.

It does NOT say, and Resource Authorization Servers MUST NOT
infer from a successful trust-policy evaluation, that:

- The named subject exists at the Assertion Issuer.
- The named subject controls the local-part of the email address
  (see {{email-local-part}}).
- The named subject is currently employed, enrolled, or otherwise
  in an active relationship with the namespace owner.
- The Assertion Issuer performed strong authentication (multi-factor,
  hardware-bound, recent, etc.) before issuing the assertion.
- The Assertion Issuer's account-linking semantics
  (case-insensitivity, plus-address normalization, alias handling)
  match those of any particular Resource Authorization Server.
- The assertion is suitable for every Resource Authorization
  Server's authorization purpose, regardless of risk class, scope
  sensitivity, or compliance regime.

These properties are out of scope for namespace authorization and
are the responsibility of the Assertion Issuer's authentication
process, the Resource Authorization Server's local policy, and
mechanisms outside this framework.

The `email_verified=true` claim required for `email` Subject
Identifiers ({{dii-authority}}) is a PREREQUISITE for deriving
namespace authority from the email's domain, not proof of current
mailbox control or organizational status. An Assertion Issuer
MAY have verified the mailbox at registration time months or
years before the assertion is issued; the Resource Authorization
Server SHOULD NOT treat `email_verified=true` as evidence of
liveness of the mailbox or of the subject's current relationship
to the namespace.

Resource Authorization Servers requiring stronger guarantees on
any of the above properties MUST obtain them through mechanisms
outside this framework: explicit authentication-method or AAL
claims, fresh-authentication signals, account-status attestations
from the namespace authority, or out-of-band verification.

The `subject_namespace_authorization` category constrains WHICH
Assertion Issuers may assert about a namespace; it does NOT
constrain which Resource Authorization Servers an authorized
Assertion Issuer may target. Audience binding is enforced by the
assertion's `aud` claim and the applicable grant profile, not by
the Subject Authority. Subject Authorities that need to restrict
their authorized issuers' audience targets rely on out-of-band
agreements; this document defines no wire-level mechanism for
that restriction.

## Per-Assertion Revocation Is Out of Scope {#per-assertion-revocation}

This framework establishes whether an Assertion Issuer is authorized
to assert about a subject; it does not define a mechanism to revoke
an individual assertion. JWT bearer tokens are stateless: once
issued, they are valid until their `exp` claim regardless of session
state at the Assertion Issuer, the user logging out, the Identity
Provider revoking the user's credentials, or the Subject Authority
withdrawing the issuer's authorization via DAI.

Per-assertion revocation needs are addressed by mechanisms outside
this framework:

- OAuth 2.0 Token Revocation {{RFC7009}} for tokens whose issuance
  passed through this framework.
- OAuth 2.0 Token Introspection {{RFC7662}} for receivers that want
  real-time validity checks.
- Short assertion lifetimes (governed by the applicable grant
  profile) reduce the exposure window for a leaked or compromised
  assertion.

Subject Authority withdrawal of an issuer's authorization via DAI
prevents NEW assertions from that issuer being accepted (subject to
the latency model in {{delegation-lifecycle}}) but does not
invalidate already-issued assertions still within their `exp`
window. Resource Authorization Servers that require synchronous
revocation MUST implement {{RFC7009}}, {{RFC7662}}, or an
equivalent at a layer outside this framework.

## Policy Document Integrity {#integrity}

The trust policy MUST be served over HTTPS with TLS server
authentication. Deployments needing integrity beyond TLS use the
`signed_policy` member ({{signed-policy-metadata}}), with the
signer binding rules defined there. Mirrored or cached copies
MUST NOT be relied on beyond their HTTP cache lifetime
({{caching}}).

## Shared Infrastructure and Hosted Well-Known Paths {#shared-infrastructure}

Many deployments host `/.well-known/` resources behind third-party
Content Delivery Networks (CDNs), shared edge platforms, or
multi-tenant cloud hosting systems. TLS server authentication proves
that the client reached an endpoint serving the requested host name;
it does not prove that every routing rule, origin-pull rule, cache
rule, or tenant boundary inside the shared platform is controlled by
the namespace owner.

This matters for both the Resource Authorization Server trust policy
and the Subject Authority's Issuer Authorization Policy. If an
attacker can exploit shared infrastructure to serve a forged
`/.well-known/oauth-issuer-policy` response for a victim domain, the
attacker can attempt to authorize an Assertion Issuer for that
victim's namespace even though the HTTPS connection itself succeeds.
Similar risks arise from misconfigured path routing, dangling origins,
tenant takeover, cache poisoning, or origin authentication failures.

Administrators SHOULD avoid delegating security-critical well-known
paths to multi-tenant infrastructure unless they can ensure exclusive
control over routing for those paths, authenticated origin access,
cache invalidation, and tenant isolation. Deployments that host policy
documents on shared infrastructure SHOULD use object-level
cryptographic integrity for the policy document itself, such as a
JWS-signed JWT {{RFC7515}}, rather than relying solely on the
TLS channel to the shared edge. The signing key SHOULD be controlled by
the Subject Authority or Resource Authorization Server independently
of CDN tenant configuration.

Consumers that require object-level integrity SHOULD require and verify
the `signed_policy` member before acting on the policy and SHOULD
treat a valid TLS connection to a shared edge as insufficient by itself
when local policy requires object-level integrity.

## Policy Disclosure

The trust policy can reveal which trust frameworks, registries,
federation anchors, and Subject Identifier formats a Resource
Authorization Server accepts. Deployments SHOULD consider whether
the policy URI is public, authenticated, or audience-specific.

If an authenticated or audience-specific policy is used, Resource
Authorization Servers SHOULD ensure that clients receive a policy
consistent with the policy enforced at the token endpoint, otherwise a
client can be induced to obtain assertions that cannot succeed.

## Downgrade Attacks {#downgrade}

A Resource Authorization Server that supports multiple Trust Methods
SHOULD define local precedence rules. Clients MUST NOT choose a
weaker Trust Method if the Resource Authorization Server requires a
stronger one for the requested subject, client, or scope. Because
OR-semantics apply within a single Trust Method category, Resource
Authorization Servers MUST apply differentiated requirements at token
request time rather than relying on the policy document alone. The
cross-category combination rule in {{rasp}} prevents downgrade across
categories (for example, satisfying only an `issuer_authentication`
method when a `subject_namespace_authorization` method is also
applicable).

## Trust Policy Enumeration

Trust policies are typically published as unauthenticated HTTPS
resources. Adversaries can scrape policies across many Resource
Authorization Servers to map an entire federation deployment
landscape: which Resource Authorization Servers participate in
which federations, which trust anchors are accepted, which Subject
Identifier formats are honored. This
information aids targeted attacks against federation infrastructure
(for example, prioritizing compromise of a heavily-relied-upon trust
anchor). Operators SHOULD treat the choice of what to publish in a
trust policy as a security disclosure decision, not only an
operational convenience, and SHOULD avoid revealing more
relationships than necessary.

## Trust Policy Caching {#caching}

Clients and Resource Authorization Servers SHOULD respect HTTP
caching headers on the trust policy document, subject to local
maximum cache lifetimes. A Resource Authorization Server changing
its trust policy SHOULD use short cache lifetimes during migration.
Revocation of a Trust Anchor or federation entity SHOULD NOT rely
on cached policy expiration alone; revocation status
of validated trust evidence MUST be checked at the cadence required
by the applicable Trust Method specification. Transport integrity of
the policy document itself is addressed in {{integrity}}.

## Subject Authority Matching

A Trust Method that depends on a Subject Authority MUST define how
that Subject Authority is determined from the assertion's Subject
Identifier (see {{dii-authority}} for the determination defined by
this document). The determination MUST avoid wildcard, suffix,
regular-expression, and substring matching unless explicitly
specified by the relevant Subject Identifier format.

## Observability

Trust policy evaluation is a security-critical decision and SHOULD
be auditable. Resource Authorization Servers SHOULD log, for each
processed identity assertion, at minimum: the Assertion Issuer
identifier; the trust policy URI and its `last_updated` value (or
an equivalent cache identifier); the Trust Method identifier or
identifiers that succeeded; the matched trust anchor or Issuer
Authorization Policy origin where applicable; the Subject
Identifier format used for subject-namespace evaluation; and the
outcome (`accepted` or the specific rejection reason). Log records
SHOULD support correlation across the issuance and verification
halves of an identity-chain transaction.

# Privacy Considerations

Publishing accepted trust methods and trust anchors can reveal
deployment relationships and security posture. Resource Authorization
Servers SHOULD publish only the information needed for clients to
determine whether they can attempt issuance. Trust policy documents
SHOULD avoid enumerating individual issuers when trust can be
expressed through trust anchors, federation, registries, or
Domain-Authorized Issuer Discovery.

Assertion Issuer discovery reveals the queried Subject Authority
to DNS resolvers and, for HTTPS policy retrieval, to the policy host.
Clients SHOULD avoid sending full subject identifiers in policy URLs,
query parameters, logs, or telemetry. Resource Authorization Servers
that verify assertions with the `domain_authorized_issuer` Trust Method
SHOULD treat the Subject Authority and Assertion Issuer relationship as
security-sensitive operational data.

A detailed analysis of discovery-flow privacy, including the specific
information leaks to the recursive DNS resolver, the authoritative
nameserver, the policy host, and the discovered Assertion Issuer, along
with available mitigations, is in {{dii-hrd}}.

# IANA Considerations

## Trust Policy Registrations

### OAuth Authorization Server Metadata Registry

Registers `identity_assertion_trust_policy_uri` in the IANA "OAuth
Authorization Server Metadata" registry:

Metadata Name:
: `identity_assertion_trust_policy_uri`

Metadata Description:
: HTTPS URI identifying the Resource Authorization Server's Identity
Assertion Issuer Trust Policy document.

Change Controller:
: IETF

Specification Document:
: This document

### OAuth Protected Resource Metadata Registry

Registers the same parameter in the IANA "OAuth Protected Resource
Metadata" registry {{RFC9728}}:

Metadata Name:
: `identity_assertion_trust_policy_uri`

Metadata Description:
: HTTPS URI identifying the Identity Assertion Issuer Trust Policy
document applied by the Resource Authorization Server associated
with this Protected Resource.

Change Controller:
: IETF

Specification Document:
: This document

### Well-Known URI for Trust Policy {#iana-trust-policy-well-known}

Registers the following well-known URI in the IANA "Well-Known URIs"
registry {{RFC8615}}:

URI Suffix:
: `identity-assertion-trust-policy`

Change Controller:
: IETF

Specification Document:
: This document

Status:
: permanent

Related Information:
: None

### Identity Assertion Issuer Trust Methods Registry {#iana-trust-methods-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Issuer Trust Methods" under the "OAuth Parameters"
registry group.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains:

Identifier:
: Short string used as the `method` value in a Trust Method object.
Identifiers MUST use the character set `[a-z0-9_]` and SHOULD
describe the trust evaluation procedure.

Categories:
: One or more values from the set defined in
{{trust-method-categories}} (`issuer_authentication`,
`subject_namespace_authorization`). Future specifications MAY define
additional categories via Specification Required.

Parameters:
: Additional JSON members defined for this Trust Method, with their
JSON types and whether each is REQUIRED or OPTIONAL.

Reference:
: A reference to the specification defining the Trust Method.

Initial entries:

| Identifier | Categories | Parameters | Reference |
|-|-|-|-|
| `openid_federation` | `issuer_authentication` | `trust_anchors` (array of string, REQUIRED); `trust_marks` (array of object, OPTIONAL) | This document |
| `domain_authorized_issuer` | `subject_namespace_authorization` | (none) | This document |
| `https_authorized_issuer` | `subject_namespace_authorization` | (none) | This document |
| `email_verification_dns` | `subject_namespace_authorization` | (none) | This document, {{WICG-EMAIL-VERIF}} |

### Trust Evaluation Diagnostic Reason Registry {#iana-diagnostic-reasons-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Issuer Trust Evaluation Diagnostic Reasons" under the
"OAuth Parameters" registry group.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains:

Reason Code:
: Short string identifying the diagnostic reason. Reason codes
MUST use the character set `[a-z0-9_]` and SHOULD describe the
trust-evaluation failure condition.

Description:
: A short description of the condition the reason code identifies.

Disclosure Class:
: One of `trusted-context` or `public`. Reason codes initially
registered by this document are `trusted-context`; future
specifications MAY register codes intended for public disclosure
if they carry no operationally sensitive information.

Reference:
: A reference to the specification defining the reason code.

Initial entries (all `trusted-context`, defined in
{{diagnostic-reasons}}): `issuer_not_authenticated`,
`issuer_not_authorized_for_subject_authority`,
`subject_authority_policy_not_found`,
`subject_authority_policy_indeterminate`,
`subject_authority_policy_malformed`,
`tenant_binding_required`, `tenant_binding_mismatch`,
`subject_identifier_format_not_authorized`, `policy_expired`,
`policy_not_yet_valid`, `trust_method_not_satisfied`,
`applicability_bypass_attempted`.

## Issuer Authorization Policy Registrations

### Well-Known URI for Issuer Authorization Policy

Registers the following well-known URI in the IANA "Well-Known URIs"
registry {{RFC8615}}, for use by {{dii}}:

URI Suffix:
: `oauth-issuer-policy`

Change Controller:
: IETF

Specification Document:
: This document, {{dii}}

Status:
: permanent

Related Information:
: None

### Underscored DNS Node Name

Registers the following entry in the IANA "Underscored and Globally
Scoped DNS Node Names" registry per {{RFC8552}} and {{RFC8553}}, for
use by {{dii}}:

RR Type:
: TXT

Node Name:
: `_oauth-issuer-policy`

Specification Document:
: This document, {{dii-dns-record}}

### Subject Authority Extraction Procedures Registry {#iana-authority-registry}

IANA is requested to establish a new registry titled "Subject
Authority Extraction Procedures" under the "OAuth Parameters"
registry group.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains:

Subject Identifier Format:
: The Subject Identifier format identifier (typically registered in
the "Security Event Subject Identifier Formats" registry per
{{RFC9493}}).

Subject Authority Form:
: A short description of the form taken by the Subject Authority for
this format (for example, "DNS domain", "URL host").

Extraction Procedure:
: A reference to the specification text that defines how the Subject
Authority is computed from a Subject Identifier of this format.

Entries in this registry specify whether they apply to direct Subject
Identifiers, actor Subject Identifiers carried by the OAuth Actor
Profile binding ({{actor-profile-binding}}), or both. Future
specifications that define new actor-profile entity types or Subject
Identifier formats are expected to register additional entries here
when those identifiers have a well-defined namespace authority.

Initial entries:

| Subject Identifier Format | Subject Authority Form | Applies To | Extraction Procedure |
|-|-|-|-|
| `email` | DNS domain | Direct Subject Identifier only | {{dii-authority}} of this document |

# Extending to New Subject Identifier Formats {#extending}

This document registers one Subject Authority extraction procedure
(`email`, {{dii-authority}}) and one `subject_namespace_authorization`
Trust Method that publishes via DNS (`domain_authorized_issuer`,
{{trust-method-domain-authorized-issuer}}). The framework is designed
to accommodate additional Subject Identifier formats and additional
publication mechanisms via future specifications. This section
describes the extension procedure for spec authors.

To extend this framework to a new Subject Identifier format, a future
specification:

1. **Registers a Subject Authority Extraction Procedure** in
   {{iana-authority-registry}}, defining how a Subject Authority is
   computed from values of the new format. The registry entry
   identifies the Subject Identifier format, the form taken by the
   Subject Authority (for example, a domain, a URL host, a DID
   method-specific identifier, a registry-issued identifier), and the
   comparison rules for matching Subject Authority values.

2. **Selects or registers a Trust Method.** If the computed Subject
   Authority is a domain that can publish DNS records, the existing
   `domain_authorized_issuer` Trust Method
   ({{trust-method-domain-authorized-issuer}}) applies without
   modification; the new format simply joins `email` as a value that
   can appear in `subject_identifier_formats` on `authorized_issuers`
   entries. The `email_verification_dns` Trust Method
   ({{trust-method-email-verification-dns}}) is intentionally
   email-specific and is NOT reusable for other formats; new formats
   that need a non-`domain_authorized_issuer` mechanism must register
   their own. If the computed Subject Authority is not a domain or
   cannot use DNS-based publication, the specification registers a
   new `subject_namespace_authorization` Trust Method in
   {{iana-trust-methods-registry}} defining its publication and
   lookup procedure.

3. **Specifies how the assertion grant profile carries the new
   Subject Identifier.** The new format may be carried as one or more
   top-level JWT claims (as `email` is in {{ID-JAG}}), as an
   {{RFC9493}} Subject Identifier JSON object, or as another
   grant-profile convention. The Subject Authority extraction operates
   on the claim shape the grant profile defines.

The Trust Policy document format ({{trust-policy-document}}), the
Issuer Authorization Policy document format ({{dii-document}}), the
verification flow ({{dii-verification}}), the Trust Method category
structure and combination rule ({{trust-method-categories}},
{{rasp}}), the RAS evaluation model ({{ras-evaluation-model}}), and
the bounded-transitivity property of namespace authorization
({{transitive-authz-bounded}}) remain unchanged across extensions.
This stability is the framework's reusability guarantee: spec
authors extending it for a new attribute-bearing claim type inherit
the trust evaluation model unchanged.

## Future Extensions (Non-Normative) {#sketch-email-domain-exact}

Candidate extensions that have been considered as future work:

- **Web Origin Subject Identifiers**: a `url_host` format whose
  Subject Authority is the registrable domain of the URL's host;
  reuses `domain_authorized_issuer` unchanged.

- **Decentralized Identifiers**: a `did` format whose extraction
  resolves the DID to its method-specific controller; uses
  `domain_authorized_issuer` when the controller is a domain, or
  a new Trust Method that reads authorization from the DID
  document otherwise.

- **Subdomain-Exact Email Authority**: an `email_domain_exact`
  extraction that derives the Subject Authority from the exact
  email host. To preserve the protection the registrable-domain
  default provides against subdomain takeover
  ({{subdomain-authority}}), the registrable-domain authority MUST
  publish an explicit delegation naming the authorized subdomain
  authorities; without that delegation, the extraction MUST fall
  back to the registrable-domain authority.

Concrete extension specifications are out of scope for this
document.

--- back

# Design Rationale

This appendix is non-normative. It records the design choices that
shaped the framework, for reviewers and implementers who want to
understand why specific decisions were made.

## Relationship to OpenID Federation {#relationship-to-oidf}

OpenID Federation {{OIDF-FEDERATION}} answers "which authorization
servers are authentic members of a recognized ecosystem?" through
trust chains rooted at federation trust anchors. It does not, by
itself, answer "is this authorization server authoritative for
subjects in this namespace?". That question is independent of
federation membership.

This framework is complementary to OpenID Federation, not a replacement
for or alternative to it:

- OpenID Federation membership is a recognized form of evidence in this
  framework's `issuer_authentication` category (via the
  `openid_federation` Trust Method,
  {{trust-method-openid-federation}}).

- This framework's distinct contribution is the
  `subject_namespace_authorization` category and the Issuer
  Authorization Policy ({{dii-document}}) that supplies its evidence.
  A federation operator does not decide which authorization servers are
  authoritative for a given email domain; only the domain owner can.

- The Identity Assertion Issuer Trust Policy is a policy layer that
  lets a Resource Authorization Server require evidence from both
  categories together, without duplicating OpenID Federation's
  trust-chain, metadata-publication, or policy-propagation mechanisms.

## Following Existing DNS Authority Patterns {#dns-authority-patterns}

Domain-Authorized Issuer Discovery applies the same
authority-publication pattern that domain owners already use for
CAA {{RFC8659}}, MTA-STS {{RFC8461}}, SPF, DKIM, and the Email
Verification Protocol {{WICG-EMAIL-VERIF}}. The Authority
Delegation Framework {{AUTHORITY-DELEGATION}} covers the abstract
pattern in detail; this document chooses DNS at
`_oauth-issuer-policy.{domain}` as the authoritative publication
channel.

Unlike CAA (deployment-time), SPF/DKIM (spam-score signal), and
MTA-STS (inbound mail), the `_oauth-issuer-policy` record is
consumed during user sign-in. Misconfiguration or staleness
directly blocks user sign-in to dependent applications, so
Subject Authorities deploying this framework SHOULD treat records
with sign-in-dependency operational rigor (change-management
review, monitoring for unexpected changes, rapid rollback).

## Why Bounded-Depth-1 Namespace Authorization

The Issuer Authorization Policy is a flat list of Assertion Issuer
identifiers, not a graph. A Subject Authority cannot delegate "to
whoever Issuer X federates"; it must list each authorized issuer
directly. This bounding is deliberate:

- A Resource Authorization Server is never required to compute a
  multi-hop authorization chain to evaluate a namespace claim. There
  is always a single, customer-published policy that names the
  Assertion Issuer.

- Revocation latency is bounded by the Subject Authority's own
  cache lifetime, not by the depth of a delegation chain.

- A compromise at any intermediate party (federation operator,
  delegated issuer) cannot expand the namespace authorization of an
  Assertion Issuer the Subject Authority did not list directly.

Federation chains for issuer authentication remain in scope under
the `issuer_authentication` category; only the namespace-authorization
graph is bounded. See {{transitive-authz-bounded}} for the full
treatment.

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
  binding, format restrictions, critical extensions) that the
  inline DNS form cannot express.

DNS remains the entry point; HTTPS is consulted only as fallback
when the DNS query returns `negative-authoritative`. This preserves
the DNS-authority pattern as the primary channel while
accommodating deployment variations.

# Policy Conflicts and Determinism {#dii-operational-conflicts}

This appendix is non-normative.

Disposition of common conflict scenarios:

- **Multiple recognized TXT records, all `issuer=` only.** Merged into
  a single virtual policy; `issuer=` values across records are
  deduplicated, with order preserved by first-seen ({{dii-lookup}},
  step 2c).

- **Mix of `uri=` and `issuer=` records.** If any recognized record
  (after `authority=` filtering) contains `uri=`, the HTTPS document
  at the `uri=` target is authoritative; all `issuer=` directives
  across all records are ignored ({{dii-lookup}}, step 2b). This rule
  prevents an attacker who can add an extra TXT record, but not modify
  existing ones, from injecting issuers when a `uri=` policy is in
  effect.

- **Multiple distinct `uri=` values.** Treated as `malformed`
  ({{dii-failures}}). The framework does not attempt to choose; the
  consumer cannot tell which is legitimate. A verifier rejects the
  assertion; a discovery client reports an error.

- **DNS and HTTPS well-known URL both populated with differing
  content.** The lookup procedure consults DNS first; an `affirmative`
  DNS response, inline or pointer, is authoritative, and the bare HTTPS
  well-known URL is not consulted. A Subject Authority that publishes
  both forms needs to keep them consistent; consumers do not reconcile
  differences.

- **Multiple `authorized_issuers` entries for the same `issuer` value
  but different `tenant` values.** Each `(issuer, tenant)` pair is an
  independent authorization. Verification matches the first entry whose
  `(issuer, tenant)` matches the assertion ({{dii-verification}},
  step 4); entries with `tenant` do not shadow entries without `tenant`
  for the same `issuer`.

- **Two Subject Authorities publishing for the same subject.** Only the
  Subject Authority computed by the extraction procedure for the
  assertion's Subject Identifier applies. An assertion carrying
  `email=alice@example.com` is evaluated only against `example.com`'s
  policy; no other namespace's policy can grant authority over
  `alice@example.com`, regardless of what other Subject Authorities
  publish.

- **Conflict between the assertion's claims and the matched policy
  entry.** Resolved by rejecting the assertion. A matched entry whose
  `tenant` value differs from the assertion's `tenant` claim does not
  authorize the assertion ({{dii-verification}}, step 4); the assertion
  fails the Trust Method.

# DNS-Based Domain-Authorized Issuer End-to-End Example

This appendix is non-normative.

This example walks through an end-to-end flow that exercises
Domain-Authorized Issuer Discovery in both directions: a
Client uses the DNS records to find Alice's Assertion Issuer,
then the Resource Authorization Server uses the same records to
verify the resulting assertion.

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

## Assertion Issuer Discovery (Client Side)

1. The Client holds `alice@acme.example` and wants to obtain an
   ID-JAG.

2. The Client computes the Subject Authority from the email's domain
   portion: `acme.example`.

3. The Client queries DNS TXT at
   `_oauth-issuer-policy.acme.example`. The response is
   `affirmative` and contains one recognized record. The
   `authority=acme.example` directive matches, so the record is
   retained.

4. The record contains no `uri=` and one `issuer=` directive. The
   Client constructs a virtual policy:

   ~~~ json
   {
     "subject_authority": "acme.example",
     "authorized_issuers": [
       { "issuer": "https://idp.example.net" }
     ]
   }
   ~~~

5. The Client applies OAuth Authorization Server Metadata
   {{RFC8414}} to `https://idp.example.net` and resolves its
   token endpoint.

## Issuance and Token Request

6. The Client authenticates Alice at `https://idp.example.net`
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

7. The Client posts to the Resource Authorization Server's token
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

8. The Resource Authorization Server validates the ID-JAG:
   signature (via
   `https://idp.example.net/.well-known/openid-configuration`
   JWKS), `aud`, `exp`, `iat`, replay protection.

9. The Resource Authorization Server evaluates the
   `domain_authorized_issuer` Trust Method.

   a. It extracts the Subject Authority from the top-level `email`
      claim (with `email_verified=true`): `acme.example`.

   b. Applying the canonical lookup procedure ({{dii-lookup}}), the
      Resource Authorization Server queries DNS TXT at
      `_oauth-issuer-policy.acme.example`. The same record
      returned to the Client is returned here.

   c. The `authority=acme.example` directive matches. No `uri=` is
      present. The Resource Authorization Server constructs the same
      virtual policy the Client used in step 4.

   d. The ID-JAG `iss` value `https://idp.example.net` matches
      the single `authorized_issuers[0].issuer`. Verification
      succeeds.

10. The Resource Authorization Server validates `private_key_jwt`,
    then issues an access token in the response body.

## Migration Variant: Pointer Form

If `acme.example` later wants to express validity windows, format
restrictions, or multiple issuers with ordering, it can switch to
the pointer form without changing any consumer behavior:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   uri=https://delegation.example.org/acme.example.json"
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

Clients and Resource Authorization Servers transparently follow the
`uri=` directive and consume the JSON document. No software changes
are required at either consumer.

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
  Client has a cached virtual policy, the Client will still attempt
  the old issuer until its cache expires. Subject Authorities are
  encouraged to use short DNS TTLs during rotation; consumers
  enforce a local cache ceiling per {{dii-caching}}.

# Agent Platform IdP End-to-End Example

This appendix is non-normative.

This example shows how the framework prevents an unauthorized
provider from impersonating users in a customer's email namespace.
The protection rests on a single deliberate choice by the customer:
publishing an Issuer Authorization Policy that lists the specific
agent platforms permitted to issue identity assertions about its
users. Without that authorization, no provider, however well-known
elsewhere, can mint an accepted ID-JAG for the namespace.

## Cast

- **Customer**, `example.com`. Owns the email domain of its users.
  Decides which agent platforms are authorized to act as Assertion
  Issuers for its users.
- **Customer's Primary IdP**, `https://idp.example.com`. Used by
  the agent platform to federate user authentication. Mentioned
  only to describe the user-side SSO step; not the ID-JAG issuer
  and not part of the verification path at the tool provider.
- **Agent Platform**, `https://agentprovider.example`. Acts as a
  downstream identity provider for tools. Authenticates users via
  federated SSO from their primary IdP, then mints ID-JAGs with
  `iss = https://agentprovider.example` carrying the user's
  identity.
- **Tool Provider**, `https://toolprovider.example`. Acts as the
  Resource Authorization Server. Verifies presented ID-JAGs against
  its trust policy and issues access tokens.
- **End user**, Alice (`alice@example.com`).

## Publication

The customer publishes an Issuer Authorization Policy authorizing
exactly the providers it has chosen to trust as Assertion Issuers
for its users:

~~~
_oauth-issuer-policy.example.com.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=example.com;
   issuer=https://agentprovider.example"
~~~

This is the gatekeeping mechanism. The customer's domain owner is
the only party that can publish this record. Any provider not
listed cannot be authorized for `example.com` subjects, regardless
of how well-known the provider is in unrelated contexts.

The tool provider publishes a Trust Policy that consults
Domain-Authorized Issuer Discovery:

~~~ json
{
  "resource_authorization_server": "https://toolprovider.example",
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

## User-Side SSO (Prerequisite, Out of Scope)

Before any ID-JAG is minted, Alice establishes a session at the
agent platform. The exact mechanism is out of scope; a typical
flow:

1. Alice opens the agent platform's interface.
2. The agent platform performs federated sign-in to
   `https://idp.example.com` per OpenID Connect.
3. Alice authenticates at `idp.example.com`; an OIDC ID Token is
   returned to the agent platform.
4. The agent platform validates the ID Token, learns Alice's
   identity (`alice@example.com`), and creates a session.

After this step, `agentprovider.example` knows Alice's
organizational identity. The primary IdP (`idp.example.com`)
plays no further role at the tool provider; the agent platform
relies on its own keys to mint subsequent ID-JAGs.

## Flow

1. Within Alice's session, the agent platform mints an ID-JAG
   identifying Alice, with audience `https://toolprovider.example`
   (decoded payload, illustrative):

   ~~~ json
   {
     "iss": "https://agentprovider.example",
     "aud": "https://toolprovider.example",
     "exp": 1780166400,
     "iat": 1780166100,
     "jti": "b9c1...",
     "sub": "user-3f81a2",
     "email": "alice@example.com",
     "email_verified": true
   }
   ~~~

2. The agent platform (as the OAuth client) posts the ID-JAG to
   the tool provider's token endpoint with `private_key_jwt`
   client authentication (governed by the ID-JAG grant profile;
   outside the trust framework's scope):

   ~~~ http
   POST /token HTTP/1.1
   Host: toolprovider.example
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion=eyJhbGciOiJSUzI1NiIs...
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=eyJhbGciOiJFUzI1NiIs...
   ~~~

3. The tool provider validates the ID-JAG: signature (key
   resolved via
   `https://agentprovider.example/.well-known/oauth-authorization-server`
   JWKS), `aud`, `exp`, `iat`, replay protection.

4. The tool provider evaluates the trust policy's
   `subject_namespace_authorization` category against the
   assertion's subject ({{rasp}}):

   a. Extracts the Subject Authority from the assertion's email
      attribution. The ID-JAG carries `email = "alice@example.com"`
      with `email_verified = true` at top level; per
      {{dii-authority}}, the registrable domain of
      `alice@example.com` is `example.com`.

   b. Queries `_oauth-issuer-policy.example.com`. The record is
      `affirmative`, `authority=example.com` matches, and the
      virtual policy lists `https://agentprovider.example` as an
      authorized Assertion Issuer.

   c. The JWT `iss` (`https://agentprovider.example`) matches the
      `authorized_issuers` entry. The Trust Method is satisfied.

5. The tool provider's client authentication and OAuth client
   identifier resolution proceed per the grant profile and the
   tool provider's OAuth configuration. Trust framework
   evaluation does not govern them.

6. Verification succeeds. The tool provider issues an access
   token scoped according to its local policy.

## What This Protects Against

The key threat is **issuer impersonation**: a provider other than
the one the customer authorized attempts to mint an ID-JAG
claiming a user in the customer's namespace.

Suppose an unrelated provider `attacker.example` operates its own
authorization server and mints an assertion:

~~~ json
{
  "iss": "https://attacker.example",
  "aud": "https://toolprovider.example",
  "email": "alice@example.com",
  "email_verified": true,
  "exp": 1780166400,
  "iat": 1780166100,
  "jti": "c21f...",
  "sub": "user-58c1bd"
}
~~~

The signature validates (the assertion is signed by
`attacker.example`'s own key, which is resolvable at
`https://attacker.example/.well-known/oauth-authorization-server`).
The `aud` is correct. The token superficially names Alice.

The tool provider's Trust Method evaluation rejects it nonetheless:

- Subject Authority extracted from the assertion's email
  attribution: `example.com`.
- DNS lookup at `_oauth-issuer-policy.example.com` returns the
  customer's policy.
- The JWT `iss` (`https://attacker.example`) is NOT in
  `authorized_issuers` (which lists only
  `https://agentprovider.example`).
- The Trust Method is not satisfied for any
  `subject_namespace_authorization` method in the policy.
- The tool provider rejects with OAuth `invalid_grant`.

Note that the attacker setting `email_verified: true` on its own
assertion has no force here; trust comes from the `iss`-vs-policy
check, not from the assertion's self-claim. The Subject Authority
is computed from the email's domain regardless of the
`email_verified` value an unauthorized issuer chooses to assert.

`attacker.example` has no path to impersonate `alice@example.com`
unless `example.com`'s DNS publisher adds them to the Issuer
Authorization Policy. The customer is the sole gatekeeper.

## Additional Failure Variants

- If the customer's DNS record is missing or unreachable, the
  Trust Method outcome is Negative or Indeterminate per
  {{dii-failures}} and the tool provider rejects the assertion.
  An attacker cannot benefit from disabling the lookup;
  fail-closed semantics ensure rejection rather than acceptance.

- If `example.com` rotates its authorized agent platform (replaces
  `agentprovider.example` with a different provider in the
  policy), assertions from the old provider are rejected at the
  next cache refresh per {{dii-caching}}.

## Extensions

- The customer might prefer to grant trust transitively via
  federation rather than authorize each agent platform directly.
  In that case the tool provider's trust policy lists
  `openid_federation` as an `issuer_authentication` method with a
  trust anchor that recognizes `agentprovider.example` as a
  federated entity. The cross-category combination rule of
  {{rasp}} still requires a `subject_namespace_authorization`
  method to be satisfied for namespace-bound subjects; federation
  membership alone does not establish authority over the
  customer's namespace.

- If the deployment additionally needs to identify the specific
  agent instance (for audit, scoping, or downstream attribution),
  the agent platform can add an `act` object to the ID-JAG
  identifying the agent instance. The OAuth Actor Profile binding
  ({{actor-profile-binding}}) governs how a Resource Authorization
  Server interprets such an `act` object. Trust evaluation of the
  agent's identity by this framework's `subject_namespace_authorization`
  category applies only when the actor's subject identifier is in a
  format with a registered Subject Authority Extraction Procedure
  ({{iana-authority-registry}}); opaque agent-instance identifiers
  are typically not in such a format and are subject to the agent
  platform's authorization-time logic and the tool provider's local
  policy.

## Tenant Binding Variant

In some deployments, the Assertion Issuer is a shared-issuer
multi-tenant Identity Provider (for example, Google Workspace)
rather than a customer-specific provider. The framework supports
this case via the `tenant` member on `authorized_issuers[]` entries
({{dii-multi-tenant}}). This subsection sketches the same
gatekeeping flow as the rest of this appendix, with
`https://accounts.google.com` substituted for
`https://agentprovider.example` as the Assertion Issuer.

Publication uses the DNS pointer form because the inline DNS form
cannot express `tenant` ({{inline-form-feature-limits}}):

~~~
_oauth-issuer-policy.example.com.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=example.com;
   uri=https://example.com/.well-known/oauth-issuer-policy"
~~~

The HTTPS-hosted Issuer Authorization Policy binds authorization to
the customer's specific Google Workspace tenant:

~~~ json
{
  "subject_authority": "example.com",
  "authorized_issuers": [
    {
      "issuer": "https://accounts.google.com",
      "tenant": "example.com",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-01T00:00:00Z"
}
~~~

A successful ID-JAG carries the matching tenant claim:

~~~ json
{
  "iss": "https://accounts.google.com",
  "aud": "https://toolprovider.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "jti": "f47a...",
  "sub": "user-285dc1",
  "tenant": "example.com",
  "email": "alice@example.com",
  "email_verified": true
}
~~~

Verification at the tool provider differs from the main flow only
at step 4c: the matched `authorized_issuers` entry contains
`tenant: "example.com"`, and the ID-JAG's top-level `tenant` claim
is `"example.com"`. Both `iss` and `tenant` match, so the Trust
Method is satisfied.

**Threat: tenant impersonation.** Suppose an attacker provisions an
unrelated Google Workspace tenant (say, `attacker-corp`) and
attempts to mint an ID-JAG claiming `alice@example.com`:

~~~ json
{
  "iss": "https://accounts.google.com",
  "aud": "https://toolprovider.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "jti": "a5d9...",
  "sub": "user-attacker-tenant",
  "tenant": "attacker-corp",
  "email": "alice@example.com",
  "email_verified": true
}
~~~

The JWT `iss` matches the policy entry. The `tenant` claim does
NOT match: the customer's policy requires `tenant=example.com`, and the
attacker's assertion carries `tenant=attacker-corp`. The Trust
Method is not satisfied; the tool provider rejects with
`invalid_grant`. The attacker cannot set `tenant=example.com` in
their assertion because Google enforces that tenant `attacker-corp`
mints tokens with `tenant=attacker-corp`, not with another tenant's
identifier. This is the tenant-isolation property described in
{{dii-multi-tenant}}; the framework surfaces the customer's choice
of authorized tenant on the wire and makes the trust assumption
explicit, but does not eliminate the assumption.

# OpenID Federation End-to-End Example

This appendix is non-normative.

This example walks through an end-to-end flow in which a Resource
Authorization Server uses OpenID Federation to authenticate the
Assertion Issuer and Domain-Authorized Issuer Discovery to verify
that the Assertion Issuer is authorized for Alice's email namespace.

## Cast

- **Resource Authorization Server**, `https://api.resource.example`.
  Accepts ID-JAGs from Assertion Issuers that prove federation
  membership under the trust anchor below and are authorized by the
  asserted subject's namespace.
- **Federation Trust Anchor**, `https://federation.example.org`. Runs
  an OpenID Federation deployment that issues Subordinate Statements
  about intermediates and certifies Trust Marks.
- **Federation Intermediate**, `https://sector.example.org`. A
  sector-level intermediate (for example, a national or industry
  federation operator) chained under the Trust Anchor. Issues
  Subordinate Statements about leaf entities within its sector.
- **Assertion Issuer**, `https://idp.partner.example`. A federation
  leaf entity registered as an OpenID provider. Holds an Entity
  Configuration at
  `https://idp.partner.example/.well-known/openid-federation` and a
  Trust Mark from the Trust Anchor attesting Level of Assurance 3.
- **Client**, an enterprise SaaS application. Authenticates to the
  Resource Authorization Server's token endpoint with `private_key_jwt`.
- **End user**, Alice (`alice@partner.example`).

The federation chain in this example is three levels deep:

~~~
https://idp.partner.example          (leaf, OpenID Provider)
  authority_hints -> https://sector.example.org
                                        (intermediate)
  authority_hints -> https://federation.example.org
                                        (trust anchor)
~~~

## Publication

The Resource Authorization Server publishes authorization server
metadata:

~~~ json
{
  "issuer": "https://api.resource.example",
  "token_endpoint": "https://api.resource.example/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "identity_assertion_trust_policy_uri":
    "https://api.resource.example/.well-known/identity-assertion-trust-policy"
}
~~~

It publishes a trust policy listing one Trust Method from each
category. The `openid_federation` method requires both federation
membership AND a Level of Assurance 3 Trust Mark; the
`domain_authorized_issuer` method requires the Subject Authority's
explicit listing of the issuer.

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"],
      "trust_marks": [
        {
          "id": "https://federation.example.org/marks/loa3",
          "issuer": "https://federation.example.org"
        }
      ]
    },
    {
      "method": "domain_authorized_issuer"
    }
  ]
}
~~~

The Assertion Issuer, Federation Intermediate, and Federation Trust
Anchor publish federation artifacts per {{OIDF-FEDERATION}}. The
following payloads are illustrative; the publication mechanics
themselves (well-known URLs, federation fetch endpoints, signature
formats) are specified in {{OIDF-FEDERATION}}.

The Assertion Issuer's federation Entity Configuration (decoded
payload, illustrative):

~~~ json
{
  "iss": "https://idp.partner.example",
  "sub": "https://idp.partner.example",
  "iat": 1780166100,
  "exp": 1780252500,
  "authority_hints": ["https://sector.example.org"],
  "metadata": {
    "openid_provider": {
      "issuer": "https://idp.partner.example",
      "jwks_uri": "https://idp.partner.example/jwks",
      "token_endpoint_auth_methods_supported": [
        "private_key_jwt"
      ]
    }
  },
  "trust_marks": [
    {
      "id": "https://federation.example.org/marks/loa3",
      "trust_mark": "eyJ...(signed Trust Mark JWT issued by federation.example.org)"
    }
  ]
}
~~~

The Federation Intermediate's Entity Configuration:

~~~ json
{
  "iss": "https://sector.example.org",
  "sub": "https://sector.example.org",
  "iat": 1780166100,
  "exp": 1782758100,
  "authority_hints": ["https://federation.example.org"],
  "metadata": {
    "federation_entity": {
      "federation_fetch_endpoint":
        "https://sector.example.org/federation/fetch"
    }
  }
}
~~~

The Federation Intermediate's Subordinate Statement about the
Assertion Issuer carries a non-trivial `metadata_policy`
constraining the leaf's `issuer` value and required authentication
methods (per {{OIDF-FEDERATION}} §6); the policy-applied `issuer`
value is the one the framework requires to match the identity
assertion's `iss` claim ({{trust-method-openid-federation}}
requirement 1):

~~~ json
{
  "iss": "https://sector.example.org",
  "sub": "https://idp.partner.example",
  "iat": 1780166100,
  "exp": 1782758100,
  "metadata_policy": {
    "openid_provider": {
      "issuer": {
        "value": "https://idp.partner.example"
      },
      "jwks_uri": {
        "essential": true
      },
      "token_endpoint_auth_methods_supported": {
        "subset_of": ["private_key_jwt", "tls_client_auth"]
      }
    }
  }
}
~~~

The Federation Trust Anchor's Subordinate Statement about the
Federation Intermediate:

~~~ json
{
  "iss": "https://federation.example.org",
  "sub": "https://sector.example.org",
  "iat": 1780166100,
  "exp": 1782758100,
  "metadata_policy": {
    "federation_entity": {}
  }
}
~~~

The Subject Authority `partner.example` publishes an inline DNS policy
authorizing the federated Assertion Issuer:

~~~
_oauth-issuer-policy.partner.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=partner.example;
   issuer=https://idp.partner.example"
~~~

## Flow

1. The Client fetches authorization server metadata for the Resource
   Authorization Server and reads
   `authorization_grant_profiles_supported`; ID-JAG is supported.
   The Client fetches the trust policy URL.

2. The Client sees that the policy requires an
   `issuer_authentication` method and a
   `subject_namespace_authorization` method. It selects
   `https://idp.partner.example`, which it knows to be federated
   under `https://federation.example.org` and authorized by
   `partner.example` for email subjects.

3. The Client authenticates Alice at `idp.partner.example` (out of
   scope for this document). It then requests an ID-JAG with
   audience `https://api.resource.example` and Alice's email.

4. The Assertion Issuer returns the ID-JAG (decoded payload,
   illustrative):

   ~~~ json
   {
     "iss": "https://idp.partner.example",
     "aud": "https://api.resource.example",
     "exp": 1780166400,
     "iat": 1780166100,
     "jti": "8e9b...",
     "sub": "user-7c2a4f",
     "email": "alice@partner.example",
     "email_verified": true
   }
   ~~~

5. The Client posts to the Resource Authorization Server token
   endpoint using `private_key_jwt`:

   ~~~ http
   POST /token HTTP/1.1
   Host: api.resource.example
   Content-Type: application/x-www-form-urlencoded

   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion=eyJhbGciOiJSUzI1NiIs...
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=eyJhbGciOiJFUzI1NiIs...
   ~~~

6. The Resource Authorization Server evaluates
   `issuer_trust_methods`. For the `issuer_authentication`
   category, it evaluates `openid_federation` by delegating the
   procedural mechanics to {{OIDF-FEDERATION}} and then applying
   the framework-specific checks defined in
   {{trust-method-openid-federation}}.

   The OIDF chain walk (illustrative, per {{OIDF-FEDERATION}} §10
   and §6):

   a. Fetch the leaf Entity Configuration at
      `https://idp.partner.example/.well-known/openid-federation`.
      Read `authority_hints`: `https://sector.example.org`.

   b. Fetch the Federation Intermediate's Entity Configuration at
      `https://sector.example.org/.well-known/openid-federation`
      and the Subordinate Statement from the Intermediate about
      the leaf (via the Intermediate's `federation_fetch_endpoint`).
      Read the Intermediate's `authority_hints`:
      `https://federation.example.org`.

   c. Fetch the Federation Trust Anchor's Entity Configuration at
      `https://federation.example.org/.well-known/openid-federation`
      and the Subordinate Statement from the Trust Anchor about
      the Intermediate.

   d. Validate each Subordinate Statement's signature using the
      issuing entity's federation keys. All signatures validate.

   e. Apply the metadata policy from the Intermediate's
      Subordinate Statement about the leaf: `issuer` MUST equal
      `https://idp.partner.example`, `jwks_uri` MUST be present,
      `token_endpoint_auth_methods_supported` MUST be a subset of
      `[private_key_jwt, tls_client_auth]`. The leaf's metadata
      satisfies all constraints.

   f. Validate the Trust Mark on the leaf: the Trust Mark's
      signature validates against the Trust Anchor's federation
      keys.

   The framework-specific checks
   ({{trust-method-openid-federation}}) then run against the OIDF
   processing's outputs:

   - The terminal trust anchor `https://federation.example.org`
     matches the listed `trust_anchors` entry. (Requirement 1.)
   - The policy-applied metadata declares the `openid_provider`
     entity type. (Requirement 2.)
   - The Trust Mark with `id =
     https://federation.example.org/marks/loa3` issued by
     `https://federation.example.org` satisfies the policy's
     `trust_marks` requirement. (Requirement 4.)
   - The JWKS for ID-JAG signature validation will be taken from
     the policy-applied `metadata.openid_provider.jwks_uri`
     (`https://idp.partner.example/jwks`). (Requirement 3.)

   The `openid_federation` Trust Method is satisfied.

7. The Resource Authorization Server validates the ID-JAG's
   signature using the federation-resolved JWKS. It validates
   `aud`, `exp`, `iat`, and applies replay protection. The signing
   key resolution uses ONLY the federation-published JWKS; the
   Resource Authorization Server does NOT fetch
   `https://idp.partner.example/.well-known/oauth-authorization-server`
   for keys, per requirement 3 of
   {{trust-method-openid-federation}}.

   Because the policy also lists a `subject_namespace_authorization`
   method, the cross-category combination rule in {{rasp}} requires
   that category to be satisfied as well; step 8 supplies the
   additional evidence.

8. For the `subject_namespace_authorization` category, the Resource
   Authorization Server extracts `partner.example` from the
   assertion's top-level `email` claim (with `email_verified=true`),
   retrieves the Issuer Authorization Policy from
   `_oauth-issuer-policy.partner.example`, and verifies that
   `https://idp.partner.example` appears in `authorized_issuers`.

9. Verification succeeds. The Resource Authorization Server issues
   an access token in the response body. Client authentication
   (`private_key_jwt`) is handled by the ID-JAG grant profile and
   the underlying OAuth client-authentication mechanism; trust
   framework evaluation does not govern it.

## Failure Variants

- If the Assertion Issuer's Entity Configuration is unreachable or
  the federation chain does not terminate at
  `https://federation.example.org`, the Resource Authorization
  Server returns `invalid_grant`.

- If a different trust anchor appears in the chain but the policy
  lists only `https://federation.example.org`, the Resource
  Authorization Server returns `invalid_grant`.

- If the metadata policy applied during chain walking constrains
  the leaf's `issuer` value to a different string than the JWT's
  `iss` claim (for example, the Intermediate's Subordinate
  Statement requires `issuer = https://idp.partner.example` but
  the leaf's policy-applied metadata produces a different value),
  the Trust Method is not satisfied and the Resource Authorization
  Server returns `invalid_grant`.

- If the leaf's Entity Configuration does not include a Trust Mark
  matching the policy's `trust_marks` requirement (missing `id`,
  wrong issuer, or invalid Trust Mark signature), the Trust Method
  is not satisfied and the Resource Authorization Server returns
  `invalid_grant`.

- If the ID-JAG signing key resolved from the federation-published
  JWKS does not match the key used to sign the presented ID-JAG,
  the Resource Authorization Server returns `invalid_grant`. A
  separate JWKS published at the assertion `iss` URL outside the
  federation is not used; this prevents downgrade attacks via the
  AS metadata endpoint.

- If `partner.example` does not authorize
  `https://idp.partner.example` in its Issuer Authorization Policy,
  the Resource Authorization Server returns `invalid_grant`.

- If the federation chain validates but the client's
  `client_assertion` is not signed by a registered key (for example
  because of unfinished key rotation), the Resource Authorization
  Server returns `invalid_client`. This is a client-authentication
  failure governed by the grant profile and OAuth client-authentication
  mechanisms, not by the trust framework; it is included here only to
  show how trust-framework outcomes interleave with other token-endpoint
  errors.

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
