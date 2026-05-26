---
title: "Attribute Authority Trust Method for OAuth Identity Assertion Issuer Trust Policy"
abbrev: "OAuth Attribute Authority Trust"
docname: draft-mcguinness-oauth-attribute-authority-trust-latest
category: info
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - trust policy
  - attribute attestation
  - claim authority
  - credentials

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
  RFC7519:
  RFC8126:
  RFC8552:
  RFC8553:
  RFC8615:
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
  RFC7515:
  RFC7521:
  RFC8659:
  RFC9493:
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  PSL:
    title: "Public Suffix List"
    target: https://publicsuffix.org/
  W3C-VC:
    title: "Verifiable Credentials Data Model 2.0"
    target: https://www.w3.org/TR/vc-data-model-2.0/
  OIDC4IDA:
    title: "OpenID Connect for Identity Assurance 1.0"
    target: https://openid.net/specs/openid-connect-4-identity-assurance-1_0.html
  IDA-VERIFIED-CLAIMS:
    title: "OpenID Identity Assurance Specifications (eKYC-IDA Working Group)"
    target: https://openid.net/wg/ekyc-ida/specifications/
  OIDC-CORE:
    title: "OpenID Connect Core 1.0"
    target: https://openid.net/specs/openid-connect-core-1_0.html

---

--- abstract

The OAuth Identity Assertion Issuer Trust Policy {{TRUST-POLICY}}
defines a Trust Policy a Resource Authorization Server publishes to
declare which Trust Methods it accepts for evaluating the issuer of
an identity assertion. It defines two Trust Method categories:
`issuer_authentication` and `subject_namespace_authorization`.

A distinct trust question arises when an identity assertion carries
attribute claims (degree status, employment status, KYC level,
professional certification) whose authority is not the subject's
namespace owner: is the asserting issuer authorized by the relevant
attribute authority to attest that claim?

This document extends the Trust Policy framework to address this
question. It registers an additional Trust Method category,
`attribute_attestation`, and a Trust Method,
`attribute_authority_authorized_issuer`, that lets an attribute
authority (a university, an employer, a KYC provider, a
credentialing body, or any other claim-type authority) publish the
set of Assertion Issuers it authorizes to attest its claim types.
The policy format uses a DNS+HTTPS publication pattern aligned with
{{DAI}} at a different owner name, accommodating any attribute
authority whose identity can be expressed as a DNS-named domain. This
document authorizes issuers to attest claim types; it does not prove
that a specific Subject actually has a specific attribute value.

--- middle

# Introduction

## Document Status

This document is Informational. The Authority Delegation Pattern,
Trust Policy framework, and `subject_namespace_authorization`
category in {{AUTHORITY-DELEGATION}}, {{TRUST-POLICY}}, and {{DAI}}
constitute the Standards-Track core of the family and address the
two primary trust questions (issuer authentication and subject
namespace authorization). The `attribute_attestation` category
defined here addresses a third trust question that arises in
specific deployments (verified-claims carriage, professional
credentialing). It is published Informational pending validated
deployment demand. The category, its wire format, and IANA
registrations are stable enough to deploy, but the document is not
yet on a Standards-Track timeline. Implementers MAY deploy this
profile; deployments inform future Standards-Track progression.

## The Third Trust Question

The OAuth Identity Assertion Issuer Trust Policy {{TRUST-POLICY}}
keeps issuer authentication and subject namespace authorization as
distinct trust questions. Federation membership does not establish
namespace authority; namespace authority does not establish
federation membership. Each question is answered by evidence from
its own category of Trust Method.

Identity assertions in real deployments often carry more than
identity. A university-affiliated assertion may carry a degree
status. An employer-issued assertion may carry an employment-status
claim. A KYC provider may attest verification level. Each of these
attribute claims has its own AUTHORITY: the entity legitimately
positioned to attest the claim. That authority is typically NOT the
subject's namespace owner.

Consider three concrete cases:

- A university `university.example` confers a degree on Alice
  (`alice@personal.example`). Years later, Alice presents an
  identity assertion carrying `degree_status: graduated`. The
  Resource Authorization Server cannot verify this attribute by
  consulting `personal.example` (which has no opinion on Alice's
  degree status) nor by consulting the asserting issuer's federation
  membership (which says nothing about which authority backs the
  degree claim). The authority is `university.example`, which is
  distinct from both the subject's namespace and the asserting
  issuer's federation.

- An employer `employer.example` employs Bob. Bob presents an
  assertion to a vendor's Resource Authorization Server carrying
  `employment_status: active` and `employer: employer.example`.
  Verification follows the same pattern: the employment claim's
  authority is `employer.example`, not Bob's email namespace.

- A KYC provider `kyc.example` attests Carol's identity verification
  level. The attestation can be carried in any assertion (regardless
  of namespace or federation context) but its authority is
  `kyc.example`.

In each case the Resource Authorization Server needs a wire-format
mechanism to discover which Assertion Issuers a given attribute
authority has authorized to attest its claim types. This document
defines that mechanism.

## Relationship to the OAuth Profile Family

This document extends the OAuth profile family of
{{AUTHORITY-DELEGATION}} with an additional Trust Method category. The
family is:

- **{{AUTHORITY-DELEGATION}}**: the abstract Authority Delegation
  Pattern (Authority Holder, Delegate, Delegation Artifact,
  Validator; cross-category combination rule; profile registry).
- **{{TRUST-POLICY}}**: the OAuth profile defining the Trust Policy
  document, Trust Method machinery, Subject Authority
  Determination, and two initial categories
  (`issuer_authentication`, `subject_namespace_authorization`).
- **{{DAI}}**: a `subject_namespace_authorization` profile defining
  DNS+HTTPS Subject Authority publication.
