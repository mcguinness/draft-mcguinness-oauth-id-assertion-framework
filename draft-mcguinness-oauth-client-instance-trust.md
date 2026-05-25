---
title: "Client Instance Trust Method for OAuth Identity Assertion Issuer Trust Policy"
abbrev: "OAuth Client Instance Trust Method"
docname: draft-mcguinness-oauth-client-instance-trust-latest
category: std
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - trust policy
  - client instance
  - client identity
  - CIMD

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC8126:
  TRUST-FRAMEWORK:
    title: "OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-issuer-trust-policy/
    date: false
  CIA:
    title: "OAuth 2.0 Client Instance Assertions using Actor Tokens"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion/
    date: false

informative:
  RFC7521:
  RFC7523:
  RFC8693:
  RFC8705:
  RFC9449:
  CIMD:
    title: "OAuth Client Identifier Metadata Document"
    target: https://datatracker.ietf.org/doc/draft-parecki-oauth-client-id-metadata-document/
    date: false

---

--- abstract

The OAuth Identity Assertion Issuer Trust Policy {{TRUST-FRAMEWORK}}
defines a Trust Policy that a Resource Authorization Server publishes
to declare which Trust Methods it accepts for evaluating the issuer of
an identity assertion. It defines two Trust Method categories: one for
authenticating the assertion issuer, and one for verifying that the
issuer is authorized to assert about subjects in a particular
namespace.

A separate trust question arises at the same OAuth token endpoint:
when an OAuth client presents a Client Instance Assertion per
{{CIA}} binding a running workload to the client's identity, is the
workload's Instance Issuer authorized by the OAuth client to attest
that binding?

This document extends the trust framework to address this question.
It registers a third Trust Method category,
`client_instance_authorization`, and a Trust Method,
`client_authorized_instance_issuer`, that defers all wire-format
validation to {{CIA}}. A Resource Authorization Server lists this
method in its Trust Policy to advertise that it evaluates Client
Instance Assertions; the cross-category combination rule of
{{TRUST-FRAMEWORK}} composes the new category with the existing
two.

--- middle

# Introduction

The OAuth Identity Assertion Issuer Trust Policy {{TRUST-FRAMEWORK}}
gives an OAuth 2.0 {{RFC6749}} Resource Authorization Server a way to
declare its trust criteria for identity assertions. The Trust Policy lists Trust
Methods in categories; each category answers a distinct trust
question about the assertion issuer. The framework defines two
categories:

- `issuer_authentication`: is the entity that signed the assertion
  authentic?
- `subject_namespace_authorization`: is that entity authorized by the
  namespace owner to assert about subjects in this namespace?

A token request at the same endpoint may also carry a Client Instance
Assertion per {{CIA}}: a JWT, presented in the request, that binds a
running workload to an OAuth client's identity by attesting that the
workload is an authorized instance of the OAuth client. {{CIA}}
defines the wire format, the validation procedure, and the trust
root (the OAuth client's `instance_issuers` list published in CIMD
{{CIMD}} or in local registration).

What {{CIA}} does not define is how a Resource Authorization Server
declares, via its Trust Policy, that it accepts Client Instance
Assertions. Today an authorization server either implements {{CIA}}
or it does not; clients have no machine-readable way to discover
which authorization servers expect the assertion or to know that the
RAS will evaluate it against {{CIA}}'s rules.

This document fills that gap. It extends the trust framework's
category registry with `client_instance_authorization` and
registers `client_authorized_instance_issuer` as a Trust Method in
that category. The Trust Method defers all wire-format validation to
{{CIA}}; this document contributes only the trust-policy framing.

A Resource Authorization Server that accepts Client Instance
Assertions lists `client_authorized_instance_issuer` in its Trust
Policy. The cross-category combination rule of {{TRUST-FRAMEWORK}}
ensures that client-instance evaluation runs alongside any required
issuer-authentication and subject-namespace-authorization checks.

## Relationship to Other Documents

This document depends normatively on:

- **{{TRUST-FRAMEWORK}}**: the parent framework defining Trust Policy,
  Trust Method categories, the cross-category combination rule, and
  the IANA Trust Methods registry that this document extends.
- **{{CIA}}**: the wire-format and validation specification for
  Client Instance Assertions. This document defers all wire-format
  details to {{CIA}}.

