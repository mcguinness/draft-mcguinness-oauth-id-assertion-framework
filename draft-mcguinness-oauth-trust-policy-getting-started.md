---
title: "Getting Started with the OAuth Authority Delegation Family"
abbrev: "OAuth Trust Policy Getting Started"
docname: draft-mcguinness-oauth-trust-policy-getting-started-latest
category: info
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - trust policy
  - getting started

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

informative:
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
  TPD:
    title: "OAuth Trust Policy Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-trust-policy-discovery/
    date: false
  CLIENT-INSTANCE-TRUST:
    title: "Client Instance Trust Method for OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-trust/
    date: false
  ATTRIBUTE-AUTHORITY-TRUST:
    title: "Attribute Authority Trust Method for OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-attribute-authority-trust/
    date: false
  TRUST-POLICY-SCENARIOS:
    title: "Deployment Scenarios for the OAuth Identity Assertion Trust Policy Family"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-trust-policy-scenarios/
    date: false

---

--- abstract

This document is a non-normative entry point to the OAuth
Authority Delegation family of specifications. It states the
problem the family solves, identifies the smallest deployable
unit, and directs the reader to the next document. It does not
add any normative content of its own.

--- middle

# The Problem

OAuth deployments increasingly accept identity assertions from
issuers that were not individually configured in advance. Two
trust questions must be answered before such an assertion can be
relied on:

1. Is the issuer authentic (signed by an entity in a recognized
   ecosystem)?
2. Is the issuer authorized by the relevant namespace owner to
   assert about subjects in that namespace?

Today's deployments answer the first question with federation
membership and the second with bilateral configuration. Bilateral
configuration does not scale to open-world deployments where
agents, AI runtimes, and cross-organization assertions discover
resources at runtime.

This family defines wire-format mechanisms answering both
questions, with policy publication mirroring the DNS-based
authority patterns operators already use (CAA, MTA-STS, SPF, DKIM).

# The Smallest Deployable Unit

The minimum deployment a Resource Authorization Server needs is:

1. **Publish a Trust Policy document** at
   `https://{ras}/.well-known/identity-assertion-trust-policy`
   declaring which Trust Methods are required.

2. **For each customer (Subject Authority) you want to accept
   assertions from**: ask the customer to publish a DAI record
   listing your accepted issuer at
   `_oauth-issuer-policy.{customer-domain}`.

3. **Implement the Trust Method evaluation procedure** at your
   token endpoint.

That is the entire MVP. Three operational artifacts (Trust Policy
JSON, DAI DNS record, evaluation code). The four other documents
in the family extend, discover, or document this core.

A minimum working example appears in {{TRUST-POLICY-SCENARIOS}}
§Scenario 1: Workforce SSO into Multi-Vendor SaaS.

# Which Document to Read

A 30-minute reading path:

| Reader | Read first | Read next |
|-|-|-|
| Architect understanding the model | {{AUTHORITY-DELEGATION}} §The Delegation Question | {{TRUST-POLICY}} §Trust Policy Document |
| Implementer building a Resource Authorization Server | {{TRUST-POLICY}} §Resource Authorization Server Processing | {{DAI}} §Lookup Procedure |
| Customer publishing a DAI policy | {{DAI}} §Issuer Authorization Policy Document, §DNS Record | {{DAI}} §Operational Considerations |
| Identity Provider integrating | {{TRUST-POLICY}} §Grant Profile and Token Bindings | (No further reading needed for federation IdPs) |
| Operator evaluating | {{TRUST-POLICY-SCENARIOS}} | {{AUTHORITY-DELEGATION}} §Threat Model |
| Researcher / WG reviewer | {{AUTHORITY-DELEGATION}} (full) | {{TRUST-POLICY}} §Trust Methods + selected leaf profiles |

# The Family at a Glance

The family is organized in two layers with companion documents:

## Core (Standards Track)

- **{{AUTHORITY-DELEGATION}}**: the abstract Authority Delegation
  Pattern. Reads as a vocabulary + threat model + profile
  registry. Read once to understand the model.
- **{{TRUST-POLICY}}**: the OAuth-side framework. Defines the
  Trust Policy document, Trust Method categories, combination
  rule, and grant-profile bindings. Required reading for
  implementers.
- **{{DAI}}**: the canonical Trust Method profile. DNS+HTTPS
  publication of which issuers a namespace owner authorizes.
  Required reading for deployers.
- **{{TPD}}**: discovery companion. Lets a Resource Owner
  publish its Trust Policy at a DNS-named authority for
  bilateral first-contact discovery.

## Standards-Track Extensions

- **{{CLIENT-INSTANCE-TRUST}}**: workload-to-client binding
  (third Trust Method category). Read if your deployment binds
  OAuth clients to specific runtime instances (SPIFFE, K8s
  service accounts, cloud IMDS).

## Informational Extensions