- **This document**: registers an additional category,
  `attribute_attestation`, applying the same Authority Delegation
  Pattern to claim-type authority. The Authority Holder is the
  attribute authority (a university, employer, KYC provider, or
  credentialing body); the Delegate is the Assertion Issuer
  authorized to attest the claim type; the Delegation Artifact is
  the Attribute Authority Authorization Policy.

This document depends normatively on {{TRUST-POLICY}} (Trust Method
machinery, cross-category combination rule, IANA registries) and
on {{DAI}} for the DNS+HTTPS authority-publication pattern.

## Relationship to Verifiable Credentials

The W3C Verifiable Credentials Data Model {{W3C-VC}} defines a
three-party trust triangle (Issuer, Holder, Verifier) for
credential exchange. VCs and this document address related but
distinct concerns:

- VCs specify the wire format and signing model for credential
  payloads. They do NOT specify how a Verifier decides that the
  credential's Issuer is authorized to attest the credential's
  claims for a given Subject.

- This document specifies the OAuth-side trust evaluation for that
  authorization question, regardless of whether the underlying
  claim payload is a VC, an OAuth identity assertion, or some
  other JWT-borne attestation.

A deployment using both typically uses VCs for the credential
issuance layer and Attribute Authority Trust for the OAuth-layer
issuer-trust question when such credentials are presented as OAuth
identity assertions.

## Relationship to eKYC-IDA Verified Claims {#ekyc-ida}

The OpenID eKYC/IDA WG specifications {{IDA-VERIFIED-CLAIMS}} carry
verified identity claims with provenance metadata. OIDC4IDA
{{OIDC4IDA}} defines the `verified_claims` payload (a `verification`
object naming a `trust_framework` plus a `claims` payload).

OIDC4IDA owns the wire format, the evidence/verification-method
vocabulary, and the verification metadata schema. It does NOT define
how a trust framework's owner (eIDAS, NIST, GLEIF, a national scheme)
publishes which Issuers it has authorized, nor an OAuth-side trust
evaluation for the Verifier to discover whether an Issuer claiming
`"trust_framework": "eidas"` is authorized under eIDAS. OIDC4IDA's
trust model is implicit: the Verifier is assumed to know out of band.

This document closes that gap when the trust framework identifier maps
to a DNS-named Attribute Authority. The `verified_claims.verification.
trust_framework` value identifies the Attribute Authority (directly if
already DNS-named, or via the mapping in
{{attribute-authority-determination}}). The JWT's `iss` is the
Assertion Issuer; the Resource Authorization Server consults the
authority's AAAP per {{lookup}} to evaluate whether `iss` is
authorized. A worked example appears in {{ekyc-ida-example}}.

A deployment using OIDC4IDA typically uses it for the claim payload
and this document for OAuth-side issuer trust. {{claim-shape}} defines
both recognized claim shapes (flat-triple and OIDC4IDA wrapper).

## Relationship to OIDC Distributed Claims

OpenID Connect Core ({{OIDC-CORE}} §5.6.2) defines distributed
claims: a mechanism by which a primary Issuer references claims
held by a separate "claims provider," with the Verifier
retrieving the referenced claims directly from the provider. The
claims-provider Issuer is structurally analogous to the Assertion
Issuer of this document, and its authority to attest the claims
maps onto the `attribute_attestation` category.

This document does not define a wire-format mechanism for the
distributed-claims reference itself; that is OIDC Core's domain.
A deployment using distributed claims to deliver verified
attribute attestations applies this document's trust evaluation
to the claims-provider Issuer, using the trust framework named
in the distributed claim's metadata as the Attribute Authority.

## Scope and Non-Goals

This document defines:

- The `attribute_attestation` Trust Method category, as a
  registered extension to {{TRUST-POLICY}}.
- The `attribute_authority_authorized_issuer` Trust Method, in that
  category.
- The Attribute Authority Authorization Policy document
  ({{aaap-document}}).
- A DNS+HTTPS publication mechanism for that policy
  ({{publication}}).

This document does not define:

- New attribute claim names. Attribute claims (such as
  `degree_status`, `employment_status`, `kyc_level`) are defined by
  other specifications or by deployment convention.
- The semantics of any particular attribute. This document
  specifies only the issuer-authorization layer.
- The OAuth grant profile carrying the attribute claims. See
  {{ID-JAG}} and related specifications.

This document does not address attribute authorities whose
identity cannot be expressed as a DNS-named domain (for example, a
purely registry-based credentialing body with no DNS presence).
Such deployments require an additional Trust Method whose
publication channel matches the authority form.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology

This document uses the terminology of {{TRUST-POLICY}} and
{{AUTHORITY-DELEGATION}}. In addition:

Attribute Authority:
: The Authority Holder for a claim type (a class of attribute
  claims). Examples: a university for degree-status claims; an
  employer for employment-status claims; a KYC provider for
  identity-verification-level claims; a credentialing body for
  professional certification claims. In this document, the
  Attribute Authority is identified as a DNS-named domain.

Attribute Claim:
: A claim in an OAuth identity assertion that asserts an attribute
  about the Subject. Examples: `degree_status`, `employment_status`,
  `kyc_level`, `professional_certification`. The specific claim
  names and value spaces are defined by other specifications.

Attribute Authority Authorization Policy (AAAP):
: The JSON document published by an Attribute Authority that
  authorizes Assertion Issuers to attest its claim types. Defined
  in {{aaap-document}}.

# Trust Policy Extension {#policy-extension}

This document defines an optional Trust Policy member.

`attribute_attestation_claims`
: OPTIONAL. JSON array of Attribute Type Identifiers ({{attribute-type-identifiers}}).
  Each value names an attribute claim that the Resource Authorization
  Server treats as requiring `attribute_attestation` evaluation.

