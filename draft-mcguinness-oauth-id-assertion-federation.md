---
title: "OpenID Federation Trust Methods for Identity Assertion Grants"
abbrev: "Federation Trust Methods"
docname: draft-mcguinness-oauth-id-assertion-federation-latest
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
  - OpenID Federation
  - client recognition

stand_alone: true
pi: [toc, sortrefs, symrefs]

author:
  -
    name: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7523:
  RFC7591:
  RFC10070:
  RFC8414:
  RFC8705:
  OIDF-FEDERATION:
    title: "OpenID Federation 1.1"
    target: https://openid.net/specs/openid-federation-1_1.html
    date: 2026-05-05
  OIDF-FEDERATION-CONNECT:
    title: "OpenID Federation for OpenID Connect 1.1"
    target: https://openid.net/specs/openid-federation-connect-1_1.html
    date: 2026-05-05
  ID-JAG:
    title: "Identity Assertion JWT Authorization Grant"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    date: false
  TRUST-FRAMEWORK:
    title: "OAuth Identity Assertion Trust Framework"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-id-assertion-framework/
    date: false

informative:
  OIDF-WKB:
    title: "OpenID Federation Well-Known Binding 1.0"
    target: https://dickhardt.github.io/well-known-binding/main.html
    date: 2026-11-11
    author:
      - name: Dick Hardt
        ins: D. Hardt
  DAI:
    title: "OAuth Domain-Authorized Issuer Trust Method"
    target: https://datatracker.ietf.org/doc/draft-mcguinness-oauth-domain-authorized-issuer/
    date: false

  I-D.ietf-oauth-client-id-metadata-document:
    title: "OAuth Client ID Metadata Document"
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/
    date: false

---

--- abstract

This document defines two Trust Methods, based on OpenID Federation,
for the OAuth Identity Assertion Trust Framework. Under
`openid_federation`, a Resource Authorization Server authenticates an
Assertion Issuer through a federation trust chain, any Trust Marks
its Trust Policy requires, and a federation-bound signing key. Under
`openid_federation_client`, it recognizes the OAuth client that
presents an identity assertion, including a client whose first
request is a token request. This document also extends the framework
with the `client_recognition` Trust Method category and the
`client_trust_methods` Trust Policy member.

--- middle

# Introduction

The OAuth Identity Assertion Trust Framework {{TRUST-FRAMEWORK}} lets
a Resource Authorization Server publish a Trust Policy that names the
Trust Methods an identity assertion must satisfy. The framework
defines the Trust Method categories, the combination rule, and the
token-endpoint procedure. It leaves concrete Trust Methods to
companion specifications, such as {{DAI}}.

This document defines two Trust Methods based on OpenID Federation
{{OIDF-FEDERATION}}:

- `openid_federation`, in the `issuer_authentication` category,
  accepts an Assertion Issuer whose trust chain validates to a listed
  trust anchor, that holds any required Trust Marks, and whose
  assertion signature verifies under a federation-bound key
  ({{trust-method-openid-federation}}).
- `openid_federation_client`, in the `client_recognition` category,
  accepts the OAuth client that presents the assertion when that
  client is a federation Entity and authenticates with a key that its
  federation metadata publishes
  ({{trust-method-openid-federation-client}}).

This document also extends {{TRUST-FRAMEWORK}} with the
`client_recognition` category, the `client_trust_methods` Trust
Policy member, and the processing they add ({{client-recognition}}).

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses the terms Resource Authorization Server, Assertion
Issuer, identity assertion, Trust Policy, Trust Method, Subject
Authority, and Consumer as defined in {{TRUST-FRAMEWORK}}. OpenID
Federation terms, such as Entity, Entity Identifier, Entity
Configuration, Subordinate Statement, trust anchor, and Trust Mark,
are used as defined in {{OIDF-FEDERATION}}.

# The `openid_federation` Trust Method {#trust-method-openid-federation}

`issuer_authentication`-category methods are satisfied by evidence
that the Assertion Issuer belongs to a recognized ecosystem.
Membership alone does not establish authority over any particular
subject namespace.

Under the `openid_federation` method, the Assertion Issuer is
acceptable if its OpenID Federation {{OIDF-FEDERATION}} trust chain
validates to a listed trust anchor and the leaf holds any Trust Marks
the policy requires.

~~~ json
{
  "method": "openid_federation",
  "trust_anchors": ["https://federation.example.org"],
  "trust_marks": [
    {
      "trust_mark_type": "https://federation.example.org/tm/loa3",
      "issuer": "https://federation.example.org"
    }
  ]
}
~~~

`trust_anchors`
: REQUIRED. Non-empty JSON array of OpenID Federation trust anchor
entity identifiers. An identifier does not authenticate the trust
anchor: the Resource Authorization Server holds each listed trust
anchor's public keys, initially obtained in a secure out-of-band way
({{OIDF-FEDERATION}} §4), not from the trust anchor's Entity
Configuration alone. A chain to a listed trust anchor whose keys it
does not hold is Indeterminate. {{trust-anchor-rollover}} covers
later key changes.

`trust_marks`
: OPTIONAL. Non-empty JSON array of Trust Mark requirement objects. When
present, each requirement object listed MUST be satisfied by at least
one Trust Mark in the leaf's Entity Configuration. Each
requirement object has:

  `trust_mark_type`
  : REQUIRED. String. The Trust Mark type identifier.

  `issuer`
  : OPTIONAL. String. The Entity Identifier of the Trust Mark
  Issuer expected to have signed the Trust Mark. When absent, the
  Trust Mark's `iss` claim MUST appear in the non-empty array that
  the terminal trust anchor's `trust_mark_issuers` claim gives for
  that Trust Mark type. If the trust anchor gives no such array,
  or an empty one, the requirement is not satisfied. This
  deliberately narrows OpenID Federation, which allows any issuer
  when no issuer restriction is published.