This document does not define:

- The wire format of a Client Instance Assertion. See {{CIA}}.
- The mechanism by which an OAuth client publishes its
  `instance_issuers` list. See {{CIA}} and {{CIMD}}.
- The OAuth client authentication mechanism. See {{RFC7521}},
  {{RFC7523}}, {{RFC8705}}, or {{CIA}} for client-instance-bound
  authentication.

This document does define:

- The `client_instance_authorization` Trust Method category, as a
  registered extension to {{TRUST-FRAMEWORK}}.
- The `client_authorized_instance_issuer` Trust Method, in that
  category, deferring to {{CIA}}.
- The optional `client_instance_assertion_required` Trust Policy
  member, signaling that a Trust Policy requires every applicable
  token request to carry a Client Instance Assertion.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

This document uses the terminology of {{TRUST-FRAMEWORK}} and
{{CIA}}. In addition:

Instance Issuer:
: The entity identified by the `iss` claim of a Client Instance
  Assertion. Per {{CIA}}, the Instance Issuer is an entity (such as
  a workload identity authority, e.g., a SPIRE deployment) that
  attests the binding between a running workload and an OAuth
  client. The OAuth client authorizes specific Instance Issuers by
  listing them in its `instance_issuers` metadata per {{CIA}}.

# The client_instance_authorization Category {#category}

This document registers `client_instance_authorization` as a Trust
Method category in the IANA "Identity Assertion Issuer Trust
Methods" registry established by {{TRUST-FRAMEWORK}}.

Category definition:

`client_instance_authorization`
: Does the Instance Issuer of a Client Instance Assertion have the
  OAuth client's permission to attest concrete runtime instances of
  that client? An attestation from the OAuth client's metadata (a
  CIMD document or local registration record) answers this
  question.

Applicability:

This category applies to a token request when the request includes a
Client Instance Assertion per {{CIA}}. The category MAY also apply
when the Trust Policy includes the `client_instance_assertion_required`
member set to `true` ({{policy-extension}}), in which case the
applicable request MUST include a Client Instance Assertion and a
request lacking one MUST be rejected.

When the category applies and the OAuth client's metadata cannot be
resolved per {{CIA}} (CIMD dereference or local registration
lookup), the Resource Authorization Server MUST reject the request.
The Resource Authorization Server MUST NOT treat the category as
inapplicable in this case; the failure to resolve the metadata is
itself a verification failure.

Evaluation subject:

The Trust Method in this category evaluates the Instance Issuer (the
`iss` claim of the Client Instance Assertion), not the Assertion
Issuer of the primary identity assertion. The `iss` of the primary
assertion continues to be evaluated under the
`issuer_authentication` and `subject_namespace_authorization`
categories of {{TRUST-FRAMEWORK}}.

# The client_authorized_instance_issuer Trust Method {#trust-method}

This document registers `client_authorized_instance_issuer` as the
initial Trust Method in the `client_instance_authorization`
category.

The `client_authorized_instance_issuer` method indicates that the
Instance Issuer of a Client Instance Assertion is acceptable if it
is authorized in the OAuth client's metadata per {{CIA}}.

~~~ json
{
  "method": "client_authorized_instance_issuer"
}
~~~

This Trust Method has no parameters.

## Evaluation

When the Resource Authorization Server evaluates this Trust Method,
it MUST:

1. Resolve the OAuth client's metadata per {{CIA}} (CIMD dereference
   when the `client_id` is a URL; local registration lookup
   otherwise). If the metadata cannot be retrieved or is malformed,
   reject the request.

2. Apply the Client Instance Assertion validation procedure defined
   in {{CIA}}, including:

   - matching the assertion's `iss` claim against an entry in the
     client's `instance_issuers` list;
   - verifying the assertion's `client_id` claim equals the
     authenticated OAuth client's `client_id`;
   - validating the assertion's signature against the issuer
     descriptor's configured key source;
   - verifying descriptor-level constraints (subject syntax, trust
     domain, signing algorithm, key source) match the assertion;
   - verifying proof of possession of any confirmation key.

3. If the {{CIA}} validation procedure succeeds, this Trust Method
   is satisfied. If validation fails, this Trust Method is not
   satisfied; per {{TRUST-FRAMEWORK}} §rasp, the Resource
   Authorization Server MUST reject the request.

