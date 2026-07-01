---
title: "OAuth Identity Assertion Trust Framework"
abbrev: "Identity Assertion Trust Framework"
docname: draft-mcguinness-oauth-id-assertion-framework-latest
date: 2026-06-23
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
  RFC7515:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8615:
  RFC8785:
  RFC9493:
  RFC9728:
  RFC5891:
  RFC8126:
  OIDF-FEDERATION:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/

informative:
  RFC5321:
  RFC6530:
  RFC7009:
  RFC7033:
  RFC7662:
  RFC8725:
  RFC9700:
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0"
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  DAI:
    title: "OAuth Domain-Authorized Issuer Trust Method"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer/
    date: false
  I-D.ietf-oauth-identity-chaining:
    title: "OAuth Identity and Authorization Chaining Across Domains"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/
    date: false
  RFC9334:

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

This document defines an Identity Assertion Trust Framework with two
parts. First, an Authority Delegation Model: an abstract pattern
(Authority Holder, Delegate, Delegation Artifact, Validator) with
independent trust-evaluation categories, a cross-category combination
rule, and a lookup-state taxonomy that profiles instantiate. Second,
the Identity Assertion Issuer Trust Policy: a JSON policy document
that a Resource Authorization Server publishes to declare which trust
methods it requires of an Assertion Issuer, including
issuer-authentication methods (such as OpenID Federation) and
subject-namespace authorization methods defined by separate
profiles.

The Domain-Authorized Issuer Trust Method is defined separately as one
subject-namespace authorization profile usable by this framework.

--- middle

# Introduction

OAuth assertion-based authorization grants {{RFC7521}} {{RFC7523}} allow
identity-bearing assertions issued by one authorization server to be
presented to another. The Identity Assertion JWT Authorization Grant
(ID-JAG) {{ID-JAG}} is one such profile, and the cross-domain delivery
of such assertions is specified in {{I-D.ietf-oauth-identity-chaining}}.
Those documents define how an assertion is constructed, presented, and
consumed; they do not define whether the Resource Authorization Server
should accept the issuer in the first place. The Resource Authorization
Server needs to decide whether the issuer of an identity assertion is
acceptable for subject resolution, account linking, or delegated access,
and base specs leave that decision to local policy.

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
policy is not to remove policy from the Resource Authorization
Server, but to give it a standard way to express which evidence it
requires and how that evidence is evaluated.

This document defines an Identity Assertion Issuer Trust Policy
that a Resource Authorization Server publishes to describe its trust
criteria. The Resource Authorization Server does not say "these are
all the Assertion Issuers I trust"; it says "these are the conditions
an Assertion Issuer must satisfy." Conditions are evaluated by
validating concrete evidence (a federation trust chain or a
domain-authorized issuer record) when an assertion is presented.

This document defines the Authority Delegation Model
({{delegation-model}}) and uses it to profile OAuth identity
assertions. It is consumed by {{DAI}}, which defines one
Subject-Authority publication mechanism. The two documents together
are described in {{family}}.

This framework operates at a layer {{ID-JAG}} and
{{I-D.ietf-oauth-identity-chaining}} do not address: those
specifications define how a Resource Authorization Server *consumes*
an identity assertion presented in a token request, but neither
answers whether the issuing authorization server is *entitled* to
assert about the subject's namespace. This document adds the
issuer-trust evaluation layer; the assertion-bearer grant and
chaining mechanics remain unchanged.

## Minimal Deployment

The smallest deployment has three moving parts:

1. A Resource Authorization Server publishes a Trust Policy
   ({{trust-policy-document}}) saying which issuer-trust evidence it
   requires.

2. A Subject Authority publishes a DAI policy ({{DAI}}) saying which
   Assertion Issuers it authorizes for its namespace.

3. At the token endpoint, the Resource Authorization Server validates
   the assertion and requires one successful Trust Method in each
   applicable trust-evaluation category.

For the common case, the Resource Authorization Server lists
`domain_authorized_issuer`; the Subject Authority publishes a DNS TXT
record at `_oauth-issuer-policy.{domain}`; and the verifier checks
that the assertion's `iss` appears in that policy for the asserted
subject namespace.

## Trust Evaluation Categories {#two-trust-questions}

This document defines two independent trust-evaluation categories
for OAuth identity assertions ({{categories}}):

- **`issuer_authentication`** is the Authenticity category. It
  asks: is the JWT `iss` claim a recognized signer?

- **`subject_namespace_authorization`** is the Delegation Authority
  category. It asks: has the namespace owner authorized this
  issuer to assert about subjects in its namespace?

Each Trust Method belongs to exactly one of these categories. The
cross-category combination rule ({{combination-rule}}) applies as
instantiated in {{rasp}}: evidence from EACH applicable category
is required.

## Motivating Use Cases {#motivation}

Deployments where the gap between issuer authentication and
namespace authorization has practical consequences include
workforce SSO into multi-vendor SaaS (per-customer bilateral
issuer configuration; no wire-format check that the configured
Assertion Issuer is the one the customer authorizes), AI agent
platforms acting across tool boundaries (the tool needs to know
the platform is entitled to assert about users in the customer's
namespace; see {{example-agent-platform}}), B2B integrations
carrying end-user identity (today either accept any authenticated
Assertion Issuer or maintain manual allowlists), and
shared-issuer multi-tenant Identity Providers (customer's choice
of authorized tenant becomes observable on the wire via
{{DAI}} §Single-Issuer Multi-Tenant Identity Providers rather than
implicit). Today's alternatives are bilateral OAuth
configuration, federation membership treated incorrectly as a
proxy for namespace authority, or implicit trust in tenant-domain
bindings; this document provides a wire-format alternative.

## Open-World Trust Evaluation

This document targets open-world deployments ({{open-world}}):
acceptance is based on verifiable evidence rather than prior
bilateral enumeration. Open-world does not mean open acceptance;
the Resource Authorization Server publishes the Trust Methods it
accepts and fails closed when required evidence is missing.

## Relationship to Existing Mechanisms

This policy is complementary to OpenID Federation: OpenID Federation
can authenticate that an issuer belongs to a trusted ecosystem, while
the Domain-Authorized Issuer Trust Method lets the namespace owner say
which issuers may assert about subjects in that namespace. This document also
follows existing DNS authority-publication patterns such as CAA,
MTA-STS, SPF, DKIM, and the Email Verification Protocol. Background and
positioning details are in {{relationship-to-oidf}} and
{{DAI}} §Following Existing DNS Authority Patterns.

This framework is distinct from issuer *discovery* mechanisms such as
WebFinger {{RFC7033}} and OpenID Connect Discovery {{OIDC-DISCOVERY}}.
Those answer "given a user identifier, which issuer should a client
use?" for a client that is about to authenticate the user. This
framework answers the verifier-side question "may this issuer, which
has already produced an assertion, be trusted for this subject's
namespace?" A namespace owner's authorization statement is not the
same as a per-user discovery record: it is published per namespace, it
expresses authorization rather than routing, and it is consumed by the
Resource Authorization Server at verification time. WebFinger could in
principle carry a discovery hint, but it is per-resource and HTTPS-only
and defines no verifier-side authorization semantics; see
{{DAI}} §Following Existing DNS Authority Patterns.

## Documents in the Family {#family}

This document and {{DAI}} form a two-document set:

- **This document**: the Authority Delegation Model
  ({{delegation-model}}) and the Identity Assertion Issuer Trust
  Policy ({{trust-policy-document}}) published by a Resource
  Authorization Server, plus the Trust Method machinery, OAuth
  grant profile bindings, and the Subject Authority Determination
  concept.
- **{{DAI}}**: the OAuth Domain-Authorized Issuer Trust Method, defining
  the `domain_authorized_issuer` Trust Method and the Issuer Authorization
  Policy wire format it consumes.

This document does not define the Issuer Authorization Policy
wire format; that lives in {{DAI}}. This document defines what an
Assertion Issuer must satisfy to be accepted; DAI defines one
class of evidence supplying that satisfaction.

## Scope {#scope}

The scope of this document is:

- The Authority Delegation Model ({{delegation-model}}): vocabulary,
  trust-evaluation categories, combination rule, lookup states,
  and the bounded-transitivity property profiles inherit.
- The Trust Policy document ({{trust-policy-document}}).
- The Trust Method machinery (categories, registry).
- The Subject Authority Determination concept and registry.
- The OAuth grant-profile bindings.

Concrete publication mechanisms for specific Trust Methods are
out of scope and live in dedicated specifications such as {{DAI}}.

The Authority Delegation Model accommodates attribute
authorities whose authority is not a namespace owner
(government-issued credentials, professional certifications,
employment attestations, decentralized credentials). This document
does not register Trust Methods for attribute attestation;
implementers needing such evaluation should look to W3C Verifiable
Credentials, OpenID for Verifiable Credentials, and domain-specific
frameworks. Where their attestations are conveyed as OAuth identity
assertions whose subject is namespace-bound, this document's
issuer-trust evaluation applies regardless of the
attribute-attestation layer beneath it.

Future extensions to additional Subject Identifier formats follow
{{future-extensions}} and require no changes to the trust evaluation
model.

The `email` Subject Identifier extraction registered in this
document is user-identity-oriented. The Authority Delegation Model
({{delegation-model}}) supports any Subject Identifier format with
a registered extraction procedure; identity chaining for workload
identities, agent identities, or other non-user subjects can be
supported by registering additional extractions
({{future-extensions}}).

## Conventions

{::boilerplate bcp14-tagged}