The Trust Policy governs Trust Mark requirements for identity
assertions. A Resource Authorization Server that also advertises
Trust Mark requirements through discovery metadata SHOULD keep those
advertisements consistent with this policy. Discovery advertisements
do not replace enforcement of its Trust Method requirements.

Trust chain validation, metadata policy resolution, and validation
of the Trust Marks that requirement 4 relies on follow
{{OIDF-FEDERATION}} §10.2 and §7.3. If any fails, this Trust Method
fails. Federation `metadata_policy` constrains Entity Type metadata
in Entity Statements, not members carried only in documents that an
extension binds by digest. {{OIDF-WKB}} is such an extension and
reads no Entity Type metadata. A Superior that needs to constrain
such a member, such as `jwks_uri`, marks it `essential`
({{OIDF-FEDERATION}} §6.1.3.1.7), so that the leaf's policy-applied
Entity Type metadata must carry it. For a key source, requirement 3
then selects that value.

In addition to the procedures in {{OIDF-FEDERATION}}, the Resource
Authorization Server MUST apply the following framework-specific
requirements:

1. **Trust anchor match.** The terminal trust anchor of the validated
   chain MUST exactly match one of the listed `trust_anchors` values. A
   chain to any listed trust anchor satisfies this requirement; failure
   to build a chain to one listed trust anchor does not prevent
   evaluating chains to the others ({{TRUST-FRAMEWORK}} §Multiple
   Evidence Sources Within a Category).

2. **Entity type constraint.** The leaf's policy-applied federation
   metadata MUST declare one of the entity types `openid_provider`
   or `oauth_authorization_server` ({{OIDF-FEDERATION-CONNECT}}
   §5.1.2 and §5.1.3). For this type-declaration check, a present
   Entity Type member whose value is an object declares that type,
   even when the object is empty. This does not waive metadata
   validation required by the applicable federation specifications.
   The presence of other entity types alone does not satisfy this
   requirement.

