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
  RFC8461:
  RFC8555:
  RFC8659:
  TXN-TOKENS:
    title: "Transaction Tokens"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/
    date: false
  WICG-EMAIL-VERIF:
    title: "Email Verification Protocol"
    target: https://wicg.github.io/email-verification-protocol/
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/

---

--- abstract

This document defines a trust policy that an OAuth authorization
server publishes to describe the conditions under which it accepts
identity assertion JWT authorization grants. The policy expresses trust
criteria, including trust methods, trust anchors, subject identifier
formats, and grant profiles, that an Assertion Issuer has to satisfy.
This enables scalable issuer discovery for deployments backed by
OpenID Federation or Domain-Authorized Issuer Discovery.

This document also defines Domain-Authorized Issuer
Discovery, a DNS and HTTPS mechanism by which the owner of a subject
namespace (for example, an email domain) authorizes one or more Assertion
Issuers and, through the same records, enables clients to discover the
Assertion Issuer for an identifier in that namespace.

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
conditions under which an Assertion Issuer is acceptable - for
example, whether the Assertion Issuer has authority for the subject
namespace being asserted, or whether the client presenting the
assertion is bound to its key.

This document defines an Identity Assertion Issuer Trust Policy
that a Resource Authorization Server publishes to describe its trust
criteria. The Resource Authorization Server does not say "these are
all the Assertion Issuers I trust"; it says "these are the conditions
an Assertion Issuer must satisfy." Conditions are evaluated by validating concrete evidence - a
federation trust chain or a domain-authorized issuer record - when
an assertion is presented.

This document also defines Domain-Authorized Issuer
Discovery, a self-contained mechanism for one of those conditions, by
which a subject namespace owner authorizes specific Assertion Issuers
via DNS or HTTPS. Because both verifiers and clients consume the same
records, the mechanism doubles as an Assertion Issuer discovery
facility for the namespace.

This document therefore defines two JSON documents that should not be
confused:

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

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

Identity Assertion JWT Authorization Grant:
: An OAuth JWT authorization grant or grant profile in which the JWT
carries identity information used by the Resource Authorization Server
for subject resolution, account linking, authorization, or delegated
access.

Assertion Issuer:
: The authorization server, identity provider, or other service that
issues an identity-bearing assertion or token consumed under this
framework, identified by the JWT `iss` claim. The specific local
term varies by context: an authorization server in OAuth grant
flows, an authorization server in JWT access token verification
({{related-token-contexts}}), and an "issuer" in the Email
Verification Protocol ({{WICG-EMAIL-VERIF}}). For trust evaluation
purposes, all are Assertion Issuers and the JWT `iss` value serves
as the unifying identifier. The same string serves as the
issuer's identifier across the discovery and federation mechanisms
referenced by this document: the `issuer` value in OAuth
Authorization Server Metadata {{RFC8414}}, the `issuer` value in
OpenID Connect Discovery {{OIDC-DISCOVERY}}, and the federation entity
identifier in OpenID Federation {{OIDF-FEDERATION}}. Issuer
identifiers are compared using the case-sensitive URL string
comparison rules of {{RFC8414}} Section 2.

Resource Authorization Server:
: The authorization server that receives the identity assertion JWT and
evaluates it as part of an OAuth grant.

Trust Method:
: A named procedure by which an Assertion Issuer can establish that it
is acceptable to the Resource Authorization Server. Trust Method
identifiers are registered in the registry defined in
{{iana-trust-methods-registry}}.

Trust Anchor:
: An entity, key, federation authority, registry, or metadata source
  that the Resource Authorization Server uses to evaluate a Trust
  Method.

Subject Identifier:
: A claim or claim value identifying the asserted subject. This
document uses Subject Identifier format names from the registry
established by {{RFC9493}} as labels in `subject_identifier_formats_supported`,
even when the subject identifier itself is carried as a top-level JWT
claim (for example, the `email` format identifies the top-level `email`
and `email_verified` claims used by {{ID-JAG}}) rather than as an
{{RFC9493}} Subject Identifier JSON object. Grant profiles and other
specifications define which carriage applies in their context.

Subject Authority:
: An authority for a Subject Identifier namespace, such as a DNS domain
for an `email` Subject Identifier.

Issuer Authorization Policy:
: The JSON document published by a Subject Authority that authorizes one
or more Assertion Issuers in the Subject Authority's namespace. Defined
in {{dii-document}}.

Assertion Issuer Discovery:
: The process by which a client determines which Assertion Issuer or
  authorization server is authoritative for a subject identifier's
  namespace.

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
  "grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer",
      "dns_discovery": true
    }
  ]
}
~~~

### Members

`resource_authorization_server`
: REQUIRED. The issuer identifier of the Resource Authorization Server.
MUST exactly match the authorization server metadata `issuer` value
{{RFC8414}}.

`grant_profiles_supported`
: REQUIRED. JSON array of identity assertion grant profile identifiers
supported by this policy. Each value MUST match a profile identifier in
the authorization server metadata
`authorization_grant_profiles_supported` parameter, where present.

