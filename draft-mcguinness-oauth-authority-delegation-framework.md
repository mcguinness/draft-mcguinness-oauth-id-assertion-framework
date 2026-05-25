---
title: "OAuth Authority Delegation Framework"
abbrev: "Authority Delegation Framework"
docname: draft-mcguinness-oauth-authority-delegation-framework-latest
category: info
submissiontype: IETF
v: 3

ipr: trust200902
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
  - OAuth
  - authority delegation
  - trust framework
  - issuer trust
  - identity assertion

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
  RFC8126:

informative:
  RFC2693:
  RFC6480:
  RFC7521:
  RFC7523:
  RFC8461:
  RFC8659:
  RFC8693:
  RFC9334:
  RFC9345:
  MACAROONS:
    title: "Macaroons: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud"
    target: https://research.google/pubs/pub41892/
    date: 2014
    author:
      - ins: A. Birgisson
      - ins: J. G. Politz
      - ins: U. Erlingsson
      - ins: A. Taly
      - ins: M. Vrable
      - ins: M. Lentczner
  ACTOR-PROFILE:
    title: "OAuth Actor Profile for Delegation"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-profile/
    date: false
  OIDF-FEDERATION:
    title: "OpenID Federation 1.0"
    target: https://openid.net/specs/openid-federation-1_0.html
  TRUST-POLICY:
    title: "OAuth Identity Assertion Issuer Trust Policy"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-identity-assertion-issuer-trust-policy/
    date: false
  DAI:
    title: "OAuth Domain-Authorized Issuer Discovery"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer-discovery/
    date: false
  CIA:
    title: "OAuth 2.0 Client Instance Assertions using Actor Tokens"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-client-instance-assertion/
    date: false
  ATTEST-CLIENT-AUTH:
    title: "OAuth 2.0 Attestation-Based Client Authentication"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/
    date: false
  I-D.ietf-oauth-identity-chaining:
    title: "OAuth Identity and Authorization Chaining Across Domains"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/
    date: false
  I-D.ietf-oauth-transaction-tokens:
    title: "Transaction Tokens"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/
    date: false
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false

---

--- abstract

In OAuth ecosystems, a Resource Authorization Server frequently needs
to decide whether some other party (an authorization server, an issuer,
a workload, an attester) is entitled to assert a claim that the
Resource Authorization Server will rely on. That entitlement comes from
a delegation: an Authority Holder with authority over a namespace,
claim type, or scope has granted the asserting party the right to
speak.

This document defines the Authority Delegation Pattern: an abstract
model in which an Authority Holder delegates authority to a Delegate,
the Delegate produces an assertion, and a Validator validates both the
delegation and the assertion before relying on the claim. The pattern
is realized by concrete profiles that specify what is delegated, how
the delegation is published, and how it is verified.

This document does not define a wire format. It defines vocabulary,
actor roles, a verification model, and conformance guidance for
specifications that choose to profile the pattern. Existing OAuth
ecosystem mechanisms can be analyzed using this vocabulary, and future
specifications can profile the pattern explicitly for workload
identity, transaction context, and other delegated-authority
deployments.

--- middle

# Introduction

OAuth deployments increasingly require Resource Authorization Servers,
clients, and resource servers to decide whether a counterparty is
entitled to make a particular claim. The question is not just "is the
JWT signed by a recognized key?" but "is the signer authorized to
assert this claim?". Examples:

- An OAuth authorization server accepts an identity assertion JWT.
  Whose authority gave the asserter the right to claim
  `alice@example.com`?

- An OAuth client presents an instance attestation. Whose authority
  gave the attester the right to bind that workload to this
  `client_id`?

- A federation member presents a token. Whose authority placed the
  member in the federation, and which intermediate authority's
  metadata policy applies?

- A Certification Authority issues a TLS certificate. Whose authority
  authorized that CA to issue for this domain?

In every case there is an underlying delegation: an entity with
authority over some scope (a namespace, a claim type, a registration)
has granted another entity the right to assert within that scope. The
Validator of the assertion does not need to trust the asserter
intrinsically; it needs to validate the delegation.

This document names that pattern and defines the abstract model.

## The Delegation Question

A Validator evaluating an assertion answers two distinct questions:

1. *Is the assertion well-formed and authentic?* Signature, audience,
   expiration, replay protection.

2. *Is the asserter authorized to make this claim?* Has the Authority
   Holder for the claim's scope delegated to this asserter?

The first question is grant-profile and JWT-validation work. The
second question is the subject of this document.

The first question conflates with the second when the framework lets a
signer's "authentic" status stand in for "authorized." A federated
authorization server, validated by its federation membership, can mint
identity assertions naming any user in any email domain unless the
framework separately requires that the email-domain owner has
delegated to it. Conflating signer authenticity with delegation
authority lets any federation member impersonate users in namespaces
they have no authority over.

This document defines the abstract pattern that prevents that
conflation, and the profile requirements that make concrete
realizations of the pattern interoperable.

## Relationships to Existing Work {#relationships}

This document is a pattern that concrete specifications can profile.
It does not replace any of the mechanisms listed below. The mappings
are non-normative and are included to show the shared structure.
Full mappings of each mechanism appear in {{appendix-profiles}}.

### OAuth Profile Family {#oauth-profile-family}

This document is the parent of an OAuth profile family for
namespace-authority delegation:

- **{{TRUST-POLICY}}** profiles this pattern for OAuth identity
  assertions, defining the Trust Policy document, Trust Method
  machinery, Subject Authority Determination, and grant-profile
  bindings.
- **{{DAI}}** specifies a DNS+HTTPS Subject-Authority publication
  mechanism consumed by trust-policy as two Trust Methods
  (`domain_authorized_issuer`, `https_authorized_issuer`).

Both register in the Authority Delegation Profile registry
({{iana-profile-registry}}).

### Other OAuth-Ecosystem Specifications

The following OAuth specifications follow or relate to this
pattern; they predate this document and are not modified by it:

- {{CIA}}: client instance binding.
- {{ATTEST-CLIENT-AUTH}}: attestation-based client authentication
  (closed-world).
- {{OIDF-FEDERATION}}: chained transitive issuer authentication.
- {{RFC8659}} (CAA) and {{RFC8461}} (MTA-STS): non-OAuth
  publication-channel authority bindings.

Future specifications intentionally profiling the pattern can
register in the IANA Authority Delegation Profile registry
({{iana-profile-registry}}).

### OAuth Actor Profile {#actor-profile-relationship}

OAuth Actor Profile {{ACTOR-PROFILE}} defines the wire-format
representation of delegated actors via the `act` claim and
explicitly defers actor trust evaluation to local policy. Actor
Profile and Authority Delegation are complementary:

- **Actor Profile** specifies HOW delegation is REPRESENTED in a
  token (the `act` claim and nested actors).
- **Authority Delegation** specifies HOW the AUTHORITY behind that
  delegation is established and validated (which Authority Holder
  authorized `act.iss` to attest the `(act.iss, act.sub)` pair).

A token carrying `act` still leaves a Validator with the question
"Was `act.iss` authorized to attest this actor identity?" Authority
Delegation provides the framework for answering it. A future
specification MAY define an actor-identity profile of Authority
Delegation; {{TRUST-POLICY}} documents the gap an actor-identity
profile would close.

### RFC 7521 (OAuth Assertion Framework) {#rfc7521-relationship}

{{RFC7521}} defines an abstract model for assertion-based
authorization (Issuer, Subject, Audience, Authority, Relying Party);
concrete profiles ({{RFC7523}} for JWT, RFC 7522 for SAML)
instantiate it. {{RFC7521}} §3 notes that the Relying Party must
trust the Issuer but does not specify how that trust is established.

This document specifies the delegation layer beneath the assertion
layer: how an Authority Holder delegates to a Delegate (the
{{RFC7521}} Issuer), how the delegation is published and validated,
and how Validators (the {{RFC7521}} Relying Parties) compose
multiple independent trust evaluations.

### RATS Architecture {#rats-relationship}

The Remote ATtestation procedureS (RATS) Architecture {{RFC9334}}
defines a structurally related trust-evaluation model for hardware
and software attestation. Vocabularies differ; structures align:

| RATS role / artifact | Authority Delegation equivalent |
|-|-|
| Endorser | Authority Holder |
| Attester | Delegate |
| Verifier | Validator (validation function) |
| Relying Party | Validator (decision function) |
| Endorsements | Delegation Artifacts |
| Evidence | Assertions |
| Attestation Results | Validator's accept/reject decision |
| Verifier Owner | (Validator configurator; out of scope) |
| Reference Values | (no direct equivalent in OAuth contexts) |

The Authority Delegation Pattern applies a similar claim-authority
model to OAuth identity, namespace, client, and assertion contexts
rather than to hardware or software platform properties.
Specifications at the intersection (for example, OAuth-bound
attestation tokens for confidential-computing workloads) MAY
reference both this document and {{RFC9334}}.

## Scope

This document defines:

- The vocabulary and actor roles for authority delegation
  ({{terminology}}).
- The abstract Authority Delegation Pattern: actors, delegation
  artifact, and verification ({{pattern}}).
- Required semantic content for a Delegation Artifact, without
  prescribing wire format ({{artifacts}}).
- An abstract verification model that profiles instantiate
  ({{verification}}).
- The composition rules for combining multiple independent authority
  sources at a Validator ({{combination-rule}}).
- The profile requirements: what a profile of this pattern MUST
  specify ({{profile-requirements}}).
- A profile registry ({{iana-profile-registry}}).

This document does NOT define:

- A wire format for Delegation Artifacts. Profiles do that.
- A specific verification procedure. Profiles do that.
- A new JWT format, claim, or grant type.
- A specific OAuth client authentication method.
- A successor to any existing OAuth or OpenID specification.

## Non-Goals

- This document does NOT replace {{RFC7521}} (Assertion Framework)
  or change how assertions are presented in OAuth grant flows.
  {{RFC7521}} addresses the assertion plumbing; this document
  addresses the delegation behind the asserter.

- This document does NOT replace OpenID Federation or define a new
  federation protocol. {{OIDF-FEDERATION}} is a chained authority
  delegation mechanism that can be described using this pattern.

- This document does NOT prescribe whether delegation chains are
  permitted; depth-1 delegation and chained delegation are both
  valid profile choices, with security implications profiles MUST
  document ({{transitivity}}).

- This document does NOT extend trust to assertion validation,
  client authentication, or token format. Those remain
  grant-profile concerns.

## When to Use This Pattern

This pattern is useful when all of the following are true:

- A Validator relies on a claim made by an asserter.
- Some other party is authoritative for the scope of that claim.
- The Validator cannot practically enumerate all acceptable asserters
  through bilateral configuration.