When this member is present, every listed attribute that appears in an
identity assertion MUST be evaluated under the `attribute_attestation`
category. If a listed attribute appears without the authority metadata
required by one of the recognized claim shapes in {{claim-shape}}, the
Resource Authorization Server MUST reject the assertion with an OAuth
`invalid_grant` error.

When this member is absent, applicability is determined by local policy
and by the presence of authority metadata in the assertion. A Resource
Authorization Server MUST NOT accept a protected attribute claim merely
because the issuer omitted authority metadata.

# The attribute_attestation Category {#category}

This document registers `attribute_attestation` as a Trust Method
category in the IANA "Identity Assertion Issuer Trust Method
Categories" registry established by {{TRUST-POLICY}}.

Category definition:

`attribute_attestation`
: Has the Attribute Authority for an asserted attribute claim
  authorized the Assertion Issuer to attest that claim? An
  attestation from the Attribute Authority's published policy
  answers this question.

Applicability:

This category applies to a token request when the identity
assertion carries one or more protected attribute claims. A protected
attribute claim is any claim listed in the applicable Trust Policy's
`attribute_attestation_claims` member, any claim identified by local
Resource Authorization Server policy as requiring attribute-authority
verification, or any claim accompanied by Attribute Authority metadata
under one of the recognized claim shapes in {{claim-shape}}. The
applicability of the category is per-attribute: the Resource
Authorization Server evaluates the category once for each protected
attribute claim.

An assertion carrying no protected attribute claims does not trigger
applicability of this category, even when the Trust Policy lists
methods in this category. An assertion carrying a protected attribute
claim without the authority metadata required by {{claim-shape}} MUST be
rejected; the issuer cannot avoid evaluation by omitting authority
metadata.

Evaluation subject:

The Trust Method in this category evaluates the Assertion Issuer
(the `iss` claim of the identity assertion) against the policy
published by the Attribute Authority of the asserted attribute
claim. The same Assertion Issuer may be evaluated against multiple
Attribute Authorities when the assertion carries multiple
attribute claims; each evaluation is independent.

# Attribute Claim Shape and Authority Determination {#claim-shape}

## Attribute Type Identifiers {#attribute-type-identifiers}

An Attribute Type Identifier names the attribute being attested for the
purposes of AAAP matching.

For the flat-triple shape, the Attribute Type Identifier is the
top-level JWT claim name `<attribute>`. The name MUST be an ASCII string
matching `^[A-Za-z][A-Za-z0-9_]*$`, MUST NOT end in `_authority` or
`_authority_verified`, and MUST NOT be a member defined by this
document, {{RFC7519}}, or the applicable OAuth grant profile unless that
profile explicitly marks the member as an attribute claim eligible for
attribute-authority verification.

For the OIDC4IDA shape, the Attribute Type Identifier is the immediate
member name under `verified_claims.claims`. Nested claim objects are not
matched by this document. A future specification MAY define JSON Pointer
or profile-specific attribute identifiers for nested claims.

For a Resource Authorization Server to evaluate
`attribute_attestation`, the assertion MUST express both the
attribute claim and the Attribute Authority backing it. This
document defines two recognized claim shapes; future specifications
MAY register additional shapes.

## Flat-Triple Shape (Default) {#claim-shape-flat}

The flat-triple shape mirrors the `email`/`email_verified`
convention of {{TRUST-POLICY}}:

| Claim | Type | Required | Meaning |
|-|-|-|-|
| `<attribute>` | profile-defined | yes (if attribute is asserted) | The attribute value |
| `<attribute>_authority` | string | yes (if attribute is asserted) | A DNS-named domain identifying the Attribute Authority for this claim |
| `<attribute>_authority_verified` | boolean | yes (if attribute is asserted) | MUST be `true` for this Trust Method to evaluate the claim |

The `<attribute>_authority_verified` claim indicates that the
Assertion Issuer has, by its own internal procedures, verified
the binding between the Subject and the asserted attribute under
the named authority. The Resource Authorization Server MUST reject
an attribute claim whose corresponding `_verified` claim is missing
or not `true`.

Example assertion using the flat-triple shape:

~~~ json
{
  "iss": "https://idp.degree-issuer.example",
  "sub": "alice-123",
  "aud": "https://api.relyingparty.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "email": "alice@personal.example",
  "email_verified": true,
  "degree_status": "graduated",
  "degree_status_authority": "university.example",
  "degree_status_authority_verified": true
}
~~~

## OIDC4IDA verified_claims Shape {#claim-shape-ida}

OIDC4IDA ({{OIDC4IDA}}) deployments use a `verified_claims`
wrapper carrying verification metadata alongside the claims
payload:

~~~ json
{
  "verified_claims": {
    "verification": {
      "trust_framework": "idp-assurance.example",
      "evidence": [
        {
          "type": "document",
          "method": "pipp"
        }
      ],
      "verification_process": "..."
    },
    "claims": {
      "given_name": "Alice",
      "birthdate": "1990-01-01"
    }
  }
}
~~~

When an assertion carries a `verified_claims` member, the
Resource Authorization Server evaluates each claim in
`verified_claims.claims` as an attribute claim with the Attribute
Authority taken from `verified_claims.verification.trust_framework`.
The presence of a well-formed `verified_claims.verification`
object substitutes for per-attribute `_verified` triples: the
verification metadata in the OIDC4IDA payload IS the issuer's
verification statement.

A single assertion MAY contain at most one `verified_claims`
member. If a future specification registers multiple-attestation
OIDC4IDA structures, that specification MUST extend this
processing rule.

When an assertion carries BOTH flat-triple claims and a
`verified_claims` member, the Resource Authorization Server
evaluates each independently. The Trust Method is satisfied only
when ALL attribute claims across both shapes are authorized by
their respective Attribute Authorities.

