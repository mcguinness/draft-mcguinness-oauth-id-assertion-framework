---
title: "OAuth Identity Assertion Trust Framework"
abbrev: "Identity Assertion Trust Framework"
docname: draft-mcguinness-oauth-id-assertion-framework-latest
date: 2026-11-16
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
  - authority delegation

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8615:
  RFC9493:
  RFC9728:
  RFC9700:
  RFC9111:
  RFC5891:
  RFC8126:
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/
  UTS46:
    title: "Unicode IDNA Compatibility Processing (Unicode Technical Standard #46, Revision 36)"
    target: https://www.unicode.org/reports/tr46/tr46-36.html
    author:
      - org: Unicode Consortium
    date: 2026-08-31

informative:
  OIDF-FEDERATION:
    title: "OpenID Federation 1.1"
    target: https://openid.net/specs/openid-federation-1_1.html
    date: 2026-05-05
  RFC5321:
  RFC6530:
  RFC7009:
  RFC7033:
  RFC9989:
  RFC7662:
  RFC7515:
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  DAI:
    title: "OAuth Domain-Authorized Issuer Trust Method"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer/
    date: false
  FEDERATION:
    title: "OpenID Federation Trust Methods for Identity Assertion Grants"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-federation/
    date: false
  I-D.ietf-oauth-identity-chaining:
    title: "OAuth Identity and Authorization Chaining Across Domains"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/
    date: false
  NOAUTH:
    title: "nOAuth: How Microsoft OAuth Misconfiguration Can Lead to Full Account Takeover"
    target: https://www.descope.com/blog/post/noauth
    author:
      - org: Descope
    date: 2023-06-20

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

This document defines an Identity Assertion Trust Framework. Its
Identity Assertion Issuer Trust Policy is a JSON policy document that
a Resource Authorization Server publishes to declare which trust
methods it requires of an Assertion Issuer, including
issuer-authentication methods (such as OpenID Federation) and
subject-namespace authorization methods defined by separate
specifications. When an Identity Assertion JWT Authorization Grant
(ID-JAG) is presented at the token endpoint, the Resource
Authorization Server requires evidence from each category of method
that the policy lists, and fails closed when it cannot obtain that
evidence.

The Domain-Authorized Issuer Trust Method is defined separately as one
subject-namespace authorization method usable by this framework.
OpenID Federation Trust Methods, including one that recognizes the
OAuth client presenting an assertion, are also defined separately.

--- middle

# Introduction

The Identity Assertion JWT Authorization Grant (ID-JAG) {{ID-JAG}}, a
profile of the JWT-bearer grant {{RFC7523}}, lets an identity
assertion issued by one authorization server be presented to
another, and {{I-D.ietf-oauth-identity-chaining}} specifies
cross-domain delivery. Both leave to local policy whether the
Resource Authorization Server accepts an assertion's issuer for
subject resolution, account linking, or delegated access.

Today a Resource Authorization Server typically answers that question
through bilateral setup. Each customer:

- proves control of its domain,
- configures its Identity Provider with the Resource Authorization
  Server, and
- repeats both steps at every other Resource Authorization Server it
  uses.

Each copy of that configuration goes stale when the customer changes
Identity Providers, and none extends to issuers, such as agent
platforms, that have no prior relationship with the customer.

Without that configuration, a Resource Authorization Server that
trusts a multi-tenant issuer can accept an assertion from a tenant
that has no authority over the subject's domain. In the nOAuth
account takeover {{NOAUTH}}, an attacker set an email address on a
user in the attacker's own tenant, and an application linked
accounts by that address.

Namespace authorization lets the domain owner state once which
issuers and tenants may assert about its subjects, for every
Resource Authorization Server to check. It complements, and does not
replace, the Resource Authorization Server's own account-linking
defenses ({{scope-of-namespace-authorization}}).

This is an open-world problem: the set of acceptable Assertion
Issuers is often large, dynamic, or governed by external trust
frameworks, so it cannot be enumerated in advance. This document
therefore defines an Identity Assertion Issuer Trust Policy, in which
a Resource Authorization Server publishes the conditions an Assertion
Issuer must satisfy, such as membership in a recognized federation or
authority for the asserted subject namespace, not a list of trusted
Assertion Issuers. The Resource Authorization Server evaluates each
condition from concrete evidence when an assertion is presented.
Despite its name, the Trust Policy is published by the Resource
Authorization Server, not an Assertion Issuer. A Subject Authority
authorizes issuers in a separate document, the Issuer Authorization
Policy ({{DAI}}).

The trust evaluation that this document defines applies only to
ID-JAG ({{bindings}}). It narrows {{ID-JAG}}: it limits which claims
resolve the subject ({{rasp}} step 5c), requires `email_verified` with
a namespace-bound `email` ({{id-jag-profile}}), and requires the
verification key to be bound to `iss` ({{rasp}} step 2).


## Overview {#overview}

A client presents an ID-JAG for `alice@acme.example` at a Resource
Authorization Server's token endpoint. The Resource Authorization
Server answers two separate questions about it:

- Issuer authentication: who signed it? The server checks that the
  issuer named by `iss` is a recognized signer, for example through
  an OpenID Federation trust chain ({{FEDERATION}}), and that the
  signature verifies under a key that this method permits.

- Namespace authorization: is that issuer authorized for
  `acme.example`? Only the owner of `acme.example`, its Subject
  Authority, can say. Under {{DAI}}, it publishes through DNS an
  Issuer Authorization Policy that lists the Assertion Issuers it
  authorizes. A recognized signer can name any domain, so a yes to
  the first question does not answer the second.

The Resource Authorization Server publishes a Trust Policy
({{trust-policy-document}}) that names the Trust Methods it requires.
When the policy lists methods in both categories, as below, both
must pass:

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
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

{{rasp}} gives the procedure. Step 2 verifies the signature, step 5b
requires a satisfied method in each category that the policy lists,
and step 5c requires a Subject Authority (here `acme.example`, from
the `email` claim) and, unless local configuration says otherwise,
limits subject resolution to that claim and issuer-scoped
identifiers. {{FEDERATION}} adds `client_recognition` methods,
which evaluate the client that presents the assertion.
{{example-agent-platform}} walks through an exchange that uses
`domain_authorized_issuer` alone.

## Motivating Use Cases {#motivation}

The gap between issuer authentication and namespace authorization
matters in:

- Workforce SSO into multi-vendor SaaS, where per-customer issuer
  configuration has no wire-format check that the customer
  authorizes the configured Assertion Issuer.
- AI agent platforms acting across tool boundaries, where the tool
  needs to know the platform is entitled to assert about users in
  the customer's namespace ({{example-agent-platform}}).
- B2B integrations carrying end-user identity, which today accept
  any authenticated Assertion Issuer or maintain manual allowlists.
- Shared-issuer multi-tenant Identity Providers, where the
  customer's choice of authorized tenant becomes observable on the
  wire rather than implicit
  ({{DAI}} §Single-Issuer Multi-Tenant Identity Providers).

## Relationship to Existing Mechanisms

{{FEDERATION}} relates this framework to OpenID Federation. Issuer
discovery mechanisms, such as WebFinger {{RFC7033}} and OpenID
Connect Discovery {{OIDC-DISCOVERY}}, answer "which issuer should a
client use for this user?" before authentication. This framework
answers the verifier-side question "may this issuer, having already
produced an assertion, be trusted for this subject's namespace?"
{{DAI}} §Following Existing DNS Authority Patterns compares DAI with
related DNS and domain-verification mechanisms.


## Documents in the Family {#family}

This document, {{DAI}}, and {{FEDERATION}} form a three-document set:

- **This document** defines the Trust Policy
  ({{trust-policy-document}}) and the token-endpoint procedure that
  evaluates it ({{rasp}}).
- **{{DAI}}** defines the `domain_authorized_issuer` Trust Method and
  the Issuer Authorization Policy that it consumes.
- **{{FEDERATION}}** defines the `openid_federation` Trust Method and
  the `client_recognition` category, which evaluates the client that
  presents an assertion.

A Resource Authorization Server that uses only {{DAI}} does not need
{{FEDERATION}}.


## Scope {#scope}

The scope of this document is:

- The Trust Policy document ({{trust-policy-document}}).
- The Trust Method machinery (categories, registry).
- The Subject Authority Determination concept and registry.
- The token-endpoint procedure ({{rasp}}) and the rules it applies
  ({{delegation-model}}): combination rule, lookup states, and
  fail-closed handling.