# Terminology {#terminology}

This document uses OAuth 2.0 {{RFC6749}} terminology. Subject
Identifier formats follow {{RFC9493}}; Trust Method identifiers are
registered in {{iana-trust-methods-registry}}.

The following terms define the Authority Delegation Model used
throughout this document.

Authority:
: The right to attest claims that a Validator will rely on, within
a specified namespace, claim type, scope, or registration. This is
distinct from "authority" in the access-management sense (the right
to perform actions or access resources). The IAM access-rights
sense of "authority" is out of scope.

Authority Holder:
: An entity holding authority over a namespace, claim type, scope,
or registration. An Authority Holder may delegate that attestation
authority to one or more Delegates. Examples: a DNS domain owner
for an email namespace; a federation operator for federation
membership.

Delegate:
: An entity that has received attestation authority from an
Authority Holder and may issue Assertions within the delegated
scope. In this document the Delegate is the Assertion Issuer.

Delegation Artifact:
: A profile-defined representation of a delegation: a signed
document, a DNS record, a JSON document at a well-known HTTPS URL,
an entry in published metadata, or any other profile-defined
publication form. The Delegation Artifact MUST carry sufficient
information to identify the Authority Holder, the Delegate, the
delegated claim type or scope, and (when applicable) the validity
bounds.

Assertion:
: A signed statement made by a Delegate about a Subject within the
delegated scope. In OAuth contexts this is typically a JWT
{{RFC7519}}.

Validator:
: The component that performs the trust evaluation: retrieves the
Delegation Artifact, validates the Assertion, applies the
combination rules, and produces an accept-or-reject result. In
this document the Validator is the Resource Authorization Server.
The Validator and Relying Party are conceptually distinct,
paralleling the analogous distinction in the RATS Architecture
({{RFC9334}}).

Authority Source:
: A trust root, registry, publication channel, or configured source
from which a Validator accepts Delegation Artifacts. In many cases
the Authority Source and the Authority Holder are the same entity
(for example, a domain publishing its own DNS record). In others
they differ (for example, a federation trust anchor as the
Authority Source for Subordinate Statements issued by
intermediates).

The following terms are specific to OAuth identity assertions.

Resource Authorization Server (RAS):
: The OAuth authorization server that receives an identity
assertion as a grant, evaluates the trust policy, and issues
access tokens for a protected resource. Not a new OAuth protocol
role; OAuth readers may read it as "the authorization server
receiving the assertion grant." The RAS is the Validator.

Assertion Issuer:
: The service issuing an identity-bearing assertion, identified by
the JWT `iss` claim. The same string is the `issuer` value in
OAuth Authorization Server Metadata {{RFC8414}}, OpenID Connect
Discovery {{OIDC-DISCOVERY}}, and the federation entity identifier
in {{OIDF-FEDERATION}}. The Assertion Issuer is the Delegate.

Subject Authority:
: The Authority Holder for a Subject Identifier namespace. For
`email`, the registrable domain.

Trust Policy:
: The JSON document defined in {{trust-policy-document}}, published
by a Resource Authorization Server to declare its trust criteria.

Issuer Authorization Policy:
: The Delegation Artifact by which a Subject Authority declares the
Assertion Issuers it authorizes for its namespace. Concrete
representations (wire format, publication channel) are supplied by
individual `subject_namespace_authorization` Trust Method specifications.

# Authority Delegation Model {#delegation-model}

This section is the explanatory model that the Trust Policy
machinery ({{trust-policy-document}}, {{trust-methods}}) and
profiles such as {{DAI}} instantiate. The vocabulary
(Authority Holder, Delegate, Delegation Artifact, Assertion,
Validator, Authority Source) is defined in {{terminology}}.

## Pattern Overview

Four roles cooperate around one delegation:

~~~
Authority Holder
   |  delegates authority via
Delegation Artifact
   |  identifies
Delegate
   |  signs
Assertion
   |  presented to
Validator
~~~