The authority source for this Trust Method is the OAuth client
itself. The OAuth client's metadata is the publication channel.
{{CIA}} defines the document format, descriptor shape, presentation
paths (request parameter or actor token), and validation procedure;
this document contributes only the trust-policy framing.

## Scope of This Trust Method

This Trust Method authenticates the binding between an OAuth client
and a running workload; it does NOT authenticate the Instance Issuer
as a member of a federation, an entity in an ecosystem registry, or
the holder of any out-of-band identity beyond what the OAuth client's
metadata names.

A deployment that requires both client-instance authorization AND
federation membership for the Instance Issuer lists both
`client_authorized_instance_issuer` (in this category) and a
federation-based method such as `openid_federation` (in the
`issuer_authentication` category) in its Trust Policy. The
cross-category combination rule of {{TRUST-FRAMEWORK}} ensures both
are evaluated.

# Trust Policy Extension {#policy-extension}

This document defines an optional Trust Policy member.

`client_instance_assertion_required`
: OPTIONAL. Boolean. When `true`, token requests evaluated under this
Trust Policy MUST include a Client Instance Assertion per {{CIA}} and
MUST satisfy the `client_instance_authorization` Trust Method
category. If this member is `true` and the Trust Policy contains no
recognized and well-formed Trust Method in the
`client_instance_authorization` category, the policy is malformed and
the Resource Authorization Server MUST reject the request. When
absent or `false`, this Trust Policy does not by itself require a
Client Instance Assertion; a Client Instance Assertion that is
present is still evaluated when the policy contains an applicable
`client_instance_authorization` method.

A Trust Policy publisher using this member SHOULD list it in the
Trust Policy's `crit` member ({{TRUST-FRAMEWORK}}
§critical-extension-identifiers) so that consumers that do not
recognize the member reject the policy rather than silently
ignoring the requirement.

Example:

~~~ json
{
  "resource_authorization_server": "https://auth.example.org",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer"
    },
    {
      "method": "client_authorized_instance_issuer"
    }
  ],
  "client_instance_assertion_required": true,
  "crit": ["client_instance_assertion_required"]
}
~~~

This Trust Policy requires every token request to carry a Client
Instance Assertion AND requires the Assertion Issuer of the primary
assertion to satisfy both federation membership and namespace
authority.

# Combined Evaluation {#combined-evaluation}

When this Trust Method category is present alongside the categories
defined in {{TRUST-FRAMEWORK}}, the cross-category combination rule
of {{TRUST-FRAMEWORK}} §rasp applies unchanged. The rule combines
categories with AND-semantics across categories and OR-semantics
within a category.

For a token request evaluated under a Trust Policy that includes all
three categories:

- The Assertion Issuer (the `iss` of the primary identity assertion)
  MUST satisfy at least one `issuer_authentication` method AND at
  least one `subject_namespace_authorization` method.
- The Instance Issuer (the `iss` of the Client Instance Assertion)
  MUST satisfy `client_authorized_instance_issuer` (or another
  registered method in the same category).

The three categories evaluate two different entities. The first two
evaluate the Assertion Issuer of the primary assertion. The third
evaluates the Instance Issuer of a Client Instance Assertion carried
in the same token request. The Trust Policy's flat list of methods
notwithstanding, these evaluations are independent in subject as
well as in security question.

This separation prevents three classes of compromise:

- An Assertion Issuer authenticated as a federation member but not
  authorized for a subject namespace cannot impersonate users in
  that namespace.
- An Instance Issuer authorized for one OAuth client cannot attest
  runtime workloads as instances of any other OAuth client.
- An OAuth client cannot transitively confer Assertion Issuer
  authority on its authorized Instance Issuers; the two trust
  decisions remain separate.

# Worked Example (Non-Normative)

This appendix is non-normative.

This example shows a token request that exercises all three Trust
Method categories: federation membership for the Assertion Issuer,
namespace authority for the asserted subject, and client-instance
authorization for the Instance Issuer.

## Cast

- **Assertion Issuer**, `https://idp.partner.example`. Authenticates
  Alice and mints an ID-JAG. Federated member of
  `https://federation.example.org`. Also listed in
  `partner.example`'s Issuer Authorization Policy.