`subject_identifier_formats_supported`
: OPTIONAL. JSON array of Subject Identifier format names that the
Resource Authorization Server accepts. Values are defined by
{{RFC9493}}, by grant profile specifications, or by other
specifications. This member applies to all Trust Methods; an
individual Trust Method MAY constrain the formats it evaluates (for
example, `domain_authorized_issuer` evaluates only formats registered
in {{iana-authority-registry}}).

`issuer_trust_methods_supported`
: REQUIRED. Non-empty JSON array of Trust Method objects (see
{{trust-methods}}). An Assertion Issuer is acceptable when it satisfies
the Trust Method combination rule in {{rasp}}. Local policy MAY impose
additional requirements for specific clients, subjects, or scopes; see
{{downgrade}}.

## Trust Methods {#trust-methods}

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
usable issuer trust method. This
document defines three Trust Methods, presented below grouped by
category. The `issuer_authentication` method is
{{trust-method-openid-federation}}; the
`subject_namespace_authorization` methods are
{{trust-method-domain-authorized-issuer}} and
{{trust-method-email-verification-dns}}.

### Trust Method Categories {#trust-method-categories}

Trust Methods answer two distinct security questions about an Assertion
Issuer:

`issuer_authentication`
: Does the entity identified by the JWT `iss` claim authentically
belong to a recognized ecosystem? Federation membership and trust-mark
issuance answer this question.

`subject_namespace_authorization`
: Is this Assertion Issuer entitled to assert about subjects in the
namespace named by the assertion's Subject Identifier? An attestation
from the namespace owner answers this question.

Each Trust Method identifier is registered in
{{iana-trust-methods-registry}} with one or more category values. A
trust policy MAY list methods from one or both categories. The
combination rule in {{rasp}} requires the Assertion Issuer to satisfy
at least one Trust Method from EACH category that is both present in
the policy and applicable to the assertion. The
`issuer_authentication` category applies unconditionally. The
`subject_namespace_authorization` category applies when the assertion
carries a Subject Identifier; if the Subject Authority cannot be
determined, the assertion fails closed as described in {{rasp}}.
Within a single category, OR-semantics apply: satisfying any one listed
method in that category is sufficient.

This separation prevents an Assertion Issuer that is authenticated by
federation or trust-mark membership from being treated as
automatically authorized to assert about subjects in a namespace the
Assertion Issuer has no authority over. Deployments that accept
identity assertions about namespace-bound subjects (for example,
email-domain users) SHOULD list at least one
`subject_namespace_authorization` method in their trust policy in
addition to any `issuer_authentication` methods.

The set of categories required for an assertion to be accepted is
publisher-driven: it is exactly the set of categories represented by
recognized Trust Method objects listed in
`issuer_trust_methods_supported`. Future specifications MAY register
additional categories, but adding a category to the registry does not
impose new requirements on existing trust policies that do not list
methods from that category.

### openid_federation {#trust-method-openid-federation}

Category: `issuer_authentication`.

The `openid_federation` method indicates that the Assertion Issuer is
acceptable if it can establish a valid OpenID Federation
{{OIDF-FEDERATION}} trust chain to one of the listed trust anchors.

~~~ json
{
  "method": "openid_federation",
  "trust_anchors": ["https://federation.example.org"]
}
~~~

`trust_anchors`
: REQUIRED. Non-empty JSON array of OpenID Federation trust anchor
entity identifiers.

The Resource Authorization Server MUST validate the Assertion Issuer's
federation trust chain according to {{OIDF-FEDERATION}}. The terminal
trust anchor of the validated chain MUST exactly match one of the
listed `trust_anchors` values using the entity identifier comparison
rules of OpenID Federation. The Resource Authorization Server MUST
verify that the Assertion Issuer's federation entity configuration
declares an entity type appropriate for an OAuth authorization server
or OpenID provider.

### domain_authorized_issuer {#trust-method-domain-authorized-issuer}

Category: `subject_namespace_authorization`.

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
Authority's host. The full mechanism — record syntax, lookup
procedure, failure handling, and security considerations - is
specified in {{dii}}.

This Trust Method supports path-bearing issuer identifiers (for
example,
`https://login.microsoftonline.com/{tenant-id}/v2.0`) directly: the
`authorized_issuers[].issuer` field accepts any absolute HTTPS URL
issuer identifier, and case-sensitive comparison against the JWT
`iss` claim distinguishes tenants under the same host. Single-issuer
multi-tenant Identity Providers that carry the tenant in a top-level
JWT claim (rather than in the issuer URL) are also supported, with
the caveats described in {{dii-multi-tenant}}.

~~~ json
{
  "method": "domain_authorized_issuer",
  "dns_discovery": true
}
~~~

`dns_discovery`
: OPTIONAL. Boolean. If `true`, the Resource Authorization Server
consults DNS-based discovery (see {{dii-dns-record}}), accepting both
inline and pointer forms. Defaults to `false`, in which case only
the HTTPS well-known location (see {{dii-https-url}}) is used. This
flag governs verifier behavior only; Assertion Issuer discovery
clients ({{dii-hrd}}) always consult DNS regardless of this flag.

### email_verification_dns {#trust-method-email-verification-dns}