- The authoritative party can publish, sign, or otherwise make
  available evidence of delegation.

This pattern is not necessary when the Validator's local configuration
is the entire source of trust, when the asserter and Authority Holder
are the same entity, or when the claim does not depend on a distinct
authority boundary.

## Delegation, Policy Publication, and Local Trust

This document distinguishes three related but different patterns:

- **Authority delegation.** An Authority Holder authorizes a distinct
  Delegate to assert claims within a scope. This document is primarily
  about this pattern.

- **Policy publication.** An Authority Holder publishes policy for its
  own namespace or service but does not authorize a separate Delegate.
  MTA-STS {{RFC8461}} is an example. Policy publication uses many of
  the same authority-binding techniques as delegation, but it is not
  itself delegation.

- **Local trust configuration.** A Validator locally configures a trust
  root, attester, or issuer without relying on an external Authority
  Holder's Delegation Artifact. This is important in OAuth deployments,
  but it is closed-world local policy rather than open-world authority
  delegation.

A profile of this pattern needs a distinct Authority Holder and
Delegate. Related mechanisms that lack one of those roles can still be
described with some of this vocabulary, but they are not full
Authority Delegation profiles.

## Conventions

{::boilerplate bcp14-tagged}

# Terminology {#terminology}

This document uses OAuth 2.0 {{RFC6749}} terminology. In addition:

Authority (as used in this document):
: The right to attest claims that a Validator will rely on, within a
  specified namespace, claim type, scope, or registration. This is
  distinct from "authority" in the access-management sense (the
  right to perform actions or access resources): a Delegate that
  has received delegated attestation authority does not thereby
  gain rights to act on the Authority Holder's behalf, only the
  right to speak about specified claims. The IAM access-rights
  sense of "authority" is out of scope for this document.

Authority Holder:
: An entity holding authority (as defined above) over a namespace,
  claim type, scope, or registration. An Authority Holder MAY
  delegate that attestation authority to one or more Delegates.
  Examples: a DNS domain owner for an email namespace; a
  Certification Authority for certificate-issuance policies; an
  OAuth client owner for instances acting as that client; a
  federation operator for federation membership; an HR system for
  employment-status attestations.

Delegate:
: An entity that has received attestation authority from an
  Authority Holder and may issue Assertions within the delegated
  scope. Profiles define the specific terms used for Delegates in
  their context (Assertion Issuer, Instance Issuer, Client
  Attester, federation member, etc.).

Delegation:
: The grant of attestation authority from an Authority Holder to a
  Delegate over a specified scope and claim type, optionally
  bounded in time and optionally constrained by additional
  conditions.

Delegated Claim:
: The class of claim a Delegate is authorized to make within the
  delegated scope. Examples include "this authorization server may
  assert email subjects in `example.com`", "this Instance Issuer
  may attest runtime instances of `client_id` X", "this HR system
  may attest current-employee status for `example.com`", and "this
  federation intermediate may issue Subordinate Statements for
  entities in its sector."

Delegation Artifact:
: A profile-defined representation of a delegation. The form may be
  a signed document, a DNS record, a JSON document at a well-known
  HTTPS URL, an entry in published metadata, or any other
  profile-defined publication form. The Delegation Artifact MUST
  carry sufficient information to identify the Authority Holder,
  the Delegate, the delegated claim type or scope, and (when
  applicable) the validity bounds.

Assertion:
: A signed statement made by a Delegate about a subject within the
  delegated scope. In OAuth contexts this is typically a JWT
  {{RFC7519}}.

Principal:
: An authenticated entity participating in an OAuth flow. The
  Subject is often a Principal (a human user, a service account,
  a workload). The Authority Holder and the Delegate are also
  typically Principals when they participate in OAuth flows
  directly, but the pattern does not require them to be: an
  Authority Holder may be represented only by its publication
  channel (a DNS domain, an HTTPS origin) without itself signing
  into any OAuth flow.

Validator:
: The component that performs the trust evaluation: retrieves the
  Delegation Artifact, validates the Assertion, applies the
  combination rules, and produces an accept-or-reject result. In
  OAuth flows the Validator is typically the Resource
  Authorization Server (when accepting identity assertions), the
  authorization server (when accepting client attestations), or
  the resource server (when validating an access token). The
  Validator and Relying Party are often co-located but are
  conceptually distinct, paralleling the analogous distinction in
  the RATS Architecture ({{RFC9334}}).

Relying Party:
: The entity that acts on the Validator's result: grants access,
  issues a token, applies policy. The Validator answers "is this
  Assertion acceptable?"; the Relying Party answers "given that it
  is acceptable, what do I do?". The Relying Party's decision is
  governed by local policy and is out of scope for this document.

Profile (of this pattern):
: A concrete specification that realizes the Authority Delegation
  Pattern by defining the Delegation Artifact format, the
  publication channel, the verification procedure, and any OAuth
  integration points. Examples are listed in
  {{appendix-profiles}}. This usage of "profile" is distinct from
  the SAML/OAuth assertion-flow sense of the word; this document
  uses "profile" exclusively in the pattern-realization sense.

Authority Source:
: A trust root, registry, publication channel, or configured source
  from which a Validator accepts Delegation Artifacts. In many
  profiles the Authority Source and Authority Holder are the same
  entity (for example, a domain publishing its own DNS record). In
  other profiles they differ: a federation trust anchor can be the
  Authority Source for Subordinate Statements issued by
  intermediates, and local Validator configuration can be the
  Authority Source for trusted attesters.

  For example, in an OpenID Federation chain the Validator may
  trust a federation trust anchor as an Authority Source, while an
  intermediate federation entity acts as the Authority Holder for
  a specific Subordinate Statement about a leaf entity. The Trust
  Anchor is the root of trust; the intermediate is the issuer of
  the particular Delegation Artifact.

Subdelegate (optional):
: An entity that has received delegated authority from another
  Delegate, where the originating delegation permits further
  delegation. Profiles MAY permit subdelegation (producing
  delegation chains) or restrict delegation to depth one; the
  trade-offs are documented in {{transitivity}}.

# The Authority Delegation Pattern {#pattern}

This section describes the abstract pattern. Profiles instantiate it
concretely.

## Actors and Relationships {#actors}

Four actors participate in the pattern:

- **Authority Holder**: holds authority over some scope.
- **Delegate**: receives delegated authority and produces
  Assertions.
- **Validator**: validates Delegation Artifacts and Assertions before
  relying on them.
- **Subject** (the entity the Assertion is about): may or may not
  participate in the protocol but is the entity the delegation
  concerns.

The relationships are:

~~~
Authority Holder
   ↓  delegates authority via
Delegation Artifact
   ↓  identifies
Delegate
   ↓  signs
Assertion
   ↓  presented to
Validator
~~~

