---
title: "Deployment Scenarios for the OAuth Identity Assertion Trust Policy Family"
abbrev: "OAuth Trust Policy Scenarios"
docname: draft-mcguinness-oauth-trust-policy-scenarios-latest
category: info
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - trust policy
  - deployment
  - examples

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

informative:
  RFC9700:
  AUTHORITY-DELEGATION:
    title: "OAuth Authority Delegation Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-authority-delegation-framework/
    date: false
  TRUST-POLICY:
    title: "OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-issuer-trust-policy/
    date: false
  DAI:
    title: "OAuth Domain-Authorized Issuer Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer-discovery/
    date: false
  CLIENT-INSTANCE-TRUST:
    title: "Client Instance Trust Method for OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-trust/
    date: false
  ATTRIBUTE-AUTHORITY-TRUST:
    title: "Attribute Authority Trust Method for OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-attribute-authority-trust/
    date: false
  TPD:
    title: "OAuth Trust Policy Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-trust-policy-discovery/
    date: false
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  CIA:
    title: "OAuth 2.0 Client Instance Assertions using Actor Tokens"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion/
    date: false
  CIMD:
    title: "OAuth Client Identifier Metadata Document"
    target: https://datatracker.ietf.org/doc/draft-parecki-oauth-client-id-metadata-document/
    date: false
  OIDC4IDA:
    title: "OpenID Connect for Identity Assurance 1.0"
    target: https://openid.net/specs/openid-connect-4-identity-assurance-1_0.html
  OIDF-FEDERATION:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
  SPIFFE:
    title: "Secure Production Identity Framework for Everyone (SPIFFE)"
    target: https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/

---

--- abstract

This document is a non-normative companion to the OAuth Identity
Assertion Issuer Trust Policy family of specifications. It
presents five end-to-end deployment scenarios that span the
family's profiles: Domain-Authorized Issuer Discovery, Client
Instance Trust, and Attribute Authority Trust. For each scenario,
the document identifies the actors, what each actor publishes or
configures, the resulting token request flow, the Resource
Authorization Server's evaluation, and the spec sections that
apply.

Implementers and deployers can use this document to identify
which specifications apply to their use case and to verify their
deployment shape against a worked example.

--- middle

# Introduction

The OAuth Identity Assertion Issuer Trust Policy family
specifies how a Resource Authorization Server publishes its trust
criteria and how Assertion Issuers prove that they meet those
criteria. The family consists of:

- {{AUTHORITY-DELEGATION}}: the abstract Authority Delegation Pattern.
- {{TRUST-POLICY}}: the OAuth-side framework (Trust Policy
  document, Trust Method machinery, Subject Authority
  Determination, grant-profile bindings).
- {{DAI}}: DNS+HTTPS Subject-Authority publication mechanism
  with two Trust Methods (`domain_authorized_issuer`,
  `https_authorized_issuer`).
- {{CLIENT-INSTANCE-TRUST}}: third Trust Method category
  `client_instance_authorization`, applying the pattern to
  workload-to-client binding.
- {{ATTRIBUTE-AUTHORITY-TRUST}}: fourth Trust Method category
  `attribute_attestation`, applying the pattern to claim-type
  authorities (universities, employers, KYC providers,
  credentialing bodies).

This document walks through five end-to-end deployment scenarios
that exercise these specifications. Each scenario is presented in
a uniform format so readers can compare them and identify the
shape closest to their deployment.

## Reading Guide

Each scenario in this document is structured as:

- **Problem**: the deployment situation the scenario addresses.
- **Actors**: the parties involved.
- **What is Published**: configurations and policies each actor
  publishes (DNS records, HTTPS documents, OAuth metadata).
- **Token Request**: the wire-format exchange.
- **Verification**: step-by-step trust evaluation at the Resource
  Authorization Server.
- **Specs Used**: the specific sections of the family that apply.