3. **Federation-bound key resolution.** The signing key for the
   identity assertion JWT MUST be taken from a key set that the
   leaf's policy-applied metadata designates, or that is otherwise
   bound to the leaf's Entity Identifier through the validated trust
   chain.

   The key sources defined in {{OIDF-FEDERATION}} §5.2.1 satisfy this
   requirement. Each is the `jwks`, `jwks_uri`, or `signed_jwks_uri`
   value in the leaf's policy-applied `openid_provider` or
   `oauth_authorization_server` metadata, processed per that
   specification. If both `openid_provider` and
   `oauth_authorization_server` metadata carry a key source, a key from
   either satisfies this requirement.

   For `jwks_uri`, the trust chain authenticates the URI, not the key
   set it returns. The returned key set relies on HTTPS to that
   endpoint unless an extension, such as {{OIDF-WKB}}, also binds the
   key set. When the metadata's `jwks_uri` names the URL whose key set
   an extension binds, a consumer that applies the binding treats a
   failed binding as failure to validate that key source.

   Specifications that extend this Trust Method MAY define additional
   key sources that satisfy it. If the policy-applied metadata carries
   a key source, that source is the only one permitted. An
   extension-defined source is used only when the metadata carries
   none. Failure to retrieve or validate the metadata's key source
   MUST NOT trigger fallback to an extension-defined source.

   A JWK Set obtained any other way, including from the `jwks_uri` of
   {{RFC8414}} metadata that no qualifying source binds, MUST NOT be
   used. This prevents a downgrade in which an attacker who can alter
   the Assertion Issuer's {{RFC8414}} metadata, but not its federation
   metadata, substitutes the key source. If no qualifying key source
   is available, or its binding cannot be verified, the outcome is
   Indeterminate ({{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed).

4. **Trust Mark satisfaction.** If the Trust Method object contains
   `trust_marks`, each requirement object MUST be satisfied by at
   least one Trust Mark that appears in the `trust_marks` claim of
   the leaf's Entity Configuration and meets all of the following:

   - It validates per {{OIDF-FEDERATION}} §7.3.
   - Its `trust_mark_type` equals the requirement's `trust_mark_type`.
   - Its `iss` claim equals the requirement's `issuer`, or, when
     `issuer` is absent, satisfies the rule in the `issuer` member
     definition above.

   The type in the Entity Configuration entry MUST match the type in
   the validated Trust Mark JWT. The issuer is taken from the JWT's
   `iss` claim, not from the enclosing entry. {{OIDF-FEDERATION}}
   §7.3 requires trust in the Trust Mark Issuer before validation.
   That trust MUST be established through a trust chain terminating
   at the same trust anchor as the leaf's chain. That is the trust
   anchor whose `trust_mark_issuers` claim applies when `issuer` is
   absent.

The proposed Well-Known Binding mechanism {{OIDF-WKB}} is one such
extension. It lets an {{RFC8414}} authorization server bind its
existing JWK Set to its Entity Identifier by listing the set's digest
in its Entity Configuration. It changes how issuer-authentication
evidence is bound, not who holds authority over a subject namespace.
This document does not require support for it. A consumer without a
supported qualifying key source returns Indeterminate. Where
{{OIDF-WKB}} §8.4 has an Entity return `trust_mark_required` for a
missing Trust Mark, naming the missing types, a Resource
Authorization Server applying the framework returns the error that
{{TRUST-FRAMEWORK}} §Resource Authorization Server Processing
specifies, which keeps trust-evaluation detail from public clients.

Lookup states ({{TRUST-FRAMEWORK}} §Lookup States and Fail-Closed):

- **Affirmative**: a fully validated chain from the leaf to a listed
  trust anchor.
- **Negative**: any of these:

  - the fetch of the leaf's Entity Configuration returns HTTP 404 or
    410 (the Assertion Issuer does not participate in federation);
  - the leaf's chains validate, but none reaches a listed trust
    anchor; or
  - every Superior in the leaf's `authority_hints` answers the fetch
    endpoint request for the leaf with the `not_found` error
    ({{OIDF-FEDERATION}} §8.1.2 and §8.9).

- **Indeterminate**: any other failure to obtain or validate the
  chain, such as:

  - an Entity Statement fetch that fails or times out;
  - a signature that cannot be verified because key material is
    unavailable; or
  - an expired statement that cannot be refreshed.

The framework-specific requirements above are evaluated against an
Affirmative chain. Every requirement MUST hold on that one chain: the
metadata, key source, and Trust Marks used are those of the chain
that satisfies requirement 1. A chain that fails requirement 1 is
Negative, as above. In each case below, the leaf does not satisfy
this Trust Method and the lookup state is unchanged:

- It does not declare a required entity type (requirement 2).
- Its assertion signature does not verify under the permitted key
  source (requirement 3).
- It holds no valid Trust Mark meeting a requirement (requirement 4).

A Trust Mark that fails validation per {{OIDF-FEDERATION}} §7.3 does
not meet a requirement. Only a retrieval the method needs and cannot
complete is Indeterminate, namely a retrieval of:

- the permitted key source;
- a Trust Mark Issuer's chain; or
- a Trust Mark Status endpoint the Resource Authorization Server
  relies on.

Neither Negative nor Indeterminate satisfies this Trust Method
({{TRUST-FRAMEWORK}} §Fail-Closed Requirements). Entity Statement
caching follows the statements' own `exp` values per
{{OIDF-FEDERATION}}, bounded by the consumer's local cache ceiling. A
cached chain MUST NOT be used past the earliest `exp` in the chain, or
more than 24 hours after it was obtained. The same 24-hour limit applies
to a key set obtained through requirement 3. These limits hold while
a fresh lookup is Indeterminate: this method allows no stale use
beyond them ({{TRUST-FRAMEWORK}} §Fail-Closed Requirements). A Trust
Mark that requirement 4 relies on MUST be re-validated whenever the
chain is. For a Trust Mark without `exp`, that includes its Trust
Mark Status endpoint ({{OIDF-FEDERATION}} §8.4) when the Trust Mark
Issuer publishes one. A Trust Mark is not used past its own `exp`.

The fetches that {{TRUST-FRAMEWORK}} §Outbound Fetches governs
include the Entity Statements reached through the leaf's
`authority_hints` and the chain of a Trust Mark Issuer.

See {{relationship-to-oidf}} for positioning against OpenID
Federation.

# Client Recognition {#client-recognition}

This section extends {{TRUST-FRAMEWORK}} with a Trust Method category
for the OAuth client that presents an identity assertion, a Trust
Policy member that lists methods in that category, the processing
they add, and one method in that category.

## The `client_recognition` Category {#client-recognition-category}

This document registers the following Trust Method category
({{iana-category}}), in addition to those of {{TRUST-FRAMEWORK}}
§Trust Method Categories:

`client_recognition`
: Is the OAuth client presenting the assertion, as authenticated at the
token endpoint, recognized by an authority the Resource Authorization
Server accepts? Methods in this category are listed in
`client_trust_methods`. They evaluate the authenticated client, never a
claim in the assertion ({{client-recognition-processing}}, step 5f).
Recognition does not authorize the client to act for a particular user.
The grant profile's client binding does that (for ID-JAG, {{ID-JAG}}
§4.4.1).

For a `client_recognition` method, the party whose evaluation its
specification defines ({{TRUST-FRAMEWORK}} §Requirements on Trust
Method Specifications) is the authenticated client.

As for `issuer_authentication` ({{TRUST-FRAMEWORK}} §Multiple
Evidence Sources Within a Category), every evidence source the Trust
Policy lists for this category is equally trusted, so a
`client_recognition` Trust Method specification MAY accept evidence
from any of them. For `openid_federation_client`, this means a chain
to any listed trust anchor.

`client_recognition`-category methods are satisfied by evidence about
the OAuth client that presents an identity assertion at the token
endpoint. These methods evaluate the client that the grant profile's
client authentication identifies ({{client-recognition-processing}},
step 5f). For ID-JAG, the assertion's `client_id` claim, set by the
Assertion Issuer, identifies the same client ({{ID-JAG}} §4.4.1).
Recognizing a client does not authorize any Assertion Issuer, and
authorizing an Assertion Issuer does not recognize any client.

The OAuth Client ID Metadata Document
{{I-D.ietf-oauth-client-id-metadata-document}} is complementary: it
lets an authorization server obtain a client's metadata and keys
from its `client_id` URL but leaves the trust decision to that
server. A client recognition method could use it as a key source
only together with evidence that the Resource Authorization Server
trusts. `openid_federation_client` does not use it: a client whose
keys are published only in a Client ID Metadata Document does not
satisfy that method ({{trust-method-openid-federation-client}}).

## The `client_trust_methods` Member {#client-trust-methods}

This document registers the following Trust Policy member
({{iana-member}}):

`client_trust_methods`
: OPTIONAL. Non-empty JSON array of Trust Method objects
({{TRUST-FRAMEWORK}} §Trust Methods) whose methods belong to the
`client_recognition` category ({{client-recognition-category}}). When
present, it states the trust requirements the Resource Authorization
Server enforces against the client presenting an identity assertion. An
assertion is rejected unless the authenticated client satisfies the
Trust Method combination rule in {{TRUST-FRAMEWORK}} §Resource
Authorization Server Processing against the methods listed here. An
object in this member counts only toward the `client_recognition`
category. A method outside that category is malformed here.

A Trust Policy that contains `client_trust_methods` MUST list
`client_trust_methods` in `crit` ({{TRUST-FRAMEWORK}} §Critical
Members). A Resource Authorization Server that implements this
document processes `client_trust_methods` whether or not `crit` lists
it. A Resource Authorization Server that implements only
{{TRUST-FRAMEWORK}} rejects a policy that lists the member in `crit`
instead of ignoring its requirements on clients, and a client that
does not implement the member cannot read that policy as placing no
requirement on clients. A policy that omits the member from `crit`
gets neither protection: a server that implements only
{{TRUST-FRAMEWORK}} ignores the member ({{TRUST-FRAMEWORK}} §Trust
Policy Document) and can accept assertions presented by clients that
the member would reject.

## Processing {#client-recognition-processing}

This section adds to the procedure of {{TRUST-FRAMEWORK}} §Resource
Authorization Server Processing, which lets a specification that
registers a category state where in that procedure its checks run
and how its errors rank. Step numbers below are those of that
procedure, and the additions to steps 1 and 5 are part of those
steps.

**Client processing.** In step 4 of {{TRUST-FRAMEWORK}} §Client
Processing, if the Trust Policy contains `client_trust_methods`, the
client also determines whether it satisfies them, for example whether
its `client_id` is an Entity Identifier in a listed federation.

**Policy selection (step 1).** A Resource Authorization Server that
holds more than one Trust Policy selects one by client only when the
request's client authentication verifies against a registration it
holds. {{trust-method-openid-federation-client}} limits which
registrations qualify. Any other request gets the policy that applies
without regard to the client, such as the one it publishes
({{TRUST-FRAMEWORK}} §Metadata Publication).

**Client evaluation (step 5).** When the policy contains
`client_trust_methods`, step 5 also verifies that the authenticated
client satisfies the Trust Method combination rule. For the
`client_recognition` category, the party that step 5b evaluates is
the authenticated client. This document adds step 5f:

**Step 5f.** When the policy contains `client_trust_methods`, the
Resource Authorization Server MUST evaluate the `client_recognition`
category against the client that the request's client authentication
({{RFC6749}} Section 2.3) identifies, never against a claim in the
assertion. If the request carries no client authentication that the
Resource Authorization Server accepts, it MUST reject the request with
an `invalid_client` error ({{RFC6749}} Section 5.2).

**Ordering.** When the policy contains `client_trust_methods`, the
Resource Authorization Server MUST authenticate the client, and evaluate
the `client_recognition` category (step 5f), before step 2. Before
authenticating the client, the Resource Authorization Server MUST NOT
perform any lookup except those a `client_recognition` method needs to
obtain the client's authentication keys. An example is the trust chain
and key set that `openid_federation_client` resolves.

**Errors.** An error from step 5f or from a `client_recognition` method
takes precedence over any other error.

**Outbound fetches.** The fetches that {{TRUST-FRAMEWORK}} §Outbound
Fetches governs also include a client's Entity Configuration, trust
chain, and key sets.

**Grant-profile binding.** {{ID-JAG}} §4.4.1 lets a client that does
not authenticate prove possession of the key in the ID-JAG's `cnf` claim
instead. When the Trust Policy contains `client_trust_methods`, step 5f
rejects such a request with `invalid_client`. This narrows {{ID-JAG}}
§4.4.1.

## The `openid_federation_client` Trust Method {#trust-method-openid-federation-client}

Under the `openid_federation_client` method, the client is acceptable
if it is an OpenID Federation {{OIDF-FEDERATION}} Entity whose trust
chain validates to a listed trust anchor and that holds any Trust
Marks the policy requires.

~~~ json
{
  "method": "openid_federation_client",
  "trust_anchors": ["https://federation.example.org"]
}
~~~

The method has the members `trust_anchors` and `trust_marks`, defined
as for `openid_federation` ({{trust-method-openid-federation}}).

The client's `client_id` is its Entity Identifier, as in Automatic
Registration ({{OIDF-FEDERATION-CONNECT}} §12.1 and §12.4).
{{OIDF-FEDERATION-CONNECT}} §12.1 starts Automatic Registration at an
Authentication Request, and §12.1.5 says that bare OAuth 2.0
deployments can use it. This method defines the case §12.1 does not
describe: a client whose first request is a token request. This
method places none of the obligations of {{OIDF-FEDERATION-CONNECT}}
§12.1 on the client toward the Resource Authorization Server. This
method is evaluated for each token request that presents an identity
assertion. The `client_id` MUST equal the `sub` of the client's
Entity Configuration by simple string comparison. A client whose
`client_id` is not an Entity Identifier ({{OIDF-FEDERATION}} §1.2)
does not satisfy this method, and the lookup state is Negative.

The Resource Authorization Server evaluates this method as it
evaluates `openid_federation`, with the client's Entity Identifier as
the leaf. Trust chain validation, the lookup states, caching, and
requirements 1 and 4 apply unchanged. Where that section refers to the
Assertion Issuer, it refers here to the client. Every requirement MUST
hold on one trust chain: the metadata, key set, and Trust Marks used
are those of the chain that satisfies requirement 1. Requirements 2
and 3 are replaced by the following:

- **Entity type constraint.** The leaf's policy-applied federation
  metadata MUST declare the entity type `oauth_client` or
  `openid_relying_party` ({{OIDF-FEDERATION-CONNECT}} §5.1.4 and
  §5.1.1), under the type-declaration rule of requirement 2. The
  client metadata is the policy-applied `oauth_client` metadata, or,
  if the leaf declares no `oauth_client` type, the
  `openid_relying_party` metadata. {{OIDF-FEDERATION-CONNECT}} §12
  uses `oauth_client` for OAuth 2.0 profiles other than OpenID
  Connect. Metadata from the two types is never combined.
- **Federation-bound client authentication key.** Client
  authentication MUST verify under a key in a key set that the client
  metadata designates through `jwks`, `jwks_uri`, or `signed_jwks_uri`
  ({{OIDF-FEDERATION}} §5.2.1), processed as that section specifies.
  For `jwks_uri`, the trust chain authenticates the URI, not the key
  set it returns. That key set relies on HTTPS to that endpoint. The
  permitted client authentication methods are:

  - JWT client authentication ({{RFC7523}} Sections 2.2 and 3, as
    updated by {{RFC10070}}) with a JWT signed by such a key, whose
    `aud` claim is the Resource Authorization Server's issuer
    identifier as its sole value, as a string or as an array with only
    that element ({{RFC10070}} Section 4); and
  - self-signed certificate mutual TLS ({{RFC8705}} Section 2.2) with
    a certificate equal to the first certificate in the `x5c` value
    of such a key, without certificate chain validation.

  {{OIDF-FEDERATION-CONNECT}} §12.1.1.2 (see also §12.1.1.2.1) names
  these two methods for a Pushed Authorization Request. There, as on
  a first token request, the client authenticates before any
  registration exists. The methods are narrower than
  {{OIDF-FEDERATION-CONNECT}} §12.1.4, under which a server should
  accept any mutually supported method. The `aud` value is the issuer
  identifier, not an Entity Identifier, because the Resource
  Authorization Server need not be a federation Entity.

  A client that authenticates any other way does not satisfy this
  method. Keys the Resource Authorization Server knows only through
  another source, such as out-of-band registration or a client
  metadata document, do not count. If no such key set is available,
  or it cannot be retrieved or validated, the outcome is
  Indeterminate.

- **Declared client metadata.** The client metadata MUST declare the
  method the client authenticates with, in
  `token_endpoint_auth_method` or
  `token_endpoint_auth_methods_supported`
  ({{OIDF-FEDERATION-CONNECT}} §12.1.4), and MUST list the token
  request's grant type in `grant_types` ({{RFC7591}} Section 2). A
  client that omits them does not satisfy this method. The
  {{RFC7591}} defaults, `client_secret_basic` and
  `authorization_code`, would not permit the request either. A
  Superior can constrain both through metadata policy
  ({{OIDF-FEDERATION}} §6.1).

When the Resource Authorization Server relies on this method to
satisfy the `client_recognition` category, it MUST NOT accept the
assertion unless client authentication verified under a key this
method permits.

The keys come from the trust chain, so the Resource Authorization
Server resolves the client's trust chain and key set before it
verifies client authentication. The error for a failure depends on
the condition, not on when it is detected. When more than one
condition applies, the first in this list determines the error:

1. `invalid_client` ({{RFC6749}} Section 5.2) when any of the
   following holds:

   - the `client_id` is not an Entity Identifier;
   - no trust chain reaches a listed trust anchor;
   - the chain or the key set is Indeterminate;
   - the metadata declares neither entity type;
   - the client authenticates with a method its metadata does not
     declare; or
   - client authentication does not verify under a key in the
     federation key set.

2. `unauthorized_client` ({{RFC6749}} Section 5.2) when the client is
   authenticated but not recognized:

   - it holds no Trust Mark that meets a requirement (requirement 4);
   - the Trust Mark evaluation is Indeterminate; or
   - its metadata does not list the request's grant type.

3. Any failure of the assertion or of an issuer method gets the error
   that {{TRUST-FRAMEWORK}} §Resource Authorization Server Processing
   specifies.

Any token request can name an `https` `client_id`, and so cause these
fetches before the client is authenticated. They include a fetch for
every Entity in the client's `authority_hints` ({{OIDF-FEDERATION}}
§18.1). Before the client is authenticated, the Resource
Authorization Server:

- MUST limit the number of `authority_hints` it follows;
- MUST apply a global rate limit to trust chain resolution for
  unauthenticated requests;
- SHOULD cache a Negative or Indeterminate outcome per `client_id`
  for a bounded period; and
- SHOULD also rate-limit trust chain resolution per `client_id`.

A client that does not publish its Entity Configuration
({{OIDF-FEDERATION}} §9) can send its trust chain in a `trust_chain`
header parameter of its client assertion ({{OIDF-FEDERATION}}
§4.3). The Resource Authorization Server MAY use that chain in place
of resolving one, after validating it per {{OIDF-FEDERATION}} §10.2.
It resolves the chain itself instead if any Entity Statement in the
supplied chain was issued more than 24 hours earlier. Without a
published Entity Configuration or a valid supplied chain, the client
does not satisfy this method.

The Resource Authorization Server caches the client's key set no
longer than the trust chain ({{trust-method-openid-federation}}). A
client assertion signed with a key whose `kid` the cached key set
lacks MAY prompt one fresh fetch of the key set, rate-limited per
client, before the request is rejected.

Clients registered through Explicit Registration
({{OIDF-FEDERATION-CONNECT}} §12.2) receive a `client_id` that need
not be their Entity Identifier. This method covers such a client only
when its `client_id` is its Entity Identifier.

