---
title: "Attribute Authority Trust Method for OAuth Identity Assertion Issuer Trust Policy"
abbrev: "OAuth Attribute Authority Trust"
docname: draft-mcguinness-oauth-attribute-authority-trust-latest
category: std
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
  RFC7519:
  RFC8126:
  RFC8552:
  RFC8553:
  RFC8615:
  TRUST-POLICY:
    title: "OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-issuer-trust-policy/
    date: false

informative:
  RFC7515:
  RFC7521:
  RFC8659:
  RFC9493:
  AUTHORITY-DELEGATION:
    title: "OAuth Authority Delegation Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-authority-delegation-framework/
    date: false
  DAI:
    title: "OAuth Domain-Authorized Issuer Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer-discovery/
    date: false
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
question. It registers a third Trust Method category,
`attribute_attestation`, and a Trust Method,
`attribute_authority_authorized_issuer`, that lets an attribute
authority (a university, an employer, a KYC provider, a
credentialing body, or any other claim-type authority) publish the
set of Assertion Issuers it authorizes to attest its claim types.
The wire format follows the DNS-based publication pattern of
{{DAI}} at a different owner name, accommodating any attribute
authority whose identity can be expressed as a DNS-named domain.

--- middle

# Introduction

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
{{AUTHORITY-DELEGATION}} with a third Trust Method category. The
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
- **This document**: registers a third category,
  `attribute_attestation`, applying the same Authority Delegation
  Pattern to claim-type authority. The Authority Holder is the
  attribute authority (a university, employer, KYC provider, or
  credentialing body); the Delegate is the Assertion Issuer
  authorized to attest the claim type; the Delegation Artifact is
  the Attribute Authority Authorization Policy.

This document depends normatively on {{TRUST-POLICY}} (Trust Method
machinery, cross-category combination rule, IANA registries) and
borrows the DNS+HTTPS publication pattern of {{DAI}}.

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

The OpenID Foundation's eKYC and Identity Assurance Working Group
publishes specifications {{IDA-VERIFIED-CLAIMS}} for carrying
verified identity claims with provenance metadata in OAuth and
OpenID Connect. OpenID Connect for Identity Assurance
({{OIDC4IDA}}) defines a `verified_claims` payload that wraps an
identity-assurance attestation:

~~~ json
{
  "verified_claims": {
    "verification": {
      "trust_framework": "eidas",
      "evidence": [
        {
          "type": "document",
          "method": "pipp",
          "document_details": { "type": "passport" }
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

OIDC4IDA owns:

- The wire format for verified-claims payloads.
- The standardized vocabulary for evidence types, verification
  methods, and trust frameworks.
- The verification metadata schema (`verification` object).

OIDC4IDA does NOT define:

- A wire-format mechanism by which the trust framework's owner
  (for example, eIDAS, NIST, GLEIF, a national identity scheme)
  publishes which Assertion Issuers it has authorized to issue
  verified claims under it.
- An OAuth-side trust-evaluation framework for the Verifier to
  discover whether the Issuer claiming
  `"trust_framework": "eidas"` is in fact authorized under eIDAS.

OIDC4IDA's trust model is implicit: the Verifier is assumed to know,
out of band, which Issuers it trusts under which trust frameworks.

This document closes that gap. The trust framework named in
`verified_claims.verification.trust_framework` is the Attribute
Authority of this document; the JWT's `iss` is the Assertion
Issuer; the `verified_claims.claims` payload is the body of
attestations the Authority authorizes the Issuer to make.

When evaluating an OIDC4IDA payload, the Resource Authorization
Server consults the trust framework's Attribute Authority
Authorization Policy (per the lookup procedure in {{lookup}}) to
discover whether the JWT's Issuer is listed as authorized under
that trust framework. A worked example appears in
{{ekyc-ida-example}}.

A deployment using OIDC4IDA typically uses OIDC4IDA for the
claim-payload wire format and this document for the OAuth-side
issuer-trust evaluation. {{claim-shape}} defines two recognized
attribute-claim shapes: the flat-triple convention and the
OIDC4IDA `verified_claims` wrapper.

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
assertion carries one or more attribute claims accompanied by an
Attribute Authority designation. The applicability of the category
is per-attribute: the Resource Authorization Server evaluates the
category once for each attribute claim that is subject to attribute
authority verification.

An assertion carrying NO attribute claims does not trigger
applicability of this category, even when the Trust Policy lists
methods in this category.

Evaluation subject:

The Trust Method in this category evaluates the Assertion Issuer
(the `iss` claim of the identity assertion) against the policy
published by the Attribute Authority of the asserted attribute
claim. The same Assertion Issuer may be evaluated against multiple
Attribute Authorities when the assertion carries multiple
attribute claims; each evaluation is independent.

# Attribute Claim Shape and Authority Determination {#claim-shape}

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
      "trust_framework": "<authority>",
      "evidence": [ ... ],
      "verification_process": "..."
    },
    "claims": {
      "<attribute>": "<value>",
      ...
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

## Attribute Authority Determination

The Attribute Authority for each attribute claim is determined
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
against an attribute claim in the identity assertion:

1. Determine the Attribute Authority `A` from the assertion's
   `<attribute>_authority` claim per {{claim-shape}}.

2. If the corresponding `<attribute>_authority_verified` claim is
   absent or not `true`, the Trust Method is not satisfied; the
   Resource Authorization Server MUST reject the assertion.

3. Retrieve the Attribute Authority Authorization Policy for `A`
   per the lookup procedure in {{lookup}}. Negative and
   Indeterminate outcomes ({{failures}}) MUST result in rejection.

4. Find an entry in `authorized_issuers` whose `issuer` matches the
   assertion's `iss` claim using case-sensitive URL string
   equality, AND whose `attribute_types` list includes the
   attribute name being evaluated.

5. If the entry contains a `tenant` member, verify the assertion's
   `tenant` claim matches per {{TRUST-POLICY}} multi-tenant
   binding rules.

6. If the entry contains `valid_from`/`valid_until`, verify the
   current time is within the validity window.

7. If all checks succeed, the Trust Method is satisfied for this
   attribute. If the assertion carries multiple attribute claims,
   the Resource Authorization Server repeats steps 1-6 for each
   attribute claim independently.

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
  : REQUIRED. Non-empty JSON array of strings, each a claim name
  this Assertion Issuer is authorized to attest under this
  Attribute Authority. Each string corresponds to the
  `<attribute>` portion of the claim triple defined in
  {{claim-shape}}.

  `tenant`
  : OPTIONAL. String. Same semantics as the `tenant` member of
  {{DAI}}'s `authorized_issuers` entries: binds the authorization
  to a specific tenant of a shared-issuer multi-tenant Identity
  Provider.

  `subject_populations`
  : OPTIONAL. JSON array of strings. When present, restricts the
  authorization to Subjects whose primary subject identifier
  (per the applicable grant profile) is within one of the listed
  namespaces. For example, an Attribute Authority might restrict
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

An Attribute Authority publishes its Authorization Policy at:

- **DNS form**: a TXT record at
  `_oauth-attribute-authority-policy.{A}`, with format identical
  to {{DAI}}'s DNS record format (version directive
  `v=oauth-attribute-authority-policy1`, `authority=`, `uri=`,
  `issuer=`, `crit=` directives).

- **HTTPS form**: a JSON document at
  `https://{A}/.well-known/oauth-attribute-authority-policy`.

The inline DNS form (`issuer=` directives) is sufficient when the
Attribute Authority authorizes one or more Assertion Issuers for
ALL of its claim types. When the Authority authorizes different
issuers for different claim types, OR uses any feature
(`attribute_types` per-entry, `subject_populations`,
`valid_from`/`valid_until`, `tenant`), the DNS pointer form
(`uri=`) MUST be used and the richer policy carried in the JSON
document.

A Subject Authority using the DNS-inline form publishes a virtual
policy whose every `authorized_issuers` entry has
`attribute_types: ["*"]` (matching all attribute names). When this
matches an attribute claim's name, the Trust Method is satisfied
for that claim. The wildcard `*` MUST NOT appear in HTTPS-hosted
policy `attribute_types` arrays; the wildcard is reserved for the
DNS-inline virtual-policy construction.

The three publication profiles of {{DAI}} §Publication Profiles
apply unchanged with the corresponding owner name and well-known
URL substitutions.

## Lookup Procedure {#lookup}

The lookup procedure mirrors {{DAI}} §Lookup Procedure with the
owner name and well-known URL substitutions noted above. The
procedure produces an Affirmative / Negative / Indeterminate
outcome per the Authority Delegation Pattern's Exception-Handling
and Fallback Model ({{AUTHORITY-DELEGATION}} §Exception-Handling
and Fallback Model). Cache freshness rules also mirror {{DAI}}
§Caching: maximum 24-hour ceiling, cumulative-Indeterminate cap of
1 hour, Negative caching subject to the same lifetime.

## Failure Handling {#failures}

Failure-state mapping is identical to {{DAI}} §Failure Handling
with owner name substitutions. Negative and Indeterminate states
MUST result in assertion rejection.

# Combined Evaluation

When the `attribute_attestation` Trust Method category is present
in a Trust Policy alongside the categories of {{TRUST-POLICY}}, the
cross-category combination rule applies unchanged:

- The Assertion Issuer MUST satisfy at least one
  `issuer_authentication` method (when present in the policy).
- The Assertion Issuer MUST satisfy at least one
  `subject_namespace_authorization` method (when the assertion
  carries a namespace-bound Subject Identifier).
- For each attribute claim in the assertion, the Assertion Issuer
  MUST satisfy at least one `attribute_attestation` method against
  the attribute's Authority.

If the assertion carries N attribute claims under N distinct
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
  attribute claim and the corresponding `<attribute>_authority` and
  `<attribute>_authority_verified` claims.

Evaluation Subject:
: The Assertion Issuer (the `iss` claim of the identity assertion)
  evaluated against the policy of the Attribute Authority named by
  the `<attribute>_authority` claim.

Reference:
: This document.

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
: DNS TXT at `_oauth-attribute-authority-policy.{authority}` with
optional HTTPS pointer; HTTPS fallback at the authority's
well-known URL; signed-policy publication at arbitrary HTTPS
origins via {{DAI}}'s Profile 3 mechanism applied to AAAPs.

Subdelegation:
: Not permitted. `authorized_issuers` is a flat list; bounded
depth-1.

Reference:
: This document.

Profile Mapping Matrix:

`source_selection`
: The Attribute Authority is extracted from the assertion's
`<attribute>_authority` claim for each attribute claim under
evaluation. Selection uses one property per attribute (the
attribute's `_authority` claim); selection is invariant under
other claims. The function is per-attribute deterministic.

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