- **{{ATTRIBUTE-AUTHORITY-TRUST}}**: claim-type authority
  publication. Read if your deployment carries OIDC4IDA
  verified claims or comparable attribute attestations.
  Informational pending validated deployment demand.

## Companions (Informational)

- **{{TRUST-POLICY-SCENARIOS}}**: five end-to-end worked
  examples spanning the family. The fastest way to understand
  the family in concrete terms.
- **This document**: the entry-point guide. You are reading it.

# What the Family Does NOT Do

To set expectations clearly:

- **It does not define a new OAuth grant type.** Identity
  assertions are issued by existing OAuth assertion-bearer grant
  profiles (RFC 7521/7523, ID-JAG). This family defines how to
  decide whether the issuer is acceptable.
- **It does not address mission governance or agent action
  authority.** Those questions (whether the user approved an
  agent to perform a specific action) are out of scope. The
  family answers identity-assertion-issuer trust, which is the
  substrate other authority layers build on.
- **It does not address session management, step-up
  authentication, or identity proofing.** These compose with the
  family but are governed by other specifications.
- **It does not require deploying every document.** A simple
  deployment may use only the parent's pattern (informally),
  Trust Policy, and DAI. The other documents address specific
  needs.

# Choosing What to Deploy

A decision flow for a Resource Authorization Server:

| Need | Profile to deploy |
|-|-|
| Accept assertions from federated issuers about any user | {{TRUST-POLICY}} with `openid_federation` Trust Method only |
| Accept assertions about users in customer namespaces (per-customer config) | Add {{DAI}}'s `domain_authorized_issuer` Trust Method |
| Discover trust policies in open-world (agent / first-contact deployments) | Publish {{TPD}} records |
| Bind tokens to workload runtime identity | Add {{CLIENT-INSTANCE-TRUST}} |
| Accept OIDC4IDA verified attribute claims | Add {{ATTRIBUTE-AUTHORITY-TRUST}} |

A typical enterprise deployment uses Trust Policy + DAI. A more
advanced deployment also publishes TPD. Workload-identity-aware
deployments add Client Instance Trust. Identity-assurance-aware
deployments add Attribute Authority Trust.

# Position in the IAM Landscape

The family addresses one specific question in the broader IAM
landscape:

| IAM concern | Addressed by |
|-|-|
| Authentication of users | Out of scope (OIDC, SAML, FIDO) |
| Provisioning lifecycle | Out of scope (SCIM, JIT provisioning) |
| Issuer trust at OAuth boundary | **This family** |
| Token format and validation | Out of scope (RFC 7515-7519, ID-JAG) |
| Resource access policy | Out of scope (local policy, OPA, Cedar) |
| Session management | Out of scope |
| Mission governance / agent action authority | Out of scope (under active separate work) |
| Continuous evaluation | Out of scope (CAEP, SSE) |

The family is the substrate that lets other IAM layers
trust-evaluate incoming identity assertions.

# Compliance Mapping

The family does not claim certifiable compliance with specific
regimes, but the profiles compose with several common ones:

| Regime | Composition |
|-|-|
| SOC 2 CC6.1 (Access Control) | A documented Trust Policy with auditable trust-method evaluation supports control documentation |
| ISO 27001 A.5.18 / A.9.4 (Information Access) | Trust Policy decisions are loggable per {{TRUST-POLICY}} §Observability and §Trust Evaluation Diagnostic Reasons |
| NIST 800-63A (Identity Proofing) | {{ATTRIBUTE-AUTHORITY-TRUST}} composes with OIDC4IDA verified-claims for IAL assertions |
| NIST 800-63B (Authenticator Assurance) | Out of scope; the family does not address authentication strength |
| eIDAS Trust Framework | {{ATTRIBUTE-AUTHORITY-TRUST}} accommodates eIDAS as an Attribute Authority |

# Next Steps for Deployers

A minimum-viable deployment in three steps:

1. **Decide your Trust Methods.** Most deployments start with
   `domain_authorized_issuer`. Federation members add
   `openid_federation`.

2. **Publish your Trust Policy.** A JSON document at
   `https://{ras}/.well-known/identity-assertion-trust-policy`
   listing your accepted Trust Methods. {{TRUST-POLICY}}
   §Trust Policy Document defines the exact format. A minimal
   policy is ~10 lines of JSON.

3. **Onboard your first customer.** Ask them to publish a DAI
   record at `_oauth-issuer-policy.{their-domain}` listing your
   accepted issuer. Test the lookup procedure.

A reference implementation companion repository (when available)
shortens this from days to hours. Until reference implementations
exist, expect to write the Trust Policy parser, DAI lookup
procedure, and validation logic from scratch.

# What This Document is Not

This document is informational and entry-point only. It does not
add normative requirements, define new wire formats, or change
the semantics of the family. The normative documents listed in
§The Family at a Glance remain authoritative.

# IANA Considerations

This document has no IANA actions.

# Security Considerations

This document has no security considerations of its own. See the
referenced family documents and {{AUTHORITY-DELEGATION}} §Threat
Model.

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