The Authority Holder publishes the Delegation Artifact through a
profile-defined channel: a DNS record under its domain, an HTTPS
document at a well-known URL on its host, a signed subordinate
statement in a federation, or another profile-defined form.
Control of that publication channel IS the authority binding:
whoever can publish at the channel is the Authority Holder for
the namespace, claim type, or scope. The Validator validates the
Delegation Artifact (which says "Authority Holder authorized
Delegate") and the Assertion (which says "Delegate asserts
something about Subject"). Both validations are required.

A delegation has a lifecycle: the Authority Holder establishes
the artifact, publishes it, the Delegate uses it to mint
Assertions, and revocation occurs by removing or expiring the
artifact. The framework provides no remote cache-invalidation
mechanism; revocation latency is bounded by the profile's cache
lifetime. Subject Authorities that need fast revocation operate
with short steady-state cache lifetimes. Planned transfers
(provider migrations, acquisitions) and unplanned revocations
(security incidents) follow the same publication path: update the
artifact, accept latency bounded by the cache window.

## Independent Trust Evaluation Categories {#categories}

A Validator combines two independent trust evaluations to accept
an Assertion. Each category answers a distinct question:

- **Authenticity**: is the entity that signed the Assertion
  cryptographically authentic and recognized as a member of some
  ecosystem? Satisfied by evidence that signers and their keys
  belong to a recognized population (federation membership,
  trust-mark issuance, signed attester key).

- **Delegation authority**: has the Authority Holder for the
  asserted scope delegated to this signer? Satisfied by a
  Delegation Artifact from the appropriate Authority Holder.

The OAuth Trust Method categories defined in this document
({{trust-method-categories}}) instantiate these:
`issuer_authentication` realizes Authenticity, and
`subject_namespace_authorization` realizes Delegation authority
for namespace-bound Subject Identifiers.

Local policy (the Relying Party's discretion: risk scoring, scope
grants, account linking, business rules) applies on top of these
two categories. Trust framework evaluation is necessary but not
sufficient; local policy is the final decision layer.

### Cross-Category Combination Rule {#combination-rule}

When more than one category is applicable to a request, the
Validator MUST require at least one satisfying evidence item from
EACH applicable category. Within a single category, OR-semantics
apply: satisfying any one applicable evidence item is sufficient.
The rule is **AND across independent categories, OR within a
category**.

Satisfying one category MUST NOT be treated as satisfying
another. A signer authenticated by federation membership has not
been delegated namespace authority by that fact; an Authority
Holder's delegation to a signer does not authenticate the
signer's identity beyond what the Delegation Artifact attests.

#### Category Applicability {#category-applicability}

Category applicability is a property of the Validator's published
Trust Policy, not of the incoming Assertion. If the Trust Policy
declares one or more Trust Methods in a given category, that
category is applicable to every Assertion the Validator evaluates
within the profile's scope. The Assertion itself MUST NOT be able
to waive the category: an Assertion lacking satisfying evidence
for an applicable category MUST be rejected. Profiles MUST NOT
define applicability conditions that depend on properties of the
Assertion under evaluation.

A `subject_namespace_authorization` method therefore does not
become inapplicable merely because an Assertion omits a Subject
Identifier. A profile that admits such a method MUST require, in
its grant-profile binding, that every Assertion in scope carry a
Subject Identifier from which a Subject Authority is determinable,
and MUST require rejection otherwise (see the processing rule in
{{rasp}} and {{applicability-bypass}} for the attack this
prevents).

### Multiple Authority Sources Within a Category {#multiple-sources}

A Validator MAY accept Delegation Artifacts from multiple
Authority Sources within the same category (multiple Subject
Authorities for different namespaces, multiple federation trust
anchors). Selection of WHICH Authority Source applies to a given
Assertion happens BEFORE the Assertion is authenticated against
any delegation, so a profile MUST define deterministic source
selection: a binding function from Assertion + request context to
exactly one Authority Source (or deterministic failure), invariant
under attacker-controlled inputs outside the binding function. The
attack against ad-hoc or fallback selection logic and the
deterministic-selection requirement are detailed in
{{unverified-claim}}.

A Validator MUST NOT fall through to a different Authority Source
if the originally-applicable Authority Source's evaluation fails
or is indeterminate ({{exception-handling}}).

## Open-World Delegation and Bounded Transitivity {#open-world}

This document targets **open-world delegation**: the Authority
Holder is an entity independent of the Validator (a customer's
domain, a federation operator) and the Validator retrieves
Delegation Artifacts through the profile's publication channel at
evaluation time. Open-world does not mean open acceptance; the
Validator publishes (or locally configures) which Authority
Sources it accepts. Closed-world delegation (Validator and
Authority Holder are the same party, evidence is local
configuration) is the degenerate case and remains compatible with
the model.

The Trust Methods defined in this document and in {{DAI}} are
bounded at depth one: the Authority Holder directly lists every
authorized Delegate; no further delegation is permitted. Depth-1
keeps revocation latency bounded by one cache and prevents
compromise of any non-Authority-Holder party from expanding the
authorized set. OpenID Federation {{OIDF-FEDERATION}} is the
notable chained-delegation profile in the OAuth ecosystem; the
cross-category combination rule still applies independently of
transitivity (a federation chain establishes Authenticity, not
Delegation authority).

## Lookup States and Fail-Closed {#exception-handling}

A Validator's lookup of a Delegation Artifact produces exactly
one of three abstract states. Profiles MUST map every concrete
outcome of the lookup operation onto exactly one of these states.

### Lookup States

- **Affirmative**: a well-formed Delegation Artifact was
  retrieved through the authoritative publication channel, its
  signature (where applicable) verified, and its validity bounds
  hold at evaluation time. The Validator proceeds to evaluate the
  Assertion against the artifact's contents.

- **Negative**: the publication channel authoritatively reports
  the absence of a Delegation Artifact. Examples include DNS
  NXDOMAIN or NODATA with a valid (possibly DNSSEC-signed)
  authoritative answer, HTTPS 404 from the authority-bound
  origin, or an explicit denial entry in the publication. A
  Negative state is itself a decision by the Authority Holder
  (the namespace exists but no delegation is in effect) and
  carries the same normative weight as any other published
  decision.

- **Indeterminate**: the lookup did not produce an authoritative
  Affirmative or Negative result. Examples include DNS SERVFAIL,
  resolver timeout, network partition, HTTPS 5xx, TLS handshake
  failure, malformed publication-channel response, and structural
  validation failure on a retrieved artifact (invalid signature,
  unparseable payload, validity bounds outside the acceptable
  window). An Indeterminate state carries no information about
  the Authority Holder's intent.

The Affirmative / Negative / Indeterminate taxonomy is
structural: Negative is the Authority Holder's affirmative
non-delegation; Indeterminate is information the Validator could
not obtain.

### Normative Requirements {#fail-closed-requirements}

A Validator MUST classify every lookup outcome into exactly one
of the three states above before producing an accept-or-reject
decision. Profiles MUST enumerate the concrete signals on their
publication channel that map to each state.

A Validator MUST fail closed on both Negative and Indeterminate
states: the access decision MUST be reject. Profiles MUST NOT
permit any condition under which an Indeterminate state is
treated as Affirmative; doing so converts the fail-closed
property into an availability-driven downgrade attack surface.

A Validator MAY rely on a fresh cached Affirmative Delegation
Artifact during a transient Indeterminate state on the live
publication channel, but only if the cached artifact is within
the cache lifetime bound the profile specifies. Repeated
Indeterminate states across consecutive lookups MUST NOT extend
the effective cache lifetime beyond the profile's stated maximum;
if the cache expires while the live channel remains
Indeterminate, the Validator MUST transition to a reject
decision.

A Validator MUST NOT fall through to a different Authority Source
on Negative or Indeterminate states from the
originally-applicable Authority Source ({{multiple-sources}}).
Fallthrough on non-Affirmative states is the same downgrade as
extended-cache-on-Indeterminate: both convert a hard denial into
a soft one driven by adversary-controllable availability of the
authoritative publication channel.

Profiles MAY define additional lookup states (for example, a
"stale-but-signed" state for transparency-log-bound artifacts)
but MUST place any such state on the Affirmative-or-reject side
of the decision boundary. Profiles MUST NOT introduce a state
that relaxes the Indeterminate-rejects-hard requirement.

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

A Resource Authorization Server may additionally publish the trust
policy at the well-known URI registered in
{{iana-trust-policy-well-known}}. The well-known URI is derived from
the Resource Authorization Server's issuer identifier by the
transformation of {{RFC8414}} Section 3: the well-known path component
`identity-assertion-trust-policy` is inserted between the host (and
port, if any) and any path component of the issuer identifier. For an
issuer with no path component this yields
`https://{host}/.well-known/identity-assertion-trust-policy`. If both
the metadata member and the well-known URI are available and identify
different documents, the metadata member is authoritative.

A consumer retrieving a Trust Policy document (from either source)
fetches it with an HTTP GET over HTTPS with TLS server authentication.
The response MUST have status 200 and a media type of `application/json`
or a `+json`-suffixed type (or, for a signed policy, the JWT form of
{{signed-policy-metadata}}); the consumer treats any other status, a
TLS failure, a cross-origin redirect, an unparseable body, or a body
exceeding a consumer-chosen limit (which MUST allow at least 64 KiB) as
a retrieval failure and MUST NOT act on a policy it could not retrieve
and validate. Redirects, if followed, MUST remain within the issuer's
origin. This retrieval failure is distinct from a successfully
retrieved policy that rejects an assertion.

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

The Identity Assertion Issuer Trust Policy is a JSON object served
over HTTPS with media type `application/json` (or a media type using
the structured `+json` suffix). Consumers MUST reject a policy whose
required members are absent or wrong-typed, MUST reject a document
containing duplicate member names at any object level, and MUST ignore
unrecognized members except those named in `crit` ({{critical-members}}).
The policy MAY include a `crit` member and a `signed_policy` member as
defined in {{signed-policy-metadata}} and {{critical-members}}.

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
({{RFC8414}}) to signal that distinction. Local policy may add
requirements per client, subject, or scope; see {{downgrade}}.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, using the
  representation defined in {{signed-policy-metadata}}. This member
  follows the signed metadata pattern used by {{RFC8414}} and
  {{RFC9728}}.

## Trust Methods {#trust-methods}

### Trust Method Object Structure {#trust-method-structure}

A Trust Method object is a JSON object with a string-valued `method`
member naming a Trust Method identifier registered in
{{iana-trust-methods-registry}} plus any members required by that
identifier. Consumers MUST reject a Trust Method object whose
required members are absent, wrong-typed, or out-of-constraint;
such objects are not usable. If no recognized, well-formed Trust
Method object remains, the policy identifies no usable issuer
trust method.

### Trust Method Categories {#trust-method-categories}

Each Trust Method belongs to one or more of the categories below
(registered in {{iana-trust-method-categories-registry}}) and is
itself registered in {{iana-trust-methods-registry}}. The
cross-category combination rule ({{rasp}}) requires evidence from
EACH category present in the policy: AND across categories, OR
within. A single evidence item is never counted toward more than one
category, even if its method is registered in several ({{rasp}}
step 5a).

`issuer_authentication`
: Is the entity identified by the JWT `iss` claim authentically a
member of a recognized ecosystem?

`subject_namespace_authorization`
: Is this Assertion Issuer entitled to assert about subjects in the
named namespace? When a policy lists a method in this category, every
in-scope assertion must carry a Subject Identifier resolving to a
Subject Authority, and the assertion fails closed if it does not
({{category-applicability}}, {{rasp}} step 5c); category
applicability is a property of the policy, not the assertion.

Deployments accepting assertions about namespace-bound subjects
SHOULD list at least one `subject_namespace_authorization`
method; federation membership alone does not establish namespace
authority. Future specifications MAY register additional categories
({{iana-trust-method-categories-registry}}).

### Requirements on Trust Method Specifications {#trust-method-spec-requirements}

A specification defining a new Trust Method (whether in this document
or a companion such as {{DAI}}) MUST provide all of the following, so
that the deferral from this framework to the method is testable:

- The Trust Method identifier and the category or categories it
  belongs to ({{iana-trust-methods-registry}}).
- The evidence the method consumes and the exact procedure by which a
  Validator decides the method is satisfied for a given (issuer,
  subject) pair, precise enough for interoperable implementation.
- The mapping of the method's retrieval and evaluation outcomes onto
  the Affirmative / Negative / Indeterminate lookup states
  ({{exception-handling}}), including which conditions fail closed.
- For methods in `subject_namespace_authorization`, the deterministic
  binding from the assertion's Subject Identifier to the Subject
  Authority whose evidence is consulted, consistent with the
  single-source-selection rule ({{multiple-sources}}).
- Cache-lifetime bounds for any retrieved evidence, or an explicit
  statement that the method caches nothing.
- Any method-specific parameters, their JSON types, and whether each
  is REQUIRED or OPTIONAL.

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

The Resource Authorization Server MUST validate the federation
trust chain, metadata policy, and Trust Marks per
{{OIDF-FEDERATION}}; failure of any is failure of this Trust
Method. Cache freshness and revocation follow {{OIDF-FEDERATION}}.

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
by evidence published by the Subject Authority itself (typically a
DNS or HTTPS record under the Subject Authority's domain). The
namespace-authorization trust graph is bounded to depth one
({{transitive-authz-bounded}}). Concrete methods in this category
are defined by companion specifications and registered in the
Trust Methods registry ({{iana-trust-methods-registry}}).

### Worked Example: OpenID Federation + DAI {#example-federation-dai}

This non-normative example shows the two trust-evaluation
categories used together. A SaaS Resource Authorization Server
(`api.saas.example`) accepts identity assertions from federation
member Assertion Issuers about users in any customer namespace;
the customer authorizes which specific Assertion Issuer serves
its users.

The Resource Authorization Server publishes:

~~~ json
{
  "resource_authorization_server": "https://api.saas.example",
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

Customer `acme.example` publishes a DAI record naming
`https://idp.example.net` as its authorized Assertion Issuer.
That Assertion Issuer is also a federation member of
`https://federation.example.org`.

A token-request flow with an ID-JAG carrying
`email: alice@acme.example`, `email_verified: true`,
`iss: https://idp.example.net`:

1. The Resource Authorization Server resolves both Trust Methods
   are applicable (the Trust Policy lists one in each category).

2. **Authenticity** (`issuer_authentication` category): the
   Resource Authorization Server validates the OpenID Federation
   trust chain from `https://idp.example.net` to
   `https://federation.example.org`. This proves the Assertion
   Issuer is an authentic federation member but says nothing
   about its authority over `acme.example`.

3. **Delegation authority** (`subject_namespace_authorization`
   category): the Resource Authorization Server extracts
   `acme.example` from the `email` claim, looks up the DAI record
   at `_oauth-issuer-policy.acme.example`, and confirms
   `https://idp.example.net` appears in `authorized_issuers`. This
   proves Acme delegated naming authority over its namespace to
   this specific Assertion Issuer.

4. The cross-category combination rule ({{combination-rule}}) is
   satisfied: one Trust Method succeeded in each applicable
   category. The Resource Authorization Server proceeds with
   `private_key_jwt` client authentication and access-token
   issuance.

If the Assertion Issuer were federation-authenticated but Acme
had not listed it in its DAI record, step 2 would succeed and
step 3 would fail; the Resource Authorization Server rejects with
`invalid_grant`.
Federation membership alone does NOT establish namespace
authority; the combination rule is what enforces this. A deeper
walkthrough including federation trust-chain validation and Trust
Mark satisfaction is in {{example-federation-walkthrough}}.


## Subject Authority Determination {#subject-authority-determination}

Determining the Subject Authority has two steps: first identify which
registered Subject Identifier format the assertion carries, then apply
that format's extraction procedure. The grant-profile binding
({{bindings}}) designates which claim conveys the Subject Identifier
and thus which format applies; for the bindings in this document, a
top-level `email` claim (with `email_verified`) is the `email` format.
A format is "used by the assertion" when the designated claim for that
format is present. If the designated claim maps to no registered
format, the `subject_namespace_authorization` category cannot be
evaluated and processing follows {{rasp}} step 5c.

The Subject Authority associated with a Subject Identifier is then
determined as registered in {{iana-authority-registry}}. This
document registers the `email` extraction as the initial entry. The
extraction-procedure pattern (Subject Identifier format to Subject
Authority) is open: future specifications register additional
formats for service identities, decentralized identifiers,
federated handles, and other namespaces by adding entries to the
same registry without changes to the rest of the framework. See
{{future-extensions}}.

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

  The domain is the substring after the single `@`. Consumers MUST
  reject an `email` claim value that does not contain exactly one `@`
  character or whose domain part is empty; this document uses the
  simple single-`@` rule rather than the full {{RFC5321}} addr-spec
  grammar, and quoted local-parts or address forms that do not reduce
  to a single unquoted `@` are out of scope for the `email` extraction.
  The local-part is not used. A trailing dot on the domain, if present,
  is removed. The domain is converted to A-label form per {{RFC5891}}
  (applying IDNA processing, including Unicode normalization) BEFORE
  any Public Suffix List matching, so that comparison operates on a
  single canonical form.

  The A-label domain is then normalized to its registrable domain
  ("eTLD+1") by applying the Public Suffix List matching algorithm
  {{PSL}} (including its wildcard `*` and exception `!` rules): the
  Subject Authority is the shortest suffix of the domain that is one
  label longer than the longest matching public suffix. Consumers MUST
  use both the ICANN and PRIVATE divisions of the list, so that a
  delegated namespace listed in the PRIVATE division (for example,
  `team.example-pages.example`) resolves to that namespace rather than
  to the operator's registrable domain; a consumer that used only the
  ICANN division would compute a different, coarser Subject Authority
  and query a different name. The resulting Subject Authority is
  compared using case-insensitive ASCII comparison of A-labels.
  Normalization to the registrable domain prevents an attacker who
  controls a subdomain (for example, via subdomain takeover) from
  publishing a Subject Authority record that would override the
  legitimate record at the registrable domain. Consumers MUST reject
  an email whose domain is itself a public suffix (no registrable
  domain exists) and MUST reject a domain that is not a valid A-label
  or U-label sequence.

Subject Identifier formats not registered for this purpose MUST NOT
be evaluated under this Trust Method; the Resource Authorization
Server MUST reject the assertion with an OAuth `invalid_grant`
error.

Subject Authority extraction MUST be exact-match: wildcard, suffix,
regular-expression, and substring matching against Subject Authority
values are forbidden unless explicitly specified by the relevant
Subject Identifier format's extraction procedure.

### Public Suffix List Versioning {#psl-versioning}

Deriving a registrable domain from a DNS name has no protocol
solution: the IETF DBOUND working group examined the problem of
determining administrative (organizational) boundaries in the DNS and
concluded without a standard, and the Public Suffix List remains the
de facto mechanism (it is used the same way by DMARC organizational-
domain discovery and by cookie same-site rules). This framework
inherits the PSL's limitations knowingly.

The PSL {{PSL}} is updated continuously, and snapshots taken at
different times can yield different registrable domains for the same
email. Because the Subject Authority is the DNS name queried, two
Resource Authorization Servers using different PSL snapshots can
compute different Subject Authorities for the same assertion and thus
query different names and reach different decisions; this is a
deterministic-source-selection hazard ({{multiple-sources}}) and a
security consideration, not merely an operational one, during the
window in which a label's public-suffix status is changing. Consumers
SHOULD use a snapshot of the {{PSL}} no older than 30 days.
The determinism property this framework claims holds for a given
computed Subject Authority; parties that require identical Subject
Authority computation across verifiers (for example, a Subject
Authority and the Resource Authorization Servers that consume its
policy) SHOULD agree on, or pin, a PSL snapshot. Subject Authorities
SHOULD monitor PSL changes affecting their namespace.

### Subdomain Authority {#subdomain-authority}

The registrable-domain default prevents subdomain-takeover from
capturing parent-domain authority. A subdomain-exact variant is
sketched in {{future-extensions}} and would require the parent
registrable-domain authority to explicitly delegate to the
subdomain.

## Signed Policy Metadata {#signed-policy-metadata}

The Trust Policy and Issuer Authorization Policy documents MAY include
a `signed_policy` member that provides cryptographic integrity for
signed policy claims. When the signed JWT contains all recognized
decision-affecting policy members, `signed_policy` can provide
object-level integrity for the policy document. This member follows
the signed metadata pattern defined for authorization server metadata
in {{RFC8414}} and protected resource metadata in {{RFC9728}}.

The `signed_policy` value is a JWT {{RFC7519}} in JWS Compact
Serialization {{RFC7515}} containing policy members as claims. The
JWT MUST be digitally signed using an asymmetric algorithm, MUST
contain an `iss` claim identifying the party attesting to the signed
policy claims, and MUST contain `iat`. It MUST contain `exp`, so that
a superseded signed policy cannot be replayed indefinitely (relevant
in the shared-infrastructure scenario for which signing is
recommended, {{shared-infrastructure}}); consumers MUST reject an expired
`signed_policy`. The JOSE header SHOULD contain a `kid` identifying
the signing key. The JWT payload SHOULD NOT contain a `signed_policy`
claim.

Algorithms: {{RFC8725}} (JWT Best Current Practices) applies
unchanged. In addition, the JWT MUST NOT use a MAC algorithm
(HS256/384/512); verification keys are widely distributed and a
MAC scheme would require sharing the signing key with every
Validator, defeating the authority binding. Implementations MUST
support ES256 and SHOULD support EdDSA and ES384; RS256 with
>=2048-bit keys MAY be supported for compatibility.

Per {{RFC8725}} §3.11 (cross-context confusion), the JWT `typ`
header MUST be `trust-policy+jwt` for Trust Policy documents or
`issuer-authorization-policy+jwt` for Issuer Authorization Policy
documents.

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

  The verification key for an Issuer Authorization Policy signer MUST
  be resolved through a channel independent of the one that carried the
  policy document, or the signature adds no protection: an attacker who
  controls the publication channel can substitute both the policy and,
  if the key is fetched over that same channel, the key. A profile that
  admits `signed_policy` on the Issuer Authorization Policy MUST specify
  at least one of the following key-resolution mechanisms and state its
  trust assumptions: (a) a key published under DNSSEC-signed records for
  the Subject Authority; (b) a key resolved through a federation or
  trust-anchor relationship established by an `issuer_authentication`
  Trust Method; or (c) a key configured out of band at the consumer.
  Absent an independent channel, `signed_policy` on an Issuer
  Authorization Policy provides integrity no stronger than control of
  the publication channel itself (which already establishes authority),
  and consumers MUST NOT rely on it to defend against compromise of
  that channel. Fetching the key over the same non-DNSSEC DNS/HTTPS
  path that delivered the policy does not satisfy this requirement.

If both unsigned policy members and `signed_policy` are present, the
signed policy claims MUST be used as the policy values for all claims
present in the JWT. Unsigned members that are not represented as claims
in the JWT MAY be used subject to the normal processing rules for
unrecognized members. A conflict exists when a member name appears in
both the unsigned outer document and the signed JWT payload AND the
two values are not equal when compared as parsed JSON values (member
order and insignificant whitespace ignored; equivalently, their JCS
{{RFC8785}} serializations differ). Consumers MUST reject a policy
that contains any such conflict; an attacker who can modify the outer
document but not the signed JWT otherwise has a lever to inject
visible-but-ignored members that may mislead operators or downstream
tooling.

A publisher that needs to require `signed_policy` processing by all
conforming consumers lists `signed_policy` in the document's `crit`
member ({{critical-members}}); a consumer that does not implement
`signed_policy` processing then rejects the document rather than
silently ignoring the signature. Absent a `crit` entry, the presence
of `signed_policy` provides integrity only for consumers that support
and verify it, or for deployments where local policy requires signed
policy processing.

If a consumer requires object-level integrity by local policy, the
consumer MUST verify the signed JWT before acting on the policy, and
the JWT payload MUST contain every recognized decision-affecting
member used by that consumer. The consumer MUST NOT use unsigned
recognized decision-affecting members that are absent from the JWT
payload. If signature verification fails, if the verification key is
unacceptable, if the JWT is malformed, if the required issuer binding
above is not satisfied, or if the JWT omits a recognized
decision-affecting member required for evaluation, the consumer MUST
reject the policy as malformed.

## Critical Members {#critical-members}

The Trust Policy and Issuer Authorization Policy documents MAY include
a `crit` member: a JSON array of strings naming other members of the
same document whose correct processing is REQUIRED for safe
interpretation. A consumer that does not recognize, or does not
implement processing for, any member named in `crit` MUST reject the
document as malformed rather than ignoring the unrecognized member.
Members not named in `crit` retain the default handling: unrecognized
members are ignored.

`crit`, when present, MUST be a non-empty array of strings; each string
SHOULD name a member that is actually present in the document. A
consumer MUST reject the document if `crit` is present but not a
non-empty array of strings, and MUST reject if `crit` names `crit`
itself. This is the same fail-closed pattern JWS ({{RFC7515}}
Section 4.1.11) uses for critical header parameters.

This mechanism is defined in the base specification, with no member
named critical by default, precisely so that a future extension can
mark a new decision-affecting member critical and have already-deployed
consumers honor it: a consumer conformant to this document already
rejects documents whose `crit` names a member it does not understand.
An extension that omitted this from the base could not retrofit
fail-closed behavior onto the deployed base. The DNS record form uses
its version token ({{DAI}}) for the analogous purpose and does not
carry `crit`.

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

## Resource Authorization Server Processing {#rasp}

This section specifies the normative processing procedure. It is the
OAuth-identity-assertion instantiation of the Authority Delegation
Model ({{delegation-model}}): each Trust Method category in the
applicable Trust Policy is evaluated under the cross-category
combination rule ({{combination-rule}}); lookup outcomes are
classified per {{exception-handling}}; source selection is
deterministic per {{multiple-sources}}.

When evaluating an identity assertion JWT presented in a token
request, the Resource Authorization Server MUST:

1. Select and parse the trust policy that applies to the token request.
   If the policy document is malformed, reject the assertion.
   (Recognition of individual Trust Method objects is handled in
   step 5a.)

2. Validate the assertion per the applicable grant profile
   ({{RFC7521}}, {{RFC7523}}, {{ID-JAG}}).

3. Verify that the applicable grant profile is listed in
   `authorization_grant_profiles_supported`.

4. If `subject_identifier_formats_supported` is present, verify that
   the Subject Identifier format used by the assertion is listed,
   unless the applicable grant profile defines a different format
   selection rule.

5. Verify that the Assertion Issuer identified by `iss` satisfies the
   Trust Method combination rule:

   a. Partition the Trust Method objects in `issuer_trust_methods` by
      category (see {{trust-method-categories}}). A Trust Method
      registered with multiple categories belongs to each. If any
      object in `issuer_trust_methods` is not recognized, is
      malformed, or cannot be assigned to a category known to the
      Resource Authorization Server, the Resource Authorization
      Server MUST reject the assertion: it cannot determine whether
      it is evaluating the full set of requirements the operator
      declared, and silently ignoring the object would evaluate a
      strict subset (a fail-open category downgrade). A single
      evidence item MUST NOT be counted toward more than one category
      even when its Trust Method is registered in several.

   b. Each category present in the partitioned policy is applicable to
      every assertion evaluated under this policy; applicability is a
      property of the policy, not the assertion ({{category-applicability}}).
      For each present category, the Assertion Issuer MUST satisfy at
      least one Trust Method from that category.

   c. When the policy contains a `subject_namespace_authorization`
      Trust Method, the assertion MUST carry a Subject Identifier from
      which a Subject Authority can be determined per
      {{subject-authority-determination}} or an equivalent registry.
      If the assertion carries no such Subject Identifier, or no
      Subject Authority can be determined, the Resource Authorization
      Server MUST reject the assertion. The Resource Authorization
      Server MUST NOT treat the `subject_namespace_authorization`
      category as inapplicable because the assertion omits a Subject
      Identifier, and MUST NOT consume any subject or identity claim
      from the assertion for account linking, authorization, or
      subject resolution when the namespace category is present but
      cannot be evaluated.

   d. Local policy MAY require satisfaction of additional Trust
      Methods for specific clients, subjects, or scopes.

   When the policy lists only `issuer_authentication` methods and the
   assertion carries a namespace-bound Subject Identifier, the
   Resource Authorization Server MUST reject the assertion (or apply a
   stricter local policy that independently establishes authority over
   the subject namespace), since federation membership alone does not
   establish authority over a particular subject namespace.

6. Apply local policy (account-linking, consent, authorization,
   risk) and the applicable grant profile's client authentication
   and sender-constraining; this document does not specify either.

Failure to satisfy issuer trust, subject identifier, or assertion
claim requirements in the trust policy MUST result in an OAuth
`invalid_grant` error unless another error is defined by the
applicable grant profile. Detailed trust-evaluation failure state
MUST NOT be returned to public clients in the OAuth error response;
it is a reconnaissance target.

# Grant Profile and Token Bindings {#bindings}

This section defines bindings for ID-JAG ({{id-jag-profile}}) and
the generic JWT-bearer assertion grant ({{jwt-bearer-profile}}).
Other assertion-bearing grant profiles would supply analogous
bindings, naming their grant profile identifier, their Subject
Identifier-bearing claim, and any profile-specific JWT claims that
participate in Trust Method evaluation.

## ID-JAG {#id-jag-profile}

For ID-JAG {{ID-JAG}}, `authorization_grant_profiles_supported` contains the value
`urn:ietf:params:oauth:grant-profile:id-jag`. When the trust policy
requires subject identifier evaluation, the Resource Authorization
Server applies the policy to the email attribution carried by the
ID-JAG. The email is taken from the top-level `email` claim,
accompanied by `email_verified=true`, per {{subject-authority-determination}}.

In addition to the processing in {{rasp}}, the Resource Authorization
Server MUST:

1. Validate the ID-JAG per {{ID-JAG}}.

2. Verify that the email attribution carried by the ID-JAG, when
   present or required, uses a Subject Identifier format registered
   in `subject_identifier_formats_supported`, if that trust policy
   member is present.

3. Use the email attribution only for subject resolution and Subject
   Authority evaluation. The Resource Authorization Server MUST NOT
   treat the mere presence or value of the email claim as proof
   that the ID-JAG issuer is authoritative for the subject; issuer
   acceptability is established only by evaluating the Trust Methods.

## Generic JWT-Bearer Assertion Grant {#jwt-bearer-profile}

This section provides the binding for the JWT-bearer authorization
grant of {{RFC7523}} Section 2.1 when the assertion carries an
identity claim to which Subject Authority Determination applies.
Other identity-carrying grant profiles (e.g., ID-JAG,
{{id-jag-profile}}) supply their own bindings.

A Resource Authorization Server that accepts generic RFC 7523
JWT-bearer assertion grants under this framework advertises
`urn:ietf:params:oauth:grant-type:jwt-bearer` in
`grant_types_supported`. RFC 7523 defines no distinct
grant-profile identifier; for the purposes of this framework the
grant-type URN `urn:ietf:params:oauth:grant-type:jwt-bearer` is
also used as the grant-profile identifier and, when a deployment
accepts this grant, MUST appear in
`authorization_grant_profiles_supported` so that the processing
rule in {{rasp}} step 3 is satisfied uniformly.

Unlike ID-JAG, the generic JWT-bearer grant does not fix which
claim conveys the subject. To preserve the deterministic
single-source-selection requirement of {{multiple-sources}}, a
Trust Policy that admits this grant MUST designate exactly one
top-level claim as the identity claim for Subject Authority
evaluation, chosen once for the policy rather than per assertion:

- the `email` claim, which MUST be accompanied by
  `email_verified=true` (the extraction registered in
  {{subject-authority-determination}}); or
- the `sub` claim, permitted only when the Assertion Issuer and
  the Resource Authorization Server have agreed that `sub` conveys
  an email-form identifier for this deployment, AND the assertion
  also carries `email_verified=true`. `sub` is an issuer-scoped
  identifier ({{RFC7519}}) that may merely resemble an email; it
  MUST NOT be used as a Subject Identifier without the same
  verification signal the `email` extraction requires.

The designated source MUST be applied to every assertion evaluated
under the policy; the Resource Authorization Server MUST NOT fall
back from one claim to another per assertion, and MUST ignore the
non-designated claim for Subject Authority evaluation. "Email form"
means a value satisfying the email-claim syntax of
{{subject-authority-determination}}. A value that is not in email
form under the designated source yields no determinable Subject
Authority and is handled per {{rasp}} step 5c.

In addition to the processing in {{rasp}}, the Resource
Authorization Server MUST:

1. Validate the JWT per {{RFC7523}}.

2. Verify that the Subject Identifier drawn from the designated
   claim uses a format registered in
   `subject_identifier_formats_supported`, if that trust policy
   member is present.

3. Treat the identity claim only as input to Subject Authority
   evaluation. Issuer acceptability is established only by
   evaluating the Trust Methods.

Because the generic JWT-bearer grant carries no `tenant` claim, an
Issuer Authorization Policy entry that omits `tenant` authorizes
every tenant of the named issuer (see {{DAI}}). A Subject Authority
that relies on a shared multi-tenant Assertion Issuer (one `iss`
serving many tenants) therefore cannot express tenant-scoped
authorization for assertions delivered under this grant, and SHOULD
NOT authorize such an issuer for the generic JWT-bearer grant unless
per-tenant issuer identifiers are used.

This binding does not apply to JWT client authentication
({{RFC7523}} Section 2.2), where the Subject Identifier is the
`client_id` registered with the authorization server and no
external namespace-owner delegation exists.

# Security Considerations

## Alignment with OAuth Security BCP {#rfc9700-alignment}

This document complements the OAuth 2.0 Security Best Current
Practice ({{RFC9700}}); it does not duplicate or override it. The
`subject_namespace_authorization` category is the wire-format
analog of {{RFC9700}} §4.4 (AS mix-up mitigations): to the extent the
Subject Authority's publication channel provides integrity, a Resource
Authorization Server will not accept an assertion from an AS that the
Subject Authority has not listed. This guarantee is only as strong as
the integrity of that channel: a Trust Method whose evidence is
published over unauthenticated DNS or a compromisable HTTPS origin can
be defeated by an attacker who controls that channel (see the
per-Trust-Method security analysis, e.g. {{DAI}}, and {{multiple-sources}}).
It is not an unconditional guarantee. Other
{{RFC9700}} topics (redirect URI validation, bearer-token replay,
client authentication) apply at the grant-profile and
client-authentication layers and are not addressed here.
Implementations SHOULD follow {{RFC9700}} at those layers.

## Unverified Claim Exploitation {#unverified-claim}

Source selection ({{multiple-sources}}) chooses which Authority
Source's delegation applies to the incoming Assertion. At the
point of selection, the Assertion's signature has not yet been
validated against any Delegation Artifact; the Assertion is an
untrusted payload. A profile whose source-selection algorithm
considers multiple claims, applies preference ordering, or falls
back from one input to another gives an attacker a selection lever.

The attack: the attacker constructs an Assertion whose claims
satisfy multiple potential bindings. The Validator's selection
algorithm picks the weakest (or most attacker-favorable) Authority
Source's policy. The Assertion is evaluated against a Delegation
Artifact that the legitimate Authority Holder for the target
namespace never authorized.

Worked example: an Assertion carries `sub: alice@victim.example` AND
`tenant_context: attacker-tenant.example`. A loosely written
profile selects the Authority Source by tenant context when
present, falling back to subject domain. The Validator selects
attacker-tenant.example, fetches that Authority Holder's
Delegation Artifact (which the attacker controls), validates the
Assertion against it, and accepts a claim about a victim.example
subject.

The mitigation is **deterministic source selection**
({{multiple-sources}}): a binding function from Assertion +
request context to exactly one Authority Source. Profiles MUST
define that function explicitly and the function MUST:

- Use explicitly defined inputs (a single named claim, header,
  request parameter, or fixed tuple).
- Produce exactly one Authority Source for any valid input, or
  fail deterministically (no preference ordering, no fallback
  across candidates).
- Be invariant under attacker-controlled inputs outside the
  binding function: the presence, absence, or value of any other
  claim MUST NOT alter the selected Authority Source.

The `email` extraction registered by this document derives the
Subject Authority deterministically from a single claim and does
not fall back; profiles registering additional extractions MUST
preserve this property.

## Applicability Bypass {#applicability-bypass}

The cross-category combination rule ({{combination-rule}})
requires the Validator to evaluate every applicable category. A
profile that lets category applicability depend on the incoming
Assertion gives the attacker a way to waive the category by
constructing a matching Assertion.

The attack: the profile (or local configuration) declares "the
delegation-authority category is not applicable when the
Assertion is from a legacy issuer" (or carries a legacy-flag
claim, or fails some heuristic that signals "legacy"). The
attacker constructs an Assertion matching the legacy condition.
The Validator silently skips the entire delegation-authority
evaluation; the open-world defense layer is bypassed and the
Validator falls back to authenticity alone, which the
cross-category combination rule was explicitly designed to forbid
as sufficient.

The mitigation: category applicability MUST be a property of the
Validator's published Trust Policy, not of the incoming
Assertion. Migration scenarios where some traffic legitimately
predates a category's availability are handled at the policy
layer, not by silently waiving the category. Per-deployment-phase
policy is a configuration boundary; Assertion-driven waiver is an
attacker-controlled boundary.

## Authority Source Compromise {#authority-source-compromise}

The authority binding (publication channel) is the highest-value
target. A compromise of the publication channel (DNS hijack, TLS
misissuance plus DNS redirect, registrar account takeover,
compromise of the federation operator's signing key) substitutes
the Authority Holder's voice with the attacker's. Delegation
Artifacts published through the compromised channel are
indistinguishable from legitimate ones.

This framework does not provide cryptographic recovery from
publication-channel compromise. Recovery is operational. Profiles
SHOULD recommend operational defenses appropriate to their
publication channel (DNSSEC, registry-lock, CAA records,
Certificate Transparency monitoring, federation key rotation).
See {{DAI}} §Security Considerations for the DNS+HTTPS
publication-channel compromise model.

## Transitive Authorization is Bounded {#transitive-authz-bounded}

The two trust categories make different transitivity choices, both
permitted within the taxonomy of {{open-world}}:

- `issuer_authentication` permits chained transitivity. The
  `openid_federation` Trust Method accepts a chain from a leaf entity
  to a trust anchor; the trust anchor's signature on intermediate
  Subordinate Statements transitively authenticates the leaf.

- `subject_namespace_authorization` requires depth-1 (no chaining).
  A Subject Authority lists specific Assertion Issuers directly; it
  cannot delegate further to whoever Issuer X federates. Revocation
  latency is bounded by the Subject Authority's own cache lifetime
  and does not compound across delegations.

The two categories are independent: federation membership does not
establish namespace authority, and namespace authorization does not
establish authenticity. The cross-category combination rule
({{rasp}}) enforces this independence at evaluation time. See
{{relationship-to-oidf}} for positioning against OpenID Federation.

## Scope of Namespace Authorization {#scope-of-namespace-authorization}

Trust-policy evaluation establishes that the Assertion Issuer is
authorized to assert this class of Subject Identifiers under the
namespace. It is necessary but not sufficient; Resource
Authorization Servers MUST still validate the assertion (signature,
audience, expiration, replay protection, client binding) per the
applicable grant profile, and MUST NOT infer any of the following
from a successful trust-policy evaluation:

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

These properties are out of scope and obtained, if needed, through
mechanisms outside this framework (authentication-method/AAL
claims, fresh-authentication signals, account-status attestations,
out-of-band verification). In particular, `email_verified=true` is
a prerequisite for deriving namespace authority from the email's
domain; it is not evidence of current mailbox control.

The `subject_namespace_authorization` category constrains WHICH
Assertion Issuers may assert about a namespace; it does NOT
constrain which Resource Authorization Servers an authorized
Assertion Issuer may target. Audience binding is enforced by the
assertion's `aud` claim and the applicable grant profile, not by
the Subject Authority.

## Per-Assertion Revocation Is Out of Scope {#per-assertion-revocation}

This framework decides whether an Assertion Issuer is authorized;
it does not revoke individual assertions. JWT bearer tokens are
stateless and remain valid until `exp` regardless of session
termination, credential revocation at the Assertion Issuer, or
Subject Authority withdrawal of the issuer's authorization via
DAI (which prevents NEW assertions but does not invalidate
already-issued ones). Deployments requiring synchronous revocation MUST use OAuth
2.0 Token Revocation {{RFC7009}}, Token Introspection {{RFC7662}},
or short assertion lifetimes at the grant-profile layer.

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
documents on shared infrastructure and treat the shared edge as
outside their trust boundary MUST use object-level cryptographic
integrity for the policy document itself, such as the `signed_policy`
member ({{signed-policy-metadata}}), rather than relying on the TLS
channel to the shared edge. The signing key MUST be controlled by the
Subject Authority or Resource Authorization Server independently of CDN
tenant configuration, and MUST be resolvable through a channel
independent of the shared edge (see the key-resolution requirement in
{{signed-policy-metadata}}).

Because this specification defines no criticality mechanism that lets a
publisher force `signed_policy` processing, a `signed_policy` member is
only effective against an on-path or edge attacker if consumers are
configured to require it: such an attacker can otherwise strip the
`signed_policy` member and serve an unsigned document, which a consumer
not configured to require signatures would accept (a signature-stripping
downgrade). A consumer operating in a shared-infrastructure trust model
therefore MUST require and verify `signed_policy` before acting on the
policy, MUST reject a policy that omits it, and MUST treat a valid TLS
connection to a shared edge as insufficient by itself.

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

## Trust Policy Caching {#caching}

HTTP caching of the trust policy follows {{?RFC9111}}, subject to
local maximum cache lifetimes. Revocation status of validated trust
evidence MUST be checked at the cadence required by the applicable
Trust Method specification, independent of policy-document cache
expiration. Transport integrity is addressed in {{integrity}}.

## Observability

Trust-policy evaluation is a security-critical decision; deployments
are encouraged to log, for each processed assertion, at minimum the
Assertion Issuer identifier, the trust policy URI and its
`last_updated`, the Trust Methods that succeeded, the matched trust
anchor or Issuer Authorization Policy origin, the Subject Identifier
format, and the accept/reject outcome, and to support correlation
across the issuance and verification halves of an identity-chain
transaction.

# Privacy Considerations

Trust policies are typically published as unauthenticated HTTPS
resources. Adversaries can scrape them across many Resource
Authorization Servers to map federation deployment landscapes:
which RASes participate in which federations, which trust anchors
are accepted, which Subject Identifier formats are honored. This
information aids targeted attacks (for example, prioritizing
compromise of a heavily-relied-upon trust anchor). Operators
should publish only what clients need to determine whether they
can attempt issuance, and prefer trust-anchor or federation
expression over enumerating individual issuers.

Subject Authority lookup at the RAS reveals the queried Subject
Authority to DNS resolvers and, for HTTPS retrieval, to the policy
host. RASes verifying with `domain_authorized_issuer` should treat
the Subject Authority and Assertion Issuer relationship as
sensitive operational data and avoid sending full subject
identifiers in policy URLs, query parameters, logs, or telemetry.

# Internationalization Considerations

The `email` Subject Identifier format carries an internationalized
domain. Consumers convert the domain to A-label form per {{RFC5891}}
(applying IDNA2008 processing, including Unicode normalization) before
Public Suffix List matching and before comparison, and compare A-labels
using case-insensitive ASCII comparison
({{subject-authority-determination}}). Performing all comparison on the
A-label form means two visually distinct Unicode domains that map to
different A-labels are correctly treated as different Subject
Authorities; conversely, this mechanism does not by itself defend
against homograph confusion presented to a human at account-linking or
display time, which is out of scope and left to the consuming
application. Comparison operates on the mechanism level, not the visual
level.

The `email` extraction uses the simple single-`@` rule and does not
implement the full {{RFC5321}} addr-spec grammar; internationalized
email addresses (SMTPUTF8, {{RFC6530}}) whose local-part requires
UTF-8 are outside the scope of the `email` extraction defined here,
though the domain of such an address is handled normally once isolated.
A future Subject Identifier format may define richer address handling
by registering its own extraction procedure ({{iana-authority-registry}}).

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

### Identity Assertion Issuer Trust Method Categories Registry {#iana-trust-method-categories-registry}

IANA is requested to establish a new registry titled "Identity
Assertion Issuer Trust Method Categories" under the "OAuth Parameters"
registry group. This registry backs the category values used by the
Trust Methods registry and referenced by the combination rule
({{combination-rule}}).

Registration policy: Specification Required {{RFC8126}}.

Each registry entry contains a Category Name (character set `[a-z0-9_]`),
a Description of the trust question the category answers, a Change
Controller, and a Specification Document.

Designated Expert instructions: the expert verifies that the proposed
category answers a trust question genuinely distinct from existing
categories (so that the cross-category AND semantics of
{{combination-rule}} remain meaningful) and that the description makes
clear what evidence satisfies it. Categories that merely rename or
subdivide an existing category SHOULD be rejected.

Initial entries:

| Category Name | Description | Change Controller | Specification Document |
|-|-|-|-|
| `issuer_authentication` | Establishes that the Assertion Issuer is an authentic, recognized entity | IETF | This document |
| `subject_namespace_authorization` | Establishes that the Assertion Issuer is authorized by the subject's namespace owner | IETF | This document |

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
: One or more values registered in the Trust Method Categories
registry ({{iana-trust-method-categories-registry}}).

Parameters:
: Additional JSON members defined for this Trust Method, with their
JSON types and whether each is REQUIRED or OPTIONAL.

Change Controller:
: The party responsible for change control.

Reference:
: A reference to the specification defining the Trust Method.

Designated Expert instructions: the expert verifies that the
identifier is unique and descriptive; that each listed category is
registered; that the Trust Method's evidence and evaluation procedure
are specified precisely enough for interoperable implementation
(including its mapping onto the Affirmative/Negative/Indeterminate
lookup states and any cache-lifetime bounds, per
{{trust-method-spec-requirements}}); and that parameter names do not
collide in meaning with parameters of other methods in a way that
would be ambiguous when methods are combined.

Initial entries:

| Identifier | Categories | Parameters | Change Controller | Reference |
|-|-|-|-|-|
| `openid_federation` | `issuer_authentication` | `trust_anchors` (array of string, REQUIRED); `trust_marks` (array of object, OPTIONAL) | IETF | This document |

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

Change Controller:
: The party responsible for change control.

Specification Document:
: A reference to the specification defining the member.

Designated Expert instructions: the expert verifies that the member
name does not collide with an existing member, that its semantics and
JSON type are specified, and that any decision-affecting member states
how a consumer that does not recognize it behaves (the default is that
unrecognized members are ignored; a member requiring fail-closed
handling needs the criticality mechanism of {{critical-members}}).

Initial entries:

| Member Name | Member Description | Change Controller | Specification Document |
|-|-|-|-|
| `resource_authorization_server` | Resource Authorization Server issuer identifier | IETF | This document |
| `authorization_grant_profiles_supported` | Supported identity assertion grant profile identifiers | IETF | This document |
| `subject_identifier_formats_supported` | Supported Subject Identifier formats | IETF | This document |
| `issuer_trust_methods` | Trust Method requirements enforced for incoming identity assertions | IETF | This document |
| `signed_policy` | Signed JWT containing policy members as claims | IETF | This document |
| `crit` | Names decision-affecting members a consumer MUST understand or reject the document | IETF | This document |

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

Designated Expert instructions: the expert verifies that the format
has a well-defined namespace authority, that the extraction procedure
is deterministic (two consumers compute the same Subject Authority
from the same Subject Identifier), and that it specifies exact-match
comparison semantics per {{subject-authority-determination}}.

Future specifications that define new Subject Identifier formats are
expected to register additional entries here when those identifiers
have a well-defined namespace authority.

Initial entries:

| Subject Identifier Format | Subject Authority Form | Extraction Procedure |
|-|-|-|
| `email` | DNS domain | {{subject-authority-determination}} of this document |

## Media Type Registrations

IANA is requested to register the following media types in the "Media
Types" registry, used as the JWT `typ` header values for the signed
document forms ({{signed-policy-metadata}}).

For `application/trust-policy+jwt`: Type name `application`; Subtype
name `trust-policy+jwt`; Required parameters none; Optional parameters
none; Encoding considerations binary (JWS Compact Serialization);
Security considerations {{signed-policy-metadata}} and the Security
Considerations of this document; Interoperability considerations none;
Published specification this document; Applications OAuth Resource
Authorization Servers; Fragment identifier considerations none; Change
controller IETF.

For `application/issuer-authorization-policy+jwt`: as above, with
Subtype name `issuer-authorization-policy+jwt` and Applications OAuth
Subject Authorities and Resource Authorization Servers.

--- back

# Design Rationale

This appendix is non-normative. It records the design choices that
shaped the framework, for reviewers and implementers who want to
understand why specific decisions were made.

## Relationship to OpenID Federation {#relationship-to-oidf}

OpenID Federation {{OIDF-FEDERATION}} and this framework address
distinct questions: federation answers "is this issuer an
authentic ecosystem member?" via trust chains; this framework
adds "is this issuer authorized for *this* namespace?" via Subject
Authority publication. Federation membership feeds the
`issuer_authentication` category through the `openid_federation`
Trust Method ({{trust-method-openid-federation}}); namespace
authority comes from the orthogonal
`subject_namespace_authorization` category. This document does
not duplicate or replace federation mechanisms; it composes with
them.

## Why Bounded-Depth-1 Namespace Authorization

The `subject_namespace_authorization` category is bounded at
depth one: the Subject Authority lists each authorized Assertion
Issuer directly, with no provision for "and whoever issuer X
federates further." This is deliberate:

- A Validator never has to compute a multi-hop authorization
  chain to evaluate a namespace claim; a single customer-published
  policy names the Assertion Issuer.
- Revocation latency is bounded by the Subject Authority's own
  cache lifetime, not by the depth of a delegation chain.
- Compromise at any intermediate party (a federation operator, a
  delegated issuer) cannot expand the namespace authorization of
  any issuer the Subject Authority did not list directly.

Federation chains for issuer authentication remain in scope under
the `issuer_authentication` category; only the
namespace-authorization graph is bounded.

# Future Extensions {#future-extensions}

This appendix is non-normative. It sketches directions intentionally
deferred from this document; future specifications may register them.

## Additional Subject Identifier Formats

This document registers one Subject Authority extraction procedure
(`email`, {{subject-authority-determination}}). The policy model
accommodates additional Subject Identifier formats. A future
specification adding a new format would register a Subject Authority
Extraction Procedure in {{iana-authority-registry}} defining how a
Subject Authority is computed from values of the new format, and
either reuse `domain_authorized_issuer` (when the computed Subject
Authority is a DNS-publishable domain) or register a new
`subject_namespace_authorization` Trust Method. The Trust Policy
document format, Issuer Authorization Policy document format, Trust
Method category structure, combination rule, and bounded-transitivity
property remain unchanged across such extensions.

Candidate formats considered as future work include URL-host
Subject Identifiers (`url_host`), Decentralized Identifiers (`did`),
and a subdomain-exact email variant that requires an explicit
delegation from the registrable-domain authority to prevent
subdomain takeover.

## Critical Directives for the DNS Record Form

The JSON document forms carry a `crit` member defined in the base
specification ({{critical-members}}), so an extension that adds a
decision-affecting member to the Trust Policy or Issuer Authorization
Policy can mark it critical and have already-deployed consumers honor
it. The DNS record form has no analogous per-directive criticality
mechanism today; it relies on the version token ({{DAI}}) to fence off
incompatible future syntax wholesale. A future extension that needs
per-directive fail-closed semantics in the DNS form (rather than a new
version token) would define a `crit=` directive and its recognition
rules at that time; it is deferred because the version token already
provides a fail-closed path for the DNS form.

## Actor Identity Trust Evaluation

OAuth assertions can carry an `act` object expressing actor
delegation (a service acting on behalf of a user). A future
extension can apply the trust-evaluation categories of
{{delegation-model}} to actor identities: a Resource Authorization
Server would evaluate, in addition to the assertion's `iss` and
`sub`, whether the asserting issuer is entitled to attest the
`(act.iss, act.sub)` pair. This requires registering a Subject
Authority extraction procedure applicable to actor-carried
Subject Identifiers and is therefore tied to the OAuth Actor
Profile being progressed separately. Until that extension lands,
trust evaluation of actor identities is governed by local policy
at the Resource Authorization Server.

## Trust Policy Discovery

This document specifies how a Resource Authorization Server
publishes its Trust Policy (via authorization server metadata or
protected resource metadata) but assumes a client or peer that
already knows the Resource Authorization Server's identity. In
open-world deployments (agent runtimes, AI tools, cross-organization
integrations), a peer may need to discover a Resource Owner's
Trust Policy before any prior bilateral relationship exists.

A future Trust Policy Discovery extension can let a Resource
Owner publish a DNS-named pointer at
`_oauth-trust-policy.{resource-domain}` to its Trust Policy
document, mirroring the DNS-authority pattern of {{DAI}} for the
resource-side. The Resource Owner's domain becomes the
publication channel for "where do I trust assertions from?", the
dual of DAI's "who do I authorize to assert about me?".

The extension is deferred because the open-world first-contact
case is not yet a deployed need for the namespace and federation
profiles defined here; bilateral configuration plus the metadata
endpoints suffice for the current target deployments.

# Frequently Asked Questions

This appendix is non-normative.

**Q: I have OpenID Federation. Why do I need DAI?**

Federation answers "is this issuer authentically a member of an
ecosystem?". It does not answer "is this issuer authorized to
assert about subjects in *my* namespace?". A federation member can
mint an identity assertion naming any email domain; federation
membership doesn't constrain which namespace. DAI lets the
namespace owner publish that constraint. {{example-federation-dai}}
shows the two together.

**Q: What's the difference between Trust Policy and DAI?**

Trust Policy is what a **Resource Authorization Server** publishes
to declare what evidence it requires of an Assertion Issuer
(metadata at `/.well-known/identity-assertion-trust-policy`). DAI
is what a **Subject Authority** publishes to declare which
Assertion Issuers it authorizes for its namespace (records at
`_oauth-issuer-policy.{domain}` and the corresponding HTTPS
well-known URL). RAS-published vs Subject-Authority-published.

**Q: Why two independent trust categories?**

To stop "federation member" from being silently treated as
"authoritative for any namespace." Each category answers a
different question and the cross-category combination rule
({{combination-rule}}) requires evidence from both when both are
configured. Conflating them is the bug
({{applicability-bypass}}, {{unverified-claim}}).

**Q: What if my Subject Authority cannot publish DNS TXT records?**

Use DAI's HTTPS-only lookup mode ({{DAI}} §HTTPS-Only Deployment
Variant): the Resource Authorization Server retrieves the policy
from `https://{authority}/.well-known/oauth-issuer-policy` directly,
skipping the `_oauth-issuer-policy` TXT lookup. A Subject Authority
with no DNS-named authority at all cannot participate in DAI.

**Q: Does this work for path-bearing issuer identifiers
(`https://login.example.com/{tenant}/v2.0`)?**

Yes. `domain_authorized_issuer` uses case-sensitive URL string
comparison and accepts any absolute HTTPS issuer identifier
including paths.

**Q: How do I revoke a delegation?**

Remove the entry from the Issuer Authorization Policy or set
`valid_until` to the past. Revocation latency is bounded by cache
lifetime; see {{DAI}} §Caching.

# Agent Platform IdP Walkthrough {#example-agent-platform}

This appendix is non-normative. It walks through how the framework
prevents an unauthorized provider from impersonating users in a
customer's email namespace. Protection rests on a single deliberate
choice by the customer: publishing an Issuer Authorization Policy
that lists the specific agent platforms permitted to assert
identities about its users.

**Cast:** customer `example.com` (owns the email domain); agent
platform `https://agentprovider.example` (mints ID-JAGs after
federated SSO from the customer's primary IdP); tool provider
`https://toolprovider.example` (the Resource Authorization Server);
end user `alice@example.com`.

**Publication.** The customer publishes:

~~~
_oauth-issuer-policy.example.com. IN TXT ( "v=oauth-issuer-policy1;"
    "authority=example.com;"
    "issuer=https://agentprovider.example" )
~~~

The tool provider publishes a Trust Policy listing
`domain_authorized_issuer`. After Alice signs in to the agent
platform (the user-side SSO is out of scope), the agent platform
mints an ID-JAG:

~~~ json
{
  "iss": "https://agentprovider.example",
  "aud": "https://toolprovider.example",
  "exp": 1780166400, "iat": 1780166100, "jti": "b9c1...",
  "sub": "user-3f81a2",
  "email": "alice@example.com", "email_verified": true
}
~~~

**Verification.** The tool provider validates the ID-JAG per
{{ID-JAG}}, extracts the Subject Authority `example.com` from the
email claim, queries `_oauth-issuer-policy.example.com`, confirms
the `iss` value matches an `authorized_issuers` entry, and proceeds
with `private_key_jwt` client authentication and token issuance.

**What this protects against.** Suppose `attacker.example` mints
its own assertion claiming `email: alice@example.com,
email_verified: true`. The signature validates against the
attacker's own JWKS; the audience is correct; the email claim is
self-asserted. The tool provider extracts Subject Authority
`example.com`, looks up the customer's policy, and finds that
`https://attacker.example` is NOT in `authorized_issuers`. The
Trust Method fails; the tool provider rejects with `invalid_grant`.
The attacker's `email_verified: true` self-claim has no force;
trust derives from the `iss`-vs-policy check, not from the
assertion's own statements. `attacker.example` has no path to
impersonate users in `example.com` unless the customer publishes
them in DAI.

# OpenID Federation Walkthrough {#example-federation-walkthrough}

This appendix is non-normative. It expands the high-level
{{example-federation-dai}} into a concrete walkthrough including
trust-chain validation, Trust Mark satisfaction, and
federation-bound JWKS resolution.

**Cast:** Resource Authorization Server
`https://api.resource.example`; Federation Trust Anchor
`https://federation.example.org`; Federation Intermediate
`https://sector.example.org` (chained under the Trust Anchor);
Assertion Issuer `https://idp.partner.example` (federation leaf
holding a Level-of-Assurance-3 Trust Mark); end user
`alice@partner.example`; Subject Authority `partner.example`.

**Publication.** The Resource Authorization Server's Trust Policy:

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
    { "method": "domain_authorized_issuer" }
  ]
}
~~~

The Assertion Issuer's federation Entity Configuration (decoded,
illustrative) declares its authority hint and its Trust Mark:

~~~ json
{
  "iss": "https://idp.partner.example",
  "sub": "https://idp.partner.example",
  "authority_hints": ["https://sector.example.org"],
  "metadata": {
    "openid_provider": {
      "issuer": "https://idp.partner.example",
      "jwks_uri": "https://idp.partner.example/jwks"
    }
  },
  "trust_marks": [
    {
      "id": "https://federation.example.org/marks/loa3",
      "trust_mark": "eyJ...(JWT signed by federation.example.org)"
    }
  ]
}
~~~

The Federation Intermediate's Subordinate Statement about the leaf
constrains `issuer` and required auth methods via `metadata_policy`
({{OIDF-FEDERATION}} §6):

~~~ json
{
  "iss": "https://sector.example.org",
  "sub": "https://idp.partner.example",
  "metadata_policy": {
    "openid_provider": {
      "issuer": { "value": "https://idp.partner.example" },
      "jwks_uri": { "essential": true }
    }
  }
}
~~~

The Subject Authority publishes a DAI record:

~~~
_oauth-issuer-policy.partner.example. IN TXT ( "v=oauth-issuer-policy1;"
    "authority=partner.example;"
    "issuer=https://idp.partner.example" )
~~~

**Verification.** When an ID-JAG arrives with
`iss: https://idp.partner.example`, `email: alice@partner.example`,
the Resource Authorization Server:

1. **`issuer_authentication` (openid_federation).** Walks the
   federation chain per {{OIDF-FEDERATION}}: fetches the leaf
   Entity Configuration, the Intermediate's Subordinate Statement
   about the leaf, and the Trust Anchor's Subordinate Statement
   about the Intermediate. Validates all signatures, applies
   `metadata_policy`, and verifies the Trust Mark signature.

2. **Framework-specific checks ({{trust-method-openid-federation}}).**
   The terminal trust anchor matches `trust_anchors`; the
   policy-applied metadata declares entity type `openid_provider`;
   the `loa3` Trust Mark satisfies the requirement; the ID-JAG
   signing key is taken ONLY from the federation-resolved JWKS,
   not from the assertion `iss` URL's `.well-known/oauth-authorization-server`.

3. **`subject_namespace_authorization` (domain_authorized_issuer).**
   Extracts `partner.example` from the email claim, queries
   `_oauth-issuer-policy.partner.example`, confirms
   `https://idp.partner.example` is an authorized issuer.

4. **Cross-category combination rule.** Both categories succeed;
   the Resource Authorization Server issues an access token.

**Selected failure variants.** A chain not terminating at the
listed trust anchor → `invalid_grant`. A leaf without the required
Trust Mark → `invalid_grant`. A federation-resolved JWKS that
doesn't match the ID-JAG signing key → `invalid_grant` (a separate
JWKS at `.well-known/oauth-authorization-server` is NOT consulted,
preventing AS-metadata downgrade). `partner.example` not listing
the Assertion Issuer in DAI → `invalid_grant` even though
federation membership is valid.

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