- The binding to ID-JAG ({{bindings}}).

Trust Methods, and the publication mechanisms they use, live in
dedicated specifications such as {{DAI}} and {{FEDERATION}}.

This document registers no Trust Methods for attribute authorities
that are not namespace owners, such as issuers of government
credentials, professional certifications, or employment
attestations. When such an attestation is conveyed in an identity
assertion whose subject is namespace-bound, the issuer-trust
evaluation still applies.

The `email` Subject Identifier extraction registered in this
document is user-identity-oriented. Workload, agent, and other
non-user subjects can be supported by registering additional
extractions ({{iana-authority-registry}}).

## Conventions

{::boilerplate bcp14-tagged}

# Terminology {#terminology}

This document uses OAuth 2.0 {{RFC6749}} terminology. Subject
Identifier formats follow {{RFC9493}}; Trust Method identifiers are
registered in {{iana-trust-methods-registry}}. This document defines
the following terms.

Resource Authorization Server (RAS):
: The OAuth authorization server that receives an identity
assertion as a grant, evaluates the Trust Policy, and issues access
tokens for a protected resource. It is not a new OAuth protocol
role.

Identity assertion:
: A signed statement that an Assertion Issuer makes about a subject,
presented to a Resource Authorization Server as an authorization
grant. It is typically a JWT {{RFC7519}}.

Assertion Issuer:
: The service issuing an identity-bearing assertion, identified by
the JWT `iss` claim. The same string is the `issuer` value in
{{RFC8414}} and {{OIDC-DISCOVERY}} metadata and the federation entity
identifier in {{OIDF-FEDERATION}}. It is a separate role from the
client that presents the assertion, but one party, such as an agent
platform, can play both.

Grant profile:
: The OAuth assertion-grant specification under which an identity
assertion arrives, such as {{ID-JAG}} ({{bindings}}).

Subject Authority:
: The party that holds authority over a Subject Identifier namespace
and can authorize Assertion Issuers to assert identities in it. The
term also names the identifier of that authority, which for `email`
is the registrable domain.

Trust Policy:
: The JSON document defined in {{trust-policy-document}}, published
by a Resource Authorization Server to declare its trust criteria.

Trust Method:
: A registered, named evaluation procedure by which a Resource
Authorization Server tests trust evidence in one or more categories,
such as evidence about an Assertion Issuer ({{trust-methods}}). A Trust
Policy names the Trust Methods it requires.

Issuer Authorization Policy:
: The document by which a Subject Authority declares the Assertion
Issuers it authorizes for its namespace. Its wire format,
publication channel, and integrity protection are defined by each
`subject_namespace_authorization` Trust Method specification.

Consumer:
: Any party that retrieves and processes a policy document (Trust
Policy or Issuer Authorization Policy): a Resource Authorization
Server evaluating an assertion, or a client reading a Trust Policy
for capability discovery. Requirements addressed to consumers apply
to both roles unless a narrower role is named.

Decision-affecting member:
: A member of a policy document whose value can change whether a
consumer accepts an assertion. Every member that this document or a
Trust Method specification registers for a policy document is
decision-affecting unless it only records when the document was
published, carries a signature over other members, or has a
registration that says otherwise.

# Trust Policy

## Metadata Publication {#metadata-publication}

A Resource Authorization Server publishes the location of its Trust
Policy in its authorization server metadata {{RFC8414}}:

`identity_assertion_trust_policy_uri`
: OPTIONAL. HTTPS URI of the Trust Policy document.

A Resource Authorization Server that applies different Trust Policies
to different clients publishes the one for clients without a prior
relationship with it. {{FEDERATION}} defines how a policy is
selected by authenticated client. How a registered client learns its
own policy is out of scope.

A Protected Resource that publishes OAuth 2.0 Protected Resource
Metadata {{RFC9728}} MAY include the same member to advertise the
policy applied by its Resource Authorization Server. Clients that
obtain the URI from Protected Resource Metadata MUST verify that the
policy document's `resource_authorization_server` value identifies an
authorization server listed in the Protected Resource's
`authorization_servers` array. If both metadata documents carry the
member, the authorization server metadata value is authoritative.

A Resource Authorization Server may also publish the Trust Policy at
the well-known URI registered in {{iana-trust-policy-well-known}}. The
URI is derived from its issuer identifier as in {{RFC8414}} Section 3:
the well-known path component `identity-assertion-trust-policy` is
inserted between the host (and port, if any) and any path component.
For an issuer with no path component this yields
`https://{host}/.well-known/identity-assertion-trust-policy`. If the
metadata member and the well-known URI identify different documents,
the metadata member is authoritative.

A consumer fetches a Trust Policy document, from either source, by
HTTP GET over HTTPS with TLS server authentication, under the rules
below. The consumer MUST NOT act on a policy it could not retrieve
and validate. A retrieval failure differs from a retrieved policy
that rejects an assertion.

- The response MUST have status 200 and a media type of
  `application/json` or a `+json`-suffixed type.
- Consumers MUST follow HTTPS redirects up to a limit of 5 hops.
  Every redirect target MUST use the `https` scheme and MUST remain
  within the origin of the URI being fetched. A redirect to a
  different origin, or beyond the hop limit, is a retrieval failure.
- Any other status, a TLS failure, an unparseable body, or a body
  exceeding the consumer's limit is a retrieval failure. That limit
  MUST allow at least 64 KiB.

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
    "https://api.resource.example/trust-policy"
}
~~~

## Trust Policy Document {#trust-policy-document}

The Identity Assertion Issuer Trust Policy is a JSON object. Consumers:

- MUST reject a policy whose required members are absent or
  wrong-typed;
- MUST reject a document containing duplicate member names at any
  object level; and
- MUST ignore unrecognized members except those named in `crit`
  ({{critical-members}}).

The policy MAY include a `crit` member ({{critical-members}}). It
carries no signature ({{integrity}}). Members are registered in
{{iana-trust-policy-members-registry}}.

Example, using `openid_federation` ({{FEDERATION}}) and
`domain_authorized_issuer` ({{DAI}}):

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
this policy supports, in the identifier space of the same-named
metadata parameter ({{ID-JAG}} §7.2). Each value MUST match a profile
identifier listed in the authorization server metadata's
`authorization_grant_profiles_supported` parameter, where present.

`subject_identifier_formats_supported`
: OPTIONAL. JSON array of Subject Identifier format names, defined by
{{RFC9493}} or other specifications, that the Resource Authorization
Server accepts. This member applies to all Trust Methods. An
individual Trust Method MAY constrain the formats it evaluates. For
example, `domain_authorized_issuer` evaluates only formats registered
in {{iana-authority-registry}}.

`issuer_trust_methods`
: REQUIRED. Non-empty JSON array of Trust Method objects
({{trust-methods}}). It states the trust requirements the Resource
Authorization Server enforces against incoming identity assertions,
not a list of capabilities. An assertion is rejected unless an
Assertion Issuer satisfies the Trust Method combination rule in
{{rasp}} against the methods listed here. An issuer that satisfies a
method not listed here is not acceptable. Local policy may add
requirements per client, subject, or scope ({{downgrade}}). An object
in this member counts only toward the issuer categories,
`issuer_authentication` and `subject_namespace_authorization`. A
method that belongs to no issuer category is malformed here.

## Trust Methods {#trust-methods}

### Trust Method Object Structure {#trust-method-structure}

A Trust Method object is a JSON object with a string-valued `method`
member naming a Trust Method identifier registered in
{{iana-trust-methods-registry}} plus any members required by that
identifier. An object whose required members are absent, wrong-typed,
or out-of-constraint is malformed. Unrecognized members of a
recognized object are ignored.

A client reading the policy for capability discovery MAY skip
objects it does not recognize, since it cannot satisfy them. For an
unrecognized or malformed object, a Resource Authorization Server
evaluating an assertion MUST apply {{rasp}} step 5a and reject the
assertion.

### Trust Method Categories {#trust-method-categories}

Each Trust Method belongs to one or more categories registered in
{{iana-trust-method-categories-registry}}, such as the two below, and
is itself registered in {{iana-trust-methods-registry}}. Evidence is
combined across categories per {{combination-rule}}.

`issuer_authentication`
: Is the entity identified by the JWT `iss` claim authentically a
member of a recognized ecosystem?

`subject_namespace_authorization`
: Is this Assertion Issuer entitled to assert about subjects in the
named namespace?