A Resource Authorization Server can apply a Trust Policy without
`client_trust_methods` to a request that authenticates as a client the
server registered itself, against that registration
({{client-recognition-processing}}). It MUST NOT do so for a request
whose `client_id` is an `https` URL, even when it holds a registration
for that `client_id`. One example is an Explicit Registration that kept
the Entity Identifier ({{OIDF-FEDERATION-CONNECT}} §12.2.2). A public
client does not authenticate against its registration ({{RFC6749}}
Section 2.3). Every other request, including one that names a registered
`client_id` without authenticating as that client, gets the policy with
`client_trust_methods`. A Resource Authorization Server that already
uses `https` URLs as `client_id` values for clients outside a federation
cannot list `client_trust_methods` without rejecting those clients. This
document defines no other way to combine registered and federation
clients. {{example-client-recognition}} walks through first contact,
rejection, cached trust, and coexistence.

# Security Considerations

The security considerations of {{TRUST-FRAMEWORK}} and
{{OIDF-FEDERATION}} apply.

## Trust Anchor Key Rollover {#trust-anchor-rollover}

{{TRUST-FRAMEWORK}} §Evidence Source Compromise covers compromise of
the federation operator's signing key.

Trust anchor key rollover in {{OIDF-FEDERATION}} §11.2 has no
mandatory hold-down period: a compromised current key can introduce
a replacement immediately. Resource Authorization Servers SHOULD
obtain trust anchor keys through an independent channel where the
Federation Operator provides one, as {{OIDF-FEDERATION}} §11.3
recommends. Routine rollover alone does not provide recovery from
trust anchor compromise.