The Validator validates the Delegation Artifact (which says "Authority
Holder authorized Delegate") and the Assertion (which says "Delegate
asserts something about Subject"). Both validations are required.
Neither alone justifies acceptance.

## The Delegation Lifecycle {#lifecycle}

A delegation has a lifecycle:

1. **Establishment**: the Authority Holder creates a Delegation
   Artifact identifying the Delegate, the delegated scope, the
   claim types in scope, and optionally validity bounds.

2. **Publication**: the Authority Holder publishes the Delegation
   Artifact through a profile-defined channel. The publication
   channel is itself an authority binding: control of the
   publication channel is what makes the Authority Holder the
   Authority Holder. Examples of publication channels include DNS
   records under the Authority Holder's domain, HTTPS documents at
   a well-known URL on the Authority Holder's host, signed
   subordinate statements in a federation, and entries in
   authoritative client metadata.

3. **Use**: the Delegate produces Assertions within the delegated
   scope. The Validator retrieves the Delegation Artifact and
   validates the Assertion against it.

4. **Revocation**: the Authority Holder removes or updates the
   Delegation Artifact, or the artifact's validity bound expires.
   Revocation latency is bounded by the profile's cache lifetime
   model; this document does not require a specific latency.

## Independent Trust Evaluation Categories {#categories}

A Validator typically combines two independent trust evaluations
to accept an Assertion. The framework distinguishes these as
categories. Each category answers a distinct trust question:

- **Authenticity**: is the entity that signed the Assertion
  cryptographically authentic and recognized as a member of some
  ecosystem? (Federation membership, trust-mark issuance, signed
  attester key.) Satisfied by evidence that signers and their keys
  belong to a recognized population.

- **Delegation authority**: has the Authority Holder for the
  asserted scope delegated to this signer? (Subject namespace
  delegation, client-instance delegation, attribute authority
  delegation.) Satisfied by a Delegation Artifact from the
  appropriate Authority Holder.

Profiles register Delegation Artifacts in the delegation-authority
category. Profiles MAY also register authenticity-category evidence.

Local policy (the Relying Party's discretion: risk scoring, scope
grants, account linking, business rules) applies on top of these
two categories but is not itself part of the pattern's category
structure. Trust framework evaluation is necessary but not
sufficient; the Relying Party's local policy is the final decision
layer.

### Cross-Category Combination Rule {#combination-rule}

When a profile defines multiple independent trust-evaluation
categories and more than one category is applicable to a request, the
Validator MUST require at least one satisfying evidence item from EACH
applicable category. Within a single category, OR-semantics apply:
satisfying any one applicable evidence item is sufficient. The rule is
AND across independent categories, OR within a category.

Category applicability is a property of the Validator's Published
Trust Policy, not of the incoming Assertion. If the Validator's
local policy declares that it implements a profile in a given
category (for example, namespace authority validation), that
category is applicable to every Assertion the Validator evaluates
within the profile's scope. The Assertion itself MUST NOT be able
to waive the category: an Assertion lacking satisfying evidence
for an applicable category MUST be rejected. Profiles MUST NOT
define applicability conditions that depend on properties of the
Assertion under evaluation (for example, "this category is not
applicable when the Assertion is from a legacy issuer" or "this
category is skipped when the Assertion carries a `legacy: true`
claim"). Assertion-driven applicability waivers create the
Applicability Bypass risk discussed in
{{applicability-bypass}}.

Migration scenarios where some traffic legitimately predates a
category's availability are handled at the policy layer, not by
silently waiving the category: the Validator MAY maintain
separate trust policies for distinct namespaces, audiences, or
deployment phases, but within any one policy the applicable
categories MUST be exhaustively enforced.

Satisfying one category MUST NOT be treated as satisfying another.
A signer authenticated by federation membership has not been
delegated namespace authority by that fact; an Authority Holder's
delegation to a signer does not authenticate the signer's identity
beyond what the Delegation Artifact attests.

OR-within-category applies when multiple evidence items from the
same Authority Source's delegation satisfy the same category;
selection of WHICH Authority Source applies to a given Assertion
is deterministic ({{multiple-sources}}) and is not subject to
OR-semantics. When a Validator implements multiple profiles in the
same category from different Authority Sources, the resulting
OR-within-category composition creates a downgrade risk that
profile authors and deployers MUST analyze
({{profile-composition-risks}}).

Profiles MAY specify additional combination rules within their own
scope. A profile MUST NOT treat evidence for one independent category
as satisfying another independent category. A profile that combines
authenticity and authority in a single cryptographic artifact MUST
document why the artifact satisfies both questions.

### Multiple Authority Sources Within a Category {#multiple-sources}

A Validator MAY accept Delegation Artifacts from multiple Authority
Sources within the same category (for example, multiple Subject
Authorities for different namespaces; multiple federation trust
anchors). Selection of WHICH Authority Source's delegation applies
to a given Assertion is the most security-sensitive step in the
verification model: it happens BEFORE the Assertion is
authenticated against any delegation, so an attacker who can
influence selection chooses which Authority Holder's authority is
invoked.

#### Deterministic Source Selection {#single-binding-primitive}

Profiles MUST define deterministic source selection: a binding
function that maps each Assertion and relevant request context to
exactly one Authority Source, or fails deterministically. The binding
function MUST:

- Define its inputs explicitly, such as a single named claim, header
  field, request parameter, or a fixed tuple of such values.
- Produce exactly one Authority Source for any valid input, or fail
  deterministically with no fallback and no preference ordering across
  candidates.
- Be invariant under additional or attacker-controlled inputs outside
  the defined binding function: the presence, absence, or value of any
  other claim in the Assertion MUST NOT alter the selected Authority
  Source.

A profile that derives the Authority Source from ad hoc combinations
of claims, that establishes preference ordering across candidate
inputs, or that falls back from one selection input to another when
the primary is absent enables Unverified Claim Exploitation
({{unverified-claim}}): an attacker constructs an Assertion satisfying
multiple potential bindings and steers the Validator to a weaker
Authority Source's policy.

The binding function is the profile's `source_selection` matrix
element ({{profile-mapping-matrix}}); profiles MUST document the
function and SHOULD include a brief argument for why inputs outside
the function cannot manipulate the selected Authority Source.

#### No Fallback on Failure

The Validator MUST NOT fall through to a different Authority Source
if the originally-applicable Authority Source's evaluation fails,
is indeterminate, or yields a Negative state
({{exception-handling}}). Falling through to a fallback Authority
Source breaks the fail-closed property and creates an
availability-driven downgrade.

## Open-World and Closed-World Delegation

The pattern accommodates both open-world and closed-world
deployments without distinction at the structural level. The
difference is who acts as Authority Holder relative to the
Validator's configuration:

- **Open-world delegation**: the Authority Holder is an entity
  independent of the Validator (a customer's domain, a federation
  operator, an OAuth client owner). The Validator retrieves
  Delegation Artifacts through the profile's publication channel
  at evaluation time. The acceptable set of Delegates is large,
  dynamic, customer-controlled, or otherwise not enumerable in
  advance.

- **Closed-world delegation**: the Authority Holder and the
  Validator's configuration administrator are the same party. The
  Validator's local configuration IS the Delegation Artifact, and
  the publication channel is internal configuration management.
  This is the degenerate case in which the pattern reduces to
  bilateral registration.

The same pattern describes both. A profile MAY target either or
both: {{TRUST-POLICY}} is open-world; {{ATTEST-CLIENT-AUTH}} is
closed-world (the Authorization Server configures trust in a Client
Attester out-of-band); some profiles support both via different
publication channels.

Open-world delegation does not mean open acceptance. The Validator
still publishes (or locally configures) which Authority Sources it
accepts, which profiles it implements, and which Delegation
Artifacts it considers valid.

## Bounded vs Chained Transitivity {#transitivity}

Some profiles permit Subdelegation, producing delegation chains
(Authority Holder → Delegate → Subdelegate → ... → final asserter).
Other profiles restrict delegation to depth one (Authority Holder →
Delegate; no further delegation possible).

Both are valid profile choices. They have different security
properties:

- **Bounded depth-1 delegation** (e.g., {{TRUST-POLICY}}, CAA):
  the Validator evaluates a single Delegation Artifact. Revocation
  latency is bounded by one cache. A compromise at any party other
  than the Authority Holder cannot expand the set of authorized
  Delegates. The cost is loss of administrative flexibility:
  Authority Holders must directly list every authorized Delegate.

- **Chained delegation** (e.g., {{OIDF-FEDERATION}}): the Validator
  walks a chain of Delegation Artifacts. Each link has its own
  validity, cache lifetime, and revocation path. The Authority
  Holder delegates administrative responsibility to intermediates,
  which delegate further. The cost is more complex revocation
  semantics, broader attack surface across multiple parties, and
  the need to enforce metadata-policy-style constraints to prevent
  Subdelegates from over-claiming.

A profile MUST specify whether it permits subdelegation. A profile
that permits subdelegation MUST specify how Subdelegate scope is
constrained relative to the Authority Holder's original delegation.

Attenuated delegation ({{attenuated-delegation}}) is the most
permissive form of subdelegation: the token format permits any
holder of an attenuable token to further attenuate it without
consulting the original Authority Holder. The bounded form
(attenuation may only narrow) prevents expansion of scope;
profiles using attenuation MUST specify how attenuation primitives
enforce monotone narrowing.

## Attenuated Delegation {#attenuated-delegation}

The pattern above describes Validators retrieving Delegation
Artifacts from external publication channels at evaluation time.
An alternative architecture carries the delegation evidence within
the token itself: the Authority Holder issues an attenuable
credential, subsequent holders may further narrow (attenuate) its
scope, and a Validator at use time verifies the cumulative chain
cryptographically without retrieving any external artifact. This
is the capability-token architecture.

Macaroons {{MACAROONS}}, Biscuit tokens, SPKI/SDSI ({{RFC2693}}),
UCAN, the W3C Authorization Capabilities specification (zcap-ld),
and OAuth Token Exchange ({{RFC8693}}) with downscoping all
realize variations of this architecture.

This pattern accommodates attenuated delegation as a form of
delegation where the publication channel IS the token: the
Authority Holder publishes the initial delegation by issuing the
token; subsequent attenuators publish their attenuations by
adding caveats and forwarding the token; the Validator extracts
the entire delegation chain from the presented Assertion. The
Authority Delegation vocabulary applies: the initial issuer is
the Authority Holder, subsequent attenuators are Subdelegates
constrained to narrow (never broaden) the original delegation,
and the chain itself is the Delegation Artifact.

The two architectures trade different risks:

| Property | Hop-by-hop evaluation | Attenuated delegation |
|-|-|-|
| Where delegation lives | External publication channel | Carried inside the token |
| External calls at use | Required (with caching) | None |
| Revocation latency | Bounded by cache lifetime | Bounded by token TTL |
| Availability of delegation source | Required at use time | Not required at use time |
| Trust openness | Open-world (any Authority Holder publishes) | Often closed-world (chain root must be recognized) |
| Cross-hop scope narrowing | Re-evaluation may apply tighter constraints | Holder attenuates the token directly |
| Audit | Per-hop logs; no end-to-end token view | End-to-end chain visible in the token |
| Token size | Compact | Grows with chain depth |

Profiles MAY use either architecture but MUST document the
choice and its consequences. The choice typically follows the
deployment shape:

- Tight revocation latency or open-world delegation favors
  hop-by-hop evaluation.
- Low-latency, offline-capable, or same-organization chained
  flows favor attenuated delegation.
- Mixed flows are possible: an attenuated token at one hop may
  be presented to a Validator that ALSO performs hop-by-hop
  evaluation of a separate delegation in another category. The
  cross-category combination rule of {{combination-rule}}
  applies unchanged.

## Multi-Hop Trust Evaluation {#multi-hop}

The pattern is described above as a single delegation evaluated at
a single Validator. Real OAuth flows often involve multiple
delegation evaluations at different protocol points: an identity
assertion is evaluated at one authorization server, the resulting
access token is evaluated at a resource server, and an actor object
in the token may carry yet another delegation evaluation.

The pattern accommodates multi-hop flows by composition: each hop
is an independent Authority Delegation evaluation, governed by the
combination rule of {{combination-rule}}. Satisfying delegation
evaluation at one hop does not propagate to other hops; each hop's
Validator evaluates its own applicable Delegation Artifacts.

This separation has two important consequences:

- Trust does not accrete across hops. A Delegate that received
  authority over a constrained scope at hop 1 cannot expand that
  scope at hop 2 by re-wrapping the assertion under its own
  authority; the Validator at hop 2 evaluates its own delegations
  independently.

- The chain's overall trustworthiness is bounded by its weakest
  hop. Profile authors deploying multi-hop flows MUST analyze each
  hop separately and reason about the cumulative attack surface.

OAuth Identity Chaining ({{I-D.ietf-oauth-identity-chaining}}) is
a worked example of multi-hop trust evaluation; see
{{identity-chaining-example}} for a mapping of Identity Chaining
flows onto composed Authority Delegation evaluations. Two specific
failure modes that profile authors of multi-hop flows MUST consider
are identity laundering ({{identity-laundering}}) and context
token laundering ({{context-token-laundering}}).

# Delegation Artifacts {#artifacts}

A Delegation Artifact is the concrete representation of a
delegation. Profiles define its wire format. This section specifies
the required SEMANTIC content; profiles MAY encode this content in
any form (JSON, DNS records, JWT, X.509, custom).

## Required Semantic Content

Every Delegation Artifact MUST identify, by name or value, the
following:

`authority_holder`
: The identity of the Authority Holder making the delegation. The
profile MUST define how this identifier is bound to the publication
channel or signature that authenticates the artifact. For
DNS-published artifacts this is often a domain; for federation
Subordinate Statements this is the federation entity identifier of
the issuer; for CIMD-published artifacts this is the client metadata
issuer or URL.

`delegate`
: The identity of the Delegate (or list of Delegates) receiving the
delegated authority. For OAuth contexts this is typically an HTTPS
URL (OAuth issuer identifier), a SPIFFE ID, or a federation entity
identifier.

`scope`
: The scope of the delegation. A profile MUST specify the scope
vocabulary it uses. Common scope kinds, any combination of which a
profile MAY use:

  - *Subject scope*: a specific Subject or set of Subjects (e.g.,
    `alice@example.com`).
  - *Namespace scope*: all Subjects within a namespace (e.g., all
    emails in `example.com`).
  - *Attribute scope*: a class of attributes about Subjects (e.g.,
    employment status, degree completion, KYC level).
  - *Resource scope*: a class of resources (e.g., a specific OAuth
    `client_id`, a workload identity authority's trust domain).

A single Delegation Artifact MAY combine multiple scope kinds
(e.g., "attest degree status for any Subject whose original
enrollment was in `university.example`"). Profiles MUST define
how the combined scopes are interpreted and resolved at
validation time.

`delegated_claims`
: The kinds of claims the Delegate is authorized to make within the
scope. Optional in profiles where the delegated claim type is
implicit in the Delegation Artifact's purpose; explicit in profiles
where one Delegation Artifact may cover multiple claim types.
Common delegated-claim categories include identity claims, attribute
claims, capability claims, and instance-binding claims.

## Optional Semantic Content

Profiles MAY specify additional semantic content:

`valid_from` / `valid_until`
: Validity window for the delegation. A Delegation Artifact without
explicit bounds is valid from its publication time until it is
removed or revoked.

`conditions`
: Profile-specific conditions on the delegation (for example, a
tenant binding restricting the delegation to one tenant of a
shared-issuer Delegate).

`subdelegation_constraints`
: For profiles permitting subdelegation, constraints on what the
Delegate may further delegate.

`signature`
: A cryptographic signature over the Delegation Artifact when the
publication channel does not by itself authenticate the Authority
Holder.

## Publication Channels

A Delegation Artifact's publication channel is part of its trust
model. The channel binds the artifact to the Authority Holder:
whoever can publish through the channel IS the Authority Holder for
purposes of the delegation. Common channels:

- **DNS at a well-known owner name under the Authority Holder's
  domain.** Used by CAA, MTA-STS, the parent draft's DAI inline
  form, and SPF/DKIM. Authority binding is DNS control.

- **HTTPS at a well-known URL on the Authority Holder's host.**
  Used by MTA-STS policies, the parent draft's DAI HTTPS form,
  CIMD documents. Authority binding is TLS-authenticated origin
  control.

- **Embedded in published metadata controlled by the Authority
  Holder.** Used by {{CIA}}'s `instance_issuers` field embedded in
  CIMD. Authority binding is whatever authenticates the metadata
  document.

- **Signed subordinate statements in a federation tree.** Used by
  OpenID Federation. Authority binding is the cryptographic chain
  rooted at a trust anchor.

- **Out-of-band configuration at the Validator.** Used by
  attestation-based client authentication where the Validator
  trusts a configured Attester key. Authority binding is the
  Validator's local trust decision.

A profile MUST specify the publication channel and how authority
binding is established through it.

## Revocation Model

Profiles MUST specify a revocation model addressing at minimum:

- The revocation primitive: how an Authority Holder removes a
  delegation's effect.
- The maximum revocation latency at the Validator under nominal
  conditions and under denial-of-service against the publication
  channel.
- Whether emergency revocation (faster-than-cache invalidation) is
  supported and through what channel.
- How revocation composes with caching.

### Revocation Primitives

Three primitive revocation mechanisms are commonly used; profiles
MAY use any one or combine them:

- **Artifact removal**: the Authority Holder removes the Delegation
  Artifact from the publication channel. Subsequent Validator
  retrievals see the absence. Revocation latency is bounded by
  the cache lifetime. Used by CAA, the {{TRUST-POLICY}} DAI
  mechanism, MTA-STS.

- **Validity expiry**: the Delegation Artifact carries a
  `valid_until` (or equivalent) and becomes ineffective at that
  time. Revocation requires waiting for natural expiry or
  publishing a replacement with shorter validity. Used by RFC 9345
  Delegated Credentials, OpenID Federation Subordinate Statements.

- **Explicit revocation event**: the Authority Holder publishes a
  revocation list, transparency-log entry, or push notification.
  Validators consult or subscribe to the revocation channel.
  Used by RPKI manifests/CRLs, Verifiable Credential Status Lists,
  OCSP-like mechanisms.

Profiles MAY combine primitives (e.g., bounded validity AND
explicit revocation events for incident response). Combination
typically tightens revocation latency at the cost of operational
complexity.

### Delegation Revocation vs. Principal Revocation {#delegation-vs-principal}

Two operations are commonly conflated and MUST be distinguished by
profiles:

- **Delegation revocation**: the Authority Holder revokes the
  Delegate's right to attest. The Delegate may continue operating
  for other Authority Holders but is no longer authorized for
  this one's scope.

- **Principal revocation**: the Delegate revokes a Subject's
  principal status (a user leaves the company; a service account
  is decommissioned). The delegation from the Authority Holder to
  the Delegate is unaffected; the Delegate simply stops attesting
  for that Subject.

This pattern addresses only delegation revocation. Principal
revocation is the Delegate's responsibility and is outside the
scope of this document. Profiles MAY note operational practices
that compose the two but MUST NOT confuse them: revoking a
delegation is a much heavier-weight operation than revoking a
single principal's status.

### Eventually Consistent Revocation

Eventual consistency at multi-hour timescales is acceptable for
delegations that change rarely (namespace authority, federation
membership). Security-incident response typically requires faster
mechanisms; profiles deployed in high-security contexts SHOULD
provide an emergency revocation channel that bypasses the normal
cache lifetime.

This pattern does not require any specific revocation latency.
Profiles MUST document the latency bound they provide so deployers
can match the bound against their threat model.

## Versioning and Clock Skew {#versioning}

Two operational concerns recur across profiles and SHOULD be
specified explicitly:

- **Artifact versioning.** When an Authority Holder publishes a
  replacement Delegation Artifact, Validators may observe the old
  and new artifacts during the transition window. Profiles SHOULD
  specify how transitions are bounded (for example, by serving the
  new artifact for at least one cache lifetime before the old one
  becomes invalid) so that Validators converge without ambiguous
  intermediate states.

- **Clock skew.** Validity-bound enforcement (`valid_from`,
  `valid_until`, `exp`, `nbf`) depends on loosely synchronized
  clocks at the Authority Holder, the Delegate, and the Validator.
  Profiles SHOULD specify the assumed skew tolerance and how
  Validators handle artifacts whose validity bounds straddle the
  tolerance window.

# Verification Model {#verification}

This section specifies the abstract verification procedure.
Profiles instantiate it with concrete steps.

## Inputs

The Validator's verification operates over:

- An Assertion (typically a JWT).
- A Delegation Artifact, retrieved from the profile-specified
  publication channel.
- The Validator's local configuration: which profiles it
  implements, which Authority Sources it accepts, which local
  policies apply.

## Abstract Procedure

The Validator:

1. Validates the Assertion's basic properties per the applicable
   grant profile or token format (signature, audience, expiration,
   replay). This is grant-profile work, not delegation-pattern
   work. This step authenticates the Assertion according to its token
   format; it does not yet prove that the signer is authorized by the
   relevant Authority Holder.

2. Determines the delegation question to answer: which Authority
   Holder's delegation must be valid for the Assertion to be
   accepted? This determination uses the Assertion's claims and
   the profile's deterministic source-selection function. Claims used
   for source selection remain untrusted for authority purposes until
   the corresponding Delegation Artifact has been validated.

3. Retrieves the relevant Delegation Artifact via the profile's
   publication channel, or, for attenuated profiles
   ({{attenuated-delegation}}), extracts the Delegation Artifact
   from the presented Assertion.

4. Verifies the artifact's authority binding: the artifact came from
   the Authority Holder, or from a publication channel, signature, or
   trust chain that the profile defines as authoritative for the
   Authority Holder.

5. Validates the Delegation Artifact's structural correctness,
   signature (if any), and validity bounds.

6. Verifies the Assertion's signer matches a Delegate listed in
   the Delegation Artifact, satisfying any profile-specific
   conditions (tenant binding, claim type match, validity
   window).

7. Composes the result with any other independent trust evaluation
   categories the profile or the Validator's local policy requires
   (see {{combination-rule}}).

8. Accepts the Assertion if all checks pass; rejects otherwise.

## Exception-Handling and Fallback Model {#exception-handling}

A Validator's lookup of a Delegation Artifact produces exactly one
of three abstract states. Profiles MUST map every concrete outcome
of the lookup operation onto exactly one of these states.

### Lookup States

- **Affirmative**: a well-formed Delegation Artifact was retrieved
  through the authoritative publication channel, its signature
  (where applicable) verified, and its validity bounds hold at
  evaluation time. The Validator proceeds to evaluate the Assertion
  against the artifact's contents.

- **Negative**: the publication channel authoritatively reports the
  absence of a Delegation Artifact. Examples include DNS NXDOMAIN
  or NODATA with a valid (possibly DNSSEC-signed) authoritative
  answer, HTTPS 404 from the authority-bound origin, or an explicit
  denial entry in the publication. A Negative state is itself a
  decision by the Authority Holder (the namespace exists but no
  delegation is in effect) and carries the same normative weight
  as any other published decision.

- **Indeterminate**: the lookup did not produce an authoritative
  Affirmative or Negative result. Examples include DNS SERVFAIL,
  resolver timeout, network partition, HTTPS 5xx, TLS handshake
  failure, malformed publication-channel response, and structural
  validation failure on a retrieved artifact (invalid signature,
  unparseable payload, validity bounds outside the acceptable
  window). An Indeterminate state carries no information about the
  Authority Holder's intent.

The Affirmative / Negative / Indeterminate taxonomy is structural:
Negative is the Authority Holder's affirmative non-delegation;
Indeterminate is information the Validator could not obtain. The
distinction matters for diagnosis and audit but not for the access
decision.

### Normative Requirements

A Validator MUST classify every lookup outcome into exactly one of
the three states above before producing an accept-or-reject
decision. Profiles MUST enumerate the concrete signals on their
publication channel that map to each state.

A Validator MUST fail closed on both Negative and Indeterminate
states: the access decision MUST be reject. Profiles MUST NOT
permit any condition under which an Indeterminate state is treated
as Affirmative; doing so converts the fail-closed property into an
availability-driven downgrade attack surface
({{design-guidance}}).

A Validator MAY rely on a fresh cached Affirmative Delegation
Artifact during a transient Indeterminate state on the live
publication channel, but only if the cached artifact is within the
cache lifetime bound the profile specifies ({{cache-freshness}}).
Repeated Indeterminate states across consecutive lookups MUST NOT
extend the effective cache lifetime beyond the profile's stated
maximum; if the cache expires while the live channel remains
Indeterminate, the Validator MUST transition to a reject decision.

A Validator MUST NOT fall through to a different Authority Source
on Negative or Indeterminate states from the originally-applicable
Authority Source ({{multiple-sources}}). Fallthrough on
non-Affirmative states is the same downgrade as
extended-cache-on-Indeterminate: both convert a hard denial into a
soft one driven by adversary-controllable availability of the
authoritative publication channel.

Profiles MAY define additional lookup states (for example, a
"stale-but-signed" state for transparency-log-bound artifacts) but
MUST place any such state on the Affirmative-or-reject side of the
decision boundary. Profiles MUST NOT introduce a state that
relaxes the Indeterminate-rejects-hard requirement.

## Cache Freshness {#cache-freshness}

Profiles SHOULD specify cache lifetime guidance. The Validator's
local cache lifetime is the primary determinant of revocation
latency in the absence of an emergency revocation channel.

Recommended properties:

- An absolute maximum cache lifetime (typical values:
  1 hour for fast-changing scopes such as workload instances,
  24 hours for slow-changing scopes such as namespace authority).
- Cumulative caps on consecutive indeterminate retrievals before
  failing closed.
- HTTP `Cache-Control` and DNS TTL honored within those caps.

The pattern does not require any specific cache mechanism.

# Profile Requirements {#profile-requirements}

A profile of this pattern MUST specify the following.

## What a Profile MUST Specify

`profile_identifier`
: A unique identifier (typically a URN or URL) for the profile,
  registered in {{iana-profile-registry}}.

`authority_holder_form`
: How an Authority Holder is identified in the profile. Examples:
  DNS domain (the parent trust framework); CIMD URL (the client
  instance profile); federation entity identifier (the OIDF
  profile).

`delegation_artifact_format`
: The wire format of the Delegation Artifact. Examples: JSON
  document at a well-known HTTPS URL; DNS TXT record with a
  specific format; JWT with specific claim shapes.

`publication_channel`
: Where and how the Authority Holder publishes the Delegation
  Artifact, and how authority binding is established through that
  channel.

`authority_binding`
: The exact rule by which the Validator determines that the artifact
  is attributable to the Authority Holder. Examples include DNS
  control of a name, TLS server authentication for an HTTPS origin,
  a signature chain to a configured trust anchor, or local Validator
  configuration.

`verification_procedure`
: The concrete steps a Validator follows to validate the Delegation
  Artifact and the Assertion together. The procedure MAY retrieve
  the Delegation Artifact from an external publication channel
  (hop-by-hop evaluation) or extract it from the Assertion itself
  (attenuated delegation, {{attenuated-delegation}}); profiles MUST
  state which architecture they use and why.

`lookup_state_mapping`
: How the profile maps concrete publication-channel outcomes onto
  the Affirmative / Negative / Indeterminate states defined in
  {{exception-handling}}, and how the Validator reacts to each.

`subdelegation_policy`
: Whether the profile permits subdelegation. If it does, the
  constraints on Subdelegate scope, the chain validation
  procedure, and the security properties of chain-based
  delegation.

`revocation_model`
: How revocation works in the profile, including the maximum
  revocation latency under nominal and denial-of-service
  conditions.

`composition_rules` (when applicable)
: How the profile composes with other profiles when multiple
  delegations apply to the same Assertion.

`applicability_rule`
: How a Validator determines that this profile applies to a given
  Assertion or token request. Applicability is a policy decision made
  by the Validator, not a waiver controlled by the Assertion.

`source_selection`
: The deterministic binding function for selecting exactly which
  Authority Source's delegation applies to the Assertion. If more than
  one Authority Source can apply, the profile MUST define ordering,
  precedence, conflict handling, and failure behavior. A Validator
  MUST NOT choose a different Authority Source merely because the
  initially applicable Authority Source's evaluation fails or is
  indeterminate.

`security_considerations`
: Profile-specific threats and mitigations, including any
  authority-source compromise scenarios specific to the
  publication channel.

`privacy_considerations`
: What information the Delegation Artifact or its retrieval leaks
  about Authority Holders, Delegates, Validators, Subjects, or
  business relationships, and what mitigations are available.

## What a Profile MAY Specify

`cache_lifetime_recommendations`
: Suggested cache lifetimes for the Delegation Artifact.

`conditions_vocabulary`
: Additional conditions a Delegation Artifact may carry (tenant
  binding, claim type restrictions, validity windows).

`emergency_revocation_channel`
: An out-of-band mechanism for invalidating cached delegations
  faster than the cache lifetime would permit.

`oauth_integration_points`
: Where in OAuth protocol flows the Delegation Artifact is
  retrieved and where the Assertion is presented.

# Design Guidance for Profile Authors {#design-guidance}

A profile of this pattern SHOULD avoid the following design
mistakes. Each has been observed in practice across the
mechanisms inventoried in {{appendix-profiles}}; profile authors
SHOULD verify their design does not exhibit any of these patterns
before publication.

- **Authenticity substitution.** Treating signer authentication
  (for example, federation membership) as sufficient proof of
  delegation authority. The two are independent trust questions
  ({{categories}}); conflating them lets an authentic-but-not-
  authorized Delegate make claims it has no business making.

- **Fallback on failure.** Falling through to a different Authority
  Source when the applicable Authority Source returns a malformed
  or indeterminate result. This converts the fail-closed property
  into an availability-driven downgrade: an attacker who can drive
  the primary source to indeterminate bypasses its protections.

- **Unbounded delegation.** Allowing subdelegation without
  defining how scope, claim types, validity windows, and
  revocation semantics are constrained at each hop. Chain depth
  without constraint enables Subdelegates to over-claim relative
  to the original Authority Holder's intent.

- **Ambiguous authority selection.** Selecting the Authority
  Source from any combination of multiple Assertion claims, from
  preference ordering across candidate inputs, or from
  fallback-on-absence rules, rather than from a Single Binding
  Primitive ({{single-binding-primitive}}). Multi-input selection
  enables Unverified Claim Exploitation ({{unverified-claim}}):
  an attacker constructs an Assertion that satisfies multiple
  potential bindings and steers the Validator to a weaker
  Authority Source's policy. The binding primitive MUST be a
  single source property invariant under additional claims.

- **Assertion-driven applicability waiver.** Defining trust
  category applicability as a property of the incoming Assertion
  rather than as a property of the Validator's Published Trust
  Policy ({{applicability-bypass}}). A profile that declares a
  category "not applicable" when the Assertion originates from a
  legacy issuer, carries a legacy-flag claim, or matches any
  Assertion-internal heuristic lets the attacker waive the
  category by constructing a matching Assertion. Applicability
  MUST be determined by the Validator's local policy.

- **Unbounded cache extension.** Letting a Validator continue to
  rely on a stale Delegation Artifact indefinitely during repeated
  publication-channel failures. This effectively suspends the
  Authority Holder's ability to revoke during the
  denial-of-service window.

- **Conflating delegation and principal revocation.** Designing a
  revocation mechanism that requires revoking the entire
  delegation in order to disable a single Subject's access. See
  {{delegation-vs-principal}}.

- **Underspecified scope vocabulary.** Allowing profile-specific
  `scope` values without clear interpretation rules creates
  divergent implementations. Profiles SHOULD enumerate which
  scope kinds (subject, namespace, attribute, resource) they
  support and how each is interpreted.

- **Silent profile composition.** Implementing multiple profiles
  in the same Validator without analyzing how they compose. The
  cross-category combination rule is well-defined, but two
  profiles in the same category can introduce OR-within-category
  weakening (see {{profile-composition-risks}}).

# Security Considerations

## Open-World Delegation Security Model

This pattern targets open-world deployments. Profiles operating in
this space accept that the set of Delegates is not enumerable in
advance at the Validator; the Validator relies on the Authority
Holder's published delegation to bound the set of acceptable
asserters.

The security of any concrete profile rests on:

- The integrity of the publication channel (DNS, HTTPS, federation
  chain, etc.).
- The Authority Holder's operational practices (key management,
  publication discipline).
- The Validator's correct implementation of the verification
  procedure.
- The Validator's cache freshness policy.

A profile that does not address each of these is incomplete.

## Pre-Authentication Trust Decisions

Two structural decisions the Validator makes before the Assertion
is cryptographically authenticated determine which Authority
Holder's policy will be evaluated and whether any policy will be
evaluated at all:

- **Source selection**: which Authority Source's delegation
  applies. Made before signature validation against any
  Delegation Artifact.
- **Category applicability**: which trust categories the
  Assertion must satisfy. Made before any claim in the Assertion
  is evaluated as authoritative.

Both decisions are taken in an environment where the Assertion is
still an untrusted input. Any decision that depends on
Assertion-internal content gives an attacker who can craft the
Assertion a lever to influence the trust outcome. The two
subsections below describe specific exploitation patterns
({{unverified-claim}} for source selection,
{{applicability-bypass}} for category applicability) and the
mitigations.

## Unverified Claim Exploitation {#unverified-claim}

Source selection chooses which Authority Source's delegation
applies to the incoming Assertion. At the point of selection,
the Assertion's signature has not yet been validated against any
Delegation Artifact; the Assertion is an untrusted payload. A
profile whose source-selection algorithm considers multiple
claims, applies preference ordering, or falls back from one
input to another gives an attacker a selection lever.

The attack: the attacker constructs an Assertion whose claims
satisfy multiple potential bindings. The Validator's selection
algorithm picks the weakest (or most attacker-favorable)
Authority Source's policy. The Assertion is evaluated against a
Delegation Artifact that the legitimate Authority Holder for the
target namespace never authorized.

Worked example: an Assertion carries `sub: alice@victim.com` AND
`tenant_context: attacker-tenant.example`. A loosely written
profile selects the Authority Source by tenant context when
present, falling back to subject domain. The Validator selects
attacker-tenant.example, fetches that Authority Holder's
Delegation Artifact (which the attacker controls), validates the
Assertion against it, and accepts a claim about a victim.com
subject. The attacker has impersonated alice@victim.com using a
delegation from a namespace they control.

The mitigation is deterministic source selection
({{single-binding-primitive}}): one defined binding function, no
preference ordering, no fallback, and invariance under
attacker-controlled inputs outside that function. Profiles MUST
document the binding function in their `source_selection` matrix entry
({{profile-mapping-matrix}}) and SHOULD include a brief justification
that the selected Authority Source cannot be manipulated by other
claims in the Assertion.

## Applicability Bypass {#applicability-bypass}

The Cross-Category Combination Rule ({{combination-rule}})
requires the Validator to evaluate every applicable category. A
profile that lets category applicability depend on the incoming
Assertion gives the attacker a way to waive the category by
constructing a matching Assertion.

The attack: the profile declares "the delegation-authority
category is not applicable when the Assertion is from a legacy
issuer" (or carries a legacy-flag claim, or fails some heuristic
that signals "legacy"). The attacker constructs an Assertion
matching the legacy condition. The Validator silently skips the
entire delegation-authority evaluation; the open-world defense
layer is bypassed and the Validator falls back to authenticity
alone, which the Cross-Category Combination Rule was explicitly
designed to forbid as sufficient.

The mitigation: category applicability MUST be a property of the
Validator's Published Trust Policy, not of the incoming
Assertion. If the Validator's local policy declares that it
implements a profile in a category, that category is applicable
to every Assertion the Validator evaluates within the profile's
scope. An Assertion lacking satisfying evidence for an
applicable category MUST be rejected.

Migration scenarios where some traffic legitimately predates a
category's availability are handled at the policy layer, not by
silently waiving the category: a Validator MAY maintain
separate trust policies for distinct namespaces, audiences, or
deployment phases, but within any one policy the applicable
categories MUST be exhaustively enforced. Per-deployment-phase
policy is a configuration boundary; Assertion-driven waiver is
an attacker-controlled boundary.

## Authority Source Compromise

The authority binding (publication channel) is the highest-value
target. A compromise of the publication channel (DNS hijack, TLS
misissuance plus DNS redirect, registrar account takeover,
compromise of the federation operator's signing key) substitutes
the Authority Holder's voice with the attacker's. The Delegation
Artifacts published through the compromised channel are
indistinguishable from legitimate ones.

Profiles MUST document the publication channel's compromise model.
Profiles SHOULD recommend operational defenses (DNSSEC,
registry-lock, CAA records, Certificate Transparency monitoring,
federation key rotation policy) appropriate to their publication
channel.

This pattern does not provide cryptographic recovery from
publication channel compromise. Recovery is operational.

## Profile Composition Risks {#profile-composition-risks}

When a Validator implements multiple profiles, profile composition
creates new failure modes that no single profile addresses.

The combination rule of {{combination-rule}} requires AND across
applicable categories. A profile that lists itself in a category
already covered by another profile may create an OR-within-category
opportunity that weakens the combined posture.

Example: a Validator that accepts BOTH `domain_authorized_issuer`
(canonical DNS-first lookup with HTTPS fallback) AND
`https_subject_authority` (HTTPS-only) for subject namespace
authorization has an OR within the namespace-authorization
category. An attacker who can drive DNS to indeterminate bypasses
the fail-closed property of `domain_authorized_issuer` by causing
fallthrough to `https_subject_authority`.

Profiles MUST document any composition risks with sibling profiles
in the same category.

## Failure Mode Asymmetry

Different publication channels have different attack surfaces.
Profiles MUST NOT imply ordering of security strengths across
channels without justification. DNS-published artifacts trade
TLS-misissuance resistance for DNS-resolution risk; HTTPS-published
artifacts trade the reverse. Federation-chain artifacts trade key
management complexity for direct DNS dependence. The right choice
depends on the deployer's threat model.

## Subject Privacy

Delegation Artifacts disclose operational relationships: which
Delegates an Authority Holder has authorized, which Authority
Sources a Validator trusts. This information may be
business-sensitive (which Identity Providers a customer is
evaluating; which authorization servers a Validator accepts;
partner relationships).

Profiles SHOULD address whether Delegation Artifacts are intended
to be public, partly public, or confidential. Most existing
profiles publish openly; deployments needing confidentiality must
rely on profile-specific access control.

## Per-Assertion Revocation

This pattern does not address per-assertion revocation. Once an
Assertion is issued by a Delegate operating under a valid
delegation, the Assertion remains valid until its `exp` claim
regardless of subsequent revocation of the delegation. Per-assertion
revocation needs are addressed by OAuth Token Revocation and Token
Introspection, not by this pattern.

## Mutual and Bilateral Delegation {#mutual-delegation}

The pattern is described in unidirectional terms (Authority Holder
→ Delegate → Assertion → Validator), but real-world deployments
often involve mutual delegation: organization A's Authority Holder
delegates to organization B's Delegate AND vice versa. Federation
mesh topologies are the most common case.

A profile MAY support mutual delegation by composing two
independent delegations, one in each direction. The pattern does
not provide a mutual-delegation primitive; it relies on
independent unidirectional delegations to express the relationship.
A profile that requires symmetry (each side MUST trust the other
to participate) MUST specify how that symmetry is enforced at
validation time.

## Consent {#consent}

The pattern does not require user consent to delegation. The
Subject's involvement in the delegation lifecycle is
profile-specific:

- In namespace authority delegation, the Subject is not a party to
  the delegation; the namespace owner authorizes Delegates without
  requiring user consent.
- In attribute authority delegation, the Subject may or may not
  consent to the attestation. Some attribute-attestation
  ecosystems require explicit user consent (Verifiable Credentials
  presentation flows); others do not (employment-status
  attestation by an HR system).
- In delegated-actor scenarios (see {{actor-profile-relationship}}),
  user consent to the actor relationship is typically a separate
  OAuth flow consent and is outside the scope of the delegation
  itself.

Profiles deployed in user-facing contexts SHOULD document whether
and how user consent to delegation is obtained, recorded, and
revocable.

## Audit and Accountability {#audit}

A Validator's reliance on a Delegation Artifact is a security
decision that SHOULD be auditable. Profiles SHOULD specify what
audit-relevant information the Validator records, at minimum:

- The Delegation Artifact's identity (URL, hash, or other
  identifier).
- The Authority Holder's identifier as extracted from the
  artifact.
- The Delegate's identifier and any tenant binding.
- The result of validation (accept/reject) and, for accept, the
  combination of categories that satisfied.
- The cache state used at evaluation time (cache hit vs. live
  retrieval; cache age).

Profiles MAY recommend integration with transparency logs,
security-event publication (CAEP/SET), or SIEM-style audit trails.
The pattern itself does not require any specific audit mechanism;
the recommendation is that profiles think about audit explicitly
rather than leaving it implicit.

## Cross-Domain Federation {#cross-domain}

Authority Delegation frequently crosses administrative-domain
boundaries: an Authority Holder in domain D1 delegates to a
Delegate in domain D2, with the Validator in domain D3. Each
boundary adds operational, legal, and trust-management complexity
beyond the technical evaluation:

- The Validator must trust the publication channel under the
  Authority Holder's domain even though it operates in a different
  administrative scope.
- The Delegate may be subject to different compliance regimes than
  the Authority Holder or the Validator.
- Cross-domain key compromise has wider blast radius than
  same-domain compromise.

Profiles deployed in cross-domain scenarios SHOULD document the
trust assumptions across boundaries: which party is responsible
for what, how disputes are resolved, and how revocation
propagates. Bilateral or multilateral governance arrangements
(out-of-band) typically supplement the technical pattern in
high-stakes cross-domain deployments.

### Multi-Hop Cross-Domain Risks

When Authority Delegation evaluation spans multiple hops across
multiple trust domains (as in Identity Chaining,
{{identity-chaining-example}}), three risks compound that no
single-hop analysis catches:

- **Bilateral trust accretion.** Each hop introduces trust
  assumptions that no single party verifies end-to-end. AS A
  trusts AS B's federation membership; AS B trusts AS A's
  namespace authority; the Resource Server trusts AS B as an
  issuer. No party in the chain verifies all the assumptions
  required for the chain's overall trustworthiness. Profile
  authors deploying multi-hop flows SHOULD document the full set
  of trust assumptions explicitly so that the operational party
  responsible for each can verify it.

- **Lost scope across hops.** A delegation that binds AS A to
  asserting only about `example.com` subjects does not
  automatically bind any token AS B subsequently issues; AS B
  may re-issue the identity under its own authority without
  carrying the original constraint forward. This is the
  identity-laundering risk; see {{identity-laundering}}. Profiles
  that need scope to propagate across hops MUST specify the
  carriage mechanism (for example, claim-binding requirements on
  intermediate tokens).

- **Cross-domain auditability gap.** No single party in a
  multi-hop chain sees the full evaluation history. The
  Validator at hop N has visibility only into the inputs it
  received and its own decision; the original Authority Holder
  may never see whether its delegation was honored downstream.
  Profiles deployed in audit-sensitive contexts SHOULD specify
  how audit information is propagated or aggregated across the
  chain (transparency logs, security-event publication per
  {{audit}}, or governance-framework reporting).

## Context Token Laundering {#context-token-laundering}

{{multi-hop}} states that trust does not accrete across hops:
each hop's Validator evaluates its own delegations independently.
A specific failure mode where this property is violated is
Context Token Laundering: an access token (or assertion) minted
at hop 1 under strict delegation criteria is exchanged, chained,
or re-issued at hop 2 in a form that drops the delegation
attributes, leaving a downstream token that asserts the original
subject identity (or other constrained claim) without carrying
the original delegation's constraints forward. The downstream
Validator, presented only with the re-issued token, has no basis
for distinguishing a legitimately constrained chained token from
an unconstrained token issued under the same key.

Token-exchange flows ({{RFC8693}}), Identity Chaining
({{I-D.ietf-oauth-identity-chaining}}), and Transaction Tokens
({{I-D.ietf-oauth-transaction-tokens}}) all involve transforming
one bearer credential into another. A profile defining such a
transformation MUST specify, for each transformation:

- The delegation attributes that MUST be carried forward into the
  downstream token (typically: the Authority Holder identifier,
  the scope bound, validity windows, and any tenant or condition
  bindings).
- The attributes that MAY be narrowed or attenuated through the
  transformation.
- The attributes that MUST NOT be added or broadened.

A profile permitting the downstream token to omit the upstream's
delegation attributes effectively reduces the downstream
evaluation to "trust the issuer of this token," discarding the
open-world authority binding that motivated delegation in the
first place. This is the laundering: the upstream's
constraint-bound trust is reissued as the downstream issuer's
unconstrained trust.

Context Token Laundering is structurally distinct from the
identity laundering case in {{identity-laundering}}: identity
laundering loses the SUBJECT-NAMESPACE binding (the downstream
token names a subject under no Authority Holder's delegation);
context laundering loses the DELEGATION-CONSTRAINT binding (the
downstream token preserves the subject but loses the scope,
tenant, or conditions the original delegation imposed). Both are
composition failures and both are addressed by the same
principle: each hop's Validator MUST evaluate its own
delegations against its own applicable Authority Holders rather
than relying on upstream evaluation by reference.

Profiles deploying token-exchange-based multi-hop flows SHOULD
adopt one of the following postures:

- **Forward-propagate**: carry the upstream delegation attributes
  explicitly in the downstream token format so that downstream
  Validators can re-evaluate the original constraints.
- **Re-evaluate**: require downstream Validators to re-resolve
  the original Authority Holder's delegation against the
  downstream-asserted subject, treating the downstream token as
  carrying only the subject identifier and not the original
  delegation's authority.
- **Restrict downstream scope**: bind the downstream token to a
  scope that does not require carrying the upstream constraint
  forward (for example, by issuing a downstream token whose
  audience is internal to a closed-world trust boundary where
  the upstream delegation does not need to apply).

A profile that takes none of these postures is unsafe for
multi-hop deployment and MUST NOT be registered as a
multi-hop-applicable profile.

# Privacy Considerations

Authority Delegation publishes the relationship between Authority
Holders and Delegates. The privacy implications vary by profile:

- DNS-based publication exposes Authority Holder identity (the
  domain being queried) to the resolver path and the authoritative
  nameserver.
- HTTPS-based publication exposes Authority Holder identity to the
  policy host (when separate from the Authority Holder) and to
  network observers.
- Federation chain walking exposes the path of intermediates to
  each fetch operation.

Profiles SHOULD document the privacy properties of their
publication channel and offer mitigations where applicable
(DNS-over-HTTPS for resolver privacy, cached delegation reuse to
reduce retrieval frequency, etc.).

The pattern itself does not impose privacy requirements; it leaves
those to profiles. Profiles deployed in privacy-sensitive contexts
SHOULD consider whether the publication channel is appropriate for
the threat model.

# IANA Considerations

## Authority Delegation Profile Registry {#iana-profile-registry}

IANA is requested to establish a new registry titled "OAuth
Authority Delegation Profiles" under the "OAuth Parameters"
registry group.

Registration policy: Specification Required {{RFC8126}}.

### Registry Entry Contents

Each registry entry contains:

Profile Identifier:
: A unique identifier (URN, URL, or short string) for the
  profile.

Authority Holder Form:
: A short description of how the Authority Holder is identified
  in the profile (DNS domain, CIMD URL, federation entity
  identifier, etc.).

Delegate Form:
: A short description of how the Delegate is identified.

Publication Channel:
: A short description of the publication channel and authority
  binding.

Subdelegation:
: Whether the profile permits subdelegation (yes / no /
  profile-specific).

Reference:
: Reference to the specification defining the profile.

Profile Mapping Matrix:
: A reference into the specification to the three matrix elements
  defined in {{profile-mapping-matrix}} (Source Selection,
  Revocation Lifecycle, Transitivity Limit). The matrix MUST be
  present in the registering specification; the registry entry
  records its location.

### Profile Mapping Matrix Requirements {#profile-mapping-matrix}

A specification registering an Authority Delegation Profile MUST
include a Profile Mapping Matrix demonstrating how the profile
implements each of the three structural elements below. The matrix
is a conformance statement: IANA expert reviewers ({{RFC8126}})
SHOULD use it to assess whether the profile is structurally
complete and SHOULD reject registrations whose mapping matrix is
absent or incomplete.

`source_selection`
: The deterministic source-selection function
  ({{single-binding-primitive}}): an algorithm mapping each Assertion
  and relevant request context to exactly one Authority Source from
  explicitly defined inputs. The specification MUST identify the
  inputs used (for example, the Subject's email-domain component, the
  `client_id` URL, the federation entity identifier, or a fixed tuple
  such as `(issuer, subject namespace)`), the deterministic procedure
  for resolving those inputs to an Authority Source, the Validator's
  behavior when no Authority Source can be selected, and a
  justification that the function is invariant under
  attacker-controlled inputs outside the function. Source selection
  MUST be deterministic: two Validators presented with the same
  Assertion and request context MUST select the same Authority Source.
  Selection MUST NOT employ preference ordering across candidate
  inputs or fall back from one input to another.

`revocation_lifecycle`
: The cache-coherency and negative-caching model. The
  specification MUST specify the maximum positive cache lifetime,
  whether and how Negative lookup states
  ({{exception-handling}}) are cached and for how long, how cache
  invalidation propagates, whether an emergency revocation
  channel is provided, and the maximum revocation latency under
  both nominal and denial-of-service conditions on the
  publication channel.

`transitivity_limit`
: The explicit transitivity model: depth-1 (no subdelegation
  permitted), chained (subdelegation permitted with stated scope
  constraints), or attenuated ({{attenuated-delegation}};
  monotone-narrowing subdelegation carried in-band). The
  specification MUST state which model applies, and for chained
  or attenuated models MUST specify how Subdelegate scope is
  constrained relative to the original Authority Holder's
  delegation.

A registration that lacks any of these three matrix elements is
incomplete.

This document makes no initial registrations. {{appendix-profiles}}
contains non-normative mappings that future specifications MAY use
as starting points if they choose to register concrete profiles.

# Examples and Related Mechanisms (Non-Normative) {#appendix-profiles}

This section sketches how existing IETF and industry mechanisms map
onto the Authority Delegation Pattern. Sketches are non-normative
and illustrative; the cited specifications remain authoritative for
their own content.

## OAuth-Ecosystem Specifications {#examples-oauth}

### OAuth Identity Assertion Issuer Trust Policy + DAI

The OAuth profile family ({{oauth-profile-family}}) is documented
across two specifications: {{TRUST-POLICY}} defines the Trust
Policy document, Trust Method machinery, and Subject Authority
Determination; {{DAI}} defines the Issuer Authorization Policy
wire format and two Trust Methods consuming it.

| Element | Mapping |
|-|-|
| Authority Holder | Subject Authority (e.g., a registrable DNS domain) |
| Delegate | Assertion Issuer (OAuth authorization server) |
| Delegation Artifact | Issuer Authorization Policy ({{DAI}}) |
| Publication Channel | DNS TXT record at `_oauth-issuer-policy.{A}` and/or HTTPS at `.well-known/oauth-issuer-policy` ({{DAI}}) |
| Source Selection | Subject Identifier format extraction per {{TRUST-POLICY}} §Subject Authority Determination |
| Subdelegation | Not permitted (bounded depth-1 for namespace authorization) |
| Revocation | Cache-lifetime bounded; recommended 24-hour ceiling |

### OAuth Identity Chaining as Composed Authority Delegation {#identity-chaining-example}

Identity Chaining {{I-D.ietf-oauth-identity-chaining}} propagates a
user's identity across OAuth trust domains: a user authenticates at
Authorization Server A in Trust Domain A, an identity assertion is
presented to Authorization Server B in Trust Domain B, and B issues
a token consumed by a Resource Server in Trust Domain B. This flow
is a composition of multiple Authority Delegation evaluations
happening at different protocol points.

A representative flow:

~~~
User (alice@example.com)
   ↓ authenticates at
Authorization Server A (Trust Domain A)
   ↓ issues identity assertion (e.g., ID-JAG per {{ID-JAG}})
   ↓ assertion presented to
Authorization Server B (Trust Domain B)
   ↓ exchanges assertion for access token
   ↓ token presented at
Resource Server (Trust Domain B)
~~~

Three independent delegations apply at different points in this
flow. Each is a separate Authority Delegation evaluation governed
by the cross-category combination rule of {{combination-rule}}.

**Delegation A: namespace authority over the user's subject identifier.**

| Element | Mapping |
|-|-|
| Authority Holder | The user's namespace owner (for `alice@example.com`, the `example.com` domain) |
| Delegate | Authorization Server A |
| Delegation Artifact | An Issuer Authorization Policy ({{DAI}}) at `example.com` listing AS A as authorized |
| Validator | Authorization Server B (evaluates this delegation when accepting the identity assertion) |
| Question answered | "Is AS A entitled to assert about subjects in `example.com`?" |

**Delegation B: peer-authorization-server trust between AS A and AS B.**

| Element | Mapping |
|-|-|
| Authority Holder | Federation operator OR bilateral deployment-owner |
| Delegate | Authorization Server A |
| Delegation Artifact | Federation Subordinate Statement ({{OIDF-FEDERATION}}) OR local configuration at AS B |
| Validator | Authorization Server B |
| Question answered | "Is AS A authentic in an ecosystem AS B recognizes?" |

**Delegation C: resource-server acceptance of AS B's tokens.**

| Element | Mapping |
|-|-|
| Authority Holder | The resource server's local trust configuration or governing federation |
| Delegate | Authorization Server B |
| Delegation Artifact | RS configuration OR federation metadata listing AS B |
| Validator | Resource Server |
| Question answered | "Does the RS accept tokens issued by AS B?" |

Authorization Server B evaluates Delegations A and B independently
when accepting the identity assertion. The cross-category
combination rule applies: AS A must satisfy both the namespace
authority question AND the authenticity question. Satisfying one
does not substitute for the other.

The Resource Server evaluates Delegation C when accepting the
access token. The RS does NOT directly evaluate Delegations A or B;
those evaluations were AS B's responsibility at token issuance
time. AS B's trust in AS A does not transitively propagate to the
RS; the RS trusts AS B (Delegation C), and AS B's evaluation of
its inputs is a precondition the RS does not re-perform.

### Identity Laundering Prevented by Composition {#identity-laundering}

A naive Identity Chaining design might let assertions accrete trust
across hops: AS A asserts about `alice@example.com`; AS B accepts
the assertion and issues a token whose `iss` is AS B; the RS
accepts the token and treats the `example.com` subject claim as if
it carried the original namespace authority. This is
"identity laundering": the namespace authority binding established
at Delegation A is lost when AS B re-wraps the identity in a token
under its own issuer.

Composition of independent Authority Delegation evaluations
prevents this. Each hop's trust evaluation is bounded by its own
Delegation Artifact. The Resource Server's trust in AS B
(Delegation C) does not extend to whatever subjects AS B's tokens
name; it extends only to AS B as an issuer. If the RS needs to
verify that a subject claim in AS B's token reflects valid
namespace authority, the RS MUST evaluate that namespace
authority itself (or AS B MUST carry forward a verifiable chain).
This pattern does not provide the chain-propagation mechanism;
Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} is one
specification that addresses the wire format for forward-chained
identity context.

An alternative architecture for Identity Chaining uses attenuated
delegation ({{attenuated-delegation}}) rather than hop-by-hop
evaluation: AS A issues an attenuable assertion bounded by
`example.com`'s delegation; AS B may further narrow but cannot
broaden the scope; the RS verifies the cumulative chain
cryptographically without retrieving any external Delegation
Artifact. The attenuation model prevents identity laundering
structurally (a chain cannot be widened) and eliminates per-hop
delegation-artifact retrieval, at the cost of looser revocation
semantics (bounded by token TTL) and a closed-world chain root.
Profiles MAY choose either architecture; see
{{attenuated-delegation}} for the trade-offs.

### Where This Pattern Helps Identity Chaining

This pattern complements Identity Chaining by:

- Providing a common vocabulary for the multi-hop trust evaluation
  (Delegations A, B, C above are named uniformly).
- Applying the two-categories framing at each hop independently,
  so authenticity and authority remain distinct trust questions
  across the chain.
- Naming "identity laundering" as a composition risk that profile
  authors and deployers MUST consider.
- Making each hop's delegation independently auditable
  ({{audit}}).

This pattern does NOT define the identity assertion wire format
(ID-JAG), the cross-hop scope-binding token format (Transaction
Tokens), or the cross-domain bilateral configuration mechanism. It
provides the trust evaluation layer that the other specifications
plug into.

### Attribute Authority Delegation (Hypothetical Profile) {#attribute-authority-example}

The previous examples delegate attestation authority over subject
identity within a namespace the Authority Holder owns. Attribute
authority delegation is structurally similar but the Authority
Holder is NOT necessarily the Subject's namespace owner. Examples
include a university that may attest a former student's degree
status (even after the student leaves the `university.example`
namespace), an HR system that may attest current-employee status
for an enterprise's employees, a KYC provider that may attest
identity-verification claims, and a credentialing body that may
attest professional certifications.

A hypothetical profile for "Attribute Authority Delegation" would
map onto the pattern as follows:

| Element | Mapping |
|-|-|
| Authority Holder | The attribute-authority entity (university, employer, KYC provider, credentialing body) |
| Delegate | An Assertion Issuer authorized to attest the specific attribute |
| Delegation Artifact | A signed delegation document or registry entry binding the Delegate to specific attribute types and (optionally) specific Subject populations |
| Publication Channel | Profile-defined: a registry, signed metadata, DID-resolvable document, etc. |
| Subdelegation | Profile-defined |
| Revocation | Profile-defined; attribute attestations often require explicit revocation channels (Verifiable Credential Status Lists, OCSP-like mechanisms) |

This case is important to the pattern because it generalizes
beyond "Authority Holder owns the Subject's namespace." The Subject
is typically named in some other namespace (`alice@home.example`
might hold a degree from `university.example`). The Authority
Holder's authority is over the CLAIM TYPE (degree status) rather
than over the Subject's identity namespace.

A future profile in this space would interoperate with W3C
Verifiable Credentials at the credential layer; this pattern
specifies the OAuth-side trust evaluation when such an attestation
is presented as an identity assertion or in an access token.

### OAuth Client Instance Assertion

| Element | Mapping |
|-|-|
| Authority Holder | OAuth client owner |
| Delegate | Instance Issuer (workload identity authority) |
| Delegation Artifact | `instance_issuers` member of CIMD or local client metadata |
| Publication Channel | CIMD document at the `client_id` URL |
| Subdelegation | Profile-defined |
| Revocation | Cache-lifetime bounded; recommended 1-hour ceiling |

### OpenID Federation

| Element | Mapping |
|-|-|
| Authority Holder | Federation Trust Anchor (root) |
| Delegate | Leaf entity (chained through intermediates) |
| Delegation Artifact | Chain of Subordinate Statements from leaf to Trust Anchor |
| Publication Channel | Federation Fetch endpoints; signed JWTs |
| Subdelegation | Permitted (intermediates Subdelegate to leaves) |
| Revocation | Per-statement `exp`; trust anchor key rotation |

## Related Local-Trust Patterns {#examples-local-trust}

### OAuth Attestation-Based Client Authentication

OAuth Attestation-Based Client Authentication uses a related local
trust pattern. It is not a complete open-world authority-delegation
profile when the authorization server's own configuration is the only
Authority Source and no distinct Authority Holder publishes a
Delegation Artifact.

| Element | Mapping |
|-|-|
| Authority Holder | Authorization Server local policy or configured trust anchor |
| Delegate | Client Attester |
| Delegation Artifact | Local configuration of trusted Attester keys (not externally published) |
| Publication Channel | Out-of-band Validator configuration |
| Subdelegation | Not applicable |
| Revocation | Local-config update |

## Related IETF Mechanisms {#examples-ietf}

### CAA ({{RFC8659}})

| Element | Mapping |
|-|-|
| Authority Holder | DNS domain holder |
| Delegate | Certification Authority |
| Delegation Artifact | CAA DNS record |
| Publication Channel | DNS TXT/CAA records |
| Subdelegation | Not permitted (depth-1) |
| Revocation | DNS TTL bounded |

### MTA-STS ({{RFC8461}})

MTA-STS is not a full delegation profile because it publishes a policy
for the Authority Holder's own mail domain rather than authorizing a
separate Delegate. It is included as a related authority-publication
mechanism because it uses the same channel-binding idea: control of
DNS and an HTTPS well-known URL establishes who may publish policy for
the namespace.

| Element | Mapping |
|-|-|
| Authority Holder | Mail-receiving domain |
| Delegate | (None; MTA-STS publishes policy, not delegations to other parties) |
| Delegation Artifact | MTA-STS policy file at `.well-known/mta-sts.txt` |
| Publication Channel | DNS TXT plus HTTPS well-known URL |
| Subdelegation | Not applicable |
| Revocation | Cache-lifetime bounded |

### Resource Public Key Infrastructure (RPKI)

RPKI {{RFC6480}} is the canonical chained authority delegation system
on the public Internet. IANA delegates IP allocations to Regional
Internet Registries (RIRs); RIRs delegate to Local Internet Registries
(LIRs); LIRs delegate to operators. At each level the parent signs a
resource certificate authorizing the child for a specific resource
range. Route Origin Authorizations (ROAs) are signed delegation
artifacts asserting that a specific Autonomous System is authorized to
originate a specific IP prefix.

| Element | Mapping |
|-|-|
| Authority Holder | IANA (root); RIRs and LIRs (intermediate); operators (leaf) |
| Delegate | The child entity in each delegation, or the originating AS in a ROA |
| Delegation Artifact | Signed resource certificate; ROA |
| Publication Channel | RPKI repositories, fetchable by relying-party software |
| Subdelegation | Permitted (multi-level chain) |
| Revocation | Per-statement validity; CRLs and manifests |

RPKI demonstrates the pattern at extreme scale (the whole Internet
number space) with strict chained delegation. The Authority
Delegation pattern is compatible with RPKI's structure; an OAuth
profile inspired by RPKI would specify hierarchical signed
delegation artifacts.

### Delegated Credentials for TLS ({{RFC9345}})

Delegated Credentials let a TLS server use a short-lived credential
signed by a long-lived certificate. The long-lived certificate acts
as an Authority Holder authorizing the short-lived credential as a
Delegate for the same name.

| Element | Mapping |
|-|-|
| Authority Holder | Long-lived TLS certificate (CA-issued) |
| Delegate | Short-lived delegated credential |
| Delegation Artifact | Delegated Credential structure signed by the long-lived certificate |
| Publication Channel | Delivered in-band in the TLS handshake |
| Subdelegation | Not permitted (depth-1) |
| Revocation | Short validity windows replace explicit revocation |

This is the closest IETF analog at the TLS layer: a long-lived
authority delegates a short-lived signing capability bounded by a
validity window, with no separate publication channel needed.

## Industry and W3C Mechanisms {#examples-industry}

### Sigstore (Industry Practice)

Sigstore's signing flow chains delegation across OIDC, an
ephemeral-certificate CA (Fulcio), a transparency log (Rekor), and
the signing tool (Cosign):

| Element | Mapping |
|-|-|
| Authority Holder | OIDC Identity Provider (authoritative for the user's identity) |
| Delegate | Fulcio-issued short-lived X.509 certificate bound to the OIDC identity |
| Delegation Artifact | The Fulcio certificate, with OIDC identity in a SAN extension; the Rekor transparency log entry |
| Publication Channel | Fulcio CA infrastructure plus Rekor transparency log |
| Subdelegation | Not permitted (depth-1 from OIDC identity to Fulcio cert) |
| Revocation | Short validity windows; transparency log used to detect anomalies |

Sigstore is non-IETF but widely deployed for software-supply-chain
signing. An OAuth-anchored profile of Authority Delegation for
software signing would likely follow the Sigstore pattern: OIDC
identity provides the Authority Holder; a short-lived certificate or
JWT acts as the Delegation Artifact; a transparency log provides
audit.

### W3C Verifiable Credentials

The W3C Verifiable Credentials Data Model defines a three-party trust
triangle: Issuer, Holder, and Verifier. The Issuer makes claims about
a Subject (the Holder, in many flows). The Verifier validates the
credential and accepts the Issuer's claims under some trust
assumption.

| Element | Mapping |
|-|-|
| Authority Holder | The party with authority over the claim type (often, but not always, the Issuer itself) |
| Delegate | The VC Issuer |
| Delegation Artifact | The trust-establishment mechanism the Verifier uses to accept the Issuer (DID resolution, trust-list, registry membership, ecosystem governance framework) |
| Publication Channel | DID method-specific; trust list publication; governance-framework artifacts |
| Subdelegation | Profile-defined; some ecosystems support delegated issuance |
| Revocation | Credential Status List, registry updates |

VC and Authority Delegation overlap in the abstract pattern but
operate on different artifacts (VC credentials vs OAuth assertions)
and through different infrastructure. A deployment using both
typically uses VC for credential issuance and Authority Delegation
for OAuth-layer issuer trust.

### Macaroons and Capability-Token Systems

Macaroons {{MACAROONS}}, Biscuit tokens, SPKI/SDSI ({{RFC2693}}),
UCAN, and W3C zcap-ld realize attenuated delegation
({{attenuated-delegation}}). The delegation chain is carried within
the token rather than retrieved from an external publication
channel; subsequent holders may add caveats that narrow the
delegation but cannot broaden it.

| Element | Mapping |
|-|-|
| Authority Holder | Initial token issuer (the root of the capability chain) |
| Delegate | Initial token holder; each subsequent attenuator is a Subdelegate |
| Delegation Artifact | The token itself, with its chain of caveats |
| Publication Channel | In-band: the token IS the artifact, carried with the Assertion |
| Subdelegation | Permitted, monotone-narrowing only (attenuation) |
| Revocation | Token TTL; explicit third-party caveats; out-of-band invalidation lists |

OAuth Token Exchange ({{RFC8693}}) with downscoping is the
OAuth-standard attenuation primitive: a new token is issued with
NARROWER scope than the input. The exchange happens at an
authorization server (rather than free-form holder-side attenuation
as in macaroons), but the architectural intent (downstream tokens
never broader than upstream) is the same.

These sketches illustrate the pattern's generality across both
external-publication and in-band-attenuation architectures. Each
existing mechanism predates this document and is not modified by
it. The sketches show that the Authority Delegation Pattern is a
description of an existing class of mechanisms rather than an
invention of this document.

--- back

# Document History

This appendix is non-normative and will be removed before publication.

-00

  * initial draft