Category: `subject_namespace_authorization`.

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
   Apply the DNS response classification of {{dii-lookup}} (step 1).

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
   - non-default port, non-empty path component, query string, or
   fragment - does not match. Consequently, this Trust Method is
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
   `authorization_grant_profiles_supported`.

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

## Resource Authorization Server Processing {#rasp}

When evaluating an identity assertion JWT presented in a token request,
the Resource Authorization Server MUST:

1. Select and parse the trust policy that applies to the token request.
   If the policy document is malformed or no recognized Trust Method is
   usable, reject the assertion.

2. Validate the assertion according to the applicable grant profile,
   including `iss`, signature, `aud`, `exp`, `iat`, replay protection,
   and any client binding requirements defined by the profile.

3. Verify that the applicable grant profile is listed in
   `grant_profiles_supported`.

4. If `subject_identifier_formats_supported` is present, verify that
   the Subject Identifier format used by the assertion is listed,
   unless the applicable grant profile defines a different format
   selection rule.

5. Verify that the Assertion Issuer identified by `iss` satisfies the
   Trust Method combination rule:

   a. Partition the recognized Trust Method objects in
      `issuer_trust_methods_supported` by category (see
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
applicable grant profile.

# Grant Profile and Token Bindings {#bindings}

The trust policy structure and Trust Method machinery defined in the
preceding sections are profile-agnostic. This section provides
bindings to the grant profiles and token contexts in which this
document expects to be deployed.

The bindings — ID-JAG ({{id-jag-profile}}) and the OAuth Actor
Profile ({{actor-profile-binding}}) — are normative for deployments
using those features. {{related-token-contexts}} describes how the
same machinery applies informally to other token contexts
(Transaction Tokens, JWT access tokens).

## ID-JAG {#id-jag-profile}

This section provides the binding for ID-JAG {{ID-JAG}}; other grant
profiles would supply analogous bindings, naming their grant profile
identifier, their Subject Identifier-bearing claim, and any
profile-specific JWT claims that participate in Trust Method
evaluation.

For ID-JAG {{ID-JAG}}, `grant_profiles_supported` contains the value
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
   extraction procedure ({{iana-authority-registry}}). The format MAY
   be carried in `act.sub_profile` or be evident from the structure of
   `act.sub`; in either case the determination uses the registered
   extraction procedure unchanged.

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

# Domain-Authorized Issuer Discovery {#dii}

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

This section is structured as a self-contained module so that it can
be extracted into a separate specification in the future without
breaking changes to existing implementations. Section anchors use the
`dii-` prefix to make extraction mechanical.

## Relationship to Existing DNS Policy Mechanisms {#dii-prior-art}

This subsection is non-normative.

Domain-Authorized Issuer Discovery follows the same general
architecture as several existing IETF mechanisms in which control of
a DNS namespace is used to publish security policy:

- CAA {{RFC8659}} lets a DNS domain holder authorize which
  Certification Authorities are permitted to issue certificates for
  the domain. This document applies the same authorization pattern
  to Assertion Issuers.

- MTA-STS {{RFC8461}} uses DNS to signal policy state and HTTPS to
  host richer policy content. This document uses the same
  DNS-plus-HTTPS split while allowing an inline DNS form for simple
  delegations.

- ACME {{RFC8555}} uses DNS and HTTPS challenges to prove current
  control of a domain. This document is not a challenge-response
  protocol; it publishes a standing authorization policy that
  verifiers can evaluate when an assertion is presented.

- The Email Verification Protocol {{WICG-EMAIL-VERIF}} (WICG) lets
  an email domain delegate verification authority to a specific
  issuer via a DNS TXT record at `_email-verification.{email_domain}`
  plus issuer metadata at a well-known URL. The trust question it
  answers ("which issuer can verify emails for this domain?") is a
  special case of the question this document answers ("which
  Assertion Issuer is authorized for this subject namespace?"). The
  `email_verification_dns` Trust Method
  ({{trust-method-email-verification-dns}}) lets verifiers honor
  existing Email Verification Protocol records without requiring a
  parallel `_oauth-issuer-policy` record. Because the Email
  Verification Protocol is a non-IETF specification still under
  active development, interoperability for the
  `email_verification_dns` Trust Method depends on the stability of
  {{WICG-EMAIL-VERIF}}; implementations should track that
  specification for breaking changes.

These precedents motivate the use of an underscored DNS owner name,
a versioned TXT record, an HTTPS well-known URL, explicit
malformed-result handling, and cache-lifetime guidance.

## Subject Authority Determination {#dii-authority}

The Subject Authority associated with a Subject Identifier is
determined as registered in {{iana-authority-registry}}. Initial
extractions:

`email`
: The Subject Authority is derived from the assertion's top-level
  `email` claim, which MUST be accompanied by a top-level
  `email_verified` claim with the boolean value `true`. If the
  `email_verified` claim is absent or has any value other than
  `true`, the email extraction does not apply.

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

Subject identifier shapes not registered for this purpose MUST NOT
be evaluated under this Trust Method; the Resource Authorization
Server MUST reject the assertion with an OAuth `invalid_grant`
error.

## Issuer Authorization Policy Document {#dii-document}

The Issuer Authorization Policy is a JSON document served over HTTPS
with media type `application/json`. The document MUST be a JSON
object. It has the following members:

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

Unrecognized members MUST be ignored. Consumers MUST reject a policy
whose `subject_authority` member is absent or is not a string, or
whose `subject_authority` value does not match the computed Subject
Authority. Consumers MUST reject a policy whose `authorized_issuers`
member is absent, empty, not an array, or contains an element that
is not an object. Consumers MUST reject an authorized issuer object
that lacks a string `issuer` member or whose `issuer` member is not
a syntactically valid issuer identifier for the applicable
assertion grant profile. For OAuth authorization server issuer
identifiers, a non-HTTPS URL, a relative URL, or a URL with a
fragment component is malformed. Consumers MUST reject a
`subject_identifier_formats` value that is present but is not an
array of strings. Consumers MUST reject `valid_from`,
`valid_until`, and `last_updated` values that are present but are
not valid RFC 3339 date-times.

Example:

~~~ json
{
  "subject_authority": "example.com",
  "authorized_issuers": [
    {
      "issuer": "https://idp.example.com",
      "subject_identifier_formats": ["email"],
      "valid_until": "2027-01-01T00:00:00Z"
    }
  ],
  "last_updated": "2026-05-01T00:00:00Z"
}
~~~

## Publication

A Subject Authority publishes the Issuer Authorization Policy in
two channels:

- A DNS TXT record (see {{dii-dns-record}}), supporting either inline
  authorization of Assertion Issuers or a pointer to a JSON document
  hosted on any HTTPS endpoint.

- A JSON document at an HTTPS well-known URL on the Subject Authority's
  own host (see {{dii-https-url}}).

A Subject Authority MAY use either channel or both. Consumers locate
the policy using the procedure in {{dii-lookup}}.

Most deployments find DNS publication easier than HTTPS publication.
Hosting the HTTPS well-known URL requires control of the Subject
Authority's apex web origin, which is often dedicated to a marketing
site behind a CDN. Subject Authorities that need richer policy than
the inline DNS form supports (validity windows, format restrictions,
multiple ordered issuers) are encouraged to use the DNS pointer
form, which lets the policy document live on any HTTPS endpoint
under separate operational control.

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

## Lookup Procedure {#dii-lookup}

To retrieve the Issuer Authorization Policy for a Subject Authority `A`,
a consumer performs the following steps. The procedure is shared by
Resource Authorization Servers performing verification
({{dii-verification}}) and clients performing Assertion Issuer
discovery ({{dii-hrd}}), with one asymmetry noted at the end.

1. If DNS is to be consulted (see asymmetry below), query the DNS TXT
   resource record set at `_oauth-issuer-policy.{A}`. Classify the
   response as:

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
      rejecting malformed `uri=` or `issuer=` values and records with
      neither `uri=` nor `issuer=`. Any such condition is a
      `malformed` outcome.

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

3. If the DNS response is `negative-authoritative` (or DNS was not
   consulted), fetch the policy from the default HTTPS well-known URL
   per {{dii-https-url}}.

4. If the DNS response is `indeterminate`, discovery has failed.
   Consumers MUST NOT fall back to HTTPS, since an attacker who
   suppresses DNS responses could otherwise force the consumer onto a
   path the attacker has compromised separately.

**Asymmetry between verifier and Assertion Issuer discovery
client:**

- A Resource Authorization Server consults DNS only when its trust
  policy advertises `dns_discovery: true`
  ({{trust-method-domain-authorized-issuer}}). When
  `dns_discovery` is
  absent or `false`, the verifier skips step 1 and retrieves only
  from the HTTPS well-known URL.

- An Assertion Issuer discovery client always consults DNS. The
  `dns_discovery` flag governs verifier behavior only; a Subject
  Authority that publishes solely via DNS still expects clients to
  find it.

### Failure Handling {#dii-failures}

Outcomes other than retrieval of a valid policy are categorized as:

`no-policy`
: A `negative-authoritative` DNS response combined with an HTTPS
retrieval that yields HTTP 404, 410, or equivalent. A verifier MUST
reject the assertion. A client reports that no policy exists.

`malformed`
: A DNS recognized record missing `authority=`, with a mismatched
`authority=` when no recognized record for the same query name has a
matching `authority=`, with neither `uri=` nor `issuer=` after
parsing, with multiple distinct `uri=` values, with a malformed
`uri=` or `issuer=` directive, or otherwise unparseable; or an HTTPS response with an unsupported
media type, an entity body larger than the consumer's configured
maximum, an HTTP status other than success, redirect, 404, 410, or
5xx, or a redirect loop or redirect target that is not HTTPS; or an
HTTPS response body that is not a syntactically valid Issuer
Authorization Policy document; or a JSON policy whose
`subject_authority` does not match `A`. Consumers MUST NOT use a
fallback channel to recover, since a malformed authoritative result may
indicate an attack. A verifier MUST reject the assertion.

`indeterminate`
: A DNS `indeterminate` response, an HTTPS transport failure (TLS
failure, connection error), or an HTTPS response status indicating
server error (5xx). Consumers MUST NOT fall back to a channel that
might be under a different adversary's control. A verifier MUST
reject the assertion unless a fresh cached policy is available
({{dii-caching}}); a client SHOULD report a retryable error.

Consumers MAY follow HTTPS redirects when retrieving an Issuer
Authorization Policy, subject to a local redirect limit. Every
redirect target MUST use the `https` scheme and MUST NOT contain a
fragment component. The final response MUST have a successful HTTP
status and a media type compatible with `application/json`.

### Examples

Inline form (no HTTPS endpoint required on `acme.example`):

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://idp.bigprovider.example;
   issuer=https://backup-idp.example.net"
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
   procedure in {{dii-lookup}}. `no-policy`, `malformed`, or
   `indeterminate` outcomes ({{dii-failures}}) MUST result in
   rejection, except that a fresh cached policy MAY be used when the
   live retrieval is `indeterminate`.

3. Verify the policy's `subject_authority` matches the computed
   Subject Authority. (Virtual policies satisfy this by
   construction.)

4. Find an entry in `authorized_issuers` whose `issuer` matches the
   JWT `iss` claim using case-sensitive URL string equality.

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

A client that holds a subject identifier and wants to obtain an
identity assertion can use the same records to find the authoritative
authorization server(s) for the namespace:

1. Determine the Subject Authority `A` from the input subject
   identifier per {{dii-authority}}.

2. Retrieve the Issuer Authorization Policy by applying the
   procedure in {{dii-lookup}}. An Assertion Issuer discovery client always
   consults DNS regardless of any verifier-side `dns_discovery`
   flag, since that flag governs verifier behavior only.

3. Treat each `authorized_issuers` entry whose validity window
   includes the current time, and whose `subject_identifier_formats`
   (if present) includes the input subject's format, as a candidate.

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
directives the same way. Interactive clients MAY present candidates
to the end user.

A client performing Assertion Issuer discovery does not require
any relationship with a Resource Authorization Server. The records
stand on their own; the trust policy in the main body of this
document governs only how a Resource Authorization Server
later verifies assertions issued by the discovered issuer.

The Assertion Issuer discovery flow defined here is intended for
back-channel (server-to-server) consumers. It is not
privacy-preserving against the resolver path, the policy host, or
the discovered Assertion Issuer: a client performing this discovery
on behalf of a specific user exposes the user's Subject Authority
to each of those parties, and contacting the discovered Assertion
Issuer reveals which Resource Authorization Server (if any) the
client intends to target. For browser-mediated identity verification
flows (for example, RP-side email verification), the Email
Verification Protocol {{WICG-EMAIL-VERIF}} provides materially
stronger privacy properties through browser intermediation and
selective-disclosure tokens. Implementers building user-facing
verification flows SHOULD use {{WICG-EMAIL-VERIF}} rather than the
discovery flow defined here.

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

## Security Considerations {#dii-security}

The Issuer Authorization Policy is a security-critical document. Its
integrity determines which Assertion Issuers can assert identities in
the Subject Authority's namespace.

### Transport Integrity

For HTTPS retrieval, the integrity of the policy depends on TLS
server authentication of the host serving the policy. For the inline
DNS form, it depends entirely on DNS resolution. For the DNS pointer
form, it depends on DNS resolution plus TLS authentication of the
host named by the `uri=` directive.

Subject Authorities SHOULD deploy DNSSEC. A consumer of DNS-published
delegations without DNSSEC SHOULD use a trustworthy resolver path and
a conservative cache maximum. Subject Authorities concerned about TLS
misissuance SHOULD publish CAA records constraining the certificate
authorities permitted to issue for the policy-hosting domain.

Consumers that perform DNSSEC validation SHOULD treat validation
failure as `indeterminate`, not as `negative-authoritative`.

### Wildcard DNS Records

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

### Third-Party Policy Hosts

The DNS pointer form lets the Subject Authority delegate policy
hosting to a different domain. This is intentional and enables
managed delegation services. Once the Subject Authority publishes a
`uri=` directive, the named host is fully trusted to publish
authorizations on its behalf. Resource Authorization Servers MUST
NOT impose a same-origin restriction on the `uri=` target beyond
requiring `https://`.

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

### Document Authentication Limits

This document does not authenticate the policy document beyond
DNS and HTTPS. A future extension MAY define a JWS-signed policy for
deployments requiring offline-verifiable delegations or cross-origin
mirroring. The inline DNS form has no defined signing mechanism;
deployments requiring cryptographic proof of intent SHOULD use HTTPS
or the DNS pointer form pending such an extension.

### Scope of Authorization

A Resource Authorization Server MUST NOT use the Issuer Authorization
Policy to establish trust in an Assertion Issuer for subjects outside
the matched Subject Authority. The policy authorizes an Assertion
Issuer only within the namespace the Subject Authority controls.

### Email Local-Part Is Not Authenticated by This Trust Method

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

### Inline-Form Feature Limits

The inline DNS form expresses only "issuer X is authorized for
Subject Authority A." It cannot express `subject_identifier_formats`,
`valid_from`, or `valid_until`. Subject Authorities needing any of
those MUST use the HTTPS or DNS pointer form.

### Single-Issuer Multi-Tenant Identity Providers {#dii-multi-tenant}

Some Identity Providers serve many independent tenants under a
single issuer identifier (for example, `https://accounts.google.com`
serving every Google Workspace tenant). The tenant is distinguished
by a top-level JWT claim such as `hd` rather than by the `iss`
value. When a Subject Authority lists such an Identity Provider in
`authorized_issuers`, the trust grant is to the Identity Provider as
a whole, not to a specific tenant: the framework cannot from the JWT
alone determine which tenant minted the assertion.

Subject Authorities authorizing such Identity Providers MUST trust
the Identity Provider to enforce email-domain-to-tenant binding (or
equivalent namespace ownership) at issuance, so that no tenant other
than the Subject Authority itself can mint assertions about the
Subject Authority's subjects. Reputable Identity Providers require
domain-ownership verification before tenant provisioning; Subject
Authorities SHOULD verify that an Identity Provider performs this
check before listing it. Identity Providers that publish per-tenant
issuer identifiers (for example,
`https://login.microsoftonline.com/{tenant-id}/v2.0`) sidestep this
issue because each tenant has a distinct `iss` value that can be
listed individually. Deployments needing wire-format-enforced
per-tenant binding for single-issuer Identity Providers require a
future extension; this document does not provide such a mechanism.

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

## Trust Policy Is Not An Issuer Allowlist

This document intentionally allows a Resource Authorization
Server to publish trust criteria instead of a complete issuer
allowlist. Implementations MUST evaluate Trust Method evidence at
token request time or according to local cache policy.

## Trust Policy Does Not Replace Assertion Validation

A trust policy describes issuer acceptability. It does not validate
an assertion's signature, audience, expiration, replay protection,
subject, or client binding. Resource Authorization Servers MUST
validate the assertion according to the applicable grant profile
before issuing an access token.

## Policy Document Integrity {#integrity}

The trust policy document is the input to a security decision. A
Resource Authorization Server MUST serve it over HTTPS with TLS
server authentication; clients MUST reject documents retrieved over
non-HTTPS transports or with TLS failures.

Deployments needing integrity beyond TLS MAY serve a JWS-signed JSON
object using `application/jose+json` with a signing key resolvable
via the Resource Authorization Server's JWKS or federation entity
configuration. The signaling mechanism is out of scope for this
version.

Mirrored or cached copies MUST NOT be relied on beyond their HTTP
cache lifetime (see {{caching}}).

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

## Subject Namespace and Audience Asymmetry

The `subject_namespace_authorization` Trust Method category lets a
Subject Authority constrain which Assertion Issuers are entitled to
assert about subjects in its namespace. It does not constrain which
Resource Authorization Servers an authorized Assertion Issuer may
target. Once an Assertion Issuer is authorized for namespace `A`,
that Assertion Issuer can mint assertions about subjects in `A` for
any Resource Authorization Server that lists the relevant Trust
Method.

Resource Authorization Servers enforce audience binding through the
assertion's `aud` claim and the applicable grant profile. Subject
Authorities that need to restrict the set of audiences their
authorized issuers may target rely on out-of-band agreements with
those issuers; this document does not provide a wire-level
mechanism for that restriction. A future extension MAY add an
audience-scoping field to the Issuer Authorization Policy
document.

## Email Authority Granularity

For `email` Subject Identifiers, this document uses the registrable
domain derived from a current Public Suffix List as the Subject
Authority. This is a conservative choice: it prevents a party that
controls only a subdomain from publishing Domain-Authorized Issuer
Discovery records that override the organizational domain's issuer
policy. The tradeoff is that independently operated subdomains cannot
publish separate `email` authority policies unless the parent domain
delegates operational control through its own policy or the relevant
suffix is represented in the Public Suffix List.

Subject Authorities that require independent policy control for
subdomains SHOULD publish separate Subject Identifiers whose authority
extraction procedure preserves that delegation boundary, or use
out-of-band arrangements until such a procedure is registered in
{{iana-authority-registry}}.

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
| `openid_federation` | `issuer_authentication` | `trust_anchors` (array of string, REQUIRED) | This document |
| `domain_authorized_issuer` | `subject_namespace_authorization` | `dns_discovery` (boolean, OPTIONAL) | This document |
| `email_verification_dns` | `subject_namespace_authorization` | (none) | This document, {{WICG-EMAIL-VERIF}} |

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

Entries in this registry apply both to direct Subject Identifiers
({{dii-authority}}) and to actor Subject Identifiers carried by the
OAuth Actor Profile binding ({{actor-profile-binding}}); the
extraction procedure is the same. Future specifications that define
new actor-profile entity types or Subject Identifier formats are
expected to register additional entries here when those identifiers
have a well-defined namespace authority.

Initial entries:

| Subject Identifier Format | Subject Authority Form | Extraction Procedure |
|-|-|-|
| `email` | DNS domain | {{dii-authority}} of this document |

--- back

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
  about intermediates and leaf entities.
- **Assertion Issuer**, `https://idp.partner.example`. A federation
  leaf entity registered as an OpenID provider. Holds an Entity
  Configuration at `https://idp.partner.example/.well-known/openid-federation`.
- **Client**, an enterprise SaaS application. Authenticates to the
  Resource Authorization Server's token endpoint with `private_key_jwt`.
- **End user**, Alice (`alice@partner.example`).

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
category: OpenID Federation for issuer authentication and
Domain-Authorized Issuer Discovery for subject-namespace
authorization:

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
  "grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer",
      "dns_discovery": true
    }
  ]
}
~~~