- **Workload (OAuth Client)**, `https://app.example.com/oauth/client-metadata.json`.
  Presents tokens at the Resource Authorization Server.
- **Workload Identity (Instance Issuer)**,
  `spiffe://prod.example.com`. The SPIFFE trust domain whose SVID
  authority issues the Client Instance Assertion.
- **Resource Authorization Server**, `https://auth.example.org`.
  Publishes the Trust Policy below.

## Trust Policy

~~~ json
{
  "resource_authorization_server": "https://auth.example.org",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods_supported": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer"
    },
    {
      "method": "client_authorized_instance_issuer"
    }
  ],
  "client_instance_assertion_required": true,
  "crit": ["client_instance_assertion_required"]
}
~~~

## Token Request

The Workload presents an ID-JAG (for Alice) AND a Client Instance
Assertion (binding the Workload to the OAuth client). For
illustration, the request uses Token Exchange {{RFC8693}}, with the
Client Instance Assertion carried in the `actor_token` parameter per
{{CIA}}:

~~~ http
POST /token HTTP/1.1
Host: auth.example.org
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&client_id=https%3A%2F%2Fapp.example.com%2Foauth%2Fclient-metadata.json
&subject_token=eyJ...(ID-JAG)
&subject_token_type=urn:ietf:params:oauth:token-type:id-jag
&actor_token=eyJ...(Client Instance Assertion)
&actor_token_type=urn:ietf:params:oauth:token-type:client-instance-jwt
~~~

## Verification

The Resource Authorization Server applies the {{TRUST-FRAMEWORK}}
§rasp procedure with the three-category combination rule. For each
applicable category:

1. **`issuer_authentication` (Assertion Issuer = ID-JAG `iss`)**:
   the Resource Authorization Server validates the ID-JAG's
   federation trust chain to `https://federation.example.org`. The
   chain validates. Method satisfied.

2. **`subject_namespace_authorization` (Assertion Issuer for Alice's
   email domain)**: the Resource Authorization Server extracts
   `partner.example` from the ID-JAG's `email` claim and retrieves
   the Issuer Authorization Policy at
   `_oauth-issuer-policy.partner.example`. The policy lists
   `https://idp.partner.example` in `authorized_issuers`. Method
   satisfied.

3. **`client_instance_authorization` (Instance Issuer = actor token
   `iss`)**: the Resource Authorization Server fetches the CIMD
   document at the `client_id` URL and validates the Client Instance
   Assertion per {{CIA}}. The Instance Issuer is listed in the
   client's `instance_issuers`. Method satisfied.

All three categories satisfied; the token request proceeds.

## Threat: Compromised Instance Issuer Outside the Client's List

Suppose an attacker controls an Instance Issuer
`spiffe://attacker.example` and obtains a SPIFFE SVID for a
workload in that trust domain. The attacker constructs a Client
Instance Assertion claiming the OAuth client and presents it.

The verification proceeds as follows: the Resource Authorization
Server fetches the CIMD document, finds the
`instance_issuers` list, and looks up `spiffe://attacker.example`
in that list. It is not present. The
`client_authorized_instance_issuer` Trust Method is not satisfied;
the request is rejected.

The attacker's correct authentication of its own workload identity
does not authorize it to act as a client whose owner did not list
the attacker's Instance Issuer in `instance_issuers`. Federation
membership and namespace authority do not extend to this question;
only the OAuth client owner's metadata decides.

# Security Considerations

## Independence of Trust Evaluations

The three Trust Method categories evaluated under a combined Trust
Policy answer three independent questions about two different
entities (the Assertion Issuer and the Instance Issuer). Satisfying
one category does not imply satisfying another. Operators
constructing Trust Policies SHOULD list methods from every category
they consider necessary for the deployments they accept; omitting a
category leaves that question unanswered for the request.

This separation is the load-bearing security property of the
combined framework. Conflating evaluations would allow:

- A federated Assertion Issuer to assert about subjects in any
  namespace.
- An Instance Issuer authorized by one OAuth client to be treated
  as authorized for any OAuth client.
- An OAuth client to confer Assertion Issuer authority on its
  Instance Issuers, or vice versa.

The trust framework's cross-category combination rule prevents each
of these by requiring evidence from every applicable category
independently.