## Attribute Authority Determination {#attribute-authority-determination}

The Attribute Authority for each protected attribute claim is determined
from the assertion as follows:

- For flat-triple claims, from the `<attribute>_authority` claim.
- For OIDC4IDA `verified_claims`, from the
  `verified_claims.verification.trust_framework` claim.

In both cases the value MUST be a DNS-named domain. The Subject
Authority comparison rules of {{TRUST-POLICY}} §Subject Authority
Determination apply unchanged: the domain is normalized to A-label
form per {{RFC5891}} and compared case-insensitively against the
policy's `attribute_authority` member.

Unlike Subject Authority extraction, this document does not
normalize the authority value to the registrable domain. The
Attribute Authority is identified by the exact domain the
Assertion Issuer claims. A deployment that wants to authorize
subdomains independently registers the exact subdomain as the
attribute authority; the registrable-domain default is not imposed
here because attribute authorities (unlike email namespaces) often
correspond to specific operational entities at specific subdomains
(`alumni.university.example`, `hr.employer.example`).

For OIDC4IDA trust frameworks identified by short names rather
than DNS domains (for example, `"eidas"`, `"nist_800_63A_ial_1"`),
the deployment MUST establish a mapping between the trust-framework
identifier and a DNS-named Attribute Authority. The mapping is
out of scope for this document; deployments MAY use bilateral
configuration, a community-maintained registry, or any other
mechanism consistent with the deterministic source-selection
requirement of {{AUTHORITY-DELEGATION}} §Profile Mapping Matrix
Requirements.

Future specifications MAY register alternative attribute-authority
extractions for claim shapes not covered by the two shapes above
(for example, attribute claims that embed VCs whose `iss` is the
attribute authority directly).

# The attribute_authority_authorized_issuer Trust Method {#trust-method}

This document registers `attribute_authority_authorized_issuer` as
the initial Trust Method in the `attribute_attestation` category.

The method indicates that the Assertion Issuer is acceptable for a
given attribute claim if the Attribute Authority named in the
`<attribute>_authority` claim authorizes the Assertion Issuer to
attest that attribute, as published in the Attribute Authority
Authorization Policy ({{aaap-document}}).

~~~ json
{
  "method": "attribute_authority_authorized_issuer"
}
~~~

This Trust Method has no parameters.

## Evaluation

When the Resource Authorization Server evaluates this Trust Method
against a protected attribute claim in the identity assertion:

1. Determine the Attribute Type Identifier `T` from the recognized claim
   shape in {{attribute-type-identifiers}}.

2. Determine the Attribute Authority `A` for `T` from the recognized
   claim shape:

   - For the flat-triple shape, use the `<attribute>_authority` claim.
     If the corresponding `<attribute>_authority_verified` claim is
     absent or not `true`, the Trust Method is not satisfied and the
     Resource Authorization Server MUST reject the assertion.

   - For the OIDC4IDA shape, use
     `verified_claims.verification.trust_framework` after applying any
     deterministic trust-framework-to-domain mapping required by
     {{attribute-authority-determination}}. The presence of a
     well-formed `verified_claims.verification` object is the issuer's
     verification statement for claims in `verified_claims.claims`.

3. Retrieve the Attribute Authority Authorization Policy for `A`
   per the lookup procedure in {{lookup}}. Negative and
   Indeterminate outcomes ({{failures}}) MUST result in rejection.

4. Find an entry in `authorized_issuers` whose `issuer` matches the
   assertion's `iss` claim using case-sensitive URL string equality,
   AND whose `attribute_types` list includes `T`.

5. If the entry contains a `tenant` member, verify the assertion's
   `tenant` claim matches per {{TRUST-POLICY}} multi-tenant binding
   rules.

6. If the entry contains `subject_populations`, determine the
   assertion's primary Subject Identifier per the applicable grant
   profile, compute its Subject Authority using {{TRUST-POLICY}}
   §Subject Authority Determination, and verify that the computed
   authority matches at least one value in `subject_populations`. If the
   primary Subject Identifier has no registered extraction procedure, or
   no listed value matches, the Trust Method is not satisfied.

7. If the entry contains `valid_from`/`valid_until`, verify the
   current time is within the validity window.

8. If all checks succeed, the Trust Method is satisfied for this
   attribute. If the assertion carries multiple protected attribute
   claims, the Resource Authorization Server repeats steps 1-7 for each
   protected attribute claim independently.

On failure, the Resource Authorization Server MUST reject the
assertion with an OAuth `invalid_grant` error.

## Scope of This Trust Method

This Trust Method authorizes the Assertion Issuer to attest the
specific attribute claim; it does NOT:

- Authorize the Assertion Issuer to attest other attribute claims
  not listed in the matched `authorized_issuers` entry's
  `attribute_types`.
- Establish that the Assertion Issuer is authoritative for the
  Subject's namespace (that is a `subject_namespace_authorization`
  question).
- Authenticate the Assertion Issuer as a federation member (that
  is an `issuer_authentication` question).
- Validate the attribute value itself. The Attribute Authority
  authorizes the Assertion Issuer to make the attestation; it does
  NOT thereby vouch for the truth of any specific assertion's
  attribute value.

# Attribute Authority Authorization Policy {#aaap-document}

The Attribute Authority Authorization Policy is a JSON document
served over HTTPS with media type `application/json`. The document
MUST be a JSON object. Consumers MUST reject a policy document that
is not syntactically valid JSON, whose top-level value is not an
object, or whose required members are absent or have the wrong
JSON type.

Example:

~~~ json
{
  "attribute_authority": "university.example",
  "authorized_issuers": [
    {
      "issuer": "https://idp.degree-issuer.example",
      "attribute_types": ["degree_status", "enrollment_status"],
      "valid_until": "2030-01-01T00:00:00Z"
    },
    {
      "issuer": "https://accounts.google.com",
      "attribute_types": ["degree_status"],
      "tenant": "university.example"
    }
  ],
  "last_updated": "2026-05-25T00:00:00Z"
}
~~~

## Members

`attribute_authority`
: REQUIRED. String. The Attribute Authority identifier this policy
applies to. Consumers MUST verify this value matches the
Attribute Authority computed from the assertion's
`<attribute>_authority` claim per {{claim-shape}}.

`authorized_issuers`
: REQUIRED. Non-empty JSON array of authorized issuer objects.
Each object has:

  `issuer`
  : REQUIRED. String. The Assertion Issuer identifier (HTTPS URL,
  compared per {{TRUST-POLICY}} comparison rules).

  `attribute_types`
  : REQUIRED. Non-empty JSON array of Attribute Type Identifiers
  ({{attribute-type-identifiers}}) this Assertion Issuer is authorized
  to attest under this Attribute Authority.

  `tenant`
  : OPTIONAL. String. Same semantics as the `tenant` member of
  {{DAI}}'s `authorized_issuers` entries: binds the authorization
  to a specific tenant of a shared-issuer multi-tenant Identity
  Provider.

  `subject_populations`
  : OPTIONAL. JSON array of strings. When present, restricts the
  authorization to Subjects whose primary subject identifier
  (per the applicable grant profile) is within one of the listed
  Subject Authorities, compared using {{TRUST-POLICY}} §Subject
  Authority Determination. For example, an Attribute Authority might restrict
  a degree-status attestor to attest only about Subjects in the
  university's own email namespace by setting
  `subject_populations: ["university.example"]`. When omitted,
  no subject-population constraint applies.

  `valid_from`
  : OPTIONAL. RFC 3339 date-time. The delegation MUST NOT be
  treated as valid before this time. Same 5-minute skew
  tolerance as {{DAI}}.

  `valid_until`
  : OPTIONAL. RFC 3339 date-time. The delegation MUST NOT be
  treated as valid at or after this time.

`last_updated`
: OPTIONAL. RFC 3339 date-time at which the policy was last
published.

`crit`
: OPTIONAL. JSON array of strings. Same semantics as
{{TRUST-POLICY}} §Critical Extension Identifiers.

`signed_policy`
: OPTIONAL. Signed JWT containing policy members as claims, per
{{TRUST-POLICY}} §Signed Policy Metadata. The JWT's `iss` claim
MUST identify a signing authority controlled by the Attribute
Authority.

# Publication {#publication}

An Attribute Authority publishes its Authorization Policy through an
HTTPS JSON document, located either by DNS pointer or by default
well-known URL:

- **DNS pointer form**: a TXT record at
  `_oauth-attribute-authority-policy.{A}` containing
  `v=oauth-attribute-authority-policy1`, `authority=`, and `uri=`
  directives. The `uri=` value points to the HTTPS JSON document. The
  record MAY also contain `crit=` using {{TRUST-POLICY}} §Critical
  Extension Identifiers.

- **HTTPS well-known form**: a JSON document at
  `https://{A}/.well-known/oauth-attribute-authority-policy`.

The DNS pointer form follows the DNS authority-publication and
third-party-hosting model of {{DAI}}, but this document deliberately
does not define an inline `issuer=` form. Attribute authorization is
claim-type scoped, and a DNS-inline issuer list would either need a new
claim-type grammar or would authorize all current and future claim
types. Consumers MUST treat any `issuer=` directive in an
`_oauth-attribute-authority-policy` TXT record as `malformed`.

The JSON document retrieved from either publication form is the AAAP
defined in {{aaap-document}}. Consumers MUST NOT interpret any other
JSON shape as an AAAP.

## Lookup Procedure {#lookup}

To retrieve the AAAP for Attribute Authority `A`, a consumer performs
the following steps:

1. Query the DNS TXT resource record set at
   `_oauth-attribute-authority-policy.{A}`. Classify the response as:

   `negative-authoritative`
   : NXDOMAIN, or NOERROR with an empty answer section or with no
   recognized records after parsing.

   `indeterminate`
   : SERVFAIL, REFUSED, timeout, truncation with no successful retry,
   or any other failure that prevents a definitive negative result.

   `affirmative`
   : NOERROR with at least one recognized record after parsing.

2. If the DNS response is `affirmative`:

   a. Discard recognized records whose `authority=` directive does not
      match `A` under the comparison rules in {{attribute-authority-determination}}.
      If any recognized record is missing an `authority=` directive,
      treat the response as `malformed`. If all recognized records are
      discarded because of `authority=` mismatch, treat the response as
      `malformed`.

   b. Validate the remaining recognized records. Each remaining record
      MUST contain exactly one `uri=` directive, MUST NOT contain an
      `issuer=` directive, and MUST satisfy the `crit=` processing rules
      of {{TRUST-POLICY}} §Critical Extension Identifiers. More than one
      distinct `uri=` value across the remaining records is `malformed`.

   c. Fetch the JSON policy from the single `uri=` target. The fetched
      document is the AAAP.

3. If the DNS response is `negative-authoritative`, fetch the AAAP from
   the default HTTPS well-known URL per {{publication}}.

4. If the DNS response is `indeterminate`, discovery has failed.
   Consumers MUST NOT fall back to HTTPS.

The procedure produces an Affirmative / Negative / Indeterminate
outcome per the Authority Delegation Pattern's Exception-Handling and
Fallback Model ({{AUTHORITY-DELEGATION}} §Exception-Handling and
Fallback Model).

## Failure Handling {#failures}