Deployments accepting assertions about namespace-bound subjects
SHOULD list at least one `subject_namespace_authorization` method.
Future specifications MAY register additional categories
({{iana-trust-method-categories-registry}}). {{FEDERATION}} registers
`client_recognition`, which evaluates the client that presents the
assertion.

### Requirements on Trust Method Specifications {#trust-method-spec-requirements}

A specification defining a new Trust Method (a companion such as
{{DAI}} or {{FEDERATION}}) MUST provide all of the following, so that
the framework's deferral to the method is testable:

- The Trust Method identifier and its categories
  ({{iana-trust-methods-registry}}).
- The evidence consumed and the exact procedure by which a Resource
  Authorization Server decides the method is satisfied for an (issuer,
  subject) pair, or for the party that the method's category
  evaluates, precise enough for interoperable implementation.
- The mapping of retrieval and evaluation outcomes onto the
  Affirmative / Negative / Indeterminate lookup states
  ({{exception-handling}}), including which conditions fail closed.
- For `subject_namespace_authorization` methods, the deterministic
  binding from the Subject Identifier to the Subject Authority
  consulted, consistent with the single-source-selection rule
  ({{multiple-sources}}).
- Cache-lifetime bounds for retrieved evidence, including any bound
  on serving cached evidence during an Indeterminate state, or an
  explicit statement that the method caches nothing.
- For signed evidence, how the signer and the verification key are
  bound to the party that holds the authority the evidence expresses
  (such as the Subject Authority for its namespace), through a channel
  independent of the signed evidence, so that an
  attacker who controls the evidence or the channel that carries it
  cannot substitute either.
- Method-specific parameters, their JSON types, and whether each is
  REQUIRED or OPTIONAL.

Evidence that a Trust Method consumes about an Assertion Issuer MUST
carry enough information to identify the party that holds the
authority it expresses (such as the Subject Authority, or the superior
that issued a federation Subordinate Statement), the Assertion
Issuer, the claim type or scope it
covers, and, when applicable, its validity bounds.

### Subject Namespace Authorization Methods {#subject-namespace-authorization-methods}

`subject_namespace_authorization`-category methods are satisfied
by evidence from the Subject Authority itself, typically a DNS or
HTTPS record published under its domain. These methods are bounded
at depth one: the Subject Authority directly lists every authorized
Assertion Issuer, and no further delegation is permitted
({{transitive-authz-bounded}}).
Companion specifications define the concrete methods in this
category and register them in {{iana-trust-methods-registry}}.

## Subject Authority Determination {#subject-authority-determination}

Determining the Subject Authority has two steps: identify which
registered Subject Identifier format the assertion carries, then apply
that format's extraction procedure. The grant-profile binding
({{bindings}}) designates which claim conveys the Subject Identifier
and thus which format applies. A format is "used by the assertion"
when the designated claim for that format is present. If the
designated claim maps to no format registered in
{{iana-authority-registry}}, the assertion yields no
Subject Authority, the `subject_namespace_authorization` category
cannot be evaluated, and processing follows {{rasp}} step 5c.

Extraction procedures are registered in {{iana-authority-registry}};
this document registers the `email` extraction as the initial entry.
Future specifications can register further formats without changing
the rest of the framework.

Initial extractions:

`email`
: The Subject Authority is derived from the assertion's top-level
  `email` claim. That claim MUST be accompanied by a top-level
  `email_verified` claim with the boolean value `true`. If the
  `email_verified` claim is absent or has any value other than
  `true`, consumers MUST treat the `email` Subject Identifier as
  invalid for Subject Authority determination and MUST reject the
  assertion when `email` is the Subject Identifier being evaluated.
  Consumers MUST NOT treat an unverified `email` claim as though the
  assertion carried no Subject Identifier.

  The domain is the substring after the single `@`. Consumers MUST
  reject an `email` claim value that does not contain exactly one `@`
  character or whose domain part is empty. This document uses this
  single-`@` rule, not the full {{RFC5321}} Mailbox grammar. Quoted
  local-parts and other forms that do not reduce to a single unquoted
  `@` are out of scope for the `email` extraction. The local-part is
  not used. A trailing dot on the domain is removed.

  The domain is converted to A-label form ({{RFC5891}}) by UTS #46
  `ToASCII` processing {{UTS46}} with these flags:

  - `Transitional_Processing=false` (nontransitional processing).
    Transitional processing is deprecated in UTS #46.
  - `IgnoreInvalidPunycode=false`
  - `UseSTD3ASCIIRules=true`
  - `CheckHyphens=true`
  - `CheckBidi=true`
  - `CheckJoiners=true`
  - `VerifyDnsLength=true`

  Consumers MUST reject a domain for which `ToASCII` reports an
  error. `UseSTD3ASCIIRules` rejects characters not valid in host
  names, such as `_`, which an email domain does not need. These
  flags pin one mapping, so that two consumers compute the same
  A-label. Transitional processing or IDNA2003 maps some characters,
  such as U+00DF, differently and would select a different Subject
  Authority.

  The A-label domain is then normalized to its registrable domain
  ("eTLD+1") by applying the Public Suffix List matching algorithm
  {{PSL}}, including its wildcard `*` and exception `!` rules. The
  Subject Authority is the shortest suffix of the domain that is one
  label longer than the longest matching public suffix. Consumers
  MUST use both the ICANN and PRIVATE divisions of the list. A
  delegated namespace listed in the PRIVATE division, such as
  `team.example-pages.example`, resolves to that namespace. Using
  only the ICANN division would compute a coarser Subject Authority
  and query a different name.

  The result is compared using case-insensitive ASCII comparison of
  A-labels. Consumers MUST reject an email whose domain is itself a
  public suffix (no registrable domain exists).

Subject Authority extraction MUST be exact-match: wildcard, suffix,
regular-expression, and substring matching against Subject Authority
values are forbidden unless explicitly specified by the relevant
Subject Identifier format's extraction procedure.

### Public Suffix List Versioning {#psl-versioning}

No IETF standard defines how to derive a registrable domain from a
DNS name. The IETF
DBOUND working group studied administrative (organizational)
boundaries in the DNS and concluded without a standard. The Public
Suffix List is the de facto mechanism, and browsers use it for
cookie same-site rules. DMARC, however, replaced its use of the list
with a DNS tree walk in {{RFC9989}} (Section 4.10). This framework
knowingly inherits the PSL's limitations. Whether a DNS tree walk
suits Subject Authority determination is left to future work.

The PSL {{PSL}} is updated continuously, and snapshots taken at
different times can yield different registrable domains for the same
email. The Subject Authority is the DNS name queried. So two Resource
Authorization Servers using different PSL snapshots can compute
different Subject Authorities for the same assertion, query
different names, and reach different decisions. While a label's
public-suffix status is changing, this divergence is a
deterministic-source-selection hazard ({{multiple-sources}}) and a
security consideration, not merely an operational one. Consumers
SHOULD use a snapshot of the {{PSL}} no older than 30 days.

The determinism that this framework claims holds for a given
computed Subject Authority. Parties that require identical Subject
Authority computation across verifiers, such as a Subject Authority
and the Resource Authorization Servers that consume its policy,
SHOULD agree on, or pin, a PSL snapshot. Subject Authorities SHOULD
monitor PSL changes affecting their namespace.

### Subdomain Authority {#subdomain-authority}

The registrable-domain default prevents an attacker who controls a
subdomain, for example through subdomain takeover, from overriding
the record at the registrable domain. A future subdomain-exact
variant would require explicit delegation from the registrable-domain
authority.

## Critical Members {#critical-members}

The Trust Policy MAY include a `crit` member. It is a JSON array of
strings naming other members of the same document whose correct
processing is REQUIRED for safe interpretation. A Trust Method
specification MAY adopt this mechanism, with the rules of this section,
for the policy documents it defines (for example, {{DAI}}).

A consumer that does not recognize, or does not implement processing
for, any member named in `crit` MUST reject the document as malformed.
`crit` names members of the object that carries it, not members of
nested objects.

When present, `crit` MUST be a non-empty array of strings. Each
string SHOULD name a member that is present in the document. A
consumer MUST reject the document if `crit` is present but not a
non-empty array of strings, and MUST reject if `crit` names `crit`
itself. JWS uses the same fail-closed pattern for critical header
parameters ({{RFC7515}} Section 4.1.11).