## Trust Source Differences

This Trust Method's authority source (the OAuth client's metadata)
is fundamentally different from the authority sources of methods in
the other categories:

- `issuer_authentication` methods (e.g., `openid_federation`) trust
  ecosystem operators (federation trust anchors).
- `subject_namespace_authorization` methods (e.g.,
  `domain_authorized_issuer`) trust namespace owners (domain
  controllers).
- `client_instance_authorization` methods (this category) trust
  OAuth client owners (CIMD publishers or local-registration
  configurers).

These three trust roots are typically different parties: a
federation operator, a customer's domain owner, and a SaaS vendor
operating a CIMD. Combining their decisions into one Trust Policy
correctly composes their authorities without conflating them.

## Wire-Format Stability

This document depends normatively on {{CIA}} for the Client Instance
Assertion wire format and validation procedure. {{CIA}} is an
individual Internet-Draft at the time of this writing. Changes to
{{CIA}}'s wire format affect this Trust Method's evaluation; advancement
of this document to Standards Track is bounded by {{CIA}}'s progress
and stability. Implementations SHOULD track {{CIA}} for wire-format
changes.

This document does not freeze any wire-format details of {{CIA}};
the Trust Method is satisfied when the {{CIA}} validation procedure
succeeds, regardless of which {{CIA}} presentation path or version
the request uses.

## Token Sender-Constraining

This Trust Method binds Instance Issuer authorization at the point
of token issuance. Once the Resource Authorization Server issues an
access token, the token's relationship to the workload is governed
by sender-constraining mechanisms ({{RFC8705}}, {{RFC9449}}) and
not by this Trust Method. Resource Authorization Servers issuing
tokens to workload-authenticated clients SHOULD sender-constrain
those tokens; this Trust Method alone does not provide end-to-end
runtime binding.

# Privacy Considerations

The Trust Policy that lists `client_authorized_instance_issuer`
discloses that the Resource Authorization Server evaluates Client
Instance Assertions. This is operationally useful (clients know what
the RAS expects) but reveals nothing about the specific Instance
Issuers the RAS accepts; that information is in each OAuth client's
metadata, not in the Trust Policy.

The `client_instance_assertion_required` Trust Policy member
discloses that every applicable token request must carry a Client
Instance Assertion. A Trust Policy that uses this member is signaling
a workload-centric deployment posture, which itself may be
deployment-sensitive information.

For privacy considerations applicable to the wire format of Client
Instance Assertions and the resolution of CIMD documents, see
{{CIA}} and {{CIMD}}.

# IANA Considerations

## Identity Assertion Issuer Trust Method Category Registration

IANA is requested to add the following entry to the "Identity
Assertion Issuer Trust Method Categories" registry established by
{{TRUST-FRAMEWORK}}. Registration policy: Specification Required
{{RFC8126}}.

Category Identifier:
: `client_instance_authorization`

Description:
: Does the Instance Issuer of a Client Instance Assertion have the
  OAuth client's permission to attest concrete runtime instances of
  that client?

Applicability:
: Applies to a token request when the request includes a Client
  Instance Assertion per {{CIA}}, or when the applicable Trust
  Policy sets `client_instance_assertion_required` to `true`.

Evaluation Subject:
: The Instance Issuer identified by the `iss` claim of the Client
  Instance Assertion.

Reference:
: This document.

## Identity Assertion Issuer Trust Method Registration

IANA is requested to add the following entry to the "Identity
Assertion Issuer Trust Methods" registry established by
{{TRUST-FRAMEWORK}}.

Identifier:
: `client_authorized_instance_issuer`

Categories:
: `client_instance_authorization`

Parameters:
: (none)

Reference:
: This document, {{CIA}}

## Trust Policy Member Registration

IANA is requested to add the following entry to the Trust Policy
member registry established by {{TRUST-FRAMEWORK}} (if such a
registry exists at the time of publication; otherwise, IANA is
requested to publish the registration alongside the trust policy
member registrations of {{TRUST-FRAMEWORK}}).

Member Name:
: `client_instance_assertion_required`

Member Description:
: Boolean. When `true`, token requests evaluated under this Trust
  Policy MUST include a Client Instance Assertion per {{CIA}}.

Change Controller:
: IETF

Specification Document:
: This document

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