AAAP lookup outcomes map onto the following states:

| State | AAAP outcomes |
|-|-|
| Affirmative | A well-formed AAAP was retrieved through a DNS pointer or HTTPS well-known URL, its `attribute_authority` matches `A`, and structural validation succeeds. HTTPS responses are 200 OK with a media type compatible with `application/json`. |
| Negative | DNS `negative-authoritative` followed by HTTPS 404 or 410 at the well-known URL, or HTTPS 404 or 410 reached from a DNS `uri=` pointer. The Attribute Authority authoritatively publishes no delegation. |
| Indeterminate | Any other outcome, fail-closed by default. |

The Indeterminate state covers DNS transport errors, HTTPS transport
errors, non-200 HTTPS responses other than 404 and 410, unsupported
media type, over-size body, malformed DNS directives, multiple distinct
`uri=` values, inline `issuer=` directives, unknown critical
extensions, syntactically invalid AAAP JSON, or an `attribute_authority`
value that does not match `A`.

Consumers MUST treat both Negative and Indeterminate as assertion
rejection. A fresh cached Affirmative policy MAY be used during
transient Indeterminate on the live channel, subject to a maximum
24-hour positive cache lifetime and a cumulative-Indeterminate cap of
1 hour. The cache lifetime MUST NOT be extended by repeated
Indeterminate retrievals.

# Combined Evaluation

When the `attribute_attestation` Trust Method category is present
in a Trust Policy alongside the categories of {{TRUST-POLICY}}, the
cross-category combination rule applies unchanged:

- The Assertion Issuer MUST satisfy at least one
  `issuer_authentication` method (when present in the policy).
- The Assertion Issuer MUST satisfy at least one
  `subject_namespace_authorization` method (when the assertion
  carries a namespace-bound Subject Identifier).
- For each protected attribute claim in the assertion, the Assertion Issuer
  MUST satisfy at least one `attribute_attestation` method against
  the attribute's Authority.

If the assertion carries N protected attribute claims under N distinct
Attribute Authorities, the Resource Authorization Server performs
N independent `attribute_attestation` evaluations, each potentially
fetching a different Attribute Authority's policy. Each evaluation
MUST succeed.

# Worked Example (Non-Normative)

This appendix is non-normative.

A university `university.example` confers degrees on its students.
Alice graduated and now uses a third-party Identity Provider
`https://idp.alumni-services.example` that the university has
contracted to assert degree status for its alumni. Alice presents
an assertion to a hiring platform at
`https://api.hiring-platform.example`.

## Publication {#example-publication}

The university publishes a DNS record at
`_oauth-attribute-authority-policy.university.example`:

~~~
_oauth-attribute-authority-policy.university.example.  IN  TXT
  "v=oauth-attribute-authority-policy1;
   authority=university.example;
   uri=https://attestations.university.example/aaap.json"
~~~

The JSON policy at the pointed-at URL:

~~~ json
{
  "attribute_authority": "university.example",
  "authorized_issuers": [
    {
      "issuer": "https://idp.alumni-services.example",
      "attribute_types": ["degree_status", "graduation_year"],
      "valid_until": "2030-01-01T00:00:00Z"
    }
  ],
  "last_updated": "2026-05-25T00:00:00Z"
}
~~~

## Assertion {#example-assertion}

Alice's IdP issues an ID-JAG carrying:

~~~ json
{
  "iss": "https://idp.alumni-services.example",
  "sub": "alice-1234",
  "aud": "https://api.hiring-platform.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "email": "alice@personal.example",
  "email_verified": true,
  "degree_status": "graduated",
  "degree_status_authority": "university.example",
  "degree_status_authority_verified": true,
  "graduation_year": "2018",
  "graduation_year_authority": "university.example",
  "graduation_year_authority_verified": true
}
~~~

## Hiring Platform Trust Policy {#example-trust-policy}

~~~ json
{
  "resource_authorization_server": "https://api.hiring-platform.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "attribute_attestation_claims": [
    "degree_status",
    "graduation_year"
  ],
  "issuer_trust_methods": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    {
      "method": "domain_authorized_issuer"
    },
    {
      "method": "attribute_authority_authorized_issuer"
    }
  ]
}
~~~

## Verification {#example-verification}

The hiring platform evaluates three Trust Methods:

1. **`openid_federation`**: validates that
   `https://idp.alumni-services.example` is a member of a
   federation rooted at `https://federation.example.org`.

2. **`domain_authorized_issuer`**: looks up the Issuer
   Authorization Policy for `personal.example` (Alice's email
   namespace). The lookup succeeds; the IdP is listed.

3. **`attribute_authority_authorized_issuer`**: evaluates each
   attribute claim:

   a. For `degree_status`: looks up the AAAP at
   `_oauth-attribute-authority-policy.university.example`. The
   policy lists the IdP authorized for `degree_status`. Satisfied.

   b. For `graduation_year`: the same AAAP lookup. The same entry
   covers `graduation_year`. Satisfied.

All three categories satisfied; the hiring platform accepts the
assertion.

## Threat: Unauthorized Attribute Attestation

Suppose a different Assertion Issuer
`https://idp.diploma-mill.example` (not authorized by the
university) attempts to attest a degree status for an unrelated
Subject:

~~~ json
{
  "iss": "https://idp.diploma-mill.example",
  "email": "carol@example.com",
  "email_verified": true,
  "degree_status": "graduated",
  "degree_status_authority": "university.example",
  "degree_status_authority_verified": true
}
~~~

The hiring platform evaluates `attribute_attestation` for the
degree_status claim with Authority `university.example`. The AAAP
lookup retrieves the policy, but
`https://idp.diploma-mill.example` is not listed in
`authorized_issuers`. The Trust Method is not satisfied; the
assertion is rejected.