JSON and DNS examples are illustrative; deployments adapt them to
their own identifiers and certificate authorities. URLs in
examples (`*.example`, `*.example.com`) are reserved per
RFC 2606 and do not refer to real services.

## Conventions

This document is non-normative. The five scenarios show how the
normative specifications of the family combine in concrete
deployments. The normative specifications remain authoritative
for any disagreement.

# Scenario 1: Workforce SSO into Multi-Vendor SaaS {#scenario-workforce}

## Problem

An enterprise customer (`acme.example`) federates its workforce
Identity Provider into many SaaS vendors. Each vendor needs to
verify that the IdP claiming to issue assertions about
`*@acme.example` users is in fact the IdP the customer
authorizes for its email namespace. Without a wire-format check,
each vendor configures the trusted IdP bilaterally per customer:
operationally expensive, fragile at scale, and undetectable when
misconfigured.

## Actors

- **Customer**, `acme.example`. Owns the email domain. Publishes
  DAI records authorizing one IdP.
- **Customer's IdP**, `https://idp.workforce-vendor.example`. The
  IdP service the customer has contracted with.
- **SaaS Vendor**, `https://api.saas.example`. The Resource
  Authorization Server that publishes the Trust Policy and
  verifies incoming assertions.
- **End user**, Alice (`alice@acme.example`).
- **Client**, a SaaS-side client application.

## What is Published

The customer publishes a DAI record:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://idp.workforce-vendor.example"
~~~

The SaaS vendor publishes a Trust Policy at
`https://api.saas.example/.well-known/identity-assertion-trust-policy`:

~~~ json
{
  "resource_authorization_server": "https://api.saas.example",
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

This Trust Policy says: any Assertion Issuer is acceptable IF
listed in the Issuer Authorization Policy published by the
asserted subject's email domain.

## Token Request

Alice authenticates to her IdP. The IdP mints an ID-JAG and the
client presents it to the SaaS:

~~~ json
{
  "iss": "https://idp.workforce-vendor.example",
  "sub": "alice-1234",
  "aud": "https://api.saas.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "email": "alice@acme.example",
  "email_verified": true
}
~~~

## Verification

The SaaS vendor evaluates the Trust Policy:

1. Validate the ID-JAG per the grant profile (signature, audience,
   expiration, replay).
2. Verify the format `email` is in
   `subject_identifier_formats_supported`. Yes.
3. Evaluate `domain_authorized_issuer`:
   - Extract Subject Authority: `acme.example` (registrable
     domain of `alice@acme.example`).
   - Look up DAI at `_oauth-issuer-policy.acme.example`.
   - Inline form: `issuer=https://idp.workforce-vendor.example` is
     listed.
   - The ID-JAG's `iss` matches. Trust Method satisfied.
4. Accept.

## Specs Used

- {{TRUST-POLICY}}: Trust Policy document, RAS Processing.
- {{DAI}}: `domain_authorized_issuer` Trust Method, DNS-inline form,
  Subject Authority extraction for `email`.
- {{ID-JAG}}: grant profile and assertion format.

## What This Prevents

A misconfigured or malicious vendor cannot accept an unrelated
IdP's claim about `alice@acme.example` without `acme.example`'s
explicit authorization. A different IdP attempting the same
assertion fails: the IdP's `iss` does not match the
`authorized_issuers` entry.

# Scenario 2: Shared-Issuer Multi-Tenant Identity Provider {#scenario-multitenant}

## Problem

The customer uses a shared-issuer multi-tenant IdP (for example,
Google Workspace at `https://accounts.google.com`, serving many
tenants under one `iss` value). Without tenant binding, listing
the shared issuer in DAI would authorize EVERY tenant of the IdP
to assert about the customer's email namespace. The deployment
needs to bind authorization to one specific tenant.

## Actors

- **Customer**, `acme.example`. Publishes DAI with tenant binding.
- **Shared-issuer IdP**, `https://accounts.google.com`. Serves
  many tenants under one issuer identifier. Each assertion
  carries a top-level `tenant` claim.
- **SaaS Vendor**, same as Scenario 1.

## What is Published

The customer publishes a DAI pointer record (the inline form
cannot carry tenant binding):

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   uri=https://acme.example/.well-known/oauth-issuer-policy"
~~~

The pointed-at JSON policy:

~~~ json
{
  "subject_authority": "acme.example",
  "authorized_issuers": [
    {
      "issuer": "https://accounts.google.com",
      "tenant": "acme.example",
      "subject_identifier_formats": ["email"]
    }
  ],
  "last_updated": "2026-05-25T00:00:00Z"
}
~~~

The SaaS vendor's Trust Policy is the same as Scenario 1
(`domain_authorized_issuer` only).

## Token Request

The customer's tenant of the shared IdP mints an ID-JAG:

~~~ json
{
  "iss": "https://accounts.google.com",
  "sub": "alice-google-1234",
  "aud": "https://api.saas.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "tenant": "acme.example",
  "email": "alice@acme.example",
  "email_verified": true
}
~~~

The `tenant` claim is set to `acme.example` by Google's tenant
isolation (only `acme.example`'s tenant can mint assertions with
this tenant value).

## Verification

The SaaS vendor evaluates:

1. Validate the ID-JAG per the grant profile.
2. Format check passes.
3. Evaluate `domain_authorized_issuer`:
   - Extract Subject Authority: `acme.example`.
   - Fetch the JSON policy via the DNS `uri=` pointer.
   - Find an entry where `issuer` matches AND `tenant` matches the
     assertion's `tenant` claim. Match.
4. Accept.

## What This Prevents

A different Google Workspace tenant (`attacker-corp`) attempting
to assert about `alice@acme.example` would mint:

~~~ json
{
  "iss": "https://accounts.google.com",
  "tenant": "attacker-corp",
  "email": "alice@acme.example",
  "email_verified": true,
  ...
}
~~~

The `iss` matches the policy entry but the `tenant` value does
not match (`acme.example` is required). Trust Method not
satisfied. Rejected.

This composition rests on Google's tenant-isolation enforcement
(a tenant cannot mint assertions claiming a different tenant's
value). The wire-format check surfaces the customer's CHOICE of
authorized tenant; tenant-isolation is the IdP's engineering
responsibility per
{{DAI}} §Single-Issuer Multi-Tenant Identity Providers.

## Specs Used

- {{TRUST-POLICY}}: Trust Policy document, RAS Processing.
- {{DAI}}: `domain_authorized_issuer` with `tenant` binding,
  DNS pointer form, §Single-Issuer Multi-Tenant Identity
  Providers.
- {{ID-JAG}}: top-level `tenant` claim.

# Scenario 3: AI Agent Platform Across Tool Boundaries {#scenario-agent}

## Problem

An AI agent platform (`https://agentprovider.example`)
authenticates users via federated SSO from their primary IdP,
then mints identity assertions toward downstream tools on behalf
of the user. The downstream tool's Resource Authorization Server
needs to know that the agent platform is authorized to assert
about users in the customer's namespace. Without this check, the
tool could accept agent-platform assertions about any user
regardless of namespace.

## Actors

- **Customer**, `acme.example`. Authorizes the agent platform via
  DAI.
- **Customer's primary IdP**, `https://idp.workforce-vendor.example`.
  Used by the agent platform for federated user authentication.
  Not the Assertion Issuer at the tool.
- **Agent Platform**, `https://agentprovider.example`. The
  Assertion Issuer at the downstream tool. Federated member of an
  ecosystem.
- **Tool Provider**, `https://api.tool.example`. The Resource
  Authorization Server.
- **End user**, Alice (`alice@acme.example`).

## What is Published

The customer publishes a DAI record listing the agent platform:

~~~
_oauth-issuer-policy.acme.example.  IN  TXT
  "v=oauth-issuer-policy1;
   authority=acme.example;
   issuer=https://agentprovider.example"
~~~

The tool provider publishes a TPD record (for open-world
discovery by agents on first contact) and the Trust Policy at
the well-known URL:

~~~
_oauth-trust-policy.api.tool.example.  IN  TXT
  "v=oauth-trust-policy1;
   authority=api.tool.example;
   policy_uri=https://api.tool.example/.well-known/identity-assertion-trust-policy;
   trust_methods=openid_federation,domain_authorized_issuer"
~~~

The Trust Policy itself requires BOTH issuer authentication AND
namespace authorization:

~~~ json
{
  "resource_authorization_server": "https://api.tool.example",
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

## Token Request

The agent platform authenticates Alice via federated SSO from her
primary IdP. The agent platform then mints its own ID-JAG with
itself as `iss` and presents it to the tool:

~~~ json
{
  "iss": "https://agentprovider.example",
  "sub": "agent-platform-user-1234",
  "aud": "https://api.tool.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "email": "alice@acme.example",
  "email_verified": true
}
~~~

## Discovery and Verification

The agent platform, encountering the tool's resource URL for the
first time, performs TPD lookup at
`_oauth-trust-policy.api.tool.example`. The DNS response reveals
the Trust Policy URL and the required Trust Methods. The agent
platform fetches the authoritative policy and confirms its
capability to satisfy both methods.

The tool evaluates the Trust Policy with the cross-category
combination rule (AND across categories):

1. Validate the ID-JAG per the grant profile.
2. Format check passes.
3. Evaluate `openid_federation`:
   - Verify the agent platform's federation trust chain to
     `https://federation.example.org`.
   - Validates. Authenticity satisfied.
4. Evaluate `domain_authorized_issuer`:
   - Extract Subject Authority: `acme.example`.
   - Look up DAI at `_oauth-issuer-policy.acme.example`.
   - `https://agentprovider.example` is listed. Authorization
     satisfied.
5. Accept.

Both categories satisfied. The agent platform is authentic (it's a
recognized federation member) AND authorized for the customer's
email namespace (the customer has listed it in DAI).

## What This Prevents

A different agent platform that is ALSO a federation member but
NOT authorized by `acme.example` cannot impersonate Alice. Its
`openid_federation` check passes (federation membership) but
`domain_authorized_issuer` fails (not in `acme.example`'s DAI
policy). The cross-category combination rule requires both;
rejection follows.

This is the load-bearing security property: federation membership
authenticates issuer identity but does NOT confer authority over
any particular subject namespace.

## Specs Used

- {{TRUST-POLICY}}: Trust Policy, Trust Method categories,
  cross-category combination rule.
- {{DAI}}: `domain_authorized_issuer`.
- {{TPD}}: open-world Trust Policy discovery from the resource URL.
- {{OIDF-FEDERATION}}: federation membership.

# Scenario 4: Workload Identity Binding with SPIFFE {#scenario-workload}

## Problem

A SaaS application (`https://app.tenant.example`) runs many
workload instances across a deployment. Each workload needs to
authenticate as the OAuth client to the platform's Resource
Authorization Server. Without workload identity binding, any
party with the client credentials could impersonate the app. With
SPIFFE/SPIRE, workloads obtain short-lived SVIDs from a trust
domain authority, and the OAuth client owner authorizes that
trust domain to issue Client Instance Assertions.

## Actors

- **OAuth Client Owner** (publishes CIMD at
  `https://app.tenant.example/oauth/client-metadata.json`).
  Authorizes SPIFFE trust domains to issue Client Instance
  Assertions.
- **SPIRE Server**, identified by SPIFFE trust domain
  `spiffe://prod.tenant.example`. Issues SVIDs to workloads.
- **Workload**, identified by SPIFFE ID
  `spiffe://prod.tenant.example/svc/api`. Runs as an instance of
  the OAuth client.
- **Resource Authorization Server**, `https://auth.platform.example`.
- **End user**, Alice (`alice@acme.example`), authenticated
  elsewhere.

## What is Published

The OAuth client's CIMD document at
`https://app.tenant.example/oauth/client-metadata.json`:

~~~ json
{
  "client_id": "https://app.tenant.example/oauth/client-metadata.json",
  "client_name": "Example App",
  "instance_issuers": [
    "spiffe://prod.tenant.example"
  ]
}
~~~

The platform's Trust Policy requires client-instance authorization:

~~~ json
{
  "resource_authorization_server": "https://auth.platform.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods": [
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

This policy says: every token request MUST carry a Client Instance
Assertion AND the Assertion Issuer must be DAI-authorized for the
subject's namespace.

## Token Request

The workload obtains an SVID from SPIRE and constructs a Client
Instance Assertion per {{CIA}} signed using the SVID's key. The
workload presents a Token Exchange request:

~~~ http
POST /token HTTP/1.1
Host: auth.platform.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&client_id=https%3A%2F%2Fapp.tenant.example%2Foauth%2Fclient-metadata.json
&subject_token=eyJ...(ID-JAG for Alice)
&subject_token_type=urn:ietf:params:oauth:token-type:id-jag
&actor_token=eyJ...(Client Instance Assertion)
&actor_token_type=urn:ietf:params:oauth:token-type:client-instance-jwt
~~~

The Client Instance Assertion (decoded) carries:

~~~ json
{
  "iss": "spiffe://prod.tenant.example",
  "sub": "spiffe://prod.tenant.example/svc/api",
  "client_id": "https://app.tenant.example/oauth/client-metadata.json",
  "aud": "https://auth.platform.example",
  "exp": 1780166400,
  "iat": 1780166100,
  "cnf": { "jwk": {...} }
}
~~~

## Verification

The platform's RAS evaluates:

1. Validate the subject_token (ID-JAG) per the grant profile.
2. Validate the actor_token (Client Instance Assertion) per
   {{CIA}}.
3. Format check passes.
4. Evaluate `domain_authorized_issuer` for Alice's namespace:
   resolves to `acme.example`'s DAI policy and the ID-JAG's `iss`
   is listed.
5. Evaluate `client_authorized_instance_issuer`:
   - Fetch the OAuth client's CIMD at the `client_id` URL.
   - Find `spiffe://prod.tenant.example` in `instance_issuers`.
     Match.
   - Validate the Client Instance Assertion per {{CIA}}
     (signature, descriptor match, `cnf` proof of possession).
6. Accept.

## What This Prevents

An attacker with a workload identity in a different SPIFFE trust
domain (`spiffe://attacker.example`) cannot impersonate the
client's workloads. The CIMD does not list
`spiffe://attacker.example`, so the
`client_authorized_instance_issuer` check fails. Workload identity
authentication in the attacker's own trust domain is not
substitutable for the OAuth client owner's explicit listing.

## Specs Used

- {{TRUST-POLICY}}: Trust Policy with three Trust Method
  categories.
- {{DAI}}: `domain_authorized_issuer` for the subject's namespace.
- {{CLIENT-INSTANCE-TRUST}}: `client_authorized_instance_issuer`,
  §Working with SPIFFE/SPIRE, §combined-evaluation.
- {{CIA}}: Client Instance Assertion wire format and validation.
- {{CIMD}}: client metadata document publication.
- {{SPIFFE}}: workload identity framework.

# Scenario 5: Verified Identity Claims (eKYC-IDA Integration) {#scenario-ekyc-ida}

## Problem

A regulator or scheme administrator
(`https://idp-assurance.example`) operates a verified-identity
trust framework. IdPs in the framework issue assertions carrying
OIDC4IDA `verified_claims` with evidence metadata. A relying
party needs to verify that an IdP claiming compliance with the
framework is in fact authorized by the framework's owner.

## Actors

- **Trust Framework**, `idp-assurance.example`. Publishes an
  Attribute Authority Authorization Policy listing IdPs authorized
  to issue under the framework.
- **Verified-Identity IdP**, `https://idp.verified-id.example`.
  Authenticates users via in-person identity proofing (passport,
  driver's license) and issues assertions with OIDC4IDA
  `verified_claims`.
- **Relying Party**, `https://api.bank.example`. Requires
  identity-verified onboarding.
- **End user**, Alice.

## What is Published

The trust framework publishes its AAAP:

~~~
_oauth-attribute-authority-policy.idp-assurance.example.  IN  TXT
  "v=oauth-attribute-authority-policy1;
   authority=idp-assurance.example;
   uri=https://attestations.idp-assurance.example/aaap.json"
~~~

The pointed-at policy:

~~~ json
{
  "attribute_authority": "idp-assurance.example",
  "authorized_issuers": [
    {
      "issuer": "https://idp.verified-id.example",
      "attribute_types": ["given_name", "family_name", "birthdate"],
      "valid_until": "2030-01-01T00:00:00Z"
    }
  ],
  "last_updated": "2026-05-25T00:00:00Z"
}
~~~

The relying party publishes a TPD record for open-world
first-contact discovery:

~~~
_oauth-trust-policy.api.bank.example.  IN  TXT
  "v=oauth-trust-policy1;
   authority=api.bank.example;
   policy_uri=https://api.bank.example/.well-known/identity-assertion-trust-policy"
~~~

And the Trust Policy itself:

~~~ json
{
  "resource_authorization_server": "https://api.bank.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "subject_identifier_formats_supported": ["email"],
  "issuer_trust_methods": [
    {
      "method": "domain_authorized_issuer"
    },
    {
      "method": "attribute_authority_authorized_issuer"
    }
  ]
}
~~~

## Token Request

The IdP mints an ID-JAG with OIDC4IDA `verified_claims`:

~~~ json
{
  "iss": "https://idp.verified-id.example",
  "sub": "alice-verified-1234",
  "aud": "https://api.bank.example",
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
      "family_name": "Anderson",
      "birthdate": "1990-01-01"
    }
  }
}
~~~

## Verification

The relying party evaluates:

1. Validate the ID-JAG per the grant profile.
2. Format check passes.
3. Evaluate `domain_authorized_issuer` for Alice's email namespace:
   the IdP must be DAI-authorized by `personal.example` (or
   whichever Subject Authority Alice's email maps to).
4. Evaluate `attribute_authority_authorized_issuer` for the
   `verified_claims` payload:
   - Read the Attribute Authority from
     `verified_claims.verification.trust_framework`:
     `idp-assurance.example`.
   - Look up the AAAP at
     `_oauth-attribute-authority-policy.idp-assurance.example`.
   - `https://idp.verified-id.example` is listed for
     `given_name`, `family_name`, `birthdate`. Match.
   - Each claim in `verified_claims.claims` is one of the
     authorized attribute types. Match.
5. Accept.

The relying party now has cryptographic and trust-framework
backing for the verified attribute values.

## What This Prevents

An unauthorized IdP attempting to issue verified claims under the
framework:

~~~ json
{
  "iss": "https://idp.diploma-mill.example",
  "verified_claims": {
    "verification": {
      "trust_framework": "idp-assurance.example",
      ...
    },
    "claims": { "given_name": "Alice", ... }
  }
}
~~~

The AAAP lookup at `idp-assurance.example` does not list the
diploma-mill IdP. Trust Method not satisfied. Rejected.

## Specs Used

- {{TRUST-POLICY}}: Trust Policy, cross-category combination rule.
- {{DAI}}: `domain_authorized_issuer` for the email namespace.
- {{ATTRIBUTE-AUTHORITY-TRUST}}:
  `attribute_authority_authorized_issuer`, OIDC4IDA
  verified_claims claim shape, §eKYC-IDA Variant worked example.
- {{TPD}}: open-world Trust Policy discovery from the resource URL.
- {{OIDC4IDA}}: OpenID Connect for Identity Assurance wire format.

# Summary Comparison {#summary}

The five scenarios in this document cover the four Trust Method
categories of the family. The combination of categories in each
scenario depends on the trust questions the deployment needs to
answer.

| Scenario | issuer_authentication | subject_namespace_authorization | client_instance_authorization | attribute_attestation |
|-|-|-|-|-|
| 1: Workforce SSO | | DAI inline | | |
| 2: Multi-Tenant IdP | | DAI pointer with tenant | | |
| 3: Agent Platform | openid_federation | DAI inline | | |
| 4: Workload (SPIFFE) | | DAI inline | client_authorized_instance_issuer | |
| 5: Verified Claims | | DAI inline | | attribute_authority_authorized_issuer |

Common to all scenarios:

- The Resource Authorization Server publishes a Trust Policy
  declaring the Trust Methods it requires.
- Authority Holders (Subject Authorities, OAuth client owners,
  attribute authorities) publish their delegation artifacts
  through the appropriate channel.
- Verification at the RAS follows the cross-category combination
  rule: AND across applicable categories, OR within a category.

# Mapping to Specifications {#spec-mapping}

Section pointers into the family for each functional area
exercised by the scenarios:

- Trust Policy document format: {{TRUST-POLICY}} §Trust Policy
  Document.
- RAS verification procedure: {{TRUST-POLICY}} §Resource
  Authorization Server Processing.
- Subject Authority extraction: {{TRUST-POLICY}} §Subject
  Authority Determination.
- DAI DNS record format: {{DAI}} §DNS Record.
- DAI HTTPS document format: {{DAI}} §Issuer Authorization
  Policy Document.
- DAI lookup procedure: {{DAI}} §Lookup Procedure.
- DAI tenant binding: {{DAI}} §Single-Issuer Multi-Tenant
  Identity Providers.
- CIMD `instance_issuers` member: {{CIA}} and {{CIMD}}.
- SPIFFE integration with Client Instance Trust:
  {{CLIENT-INSTANCE-TRUST}} §Working with SPIFFE/SPIRE.
- OIDC4IDA verified_claims integration:
  {{ATTRIBUTE-AUTHORITY-TRUST}} §Relationship to eKYC-IDA
  Verified Claims and §OIDC4IDA verified_claims Shape.
- AAAP DNS+HTTPS publication:
  {{ATTRIBUTE-AUTHORITY-TRUST}} §Publication.
- TPD DNS record format: {{TPD}} §DNS Record Format.
- TPD lookup procedure: {{TPD}} §Lookup Procedure.
- TPD composition with Trust Policy and Token Exchange Discovery:
  {{TPD}} §Relationship to Token Exchange Discovery.

# Security Considerations

This document is non-normative. The scenarios it describes
exercise the normative security considerations of the family:

- The cross-category combination rule prevents weaker-category
  evidence from substituting for stronger-category evidence (see
  {{TRUST-POLICY}} §Resource Authorization Server Processing).
- Tenant binding in DAI is a wire-format expression of trust in
  the IdP's tenant-isolation enforcement, not a cryptographic
  guarantee (see {{DAI}} §Single-Issuer Multi-Tenant Identity
  Providers).
- Workload identity authorities (SPIFFE trust domains, cloud
  IMDS, hardware-rooted attestation) compose with Client
  Instance Trust without specifying which mechanism is used; the
  OAuth client owner's choice of which authority to authorize
  is the load-bearing decision (see
  {{CLIENT-INSTANCE-TRUST}} §Working with SPIFFE/SPIRE).
- Attribute authority authorization is per-attribute-per-authority;
  an IdP authorized by one attribute authority cannot synthesize
  claims under a different attribute authority (see
  {{ATTRIBUTE-AUTHORITY-TRUST}} §Security Considerations).

For threat-model and adversary-class analysis, see
{{AUTHORITY-DELEGATION}} §Threat Model.

For alignment with OAuth Security BCP, see {{TRUST-POLICY}}
§Alignment with OAuth Security BCP and {{RFC9700}}.

# IANA Considerations

This document has no IANA actions.

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