This document defines `crit` but names no member critical by
default. Defining it here lets a future extension mark a new
decision-affecting member critical and have already-deployed
consumers honor it, as {{FEDERATION}} does for
`client_trust_methods`.

# Trust Policy Processing

The Trust Policy governs whether an Assertion Issuer's identity
assertion is acceptable to the Resource Authorization Server. It is
necessary but not sufficient: it does not, by itself, authorize any
particular access, scope, role, or attribute. Local policy decides
which scopes an accepted assertion receives and how its subject is
linked to an account, within the limits of {{rasp}} step 5c.

## Client Processing {#client-processing}

A client that wants to obtain an identity assertion JWT for a Resource
Authorization Server:

1. Discovers the Resource Authorization Server metadata {{RFC8414}}.

2. Confirms that the authorization server metadata lists the desired
   grant profile in `authorization_grant_profiles_supported`
   ({{ID-JAG}} §7.2).

3. Fetches the `identity_assertion_trust_policy_uri` document.

4. Determines whether an available Assertion Issuer can satisfy the
   Trust Method combination rule in {{rasp}}.

5. Determines whether the Assertion Issuer can produce a Subject
   Identifier in a format listed in
   `subject_identifier_formats_supported`, if that member is present,
   and required by the applicable grant profile.

6. Requests an identity assertion JWT from the selected Assertion
   Issuer.

The client MUST NOT treat the Trust Policy as a guarantee that a
particular assertion will be accepted.

## Resource Authorization Server Processing {#rasp}

The following procedure is the Resource Authorization Server's trust
decision. {{delegation-model}} states the rules it applies to each
Trust Method's evidence.

When evaluating an identity assertion JWT presented in a token
request, the Resource Authorization Server MUST:

1. Select and parse the Trust Policy that applies to the token request.
   A Resource Authorization Server that holds more than one Trust
   Policy never selects one from a claim in the assertion
   ({{category-applicability}}). An extension, such as {{FEDERATION}},
   can define selection from authenticated properties of the request.
   Any other request gets the policy that applies without regard to
   those properties, such as the one the Resource Authorization
   Server publishes ({{metadata-publication}}). If the policy document
   is malformed, reject the assertion. Step 5a handles recognition of
   individual Trust Method objects.

2. Validate the assertion per the applicable grant profile
   ({{RFC7521}}, {{RFC7523}}, {{ID-JAG}}). The verification key MUST
   be bound to the `iss` value independently of the assertion,
   through the `jwks_uri` in that issuer's metadata (located per
   {{ID-JAG}} §7.1, including its fallback to OpenID Connect
   Discovery), local configuration for that issuer, or a key source
   authenticated by a Trust Method that counts toward step 5b. A key
   or key location in the assertion's JOSE header, such as `jwk`,
   `jku`, or `x5u`, satisfies this only when one of those sources
   authenticates it. This strengthens {{ID-JAG}} §4.4.1, which only
   recommends `jwks_uri`.

   A Trust Method MAY constrain the source of the verification key.
   For example, `openid_federation` requires a federation-resolved
   key set ({{FEDERATION}}). Such a method MUST NOT count toward step
   5b unless the assertion signature verifies under a key it permits,
   so a signature verified only under an unconstrained source, such as
   the `iss` URL's authorization server metadata, does not satisfy
   it. Signature verification can therefore complete in step 5, once
   the method's key source is resolved.

3. Verify that the applicable grant profile has a binding
   ({{bindings}}) and is listed in
   `authorization_grant_profiles_supported`.

4. If `subject_identifier_formats_supported` is present, verify that
   the Subject Identifier format used by the assertion is listed,
   unless the applicable grant profile defines a different format
   selection rule.

5. Verify that the Assertion Issuer identified by `iss`, and any
   other party that a listed category evaluates, satisfy the Trust
   Method combination rule:

   a. Partition the Trust Method objects in `issuer_trust_methods`,
      and in any other Trust Policy member that lists Trust Method
      objects, by category ({{trust-method-categories}}). A Trust
      Method registered with multiple categories belongs to each
      category that its member allows ({{trust-policy-document}}). A
      method listed in the wrong member is malformed
      ({{trust-policy-document}}). The Resource Authorization Server
      MUST reject the assertion if any object in such a member is not
      recognized, is malformed, or cannot be assigned to a category it
      knows. Silently ignoring the object would evaluate a strict
      subset of the requirements the operator declared: a fail-open
      category downgrade. A single evidence item MUST NOT be counted
      toward more than one category even when its Trust Method is
      registered in several.

   b. Each category present in the partitioned policy is applicable to
      every assertion evaluated under this policy, and nothing in the
      assertion can waive it ({{category-applicability}}). For each
      present category, the party that the category evaluates, which
      for the categories this document defines is the Assertion
      Issuer, MUST satisfy at least one Trust Method from that
      category, subject to the finality rule of {{multiple-sources}}.

   c. Whatever categories the policy lists, the Resource Authorization
      Server MUST identify the subject, for subject resolution, account
      linking, and authorization, only by issuer-scoped identifiers or
      by the Subject Identifier that a
      `subject_namespace_authorization` method evaluated in step 5b,
      unless local configuration trusts the Assertion Issuer for
      another subject claim. That exception does not cover a
      namespace-bound Subject Identifier that no namespace method
      evaluated, which step 5e governs. Issuer-scoped identifiers are
      `iss` and `sub`, with `tenant` when the assertion carries it
      ({{ID-JAG}} §6.3). Claims such as `aud_sub` and `sub_id`
      ({{ID-JAG}} §3.1) are outside the namespace check. An account
      linked by `email` MUST be linked on
      the address with its full domain in the A-label form that
      {{subject-authority-determination}} computes, not on the
      registrable domain or the result of any other Unicode mapping.
      In addition, when the policy contains a
      `subject_namespace_authorization` Trust Method:

      - The assertion MUST carry a Subject Identifier from which a
        Subject Authority can be determined per
        {{subject-authority-determination}} or an equivalent
        registry. If the assertion carries no such Subject
        Identifier, or no Subject Authority can be determined, the
        Resource Authorization Server MUST reject the assertion.
      - The Resource Authorization Server MUST NOT treat the
        `subject_namespace_authorization` category as inapplicable
        because the assertion omits a Subject Identifier.
      - When the namespace category is present but cannot be
        evaluated, the Resource Authorization Server MUST NOT consume
        any subject or identity claim from the assertion for account
        linking, authorization, or subject resolution.

   d. Local policy MAY require satisfaction of additional Trust
      Methods for specific clients, subjects, or scopes.

   e. When the policy lists no `subject_namespace_authorization`
      method and the assertion carries a namespace-bound Subject
      Identifier, the Resource Authorization Server MUST reject the
      assertion unless local policy independently establishes
      authority over the subject namespace. A namespace-bound Subject
      Identifier is one whose format has a registered Subject
      Authority extraction procedure ({{iana-authority-registry}}), so
      the Resource Authorization Server can recognize one without
      running a namespace method. Federation membership alone does not
      establish authority over a particular subject namespace.

6. Apply local policy (account-linking, consent, authorization,
   risk) and the applicable grant profile's sender-constraining;
   this document specifies neither.

Client authentication that the grant profile requires SHOULD be
completed before step 5, so that unauthenticated requests cannot
cause the lookups that step performs. Every fetch in this processing
follows {{outbound-fetches}}.

Rejection at any of steps 1 through 5 MUST result in an OAuth
`invalid_grant` error unless another error is defined by the
applicable grant profile, by the specification of a category that
the Trust Policy lists, or by the Trust Method that failed. This rule
covers a malformed Trust Policy (step 1) and an unlisted grant
profile (step 3). Detailed trust-evaluation failure state MUST NOT be
returned to public clients in the OAuth error response. That state is
a reconnaissance target.

A specification that registers a new Trust Method category states
where in this procedure its checks run, how its errors rank against
those defined here, and how its Trust Methods select among the
evidence sources a Trust Policy lists ({{multiple-sources}}).

## Trust Evaluation Rules {#delegation-model}

The procedure in {{rasp}} and every Trust Method specification follow
the rules below. A Trust Method's evidence is what it consults, such
as an Issuer Authorization Policy or a federation trust chain. Its
evidence source is the trust anchor, registry, publication channel,
or configured source from which the Resource Authorization Server
accepts that evidence, such as a Subject Authority's DNS zone or a
federation trust anchor. Evaluating evidence from parties the Resource
Authorization Server did not configure in advance does not mean
accepting any party: the server publishes or locally configures which
evidence sources it accepts.