## Key Substitution by a Superior {#superior-key-substitution}

{{TRUST-FRAMEWORK}} §Key Binding Is Outside Namespace Authorization
lists the parties that decide which key speaks for an authorized
issuer identifier.

Under `openid_federation`, the leaf's policy-applied metadata supplies
the key source ({{trust-method-openid-federation}}). An Immediate
Superior can supply that metadata ({{OIDF-FEDERATION}} §3.1.1). A
compromised intermediate or trust anchor can therefore substitute the
key of a listed issuer beneath it, and both categories then pass.

Requirement 3 accepts a key source from either `openid_provider` or
`oauth_authorization_server` metadata. A Superior that constrains a
leaf's key source through metadata policy therefore constrains it
for both types; otherwise the leaf can supply an unconstrained key
source under the other type.

## Client Recognition {#client-recognition-security}

The `client_recognition` category evaluates the client identified by the
token request's client authentication
({{client-recognition-processing}}, step 5f), not by the assertion. An
attacker who controls an assertion's claims therefore cannot choose
which client is evaluated ({{TRUST-FRAMEWORK}} §Unverified Claim
Exploitation). Recognition admits a client to the exchange. It does not
authorize the client to act for a particular user, says nothing about
the subject or the Assertion Issuer, and leaves the other categories in
force.

`openid_federation_client` requires the client to authenticate with
a key its federation metadata publishes. Without that rule, a client
could authenticate with a key the Resource Authorization Server holds
from another source, such as a client metadata document, and be
recognized as the federation Entity its `client_id` names. A Resource
Authorization Server that also registers clients selects their policy
only after they authenticate against their registration. Naming a
registered `client_id` without authenticating as that client
therefore does not avoid the client method. A request whose
`client_id` is an `https` URL gets the client method even when a
registration exists for that URL. A party holding such a
registration therefore cannot redeem an assertion issued to the
federation Entity of that name
({{trust-method-openid-federation-client}}).

A compromised intermediate or trust anchor can substitute a client's
keys through the metadata in its Subordinate Statement
({{OIDF-FEDERATION}} §3.1.1), as it can an Assertion Issuer's
({{superior-key-substitution}}). It can therefore authenticate as any
client beneath it.

Under a client method that lets a client opt in by publishing
evidence, such as a DNS record, a client that has published nothing
is Negative, as under a namespace method. Under the combination
rule, that client is rejected wherever the category is listed unless
another listed client method is satisfied. This document defines no
mechanism for accepting such clients through local policy instead.

# Privacy Considerations

This document adds no privacy considerations beyond those of
{{TRUST-FRAMEWORK}} and {{OIDF-FEDERATION}}.

# IANA Considerations

## Identity Assertion Trust Method Categories Registration {#iana-category}

This document registers the following entry in the Identity
Assertion Trust Method Categories registry ({{TRUST-FRAMEWORK}}
§Identity Assertion Trust Method Categories Registry):

Category Name:
: `client_recognition`

Description:
: Establishes that the client presenting the assertion is recognized
by an authority the Resource Authorization Server accepts

Change Controller:
: IETF

Specification Document:
: This document, {{client-recognition-category}}

## Identity Assertion Trust Methods Registrations {#iana-methods}

This document registers the following entries in the Identity
Assertion Trust Methods registry ({{TRUST-FRAMEWORK}} §Identity
Assertion Trust Methods Registry):

Identifier:
: `openid_federation`

Category:
: `issuer_authentication`

Parameters:
: `trust_anchors` (array of string, REQUIRED); `trust_marks` (array
  of object, OPTIONAL; see {{trust-method-openid-federation}})

Change Controller:
: IETF

Reference:
: This document

Identifier:
: `openid_federation_client`

Category:
: `client_recognition`

Parameters:
: `trust_anchors` (array of string, REQUIRED); `trust_marks` (array
  of object, OPTIONAL; see {{trust-method-openid-federation-client}})

Change Controller:
: IETF

Reference:
: This document

## Trust Policy Members Registration {#iana-member}

This document registers the following entry in the Identity
Assertion Issuer Trust Policy Members registry ({{TRUST-FRAMEWORK}}
§Trust Policy Members Registry):

Member Name:
: `client_trust_methods`

Member Description:
: Trust Method requirements enforced for the client presenting an
identity assertion

Decision-Affecting:
: Yes