The Federation Trust Anchor has previously issued a Subordinate
Statement about `https://idp.partner.example` listing it as an active
leaf with `openid_provider` entity type. The Assertion Issuer's
Entity Configuration includes `authority_hints` pointing back at the
trust anchor.

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
     "exp": 1748630400,
     "iat": 1748630100,
     "jti": "8e9b...",
     "sub": "user-7c2a4f",
     "email": "alice@partner.example",
     "email_verified": true,
     "acr": "urn:mace:incommon:iap:silver"
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

6. The Resource Authorization Server validates the ID-JAG:
   signature, `aud`, `exp`, `iat`, replay protection. The signing
   key is resolved via the Assertion Issuer's federation Entity
   Configuration.

7. The Resource Authorization Server evaluates
   `issuer_trust_methods_supported`. For the `issuer_authentication`
   category, it evaluates `openid_federation`. It builds a trust chain by walking
   `authority_hints` from `https://idp.partner.example` upward,
   collecting Subordinate Statements, and terminating at
   `https://federation.example.org`. The terminal trust anchor
   matches the listed `trust_anchors` entry.

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

- If `partner.example` does not authorize
  `https://idp.partner.example` in its Issuer Authorization Policy,
  the Resource Authorization Server returns `invalid_grant`.

- If the federation chain validates but the client's
  `client_assertion` is not signed by a registered key (for example
  because of unfinished key rotation), the Resource Authorization
  Server returns `invalid_client`.

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
- **Assertion Issuer**, `https://idp.bigprovider.example`. A managed
  authorization server service `acme.example` has contracted with.