### Cross-Category Combination Rule {#combination-rule}

Step 5b of {{rasp}} states the cross-category combination rule:
*and* across categories, *or* within a category. An evidence item is
the outcome of evaluating one Trust Method that the Trust Policy
lists. Each category that the policy lists needs at least one
satisfying evidence item, subject to the finality rule of
{{multiple-sources}}.

Satisfying one category MUST NOT be treated as satisfying another.
Federation membership does not authorize an Assertion Issuer for a
namespace, and a Subject Authority's authorization does not
authenticate the Assertion Issuer beyond what the Issuer
Authorization Policy attests.

#### Category Applicability {#category-applicability}

Category applicability is a property of the Resource Authorization
Server's published Trust Policy, not of the incoming identity
assertion. A Resource Authorization Server that holds more than one
Trust Policy never selects one from properties of the identity
assertion ({{rasp}} step 1). The identity assertion itself MUST NOT
be able to waive a listed category.

Trust Method specifications MUST NOT define applicability conditions
that depend on properties of the identity assertion under
evaluation. The prohibition is directional: assertion properties
never relax what the policy requires, but rejection rules keyed on
them, such as {{rasp}} step 5e, only tighten it and are permitted.
Nor can published evidence waive the category. A Trust Method
specification MUST NOT define a published state, such as a
monitoring mode, under which an identity assertion whose issuer the
evidence does not authorize satisfies the category. Under step 5c,
omitting a Subject Identifier does not make the
`subject_namespace_authorization` category inapplicable
({{applicability-bypass}}).

### Multiple Evidence Sources Within a Category {#multiple-sources}

A Resource Authorization Server MAY accept evidence from multiple
evidence sources within the same category (multiple Subject
Authorities for different namespaces, multiple federation trust
anchors). For `subject_namespace_authorization`, a Trust Method
specification MUST define deterministic source selection: a binding
function from explicitly defined inputs (a single named claim,
header, request parameter, or fixed tuple) to exactly one evidence
source or a deterministic failure, with no preference ordering and no
fallback across candidates, and invariant under the presence,
absence, or value of any input outside the function
({{unverified-claim}}). Within such a method's evaluation, a Resource
Authorization Server MUST NOT fall through to a different evidence
source if the selected evidence source's evaluation fails or is
indeterminate ({{exception-handling}}).

For `issuer_authentication`, every evidence source the Trust Policy
lists is equally trusted, so a Trust Method specification MAY accept
evidence from any of them. For `openid_federation`, this means a
chain to any listed trust anchor ({{FEDERATION}}).

Across Trust Methods in the same category, or-semantics apply, with
one exception, the finality rule. When a
`subject_namespace_authorization` method retrieves Affirmative
evidence from the Subject Authority that does not authorize the
Assertion Issuer, or explicitly denies it, the category is not
satisfied, whatever other methods in the category yield: the Subject
Authority's published decision is final for its namespace. An
Indeterminate state from such a method that consults the Subject
Authority's own publication has the same effect, because it may hide
that decision. A Negative state, in which the Subject Authority has
published nothing through that method's channel, does not: other
methods in the category can still supply evidence. If that channel
does not authenticate negative answers, as with DNS without DNSSEC,
an attacker who forges one can turn a published denial into a
Negative and let another method decide ({{DAI}} §DNS Integrity and
Compromise).

For a `subject_namespace_authorization` method that consults the
Subject Authority's own publication:

| Outcome of that method | Effect on the category |
|-|-|
| Affirmative, issuer authorized | Satisfies it, unless another method in the category yields one of the next two outcomes |
| Affirmative, issuer not authorized (including an explicit denial) | Final: the category is not satisfied, whatever other methods yield |
| Indeterminate | Final, as above: it may hide a published decision |
| Negative: nothing published through that channel | None: other methods in the category decide |

### Lookup States and Fail-Closed {#exception-handling}

Every lookup of a Trust Method's evidence, in any category, produces
exactly one of three states. Trust Method specifications MUST map
every concrete outcome of the lookup operation onto exactly one of
these states and enumerate the concrete signals on their publication
channel that map to each state.

#### Lookup States

- **Affirmative**: well-formed evidence was retrieved through the
  authoritative publication channel, its signature (where applicable)
  verified, and its validity bounds hold at evaluation time. The
  Resource Authorization Server then evaluates the identity
  assertion against that evidence.

- **Negative**: the publication channel authoritatively reports the
  absence of evidence. Examples: DNS NXDOMAIN or NODATA with a valid
  (possibly DNSSEC-signed) authoritative answer where DNS is the
  Trust Method's sole or final publication channel; for
  `openid_federation` ({{FEDERATION}}), HTTP 404 for the leaf's
  Entity Configuration.
  A Trust Method with several publication channels for the same
  evidence source reaches Negative only when the channels its lookup
  procedure consults authoritatively report no authorization. A
  Trust Method specification MAY additionally define an explicitly
  published denial. For a `subject_namespace_authorization` method,
  such a denial MUST map to an Affirmative retrieval that authorizes
  nothing, so that it is final ({{multiple-sources}}). For other
  methods, the specification's state mapping decides whether it maps
  to Negative or to such an Affirmative retrieval. A Negative state
  records that no authorization is in effect through that channel and
  is final for that Trust Method;
  {{multiple-sources}} states its effect on other methods.

- **Indeterminate**: the lookup did not produce an authoritative
  Affirmative or Negative result, for example on DNS SERVFAIL, a
  resolver timeout, a network partition, HTTPS 5xx, a TLS handshake
  failure, a malformed publication-channel response, or a structural
  validation failure on retrieved evidence (invalid signature,
  unparseable payload, validity bounds of the evidence as a whole
  outside the acceptable window). The Trust Method evaluates bounds
  on individual entries inside the evidence. An Indeterminate state
  carries no information about the intent of the party whose evidence
  the Trust Method consults.

#### Fail-Closed Requirements {#fail-closed-requirements}

A Resource Authorization Server MUST classify every lookup outcome
into exactly one of the three states above before producing an
accept-or-reject decision. It MUST fail closed on both Negative and
Indeterminate states: neither state satisfies the Trust Method that
produced it. {{rasp}} step 5b and {{multiple-sources}} then decide
the category. Trust Method specifications MUST NOT permit any
condition under which an Indeterminate state is treated as
Affirmative.

A Resource Authorization Server MAY rely on cached Affirmative
evidence during a transient Indeterminate state on the live
publication channel, but only within the stale-serving bound the
Trust Method specification sets. Repeated Indeterminate states across
consecutive lookups MUST NOT extend the effective cache lifetime
beyond that specification's stated maximum. Once the cache expires,
a live channel that remains Indeterminate does not satisfy the Trust
Method. Treating Indeterminate as Affirmative, extending a cache
across Indeterminate states, and falling through to another evidence
source within the evaluation of a method with deterministic source
selection ({{multiple-sources}}) are the same downgrade:
each lets an adversary who controls the availability of the
publication channel turn a hard denial into a soft one.

Trust Method specifications MAY define additional lookup states (for
example, a "stale-but-signed" state for transparency-log-bound
evidence) but MUST place any such state on the Affirmative-or-reject
side of the decision boundary. Trust Method specifications MUST NOT
introduce a state that relaxes the Indeterminate-rejects-hard
requirement.

## Outbound Fetches {#outbound-fetches}

Evaluating Trust Methods makes the Resource Authorization Server
fetch resources at locations an attacker can choose:

- the Assertion Issuer's metadata and key sets;
- Entity Statements reached through a leaf's `authority_hints`;
- a Trust Mark Issuer's chain; and
- policy documents named by a Subject Authority that the attacker
  controls.

For every such fetch, the Resource Authorization Server:

- MUST NOT connect to an address that is private-use, loopback,
  link-local, unique-local, or otherwise not globally routable,
  unless local configuration explicitly permits that address range,
  for example for a deployment whose internal resolver maps public
  names to private addresses;
- MUST connect to the address it checked rather than resolving the
  name again, so that a DNS answer that changes between the check
  and the connection cannot redirect the fetch;
- MUST apply a timeout to each fetch and MUST bound response sizes;
  and
- MUST bound the number of fetches one token request can cause; a bound
  of no more than 20 is RECOMMENDED. A lookup that would exceed the
  bound is Indeterminate.