Change Controller:
: IETF

Specification Document:
: This document, {{client-trust-methods}}

--- back

# Design Rationale

This appendix is non-normative.

## Relationship to OpenID Federation {#relationship-to-oidf}

OpenID Federation {{OIDF-FEDERATION}} answers "is this issuer an
authentic ecosystem member?"; {{TRUST-FRAMEWORK}} adds "is this issuer
authorized for *this* namespace?". Federation membership feeds the
`issuer_authentication` category through the `openid_federation`
Trust Method ({{trust-method-openid-federation}}). Namespace
authority comes from the separate `subject_namespace_authorization`
category. The framework composes with federation rather than
replacing it.



# OpenID Federation Walkthrough {#example-federation-walkthrough}

This appendix is non-normative. Like the example in
{{TRUST-FRAMEWORK}} §Overview, it combines `openid_federation` with
`domain_authorized_issuer`, and it adds trust-chain validation, Trust
Mark satisfaction, and federation-bound JWKS resolution.

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
      "trust_marks": [{
        "trust_mark_type": "https://federation.example.org/tm/loa3",
        "issuer": "https://federation.example.org"
      }]
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
      "trust_mark_type": "https://federation.example.org/tm/loa3",
      "trust_mark": "eyJ...(JWT signed by federation.example.org)"
    }
  ]
}
~~~

The `trust_mark` value is a JWT with JOSE header `typ`
`trust-mark+jwt`. Its decoded payload (illustrative):

~~~ json
{
  "iss": "https://federation.example.org",
  "sub": "https://idp.partner.example",
  "trust_mark_type": "https://federation.example.org/tm/loa3",
  "iat": 1790236800,
  "exp": 1821772800
}
~~~

Requirement 4 of {{trust-method-openid-federation}} matches against this
payload: its `trust_mark_type` equals both the enclosing entry's type
and the requirement's, and its `iss` equals the requirement's
`issuer`. The enclosing entry carries no issuer. {{OIDF-FEDERATION}}
§7.3 separately requires `sub` to equal the leaf's Entity
Identifier.

The Federation Intermediate's Subordinate Statement about the leaf
constrains `issuer` and requires `jwks_uri` via `metadata_policy`
({{OIDF-FEDERATION}} §6.1):

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
_oauth-issuer-policy.partner.example. IN TXT (
    "v=oauth-issuer-policy1;"
    "authority=partner.example;"
    "issuer=https://idp.partner.example" )
~~~

**Verification.** When an ID-JAG arrives with
`iss: https://idp.partner.example`, `email: alice@partner.example`,
and `email_verified: true`, the Resource Authorization Server:

1. **`issuer_authentication` (openid_federation).** Walks the
   federation chain per {{OIDF-FEDERATION}}: fetches the leaf
   Entity Configuration, the Intermediate's Subordinate Statement
   about the leaf, and the Trust Anchor's Subordinate Statement
   about the Intermediate. Validates all signatures, applies
   `metadata_policy`, and verifies the Trust Mark signature.

2. **Framework-specific checks ({{trust-method-openid-federation}}).**
   The terminal trust anchor matches `trust_anchors`; the policy-applied
   metadata declares entity type `openid_provider`; the `loa3` Trust
   Mark satisfies the requirement; the ID-JAG signing key is taken only
   from the federation-resolved JWKS, not from the assertion `iss` URL's
   `.well-known/oauth-authorization-server`.

3. **`subject_namespace_authorization` (domain_authorized_issuer).**
   Extracts `partner.example` from the email claim, queries
   `_oauth-issuer-policy.partner.example`, confirms
   `https://idp.partner.example` is an authorized issuer.

4. **Cross-category combination rule.** Both categories succeed;
   the Resource Authorization Server issues an access token.

**Selected failure variants:**

- The chain does not terminate at the listed trust anchor:
  `invalid_grant`.
- The leaf lacks the required Trust Mark: `invalid_grant`.
- The federation-resolved JWKS does not match the ID-JAG signing key:
  `invalid_grant`. The metadata key source takes precedence, so there
  is no fallback to a separate JWKS referenced by
  `.well-known/oauth-authorization-server`. An extension-defined key
  source is eligible only when the metadata carries none
  ({{trust-method-openid-federation}}, requirement 3).
- `partner.example` does not list the Assertion Issuer in DAI:
  `invalid_grant`, even though federation membership is valid.

## Trust Mark Requirement Without an Issuer

With the same cast, the Trust Policy could omit `issuer` from the
requirement object:

~~~ json
{
  "method": "openid_federation",
  "trust_anchors": ["https://federation.example.org"],
  "trust_marks": [
    { "trust_mark_type": "https://federation.example.org/tm/loa3" }
  ]
}
~~~

The acceptable issuers then come from the trust anchor's Entity
Configuration, which includes (excerpt):

~~~ json
"trust_mark_issuers": {
  "https://federation.example.org/tm/loa3": [
    "https://federation.example.org",
    "https://assurance.example.net"
  ]
}
~~~

The leaf's Trust Mark satisfies the requirement because its `iss`
appears in that array. So would a Trust Mark of the same type issued
by `https://assurance.example.net`, provided that issuer's own trust
chain terminates at the same trust anchor (requirement 4 of
{{trust-method-openid-federation}}). The requirement would not be
satisfied if the array for this type were empty, which
{{OIDF-FEDERATION}} §3.1.2 reads as allowing any issuer, or if the
type were absent from `trust_mark_issuers`.



# Client Recognition Walkthrough {#example-client-recognition}

This appendix is non-normative. It follows `openid_federation_client`
({{trust-method-openid-federation-client}}) from a client's first
token request to later requests, and shows federation clients beside
clients the Resource Authorization Server registered itself.