- **Resource Authorization Server**, `https://api.resource.example`.
- **Client**, an agent provider acting on behalf of Alice.
- **End user**, Alice (`alice@acme.example`).

## Publication

The Subject Authority publishes a single DNS TXT record:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://idp.bigprovider.example"
~~~

No HTTPS endpoint is operated on `acme.example`.

The Resource Authorization Server publishes a trust policy that accepts
domain-authorized issuer delegations with DNS-based discovery:

~~~ json
{
  "resource_authorization_server": "https://api.resource.example",
  "grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "domain_authorized_issuer",
      "dns_discovery": true
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
       { "issuer": "https://idp.bigprovider.example" }
     ]
   }
   ~~~

5. The Client applies OAuth Authorization Server Metadata
   {{RFC8414}} to `https://idp.bigprovider.example` and resolves its
   token endpoint.

## Assertion Issuance

6. The Client authenticates Alice at `https://idp.bigprovider.example`
   and requests an ID-JAG with audience
   `https://api.resource.example` carrying Alice's email:

   ~~~ json
   {
     "iss": "https://idp.bigprovider.example",
     "aud": "https://api.resource.example",
     "exp": 1748630400,
     "iat": 1748630100,
     "jti": "5a17...",
     "sub": "user-9241",
     "email": "alice@acme.example",
     "email_verified": true
   }
   ~~~