Trust Method specifications MAY add requirements; for example,
{{DAI}} forbids following redirects for policy fetches.

# Grant Profile and Token Bindings {#bindings}

This section defines the binding for ID-JAG ({{id-jag-profile}}),
the only grant profile to which this document applies.
Other assertion-bearing grant profiles would supply analogous
bindings, naming their grant profile identifier, their Subject
Identifier-bearing claim, and any profile-specific JWT claims that
participate in Trust Method evaluation. A binding under which a
`subject_namespace_authorization` method can apply MUST name a Subject
Identifier-bearing claim from which a Subject Authority is
determinable.

An ID-JAG is presented with a `grant_type` value from {{ID-JAG}}
§4.4, and its JWT `typ` header is `oauth-id-jag+jwt` ({{ID-JAG}}
§4.4.1). This document
defines no binding for any other identity assertion, so {{rasp}}
step 3 rejects one unless another specification defines a binding
for its grant profile and the Trust Policy lists that profile.

## ID-JAG {#id-jag-profile}

For ID-JAG {{ID-JAG}}, `authorization_grant_profiles_supported`
contains `urn:ietf:params:oauth:grant-profile:id-jag`. The Subject
Identifier for ID-JAG is the top-level `email` claim, accompanied by
`email_verified=true`, extracted per
{{subject-authority-determination}}. {{ID-JAG}} §3.1 does not define
`email_verified`, so an Assertion Issuer whose ID-JAGs a namespace
method evaluates sends it, a narrowing of {{ID-JAG}}. When the Trust
Policy contains
a `subject_namespace_authorization` method, {{rasp}} step 5c
requires the ID-JAG to carry that Subject Identifier and requires
rejection if it is absent or unresolvable.

Passing step 5 of {{rasp}} is how the Resource Authorization Server
decides that an ID-JAG's `iss` identifies an IdP Authorization Server
it trusts ({{ID-JAG}} §4.4.1). It does not decide the trust in `iss`
and `tenant` together that the same section requires of a
multi-tenant issuer for the recipient tenant, which `aud`,
`aud_tenant`, or the server's trust relationship with the issuer
identifies. That trust remains the Resource Authorization Server's
own configuration.

A namespace method can bind authorization to a tenant using the
`tenant` claim that {{ID-JAG}} §6.1 requires of a multi-tenant
issuer. {{DAI}} §Shared Issuer Variant: Multi-Tenant Identity
Provider gives an example. A match on `tenant` establishes that the
Subject Authority authorized that (`iss`, `tenant`) pair for its
namespace, not the recipient-tenant trust above.

In addition to the processing in {{rasp}}, the Resource Authorization
Server MUST:

1. Verify that the `email` Subject Identifier, when the assertion
   carries one, uses a format listed in
   `subject_identifier_formats_supported`, if that Trust Policy member
   is present.

2. Use the `email` claim only as {{rasp}} step 5c permits and for
   Subject Authority evaluation. The Resource Authorization Server
   MUST NOT
   treat the mere presence or value of the email claim as proof
   that the ID-JAG issuer is authoritative for the subject. Issuer
   acceptability is established only by evaluating the Trust Methods.

# Security Considerations

## Alignment with OAuth Security BCP {#rfc9700-alignment}

This document complements the OAuth 2.0 Security Best Current
Practice ({{RFC9700}}) and does not override it.

The `subject_namespace_authorization` category is the wire-format
analog of {{RFC9700}} §4.4.2 (mix-up countermeasures): a Resource
Authorization Server does not accept an assertion from an
authorization server that the Subject Authority has not listed. The
guarantee is only as strong as the integrity of the Subject
Authority's publication channel. An attacker who controls that
channel, such as unauthenticated DNS or a compromisable HTTPS
origin, can defeat it. See the per-Trust-Method analysis (for
example, {{DAI}}) and {{multiple-sources}}.

Other {{RFC9700}} topics (redirect URI validation, bearer-token
replay, client authentication) apply at the grant-profile and
client-authentication layers and are not addressed here.
Implementations SHOULD follow {{RFC9700}} at those layers.

## Unverified Claim Exploitation {#unverified-claim}

Source selection ({{multiple-sources}}) runs before the identity
assertion's claims are validated against any Trust Method's
evidence, so it acts on an untrusted payload. A Trust Method whose
selection considers multiple claims, applies preference ordering, or
falls back from one input to another gives an attacker a selection
lever. The attacker constructs an identity assertion whose claims
satisfy several bindings, so that selection picks the weakest or
most attacker-favorable evidence source.

For example, an identity assertion carries
`sub: alice@victim.example` and
`tenant_context: attacker-tenant.example`, and a Trust Method selects
the evidence source by tenant context when present, falling back to
subject domain. The Resource Authorization Server validates the
identity assertion against the attacker-controlled Issuer
Authorization Policy of attacker-tenant.example and accepts a claim
about a victim.example subject.

Deterministic source selection ({{multiple-sources}}) removes the
lever. The `email` extraction registered by this document derives the
Subject Authority deterministically from a single claim and does
not fall back; specifications registering additional extractions
MUST preserve this property.


## Applicability Bypass {#applicability-bypass}

The combination rule ({{combination-rule}}) requires the Resource
Authorization Server to evaluate every applicable category. If a
Trust Method specification or local configuration lets
applicability depend on the incoming identity assertion, an attacker
can waive a category by constructing a matching identity assertion.
For example, if namespace authorization does not apply to an
assertion from a legacy issuer, or to one carrying a legacy-flag
claim, the attacker sends such an assertion, and the Resource
Authorization Server relies on issuer authentication alone.
{{category-applicability}} therefore makes applicability a property
of the published Trust Policy. Traffic that legitimately predates a
category's availability is handled at the policy layer, not by
waiving the category. Policy per deployment phase is a configuration
boundary, while a waiver driven by the identity assertion is a
boundary the attacker controls.


## Evidence Source Compromise {#authority-source-compromise}