**Cast:**

- Resource Authorization Server `https://api.saas.example`, with
  token endpoint `https://api.saas.example/token`.
- Trust anchor `https://federation.example.org`.
- Client `https://agent.example`, an agent platform that is a
  federation Entity of type `oauth_client`. It has never contacted
  the Resource Authorization Server.
- Assertion Issuer `https://idp.partner.example`, which issues the
  client an ID-JAG for `alice@partner.example` whose `client_id`
  claim is `https://agent.example`.

**Trust Policy.** The Resource Authorization Server publishes:

~~~ json
{
  "resource_authorization_server": "https://api.saas.example",
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag"
  ],
  "issuer_trust_methods": [
    {
      "method": "openid_federation",
      "trust_anchors": ["https://federation.example.org"]
    },
    { "method": "domain_authorized_issuer" }
  ],
  "client_trust_methods": [
    {
      "method": "openid_federation_client",
      "trust_anchors": ["https://federation.example.org"]
    }
  ],
  "crit": ["client_trust_methods"]
}
~~~

**Client metadata.** The client's Entity Configuration, at
`https://agent.example/.well-known/openid-federation`, includes:

~~~ json
{
  "iss": "https://agent.example",
  "sub": "https://agent.example",
  "metadata": {
    "oauth_client": {
      "grant_types": ["urn:ietf:params:oauth:grant-type:jwt-bearer"],
      "token_endpoint_auth_method": "private_key_jwt",
      "jwks_uri": "https://agent.example/jwks.json"
    }
  },
  "authority_hints": ["https://federation.example.org"]
}
~~~

**First contact.** The client's first request to the Resource
Authorization Server is a token request:

~~~ http
POST /token HTTP/1.1
Host: api.saas.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&assertion=eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0Ii...
&client_id=https%3A%2F%2Fagent.example
&client_assertion_type=
urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6...
~~~

Line breaks in the request body are for display only. The client
assertion's JOSE header carries `typ` `client-authentication+jwt`
({{RFC10070}} Section 4), and its claims are:

~~~ json
{
  "iss": "https://agent.example",
  "sub": "https://agent.example",
  "aud": "https://api.saas.example",
  "exp": 1793880000,
  "jti": "c1f0..."
}
~~~

**Processing.** The Resource Authorization Server:

1. Applies the policy above, because the `client_id` is an `https`
   URL ({{trust-method-openid-federation-client}}).
2. Resolves the client's Entity Configuration and a trust chain to
   `https://federation.example.org`.
3. Checks that the policy-applied metadata declares `oauth_client`.
4. Fetches the key set at `jwks_uri`, verifies the client assertion's
   signature with a key in it, and checks that its `aud` is
   `https://api.saas.example` and that `private_key_jwt` is the
   declared method. The client is now authenticated, and because
   `grant_types` lists the request's grant type, it is recognized.
   Steps 2 through 4 are the only lookups before the client is
   authenticated.
5. Validates the ID-JAG, including that its `client_id` claim is
   `https://agent.example` ({{ID-JAG}} §4.4.1).
6. Evaluates `openid_federation` for `https://idp.partner.example`
   and `domain_authorized_issuer` for `partner.example`. With the
   client recognized in step 4, every category is satisfied, so it
   issues an access token.

**Rejections:**

- The request carries no client authentication: `invalid_client`
  ({{client-recognition-processing}}, step 5f).
- The `client_id` is not an Entity Identifier: the method is
  Negative, the request has no accepted client authentication, and
  the result is `invalid_client`.
- The trust chain ends at a trust anchor that is not listed:
  `invalid_client`.
- The client assertion is signed with a key that is not in the
  federation key set, even one published in a client metadata
  document: `invalid_client`.
- The Entity Configuration fetch times out or returns a 5xx status:
  the method is Indeterminate, and the result is `invalid_client`.
- The policy required a Trust Mark that the client does not hold:
  the client is authenticated but not recognized, and the result is
  `unauthorized_client`.
- The client lacks a required Trust Mark and the ID-JAG has also
  expired: `unauthorized_client`, because client recognition is
  evaluated before the ID-JAG.
- The ID-JAG's `client_id` claim names a different client:
  `invalid_grant` ({{ID-JAG}} §4.4.1).
- The client is recognized, but the Assertion Issuer fails an issuer
  method: `invalid_grant`.

**Later requests.** The Resource Authorization Server caches the
trust chain no longer than its earliest `exp`
({{trust-method-openid-federation}}), and the key set no longer than
the chain. Later requests use the cached chain, and each is still
evaluated against the policy. After the client rotates its keys, a
signature with a key the cached key set lacks can prompt one fresh
fetch of the key set, rate-limited per client, before rejection. If
the federation stops recognizing the client, the change takes effect
when the cached chain expires.

**Coexistence with registered clients.** The Resource Authorization
Server also has confidential clients it registered itself, with opaque
`client_id` values and credentials it holds. A request that
authenticates as one of those clients gets a policy without
`client_trust_methods` ({{client-recognition-processing}}). Any other
request, including one that names a registered `client_id` without
authenticating as that client and any request whose `client_id` is an
`https` URL, gets the published policy above and is admitted only
through the client method.

# Document History

This appendix is non-normative and will be removed before publication.
A number in parentheses is a closed issue in the
[issue tracker](https://github.com/mcguinness/draft-mcguinness-oauth-id-assertion-framework/issues)
that this document shares with {{TRUST-FRAMEWORK}}, which holds the
details of that change.

-00

  * Initial version, split from {{TRUST-FRAMEWORK}} (#38); includes
    client recognition (#32) and the OpenID Federation 1.1 alignment
    (#20).