The diploma mill's assertion-signing authority and any federation
membership it may have are irrelevant. Only the university's
published policy determines who can attest its degree-status
claims.

## Threat: Cross-Authority Attribute Substitution

Suppose an attacker controls an Authorized Issuer for
`other-university.example` (issuing valid attestations for that
university) and attempts to attest a degree from
`university.example`:

~~~ json
{
  "iss": "https://idp.other-issuer.example",
  "email": "alice@personal.example",
  "email_verified": true,
  "degree_status": "graduated",
  "degree_status_authority": "university.example",
  "degree_status_authority_verified": true
}
~~~

The hiring platform fetches `university.example`'s AAAP and finds
that `https://idp.other-issuer.example` is NOT authorized.
Rejection. The attacker's authorization for a DIFFERENT Attribute
Authority does not extend to claims under `university.example`.

This is the same wire-format protection as DAI provides for
namespace authorization, applied per-attribute-per-authority.

## eKYC-IDA Variant {#ekyc-ida-example}

The same hiring-platform scenario using an OIDC4IDA
`verified_claims` payload rather than flat triples.

The trust framework owner (a regulator or scheme administrator,
identified for example as `idp-assurance.example`) publishes an
AAAP at `_oauth-attribute-authority-policy.idp-assurance.example`
listing IdPs authorized to issue verified claims under the
framework:

~~~ json
{
  "attribute_authority": "idp-assurance.example",
  "authorized_issuers": [
    {
      "issuer": "https://idp.alumni-services.example",
      "attribute_types": ["given_name", "birthdate", "degree_status"],
      "valid_until": "2030-01-01T00:00:00Z"
    }
  ],
  "last_updated": "2026-05-25T00:00:00Z"
}
~~~

The IdP issues an OIDC4IDA-shaped assertion:

~~~ json
{
  "iss": "https://idp.alumni-services.example",
  "sub": "alice-1234",
  "aud": "https://api.hiring-platform.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "email": "alice@personal.example",
  "email_verified": true,
  "verified_claims": {
    "verification": {
      "trust_framework": "idp-assurance.example",
      "evidence": [
        {
          "type": "document",
          "method": "pipp",
          "document_details": { "type": "passport" }
        }
      ]
    },
    "claims": {
      "given_name": "Alice",
      "birthdate": "1990-01-01",
      "degree_status": "graduated"
    }
  }
}
~~~

The hiring platform evaluates this assertion exactly as in the
flat-triple example, with one substitution: the Attribute
Authority is read from
`verified_claims.verification.trust_framework` rather than per-claim
`_authority` triples. All three claims in `verified_claims.claims`
(`given_name`, `birthdate`, `degree_status`) share the single
trust-framework Authority and are evaluated against the same
AAAP lookup.

The trust evaluation outcome is identical to the flat-triple
case: the IdP must be listed in the trust framework's AAAP for
ALL claim names being attested. The OIDC4IDA payload's
`verification.evidence` and `verification_process` are claim-layer
metadata that the assertion grant profile and the Resource
Authorization Server's local policy interpret; they are not
evaluated by this Trust Method.

# Security Considerations

## Independence of Attribute Claims

Each attribute claim is evaluated independently against its own
Attribute Authority. An assertion carrying multiple attribute
claims under different authorities requires the asserting Issuer
to be authorized by EVERY relevant authority. This is the
load-bearing security property: an Issuer authorized by one
attribute authority cannot synthesize claims under a different
attribute authority.

## Attribute Authority vs Subject Namespace

The Attribute Authority is conceptually distinct from the
Subject's namespace. Attribute claims travel with the Subject
across namespaces (Alice's degree from `university.example` remains
hers when she changes her email from `alice@school.edu` to
`alice@personal.example`). Resource Authorization Servers MUST NOT
infer that an Attribute Authority has authority over the
corresponding Subject's namespace, or vice versa.

## Protected Attribute Omission

An Assertion Issuer might try to avoid attribute-authority evaluation
by omitting `<attribute>_authority`, `<attribute>_authority_verified`,
or `verified_claims` metadata while still including a claim that the
Resource Authorization Server treats as security-relevant. Resource
Authorization Servers MUST maintain a deterministic set of protected
attribute claims, either from `attribute_attestation_claims` or local
policy, and MUST reject protected claims whose required authority
metadata is absent or malformed.

## Attribute Verification Is Issuer-Level

The `<attribute>_authority_verified` claim is asserted by the
Issuer, not by the Attribute Authority. The Authority's
authorization establishes that the Issuer is permitted to attest
the attribute; the Issuer's internal procedures (registration
verification, periodic recheck, etc.) establish the truth of any
specific assertion's value. This Trust Method does not provide
defense against an authorized but malicious or compromised Issuer
attesting false attribute values about Subjects.

Attribute Authorities listing Assertion Issuers in their AAAP
SHOULD apply the same diligence as Subject Authorities listing
Assertion Issuers in DAI: contract review, audit reports,
operational practices, and revocation readiness.

## Subdomain Authority

Unlike {{DAI}}'s registrable-domain Subject Authority extraction,
this document treats the `<attribute>_authority` claim's value as
the exact Attribute Authority identifier. This permits Attribute
Authorities to be operationally distinct at the subdomain level
(`alumni.university.example` separate from `university.example`).

Consequences:

- The protection against subdomain takeover that the
  registrable-domain default provides for namespace authorities is
  NOT present here. A subdomain-takeover attacker who controls
  `alumni.university.example`'s DNS can publish a competing AAAP at
  that subdomain.

- Resource Authorization Servers SHOULD log the exact
  `<attribute>_authority` value used and SHOULD treat unexpected
  subdomain attestations as suspicious.