## Token Exchange

7. The Client posts to the Resource Authorization Server's token
   endpoint with a DPoP proof and `private_key_jwt`:

   ~~~ http
   POST /token HTTP/1.1
   Host: api.resource.example
   Content-Type: application/x-www-form-urlencoded
   DPoP: eyJ0eXAiOiJkcG9wK2p3dCIs...

   grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
   &assertion=eyJhbGciOiJSUzI1NiIs...
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=eyJhbGciOiJFUzI1NiIs...
   ~~~

## Verification (Resource Authorization Server Side)

8. The Resource Authorization Server validates the ID-JAG:
   signature (via
   `https://idp.bigprovider.example/.well-known/openid-configuration`
   JWKS), `aud`, `exp`, `iat`, replay protection.

9. The Resource Authorization Server evaluates the
   `domain_authorized_issuer` Trust Method.

   a. It extracts the Subject Authority from the top-level `email`
      claim (with `email_verified=true`): `acme.example`.

   b. Because the trust policy has `dns_discovery: true`, the
      Resource Authorization Server queries DNS TXT at
      `_oauth-issuer-policy.acme.example`. The same record
      returned to the Client is returned here.

   c. The `authority=acme.example` directive matches. No `uri=` is
      present. The Resource Authorization Server constructs the same
      virtual policy the Client used in step 4.

   d. The ID-JAG `iss` value `https://idp.bigprovider.example` matches
      the single `authorized_issuers[0].issuer`. Verification
      succeeds.