The publication channel is the authority binding and the
highest-value target. An attacker who compromises it (DNS hijack,
TLS misissuance plus DNS redirect, registrar account takeover,
compromise of the federation operator's signing key) speaks as the
party whose evidence the Trust Method consults, and the evidence it
publishes is indistinguishable from legitimate evidence.

This framework provides no cryptographic recovery from
publication-channel compromise. Recovery is operational. Trust
Method specifications SHOULD recommend operational defenses
appropriate to their
publication channel (DNSSEC, registry-lock, CAA records,
Certificate Transparency monitoring, federation key rotation).
{{DAI}} §Security Considerations gives the DNS+HTTPS
publication-channel compromise model.

{{FEDERATION}} covers trust anchor key rollover.

## Transitive Authorization is Bounded {#transitive-authz-bounded}

`issuer_authentication` permits chains: under `openid_federation`
({{FEDERATION}}), a trust anchor's signatures on intermediate
Subordinate Statements transitively authenticate the leaf.
`subject_namespace_authorization` requires depth one. A Subject
Authority lists Assertion Issuers directly and cannot delegate to
whoever an issuer federates, so no intermediate party can add an
issuer that the Subject Authority did not list, and revocation
latency does not compound across delegations. Parties that bind keys
to listed issuers, or host the list, remain trusted ({{key-binding}}).

Federation membership does not establish namespace authority, and
namespace authorization does not establish authenticity. The
combination rule ({{combination-rule}}) enforces both at evaluation
time.


## Key Binding Is Outside Namespace Authorization {#key-binding}

A `subject_namespace_authorization` method authorizes an issuer
identifier, not a key. Which key speaks for that identifier is
decided elsewhere. Every party that can make that decision is
trusted for every namespace that lists the issuer:

- Under `openid_federation`, an intermediate or trust anchor above
  the issuer can supply its key source ({{FEDERATION}}).
- Without federation, the key comes from the Assertion Issuer's own
  metadata unless locally configured ({{rasp}} step 2), and whoever
  controls that metadata controls the key.
- A policy host named by a {{DAI}} `uri=` pointer controls the list
  itself, unless the Subject Authority pins the policy's signing key.

The combination rule does not defend against these parties:
the first two act as a listed issuer, and the third decides the
list. A future extension could let a Subject Authority pin a key
thumbprint for each authorized issuer.

## Scope of Namespace Authorization {#scope-of-namespace-authorization}

Trust-policy evaluation establishes that the Assertion Issuer is
authorized to assert this class of Subject Identifiers under the
namespace. Assertion validation under the applicable grant profile
({{rasp}} step 2), including any replay and client-binding checks
that profile defines, still applies. Resource Authorization Servers
MUST NOT infer any of the following from a successful trust-policy
evaluation:

- The named subject exists at the Assertion Issuer or controls
  the email local-part ({{DAI}} §Email Local-Part Is Not Authenticated).
- The subject's current employment, enrollment, or organizational
  status.
- The strength, freshness, or method of the Assertion Issuer's
  authentication.
- Account-linking semantics (case sensitivity, plus-address or
  alias handling) compatible with this Resource Authorization
  Server's.
- Suitability for any specific risk class, scope sensitivity, or
  compliance regime.

A deployment that needs these properties obtains them outside this
framework, such as through authentication-method or authentication
assurance level claims, fresh-authentication signals, account-status
attestations, or out-of-band verification. `email_verified=true` is
a prerequisite for deriving namespace authority from the email's
domain, not evidence of current mailbox control.

Namespace authorization does not make an email address a safe
account key by itself. A Resource Authorization Server keys accounts
on issuer-scoped identifiers ({{rasp}} step 5c). It links an account by email only under
its own account-linking rules.

Nor does namespace authorization cover other subject claims. An
issuer authorized for `attacker.example` can send an `email` in that
namespace with an `aud_sub` or `sub_id` that names an account
elsewhere. Step 5c of {{rasp}} therefore limits subject resolution,
under every Trust Policy, to the evaluated Subject Identifier and
issuer-scoped identifiers.

Nor is the authorization audience-scoped. The
`subject_namespace_authorization` category constrains which
Assertion Issuers can assert about a namespace, not which Resource
Authorization Servers they target. A compromised but authorized
issuer can therefore mint assertions about the namespace's subjects
for any consumer. Audience binding comes from the assertion's `aud`
claim and the applicable grant profile, not from the Subject
Authority. A future extension could let Subject Authorities
constrain acceptable audiences ({{DAI}} §Future Extensions).

## Per-Assertion Revocation Is Out of Scope {#per-assertion-revocation}

This framework decides whether an Assertion Issuer is authorized;
it does not revoke individual assertions. An assertion remains valid
until `exp` even after session termination, credential revocation at
the Assertion Issuer, or the Subject Authority's withdrawal of the
issuer's authorization. Withdrawal stops new acceptances only after a
window that depends on the Trust Method, for planned transfers and
security incidents alike, and that can exceed the Resource
Authorization Server's own cache lifetime. Under {{DAI}}, the window
also includes propagation, upstream DNS and HTTP caches, and stale
use ({{DAI}} §Caching). Withdrawal does not revoke access or refresh
tokens already issued, and the framework cannot invalidate remote
caches. Subject Authorities that need fast revocation use short cache
lifetimes. Synchronous revocation needs
OAuth 2.0 Token Revocation {{RFC7009}}, Token Introspection
{{RFC7662}}, or short assertion lifetimes at the grant-profile layer.


## Policy Document Integrity {#integrity}

The Trust Policy MUST be served over HTTPS with TLS server
authentication, and its integrity rests on TLS. It carries no
signature. Mirrored or cached copies of a Trust Policy MUST NOT be
relied on beyond their HTTP cache lifetime ({{caching}}). A Resource
Authorization Server enforces its Trust Policy from its own
configuration, and a client uses the published copy only for
discovery ({{client-processing}}). A substituted Trust Policy can
therefore mislead a client's choice of Assertion Issuer but not
change what the Resource Authorization Server accepts.

An Issuer Authorization Policy decides acceptance. The Trust Method
specification that defines it supplies its integrity protection, such
as {{DAI}} §Signed Policy, and its cache and stale-use bounds
({{trust-method-spec-requirements}}).

## Shared Infrastructure and Hosted Well-Known Paths {#shared-infrastructure}

Many deployments host `/.well-known/` resources on third-party
Content Delivery Networks, shared edge platforms, or multi-tenant
cloud hosting. TLS server authentication proves that the client
reached an endpoint serving the requested host name, not that the
namespace owner controls every routing rule, origin-pull rule, cache
rule, or tenant boundary inside the shared platform.

The risk is greatest for the Subject Authority's Issuer
Authorization Policy, which decides acceptance. An attacker who can
make shared infrastructure serve a forged policy for a victim domain
can try to authorize an Assertion Issuer for the victim's namespace
even though the HTTPS connection succeeds. Similar risks arise from:

- misconfigured path routing;
- dangling origins;
- tenant takeover;
- cache poisoning; and
- origin authentication failures.

Administrators SHOULD avoid delegating security-critical well-known
paths to multi-tenant infrastructure unless they can ensure exclusive
control over routing for those paths, authenticated origin access,
cache invalidation, and tenant isolation. The Trust Method
specification defines object-level integrity for an Issuer
Authorization Policy hosted this way ({{DAI}} §Policy Hosts).

## Downgrade Attacks {#downgrade}

Or-semantics apply within a category, and a client cannot observe
per-subject local requirements. A client can therefore choose a
weaker listed Trust Method. That choice is not a protocol violation
the client can be held to but a condition the Resource Authorization
Server rejects. A Resource Authorization Server that supports
multiple Trust Methods SHOULD define local precedence rules.
Resource Authorization Servers MUST enforce differentiated
requirements at token request time rather than relying on the policy
document alone. The combination rule in {{rasp}} prevents downgrade
across categories, such as satisfying only an
`issuer_authentication` method when a
`subject_namespace_authorization` method also applies.

## Trust Policy Caching {#caching}

Consumers cache the Trust Policy using HTTP caching ({{RFC9111}}),
bounded by a local maximum cache lifetime. Independent of any
policy-document cache expiration, revocation status of validated
trust evidence MUST be checked at the cadence required by the
applicable Trust Method specification. Transport integrity is
addressed in {{integrity}}.

## Observability

Trust-policy evaluation is a security-critical decision. For each
processed assertion, deployments are encouraged to log at minimum:

- the Assertion Issuer identifier;
- the version or configuration identifier of the Trust Policy in
  effect;
- the Trust Methods that succeeded;
- the matched trust anchor or Issuer Authorization Policy origin;
- the Subject Identifier format; and
- the accept/reject outcome.

Deployments are also encouraged to support correlation across the
issuance and verification halves of an identity-chain transaction.

# Privacy Considerations

Trust policies are typically unauthenticated HTTPS resources. An
adversary can scrape them across many Resource Authorization Servers
to learn:

- which RASes participate in which federations;
- which trust anchors are accepted; and
- which Subject Identifier formats are honored.

This aids targeted attacks, such as prioritizing compromise of a
heavily relied-upon trust anchor. Operators SHOULD publish only what
clients need to determine whether they can attempt issuance, and
prefer trust-anchor or federation expression over enumerating
individual issuers.

Subject Authority lookup at the RAS reveals the queried Subject
Authority to DNS resolvers and, for HTTPS retrieval, to the policy
host. RASes verifying with `domain_authorized_issuer` should treat
the Subject Authority and Assertion Issuer relationship as
sensitive operational data and avoid sending full subject
identifiers in policy URLs, query parameters, logs, or telemetry.

# Internationalization Considerations

The `email` format can carry an internationalized domain. Consumers
convert the domain to A-label form by UTS #46 nontransitional
processing before Public Suffix List matching, and compare A-labels
using case-insensitive ASCII comparison
({{subject-authority-determination}}).

Because comparison uses the A-label form, Unicode domains that map
to different A-labels are different Subject Authorities, however
similar they look. A-label comparison does not by itself defend
against homograph confusion shown to a human at account-linking or
display time, which is out of scope and left to the consuming
application. A Resource Authorization Server that links accounts by
email compares the full domain in this A-label form ({{rasp}} step
5c), so that a Unicode string that maps to a different Subject
Authority cannot land on another user's account.

The `email` extraction uses the simple single-`@` rule, not the full
{{RFC5321}} Mailbox grammar. Internationalized email addresses
(SMTPUTF8, {{RFC6530}}) whose local-part requires UTF-8 are outside
its scope. The domain of such an address is handled normally once
isolated. A future Subject Identifier format could define richer
address handling by registering its own extraction procedure
({{iana-authority-registry}}).

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

### Identity Assertion Trust Method Categories Registry {#iana-trust-method-categories-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Trust Method Categories" under the "OAuth Parameters"
registry group. This registry backs the category values used by the
Trust Methods registry.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains a Category Name (character set `[a-z0-9_]`),
a Description of the trust question the category answers, a Change
Controller, and a Specification Document.

Designated Expert instructions: the expert verifies that the proposed
category answers a trust question distinct from existing categories,
so that the cross-category AND semantics of {{combination-rule}}
remain meaningful, and that the description makes clear what
evidence satisfies it. Categories that merely rename or subdivide an
existing category SHOULD be rejected.

Initial entries, each with Change Controller IETF and Specification
Document this document:

| Category Name | Description |
|-|-|
| `issuer_authentication` | Establishes that the Assertion Issuer is an authentic, recognized entity |
| `subject_namespace_authorization` | Establishes that the Assertion Issuer is authorized by the subject's namespace owner |

### Identity Assertion Trust Methods Registry {#iana-trust-methods-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Trust Methods" under the "OAuth Parameters"
registry group.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains:

Identifier:
: Short string used as the `method` value in a Trust Method object.
Identifiers MUST use the character set `[a-z0-9_]` and SHOULD
describe the trust evaluation procedure.

Categories:
: One or more values registered in the Trust Method Categories
registry ({{iana-trust-method-categories-registry}}).

Parameters:
: Additional JSON members defined for this Trust Method, with their
JSON types and whether each is REQUIRED or OPTIONAL.

Change Controller:
: The party responsible for change control.

Reference:
: A reference to the specification defining the Trust Method.

Designated Expert instructions: the expert verifies that:

- the identifier is unique and descriptive;
- each listed category is registered;
- the Trust Method's evidence and evaluation procedure are specified
  precisely enough for interoperable implementation (including its
  mapping onto the Affirmative/Negative/Indeterminate lookup states
  and any cache-lifetime bounds, per
  {{trust-method-spec-requirements}}); and
- parameter names do not collide in meaning with parameters of other
  methods in a way that would be ambiguous when methods are combined.

This document registers no initial entries. {{DAI}} and
{{FEDERATION}} register the first Trust Methods.

### Trust Policy Members Registry {#iana-trust-policy-members-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Issuer Trust Policy Members" under the "OAuth Parameters"
registry group.

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains:

Member Name:
: JSON member name used in the Trust Policy document.

Member Description:
: A short description of the member's semantics.

Decision-Affecting:
: Whether the member is decision-affecting ({{terminology}}).

Change Controller:
: The party responsible for change control.

Specification Document:
: A reference to the specification defining the member.

Designated Expert instructions: the expert verifies that:

- the member name does not collide with an existing member;
- the member's semantics and JSON type are specified;
- the registration states whether the member is decision-affecting
  ({{terminology}}); and
- any decision-affecting member states how a consumer that does not
  recognize it behaves.

Unrecognized members are ignored by default. A member requiring
fail-closed handling needs the criticality mechanism of
{{critical-members}}.

Initial entries, each decision-affecting, with Change Controller IETF
and Specification Document this document:

| Member Name | Member Description |
|-|-|
| `resource_authorization_server` | Resource Authorization Server issuer identifier |
| `authorization_grant_profiles_supported` | Supported identity assertion grant profile identifiers |
| `subject_identifier_formats_supported` | Supported Subject Identifier formats |
| `issuer_trust_methods` | Trust Method requirements enforced for incoming identity assertions |
| `crit` | Names decision-affecting members that a consumer has to understand to process the document |

## Subject Authority Extraction Procedures Registry {#iana-authority-registry}

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

Change Controller:
: The party responsible for change control.

Designated Expert instructions: the expert verifies that:

- the format has a well-defined namespace authority;
- the extraction procedure is deterministic (two consumers compute
  the same Subject Authority from the same Subject Identifier); and
- the procedure specifies exact-match comparison semantics per
  {{subject-authority-determination}}.

Specifications that define a Subject Identifier format with a
well-defined namespace authority are expected to register it here.

Initial entries:

| Subject Identifier Format | Subject Authority Form | Extraction Procedure | Change Controller |
|-|-|-|-|
| `email` | DNS domain | {{subject-authority-determination}} of this document | IETF |

--- back

# Agent Platform Walkthrough {#example-agent-platform}

This appendix is non-normative. It shows how a customer's Issuer
Authorization Policy, listing the agent platforms it authorizes,
keeps an unauthorized provider from impersonating users in the
customer's email namespace.

**Cast:** customer `example.com` (owns the email domain); agent
platform `https://agentprovider.example` (mints ID-JAGs after
federated SSO from the customer's primary Identity Provider); tool
provider `https://toolprovider.example` (the Resource Authorization
Server); end user `alice@example.com`. The agent platform gives each
customer its own issuer identifier, here
`https://agentprovider.example/example-com`. A platform using one
issuer identifier for every customer would send `tenant` in every
ID-JAG ({{id-jag-profile}}), and the customer would list it with
that `tenant` value in the DNS pointer form ({{DAI}}
§Single-Issuer Multi-Tenant Identity Providers).

**Publication.** The customer publishes:

~~~
_oauth-issuer-policy.example.com. IN TXT (
    "v=oauth-issuer-policy1;"
    "authority=example.com;"
    "issuer=https://agentprovider.example/example-com" )
~~~

The tool provider's Trust Policy lists `domain_authorized_issuer`.
After Alice signs in to the agent platform (out of scope here), the
agent platform mints an ID-JAG:

~~~ json
{
  "iss": "https://agentprovider.example/example-com",
  "aud": "https://toolprovider.example",
  "exp": 1780166400, "iat": 1780166100, "jti": "b9c1...",
  "sub": "user-3f81a2", "client_id": "f53f191f9311af35",
  "email": "alice@example.com", "email_verified": true
}
~~~

**Verification.** The tool provider:

- authenticates the client (`private_key_jwt`);
- validates the ID-JAG per {{ID-JAG}};
- extracts the Subject Authority `example.com` from the email claim;
- queries `_oauth-issuer-policy.example.com` and confirms that the
  `iss` value matches an authorized issuer; and
- issues a token.

**What this protects against.** Suppose `attacker.example` mints
its own assertion claiming `email: alice@example.com,
email_verified: true`, with a signature that validates against its
own JWKS and the correct audience. The tool provider extracts
Subject Authority `example.com`, looks up the customer's policy, and
finds that `https://attacker.example` is not in
`authorized_issuers`. The Trust Method fails, and the tool provider
rejects with `invalid_grant`. The self-asserted `email_verified`
has no force: unless the customer lists `https://attacker.example`,
that issuer cannot impersonate users in `example.com`.

# Document History

This appendix is non-normative and will be removed before publication.
A number in parentheses is a closed issue in the
[issue tracker](https://github.com/mcguinness/draft-mcguinness-oauth-id-assertion-framework/issues)
for this document, which holds the details of that change.

-01

  * Move the `openid_federation` Trust Method, including its OpenID
    Federation 1.1 alignment (#20), and client recognition (#32) to
    {{FEDERATION}} (#38). A policy written against an earlier
    editor's copy that uses `client_trust_methods` needs the member
    added to `crit` before a server that implements only this document
    consumes it.
  * Remove `signed_policy` and the `trust-policy+jwt` media type from
    the Trust Policy; signed-policy processing moves to {{DAI}} (#21).
  * Forbid waivers that a Subject Authority publishes, such as a
    monitoring mode, and define how methods in one category combine
    (#22).
  * Rely on {{ID-JAG}} §6.1, which requires a multi-tenant Assertion
    Issuer to send `tenant` in every ID-JAG (#26).
  * Return `invalid_grant` for a rejection in steps 1 through 5 of
    Resource Authorization Server processing, and recommend client
    authentication before Trust Method lookups (#28).
  * Add outbound-fetch requirements, UTS #46 processing for
    internationalized domains, and a key-binding security
    consideration (#27).
  * Expand the motivation, and add tables and explanations for
    readers (#29).
  * Limit subject resolution to the evaluated identifier, bound reuse
    of federation evidence to 24 hours, and evaluate client
    recognition before the assertion (#34).
  * State the trust decision once, in OAuth terms, as part of
    processing; open with one example exchange (#38).
  * Apply the framework to ID-JAG only, removing the generic
    JWT-bearer binding; keep one worked example; move the Future
    Extensions and FAQ appendices to the issue tracker; trim the
    Introduction and Security Considerations (#38).
  * Require that an assertion's verification key be bound to `iss`
    independently of the assertion (#38).
  * Editorial changes for readability.

-00

  * initial draft