- Attribute Authorities deploying at subdomain granularity SHOULD
  apply registry-lock and DNSSEC at the relevant subdomain.

## Same-Authority Attribute Confusion

An Attribute Authority publishing one AAAP that authorizes
different issuers for different attribute types relies on the
issuer-to-attribute-type binding in `authorized_issuers[].attribute_types`.
A misconfiguration that authorizes too many attribute types for an
issuer effectively expands the issuer's authority.

Resource Authorization Servers SHOULD log the matched
`authorized_issuers` entry and the specific
`attribute_types` element that satisfied the evaluation. Attribute
Authorities SHOULD apply least-privilege when authoring AAAP
entries.

## Composition with Subject Namespace Authorization

When an assertion is evaluated under BOTH
`subject_namespace_authorization` (for the Subject's namespace) AND
`attribute_attestation` (for the attribute claims), the two
evaluations are independent. The Assertion Issuer must be
authorized in BOTH places. An attacker who is authorized in one but
not the other cannot mint accepted assertions.

# Privacy Considerations

The AAAP discloses which Assertion Issuers an Attribute Authority
trusts to attest its claims. For some Authorities (universities
listing third-party degree-attestation services; employers listing
contracted identity providers), this discloses operational
relationships and may be commercially sensitive.

Attribute Authorities SHOULD consider whether AAAP publication is
appropriate for their deployment. Authorities with privacy
concerns MAY use Profile 3 (signed-policy publication) of {{DAI}}
applied to AAAPs, hosting the document at a third-party endpoint
while preserving signature-based authority.

Attribute attestations themselves typically expose claim values
(degree status, employment status, KYC level) that are more
sensitive than namespace assertions. The privacy properties of
attribute attestations are governed by the assertion grant profile
and the Resource Authorization Server's use of the claims, not by
the trust-method evaluation defined here.

# IANA Considerations

## Trust Method Category Registration

IANA is requested to add the following entry to the "Identity
Assertion Issuer Trust Method Categories" registry established by
{{TRUST-POLICY}}. Registration policy: Specification Required
{{RFC8126}}.

Category Identifier:
: `attribute_attestation`

Description:
: Has the Attribute Authority for an asserted attribute claim
  authorized the Assertion Issuer to attest that claim?

Applicability:
: Applies per-attribute when the identity assertion carries an
  attribute claim protected by `attribute_attestation_claims`, local
  Resource Authorization Server policy, or authority metadata in a
  recognized claim shape.

Evaluation Subject:
: The Assertion Issuer (the `iss` claim of the identity assertion)
  evaluated against the policy of the Attribute Authority named by
  the `<attribute>_authority` claim.

Reference:
: This document.

## Trust Policy Member Registration

IANA is requested to add the following entry to the "Identity Assertion
Issuer Trust Policy Members" registry established by {{TRUST-POLICY}}:

Member Name:
: `attribute_attestation_claims`

Member Description:
: JSON array of Attribute Type Identifiers for claims that require
  `attribute_attestation` evaluation.

Change Controller:
: IETF

Specification Document:
: This document

## Trust Method Registration

IANA is requested to add the following entry to the "Identity
Assertion Issuer Trust Methods" registry established by
{{TRUST-POLICY}}.

Identifier:
: `attribute_authority_authorized_issuer`

Categories:
: `attribute_attestation`

Parameters:
: (none)

Reference:
: This document.

## Well-Known URI

Registers the following well-known URI in the IANA "Well-Known
URIs" registry {{RFC8615}}:

URI Suffix:
: `oauth-attribute-authority-policy`

Change Controller:
: IETF

Specification Document:
: This document, {{publication}}

Status:
: permanent

Related Information:
: None

## Underscored DNS Node Name

Registers the following entry in the IANA "Underscored and
Globally Scoped DNS Node Names" registry per {{RFC8552}} and
{{RFC8553}}:

RR Type:
: TXT

Node Name:
: `_oauth-attribute-authority-policy`

Specification Document:
: This document, {{publication}}

## Authority Delegation Profile Registration

This document registers an entry in the Authority Delegation
Profile registry of {{AUTHORITY-DELEGATION}} (§Authority Delegation
Profile Registry):

Profile Identifier:
: `urn:ietf:params:oauth:authority-delegation-profile:attribute-authority-trust`

Authority Holder Form:
: DNS domain (exact, not registrable; subdomain-level granularity
permitted).

Delegate Form:
: Assertion Issuer (OAuth authorization server identifier; HTTPS
URL).

Publication Channel:
: DNS TXT at `_oauth-attribute-authority-policy.{authority}` carrying
an HTTPS pointer; HTTPS fallback at the authority's well-known URL;
signed-policy publication at arbitrary HTTPS origins via {{DAI}}'s
Profile 3 mechanism applied to AAAPs. Inline DNS issuer lists are not
defined by this profile.

Subdelegation:
: Not permitted. `authorized_issuers` is a flat list; bounded
depth-1.

Reference:
: This document.

Profile Mapping Matrix:

`source_selection`
: The Attribute Authority is extracted from the assertion's
recognized claim shape for each protected attribute claim under
evaluation. Selection uses one authority value per attribute; selection
is invariant under unrelated claims. The function is per-attribute
deterministic.

`revocation_lifecycle`
: Cache-bounded with no push mechanism, mirroring {{DAI}}'s cache
model. Positive cache lifetime is bounded by DNS TTL and HTTP
`Cache-Control`, capped at the profile-specified maximum
(recommended ≤24h ceiling). Cumulative-Indeterminate cap is
1 hour. Negative caching subject to the same lifetime ceiling.

`transitivity_limit`
: Depth-1. The Attribute Authority lists Assertion Issuers
directly; no further delegation is permitted.

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