10. The Resource Authorization Server validates the DPoP proof and
    `private_key_jwt`, then issues a DPoP-bound access token.

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
      "issuer": "https://idp.bigprovider.example",
      "subject_identifier_formats": ["email"],
      "valid_until": "2027-01-01T00:00:00Z"
    },
    {
      "issuer": "https://backup-idp.example.net",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-22T00:00:00Z"
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
  the old issuer until its cache expires. Subject Authorities
  SHOULD use short DNS TTLs during rotation; consumers SHOULD
  enforce a local cache ceiling ({{dii-caching}}).

# Agent Platform IdP End-to-End Example

This appendix is non-normative.

This example shows how the framework prevents an unauthorized
provider from impersonating users in a customer's email namespace.
The protection rests on a single deliberate choice by the customer:
publishing an Issuer Authorization Policy that lists the specific
agent platforms permitted to issue identity assertions about its
users. Without that authorization, no provider — however well-known
elsewhere — can mint an accepted ID-JAG for the namespace.

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
- **Tool Provider / Resource Authorization Server**,
  `https://toolprovider.example`. Verifies presented ID-JAGs
  against its trust policy and issues access tokens.
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
  "grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "domain_authorized_issuer",
      "dns_discovery": true
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
     "exp": 1748630400,
     "iat": 1748630100,
     "jti": "b9c1...",
     "sub": "alice-user-id",
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
  "exp": 1748630400,
  "iat": 1748630100,
  "jti": "c21f",
  "sub": "attacker-local-user"
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

`attacker.example` has no path to impersonate `alice@example.com`
unless `example.com`'s DNS publisher adds them to the Issuer
Authorization Policy. The customer is the sole gatekeeper.

## Additional Failure Variants

- If the customer's DNS record is missing or unreachable, the
  Trust Method outcome is `no-policy` or `indeterminate` per
  {{dii-failures}} and the tool provider rejects the assertion.
  An attacker cannot benefit from disabling the lookup;
  fail-closed semantics ensure rejection rather than acceptance.

- If `example.com` rotates its authorized agent platform (replaces
  `agentprovider.example` with a different provider in the
  policy), assertions from the old provider are rejected at the
  next cache refresh per {{dii-caching}}.

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
  the agent platform MAY add an `act` object to the ID-JAG
  identifying the agent instance. The OAuth Actor Profile binding
  ({{actor-profile-binding}}) governs how a Resource Authorization
  Server interprets such an `act` object. Trust evaluation of the
  agent's identity by this framework's `subject_namespace_authorization`
  category applies only when the actor's subject identifier is in a
  shape with a registered Subject Authority Extraction Procedure
  ({{iana-authority-registry}}); opaque agent-instance identifiers
  are typically not in such a shape and are subject to the agent
  platform's authorization-time logic and the tool provider's local
  policy.

# Related Token Contexts {#related-token-contexts}

This appendix is non-normative.

The trust framework's mechanics — Trust Method evaluation, Issuer
Authorization Policy lookup, Subject Authority extraction — apply
to several token contexts beyond identity assertion JWT
authorization grants. This appendix sketches how the same
machinery can be applied to two related contexts.

## Transaction Tokens

Transaction Tokens {{TXN-TOKENS}} carry authorization context
across calls in an identity-chained transaction. A txn-token
typically includes an `act` object identifying the actor that
initiated the chain together with the subject's identity claims.

A receiver verifying a txn-token applies:

- The OAuth Actor Profile binding ({{actor-profile-binding}}) when
  the txn-token carries an `act` object.
- The Resource Authorization Server Processing rules ({{rasp}})
  for evaluating the txn-token issuer's authority over the carried
  subject.

A deployment using txn-tokens MAY list a txn-token profile
identifier in the trust policy's `grant_profiles_supported` so
that a single trust policy document can scope itself to both
grant-endpoint and txn-token verification.

## JWT Access Token Verification

A resource server that verifies JWT access tokens carrying
identity claims faces the same issuer-trust questions as a
Resource Authorization Server: is the token's `iss` an authentic
issuer, and is it entitled to assert the identity claims the token
carries about the subject?

For this informal binding, the JWT access token's issuing
authorization server (identified by the token's `iss` claim) is
treated as the Assertion Issuer; Trust Method evaluation proceeds
against this identifier as it would against the `iss` of an
identity assertion.

A resource server MAY:

1. Reuse the trust policy published by its associated
   authorization server, when an exclusive AS-RS pairing exists.

2. Publish its own Identity Assertion Issuer Trust Policy by
   including the `identity_assertion_trust_policy_uri` member in
   its Protected Resource Metadata {{RFC9728}}. The policy's
   `resource_authorization_server` member names the authorization
   server whose tokens this particular policy governs - not the
   publisher of the policy. A resource server that accepts tokens
   from multiple authorization servers MAY publish multiple policy
   documents, each naming a different
   `resource_authorization_server`.

When the JWT access token carries a subject identifier recognized
by a registered Subject Authority Extraction Procedure
({{iana-authority-registry}}) — for example, top-level
`email`/`email_verified` claims — the resource server applies the
trust policy as described in {{rasp}}, with the JWT access token
in place of an identity assertion JWT.

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
